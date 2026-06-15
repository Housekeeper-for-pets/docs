# 더미데이터를 운영 분포(skew)로 모사하니 인덱스 설계가 통째로 뒤집혔다

> 카테고리: 성능(인덱스) · 2026-06-05 · 도메인: 공고(post) 인덱스 · PET-171

## 문제
공고 검색 인덱스를 처음엔 더미데이터 **OPEN 70% / CLOSED 30%** 분포 위에서 설계했다. 그런데 실서비스는 정반대다 — 공고는 시간이 지나면 마감/만료되어 **CLOSED가 누적**되고 OPEN은 소수가 된다(성숙한 서비스면 OPEN 5~15% 수준). 비현실적 분포 위에서 내린 인덱스 결정은 운영에서 틀린다.

## 문제 정의 — 왜 분포가 인덱스 선택을 바꾸는가
대상 쿼리:
```sql
SELECT p.*, m.region FROM post p
LEFT JOIN member m ON p.member_id = m.id
WHERE p.deleted = false AND p.status = 'OPEN'
ORDER BY p.created_at DESC LIMIT 10;
```
인덱스 후보 2개가 있었다.
- `idx_post_deleted_created` **(deleted, created_at)** — 2컬럼
- `idx_post_deleted_status_created` **(deleted, status, created_at)** — 3컬럼(status 포함)

옵티마이저는 **추정 선택도**로 둘 중 하나를 고른다. OPEN이 흔하면 "created_at 정렬 인덱스를 역순으로 읽으며 OPEN 10건만 금방 채우는" 2컬럼이 유리하고, OPEN이 희소하면 "status로 먼저 OPEN만 모으는" 3컬럼이 유리하다.

## 해결 — 운영 비율로 재삽입 후 재측정
`PostDummyDataInserter`의 status 생성 비율을 한 줄 수정:
```java
// 변경 전: OPEN 70%
String status = random.nextDouble() < 0.70 ? "OPEN" : "CLOSED";
// 변경 후: OPEN 10% / CLOSED 90% (실서비스 누적 분포 반영)
String status = random.nextDouble() < 0.10 ? "OPEN" : "CLOSED";
```
재삽입 후 분포 확인 → **OPEN 1,024 / CLOSED 9,006 (10.2% / 89.8%)**. 같은 인덱스 2개를 둔 채 같은 쿼리로 EXPLAIN을 다시 떴다.

## 전/후 결과 — 옵티마이저가 선택을 뒤집었다

| 항목 | Before: OPEN 70% (비현실) | After: OPEN 10% (운영 분포) |
| --- | --- | --- |
| 선택된 인덱스 | `idx_post_deleted_created` (2컬럼) | **`idx_post_deleted_status_created` (3컬럼)** |
| `key_len` | 1 (deleted까지) | **2** (deleted + status까지) |
| status 처리 | `Using where`로 **후처리** | **인덱스로 직접** 좁힘 |
| 추정 rows | ~10,080 (전체) | **1,024** (OPEN 건수로 정확히 축소) |
| 정렬 | (정렬은 흡수되나 status 후처리) | `Backward index scan` → **filesort 없음** |
| 그래서 내린 결론 | "status 인덱스 **DROP** (옵티마이저가 안 씀)" | "status 인덱스 **유지 = 필수**" |

> ⚠️ 측정 도구 정직하게: 이 공고 쿼리는 `EXPLAIN`(실행계획)으로 확인했고 **`EXPLAIN ANALYZE`가 아니라 actual time(ms) 수치는 없다.** 증거는 *인덱스 선택 전환·`key_len` 1→2·추정 rows 10,080→1,024·filesort 제거*다. (시터 region 쿼리는 별도 케이스로, 거기서도 핵심 근거는 EXPLAIN 구조 지표 `rows 10,036→404`다 — [perf-02](./perf-02-explain-based-index-design.md).)

## 최종 인덱스 확정 — 둘 다 유지
| 인덱스 | 결정 | 근거(실측) |
| --- | --- | --- |
| `idx_post_deleted_created` (deleted, created_at) | ✅ 유지 | status 없는 기본 목록(빈도 최고)이 이걸 타고 filesort 제거 |
| `idx_post_deleted_status_created` (deleted, status, created_at) | ✅ 유지 | OPEN 10% 현실 분포에서 status 필터 쿼리가 선택, rows 10,080→1,024 + filesort 제거 |

처음 70% 분포에서 내렸던 "3컬럼 인덱스 DROP" 결론이, 현실 분포에서 **정반대(필수 유지)**로 뒤집혔다.

## 배운 점
더미데이터의 핵심은 **양(규모)이 아니라 분포(선택도)의 현실성**이다. "고르게 분산해 카디널리티만 확보"하면 오히려 운영과 멀어진다. **같은 쿼리·같은 인덱스인데 데이터 분포(OPEN 70%→10%) 하나로 옵티마이저의 인덱스 선택이 뒤집혔고, 그게 곧 인덱스 설계 결론을 뒤집었다.** 인덱스 설계는 반드시 **운영 분포를 모사한 데이터 위에서** 해야 한다.

이 케이스는 [`perf-02`](./perf-02-explain-based-index-design.md)의 "선택도 0(전부 APPROVED)이면 인덱스 무시"와 같은 뿌리다 — 한쪽은 분포가 쏠려서 인덱스가 *살고*, 다른 쪽은 분포가 없어서(전부 같은 값) 인덱스가 *죽는다*. 둘 다 *분포가 옵티마이저를 좌우한다*는 같은 원리.
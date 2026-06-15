# 🧯 ForPets 트러블슈팅

> 작성자: **선경안** · 집사조
> 담당 영역: 시터/공고 검색 · Redis 캐싱 · 성능 개선(인덱스/부하 테스트) · 후기·평점 · 로그/모니터링
> 기간: 2026-05-12 ~ 2026-06-11

펫시터 매칭 플랫폼 **ForPets**(forpetscare.uk)를 구현하며 직접 마주친 문제와 해결을, 제가 담당한 도메인 위주로 정리했습니다. 각 항목은 `문제 / 원인·진단 / 해결 / 배운 점` 형식입니다.

---

## 🔍 검색 (시터/공고)

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [시터 목록 N+1](search/search-01-sitter-list-n-plus-1.md) | JOIN을 해도 `selectFrom`이면 N+1 → Tuple로 추출 |
| 2 | [공고 목록 N+1](search/search-02-post-list-n-plus-1.md) | 자식 컬렉션은 `IN절 일괄조회 + groupingBy` |

## 🗄 캐싱 (Redis)

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [Graceful Degradation](cache/cache-01-graceful-degradation.md) | Redis 죽어도 DB 폴백, 서비스 안 멈춤 |
| 2 | [승인/거절 Evict 범위](cache/cache-02-approve-reject-evict-scope.md) | 상태 전이 의미대로 무효화 범위 차등 |
| 3 | [cross-domain stale](cache/cache-03-cross-domain-stale.md) | 회원수정 후 시터 캐시 stale → `SitterCacheEvictor` |
| 4 | [ElastiCache CONFIG 차단](cache/cache-04-elasticache-config-blocked.md) | 운영 Redis는 `CONFIG` 막힘 → `INFO`로 |
| 5 | [maxmemory 환경 분리](cache/cache-05-maxmemory-env-split.md) | 로컬 command vs ElastiCache 파라미터 그룹 |

## ⚡ 성능 (인덱스 / 부하 테스트)

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [더미데이터 규모의 함정](performance/perf-01-dummy-data-scale-trap.md) | 13건으론 효과 안 보임 + `ddl-auto: create` 함정 |
| 2 | [EXPLAIN 기반 인덱스 설계](performance/perf-02-explain-based-index-design.md) | 진짜 병목은 member.region 풀스캔, 선택도 0 인덱스 무시 |
| 3 | [캐시 Before/After 측정 방법론](performance/perf-03-cache-before-after-measurement.md) | 조건 동일·워밍업·규모 확대 → P99 98%↓ |
| 4 | [운영 분포(skew) 모사](performance/perf-04-realistic-data-distribution.md) | CLOSED 누적 분포로 모사하니 인덱스 설계가 뒤집힘 |

## ⭐ 후기·평점

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [평점 연동 A/B/C](review/review-01-rating-sync-tradeoff.md) | 캐시 컬럼(B) 채택 + 전체 재계산 |

## 📊 로그·모니터링

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [캐시 hit/miss 계측](monitoring/monitor-01-cache-hit-miss-instrumentation.md) | `@Cacheable` 프록시 구조 → `MeteredCacheManager` |
| 2 | [Prometheus scrape](monitoring/monitor-02-prometheus-scrape.md) | pull 방향·지표 등록 시점·테스트 NPE |
| 3 | [Grafana 프로비저닝 No data](monitoring/monitor-03-grafana-provisioning-no-data.md) | datasource UID 하드코딩 + dashboard-as-code |

---

> 📌 반복해서 배운 것: ① 측정이 이상하면 측정 **환경**을 먼저 의심 ② 로컬 ≠ 배포 ③ 문서와 코드는 같은 커밋 흐름 ④ 프레임워크가 도는 **레이어**를 알아야 계측·캐싱·인증이 잡힌다 ⑤ 캐시는 필수 인프라가 아니라 성능 수단 ⑥ 근거를 남기는 게 곧 실력.

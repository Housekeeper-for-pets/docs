# 옵티마이저는 선택만 할 뿐, 인덱스 설계는 개발자의 몫이다

> 카테고리: 성능(인덱스) · 2026-06-04 · 도메인: 인덱스 튜닝

## 문제
문서(캐시 키 구성)만 보고 복합 인덱스를 먼저 설계했다가 **2~3번**정도 다시 하게되었다. 만든 인덱스를 옵티마이저가 쓰지 않았다.

## 문제 정의 / 진단
대표 검색 쿼리를 `EXPLAIN ANALYZE`로 떠보니, 시터 검색의 진짜 병목은 시터 테이블이 아니라 **`member.region` 풀스캔**이었다. 처음 만든 `idx_sitter_search`(approval_status 선두)는 **선택도 0(전부 APPROVED)**이라 옵티마이저가 무시했다.

## 해결
옵티마이저가 실제 택하는 경로를 EXPLAIN으로 확인하고 거기에 맞춰 인덱스를 확정. `idx_sitter_search`는 근거를 문서화하고 **DROP**, region 병목은 idx_member_region으로 해결. **actual time 27.8ms → 16.6ms**로 SLO를 통과했다.

**EXPLAIN 구조 변화 — region 필터 쿼리 (Before / After)**

| 항목 | Before (인덱스 없음) | After (`idx_member_region`) |
| --- | --- | --- |
| 접근 type | **ALL** (풀스캔) | **ref** (인덱스 조회) |
| 사용 key | NULL | **idx_member_region** |
| 스캔 rows | **10,036** | **404** |
| Extra | `Using where; Using filesort` | region 필터 후 ~404행만 남아 정렬이 인메모리 수준 |

`idx_sitter_search`(approval_status 선두)는 검색 대상이 **전부 APPROVED라 선택도 0** → 행을 전혀 못 줄여 옵티마이저가 무시했다(그래서 DROP). 실제로 스캔 범위를 `10,036 → 404`로 줄인 건 **member.region 인덱스**였다.

## 배운 점
인덱스는 **EXPLAIN으로 옵티마이저의 실제 선택을 보고 개발자가 재설계하는 과정**이다. 선택도 0인 선두 컬럼 인덱스는 만들어도 무시된다.
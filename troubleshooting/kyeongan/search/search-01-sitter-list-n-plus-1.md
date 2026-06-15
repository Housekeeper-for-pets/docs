# 시터 목록 조회 N+1 — JOIN을 해도 selectFrom이면 터진다

> 카테고리: 검색 · 2026-05-18 · 도메인: 시터 검색(QueryDSL)

## 문제
시터 목록을 조회할 때 각 시터의 `member.region`을 루프에서 매번 다시 조회해, **시터 수만큼 추가 쿼리**가 나갔다(N+1).

## 원인
QueryDSL에서 `member`를 JOIN까지 걸어놓고도 `selectFrom(sitter)`로 **sitter 엔티티만 꺼내고 member는 버렸다**. 결과적으로 region이 필요할 때마다 다시 지연 로딩이 발생.

## 해결
`select(sitter, member.region)` **Tuple**로 JOIN 결과를 함께 추출하도록 변경. 쿼리 수가 `N+2` → **2회로 고정**됐다.

## 해결 과정
`git stash`로 적용 전 코드와 적용 후 코드의 실제 발생 쿼리 수를 직접 비교해, 개선을 눈으로 검증했다.

## 배운 점
JOIN을 걸었다고 N+1이 사라지는 게 아니다. `selectFrom`은 JOIN 대상을 결과로 안 가져오므로, **Tuple로 꺼내야 JOIN을 실제로 활용**하는 것이다.

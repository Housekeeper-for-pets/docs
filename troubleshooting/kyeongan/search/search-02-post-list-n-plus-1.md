# 공고 목록 조회 N+1 — 자식 컬렉션은 IN절 + groupingBy로

> 카테고리: 검색 · 2026-05-20~21 · 도메인: 공고 검색(QueryDSL)

## 문제
공고 목록에서 `memberRegion` + `postPet` + `postTimeSlot`을 각 공고마다 루프 조회 → **`1 + 3N` 쿼리**.

## 해결
- 단일 값(region)은 **Tuple JOIN**으로 함께 추출.
- 자식 컬렉션(pets, timeSlots)은 `findAllByXxxIn`으로 **IN절 일괄 조회** 후 `groupingBy`로 `Map<공고ID, List>` 구성해 메모리에서 매핑.

## 배운 점
단일 연관은 JOIN으로, **1:N 자식 컬렉션은 `IN절 일괄 조회 + groupingBy`**로 N+1을 한 번에 해소한다. 컬렉션을 JOIN으로 풀면 카테시안 곱·페이징 문제가 생기므로 분리 조회가 더 안전하다.

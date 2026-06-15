# Redis가 죽어도 서비스는 멈추면 안 된다 — Graceful Degradation

> 카테고리: 캐싱 · 2026-05-27 · 도메인: Redis 캐싱 · 장애 대응

## 문제
Spring Cache는 Redis 장애 시 예외를 그대로 던져, **캐시와 무관한 조회 API까지 500**이 났다.

## 해결
`CacheErrorHandler`를 구현한 `GracefulDegradationCacheErrorHandler`로 GET/PUT/EVICT/CLEAR 실패를 `log.warn`으로 삼키고 **DB 직접 조회로 폴백**. 로그에 `[CACHE_DEGRADED]` 키워드를 상수로 박아 필터링·알람 연동을 쉽게 했다.

## 배운 점
**캐시는 성능 수단이지 필수 인프라가 아니다.** Redis가 죽으면 서비스는 느려질 수는 있어도 멈추면 안 된다. 장애 시 동작을 명시적으로 설계해야 "캐시 때문에 전체가 죽는" 사고를 막는다.

# maxmemory는 로컬과 운영을 나눠 적는다

> 카테고리: 캐싱 · 2026-06-11 · 도메인: Redis 운영

## 문제
maxmemory / eviction 정책을 한 곳에 통일해 적으려 했으나, 로컬 Docker Redis와 운영 ElastiCache는 설정 위치 자체가 다르다.

## 해결
- **로컬**: docker-compose Redis `command`에 `--maxmemory 256mb --maxmemory-policy volatile-lru` 지정. 256mb는 가용 메모리의 60~70%로 OS·Redis 헤드룸을 확보하는 근거.
- **운영**: AWS ElastiCache는 노드 타입이 maxmemory를 결정하므로 `command` 지정 불가 → **파라미터 그룹**에서 관리.

## 배운 점
인프라 설정은 "로컬 = 운영"이 아니다. **환경별로 설정 주체(컨테이너 command vs 관리형 파라미터 그룹)가 다르다**는 걸 문서에 분리해 적어야 검증 때 혼선이 없다.

# 운영 Redis(ElastiCache) CONFIG 차단 — INFO로 우회

> 카테고리: 캐싱 · 2026-06-10(배포검증) · 도메인: Redis 운영

## 문제
EC2에서 `redis-cli`가 `command not found`. 설치 후에도 `CONFIG GET maxmemory-policy`가 막혀 정책을 확인할 수 없었다.

## 원인
EC2에 redis-cli 미설치 + **ElastiCache는 보안상 `CONFIG` 명령을 차단**한다.

## 해결
- 접속: `docker run --rm -it --network host redis:7-alpine redis-cli -h <host> -p 6379 --tls`
  - env `SSL_ENABLED=true`라 `--tls`(보안접속 옵션) 필요, PASSWORD가 비어 있어 AUTH 불필요.
- `maxmemory-policy`: `CONFIG` 대신 **`INFO memory`의 `maxmemory_policy` 필드**로 확인. 런타임 적용값이라 파라미터 그룹 설정보다 더 강력한 증거다.

## 배운 점
관리형 Redis는 로컬과 명령 가용 범위가 다르다. 운영 점검 절차에 "**CONFIG 불가 → INFO로 대체**"를 명시해두면 현장에서 헤매지 않는다.

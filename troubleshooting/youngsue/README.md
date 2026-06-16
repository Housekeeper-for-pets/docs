# ForPets 트러블슈팅

> 작성자: 박영수  
> 담당 영역: 실시간 알림 · Redis Streams/Kafka 전략 구조 · SSE · 배포/멀티 인스턴스 · 모니터링  
> 기간: 2026-05 ~ 2026-06

ForPets 최종 프로젝트에서 실시간 알림 시스템과 운영 배포 구조를 고도화하며 마주친 문제와 해결 과정을 정리했습니다.  
각 문서는 `문제 / 원인 / 해결 / 결과 / 배운 점` 형식으로 작성했습니다.

---

## 알림 / 비동기 처리

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [Redis Streams Consumer Group 메시지 미소비](notification/notification-01-redis-stream-consumer-group-offset.md) | Stream에는 메시지가 쌓였지만 Consumer가 못 읽음 → Consumer Group 생성/offset/ACK 흐름 정리 |
| 2 | [멀티 인스턴스 SSE 알림 누락](notification/notification-02-multi-instance-sse-pubsub-fanout.md) | SSE emitter는 JVM 메모리라 인스턴스가 다르면 누락 → Redis Pub/Sub fan-out 추가 |

## 배포 / 멀티 인스턴스

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [단일 EC2 내부 멀티 인스턴스 운영 전환](infra/infra-01-single-ec2-multi-instance-compose.md) | `docker run` 직접 실행에서 `docker-compose.prod.yml` 기반 운영 구성으로 전환 |
| 2 | [EC2 Docker Compose 플러그인 누락](infra/infra-02-docker-compose-plugin-missing.md) | 운영 compose 배포로 바꾸자 EC2에 compose 플러그인이 없어 CI 실패 |

## 모니터링

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [Prometheus target 환경 분리](monitoring/monitoring-01-prometheus-target-env-split.md) | 로컬 싱글/로컬 멀티/운영 멀티의 target이 달라 설정 파일 분리 |

---

> 반복해서 배운 것: 멀티 인스턴스에서는 "메모리에 있다"는 말이 곧 "인스턴스마다 따로 있다"는 뜻이다. SSE, 스케줄러, 모니터링 target, 배포 스크립트는 모두 단일 서버일 때와 다르게 봐야 한다. 또한 운영 환경은 로컬 Docker Compose와 같지 않다. RDS, ElastiCache 같은 관리형 서비스를 쓰면 compose 파일의 책임과 CI의 책임을 분리해야 한다.

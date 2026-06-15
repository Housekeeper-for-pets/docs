# ForPets 트러블슈팅

> 작성자: 최길중  
> 담당 영역: 인증 · WebSocket · 채팅 동시성 · 쿼리 최적화  
> 기간: 2026-05 ~ 2026-06

ForPets에서 인증/인가, 쿠폰, 채팅 기능을 구현하며 마주친 문제와 해결 과정을 정리했습니다.
각 문서는 `문제 / 원인 / 해결 / 결과` 형식으로 짧게 정리했습니다.

---

## 인증 / WebSocket

|  | 케이스 | 한 줄 요약 |
| -- | --- | --- |
| 1 | [만료된 Access Token으로 로그아웃 시 500 에러](troubles/auth-websocket-issues.md) | `ExpiredJwtException` 미처리로 로그아웃 전체 실패 → 만료 토큰도 Refresh Token 삭제로 정상 처리 |
| 2 | [throwUnauthorized() 호출 후 NPE 경고](troubles/auth-websocket-issues.md) | `void` 반환 타입 메서드가 항상 throw한다는 것을 컴파일러가 알 수 없어 경고 → `RuntimeException` 반환으로 해결 |

## 채팅 / 동시성

|  | 케이스 | 한 줄 요약 |
| -- | --- | --- |
| 1 | [ChatRoom 마지막 메시지 Lost Update](troubles/chat-concurrent-issues.md) | JPA dirty checking은 DB 현재 값 확인 없이 덮어씀 → `WHERE` 조건부 JPQL UPDATE로 해결 |
| 2 | [동시 채팅방 생성 시 중복 생성 문제](troubles/chat-concurrent-issues.md) | check-then-act 사이 공백 → DB UNIQUE 제약 + `REQUIRES_NEW` 분리로 방지 |

## 채팅 / 쿼리 최적화

|  | 케이스 | 한 줄 요약 |
| -- | --- | --- |
| 1 | [채팅방 목록 조회 N+1 문제](troubles/chat-query-optimization.md) | `countUnread` 개별 호출을 서브쿼리로 통합 → 채팅방 20개 기준 22번 → 3번 |
| 2 | [채팅방 목록 메모리 정렬/페이징](troubles/chat-query-optimization.md) | 전체 조회 후 Java에서 정렬/페이징 → DB `ORDER BY` + `LIMIT`으로 이전 |

---

> 반복해서 배운 것: 프레임워크가 내부에서 어떻게 동작하는지 모르면 동시성과 쿼리 두 곳에서 동시에 당한다. JPA dirty checking은 DB의 현재 값을 보지 않고 덮어쓰고, `UNIQUE` 충돌을 catch해 fallback 처리하려면 트랜잭션을 분리해야 한다. 정렬과 페이징은 코드로 해결할 문제가 아니라 처음부터 DB에 맡겨야 할 일이다.
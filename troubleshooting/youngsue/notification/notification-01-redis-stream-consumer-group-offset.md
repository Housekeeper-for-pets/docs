# Redis Streams Consumer Group 메시지 미소비

> 카테고리: 알림/Redis Streams · 2026-06 · 도메인: Notification · SSE

## 문제

시터가 공고에 제안을 등록하면 `PROPOSAL_ARRIVED` 알림이 보호자에게 실시간으로 도착해야 했다.  
하지만 API 응답은 성공했고 Redis Stream에도 메시지가 쌓였는데, SSE 구독 터미널에는 알림이 오지 않았다.

확인 당시 Redis에는 메시지가 존재했다.

```bash
XREVRANGE notification-stream + - COUNT 3
```

결과에는 아래와 같은 메시지가 있었다.

```text
type
PROPOSAL_ARRIVED
receiverId
2
referenceType
PROPOSAL
```

그런데 DB `notification` 테이블에는 저장되지 않았고, SSE도 발송되지 않았다.  
즉 Producer는 성공했지만 Consumer가 메시지를 처리하지 못한 상태였다.

## 원인

Redis Streams는 단순 Pub/Sub과 다르게 **Stream + Consumer Group + Consumer + ACK** 흐름을 가진다.

```text
Producer XADD
→ Stream에 메시지 저장
→ Consumer Group에서 메시지 읽기
→ DB 저장/SSE 처리
→ XACK
```

문제는 Consumer Group 생성 위치와 Consumer가 읽는 offset이 정확히 맞지 않으면 메시지가 Stream에 있어도 애플리케이션 Consumer가 읽지 못할 수 있다는 점이었다.

특히 아래 내용을 구분해야 했다.

| 개념 | 의미 |
| --- | --- |
| `0-0` | Stream의 처음부터 읽을 수 있게 Consumer Group 생성 |
| `>` | Consumer Group에서 아직 어떤 Consumer에게도 전달되지 않은 새 메시지 읽기 |
| `lastConsumed()` | Spring Stream listener가 Consumer Group 기준 다음 메시지를 읽도록 설정 |
| `XACK` | 처리 완료한 메시지를 Pending 상태에서 제거 |

중간에 `XREADGROUP`을 직접 실행해보니 메시지가 읽혔다. 이 말은 Stream 자체가 깨진 것이 아니라, 애플리케이션 Consumer 설정/Consumer Group 흐름 문제라는 뜻이었다.

## 해결

Consumer Group 생성과 Stream listener 설정을 명확히 분리했다.

```java
connection.streamCommands()
        .xGroupCreate(
                streamKey,
                GROUP_NAME,
                ReadOffset.from("0-0"),
                true
        );
```

그리고 listener는 Consumer Group이 관리하는 offset을 기준으로 읽도록 설정했다.

```java
StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed())
```

처리 흐름은 다음처럼 정리했다.

```text
RedisStreamNotificationBroker
→ notification-stream XADD
→ NotificationStreamConsumer
→ NotificationEvent 변환
→ NotificationService.notify()
→ DB 저장
→ XACK
```

이미 Consumer Group이 존재하는 경우에는 `BUSYGROUP` 예외를 정상 케이스로 보고 넘어가도록 처리했다.

## 결과

알림 발행 후 Consumer가 메시지를 정상 처리했고, DB 저장과 SSE 발송 흐름이 이어졌다.

확인 포인트는 아래와 같다.

```bash
XLEN notification-stream
XRANGE notification-stream - + COUNT 5
XINFO GROUPS notification-stream
```

정상 처리되면 Consumer Group의 pending이 증가한 채 남아있지 않고, 알림 테이블에도 데이터가 저장된다.

## 배운 점

Redis Streams는 Pub/Sub처럼 "발행하면 누군가 바로 받겠지"가 아니다.  
메시지가 Stream에 있다는 것과 Consumer가 처리했다는 것은 다르다.

Streams를 볼 때는 항상 아래 순서로 확인해야 한다.

```text
1. XADD 되었는가?
2. Consumer Group이 존재하는가?
3. Consumer가 읽고 있는가?
4. Pending에 쌓여 있는가?
5. XACK 되었는가?
6. DB 저장/SSE 전송까지 이어졌는가?
```

면접 포인트로는 Redis Pub/Sub과 Redis Streams의 차이를 설명할 수 있어야 한다. Pub/Sub은 구독자가 없으면 메시지가 사라지지만, Streams는 메시지가 저장되고 Consumer Group이 처리 상태를 관리한다.

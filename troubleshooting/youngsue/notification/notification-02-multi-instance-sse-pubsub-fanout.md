# 멀티 인스턴스 SSE 알림 누락

> 카테고리: 알림/SSE · 2026-06 · 도메인: Notification · Redis Pub/Sub

## 문제

운영 구조를 단일 EC2 내부 멀티 인스턴스로 바꾸면서 Spring Boot 앱이 2개가 됐다.

```text
EC2
├── forpets-nginx
├── forpets-blue
└── forpets-green
```

이때 알림 DB 저장은 Redis Streams Consumer Group으로 한 번만 처리되지만, 실시간 SSE 전송은 누락될 수 있는 구조였다.

예를 들어 사용자가 `forpets-green`에 SSE 연결을 맺고 있는데, Redis Streams 메시지를 `forpets-blue` Consumer가 처리하면 문제가 생긴다.

```text
사용자 SSE 연결: forpets-green 메모리
알림 Consumer 처리: forpets-blue
forpets-blue에서 SSE 전송 시도
→ forpets-blue에는 해당 사용자의 emitter가 없음
→ 실시간 알림 누락
```

## 원인

`SseEmitter`는 Redis나 DB에 저장되는 값이 아니라 **각 JVM 메모리 안에 저장되는 연결 객체**다.

```java
private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();
```

멀티 인스턴스에서는 이 Map이 인스턴스마다 따로 존재한다.

| 인스턴스 | 메모리 안의 SSE 연결 |
| --- | --- |
| `forpets-blue` | blue로 접속한 사용자만 보유 |
| `forpets-green` | green으로 접속한 사용자만 보유 |

Redis Streams Consumer Group은 메시지를 **하나의 Consumer만 처리**하게 만든다. 이 특성은 DB 중복 저장을 막는 데는 좋지만, SSE emitter 위치와 Consumer 처리 인스턴스가 다를 수 있다는 문제가 있었다.

## 해결

알림 처리를 두 단계로 분리했다.

```text
1단계: Redis Streams/Kafka Consumer Group
→ 알림 DB 저장은 한 번만 처리

2단계: Redis Pub/Sub
→ 저장된 알림을 모든 앱 인스턴스에 broadcast
→ 각 인스턴스가 자기 메모리에 emitter가 있으면 전송
```

구조는 아래와 같다.

```text
NotificationMessageBroker
→ Redis Streams or Kafka
→ Consumer 1개가 NotificationService.notify()
→ DB 저장
→ Redis Pub/Sub publish
→ forpets-blue subscriber 수신
→ forpets-green subscriber 수신
→ emitter가 있는 인스턴스만 SSE 전송 성공
```

추가한 구성은 다음과 같다.

| 구성 | 역할 |
| --- | --- |
| `NotificationRealtimeMessage` | DB 저장 후 실시간 전송에 필요한 데이터 |
| `NotificationRedisPublisher` | Redis Pub/Sub 채널로 실시간 알림 broadcast |
| `NotificationRedisSubscriber` | 모든 앱 인스턴스에서 Pub/Sub 메시지 수신 |
| `NotificationService.sendRealtime()` | 각 인스턴스의 SSE emitter에 전송 시도 |
| `ChatPubSubConfig` listener 추가 | 기존 Redis listener container에 알림 subscriber 등록 |

## 결과

DB 저장은 Consumer Group 덕분에 한 번만 일어나고, 실시간 SSE 전송은 Pub/Sub 덕분에 모든 인스턴스에서 시도된다.

```text
DB 저장 중복 방지: Redis Streams Consumer Group
실시간 전송 누락 방지: Redis Pub/Sub fan-out
```

운영에서 `forpets-blue`, `forpets-green` 두 앱이 떠 있는 상태에서도 SSE 알림이 정상 동작하는 구조가 됐다.

## 배운 점

멀티 인스턴스에서 가장 먼저 의심해야 하는 것은 "메모리에 저장된 상태"다.  
SSE emitter, WebSocket session, 로컬 캐시, 스케줄러 상태는 모두 JVM마다 따로 존재한다.

따라서 멀티 인스턴스 실시간 알림은 아래처럼 역할을 나눠야 안정적이다.

| 요구사항 | 적합한 도구 |
| --- | --- |
| 알림을 유실 없이 비동기로 처리 | Redis Streams 또는 Kafka |
| 여러 인스턴스의 SSE 연결에 fan-out | Redis Pub/Sub |
| 알림 이력 조회 | DB |

면접 포인트로는 "Consumer Group은 중복 처리를 막지만, SSE fan-out을 보장하지 않는다"는 점을 설명할 수 있어야 한다.

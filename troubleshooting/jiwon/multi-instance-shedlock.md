# 멀티 인스턴스 스케줄러 중복 실행 방지 ShedLock
작성자: 권지원

### 멀티 인스턴스 환경에서 스케줄러 중복 실행 방지를 위한 Lock 도입 과정 

---

## 1. 배경

ForPets 백엔드는 수평 확장(scale-out)을 전제로 설계되어 있다. 트래픽이 늘어나면 동일한 애플리케이션을 N대 복제해서 로드밸런서 뒤에 배치한다.

문제는 `@Scheduled` 어노테이션이 JVM 단위로 동작한다는 점이다. 인스턴스가 3대라면 매 주기마다 스케줄러가 3번 실행된다.

현재 ForPets 에는 4개의 스케줄러가 존재한다:

| 스케줄러 | 주기 | 주요 작업 | 외부 I/O |
| --- | --- | --- | --- |
| `ReservationExpireScheduler` | 1분 | PENDING 예약 만료 + 환불 + 상태 복원 | **PortOne 환불 API** |
| `PostExpireScheduler` | 10분 | OPEN 공고 만료 + Proposal cascade 만료 | DB only |
| `CareRequestExpireScheduler` | 10분 | PENDING/ACCEPTED 돌봄 요청 만료 | DB only |
| `UnavoidableCancelAutoApproveScheduler` | 매일 00:00 | UNAVOIDABLE 취소 자동 승인 + 알림 발송 | **Kafka** |

---

## 2. 멀티 인스턴스에서 예상되는 문제

실제 장애를 재현하기 전에 설계 단계에서 리스크를 먼저 분석했다. 아래는 인스턴스 3대(A, B, C)가 동시에 스케줄러를 실행할 때 발생 가능한 문제들이다.

### 2.1 PortOne 환불 API 중복 호출

`ReservationExpireScheduler` 는 만료 대상 예약에 대해 PortOne 환불 API 를 호출한다.

```
T=0    A, B, C 가 동시에 만료 대상 예약 조회
       → 동일한 reservation_id 목록을 각자 조회
T=0.1  A, B, C 가 동시에 PortOne 환불 API 호출
       → 동일한 건에 대해 환불 요청 3회 발생
```

PortOne 은 멱등성 키(`paymentId`)가 있어 실제 이중 환불은 막히지만, **의도치 않은 API 호출이 발생했다는 사실 자체가 시스템 이상**이다. 환불 API 스펙이 바뀌거나, 멱등성 키 없는 다른 외부 API 로 교체될 경우 실제 이중 환불로 이어질 수 있다.

### 2.2 Kafka 알림 중복 발행

`UnavoidableCancelAutoApproveScheduler` 는 cron 으로 정확히 00:00 에 트리거된다. N대가 거의 동시에 같은 대상을 조회하고 Kafka 에 알림 이벤트를 발행하면, 사용자는 동일한 알림을 N번 받는다.

엔티티 락(`lock:reservation:{id}`)이 있어 실제 상태 변경은 1회만 일어나지만, **락 획득 이전에 이미 알림 발행 코드가 실행되는 race window 가 존재**한다. 이 경우 상태는 정합성을 유지하더라도 알림은 중복 발송된다.

### 2.3 DB 부하 N배 증가

4개 스케줄러 모두 실행 시작 시점에 후보 조회 쿼리를 날린다. 인스턴스 3대 기준으로 매 주기마다 동일한 쿼리가 3번 실행된다. 현재 스케일에서는 큰 문제가 아니지만, 인스턴스를 늘릴수록 선형적으로 증가하는 불필요한 부하다.

### 2.4 로그 가독성 저하

운영 중 로그에서 특정 스케줄러의 실행 흔적을 추적할 때, 3대가 동시에 동일한 로그를 찍으면 타임스탬프가 겹쳐서 한 번의 실행인지 세 번의 실행인지 구분이 안 된다. 장애 분석 시 노이즈로 작용한다.

---

## 3. 해결 방향 탐색

### 검토한 방법 세 가지

### 방법 1: 커스텀 Redis SETNX 락

기존에 비즈니스 로직용 분산 락(`lock:reservation:{id}` 등)과 동일한 방식으로, 스케줄러 전용 키(`scheduler:lock:ReservationExpireScheduler`)를 추가하는 방식.

**문제점**: 직접 구현 시 두 가지 케이스를 반드시 추가로 처리해야 한다.

- **크래시 복구**: 인스턴스 A 가 락을 획득한 상태에서 SIGKILL 되면 락을 해제하지 못한다. TTL 만료까지 다른 인스턴스가 락을 못 잡는다. TTL 을 짧게 설정하면 정상 처리 중에 락이 끊겨 중복 실행이 발생하고, 길게 설정하면 크래시 후 복구가 늦어진다. **이 트레이드오프를 직접 조율하는 로직이 필요**하다.
- **직후 재실행 방지**: 처리가 매우 빠르게 끝났을 때(예: 만료 대상이 없는 경우), 락이 즉시 해제되면 같은 주기 내에 다른 인스턴스가 락을 잡아서 한 번 더 실행할 수 있다. 이를 막으려면 최소 보유 시간(lockAtLeastFor) 개념을 별도로 구현해야 한다.

결국 커스텀 구현은 비즈니스 코드와 관계없는 인프라 로직을 직접 유지보수하게 만든다.

### 방법 2: Prefix 분리 + 로그 필터링

스케줄러 메서드 진입 시 Redis 에서 락 키를 확인하고, 락을 획득한 인스턴스의 로그만 실제로 찍히도록 분기 처리하는 방식. 튜터 제안으로 나왔던 아이디어다.

**문제점**: 이 방법은 로그 노이즈만 줄여줄 뿐, **중복 실행 자체를 막지 않는다**. 모든 인스턴스가 후보 조회 쿼리를 실행하고, 외부 API 도 호출한다. 로그가 깔끔해 보여도 실제로는 N배 실행이 일어나고 있는 것이다.

### 방법 3: ShedLock

`@SchedulerLock` 어노테이션 하나로 스케줄러 메서드 진입 자체를 단일 인스턴스로 제한하는 라이브러리.

**방법 1 과의 차이**: `lockAtMostFor` (최대 보유 시간, 크래시 복구 자동화)와 `lockAtLeastFor` (최소 보유 시간, 직후 재실행 방지)가 이미 구현되어 있다. 클럭 드리프트 대응, 다양한 Redis 클라이언트 호환성 등 엣지케이스가 수년간 검증된 상태로 제공된다.

**방법 2 와의 차이**: 로그 정리가 아니라 실행 자체를 차단한다. PortOne 환불 API 중복 호출, Kafka 이벤트 중복 발행 등의 근본 원인을 해결한다.

---

## 4. ShedLock 적용

### 4.1 의존성 추가

```groovy
// build.gradle
implementation 'net.javacrumbs.shedlock:shedlock-spring:5.16.0'
implementation 'net.javacrumbs.shedlock:shedlock-provider-redis-spring:5.16.0'
```

### 4.2 설정 Bean

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT30S")
public class SchedulerLockConfig {

    @Bean
    public LockProvider lockProvider(RedisConnectionFactory connectionFactory) {
        return new RedisLockProvider(connectionFactory, "for-pets");
    }
}
```

`"for-pets"` 는 Redis 키 prefix 에 포함되어 기존 비즈니스 락 키(`lock:reservation:{id}`)와 네임스페이스가 분리된다.

### 4.3 스케줄러 어노테이션 추가

```java
@Scheduled(fixedDelay = 60_000)
@SchedulerLock(
    name = "ReservationExpireScheduler",
    lockAtMostFor = "PT50S",
    lockAtLeastFor = "PT15S"
)
public void expire() { ... }
```

기존 스케줄러 비즈니스 로직은 변경 없이 어노테이션만 추가.

### 4.4 락 설정값 결정 근거

| 스케줄러 | 주기 | lockAtMostFor | lockAtLeastFor | 이유 |
| --- | --- | --- | --- | --- |
| `ReservationExpireScheduler` | 1분 | PT50S | PT15S | 외부 I/O 포함, 50초 내 처리 완료 가정. 주기(60초) 직전 해제 보장 |
| `PostExpireScheduler` | 10분 | PT9M | PT30S | DB only, 여유 충분. 10분 직전까지 보장 |
| `CareRequestExpireScheduler` | 10분 | PT9M | PT30S | 동일 |
| `UnavoidableCancelAutoApproveScheduler` | 24h | PT30M | PT1H | cron 특성상 N대가 00:00 에 동시 트리거. 빠른 처리 후에도 1시간 락 유지 |

설정 원칙:

```
실행 평균 시간 < lockAtMostFor < 실행 주기
```

`lockAtMostFor` 가 주기보다 길면 크래시 복구가 다음 주기를 넘길 수 있고, 실행 시간보다 짧으면 정상 처리 중 락이 만료되어 중복 실행이 일어난다.

---

## 5. 2계층 락 구조

ShedLock 도입으로 락이 두 계층이 됐다.

```
┌──────────────────────────────────────────────────────┐
│  Layer 1: ShedLock (스케줄러 메서드 진입 차단)        │
│  shedlock:for-pets:shedlock:{schedulerName}          │
│                                                       │
│  → 한 주기에 한 인스턴스만 진입                       │
│  → 후보 조회 쿼리, for-loop 자체가 1번만 실행         │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼ (단일 인스턴스 진입 보장 후)
┌──────────────────────────────────────────────────────┐
│  Layer 2: 엔티티 ID 락 (Redisson SETNX)              │
│  lock:reservation:{id}, lock:post:{id}, ...          │
│                                                       │
│  → 사용자 요청(결제 confirm 등)과의 충돌 방지         │
│  → 스케줄러 ↔ 사용자 동시성 보호                      │
└──────────────────────────────────────────────────────┘
```

각 계층의 역할이 명확하게 분리된다:

- **ShedLock**: "이 주기에 스케줄러를 실행할 인스턴스는 하나" → 배치 주기 단일성 보장
- **엔티티 락**: "이 예약을 현재 처리 중인 주체는 하나" → 스케줄러 ↔ 사용자 요청 충돌 방지

ShedLock 만 있으면 스케줄러가 사용자의 결제 요청과 동시에 같은 예약을 건드릴 때 충돌이 발생한다. 엔티티 락만 있으면 N대 모두가 쿼리를 실행하고 락 경합을 만들어 비효율이 생긴다.

---

## 6. 적용 결과 확인

### 6.1 Redis 키 구조

```
# ShedLock 키 (스케줄러 단일성)
shedlock:for-pets:shedlock:ReservationExpireScheduler
shedlock:for-pets:shedlock:PostExpireScheduler
shedlock:for-pets:shedlock:CareRequestExpireScheduler
shedlock:for-pets:shedlock:UnavoidableCancelAutoApproveScheduler

# 엔티티 락 키 (비즈니스 동시성)
lock:reservation:9001
lock:post:42
```

두 용도의 락이 prefix 로 완전히 분리되어 Redis 에서도 목적이 구분된다.

### 6.2 멀티 인스턴스 실행 흐름 (ShedLock 적용 후)

```
T=0      인스턴스 A, B, C 모두 @Scheduled 트리거
T=0.001  세 인스턴스 모두 ShedLock setIfAbsent 시도
T=0.002  A 만 성공 → B, C 는 즉시 메서드 종료
         A 만 후보 조회 + PortOne 환불 API 호출 + 상태 변경
```

결과: 후보 조회 쿼리 1회, PortOne API 호출 1회, Kafka 이벤트 발행 1회.

### 6.3 검증 방법

로컬 2대 실행:

```bash
# 터미널 1
SERVER_PORT=8080 ./gradlew bootRun

# 터미널 2
SERVER_PORT=8081 ./gradlew bootRun
```

기대 동작: 매 주기마다 **한 인스턴스의 로그에만** 스케줄러 실행 결과가 찍힘. 다른 인스턴스는 해당 주기에 로그 없음.

```bash
# 두 로그를 비교 — 타임스탬프가 겹치지 않고 번갈아 찍혀야 함
grep "ReservationExpireScheduler" instance1.log
grep "ReservationExpireScheduler" instance2.log
```

Redis 락 상태 확인:

```bash
# 실행 중 락 키 존재 확인
redis-cli GET 'shedlock:for-pets:shedlock:ReservationExpireScheduler'

# lockAtLeastFor 이후 해제 확인
watch -n 1 "redis-cli KEYS 'shedlock:*'"
```

---

## 7. 핵심 요약

| 구분 | 문제 | 선택한 해결 방법 | 이유 |
| --- | --- | --- | --- |
| 중복 실행 | N대가 동시에 스케줄러 실행 | ShedLock `@SchedulerLock` | 메서드 진입 자체를 단일 인스턴스로 제한 |
| 크래시 복구 | 인스턴스 종료 시 락 stale 상태 | `lockAtMostFor` | 직접 구현 없이 자동 TTL 기반 복구 |
| 직후 재실행 | 빠른 처리 완료 후 다른 인스턴스가 재진입 | `lockAtLeastFor` | 동일 주기 내 재실행 차단 |
| 사용자 충돌 | 스케줄러 실행 중 사용자 요청이 같은 엔티티 접근 | 엔티티 ID 락 (기존 유지) | ShedLock 은 스케줄러끼리만 차단, 사용자 요청은 별도 계층이 담당 |
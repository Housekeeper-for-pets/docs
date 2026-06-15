# Lock과 Transaction의 Race Condition 

작성자: 권지원


### 왜 Lock은 Transaction 바깥에 있어야 하는가에 대한 실험


---

## 1. 문제 정의

분산 Lock을 도입할 때 흔히 발생하는 패턴이 있다.

```java
@Transactional                     // ① 트랜잭션 시작
public void someMethod(Long id) {
    lock.acquire(id);              // ② Lock 획득
    // ... 비즈니스 로직 ...
    lock.release(id);              // ③ Lock 해제
}                                  // ④ 트랜잭션 커밋
```

> ⚠️ **문제:** ③ Lock 해제와 ④ 커밋 사이에 공백이 생긴다.
이 짧은 순간 동안 다른 스레드가 Lock을 획득하여 아직 커밋되지 않은 데이터를 읽을 수 있다.
>

---

## 2. 왜 공백이 생기는가

`@Transactional`은 Spring AOP 프록시를 통해 동작한다. 실제 실행 순서는 다음과 같다.

```java
// Spring이 내부적으로 처리하는 순서

TransactionManager.begin();        // 트랜잭션 시작
try {
    target.someMethod();           // 실제 메서드 실행
    //  └─ lock.acquire()
    //  └─ 비즈니스 로직
    //  └─ lock.release()  ← 여기서 Lock 풀림
    TransactionManager.commit();   // 그 다음에 커밋
} catch (Exception e) {
    TransactionManager.rollback();
}
```

`lock.release()` 시점에는 **아직 커밋이 완료되지 않은 상태**다.
DB에는 변경사항이 반영되지 않았지만 Lock은 이미 반환됐으므로, 다음 스레드가 즉시 진입할 수 있다.

---

## 3. Race Condition 시나리오

예약 만료(Expire) 스케줄러와 결제 확정(PaymentConfirm)이 동시에 실행되는 경우를 예시로 본다.

| 시간 | 스레드 A (Expire) | 스레드 B (PaymentConfirm) |
| --- | --- | --- |
| T1 | Lock 획득 (reservation:1) | Lock 대기 |
| T2 | EXPIRED 상태로 변경 | Lock 대기 |
| T3 | **Lock 해제 ← 여기가 문제** | Lock 획득 성공 |
| T4 | (커밋 전) | DB 조회 → 아직 PENDING으로 읽힘 |
| T5 | 트랜잭션 커밋 → EXPIRED | CONFIRMED 처리 진행 |
| T6 | - | 트랜잭션 커밋 → CONFIRMED |

> ❌ **결과:** 예약 상태가 EXPIRED인데 결제는 CONFIRMED. DB 정합성이 깨진다.
>

---

## 4. 올바른 순서: Lock이 Transaction을 감싸야 한다

Lock의 수명이 Transaction의 수명보다 길어야 한다.

| ❌ 잘못된 순서 (TX 안에 Lock) | ✅ 올바른 순서 (Lock 안에 TX) |
| --- | --- |
| TX 시작 | **Lock 획득** |
| Lock 획득 | TX 시작 |
| 비즈니스 로직 | 비즈니스 로직 |
| **Lock 해제 ← 커밋 전!** | TX 커밋 |
| TX 커밋 | **Lock 해제 ← 커밋 후** |

Lock을 먼저 잡고 그 안에서 트랜잭션을 실행하면, 커밋이 완료되기 전까지 다른 스레드는 Lock을 획득할 수 없다.

---

## 5. 구현 방법

### 방법 1: LockService 분리

Lock을 담당하는 별도 서비스를 만들고, 그 안에서 `@Transactional` 메서드를 호출한다.

```java
// LockService: Lock 획득 → TX 메서드 호출 → Lock 해제
@Service
public class ReservationLockService {

    public <T> T executeWithReservationLock(Long reservationId, Supplier<T> task) {
        String lockKey = "lock:reservation:" + reservationId;
        String lockValue = UUID.randomUUID().toString();

        Boolean acquired = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10));

        if (!Boolean.TRUE.equals(acquired)) {
            throw new ReservationException(RESERVATION_LOCK_FAILED);
        }

        try {
            return task.get();   // @Transactional 메서드가 여기서 실행 & 커밋
        } finally {
            releaseLock(lockKey, lockValue);  // 커밋 완료 후 해제
        }
    }
}

// 호출부
return reservationLockService.executeWithReservationLock(
    reservationId, () -> {
        validateNoConfirmedConflict(reservation);
        reservation.confirm();
        return toResponseDto(reservation);
    }
);
```

**Supplier<T>를 쓰는 이유:**
코드 블록을 즉시 실행하지 않고 `.get()` 호출 시점까지 실행을 미룬다.
Lock 획득 이후에만 실행되도록 흐름을 제어하기 위함이다.

---

### 방법 2: AOP + @DistributedLock 어노테이션

Lock이 필요한 메서드가 많을 때 보일러플레이트를 줄이는 방법이다.
`@Order`로 Aspect 실행 순서를 제어한다.

```java
// 어노테이션 정의
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();
}

// AOP Aspect: @Transactional보다 먼저 실행되도록 HIGHEST_PRECEDENCE
@Aspect
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RedisLockAspect {

    @Around("@annotation(com.example.DistributedLock)")
    public Object lock(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        DistributedLock annotation = signature.getMethod()
                                             .getAnnotation(DistributedLock.class);

        String lockKey = "lock:" + parseKey(annotation.key(), joinPoint);
        String lockValue = UUID.randomUUID().toString();

        Boolean acquired = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10));

        if (!Boolean.TRUE.equals(acquired)) {
            throw new ReservationException(RESERVATION_LOCK_FAILED);
        }

        try {
            return joinPoint.proceed();  // TX 시작 → 비즈니스 로직 → TX 커밋
        } finally {
            releaseLock(lockKey, lockValue);  // 커밋 후 Lock 해제
        }
    }
}

// 사용
@DistributedLock(key = "'reservation:' + #reservationId")
@Transactional
public ReservationResponseDto complete(Long memberId, Long reservationId) { ... }
```

> ⚠️ **주의:** self-invocation(같은 클래스 내부 호출)은 프록시를 경유하지 않아 AOP가 적용되지 않는다. 반드시 외부에서 프록시를 통해 호출해야 한다.
>

---

### 두 방법 비교

| 구분 | LockService 분리 | AOP 어노테이션 |
| --- | --- | --- |
| 명시성 | 호출 흐름이 코드에서 직접 보임 | 어노테이션 선언만으로 의도 파악 가능 |
| 보일러플레이트 | 메서드마다 래퍼 메서드 필요 | 어노테이션 한 줄 |
| self-invocation | 함정 없음 | 프록시 경유 필수 (함정 있음) |
| Key 처리 | 호출부에서 직접 넘김 | SpEL로 파라미터에서 추출 |
| 적합한 상황 | Lock 대상 메서드가 적을 때 / merchantUid → reservationId 조회가 필요할 때 | Lock 대상 메서드가 많을 때 / reservationId가 파라미터에 명확히 있을 때 |

두 방법은 혼용 가능하다. webhook처럼 파라미터에 reservationId가 없어서 DB 조회가 먼저 필요한 경우는 LockService, 나머지는 어노테이션.

---

## 6. @Version으로 마지막 방어선 추가

Lock으로 1차 직렬화를 해도, Lock 획득 전 stale read 이후의 write는 막지 못한다.
`@Version`(낙관적 락)으로 마지막 방어선을 추가한다.

```java
@Entity
public class Reservation {
    @Version
    @Column(nullable = false)
    private Long version;
    // ...
}
```

JPA는 UPDATE 시 자동으로 version을 +1 하고 WHERE 절에 기존 version을 추가한다.
동시에 두 트랜잭션이 같은 row를 수정하면 한쪽은 `OptimisticLockException`으로 실패한다.

> ❓ **@Version만 쓰면 안 되는 이유**
>
>
> version 검사는 DB commit 단계에서 일어난다.
> 두 스레드가 모두 외부 API(PG사 환불 등)를 호출한 뒤에야 한쪽이 실패하므로, 이미 환불이 두 번 실행된 상태가 된다.
> Lock이 있으면 B 스레드는 외부 API 호출 자체를 못 한다.
>

---

## 7. 최종 방어 계층 정리

| 계층 | 방법 | 역할 |
| --- | --- | --- |
| 1차 | Distributed Lock (key 통일) | 같은 reservation에 대한 모든 진입점 직렬화 |
| 2차 | 상태 가드 (isPending() 등) | 이미 다른 상태면 조용히 skip |
| 3차 | @Version (낙관적 락) | Lock이 못 잡은 미세 race도 commit 단계에서 차단 |
| 추가 | 건별 트랜잭션 (스케줄러) | 1건 실패해도 나머지 건 진행 |

---

## 정리

**Lock은 반드시 Transaction을 감싸는 구조여야 한다.**
TX 안에 Lock을 잡으면 Lock 해제와 커밋 사이에 빈 구간이 생겨 다른 스레드가 커밋 전 데이터를 읽을 수 있다.

구현 방법은 상황에 따라 선택한다.
Lock 대상 메서드가 적으면 LockService 분리 방법이 명확하고 안전하다.
많으면 AOP 어노테이션이 유지보수성이 좋지만 self-invocation 함정을 항상 의식해야 한다.
`@Version`으로 마지막 방어선을 추가하면 더 견고해진다.
작성자: 권지원

# Spring AOP 체이닝 시 JoinPointMatch 바인딩 버그

작성자: 권지원

### 두 개의 Advice가 같은 메서드에 붙었을 때 발생하는 IllegalStateException


---

## 1. 문제 상황

`@DistributedLock`(분산락 AOP)과 `@TrackExecutionTime`(모니터링 AOP)을 같은 메서드에 붙였더니 서버 오류가 발생했다.

```java
@DistributedLock(key = "'reservation:' + #reservationId")
@TrackExecutionTime("reservation.complete")
@Transactional
public ReservationResponseDto complete(Long memberId, Long reservationId) {
    // ...
}
```

에러 메시지:

```
java.lang.IllegalStateException: Required to bind 2 arguments, but only bound 1
(JoinPointMatch was NOT bound in invocation)
  at org.springframework.aop.aspectj.AbstractAspectJAdvice.argBinding(...)
  at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(...)
  at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(...)
```

---

## 2. 용어 정리

에러를 이해하려면 먼저 용어부터 짚고 넘어간다.

| 용어 | 의미 |
| --- | --- |
| Advice | `@Around`가 붙은 메서드 그 자체. 원래 메서드 전후에 끼어들어 실행되는 코드. `lock()`, `measure()` 같은 메서드를 말함 |
| Advice Method Parameter | Advice 메서드의 파라미터. `ProceedingJoinPoint joinPoint`, `DistributedLock distributedLock` 등 |
| 바인딩 (Binding) | Spring이 Advice 메서드 파라미터에 값을 자동으로 채워주는 것. `joinPoint`와 `distributedLock`에 Spring이 알아서 값을 넣어주는 과정 |
| 체이닝 (Chaining) | 한 메서드에 Advice가 여러 개 붙으면 줄줄이 호출되는 형태 |
| JoinPointMatch | 어떤 파라미터가 몇 번째 인덱스인지 Spring이 내부적으로 관리하는 매핑 정보 |

**체이닝 실행 순서 예시:**

```
RedisLockAspect.lock()
  └─ joinPoint.proceed() 호출
       └─ ExecutionTimeAspect.measure()
            └─ joinPoint.proceed() 호출
                 └─ 실제 complete() 실행
```

---

## 3. 왜 이 버그가 발생하는가

두 Advice 모두 파라미터 바인딩 형태로 어노테이션 인스턴스를 받고 있었다.

```java
// 분산락 AOP
@Around("@annotation(distributedLock)")
public Object lock(ProceedingJoinPoint joinPoint,
                   DistributedLock distributedLock) throws Throwable { }

// 모니터링 AOP
@Around("@annotation(trackExecutionTime)")
public Object measure(ProceedingJoinPoint joinPoint,
                      TrackExecutionTime trackExecutionTime) throws Throwable { }
```

`@annotation(distributedLock)` 처럼 **소문자 변수명**을 쓰면 Spring에게 두 가지를 동시에 요청하는 것이다.

1. 이 어노테이션이 붙은 메서드를 매칭해줘
2. 어노테이션 인스턴스를 파라미터에 바인딩해줘

Spring AOP가 두 Advice를 체이닝할 때, 두 번째 Advice 진입 시점에 JoinPointMatch(파라미터명 → 인덱스 매핑 정보)가 누락된다. 그래서 2개 바인딩이 필요한데 1개만 바인딩됐다는 오류가 발생한다.

---

## 4. 해결 시도 1: argNames 명시

파라미터 이름을 직접 알려주면 Spring이 자동 추론 과정에서 실수하지 않을 거라고 생각했다.

```java
@Around(value = "@annotation(distributedLock)", argNames = "joinPoint,distributedLock")
public Object lock(ProceedingJoinPoint joinPoint,
                   DistributedLock distributedLock) throws Throwable { }

@Around(value = "@annotation(trackExecutionTime)", argNames = "joinPoint,trackExecutionTime")
public Object measure(ProceedingJoinPoint joinPoint,
                      TrackExecutionTime trackExecutionTime) throws Throwable { }
```

**결과: 동일한 오류 발생.**

`argNames`는 "파라미터 이름을 직접 알려주는 것"이지, 체이닝 중 JoinPointMatch가 누락되는 근본 원인을 해결하지 못한다.

---

## 5. 해결 시도 2: Reflection으로 직접 꺼내기 (해결)

바인딩 자체를 포기하고, `joinPoint`만 받은 뒤 Reflection으로 어노테이션 인스턴스를 직접 꺼내는 방식으로 우회했다.

**핵심 차이:**

| 형태 | 의미 | 바인딩 주체 |
| --- | --- | --- |
| `@annotation(distributedLock)` | 매칭 + 어노테이션 인스턴스를 파라미터에 넣어줘 | Spring이 자동으로 채워줌 |
| `@annotation(com.example.DistributedLock)` | 매칭만 해줘 | 내가 Reflection으로 직접 꺼냄 |

소문자 변수명이면 바인딩 요청, 풀 클래스명이면 매칭만 요청이다.

```java
// 기존: 파라미터로 직접 어노테이션 인스턴스를 받음 (바인딩 오류 발생)
@Around(value = "@annotation(distributedLock)", argNames = "joinPoint,distributedLock")
public Object lock(ProceedingJoinPoint joinPoint,
                   DistributedLock distributedLock) throws Throwable {
    String lockKey = LOCK_PREFIX + parseKey(distributedLock.key(), joinPoint);
}
```

```java
// 수정: 풀 클래스명으로 매칭만 하고, Reflection으로 직접 꺼냄
@Around("@annotation(com.forpets.global.aspect.DistributedLock)")
public Object lock(ProceedingJoinPoint joinPoint) throws Throwable {

    MethodSignature signature = (MethodSignature) joinPoint.getSignature();
    DistributedLock distributedLock = signature.getMethod()
                                               .getAnnotation(DistributedLock.class);

    String lockKey = LOCK_PREFIX + parseKey(distributedLock.key(), joinPoint);
    // 이하 동일
}
```

`JoinPointMatch` 바인딩 자체가 필요 없어지기 때문에 체이닝 중 누락 문제가 발생하지 않는다.

---

## 6. Reflection이란

> Runtime에 코드가 자기 자신(class, method, annotation)을 들여다보는 기능
>

`signature.getMethod().getAnnotation()` 처럼 실행 중에 메서드의 어노테이션 정보를 직접 조회하는 것을 말한다.

기존 바인딩 방식은 컴파일 타임 정보를 기반으로 Spring이 값을 채워주는 방식이었다면, Reflection은 런타임에 내가 직접 꺼내는 방식이다.

---

## 7. 정리

```
문제: Advice 2개 체이닝 → 두 번째 진입 시 JoinPointMatch 누락 → 바인딩 실패
        ↓
시도 1: argNames 명시 → 근본 원인이 아니라서 실패
        ↓
해결: 풀 클래스명으로 매칭만 요청 + Reflection으로 직접 꺼내기
```

**핵심 교훈:**

Spring AOP가 알아서 채워주는 바인딩에 의존하면, 여러 Advice가 체이닝될 때 예상치 못한 오류가 생길 수 있다.
Advice가 2개 이상 붙는 상황이라면 처음부터 Reflection으로 직접 꺼내는 방식을 쓰는 게 안전하다.
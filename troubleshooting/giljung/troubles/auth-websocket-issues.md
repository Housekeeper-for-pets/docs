# 인증/WebSocket 트러블슈팅

---

## 1. 만료된 Access Token으로 로그아웃 시 500 에러

### 문제 상황

Access Token이 만료된 상태에서 로그아웃을 요청하면 500 에러가 반환됐다. 사용자 입장에서는 로그아웃을 시도했는데 서버 오류가 발생하는 상황이었다.

### 원인

`logout()` 내부에서 `getMemberId(accessToken)`을 호출하는데, 이 메서드는 내부에서 `parseClaims()`를 실행한다. 만료된 토큰이면 `ExpiredJwtException`이 발생하고, 이를 잡는 코드가 없어 `GlobalExceptionHandler`의 `handleException()`까지 올라가 500으로 처리됐다.

```java
// 변경 전
@Transactional
public void logout(HttpServletRequest request) {
    String accessToken = bearerTokenResolver.resolve(request);
    if (accessToken == null) {
        throw new BusinessException(CommonErrorCode.UNAUTHORIZED);
    }

    Long memberId = jwtTokenProvider.getMemberId(accessToken);     // ← 만료 토큰이면 ExpiredJwtException
    long remainingTime = jwtTokenProvider.getRemainingTime(accessToken);
    tokenRedisService.addToBlacklist(accessToken, remainingTime);
    tokenRedisService.deleteRefreshToken(memberId);
}
```

### 해결

`ExpiredJwtException`을 catch해 만료 토큰도 정상적으로 로그아웃 처리되도록 변경했다.

만료된 Access Token은 어차피 인증이 거부되므로 블랙리스트 등록은 의미가 없다. 대신 `ExpiredJwtException`의 클레임에서 `memberId`를 꺼내 Refresh Token만 삭제한다.

```java
// 변경 후
@Transactional
public void logout(HttpServletRequest request) {
    String accessToken = bearerTokenResolver.resolve(request);
    if (accessToken == null) {
        throw new BusinessException(CommonErrorCode.UNAUTHORIZED);
    }

    try {
        Long memberId = jwtTokenProvider.getMemberId(accessToken);
        long remainingTime = jwtTokenProvider.getRemainingTime(accessToken);
        tokenRedisService.addToBlacklist(accessToken, remainingTime);
        tokenRedisService.deleteRefreshToken(memberId);
    } catch (ExpiredJwtException e) {
        // 만료 토큰은 블랙리스트 등록 불필요, Refresh Token만 삭제
        Long memberId = Long.parseLong(e.getClaims().getSubject());
        tokenRedisService.deleteRefreshToken(memberId);
    }
}
```

`ExpiredJwtException`에서 클레임 추출이 가능한 이유는 jjwt의 파싱 순서 때문이다. jjwt는 서명 검증을 먼저 수행한 뒤 만료 여부를 확인한다. 만료 시점에는 서명 검증이 이미 통과했으므로 파싱된 클레임을 예외 객체에 담아 던진다. 따라서 `ExpiredJwtException`에서는 클레임을 안전하게 신뢰하고 꺼낼 수 있다. 반면 서명 자체가 유효하지 않은 경우(`JwtException`)에는 클레임 파싱이 완료되지 않으므로 클레임을 신뢰할 수 없다.

### 결과

만료된 토큰으로 로그아웃을 요청해도 500 에러 없이 정상적으로 Refresh Token이 삭제되고 로그아웃이 완료된다.

---

## 2. StompChannelInterceptor — throwUnauthorized() 호출 후 NPE 경고

### 문제 상황

`handleConnect()`, `validateSession()`, `getMemberIdFromSession()` 내부에서 null 체크 후 `throwUnauthorized()`를 호출했는데, 이후 코드에서 IntelliJ가 NPE 경고를 띄웠다.

```java
if (authorizationHeader == null || !authorizationHeader.startsWith("Bearer ")) {
    throwUnauthorized(message);  // 실제로는 항상 throw하지만 컴파일러는 모름
}

String token = authorizationHeader.substring(7);  // ← NPE 경고
```

### 원인

`throwUnauthorized()`의 반환 타입이 `void`였다. 메서드 내부에서 항상 `throw new MessageDeliveryException(...)`을 실행하지만, 컴파일러는 `void` 반환 타입 메서드가 항상 예외를 던진다는 것을 알 수 없다.

그래서 컴파일러 입장에서는 `throwUnauthorized()` 호출 이후에도 `authorizationHeader`가 null일 수 있다고 판단하고, 다음 줄의 `substring(7)` 호출에 NPE 경고를 띄웠다.

같은 패턴이 `handleConnect()` 2곳, `validateSession()` 3곳, `getMemberIdFromSession()` 1곳, 총 6곳에 반복됐다.

### 해결

`throwUnauthorized()`의 반환 타입을 `RuntimeException`으로 변경하고, 호출부에서 `throw throwUnauthorized(message)`로 사용하도록 수정했다.

```java
// 변경 전
private void throwUnauthorized(Message<?> message) {
    throw new MessageDeliveryException(
            message, ChatErrorCode.CHAT_WEBSOCKET_UNAUTHORIZED.getMessage()
    );
}

// 호출부 예시
if (authorizationHeader == null || ...) {
throwUnauthorized(message);  // throw 없음
}

// 변경 후
private RuntimeException throwUnauthorized(Message<?> message) {
    throw new MessageDeliveryException(
            message, ChatErrorCode.CHAT_WEBSOCKET_UNAUTHORIZED.getMessage()
    );
}

// 호출부 예시 (handleConnect 2곳, validateSession 3곳, getMemberIdFromSession 1곳 동일 적용)
if (authorizationHeader == null || ...) {
        throw throwUnauthorized(message);  // throw 추가
}
```

반환 타입이 `RuntimeException`이면 컴파일러가 `throw throwUnauthorized(message)` 구문을 보고 이 지점에서 예외가 던져진다는 것을 인식한다. 이후 코드가 실행되지 않는다는 것을 알게 되어 NPE 경고가 사라진다.

### 결과

정적 분석 경고가 제거됐고, 실수로 `throw` 키워드를 빠뜨리면 컴파일 에러가 발생해 누락을 방지할 수 있는 구조가 됐다.
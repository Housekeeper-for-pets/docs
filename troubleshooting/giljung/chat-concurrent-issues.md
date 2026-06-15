# 채팅 동시성 트러블슈팅

---

## 1. ChatRoom 마지막 메시지 Lost Update

### 문제 상황

두 사용자가 동시에 메시지를 보낼 때 채팅방의 `lastMessageId`가 잘못된 값으로 저장되는 문제가 발생했다.

A가 `message#100`을, B가 `message#101`을 거의 동시에 전송하면 채팅방 목록에 `#101`이 아닌 `#100`이 마지막 메시지로 표시됐다.

### 원인

기존 코드는 메시지 저장 후 JPA dirty checking으로 `ChatRoom`의 `lastMessageId`를 갱신했다.

```java
chatRoom.updateLastMessage(message.getId(), createdAt);
```

JPA dirty checking은 트랜잭션 커밋 시점에 Java 객체의 변경값을 DB에 그대로 덮어쓴다. 이때 **현재 DB에 어떤 값이 있는지 확인하지 않는다.**

두 트랜잭션이 동시에 실행되면 다음과 같은 순서로 문제가 발생한다.

```
스레드A: chatRoom 조회 → lastMessageId = 99
스레드B: chatRoom 조회 → lastMessageId = 99

스레드A: message#100 저장, 메모리에서 lastMessageId = 100으로 변경
스레드B: message#101 저장, 메모리에서 lastMessageId = 101로 변경

스레드B 커밋: UPDATE SET last_message_id = 101 → DB: 101
스레드A 커밋: UPDATE SET last_message_id = 100 → DB: 100  ← 덮어씌워짐
```

스레드A는 자신이 조회한 시점(99) 이후에 B가 101을 저장했다는 사실을 모른 채 100으로 덮어쓴다.

### 해결

JPA dirty checking 대신 `WHERE` 조건이 있는 JPQL UPDATE를 사용했다.

```java
// ChatRoomRepository.java
@Modifying(clearAutomatically = true)
@Query("""
            UPDATE ChatRoom c
            SET c.lastMessageId = :messageId,
                c.lastMessageAt = :messageAt
            WHERE c.id = :chatRoomId
              AND (c.lastMessageId IS NULL OR c.lastMessageId < :messageId)
        """)
int updateLastMessageIfNewer(
        @Param("chatRoomId") Long chatRoomId,
        @Param("messageId") Long messageId,
        @Param("messageAt") LocalDateTime messageAt
);
```

`AND (c.lastMessageId IS NULL OR c.lastMessageId < :messageId)` 조건이 핵심이다.

DB에 이미 더 큰 ID가 저장되어 있으면 UPDATE가 실행되지 않는다.

```
스레드B 커밋: DB(99) < 101 → UPDATE 실행 → DB: 101
스레드A 커밋: DB(101) < 100 → false → UPDATE 실행 안 됨
```

이 방식이 가능한 이유는 `messageId`가 `AUTO_INCREMENT`이기 때문이다. 나중에 생성된 메시지는 반드시 더 큰 ID를 가지므로 **"더 큰 ID = 더 최신 메시지"** 가 항상 보장된다.

`clearAutomatically = true`를 설정한 이유는 JPQL UPDATE가 DB를 직접 수정하지만 JPA 1차 캐시의 엔티티는 갱신하지 않기 때문이다. 캐시를 비우지 않으면 같은 트랜잭션 안에서 `ChatRoom`을 재조회할 때 캐시의 옛 값을 반환한다. `clearAutomatically = true`로 캐시를 비워 재조회 시 DB에서 최신 값을 읽어오도록 보장한다.

**서비스 코드 변경**

```java
// 변경 전
chatRoom.updateLastMessage(message.getId(), createdAt);

// 변경 후
chatRoomRepository.updateLastMessageIfNewer(chatRoomId, message.getId(), createdAt);
```

---

## 2. 동시 채팅방 생성 시 중복 생성 문제

### 문제 상황

1번 회원과 5번 회원이 거의 동시에 채팅 시작하기 버튼을 누르면 같은 두 회원 사이에 채팅방이 2개 생성될 수 있었다.

### 원인

기존 채팅방 생성 로직은 다음 순서로 동작한다.

```
① roomKey로 기존 채팅방 조회
② 없으면 새 채팅방 생성
```

두 요청이 동시에 들어오면 ①에서 둘 다 "채팅방 없음"을 확인한 뒤 ②를 동시에 실행해 채팅방 2개가 생성된다.

```
요청A: roomKey 조회 → 없음 → 생성 시작
요청B: roomKey 조회 → 없음 → 생성 시작  ← A가 아직 커밋 전이라 없는 것처럼 보임
요청A: 채팅방 저장 완료
요청B: 채팅방 저장 완료  ← 중복 생성
```

### 해결

두 가지를 함께 적용해 해결했다.

**① DB UNIQUE 제약**

```java
@Table(
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_chat_room_room_key", columnNames = "room_key")
    }
)
```

같은 `roomKey`의 채팅방을 DB 레벨에서 1개만 허용한다.

**② `REQUIRES_NEW` + `DataIntegrityViolationException` catch**

DB UNIQUE 제약만으로는 충분하지 않다. 같은 트랜잭션 안에서 UNIQUE 충돌이 발생하면 트랜잭션 자체가 rollback-marked 상태가 되어 `catch` 이후 어떤 DB 작업도 수행할 수 없다. 충돌을 catch하고 이미 생성된 채팅방을 재조회하는 fallback 처리를 하려면 채팅방 생성을 별도 트랜잭션으로 분리해야 한다.

이를 위해 채팅방 생성을 별도 클래스(`ChatRoomCreateHelper`)로 분리하고 `Propagation.REQUIRES_NEW`로 독립 트랜잭션에서 실행했다. UNIQUE 충돌이 발생하면 `DataIntegrityViolationException`을 잡아 에러로 끝내지 않고 기존 채팅방을 재조회해서 반환한다.

```java
// ChatRoomCreateHelper.java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public ChatRoom createChatRoomWithParticipants(Long memberAId, Long memberBId, String roomKey) {
    ChatRoom chatRoom = chatRoomRepository.save(ChatRoom.builder()
            .memberAId(memberAId)
            .memberBId(memberBId)
            .roomKey(roomKey)
            .build());

    chatRoomParticipantRepository.save(ChatRoomParticipant.builder()
            .chatRoomId(chatRoom.getId()).memberId(memberAId).build());

    chatRoomParticipantRepository.save(ChatRoomParticipant.builder()
            .chatRoomId(chatRoom.getId()).memberId(memberBId).build());

    return chatRoom;
}

// ChatRoomService.java
try {
    ChatRoom newChatRoom = chatRoomCreateHelper.createChatRoomWithParticipants(
            memberAId, memberBId, roomKey
    );
    return new ChatRoomCreateResponse(newChatRoom.getId(), ...);

} catch (DataIntegrityViolationException e) {
    // 동시 요청으로 roomKey 충돌 발생 → 이미 생성된 채팅방 재조회
    ChatRoom fallbackRoom = chatRoomRepository.findByRoomKey(roomKey)
            .orElseThrow(() -> new ChatException(ChatErrorCode.CHAT_ROOM_NOT_FOUND));

    rejoinIfLeft(fallbackRoom.getId(), currentMemberId);

    return new ChatRoomCreateResponse(fallbackRoom.getId(), ...);
}
```

단, `REQUIRES_NEW`는 부모 트랜잭션 커넥션을 유지한 채 새 커넥션을 추가로 획득하므로 HikariCP 커넥션을 일시적으로 2개 점유한다. 동시 요청이 많은 환경에서는 커넥션 풀 고갈 위험이 있다. 현재는 단일 EC2 + 소규모 트래픽 환경이므로 문제없다고 판단했으나, 트래픽이 증가하면 커넥션 풀 크기 조정 또는 다른 방식(예: DB 레벨 잠금)을 고려해야 한다.

### 결과

동시 요청이 들어와도 채팅방은 항상 1개만 생성되고, 늦게 들어온 요청은 이미 생성된 채팅방을 반환받는다. 두 요청 모두 정상 응답을 받는다.
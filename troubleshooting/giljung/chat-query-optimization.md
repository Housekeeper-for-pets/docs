# 채팅 쿼리 최적화 트러블슈팅

---

## 1. 채팅방 목록 조회 N+1 문제

### 문제 상황

채팅방 목록 조회 시 채팅방 수만큼 `countUnread()` 쿼리가 반복 실행됐다.

채팅방이 20개면 쿼리가 22번 나갔다.

```
① SELECT * FROM chat_room_participant WHERE member_id = ?   (1번)
② SELECT * FROM chat_room WHERE id IN (...)                 (1번)
③ SELECT COUNT(*) FROM chat_message WHERE chat_room_id = 1  (채팅방 수만큼 반복)
④ SELECT COUNT(*) FROM chat_message WHERE chat_room_id = 2
...
```

### 원인

`toChatRoomListItem()` 내부에서 채팅방마다 `countUnread()`를 개별 호출했다.

```java
for (ChatRoom chatRoom : pagedRooms) {
long unreadCount = chatMessageRepository.countUnread(
        chatRoom.getId(), memberId, ...
        );
        }
```

채팅방이 N개면 쿼리가 N번 나가는 구조다.

### 해결

`countUnread()` 개별 호출을 서브쿼리로 대체해 채팅방 목록 조회 쿼리에 통합했다.

```java
// ChatRoomParticipantRepository.java
@Query("""
            SELECT new com.forpets.domain.chat.dto.ChatRoomSummary(
                r.id, r.memberAId, r.memberBId,
                r.lastMessageId, r.lastMessageAt,
                p.lastReadMessageId, p.visibleFromAt,
                (SELECT COUNT(m) FROM ChatMessage m
                 WHERE m.chatRoomId = r.id
                   AND m.senderId != :memberId
                   AND (p.lastReadMessageId IS NULL OR m.id > p.lastReadMessageId)
                   AND (p.visibleFromAt IS NULL OR m.createdAt > p.visibleFromAt))
            )
            FROM ChatRoomParticipant p
            JOIN ChatRoom r ON r.id = p.chatRoomId
            WHERE p.memberId = :memberId
              AND p.isLeft = false
              AND r.lastMessageAt IS NOT NULL
            ORDER BY r.lastMessageAt DESC, r.id DESC
        """)
List<ChatRoomSummary> findChatRoomSummaries(...);
```

쿼리 결과를 담기 위해 `ChatRoomSummary` Projection DTO를 추가했다.

```java
public record ChatRoomSummary(
        Long chatRoomId, Long memberAId, Long memberBId,
        Long lastMessageId, LocalDateTime lastMessageAt,
        Long lastReadMessageId, LocalDateTime visibleFromAt,
        long unreadCount
) {}
```

서브쿼리 방식은 각 채팅방에 대한 집계 범위가 해당 채팅방으로 한정되어 실행 계획이 단순하고 예측하기 쉽다는 장점이 있다. 대안으로 `LEFT JOIN + GROUP BY` 방식도 고려했으나, 현재 채팅방 수 규모에서 서브쿼리가 충분히 효율적이라 판단했다. 채팅방이 크게 늘어난다면 커버링 인덱스를 활용한 `LEFT JOIN` 방식을 재검토할 수 있다.

### 결과

| | 변경 전 | 변경 후 |
|---|---|---|
| 채팅방 목록 조회 쿼리 수 | 채팅방 수 + 2번 | 3번 (고정) |
| 채팅방 20개 기준 | 22번 | 3번 |

변경 후 3번의 구성:

```
① findChatRoomSummaries — 채팅방 목록 + 안읽음 수 서브쿼리 통합
② 상대방 회원 정보 IN 쿼리 (opponentMap 구성)
③ lastMessageId 기준 마지막 메시지 IN 쿼리 (lastMessageMap 구성)
```

---

## 2. 채팅방 목록 메모리 정렬/페이징

### 문제 상황

채팅방 목록 정렬과 페이징이 DB가 아닌 Java 메모리에서 처리됐다.

참여 채팅방 100개를 전부 DB에서 가져온 뒤 20개만 쓰고 나머지 80개를 버리는 방식이었다.

### 원인

```java
// DB에서 전체 조회
List<ChatRoom> chatRooms = chatRoomRepository.findAllById(chatRoomIds);

// Java에서 정렬
List<ChatRoom> sorted = chatRooms.stream()
        .sorted(Comparator.comparing(ChatRoom::getLastMessageAt).reversed())
        .toList();

// Java에서 페이징
List<ChatRoom> paged = sorted.stream()
        .limit(20)
        .toList();
```

20개를 보여주기 위해 전체를 DB에서 가져오고 있었다. 채팅방이 많은 사용자일수록 불필요한 데이터 전송과 메모리 낭비가 커진다.

### 해결

N+1 해결과 함께 `findChatRoomSummaries()` 쿼리에 `ORDER BY`와 `LIMIT`을 포함시켜 DB에서 처리하도록 변경했다.

```sql
ORDER BY r.lastMessageAt DESC, r.id DESC  -- DB에서 정렬
LIMIT ?                                    -- DB에서 페이징
```

서비스 코드에서 `size + 1`개를 요청해 다음 페이지 존재 여부를 판단한다.

```java
List<ChatRoomSummary> summaries = chatRoomParticipantRepository.findChatRoomSummaries(
        memberId,
        cursorLastMessageAt,
        cursorChatRoomId,
        PageRequest.of(0, effectiveSize + 1)  // +1로 hasNext 판단
);

boolean hasNext = summaries.size() > effectiveSize;
List<ChatRoomSummary> pagedSummaries = summaries.subList(
        0, Math.min(effectiveSize, summaries.size())
);
```

### 결과

채팅방 수에 관계없이 항상 필요한 개수만 DB에서 가져온다. N+1 해결과 합쳐서 전체 쿼리 수가 채팅방 20개 기준 22번에서 3번으로 줄었다.
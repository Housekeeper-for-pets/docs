# 📋 ADR (Architecture Decision Records) — ForPets (포펫츠)

> **프로젝트명:** ForPets (포펫츠)
> **팀명:** 집사조
> **형식:** 결정마다 1개 ADR
> **문서 버전:** v1.0
> **최종 수정일:** 2026-05-12
> **연결 문서:** 01_프로젝트_개요서, 04_기능_명세서, 05_ERD, 06_API_명세서

---

## ADR-001. JWT AccessToken 단독 인증 채택 (Refresh Token 미구현)

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-AUTH-01, FN-AUTH-02 |

### 컨텍스트

회원 인증 방식을 결정해야 한다. 세션 기반과 JWT 기반을 비교하고, JWT 선택 시 Refresh Token 포함 여부를 결정해야 한다. ForPets는 보호자·시터·구조단체 3가지 역할이 있어 역할 기반 인가(Role-Based Access Control)도 함께 설계해야 한다.

### 비교

| 항목 | 세션 기반 (Spring Session + Redis) | JWT AccessToken 단독 | JWT AccessToken + RefreshToken |
|------|-----------------------------------|---------------------|-------------------------------|
| 서버 상태 | Stateful (Redis 필수) | Stateless | Stateless + Redis(RefreshToken) |
| 스케일아웃 | Redis 세션 공유 필요 | 추가 인프라 불필요 | Redis 필요 |
| 구현 복잡도 | 낮음 | 낮음 | 중간 |
| 토큰 즉시 무효화 | 가능 (Redis 삭제) | 어려움 (만료 대기) | 가능 (RefreshToken 삭제) |
| 6주 일정 적합성 | ✅ | ✅ | ❌ (블랙리스트 로직 복잡) |

### 결정

**JWT AccessToken 단독 (만료 1시간), payload에 userId + role 포함. Refresh Token은 v1에서 제외.**

### 이유

1. **일정**: Refresh Token 블랙리스트 관리 구현 복잡도가 6주 범위에서 ROI가 낮음
2. **케어 플로우 특성**: 공고 등록 → 제안 수락 → 케어 확정은 1시간 이내 완료 가능
3. **역할 분리**: JWT payload에 role(GUARDIAN/SITTER/RESCUE)을 포함하여 `@PreAuthorize`로 API별 접근 제어
4. **Stateless**: JWT 기반 Stateless 인증은 서버 증설 시 인증 로직 변경 없음

### 트레이드오프

AccessToken 만료(1시간) 전 강제 무효화 불가. 로그아웃 시 클라이언트 토큰 폐기만 수행.

---

> ### 📌 v1 구현 범위 정리 (팀원 필독)
>
> | 항목 | 설계 | v1 실제 구현 |
> |------|------|------------|
> | AccessToken | HS256, 1시간 | HS256, 1시간 (동일) |
> | payload | userId, role, iat, exp | 동일 |
> | RefreshToken | 7일, Redis 저장 | **미구현** |
> | 로그아웃 | Redis RefreshToken 삭제 | 클라이언트 토큰 폐기만 |
>
> **영향 문서:** FN-AUTH-02(로그인), UC-001, UC-002, API 명세서 §1

---

## ADR-002. Redisson 분산락 채택 — 시터 제안 입찰 & 렌탈 재고 차감

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-PROP-01, FN-RENTAL-03 |

### 컨텍스트

ForPets에는 두 가지 핵심 동시성 이슈가 있다.

1. **시터 제안 동시 입찰**: 하나의 케어 공고에 여러 시터가 동시에 제안하면 `proposal_count` Lost Update 발생
2. **렌탈 재고 동시 차감**: 마지막 1개 남은 용품에 여러 보호자가 동시 신청하면 `stock` 음수 발생

### 분산락 전략 비교

| 전략 | 정합성 | 성능 | 구현 난이도 | 분산 환경 | 실패 전략 |
|------|--------|------|------------|-----------|---------|
| 낙관적 락 (`@Version`) | 재시도 시 보장 | 충돌 빈번 시 저하 | 중간 | 단일 DB 의존 | 재시도 복잡 |
| 비관적 락 (`FOR UPDATE`) | 확실 | 락 경합 시 지연 | 낮음 | 단일 DB 의존 | 대기 발생 |
| Lettuce SETNX | 확실 | DB 접근 전 차단 | 중간 | 가능 | 즉시 실패 |
| **Redisson RLock** ✅ | 확실 | DB 접근 전 차단 | 중간 (AOP 가능) | 가능 | tryLock + waitTime 설정 |

### 결정

**Redisson RLock 채택. 필수 구현 후 @RedisLock AOP 패턴으로 리팩토링(도전).**

### 전략별 실패 전략 설계

| 시나리오 | 락 키 | waitTime | 실패 전략 | 선택 이유 |
|----------|-------|----------|---------|---------|
| 시터 제안 입찰 | `lock:proposal:{postingId}` | 3초 | Fail Fast (즉시 429) | 제안은 빠른 응답이 UX에 중요 |
| 렌탈 재고 차감 | `lock:rental:{itemId}` | 5초 | Retry Backoff 3회 | "받느냐 못 받느냐" — 잠깐 기다려서라도 성공이 유리 |

### AOP 리팩토링 전략 (도전)

```yaml
# application.yml — Feature Flag
lock:
  provider: redisson   # redisson | lettuce
```

```java
// @RedisLock 어노테이션으로 락 로직 분리
@RedisLock(key = "'lock:proposal:' + #postingId", waitTime = 3, leaseTime = 10)
public ProposalResponse submitProposal(Long postingId, ...) { ... }

@RedisLock(key = "'lock:rental:' + #itemId", waitTime = 5, leaseTime = 15)
public RentalResponse requestRental(Long itemId, ...) { ... }
```

### 트레이드오프

Redis 의존성 추가. Redis 장애 시 분산락 불가. 단, ForPets는 이미 캐싱·SSE 세션 등으로 Redis를 사용하므로 추가 인프라 비용 없음. Redis 장애 시 DB UNIQUE 제약(MATCHING.posting_id)이 2차 방어선으로 동작.

---

## ADR-003. Kafka 도입 — 매칭 확정 이벤트 비동기 처리

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-MATCH-01, FN-TEMP-04 |

### 컨텍스트

보호자가 시터 제안을 수락(매칭 확정)하면 동시에 여러 후속 처리가 필요하다.
- 결제 처리
- 시터에게 "매칭 확정" SSE 알림
- 케어 일정 생성

이것을 동기로 처리하면 하나가 느려지면 전체 응답이 지연된다.

### 비교

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| 동기 순차 처리 | Service 내에서 순차 호출 | 구현 단순 | 하나 실패 시 전체 롤백, 응답 지연 |
| Spring Event (`@EventListener`) | 트랜잭션 내/외 이벤트 발행 | Spring 내장, 추가 인프라 없음 | 서버 재시작 시 이벤트 유실, 분산 환경 불가 |
| **Kafka** ✅ | 토픽 발행, 독립 소비자 | 장애 격리, DLQ 재처리, 분산 환경 지원 | 운영 복잡도, 초기 학습 비용 |

### 결정

**Kafka 채택. `matching-confirmed` 토픽으로 이벤트 발행 → 독립 소비자 3개가 병렬 처리.**

### Kafka 파이프라인 설계

```
Producer (MatchingService)
    │
    ▼ topic: matching-confirmed
    │
    ├── Consumer 1: PaymentService     (결제 처리)
    ├── Consumer 2: NotificationService (시터 SSE 알림 발송)
    └── Consumer 3: ScheduleService    (케어 일정 생성)
```

```json
// 이벤트 페이로드
{
  "topic": "matching-confirmed",
  "payload": {
    "matchingId": 1,
    "postingId": 10,
    "guardianId": 5,
    "sitterId": 20,
    "finalPrice": 45000,
    "startDate": "2026-07-01",
    "endDate": "2026-07-04"
  }
}
```

### 장애 처리

- 소비자 처리 실패 시 DLQ(Dead Letter Queue)에 적재 → 수동 재처리
- Producer 발행 실패 시 DB 트랜잭션 롤백 (Kafka 발행 전 MATCHING INSERT 완료)
- 보호자 응답은 Kafka 소비 완료를 기다리지 않음 (비동기 — 매칭 ID만 반환)

### 트레이드오프

Kafka 운영 복잡도. 로컬 개발 환경에 Docker Compose로 Kafka + Zookeeper 추가 필요. 팀원 초기 학습 비용 발생. 그러나 분산 메시지 처리 경험이 핵심 학습 목표 중 하나이므로 채택.

---

## ADR-004. SSE 채택 — 케어 일지·알림 실시간 전송

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-LOG-01, FN-LOG-03 |

### 컨텍스트

시터가 케어 일지를 등록하면 보호자에게 실시간으로 전달해야 한다. 또한 제안 도착·매칭 확정 알림도 실시간으로 전달해야 한다.

### 비교

| 방식 | 통신 방향 | 연결 비용 | 채택 여부 |
|------|---------|---------|---------|
| 폴링 (Polling) | 클라이언트→서버 반복 | 높음 (반복 HTTP) | ❌ |
| Long Polling | 클라이언트→서버 | 중간 | ❌ |
| **SSE** ✅ | 서버→클라이언트 (단방향) | 낮음 (단일 HTTP 연결 유지) | ✅ 알림·일지 전용 |
| WebSocket | 양방향 | 높음 | ✅ 채팅 전용 (도전) |

### 결정

**알림·케어 일지는 SSE (단방향). 1:1 채팅은 WebSocket STOMP (양방향, 도전 기능)로 분리.**

### 다중 서버 환경 대응

```
시터 앱 → POST /api/matchings/{id}/logs
         → DB 저장
         → Redis Pub/Sub channel: care-log:{matchingId} 발행
         → SSE 구독 중인 보호자 앱에 실시간 전송 (서버 여러 대에서도 전달 보장)
```

### SSE 이벤트 타입

| 타입 | 발생 시점 |
|------|---------|
| `CARE_LOG` | 시터가 케어 일지 등록 시 |
| `PROPOSAL_ARRIVED` | 공고에 새 제안 도착 시 |
| `MATCHING_CONFIRMED` | 매칭 확정 시 |
| `RENTAL_OVERDUE` | 렌탈 반납 기한 초과 시 |
| `TEMP_CARE_MATCHED` | 임시보호 매칭 확정 시 |

---

## ADR-005. MATCHING.posting_id UNIQUE — DB 레벨 중복 매칭 방어

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-MATCH-01 |

### 컨텍스트

보호자가 빠르게 두 번 클릭하거나, 네트워크 재시도로 동일 공고에 대한 매칭이 2건 생성될 위험이 있다. 1개의 케어 공고에는 반드시 1개의 매칭만 존재해야 한다.

### 결정

**MATCHING.posting_id에 UNIQUE 제약을 추가. 2차 방어선으로 설계.**

```sql
CREATE TABLE matching (
    matching_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    posting_id  BIGINT UNIQUE NOT NULL,  -- 공고당 1건만 허용
    -- ...
);
```

### 방어선 구조

```
1차 방어선: Redisson 분산락 (소프트웨어 레벨)
2차 방어선: posting_id UNIQUE 제약 (DB 레벨)
         → 락 버그나 Redis 장애 시에도 중복 매칭을 DB 레벨에서 원천 차단
```

### 트레이드오프

UNIQUE 충돌 시 `DataIntegrityViolationException` 발생 → GlobalExceptionHandler에서 409로 변환 필요.

---

## ADR-006. Redis 캐싱 전략 — 조회 성능 최적화

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 |
| **관련 기능** | FN-POST-02, FN-PROFILE-01, FN-REVIEW-02 |

### 컨텍스트

ForPets의 주요 읽기 패턴은 공고 목록 검색, 시터 프로필 조회, 인기 시터 랭킹이다. 이 조회들은 빈번하고 결과가 자주 변경되지 않아 캐싱 효과가 크다.

### 결정

**Redis 캐싱 채택. 캐시 대상별 TTL 설계.**

| 캐시 대상 | 키 패턴 | TTL | 무효화 트리거 |
|-----------|---------|-----|------------|
| 공고 목록 검색 | `cache:postings:{조건해시}` | 60초 | 신규 공고 등록·취소 |
| 시터 프로필 | `cache:sitter:{userId}` | 5분 | 프로필 수정 |
| 시터 평점 랭킹 | `sitter:ratings` (ZSET) | 없음 (ZINCRBY 관리) | 후기 작성 시 갱신 |
| 렌탈 용품 목록 | `cache:rental:items:{조건해시}` | 30초 | 신규 용품 등록 |
| AI 프로필 생성 결과 | `ai:profile:cache:{sitterId}` | 1시간 | 수동 재생성 |

### Redis ZSET 시터 평점 랭킹

```
ZINCRBY sitter:ratings {rating} {sitterId}
→ 후기 작성 시 즉시 반영. TTL 없이 항상 최신 상태 유지.
→ 인기 시터 Top 20 조회: ZREVRANGE sitter:ratings 0 19 WITHSCORES
```

### Graceful Degradation

Redis 연결 실패 시 캐시 미스로 처리 → DB 직접 조회로 폴백. 서비스 장애로 이어지지 않음.

---

## ADR-007. QueryDSL 채택 — 케어 공고 동적 필터 검색

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 (박영수 주도) |
| **관련 기능** | FN-POST-02, FN-RENTAL-02 |

### 컨텍스트

케어 공고 검색 조건이 지역·날짜·견종 크기·예산 범위 등 조합이 수십 가지다. JPQL 문자열 조합은 런타임 오류 위험이 있고 가독성이 낮다.

### 비교

| 방식 | 타입 안전 | 동적 조건 | 가독성 |
|------|---------|---------|------|
| JPQL 문자열 | ❌ 런타임 오류 | 조건마다 문자열 붙임 | 낮음 |
| Criteria API | ✅ | ✅ | 매우 낮음 (장황함) |
| **QueryDSL** ✅ | ✅ 컴파일 타임 | ✅ BooleanBuilder | 높음 |

### 결정

**QueryDSL 채택. `CarePostingRepositoryCustom` 인터페이스로 JPA Repository와 분리.**

```java
// CarePostingRepositoryImpl.java
public Page<CarePosting> searchPostings(PostingSearchCondition condition, Pageable pageable) {
    BooleanBuilder builder = new BooleanBuilder();
    if (StringUtils.hasText(condition.getLocation())) {
        builder.and(carePosting.location.contains(condition.getLocation()));
    }
    if (condition.getPetSize() != null) {
        builder.and(pet.size.eq(condition.getPetSize()));
    }
    if (condition.getMaxBudget() != null) {
        builder.and(carePosting.budget.loe(condition.getMaxBudget()));
    }
    // ... 기타 조건
    return applyPagination(pageable, builder);
}
```

### 트레이드오프

Q클래스 생성으로 빌드 시간 증가. 설정 복잡성 있음. 그러나 동적 검색 쿼리 안정성이 더 중요.

---

## ADR-008. AI 장애 격리 — 핵심 서비스와 AI 기능 완전 분리

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 팀 전원 (이지민 주도) |
| **관련 기능** | FN-PROFILE-03, FN-AI-01, FN-AI-02 |

### 컨텍스트

LLM API는 외부 서비스다. 장애·지연·비용 초과 시 핵심 서비스(케어 공고 등록, 시터 매칭, 렌탈)가 영향을 받으면 안 된다.

### 결정

**AI 기능은 핵심 서비스와 완전 격리. Fallback 2가지 + Circuit Breaker 필수 구현.**

### 장애 격리 전략

| 장애 유형 | 처리 방식 | 핵심 서비스 영향 |
|----------|---------|-------------|
| 타임아웃 (5초 초과) | Fallback ① 캐시된 이전 결과 반환 | 없음 |
| Rate Limit (429) | Circuit Breaker OPEN → Fallback ② 기본 템플릿 반환 | 없음 |
| 5xx 서버 오류 | 최대 2회 재시도 후 Fallback | 없음 |
| 환각 방어 | AI 추천 결과를 DB 교차 검증 → 불일치 시 차단 | 없음 |

### Circuit Breaker 설정 (Resilience4j)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      ai-service:
        slidingWindowSize: 10
        failureRateThreshold: 50       # 50% 실패율 시 OPEN
        waitDurationInOpenState: 30s   # 30초 후 HALF-OPEN
        permittedNumberOfCallsInHalfOpenState: 3
```

### AI 서비스별 격리 원칙

```
AI 기능 장애 → 케어 공고 등록 정상 동작
AI 기능 장애 → 시터 매칭 정상 동작
AI 기능 장애 → 렌탈 신청 정상 동작
AI 기능 장애 → SSE 알림 정상 동작

즉, AI 기능은 "있으면 좋은 것"이지 "없으면 안 되는 것"이 아님
```

---

## ADR-009. 페이징 전략 — Offset vs Cursor

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **관련 API** | 공고 목록, 매칭 목록, 채팅 메시지 |

### 결정

| 대상 | 페이징 방식 | 이유 |
|------|-----------|------|
| 공고 목록·시터 목록 | **Offset** (`page`, `size`) | 총 페이지 수 표시가 필요한 탐색형 UI |
| 내 매칭·렌탈 내역 | **Offset** | 사용자가 직접 페이지 선택하는 패턴 |
| 케어 일지 목록 | **Offset** | 총 건수 표시 필요, 실시간 추가 빈도 낮음 |
| 채팅 메시지 | **Cursor** (`cursor`, `size`) | 실시간 추가 환경 — Offset은 누락/중복 위험 |

### 채팅에 Cursor를 선택한 근거

```
T=0: 메시지 100개 있음. 사용자가 page=1 (51~100번) 조회
T=1: 새 메시지 5개 추가 (101~105번)
T=2: 사용자가 page=2 (1~50번) 조회

Offset 방식: OFFSET 50 → 46~95번 반환 (51~100번 중 5개 누락)
Cursor 방식: WHERE id < 50 ORDER BY id DESC → 정확히 1~49번 반환
```

---

## ADR-010. WebSocket STOMP 채택 — 1:1 채팅 (도전 기능)

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 (도전 기능) |
| **관련 기능** | FN-CHAT (도전) |

### 컨텍스트

보호자-시터 간 1:1 채팅이 필요하다. 제안 금액 협의, 케어 중 소통 등에 사용된다.

### 비교

| 항목 | Long Polling | SSE | WebSocket + STOMP |
|------|-------------|-----|-------------------|
| 통신 방향 | 단방향 | 단방향 (서버→클라이언트) | ✅ 양방향 |
| 채팅 적합성 | 부적합 | 부적합 (전송 별도 HTTP 필요) | ✅ 단일 연결로 송수신 |
| 채팅방 라우팅 | 직접 구현 | 직접 구현 | ✅ STOMP pub/sub 자동화 |

### 결정

**WebSocket + STOMP (Spring 내장 Simple Broker)**

```
연결:    wss://api.forpets.io/ws-stomp
구독:    /sub/chat/room/{chatRoomId}
전송:    /pub/chat/message
```

### 트레이드오프

Spring 내장 Simple Broker 사용 → 서버 재시작 시 구독 정보 초기화. 스케일아웃 시 서버 간 메시지 공유 불가. 향후 Redis Pub/Sub 브로커로 전환 필요. MVP 범위에서는 단일 서버이므로 문제없음.

---

## ADR-011. 스케줄러 채택 — 렌탈 반납 지연 자동 패널티

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **결정자** | 이지민 |
| **관련 기능** | FN-RENTAL-05 |

### 컨텍스트

렌탈 반납 기한 초과 시 패널티를 자동으로 부과해야 한다. 수동 처리하면 누락 가능성이 있다.

### 비교

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| 수동 처리 | 관리자가 직접 패널티 부과 | 구현 단순 | 누락 가능성 높음 |
| **스케줄러** ✅ | `@Scheduled`로 매일 00:00 자동 감지 | 자동화, 누락 없음 | 스케줄러 구현 필요 |
| 조회 시점 동적 판단 | 응답 DTO에서 계산 | DB 변경 없음 | DB 상태와 응답 불일치 |

### 결정

**`@Scheduled(cron = "0 0 0 * * *")` 매일 00:00 자동 패널티 부과.**

```java
@Scheduled(cron = "0 0 0 * * *")
public void processOverdueRentals() {
    // ACTIVE + endDate < 오늘 목록 조회
    // status → OVERDUE + penaltyAmount 계산 (지연일 × dailyPrice × 50%)
    // 보호자에게 SSE 알림 발송
}
```

### 트레이드오프

스케줄러 장애 시 다음 실행 시 자동 복구 (누적 미처리 건 일괄 처리). 최대 24시간 지연 발생 가능 (허용 범위).

---

## ADR-012. API 설계 — 취소·상태 변경은 POST /action 사용

| 항목 | 내용 |
|------|------|
| **날짜** | 2026-05-12 |
| **상태** | 승인됨 |
| **관련 API** | POST /api/postings/{id}/close, POST .../proposals/{id}/accept |

### 컨텍스트

케어 공고 취소, 제안 수락, 케어 완료 확인 등 상태 전이가 수반되는 작업의 HTTP 메서드를 결정해야 한다.

### 결정

**복잡한 상태 전이가 수반되는 API는 `POST /{resource}/{id}/{action}` 사용.**

| API | 채택 방식 | 이유 |
|-----|---------|------|
| 공고 취소 | `PATCH /postings/{id}/close` | status만 변경하는 단순 상태 전이 |
| 제안 수락 | `POST /proposals/{id}/accept` | Kafka 이벤트·알림 등 복잡한 부수 효과 수반 |
| 케어 완료 | `PATCH /matchings/{id}/complete` | status 변경 + 후기 가능 상태로 전환 |
| 반납 처리 | `POST /rentals/{id}/return` | 재고 복원·패널티 계산 수반 |

### 이유

1. **의미론**: 단순 삭제(`DELETE`)가 아닌 상태 전이임을 명확히 표현
2. **복잡한 부수 효과**: Kafka 이벤트 발행, 알림 전송, 재고 복원 등은 단순 `DELETE`/`PUT` 시맨틱으로 표현 불가
3. **HTTP 멱등성**: `DELETE`는 멱등성을 기대하지만, 이미 취소된 리소스에 재요청 시 400을 반환해야 함

---

## 주요 변경 이력

| 버전 | ADR | 변경 내용 | 일자 |
|------|-----|----------|------|
| v1.0 | 전체 | 최초 작성 — ForPets 포펫츠 프로젝트 기준 | 2026-05-12 |

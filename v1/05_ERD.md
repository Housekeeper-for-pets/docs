# 🗄️ ERD (Entity Relationship Diagram)

> **프로젝트명:** ForPets (포펫츠)
> **팀명:** 집사조
> **문서 버전:** v1.2
> **최종 수정일:** 2026-05-13
> **연결 문서:** 01_프로젝트_개요서, 03_유스케이스_명세서, 04_기능_명세서

---

## 1. ERD 다이어그램

```mermaid
erDiagram

    %% ====================== User Context ======================
    USER {
        bigint id PK
        varchar email UK "로그인 아이디"
        varchar password "BCrypt hashed"
        varchar nickname
        varchar phone "전화번호 (Nullable)"
        varchar gender "성별 (Nullable)"
        varchar role "MEMBER, SITTER, ADMIN"
        varchar status "ACTIVE, SUSPENDED, DELETED"
        datetime created_at
        datetime updated_at
    }

    PET {
        bigint id PK
        bigint owner_id FK "보호자 (USER)"
        varchar name "반려동물 이름"
        varchar species "DOG, CAT, ETC"
        varchar breed "품종 (Nullable)"
        varchar size "SMALL, MEDIUM, LARGE (Nullable)"
        int age "(Nullable)"
        varchar gender "MALE, FEMALE, UNKNOWN (Nullable)"
        varchar profile_image_url "프로필 이미지 URL (Nullable)"
        text special_notes "특이사항 (Nullable)"
        datetime created_at
        datetime updated_at
    }

    SITTER_PROFILE {
        bigint id PK
        bigint user_id FK UK "시터 (USER)"
        varchar region "서비스 지역"
        text introduction "자기소개 (Nullable)"
        int experience_years "경력 (연 단위)"
        varchar available_pet_type "DOG, CAT, DOG_CAT, ETC"
        varchar available_pet_size "SMALL, MEDIUM, LARGE, ALL (Nullable)"
        int hourly_rate "시급 (원)"
        decimal avg_rating "평균 평점 (Redis ZSET 캐싱)"
        int review_count "누적 후기 수"
        varchar status "ACTIVE, INACTIVE, DELETED"
        datetime created_at
        datetime updated_at
    }

    SITTER_SCHEDULE {
        bigint id PK
        bigint sitter_profile_id FK "시터 프로필"
        date available_date "가능 일자"
        time start_time "시작 시간"
        time end_time "종료 시간"
        varchar status "AVAILABLE, RESERVED, CLOSED"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Posting Context (역방향 매칭) ======================
    POST {
        bigint id PK
        bigint author_id FK "작성자 (USER)"
        bigint pet_id FK "반려동물 (Nullable)"
        varchar title "제목"
        text content "내용"
        varchar post_type "OWNER_REQUEST (v1 사용값)"
        varchar category "CARE, WALK, TEMP_CARE, ETC"
        varchar region "지역 (Nullable)"
        date care_start_date "케어 시작일"
        date care_end_date "케어 종료일"
        time care_start_time "케어 시작 시간"
        time care_end_time "케어 종료 시간"
        varchar status "ACTIVE, CLOSED, DELETED"
        int view_count "조회수"
        datetime created_at
        datetime updated_at
    }

    SITTER_PROPOSAL {
        bigint id PK
        bigint post_id FK "대상 게시글"
        bigint sitter_profile_id FK "제안 시터 프로필"
        int proposed_price "제안 금액 (원)"
        date available_date "가능 날짜"
        time available_start_time "가능 시작 시간"
        time available_end_time "가능 종료 시간"
        text message "한줄 메시지 (Nullable)"
        varchar status "PENDING, ACCEPTED, REJECTED, WITHDRAWN"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Forward Matching Context (순방향 매칭) ======================
    CARE_REQUEST {
        bigint id PK
        bigint owner_id FK "보호자 (USER)"
        bigint sitter_profile_id FK "대상 시터 프로필"
        bigint pet_id FK "반려동물"
        date desired_date "희망 돌봄 날짜"
        time desired_start_time "희망 시작 시간"
        time desired_end_time "희망 종료 시간"
        varchar care_type "VISIT, BOARDING"
        text message "요청 메시지 (Nullable)"
        varchar status "PENDING, ACCEPTED, REJECTED, CANCELLED"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Reservation Context ======================
    RESERVATION {
        bigint id PK
        bigint owner_id FK "보호자 (USER)"
        bigint sitter_profile_id FK "시터 프로필"
        bigint pet_id FK "예약 대상 반려동물"
        bigint sitter_schedule_id FK "시터 일정"
        date reservation_date "예약 날짜"
        time start_time "시작 시간"
        time end_time "종료 시간"
        int price "가격"
        varchar status "PAYMENT_PENDING, CONFIRMED, IN_PROGRESS, COMPLETED, CANCELED, EXPIRED"
        text request_memo "요청 메모 (Nullable)"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Review Context (양방향) ======================
    REVIEW {
        bigint id PK
        bigint reservation_id FK "대상 예약"
        bigint reviewer_id FK "평가자 (USER)"
        bigint reviewee_id FK "대상자 (USER)"
        int rating "1~5 별점"
        text content "후기 내용 (Nullable)"
        varchar status "ACTIVE, HIDDEN, DELETED"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Payment Context ======================
    PAYMENT {
        bigint id PK
        bigint reservation_id FK "대상 예약"
        bigint user_id FK "결제자 (USER)"
        int amount "결제금액"
        varchar method "PORTONE_CARD, PORTONE_KAKAO_PAY, ETC"
        varchar status "READY, PAID, FAILED, CANCELED"
        varchar merchant_order_id UK "서버 생성 결제 고유번호"
        varchar portone_payment_id "PortOne 결제 ID (Nullable)"
        varchar portone_transaction_id "PortOne 거래 ID (Nullable)"
        datetime paid_at "결제 완료 일시 (Nullable)"
        varchar failure_reason "결제 실패 사유 (Nullable)"
        datetime created_at
        datetime updated_at
    }

    WEBHOOK_EVENT {
        bigint id PK
        varchar webhook_uid "웹훅 고유 ID (Nullable)"
        bigint payment_id FK "결제 (Nullable)"
        varchar event_status "이벤트 상태 (Nullable)"
        datetime received_at "수신 시각 (Nullable)"
        datetime processed_at "처리 완료 시각 (Nullable)"
        datetime created_at
        datetime updated_at
    }

    %% ====================== 관계 정의 ======================
    USER ||--o{ PET : "owns"
    USER ||--o| SITTER_PROFILE : "has"
    SITTER_PROFILE ||--o{ SITTER_SCHEDULE : "has"

    USER ||--o{ POST : "writes"
    PET ||--o{ POST : "featured_in"

    POST ||--o{ SITTER_PROPOSAL : "receives"
    SITTER_PROFILE ||--o{ SITTER_PROPOSAL : "submits"

    USER ||--o{ CARE_REQUEST : "sends"
    SITTER_PROFILE ||--o{ CARE_REQUEST : "receives"
    PET ||--o{ CARE_REQUEST : "for"

    USER ||--o{ RESERVATION : "books"
    SITTER_PROFILE ||--o{ RESERVATION : "serves"
    PET ||--o{ RESERVATION : "cared_in"
    SITTER_SCHEDULE ||--o{ RESERVATION : "scheduled_in"

    RESERVATION ||--o{ REVIEW : "reviewed_by"
    USER ||--o{ REVIEW : "writes_review"
    USER ||--o{ REVIEW : "receives_review"

    RESERVATION ||--o| PAYMENT : "paid_by"
    USER ||--o{ PAYMENT : "pays"

    PAYMENT ||--o{ WEBHOOK_EVENT : "triggers"
```

---

## 2. 테이블 설계 주요 결정 사항

### 2-1. 역방향 매칭 구조 — POST + SITTER_PROPOSAL

보호자가 케어 공고를 등록하면 시터가 역으로 제안을 입찰하는 구조.

| 테이블 | 역할 | 결정 사항 |
|--------|------|-----------|
| `POST` | 보호자가 올리는 케어 공고 | status(ACTIVE→CLOSED), 케어 시간 명시 필수 |
| `SITTER_PROPOSAL` | 시터의 역방향 제안 입찰 | `(post_id, sitter_profile_id)` UNIQUE — 동일 시터 중복 제안 DB 레벨 차단 |

**SITTER_PROPOSAL (post_id, sitter_profile_id) UNIQUE 근거:**

여러 시터가 동시에 같은 공고에 제안을 입찰할 때, 한 시터가 중복 제안을 등록하는 것을 방지한다. DB Unique 제약으로 원천 차단하여 별도 분산 락 없이도 정합성을 보장한다.

```sql
CREATE UNIQUE INDEX uk_proposal_post_sitter
    ON sitter_proposal (post_id, sitter_profile_id);
```

### 2-2. 순방향 매칭 구조 — CARE_REQUEST

보호자가 시터를 직접 검색하고 케어를 신청하는 구조.

| 테이블 | 역할 | 결정 사항 |
|--------|------|-----------|
| `CARE_REQUEST` | 보호자가 시터에게 직접 보내는 케어 신청 | 희망 시간 명시 필수, 시터가 전체 시간 수용 가능해야 수락 가능 |

시터가 신청을 수락(ACCEPTED)하면 예약(RESERVATION)이 생성되어 케어가 진행된다.

### 2-3. v1 시간 제약 정책

> v1에서는 시간 조율 없이, 게시글/요청에 명시된 시간 전체에 가능한 시터만 제안·수락할 수 있다.
> 시간 조율(부분 수락, 역제안 등)은 v2에서 1:1 채팅 도입과 함께 지원한다.

```
v1:  Request/Offer ──────────────────────→ 수락 → Reservation

v2:  Request/Offer → Chat/협상 → 최종 Offer → 수락 → Reservation
```

**역방향 (제안 입찰 시):**

시터가 제안을 보낼 때 POST에 명시된 케어 시간(`care_start_date ~ care_end_date`, `care_start_time ~ care_end_time`) 전체에 대해 가능해야 한다. 시스템은 시터의 `SITTER_SCHEDULE`과 POST의 시간을 비교하여 전체 커버 가능 여부를 검증한다.

**순방향 (요청 수락 시):**

보호자가 요청에 명시한 시간(`desired_date`, `desired_start_time ~ desired_end_time`) 전체에 대해 시터가 가능해야 수락할 수 있다.

**수락 시 시터 스케줄 충돌 처리:**

제안/요청이 수락되어 RESERVATION이 생성되는 시점에, 해당 시터의 다른 PENDING 제안 중 시간이 겹치는 것을 자동 WITHDRAWN 처리하고 SSE 알림을 전송한다.

```
시간 겹침 판정:
  same date range AND startTime < other.endTime AND endTime > other.startTime
```

알림 예시: "보호자님이 해당 시간대에 다른 시터와 예약을 확정했습니다. 시간을 수정하여 다시 제안하시겠습니까?"

WITHDRAWN된 제안의 게시글(POST)이 아직 ACTIVE 상태이면, 시터는 시간을 수정하여 재제안할 수 있다.

### 2-4. RESERVATION — 매칭 확정 후 예약 관리

순방향(CARE_REQUEST 수락)과 역방향(SITTER_PROPOSAL 수락) 양쪽 플로우에서 매칭이 확정되면 RESERVATION이 생성된다.

```
순방향: CARE_REQUEST(ACCEPTED) → RESERVATION 생성
역방향: SITTER_PROPOSAL(ACCEPTED) → RESERVATION 생성
```

**예약 상태 흐름:**

```
PAYMENT_PENDING → CONFIRMED → IN_PROGRESS → COMPLETED → (양방향 후기 작성)
       │               │
       ▼               ▼
    EXPIRED         CANCELED
```

| 상태 | 설명 | 전이 가능 상태 |
|------|------|---------------|
| `PAYMENT_PENDING` | 결제 대기 중 | → CONFIRMED, EXPIRED |
| `CONFIRMED` | 결제 완료, 케어 예약 확정 | → IN_PROGRESS, CANCELED |
| `IN_PROGRESS` | 케어 진행 중 (취소 불가) | → COMPLETED |
| `COMPLETED` | 케어 완료. 양방향 후기 작성 가능 | — (종료 상태) |
| `CANCELED` | 보호자 또는 시터가 취소 (CONFIRMED 이전만 가능) | — (종료 상태) |
| `EXPIRED` | 결제 기한 초과 자동 만료 | — (종료 상태) |

> **CANCELED는 PAYMENT_PENDING 또는 CONFIRMED 상태에서만 전이 가능하다. IN_PROGRESS 이후에는 취소할 수 없다.**

### 2-5. REVIEW — 양방향 후기 (보호자 ↔ 시터)

Reservation 상태가 `COMPLETED`로 변경된 시점부터 보호자와 시터 모두 상대방에 대한 후기를 작성할 수 있다.

| 작성자 | 대상 | 제약 |
|--------|------|------|
| 보호자 | 시터 | `(reservation_id, reviewer_id)` UNIQUE — 중복 작성 방지 |
| 시터 | 보호자 | `(reservation_id, reviewer_id)` UNIQUE — 중복 작성 방지 |

```sql
CREATE UNIQUE INDEX uk_review_reservation_reviewer
    ON review (reservation_id, reviewer_id);
```

1건의 예약에 대해 최대 2건의 후기(보호자 1건 + 시터 1건)가 작성된다.

### 2-6. PAYMENT + WEBHOOK_EVENT — Portone 결제 연동

| 테이블 | 역할 | 결정 사항 |
|--------|------|-----------|
| `PAYMENT` | 결제 정보 관리 | `merchant_order_id` UNIQUE — 서버에서 생성한 결제 고유번호로 멱등성 보장 |
| `WEBHOOK_EVENT` | Portone 웹훅 이력 | 웹훅 수신·처리 이력 보관, 재처리 대비 |

**결제 상태 흐름:**

```
READY → PAID
  │       │
  ▼       ▼
FAILED  CANCELED
```

### 2-7. SITTER_SCHEDULE — 시터 가능 시간 관리

시터가 서비스 가능한 일자·시간대를 등록하고, 예약 시 해당 일정의 상태가 `RESERVED`로 변경된다.

| 상태 | 설명 |
|------|------|
| `AVAILABLE` | 예약 가능 |
| `RESERVED` | 예약 확정됨 |
| `CLOSED` | 시터가 직접 비활성화 |

### 2-8. POST — 케어 공고 게시물

v1에서 게시물은 보호자의 **케어 공고(`OWNER_REQUEST`)** 용도로만 사용한다. `post_type` 필드는 확장을 위해 enum으로 유지하되, v1에서는 `OWNER_REQUEST` 값만 허용한다.

| post_type | 용도 | 비고 |
|-----------|------|------|
| `OWNER_REQUEST` | 보호자 케어 공고 (역방향 매칭 대상) | v1 사용 |

> 확장 시 `SITTER_OFFER`(시터 홍보), `GENERAL`(일반 커뮤니티) 등 추가 예정.

케어 공고에는 반드시 케어 시간(`care_start_date`, `care_end_date`, `care_start_time`, `care_end_time`)을 명시해야 하며, 시터는 해당 시간 전체에 가능해야만 제안할 수 있다.

### 2-9. USER.role — 역할 기반 접근 제어

| 역할 | 주요 기능 |
|------|-----------|
| `MEMBER` | 케어 공고 등록, 시터 검색·신청, 후기 작성 |
| `SITTER` | MEMBER의 모든 기능 + 시터 프로필 관리, 제안 입찰, 케어 일지 등록 |
| `ADMIN` | 관리자 기능 |

역할은 회원가입 시 선택하며, 이후에도 시터 등록/탈퇴를 통해 `MEMBER ↔ SITTER` 전환이 가능하다.

```
MEMBER → 시터 등록 → SITTER
SITTER → 시터 탈퇴 → MEMBER
```

### 2-10. 서버 타임존

모든 `datetime`, `date`, `time` 필드는 **KST(UTC+9)** 기준으로 저장하고 반환한다.
```
spring.jpa.properties.hibernate.jdbc.time_zone=Asia/Seoul
MySQL: serverTimezone=Asia/Seoul
```

---

## 3. 인덱스 설계

### v1 MVP 인덱스

| 테이블 | 인덱스 대상 컬럼 | 타입 | 이유 |
|--------|----------------|------|------|
| SITTER_PROPOSAL | `(post_id, sitter_profile_id)` | 복합 UK | 동일 시터 중복 제안 방지 (UNIQUE) |
| SITTER_PROPOSAL | `(post_id, status)` | 복합 | 게시글별 제안 목록 조회 |
| SITTER_PROPOSAL | `(sitter_profile_id, status)` | 복합 | 시터별 제안 내역 조회 |
| POST | `(post_type, status, region)` | 복합 | 타입·지역별 ACTIVE 게시글 검색 |
| POST | `(author_id, status)` | 복합 | 내 게시글 목록 조회 |
| CARE_REQUEST | `(owner_id, status)` | 복합 | 보호자별 신청 내역 조회 |
| CARE_REQUEST | `(sitter_profile_id, status)` | 복합 | 시터별 수신 신청 조회 |
| RESERVATION | `(owner_id, status)` | 복합 | 보호자별 예약 목록 조회 |
| RESERVATION | `(sitter_profile_id, status)` | 복합 | 시터별 예약 목록 조회 |
| RESERVATION | `(status, reservation_date)` | 복합 | 상태별 예약 일자 조회 (스케줄러) |
| REVIEW | `(reservation_id, reviewer_id)` | 복합 UK | 중복 후기 방지 (UNIQUE) |
| REVIEW | `(reviewee_id)` | 단일 | 대상자별 후기 목록 조회 |
| PAYMENT | `merchant_order_id` | 단일 UK | 결제 고유번호 유일성 보장 (UNIQUE) |
| PAYMENT | `(reservation_id)` | 단일 | 예약별 결제 조회 |
| SITTER_SCHEDULE | `(sitter_profile_id, available_date, status)` | 복합 | 시터별 가능 일정 조회 |
| SITTER_PROFILE | `(region, status)` | 복합 | 지역별 활성 시터 검색 |

---

## 4. 도메인 경계 정리

```
[User Context]
  └─ USER, PET, SITTER_PROFILE, SITTER_SCHEDULE

[Posting Context]  ← 역방향 매칭 핵심 도메인
  └─ POST, SITTER_PROPOSAL

[Matching Context]  ← 순방향 매칭
  └─ CARE_REQUEST

[Reservation Context]  ← 매칭 확정 후 케어 관리
  └─ RESERVATION

[Review Context]
  └─ REVIEW

[Payment Context]
  └─ PAYMENT, WEBHOOK_EVENT
```

각 컨텍스트는 다른 컨텍스트의 엔티티를 직접 참조하지 않고 ID(FK)로만 참조한다.

**확장 시 추가 예정 컨텍스트:**

```
[Rental Context]       ← 단기 렌탈 (확장)
[TempCare Context]     ← 임시보호 매칭 (확장)
[Notification Context] ← SSE 알림 이력 (확장)
[Chat Context]         ← 1:1 채팅 (확장) — 시간 조율 지원
[Care Log Context]     ← 케어 일지 (확장)
```

---

## 5. Redis 키 설계

| Redis 키 | 값 | TTL | 용도 |
|----------|---|-----|------|
| `sitter:ratings` | ZSET (sitterId, 평점) | 없음 | 시터 평점 랭킹 (ZINCRBY) |
| `cache:posts:{조건해시}` | JSON (게시글 목록) | 5분 | 게시글 검색 결과 캐싱 |
| `cache:sitters:{조건해시}` | JSON (시터 목록) | 1시간 | 시터 목록 검색 결과 캐싱 |
| `cache:sitter-profile:{sitterProfileId}` | JSON (시터 프로필) | 1시간 | 시터 프로필 캐싱 |
| `cache:pet-profile:{petId}` | JSON (반려동물 프로필) | 1시간 | 반려동물 프로필 캐싱 |
| `cache:top-sitters` | JSON (인기 시터 Top 10) | 1시간 | 인기 시터 목록 캐싱 |
| `jwt:blacklist:{token}` | "1" | Access Token TTL | JWT 로그아웃 블랙리스트 |
| `sse:session:{userId}` | JSON (세션 정보) | 30분 | SSE 연결 세션 관리 |

---

## 6. 주요 변경 이력

| 버전 | 변경 내용 | 사유 |
|------|-----------|------|
| v1.0 | 최초 작성 — 역방향 매칭 구조 반영 | 프로젝트 시작 |
| v1.1 | 회의 후 수정사항 반영 | ERD 재설계 및 프로젝트 개요서 v1.1 피드백 반영 |
| v1.2 | v1 시간 제약 정책·역할·예약 상태 수정 | edge case 분석 결과 반영 |

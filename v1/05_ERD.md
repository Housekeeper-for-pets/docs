# 🗄️ ERD (Entity Relationship Diagram)

> **프로젝트명:** ForPets (포펫츠)
> **팀명:** 집사조
> **문서 버전:** v1.0
> **최종 수정일:** 2026-05-12
> **연결 문서:** 01_프로젝트_개요서, 03_유스케이스_명세서, 04_기능_명세서

---

## 1. ERD 다이어그램

```mermaid
erDiagram

    %% ====================== User Context ======================
    USER {
        bigint user_id PK
        varchar email UK "로그인 아이디"
        varchar password "BCrypt hashed"
        varchar nickname
        enum role "GUARDIAN, SITTER, RESCUE"
        varchar profile_image "프로필 이미지 URL (Nullable)"
        varchar address "주소 (시·구 단위)"
        boolean is_certified "시터 인증 여부"
        datetime created_at
        datetime updated_at
    }

    PET {
        bigint pet_id PK
        bigint user_id FK "보호자 (GUARDIAN)"
        varchar name "반려동물 이름"
        enum species "DOG, CAT, ETC"
        varchar breed "견종/묘종"
        int age
        decimal weight "체중 (kg)"
        enum size "SMALL, MEDIUM, LARGE"
        boolean neutered "중성화 여부"
        text special_notes "특이사항 (분리불안, 알러지 등)"
        varchar image_url "반려동물 사진 URL (Nullable)"
        datetime created_at
    }

    SITTER_PROFILE {
        bigint sitter_profile_id PK
        bigint user_id FK "시터 (SITTER)"
        text introduction "자기소개"
        int experience_years "경력 연수"
        varchar specialty "전문 분야 (소형견, 노령견, 의료케어)"
        varchar service_area "서비스 지역"
        decimal avg_rating "평균 평점 (Redis ZSET 캐싱)"
        int review_count "누적 후기 수"
        text ai_generated_bio "AI 자동 생성 프로필 (Nullable)"
        boolean is_active "활성 여부"
        datetime updated_at
    }

    %% ====================== Posting Context (역방향 매칭 핵심) ======================
    CARE_POSTING {
        bigint posting_id PK
        bigint guardian_id FK "보호자 (USER)"
        bigint pet_id FK "케어 대상 반려동물"
        varchar title "공고 제목"
        text description "요구사항 상세"
        date start_date "케어 시작일"
        date end_date "케어 종료일"
        varchar location "지역 (서울 마포구)"
        int budget "최대 예산 (원)"
        datetime proposal_deadline "제안 마감일시"
        enum status "OPEN, MATCHED, CLOSED"
        int proposal_count "제안 수 (Redisson 분산락으로 정합성 보장)"
        datetime created_at
        datetime updated_at
    }

    SITTER_PROPOSAL {
        bigint proposal_id PK
        bigint posting_id FK "대상 공고"
        bigint sitter_id FK "제안한 시터 (USER)"
        int proposed_price "제안 금액 (원)"
        text message "제안 메시지"
        varchar special_offer "추가 제안 (Nullable)"
        enum status "PENDING, ACCEPTED, REJECTED"
        datetime created_at
    }

    MATCHING {
        bigint matching_id PK
        bigint posting_id FK UK "1공고 = 1매칭 (UNIQUE — 중복 매칭 DB 레벨 차단)"
        bigint proposal_id FK "수락된 제안"
        bigint guardian_id FK "보호자 (USER)"
        bigint sitter_id FK "시터 (USER)"
        int final_price "최종 확정 금액"
        enum status "CONFIRMED, IN_PROGRESS, DONE, CANCELLED"
        enum payment_status "PENDING, PAID, REFUNDED"
        datetime created_at
        datetime updated_at
    }

    %% ====================== Care Log Context ======================
    CARE_LOG {
        bigint log_id PK
        bigint matching_id FK "소속 매칭"
        bigint sitter_id FK "작성 시터"
        text content "일지 내용"
        json image_urls "사진 URL 목록 (Nullable)"
        datetime created_at
    }

    %% ====================== Review Context ======================
    REVIEW {
        bigint review_id PK
        bigint matching_id FK UK "1매칭 = 1후기 (UNIQUE)"
        bigint reviewer_id FK "작성자 (보호자, USER)"
        bigint reviewee_id FK "대상 시터 (USER)"
        decimal rating "별점 (1.0 ~ 5.0, 0.5 단위)"
        text content "후기 내용"
        text ai_summary "AI 자동 요약 (Nullable)"
        datetime created_at
    }

    %% ====================== Rental Context ======================
    RENTAL_ITEM {
        bigint item_id PK
        bigint owner_id FK "등록자 (USER)"
        varchar name "용품명"
        enum category "KENNEL, STROLLER, CARRIER, ETC"
        int daily_price "일일 대여료 (원)"
        int deposit "보증금 (원)"
        int stock "재고 수량 (Redisson 분산락으로 정합성 보장)"
        enum status "AVAILABLE, RENTED"
        varchar location "보관 지역"
        datetime created_at
    }

    RENTAL {
        bigint rental_id PK
        bigint item_id FK "렌탈 용품"
        bigint user_id FK "대여자 (GUARDIAN)"
        date start_date "대여 시작일"
        date end_date "반납 예정일"
        date return_date "실제 반납일 (Nullable)"
        int total_price "대여료 합계"
        int penalty_amount "지연 패널티 금액 (0 기본)"
        enum status "ACTIVE, RETURNED, OVERDUE"
        datetime created_at
    }

    %% ====================== TempCare Context ======================
    TEMP_CARE_POSTING {
        bigint temp_care_id PK
        bigint rescue_id FK "등록한 구조단체 (USER, RESCUE 역할)"
        varchar animal_name "동물 이름"
        enum species "DOG, CAT, ETC"
        int age "나이"
        boolean neutered "중성화 여부"
        date period_start "임시보호 시작일"
        date period_end "임시보호 종료일"
        text requirements "보호 조건 (아파트 가능 여부, 선임묘 여부 등)"
        varchar image_url "사진 URL (Nullable)"
        enum status "OPEN, MATCHED, ADOPTED"
        datetime created_at
    }

    TEMP_CARE_APPLICATION {
        bigint application_id PK
        bigint temp_care_id FK "대상 임시보호 공고"
        bigint applicant_id FK "지원자 (GUARDIAN, USER)"
        text message "지원 메시지"
        varchar home_environment "주거 환경 설명"
        enum status "PENDING, ACCEPTED, REJECTED"
        datetime created_at
    }

    %% ====================== Notification Context ======================
    NOTIFICATION {
        bigint notification_id PK
        bigint user_id FK "수신자"
        enum type "PROPOSAL_ARRIVED, MATCHING_CONFIRMED, CARE_LOG, RENTAL_OVERDUE, TEMP_CARE_MATCHED"
        varchar message "알림 메시지"
        boolean is_read "읽음 여부"
        datetime created_at
    }

    %% ====================== Chat Context (도전) ======================
    CHAT_ROOM {
        bigint chat_room_id PK
        bigint guardian_id FK "보호자 (USER)"
        bigint sitter_id FK "시터 (USER)"
        bigint matching_id FK "연관 매칭 (Nullable)"
        enum status "OPEN, CLOSED"
        datetime created_at
        datetime updated_at
    }

    CHAT_MESSAGE {
        bigint chat_message_id PK
        bigint chat_room_id FK
        bigint sender_id FK "발신자 (USER)"
        enum sender_role "GUARDIAN, SITTER"
        text content
        datetime sent_at
    }

    %% ====================== 관계 정의 ======================
    USER ||--o{ PET : "owns (GUARDIAN)"
    USER ||--o| SITTER_PROFILE : "has (SITTER)"

    USER ||--o{ CARE_POSTING : "creates (GUARDIAN)"
    PET ||--o{ CARE_POSTING : "featured_in"

    CARE_POSTING ||--o{ SITTER_PROPOSAL : "receives"
    USER ||--o{ SITTER_PROPOSAL : "submits (SITTER)"

    CARE_POSTING ||--o| MATCHING : "confirmed_as"
    SITTER_PROPOSAL ||--o| MATCHING : "accepted_as"
    USER ||--o{ MATCHING : "guardian_in"
    USER ||--o{ MATCHING : "sitter_in"

    MATCHING ||--o{ CARE_LOG : "has"
    MATCHING ||--o| REVIEW : "reviewed_as"

    USER ||--o{ RENTAL_ITEM : "registers"
    RENTAL_ITEM ||--o{ RENTAL : "rented_as"
    USER ||--o{ RENTAL : "rents (GUARDIAN)"

    USER ||--o{ TEMP_CARE_POSTING : "creates (RESCUE)"
    TEMP_CARE_POSTING ||--o{ TEMP_CARE_APPLICATION : "receives"
    USER ||--o{ TEMP_CARE_APPLICATION : "applies (GUARDIAN)"

    USER ||--o{ NOTIFICATION : "receives"

    USER ||--o{ CHAT_ROOM : "guardian_chat"
    USER ||--o{ CHAT_ROOM : "sitter_chat"
    CHAT_ROOM ||--|{ CHAT_MESSAGE : "contains"
```

---

## 2. 테이블 설계 주요 결정 사항

### 2-1. 역방향 매칭 구조 — CARE_POSTING + SITTER_PROPOSAL + MATCHING

ForPets의 핵심 차별화인 역방향 매칭을 3개 테이블로 구현한다.

| 테이블 | 역할 | 결정 사항 |
|--------|------|-----------|
| `CARE_POSTING` | 보호자가 올리는 케어 공고 | status(OPEN→MATCHED→CLOSED), proposal_count (Redisson 분산락) |
| `SITTER_PROPOSAL` | 시터의 역방향 제안 입찰 | 1공고에 N개 제안, PENDING→ACCEPTED/REJECTED |
| `MATCHING` | 확정된 케어 계약 | posting_id UNIQUE — DB 레벨 중복 매칭 차단 |

**MATCHING.posting_id UNIQUE 근거:**

하나의 케어 공고에 반드시 1개의 매칭만 존재해야 한다. Redisson 분산락이 1차 방어선이라면, posting_id UNIQUE 제약이 2차 방어선이다. 락 구현 버그나 Redis 장애 상황에서도 중복 매칭을 DB 레벨에서 원천 차단한다.

```sql
CREATE TABLE matching (
    matching_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    posting_id  BIGINT UNIQUE NOT NULL,    -- 공고당 1건만 허용
    proposal_id BIGINT UNIQUE NOT NULL,
    -- ...
    FOREIGN KEY (posting_id)  REFERENCES care_posting(posting_id),
    FOREIGN KEY (proposal_id) REFERENCES sitter_proposal(proposal_id)
);
```

### 2-2. CARE_POSTING.proposal_count — Redisson 분산락으로 정합성 보장

여러 시터가 동시에 하나의 공고에 제안을 입찰할 때 `proposal_count`의 정합성 문제가 발생한다.

```
T=0ms  시터 A: POST .../proposals  ──┐
T=1ms  시터 B: POST .../proposals  ──┤──▶ 동시 도달
T=2ms  시터 C: POST .../proposals  ──┘

기대: proposal_count = 3 (순서대로 각 1씩 증가)
위험: 3명이 동시에 proposal_count = 0 을 읽고 모두 1로 업데이트
결과(제어 없을 때): proposal_count = 1 (Lost Update)
```

**해결책:** `lock:proposal:{postingId}` Redisson 분산락으로 직렬 처리.

### 2-3. SITTER_PROFILE — AI 연계 필드

`ai_generated_bio` 필드에 LLM이 자동 생성한 프로필 문구를 JSON으로 저장한다.

```json
{
  "title": "소형견 케어 전문 집사",
  "description": "말티즈·포메라니안 3년 케어 경험...",
  "specialties": ["분리불안 케어", "산책 전문"],
  "keywords": ["말티즈", "포메", "집사"]
}
```

AI 장애 시에도 시터 프로필 조회는 `introduction` 필드(수동 작성)로 정상 동작한다. AI 기능과 핵심 서비스는 완전 격리된다.

### 2-4. RENTAL — Redisson 재고 락 + 스케줄러 자동 패널티

| 테이블 | 동시성 제어 | 자동화 |
|--------|-----------|--------|
| `RENTAL_ITEM.stock` | Redisson 분산락 (`lock:rental:{itemId}`) | — |
| `RENTAL.status` | — | 스케줄러: ACTIVE + endDate 초과 → OVERDUE + 패널티 자동 부과 |

**패널티 계산식:** `지연 일수 × daily_price × 50%`

### 2-5. REVIEW.matching_id UNIQUE — 1매칭 1후기

동일 매칭에 중복 후기를 방지하기 위해 `matching_id`에 UNIQUE 제약을 건다. `ai_summary` 필드에는 후기 텍스트를 LLM이 요약한 결과를 비동기로 저장한다.

### 2-6. USER.role — 역할 기반 접근 제어

| 역할 | 주요 기능 |
|------|-----------|
| `GUARDIAN` | 케어 공고 등록, 렌탈 신청, 임시보호 지원, 후기 작성 |
| `SITTER` | 시터 프로필 관리, 제안 입찰, 케어 일지 등록 |
| `RESCUE` | 임시보호 공고 등록, 임시보호자 선정 |

역할은 회원가입 시 1회만 선택 가능하며 이후 변경 불가.

### 2-7. NOTIFICATION — SSE 기반 실시간 알림 이력

SSE로 실시간 전송된 알림의 DB 이력을 보관한다. 보호자가 SSE 연결이 끊긴 상태에서 발생한 알림은 재연결 시 `is_read=false` 목록으로 제공된다.

### 2-8. 서버 타임존

모든 `datetime`, `date` 필드는 **KST(UTC+9)** 기준으로 저장하고 반환한다.
```
spring.jpa.properties.hibernate.jdbc.time_zone=Asia/Seoul
MySQL: serverTimezone=Asia/Seoul
```

---

## 3. 인덱스 설계

| 테이블 | 인덱스 대상 컬럼 | 타입 | 이유 |
|--------|----------------|------|------|
| CARE_POSTING | `(status, location)` | 복합 | 지역별 OPEN 공고 검색 빈번 |
| CARE_POSTING | `(guardian_id, status)` | 복합 | 내 공고 목록 조회 |
| CARE_POSTING | `(start_date, end_date)` | 복합 | 기간 조건 검색 |
| SITTER_PROPOSAL | `(posting_id, status)` | 복합 | 공고별 제안 목록 조회 |
| SITTER_PROPOSAL | `(sitter_id, status)` | 복합 | 시터별 제안 내역 조회 |
| MATCHING | `posting_id` | 단일 UK | 중복 매칭 방지 (UNIQUE) |
| MATCHING | `(guardian_id, status)` | 복합 | 보호자별 매칭 목록 조회 |
| MATCHING | `(sitter_id, status)` | 복합 | 시터별 매칭 목록 조회 |
| CARE_LOG | `(matching_id, created_at DESC)` | 복합 | 매칭별 일지 최신순 조회 |
| REVIEW | `matching_id` | 단일 UK | 중복 후기 방지 (UNIQUE) |
| REVIEW | `reviewee_id` | 단일 | 시터별 후기 목록 조회 |
| RENTAL | `(item_id, status)` | 복합 | 용품별 렌탈 상태 조회 |
| RENTAL | `(user_id, status)` | 복합 | 내 렌탈 내역 조회 |
| RENTAL | `(status, end_date)` | 복합 | 스케줄러 — ACTIVE + 기한 초과 탐색 |
| TEMP_CARE_POSTING | `(status, species)` | 복합 | 동물 종별 OPEN 공고 조회 |
| NOTIFICATION | `(user_id, is_read)` | 복합 | 미읽음 알림 조회 |
| CHAT_MESSAGE | `(chat_room_id, chat_message_id DESC)` | 복합 | 커서 기반 최신 메시지 조회 |

---

## 4. 도메인 경계 정리

```
[User Context]
  └─ USER, PET, SITTER_PROFILE

[Posting Context]  ← ForPets 핵심 차별화 도메인
  └─ CARE_POSTING, SITTER_PROPOSAL, MATCHING

[Care Log Context]
  └─ CARE_LOG

[Review Context]
  └─ REVIEW

[Rental Context]
  └─ RENTAL_ITEM, RENTAL

[TempCare Context]
  └─ TEMP_CARE_POSTING, TEMP_CARE_APPLICATION

[Notification Context]
  └─ NOTIFICATION

[Chat Context]  ← 도전 기능
  └─ CHAT_ROOM, CHAT_MESSAGE
```

각 컨텍스트는 다른 컨텍스트의 엔티티를 직접 참조하지 않고 ID(FK)로만 참조한다.

---

## 5. Redis 키 설계

| Redis 키 | 값 | TTL | 용도 |
|----------|---|-----|------|
| `lock:proposal:{postingId}` | uuid (락 소유자) | 10초 | 시터 제안 동시 입찰 Redisson 분산락 |
| `lock:rental:{itemId}` | uuid (락 소유자) | 15초 | 렌탈 재고 동시 차감 Redisson 분산락 |
| `sitter:ratings` | ZSET (sitterId, 평점) | 없음 | 시터 평점 랭킹 (ZINCRBY) |
| `cache:postings:{조건해시}` | JSON (공고 목록) | 60초 | 공고 검색 결과 캐싱 |
| `cache:sitter:{userId}` | JSON (시터 프로필) | 5분 | 시터 프로필 캐싱 |
| `cache:rental:items:{조건해시}` | JSON (용품 목록) | 30초 | 렌탈 용품 목록 캐싱 |
| `sse:session:{userId}` | JSON (세션 정보) | 30분 | AI 챗봇 멀티턴 세션 히스토리 |
| `ai:profile:cache:{sitterId}` | JSON (AI 프로필) | 1시간 | LLM 비용 절감 캐싱 |

---

## 6. 주요 변경 이력

| 버전 | 변경 내용 | 사유 |
|------|-----------|------|
| v1.0 | 최초 작성 — 포펫츠 역방향 매칭 구조 반영 | 프로젝트 시작 |

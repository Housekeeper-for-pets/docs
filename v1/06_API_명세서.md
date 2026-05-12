# 📡 API 명세서 — ForPets (포펫츠)

> **프로젝트명:** ForPets (포펫츠)
> **팀명:** 집사조
> **문서 버전:** v1.0
> **최종 수정일:** 2026-05-12
> **연결 문서:** 03_유스케이스_명세서, 04_기능_명세서, 05_ERD

---

## 문서 읽는 법

| 아이콘 | 의미 |
|--------|------|
| 🔐 | JWT 인증 필수 (`Authorization: Bearer {token}`) |
| 🔐👑 | JWT 인증 + RESCUE 또는 ADMIN 권한 필수 |
| ⚠️ | 동시성 민감 API — Redisson 분산락 적용 |
| 💾 | 캐싱 대상 API (Redis) |
| 🔓 | 인증 불필요 (공개 API) |
| 📡 | SSE (Server-Sent Events) 스트리밍 |

**Base URL:** `https://api.forpets.io`

**공통 에러 응답 형식:**
```json
{
  "status": 409,
  "code": "PROPOSAL_LOCK_FAILED",
  "message": "현재 처리 중입니다. 잠시 후 다시 시도해주세요.",
  "timestamp": "2026-05-12T10:00:00"
}
```

**서버 타임존:** 모든 `datetime` 필드는 **KST(UTC+9)** 기준으로 저장하고 반환한다.

---

## 1. 인증 (Auth)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/auth/signup` | 회원가입 (역할 선택 포함) | 🔓 | UC-001 |
| POST | `/api/auth/login` | 로그인 (JWT 발급) | 🔓 | UC-002 |

### POST /api/auth/signup

**Request**
```json
{
  "email": "kimboho@example.com",
  "password": "pass1234",
  "nickname": "뭉치보호자",
  "role": "GUARDIAN"
}
```
> `role`: `GUARDIAN` (보호자) / `SITTER` (시터) / `RESCUE` (구조단체)

**Response 201 Created**
```json
{
  "userId": 1,
  "email": "kimboho@example.com",
  "nickname": "뭉치보호자",
  "role": "GUARDIAN",
  "createdAt": "2026-05-12T10:00:00"
}
```

**Error Cases**
```json
// 409 — 이메일 중복
{ "status": 409, "code": "EMAIL_DUPLICATED", "message": "이미 가입된 이메일입니다." }

// 400 — 유효성 실패
{ "status": 400, "code": "VALIDATION_FAILED", "message": "비밀번호는 8자 이상, 영문+숫자를 포함해야 합니다." }
```

---

### POST /api/auth/login

**Request**
```json
{
  "email": "kimboho@example.com",
  "password": "pass1234"
}
```

**Response 200 OK**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

**Error Cases**
```json
// 401 — 인증 실패
{ "status": 401, "code": "AUTHENTICATION_FAILED", "message": "이메일 또는 비밀번호가 올바르지 않습니다." }
```

---

## 2. 사용자 (Users)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| GET | `/api/users/me` | 내 정보 조회 | 🔐 | UC-003 |
| PATCH | `/api/users/me` | 내 정보 수정 | 🔐 | UC-003 |
| GET | `/api/users/me/matchings` | 내 매칭 목록 조회 | 🔐 | UC-008 |

### GET /api/users/me

**Response 200 OK**
```json
{
  "userId": 1,
  "email": "kimboho@example.com",
  "nickname": "뭉치보호자",
  "role": "GUARDIAN",
  "address": "서울 마포구",
  "createdAt": "2026-05-12T10:00:00"
}
```

---

### PATCH /api/users/me

**Request**
```json
{
  "nickname": "뭉치집사",
  "currentPassword": "pass1234",
  "password": "newpass5678"
}
```

**Response 200 OK**
```json
{
  "message": "정보가 수정되었습니다."
}
```

**Error Cases**
```json
// 401 — 현재 비밀번호 불일치
{ "status": 401, "code": "INVALID_CURRENT_PASSWORD", "message": "현재 비밀번호가 올바르지 않습니다." }

// 409 — 닉네임 중복
{ "status": 409, "code": "NICKNAME_DUPLICATED", "message": "이미 사용 중인 닉네임입니다." }
```

---

### GET /api/users/me/matchings

**Query Parameters:** `?page=0&size=10&status=CONFIRMED`

**Response 200 OK**
```json
{
  "content": [
    {
      "matchingId": 1,
      "status": "CONFIRMED",
      "postingTitle": "말티즈 3박 4일 케어 구해요",
      "sitterNickname": "박시터",
      "finalPrice": 45000,
      "startDate": "2026-07-01",
      "endDate": "2026-07-04",
      "createdAt": "2026-06-25T15:00:00"
    }
  ],
  "totalPages": 1,
  "totalElements": 1,
  "page": 0,
  "size": 10
}
```

---

## 3. 반려동물 (Pets)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/pets` | 반려동물 등록 | 🔐 (GUARDIAN) | UC-004 |
| GET | `/api/pets` | 내 반려동물 목록 조회 | 🔐 | UC-004 |
| PUT | `/api/pets/{petId}` | 반려동물 정보 수정 | 🔐 | UC-004 |
| DELETE | `/api/pets/{petId}` | 반려동물 삭제 | 🔐 | UC-004 |

### POST /api/pets

**Request**
```json
{
  "name": "뭉치",
  "species": "DOG",
  "breed": "말티즈",
  "age": 3,
  "weight": 3.2,
  "size": "SMALL",
  "neutered": true,
  "specialNotes": "분리불안이 있어요. 낯선 사람에게 짖을 수 있어요."
}
```

**Response 201 Created**
```json
{
  "petId": 1,
  "name": "뭉치",
  "species": "DOG",
  "breed": "말티즈",
  "size": "SMALL"
}
```

---

## 4. 시터 프로필 (Sitter Profile)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| GET | `/api/sitters/{userId}` | 시터 프로필 조회 💾 | 🔓 | UC-006 |
| PUT | `/api/users/me/sitter` | 시터 프로필 수정 | 🔐 (SITTER) | UC-003 |
| POST | `/api/users/me/sitter/ai-generate` | AI 프로필 자동 생성 | 🔐 (SITTER) | UC-015 |

### GET /api/sitters/{userId}

> 💾 Redis 캐싱 TTL 5분. 후기 작성 시 `@CacheEvict`.

**Response 200 OK**
```json
{
  "userId": 10,
  "nickname": "박시터",
  "introduction": "말티즈·포메라니안 3년 케어 경험. 분리불안 있는 아이도 안심하고 맡겨주세요.",
  "specialty": "SMALL_DOG",
  "serviceArea": "서울 마포구",
  "experienceYears": 3,
  "avgRating": 4.9,
  "reviewCount": 98,
  "aiGeneratedBio": {
    "title": "소형견 케어 전문 집사 박시터",
    "description": "...",
    "specialties": ["분리불안 케어", "소형견 전문", "하루 3회 산책"],
    "keywords": ["말티즈", "포메", "집사"]
  },
  "isCertified": true
}
```

---

### POST /api/users/me/sitter/ai-generate

**Request**
```json
{
  "specialty": "SMALL_DOG",
  "experienceYears": 3,
  "breeds": ["말티즈", "포메라니안"],
  "skills": ["분리불안 케어", "하루 3회 산책", "응급처치 가능"]
}
```

**Response 200 OK**
```json
{
  "title": "소형견 케어 전문 집사 박시터",
  "description": "말티즈·포메라니안 3년 케어 경험. 분리불안 있는 아이도 안심하고 맡겨주세요...",
  "specialties": ["분리불안 케어", "소형견 전문", "하루 3회 산책"],
  "keywords": ["말티즈", "포메", "집사", "분리불안"],
  "aiGenerated": true
}
```

**Fallback Response (AI 장애 시)**
```json
{
  "title": "펫시터",
  "description": "경력 있는 펫시터입니다.",
  "specialties": [],
  "keywords": [],
  "aiGenerated": false
}
```

**Error Cases**
```json
// 503 — Circuit Breaker OPEN
{ "status": 503, "code": "AI_SERVICE_UNAVAILABLE", "message": "AI 서비스를 일시적으로 이용할 수 없습니다. 직접 작성해주세요." }
```

---

## 5. 케어 공고 (Care Postings)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/postings` | 케어 공고 등록 | 🔐 (GUARDIAN) | UC-005 |
| GET | `/api/postings` | 공고 목록 검색 (동적 필터) 💾 | 🔓 | UC-006 |
| GET | `/api/postings/{postingId}` | 공고 상세 조회 | 🔓 | UC-006 |
| PATCH | `/api/postings/{postingId}/close` | 공고 취소 | 🔐 (GUARDIAN) | UC-005 |
| GET | `/api/postings/my` | 내 공고 목록 | 🔐 | UC-005 |

### POST /api/postings

**Request**
```json
{
  "petId": 1,
  "title": "말티즈 3박 4일 케어 구해요",
  "description": "분리불안이 있어요. 하루 2회 이상 산책 부탁드려요. 낯선 사람에게 조금 경계해요.",
  "startDate": "2026-07-01",
  "endDate": "2026-07-04",
  "location": "서울 마포구",
  "budget": 60000,
  "proposalDeadline": "2026-06-30T23:59:59"
}
```

**Response 201 Created**
```json
{
  "postingId": 10,
  "title": "말티즈 3박 4일 케어 구해요",
  "status": "OPEN",
  "createdAt": "2026-06-20T10:00:00"
}
```

**Error Cases**
```json
// 400 — 과거 기간
{ "status": 400, "code": "INVALID_CARE_DATE", "message": "케어 기간은 오늘 이후여야 합니다." }

// 400 — 반려동물 없음
{ "status": 400, "code": "PET_NOT_FOUND", "message": "반려동물을 먼저 등록해주세요." }

// 403 — 권한 없음
{ "status": 403, "code": "FORBIDDEN", "message": "보호자만 케어 공고를 등록할 수 있습니다." }
```

---

### GET /api/postings

> 💾 Redis 캐싱 TTL 1분. 신규 공고 등록·취소 시 `@CacheEvict`.

**Query Parameters:** `?location=서울 마포구&petSize=SMALL&startDate=2026-07-01&endDate=2026-07-04&minBudget=0&maxBudget=80000&status=OPEN&page=0&size=20&sort=proposalDeadline,asc`

**Response 200 OK**
```json
{
  "content": [
    {
      "postingId": 10,
      "title": "말티즈 3박 4일 케어 구해요",
      "petName": "뭉치",
      "petSize": "SMALL",
      "petBreed": "말티즈",
      "location": "서울 마포구",
      "budget": 60000,
      "proposalCount": 3,
      "proposalDeadline": "2026-06-30T23:59:59",
      "status": "OPEN",
      "createdAt": "2026-06-20T10:00:00"
    }
  ],
  "totalPages": 3,
  "totalElements": 42,
  "page": 0,
  "size": 20
}
```

---

### GET /api/postings/{postingId}

**Response 200 OK**
```json
{
  "postingId": 10,
  "title": "말티즈 3박 4일 케어 구해요",
  "description": "분리불안이 있어요. 하루 2회 이상 산책 부탁드려요.",
  "pet": {
    "petId": 1,
    "name": "뭉치",
    "species": "DOG",
    "breed": "말티즈",
    "age": 3,
    "size": "SMALL",
    "neutered": true,
    "specialNotes": "분리불안 있음"
  },
  "startDate": "2026-07-01",
  "endDate": "2026-07-04",
  "location": "서울 마포구",
  "budget": 60000,
  "proposalCount": 3,
  "proposalDeadline": "2026-06-30T23:59:59",
  "status": "OPEN",
  "createdAt": "2026-06-20T10:00:00"
}
```

---

## 6. 시터 제안 (Proposals)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/postings/{postingId}/proposals` | 시터 제안 입찰 ⚠️ | 🔐 (SITTER) | UC-007 |
| GET | `/api/postings/{postingId}/proposals` | 제안 목록 조회 | 🔐 (GUARDIAN) | UC-007 |

### POST /api/postings/{postingId}/proposals ⚠️

> **동시성 제어:** Redisson 분산락 (`lock:proposal:{postingId}`, waitTime=3s, leaseTime=10s, Fail Fast)
> 여러 시터가 동시에 제안해도 `proposal_count` 정합성 보장

**Request**
```json
{
  "proposedPrice": 45000,
  "message": "말티즈 케어 경험이 많습니다. 분리불안 전문이에요. 하루 3회 산책 가능합니다.",
  "specialOffer": "케어 일지 하루 2회 전송 드릴게요."
}
```

**Response 201 Created**
```json
{
  "proposalId": 5,
  "postingId": 10,
  "proposedPrice": 45000,
  "status": "PENDING",
  "createdAt": "2026-06-21T09:00:00"
}
```

**Error Cases**
```json
// 409 — 중복 제안
{ "status": 409, "code": "PROPOSAL_DUPLICATED", "message": "이미 이 공고에 제안하셨습니다." }

// 400 — 마감된 공고
{ "status": 400, "code": "POSTING_CLOSED", "message": "이미 마감된 공고입니다." }

// 429 — 분산락 획득 실패
{ "status": 429, "code": "PROPOSAL_LOCK_FAILED", "message": "현재 처리 중입니다. 잠시 후 다시 시도해주세요." }
```

---

### GET /api/postings/{postingId}/proposals

> 공고 작성자(보호자)만 조회 가능. 가격 오름차순 기본 정렬.

**Response 200 OK**
```json
[
  {
    "proposalId": 5,
    "sitter": {
      "userId": 10,
      "nickname": "박시터",
      "avgRating": 4.9,
      "reviewCount": 98,
      "specialty": "소형견 전문",
      "isCertified": true
    },
    "proposedPrice": 45000,
    "message": "말티즈 케어 경험이 많습니다.",
    "specialOffer": "케어 일지 하루 2회 전송 드릴게요.",
    "status": "PENDING",
    "createdAt": "2026-06-21T09:00:00"
  },
  {
    "proposalId": 6,
    "sitter": {
      "userId": 11,
      "nickname": "이시터",
      "avgRating": 4.7,
      "reviewCount": 42,
      "specialty": "소형견 전문",
      "isCertified": false
    },
    "proposedPrice": 40000,
    "message": "홍대 근처 거주, 분리불안 케어 경험 있어요.",
    "specialOffer": null,
    "status": "PENDING",
    "createdAt": "2026-06-21T09:30:00"
  }
]
```

---

## 7. 매칭 확정 (Matchings)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/postings/{postingId}/proposals/{proposalId}/accept` | 제안 수락 (매칭 확정) ⚠️ | 🔐 (GUARDIAN) | UC-008 |
| GET | `/api/matchings` | 내 매칭 목록 조회 | 🔐 | UC-008 |
| GET | `/api/matchings/{matchingId}` | 매칭 상세 조회 | 🔐 | UC-008 |
| PATCH | `/api/matchings/{matchingId}/complete` | 케어 완료 확인 | 🔐 (GUARDIAN) | UC-008 |

### POST /api/postings/{postingId}/proposals/{proposalId}/accept ⚠️

> **동시성 제어:** `posting_id UNIQUE` DB 제약으로 중복 매칭 DB 레벨 차단
> **비동기:** Kafka `matching-confirmed` 이벤트 발행 → 결제·알림·일정 생성 독립 소비

**Response 200 OK**
```json
{
  "matchingId": 1,
  "postingId": 10,
  "sitterNickname": "박시터",
  "finalPrice": 45000,
  "status": "CONFIRMED",
  "createdAt": "2026-06-25T15:00:00"
}
```

**Error Cases**
```json
// 409 — 이미 매칭된 공고 (UNIQUE 제약)
{ "status": 409, "code": "POSTING_ALREADY_MATCHED", "message": "이미 매칭이 확정된 공고입니다." }

// 404 — 제안 없음
{ "status": 404, "code": "PROPOSAL_NOT_FOUND", "message": "존재하지 않는 제안입니다." }
```

---

### GET /api/matchings

**Query Parameters:** `?page=0&size=10&status=CONFIRMED`

**Response 200 OK**
```json
{
  "content": [
    {
      "matchingId": 1,
      "status": "CONFIRMED",
      "postingTitle": "말티즈 3박 4일 케어 구해요",
      "sitterNickname": "박시터",
      "finalPrice": 45000,
      "startDate": "2026-07-01",
      "endDate": "2026-07-04"
    }
  ],
  "totalElements": 1,
  "totalPages": 1
}
```

---

## 8. 케어 일지 (Care Logs)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/matchings/{matchingId}/logs` | 케어 일지 등록 (SSE 푸시) | 🔐 (SITTER) | UC-009 |
| GET | `/api/matchings/{matchingId}/logs` | 케어 일지 목록 조회 | 🔐 | UC-009 |
| GET | `/api/notifications/stream` 📡 | SSE 알림 구독 | 🔐 | UC-009 |

### POST /api/matchings/{matchingId}/logs

> SSE로 보호자에게 실시간 전송. Redis Pub/Sub 경유 (다중 서버 환경 지원).

**Request**
```json
{
  "content": "뭉치 오늘 산책 2회 완료했어요! 밥도 잘 먹었습니다 🐶",
  "imageUrls": [
    "https://cdn.forpets.io/logs/img1.jpg",
    "https://cdn.forpets.io/logs/img2.jpg"
  ]
}
```

**Response 201 Created**
```json
{
  "logId": 1,
  "content": "뭉치 오늘 산책 2회 완료했어요!",
  "imageUrls": ["https://cdn.forpets.io/logs/img1.jpg"],
  "createdAt": "2026-07-01T14:30:00"
}
```

---

### GET /api/notifications/stream 📡

> **SSE 스트리밍 엔드포인트** — 보호자가 구독하면 케어 일지·제안 도착·매칭 확정 등을 실시간으로 수신.
> `Content-Type: text/event-stream`

**Response — SSE 이벤트 형식**
```
data: {"type":"CARE_LOG","matchingId":1,"content":"뭉치 산책 완료!","imageUrls":["..."],"sentAt":"2026-07-01T14:30:00"}

data: {"type":"PROPOSAL_ARRIVED","postingId":10,"sitterNickname":"박시터","proposedPrice":45000}

data: {"type":"MATCHING_CONFIRMED","matchingId":1,"sitterNickname":"박시터","finalPrice":45000}

data: {"type":"RENTAL_OVERDUE","rentalId":3,"itemName":"중형 켄넬","penaltyAmount":4000}
```

---

## 9. 후기·평점 (Reviews)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/reviews` | 후기 작성 | 🔐 (GUARDIAN) | UC-010 |
| GET | `/api/sitters/{userId}/reviews` | 시터 후기 목록 조회 💾 | 🔓 | UC-010 |

### POST /api/reviews

**Request**
```json
{
  "matchingId": 1,
  "rating": 5.0,
  "content": "뭉치를 정말 잘 돌봐주셨어요. 케어 일지도 매일 올려주셔서 안심이 됐어요. 다음에도 꼭 부탁드릴게요!"
}
```

**Response 201 Created**
```json
{
  "reviewId": 1,
  "rating": 5.0,
  "content": "뭉치를 정말 잘 돌봐주셨어요...",
  "createdAt": "2026-07-05T10:00:00"
}
```

**Error Cases**
```json
// 409 — 중복 후기
{ "status": 409, "code": "REVIEW_DUPLICATED", "message": "이미 후기를 작성하셨습니다." }

// 400 — 케어 미완료
{ "status": 400, "code": "MATCHING_NOT_DONE", "message": "케어 완료 후 후기를 작성할 수 있습니다." }
```

---

### GET /api/sitters/{userId}/reviews

> 💾 Redis ZSET 시터 평점 캐싱. 후기 작성 시 ZINCRBY 갱신.

**Response 200 OK**
```json
{
  "sitterId": 10,
  "nickname": "박시터",
  "avgRating": 4.9,
  "reviewCount": 99,
  "reviews": [
    {
      "reviewId": 1,
      "reviewerNickname": "뭉치보호자",
      "rating": 5.0,
      "content": "뭉치를 정말 잘 돌봐주셨어요.",
      "aiSummary": "보호자가 매우 만족하며 케어 일지와 소통을 특히 칭찬함.",
      "createdAt": "2026-07-05T10:00:00"
    }
  ]
}
```

---

## 10. 용품 렌탈 (Rentals)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/rentals/items` | 용품 등록 | 🔐 | — |
| GET | `/api/rentals/items` | 용품 목록 검색 💾 | 🔓 | UC-011 |
| POST | `/api/rentals` | 렌탈 신청 ⚠️ | 🔐 (GUARDIAN) | UC-011 |
| GET | `/api/rentals/my` | 내 렌탈 내역 | 🔐 | UC-012 |
| POST | `/api/rentals/{rentalId}/return` | 반납 처리 | 🔐 (GUARDIAN) | UC-012 |

### POST /api/rentals ⚠️

> **동시성 제어:** Redisson 분산락 (`lock:rental:{itemId}`, waitTime=5s, leaseTime=15s, Retry 3회 Backoff)

**Request**
```json
{
  "itemId": 3,
  "startDate": "2026-07-01",
  "endDate": "2026-07-04"
}
```

**Response 201 Created**
```json
{
  "rentalId": 7,
  "itemName": "중형 켄넬",
  "startDate": "2026-07-01",
  "endDate": "2026-07-04",
  "totalPrice": 24000,
  "status": "ACTIVE"
}
```

**Error Cases**
```json
// 409 — 재고 없음
{ "status": 409, "code": "RENTAL_OUT_OF_STOCK", "message": "재고가 없습니다." }

// 429 — 락 획득 실패 (3회 재시도 후)
{ "status": 429, "code": "RENTAL_LOCK_FAILED", "message": "현재 신청이 많습니다. 잠시 후 다시 시도해주세요." }
```

---

## 11. 임시보호 (TempCare)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/temp-care` | 임시보호 공고 등록 | 🔐👑 (RESCUE) | UC-013 |
| GET | `/api/temp-care` | 임시보호 공고 목록 | 🔓 | UC-014 |
| GET | `/api/temp-care/{id}` | 임시보호 공고 상세 | 🔓 | UC-014 |
| POST | `/api/temp-care/{id}/apply` | 임시보호 지원 | 🔐 (GUARDIAN) | UC-014 |
| POST | `/api/temp-care/{id}/match/{applicantId}` | 임시보호자 선정 | 🔐👑 (RESCUE) | UC-014 |

### POST /api/temp-care

**Request**
```json
{
  "animalName": "솜이",
  "species": "DOG",
  "breed": "포메라니안",
  "age": 2,
  "neutered": true,
  "periodStart": "2026-06-15",
  "periodEnd": "2026-07-15",
  "requirements": "아파트 가능, 선임묘 없는 가정 선호. 산책 하루 2회 필요.",
  "imageUrl": "https://cdn.forpets.io/temp-care/som1.jpg"
}
```

**Response 201 Created**
```json
{
  "tempCareId": 1,
  "animalName": "솜이",
  "status": "OPEN",
  "createdAt": "2026-05-20T10:00:00"
}
```

---

## 12. AI 기능 (AI)

| Method | Endpoint | 설명 | 인증 | 관련 UC |
|--------|----------|------|------|---------|
| POST | `/api/ai/chat` 📡 | AI 시터 추천 챗봇 (Tool Calling + SSE) | 🔐 | UC-016 |
| GET | `/api/sitters/semantic-search` | RAG 시맨틱 시터 탐색 | 🔐 | UC-016 |

### POST /api/ai/chat 📡

> **Tool Calling:** LLM이 `searchSitters()`, `getReviews()`, `checkAvailability()` 순차 호출
> **SSE 스트리밍:** 응답이 글자 단위로 실시간 전송됨
> **멀티턴:** Redis에 세션 히스토리 저장 (최근 10턴 윈도우, TTL 30분)
> `Content-Type: text/event-stream`

**Request**
```json
{
  "message": "관절 안 좋은 7살 진돗개 맡길 시터 마포구에서 찾아줘",
  "sessionId": "session-uuid-1234"
}
```

**Response — SSE 스트리밍**
```
data: {"delta": "관절 케어 경험이 있는 시터 2명을 찾았어요."}
data: {"delta": " \n\n1. 김시터 — "}
data: {"delta": "'14살 진돗개 2주 케어' 후기 있음 (출처: 후기 #87)"}
data: {"delta": "\n2. 오시터 — 수의보조 자격증 보유, 노령견 전문"}
data: [DONE]
```

**Fallback (AI 전체 장애)**
```json
{
  "message": "AI 서비스를 이용할 수 없습니다. 직접 검색 기능을 이용해주세요.",
  "aiGenerated": false
}
```

---

### GET /api/sitters/semantic-search

> **RAG:** 시터 후기·프로필 벡터 탐색. 유사도 임계값 ≥ 0.7 필터.

**Query Parameters:** `?q=분리불안 있는 노령견 경험 시터&location=서울 마포구&page=0&size=10`

**Response 200 OK**
```json
{
  "results": [
    {
      "userId": 10,
      "nickname": "박시터",
      "relevanceScore": 0.94,
      "matchedContext": "분리불안 케어 후기가 23건이에요.",
      "evidenceSources": ["후기 #142", "후기 #98"],
      "avgRating": 4.9,
      "reviewCount": 98
    }
  ],
  "query": "분리불안 있는 노령견 경험 시터"
}
```

---

## 13. WebSocket 채팅 (STOMP) — 도전 기능

> **연결 엔드포인트:** `wss://api.forpets.io/ws-stomp`
> **프로토콜:** STOMP over WebSocket
> **인증:** STOMP CONNECT 프레임의 `Authorization` 헤더에 JWT 포함

| 구분 | 경로 | 설명 |
|------|------|------|
| 연결 | `wss://api.forpets.io/ws-stomp` | WebSocket 핸드셰이크 |
| 구독 (보호자/시터) | `/sub/chat/room/{chatRoomId}` | 해당 채팅방 메시지 수신 |
| 메시지 전송 | `/pub/chat/message` | 메시지 발행 (보호자/시터 동일) |

### 메시지 전송 (/pub/chat/message)
```json
{
  "chatRoomId": 1,
  "content": "산책 몇 시에 나가실 예정인가요?"
}
```

### 메시지 수신 (/sub/chat/room/{chatRoomId})
```json
{
  "chatMessageId": 10,
  "chatRoomId": 1,
  "senderId": 5,
  "senderRole": "GUARDIAN",
  "senderNickname": "뭉치보호자",
  "content": "산책 몇 시에 나가실 예정인가요?",
  "sentAt": "2026-07-01T09:00:00"
}
```

---

## 14. API 전체 요약 인덱스

| # | Method | Endpoint | 설명 | 인증 | 특이사항 |
|---|--------|----------|------|------|----------|
| 1 | POST | `/api/auth/signup` | 회원가입 | 🔓 | 역할 선택 필수 |
| 2 | POST | `/api/auth/login` | 로그인 | 🔓 | |
| 3 | GET | `/api/users/me` | 내 정보 조회 | 🔐 | |
| 4 | PATCH | `/api/users/me` | 내 정보 수정 | 🔐 | |
| 5 | GET | `/api/users/me/matchings` | 내 매칭 목록 | 🔐 | |
| 6 | POST | `/api/pets` | 반려동물 등록 | 🔐 | GUARDIAN |
| 7 | GET | `/api/pets` | 내 반려동물 목록 | 🔐 | |
| 8 | PUT | `/api/pets/{petId}` | 반려동물 수정 | 🔐 | |
| 9 | GET | `/api/sitters/{userId}` | 시터 프로필 조회 | 🔓 | 💾 Redis |
| 10 | PUT | `/api/users/me/sitter` | 시터 프로필 수정 | 🔐 | SITTER |
| 11 | POST | `/api/users/me/sitter/ai-generate` | AI 프로필 자동 생성 | 🔐 | LLM 구조화 출력 |
| 12 | POST | `/api/postings` | 케어 공고 등록 | 🔐 | GUARDIAN |
| 13 | GET | `/api/postings` | 공고 목록 검색 | 🔓 | 💾 Redis, QueryDSL |
| 14 | GET | `/api/postings/{postingId}` | 공고 상세 조회 | 🔓 | |
| 15 | PATCH | `/api/postings/{postingId}/close` | 공고 취소 | 🔐 | GUARDIAN |
| 16 | GET | `/api/postings/my` | 내 공고 목록 | 🔐 | |
| 17 | POST | `/api/postings/{postingId}/proposals` | 시터 제안 입찰 | 🔐 | ⚠️ Redisson |
| 18 | GET | `/api/postings/{postingId}/proposals` | 제안 목록 조회 | 🔐 | GUARDIAN |
| 19 | POST | `…/proposals/{proposalId}/accept` | 제안 수락 (매칭) | 🔐 | ⚠️ UNIQUE 제약 |
| 20 | GET | `/api/matchings` | 내 매칭 목록 | 🔐 | |
| 21 | GET | `/api/matchings/{matchingId}` | 매칭 상세 | 🔐 | |
| 22 | PATCH | `/api/matchings/{matchingId}/complete` | 케어 완료 확인 | 🔐 | GUARDIAN |
| 23 | POST | `/api/matchings/{matchingId}/logs` | 케어 일지 등록 | 🔐 | SITTER, SSE 푸시 |
| 24 | GET | `/api/matchings/{matchingId}/logs` | 케어 일지 목록 | 🔐 | |
| 25 | GET | `/api/notifications/stream` | SSE 알림 구독 | 🔐 | 📡 SSE |
| 26 | POST | `/api/reviews` | 후기 작성 | 🔐 | GUARDIAN |
| 27 | GET | `/api/sitters/{userId}/reviews` | 시터 후기 목록 | 🔓 | 💾 Redis ZSET |
| 28 | POST | `/api/rentals/items` | 렌탈 용품 등록 | 🔐 | |
| 29 | GET | `/api/rentals/items` | 렌탈 용품 목록 | 🔓 | 💾 Redis |
| 30 | POST | `/api/rentals` | 렌탈 신청 | 🔐 | ⚠️ Redisson |
| 31 | GET | `/api/rentals/my` | 내 렌탈 내역 | 🔐 | |
| 32 | POST | `/api/rentals/{rentalId}/return` | 반납 처리 | 🔐 | |
| 33 | POST | `/api/temp-care` | 임시보호 공고 등록 | 🔐👑 | RESCUE |
| 34 | GET | `/api/temp-care` | 임시보호 공고 목록 | 🔓 | |
| 35 | POST | `/api/temp-care/{id}/apply` | 임시보호 지원 | 🔐 | GUARDIAN |
| 36 | POST | `/api/temp-care/{id}/match/{applicantId}` | 임시보호자 선정 | 🔐👑 | RESCUE, Kafka |
| 37 | POST | `/api/ai/chat` | AI 챗봇 (Tool Calling) | 🔐 | 📡 SSE 스트리밍 |
| 38 | GET | `/api/sitters/semantic-search` | RAG 시맨틱 탐색 | 🔐 | 도전 기능 |
| WS | STOMP | `wss://.../ws-stomp` | 1:1 채팅 | 🔐 JWT | 도전 기능 |

---

## 15. 주요 설계 결정 요약

### 동시성 민감 API ⚠️ 설계 원칙

| API | 제어 방식 | 실패 전략 | 이유 |
|-----|-----------|-----------|------|
| 시터 제안 입찰 | Redisson 분산락 (waitTime=3s) | Fail Fast (즉시 429) | 제안은 "빠른 응답"이 UX에 중요. 대기보다 즉시 안내가 유리 |
| 렌탈 신청 재고 차감 | Redisson 분산락 (waitTime=5s) | Retry 3회 Backoff | "받느냐 못 받느냐"가 UX 핵심. 잠깐 기다려서라도 받는 게 유리 |
| 매칭 확정 | DB UNIQUE 제약 (posting_id) | 409 POSTING_ALREADY_MATCHED | 1공고 1매칭 불변 원칙. DB 레벨 최종 방어선 |

### AI 장애 격리 원칙

AI 기능(프로필 생성, 챗봇, RAG) 장애 시 핵심 서비스(공고·매칭·렌탈)는 **정상 동작 보장**.
- 타임아웃(5초) → Fallback 응답 반환
- Rate Limit(429) → Circuit Breaker 오픈 → 안내 메시지
- 핵심 서비스 API는 AI 모듈과 완전 독립

### 페이징 전략

| 대상 | 페이징 방식 | 이유 |
|------|-----------|------|
| 공고 목록·시터 목록 | offset 기반 (`page`, `size`) | 총 페이지 표시가 필요한 탐색형 UI |
| 채팅 메시지 | cursor 기반 (`cursor`, `size`) | 무한 스크롤, 실시간 추가 환경에서 offset은 데이터 누락 발생 |
| 내 매칭·렌탈 내역 | offset 기반 | 사용자가 직접 페이지 선택하는 패턴 |

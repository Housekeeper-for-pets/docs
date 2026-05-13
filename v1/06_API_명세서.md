# 📡 API 명세서 — ForPets (포펫츠)

> **프로젝트명:** ForPets (포펫츠)
> **팀명:** 집사조
> **문서 버전:** v1.2
> **최종 수정일:** 2026-05-13
> **연결 문서:** 01_프로젝트_개요서, 05_ERD

---

## 문서 읽는 법

| 아이콘 | 의미 |
|--------|------|
| 🔐 | JWT 인증 필수 (`Authorization: Bearer {token}`) |
| 🔐👑 | JWT 인증 + ADMIN 권한 필수 |
| 💾 | 캐싱 대상 API (Redis) |
| 🔓 | 인증 불필요 (공개 API) |
| 📡 | SSE (Server-Sent Events) 스트리밍 |

**Base URL:** `https://api.forpets.io`

**공통 에러 응답 형식:**
```json
{
  "status": 409,
  "code": "PROPOSAL_DUPLICATED",
  "message": "이미 이 공고에 제안하셨습니다.",
  "timestamp": "2026-05-13T10:00:00"
}
```

**서버 타임존:** 모든 `datetime` 필드는 **KST(UTC+9)** 기준으로 저장하고 반환한다.

---

## 1. 인증 (Auth)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/auth/signup` | 회원가입 (역할 선택 포함) | 🔓 |
| POST | `/api/auth/login` | 로그인 (JWT 발급) | 🔓 |
| POST | `/api/auth/logout` | 로그아웃 (JWT 블랙리스트) | 🔐 |
| POST | `/api/auth/reissue` | 토큰 재발급 | 🔐 |

### POST /api/auth/signup

**Request**
```json
{
  "email": "kimboho@example.com",
  "password": "pass1234",
  "nickname": "뭉치보호자",
  "role": "MEMBER"
}
```
> `role`: `MEMBER` (일반 회원) / `SITTER` (시터) / `ADMIN` (관리자)
> SITTER로 가입 시 시터 프로필 등록(`POST /api/sitters`)을 추가로 완료해야 시터 기능 이용 가능.

**Response 201 Created**
```json
{
  "userId": 1,
  "email": "kimboho@example.com",
  "nickname": "뭉치보호자",
  "role": "MEMBER",
  "createdAt": "2026-05-13T10:00:00"
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
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
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

### POST /api/auth/logout

> JWT 블랙리스트 방식. Redis에 해당 토큰을 블랙리스트로 등록하여 만료 전 재사용을 차단한다.

**Response 200 OK**
```json
{
  "message": "로그아웃 되었습니다."
}
```

---

### POST /api/auth/reissue

> Refresh Token으로 새로운 Access Token을 발급받는다.

**Request**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
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
// 401 — 유효하지 않은 Refresh Token
{ "status": 401, "code": "INVALID_REFRESH_TOKEN", "message": "유효하지 않은 토큰입니다. 다시 로그인해주세요." }
```

---

## 2. 회원 (Members)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/members/me` | 내 정보 조회 | 🔐 |
| PUT | `/api/members/me` | 내 정보 수정 | 🔐 |
| PATCH | `/api/members/me/password` | 비밀번호 변경 | 🔐 |
| POST | `/api/members/me/profile-image` | 프로필 이미지 업로드 (S3) | 🔐 |
| DELETE | `/api/members/me` | 회원 탈퇴 | 🔐 |

### GET /api/members/me

**Response 200 OK**
```json
{
  "userId": 1,
  "email": "kimboho@example.com",
  "nickname": "뭉치보호자",
  "phone": "010-1234-5678",
  "gender": "FEMALE",
  "role": "MEMBER",
  "status": "ACTIVE",
  "createdAt": "2026-05-13T10:00:00"
}
```

---

### PUT /api/members/me

**Request**
```json
{
  "nickname": "뭉치집사",
  "phone": "010-9876-5432",
  "gender": "FEMALE"
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
// 409 — 닉네임 중복
{ "status": 409, "code": "NICKNAME_DUPLICATED", "message": "이미 사용 중인 닉네임입니다." }
```

---

### PATCH /api/members/me/password

**Request**
```json
{
  "currentPassword": "pass1234",
  "newPassword": "newpass5678"
}
```

**Response 200 OK**
```json
{
  "message": "비밀번호가 변경되었습니다."
}
```

**Error Cases**
```json
// 401 — 현재 비밀번호 불일치
{ "status": 401, "code": "INVALID_CURRENT_PASSWORD", "message": "현재 비밀번호가 올바르지 않습니다." }
```

---

### POST /api/members/me/profile-image

> `Content-Type: multipart/form-data`. S3 업로드 후 URL 반환.

**Request:** `file` (MultipartFile)

**Response 200 OK**
```json
{
  "imageUrl": "https://cdn.forpets.io/profiles/user-1-uuid.jpg"
}
```

---

### DELETE /api/members/me

> 소프트 삭제. 회원 상태를 `DELETED`로 변경한다.

**Response 200 OK**
```json
{
  "message": "회원 탈퇴가 완료되었습니다."
}
```

---

## 3. 반려동물 (Pets)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/pets` | 내 반려동물 목록 조회 | 🔐 |
| POST | `/api/pets` | 반려동물 등록 | 🔐 |
| GET | `/api/pets/{petId}` | 반려동물 상세 조회 | 🔐 |
| PUT | `/api/pets/{petId}` | 반려동물 정보 수정 | 🔐 |
| DELETE | `/api/pets/{petId}` | 반려동물 삭제 | 🔐 |

### POST /api/pets

**Request**
```json
{
  "name": "뭉치",
  "species": "DOG",
  "breed": "말티즈",
  "size": "SMALL",
  "age": 3,
  "gender": "MALE",
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
  "size": "SMALL",
  "createdAt": "2026-05-13T10:00:00"
}
```

---

### GET /api/pets/{petId}

**Response 200 OK**
```json
{
  "petId": 1,
  "name": "뭉치",
  "species": "DOG",
  "breed": "말티즈",
  "size": "SMALL",
  "age": 3,
  "gender": "MALE",
  "profileImageUrl": "https://cdn.forpets.io/pets/pet-1.jpg",
  "specialNotes": "분리불안이 있어요.",
  "createdAt": "2026-05-13T10:00:00"
}
```

---

## 4. 시터 (Sitters)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/sitters` | 시터 프로필 등록 (MEMBER→SITTER 역할 전환) | 🔐 |
| GET | `/api/sitters/me` | 내 시터 프로필 조회 | 🔐 (SITTER) |
| PUT | `/api/sitters/me` | 시터 프로필 수정 | 🔐 (SITTER) |
| PATCH | `/api/sitters/me/availability` | 시터 매칭 상태 변경 | 🔐 (SITTER) |
| POST | `/api/sitters/me/profile-image` | 시터 프로필 이미지 업로드 (S3) | 🔐 (SITTER) |
| GET | `/api/sitters` | 시터 검색 (동적 필터) 💾 | 🔓 |
| GET | `/api/sitters/{sitterId}` | 시터 상세 조회 💾 | 🔓 |
| DELETE | `/api/sitters/me` | 시터 탈퇴 (SITTER→MEMBER 역할 전환) | 🔐 (SITTER) |

### POST /api/sitters

> 시터 프로필 등록 시 회원 역할이 `MEMBER → SITTER`로 전환된다.
> SITTER는 MEMBER의 모든 기능(공고 등록, 시터 검색·신청, 후기 작성)을 포함한다.

**Request**
```json
{
  "region": "서울 마포구",
  "introduction": "말티즈·포메라니안 3년 케어 경험. 분리불안 있는 아이도 안심하고 맡겨주세요.",
  "experienceYears": 3,
  "availablePetType": "DOG",
  "availablePetSize": "SMALL",
  "hourlyRate": 15000
}
```

**Response 201 Created**
```json
{
  "sitterProfileId": 1,
  "region": "서울 마포구",
  "availablePetType": "DOG",
  "hourlyRate": 15000,
  "status": "ACTIVE",
  "role": "SITTER",
  "createdAt": "2026-05-13T10:00:00"
}
```

**Error Cases**
```json
// 409 — 이미 시터 등록됨
{ "status": 409, "code": "SITTER_ALREADY_EXISTS", "message": "이미 시터 프로필이 등록되어 있습니다." }
```

---

### GET /api/sitters (검색)

> 💾 Redis 캐싱 TTL 1시간. QueryDSL 동적 필터.

**Query Parameters:** `?region=서울 마포구&availablePetType=DOG&availablePetSize=SMALL&minRate=10000&maxRate=20000&page=0&size=20&sort=avgRating,desc`

**Response 200 OK**
```json
{
  "content": [
    {
      "sitterProfileId": 1,
      "nickname": "박시터",
      "region": "서울 마포구",
      "availablePetType": "DOG",
      "availablePetSize": "SMALL",
      "experienceYears": 3,
      "hourlyRate": 15000,
      "avgRating": 4.9,
      "reviewCount": 98,
      "status": "ACTIVE"
    }
  ],
  "totalPages": 3,
  "totalElements": 42,
  "page": 0,
  "size": 20
}
```

---

### GET /api/sitters/{sitterId}

> 💾 Redis 캐싱 TTL 1시간. 후기 작성 시 `@CacheEvict`.

**Response 200 OK**
```json
{
  "sitterProfileId": 1,
  "userId": 10,
  "nickname": "박시터",
  "region": "서울 마포구",
  "introduction": "말티즈·포메라니안 3년 케어 경험.",
  "experienceYears": 3,
  "availablePetType": "DOG",
  "availablePetSize": "SMALL",
  "hourlyRate": 15000,
  "avgRating": 4.9,
  "reviewCount": 98,
  "status": "ACTIVE"
}
```

---

### PATCH /api/sitters/me/availability

> 시터 프로필 상태를 ACTIVE/INACTIVE로 전환한다.

**Request**
```json
{
  "status": "INACTIVE"
}
```

**Response 200 OK**
```json
{
  "message": "매칭 상태가 변경되었습니다.",
  "status": "INACTIVE"
}
```

---

### DELETE /api/sitters/me

> 시터 프로필을 삭제(소프트 삭제)하고 회원 역할이 `SITTER → MEMBER`로 전환된다.

**Response 200 OK**
```json
{
  "message": "시터 탈퇴가 완료되었습니다.",
  "role": "MEMBER"
}
```

---

## 5. 게시물 (Posts) — 케어 공고

> v1에서 게시물은 보호자의 케어 공고(`OWNER_REQUEST`) 용도로만 사용한다.
> `post_type` 필드는 확장을 위해 enum으로 유지하되, v1에서는 `OWNER_REQUEST` 값만 허용한다.
> 케어 공고에는 반드시 케어 시간을 명시해야 하며, 시터는 해당 시간 전체에 가능해야만 제안할 수 있다.

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/posts` | 케어 공고 작성 | 🔐 |
| GET | `/api/posts` | 케어 공고 목록 (전체/홈) 💾 | 🔓 |
| GET | `/api/posts/me` | 내 케어 공고 목록 | 🔐 |
| GET | `/api/posts/{postId}` | 케어 공고 상세 조회 | 🔓 |
| PUT | `/api/posts/{postId}` | 케어 공고 수정 | 🔐 |
| DELETE | `/api/posts/{postId}` | 케어 공고 삭제 | 🔐 |
| PATCH | `/api/posts/{postId}/status` | 케어 공고 상태 변경 | 🔐 |

### POST /api/posts

**Request**
```json
{
  "petId": 1,
  "title": "말티즈 3박 4일 케어 구해요",
  "content": "분리불안이 있어요. 하루 2회 이상 산책 부탁드려요.",
  "postType": "OWNER_REQUEST",
  "category": "CARE",
  "region": "서울 마포구",
  "careStartDate": "2026-07-01",
  "careEndDate": "2026-07-04",
  "careStartTime": "09:00",
  "careEndTime": "18:00"
}
```
> `postType`: v1에서는 `OWNER_REQUEST` (보호자 케어 공고)만 허용
> `category`: `CARE` / `WALK` / `TEMP_CARE` / `ETC`

**Response 201 Created**
```json
{
  "postId": 10,
  "title": "말티즈 3박 4일 케어 구해요",
  "postType": "OWNER_REQUEST",
  "careStartDate": "2026-07-01",
  "careEndDate": "2026-07-04",
  "careStartTime": "09:00",
  "careEndTime": "18:00",
  "status": "ACTIVE",
  "createdAt": "2026-06-20T10:00:00"
}
```

**Error Cases**
```json
// 400 — 반려동물 없음
{ "status": 400, "code": "PET_NOT_FOUND", "message": "반려동물을 먼저 등록해주세요." }

// 400 — 과거 기간
{ "status": 400, "code": "INVALID_CARE_DATE", "message": "케어 기간은 오늘 이후여야 합니다." }
```

---

### GET /api/posts

> 💾 Redis 캐싱 TTL 5분. 신규 등록·삭제 시 `@CacheEvict`.

**Query Parameters:** `?category=CARE&region=서울 마포구&status=ACTIVE&page=0&size=20&sort=createdAt,desc`

**Response 200 OK**
```json
{
  "content": [
    {
      "postId": 10,
      "title": "말티즈 3박 4일 케어 구해요",
      "postType": "OWNER_REQUEST",
      "category": "CARE",
      "region": "서울 마포구",
      "careStartDate": "2026-07-01",
      "careEndDate": "2026-07-04",
      "careStartTime": "09:00",
      "careEndTime": "18:00",
      "viewCount": 42,
      "status": "ACTIVE",
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

### GET /api/posts/{postId}

**Response 200 OK**
```json
{
  "postId": 10,
  "author": {
    "userId": 1,
    "nickname": "뭉치보호자"
  },
  "pet": {
    "petId": 1,
    "name": "뭉치",
    "species": "DOG",
    "breed": "말티즈",
    "size": "SMALL",
    "age": 3,
    "specialNotes": "분리불안 있음"
  },
  "title": "말티즈 3박 4일 케어 구해요",
  "content": "분리불안이 있어요. 하루 2회 이상 산책 부탁드려요.",
  "postType": "OWNER_REQUEST",
  "category": "CARE",
  "region": "서울 마포구",
  "careStartDate": "2026-07-01",
  "careEndDate": "2026-07-04",
  "careStartTime": "09:00",
  "careEndTime": "18:00",
  "viewCount": 42,
  "status": "ACTIVE",
  "createdAt": "2026-06-20T10:00:00"
}
```

---

### PATCH /api/posts/{postId}/status

**Request**
```json
{
  "status": "CLOSED"
}
```

**Response 200 OK**
```json
{
  "message": "게시글 상태가 변경되었습니다.",
  "status": "CLOSED"
}
```

---

## 6. 시터 역제안 (Proposals)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/posts/{postId}/proposals` | 제안 보내기 (시터 → Post) | 🔐 (SITTER) |
| GET | `/api/posts/{postId}/proposals` | 특정 Post 제안 목록 조회 | 🔐 |
| GET | `/api/proposals/{proposalId}` | 제안 상세 조회 | 🔐 |
| PATCH | `/api/proposals/{proposalId}/accept` | 제안 수락 → 예약 생성 | 🔐 |
| PATCH | `/api/proposals/{proposalId}/reject` | 제안 거절 | 🔐 |

### POST /api/posts/{postId}/proposals

> **동시성 제어:** `(post_id, sitter_profile_id)` DB Unique 제약으로 동일 시터 중복 제안 차단.
> **v1 시간 제약:** 시터는 POST에 명시된 케어 시간 전체에 가능해야 제안할 수 있다. 시스템이 시터의 `SITTER_SCHEDULE`과 POST의 시간을 비교하여 전체 커버 가능 여부를 검증한다.

**Request**
```json
{
  "proposedPrice": 45000,
  "availableDate": "2026-07-01",
  "availableStartTime": "09:00",
  "availableEndTime": "18:00",
  "message": "말티즈 케어 경험이 많습니다. 분리불안 전문이에요."
}
```

**Response 201 Created**
```json
{
  "proposalId": 5,
  "postId": 10,
  "proposedPrice": 45000,
  "status": "PENDING",
  "createdAt": "2026-06-21T09:00:00"
}
```

**Error Cases**
```json
// 409 — 중복 제안 (DB Unique 제약)
{ "status": 409, "code": "PROPOSAL_DUPLICATED", "message": "이미 이 공고에 제안하셨습니다." }

// 400 — 마감된 공고
{ "status": 400, "code": "POST_CLOSED", "message": "이미 마감된 공고입니다." }

// 400 — 스케줄 불일치
{ "status": 400, "code": "SCHEDULE_NOT_AVAILABLE", "message": "해당 시간대에 등록된 가능 일정이 없습니다. 스케줄을 먼저 등록해주세요." }
```

---

### GET /api/posts/{postId}/proposals

> 공고 작성자(보호자)만 조회 가능. 가격 오름차순 기본 정렬.

**Response 200 OK**
```json
[
  {
    "proposalId": 5,
    "sitter": {
      "sitterProfileId": 1,
      "nickname": "박시터",
      "avgRating": 4.9,
      "reviewCount": 98,
      "region": "서울 마포구"
    },
    "proposedPrice": 45000,
    "availableDate": "2026-07-01",
    "availableStartTime": "09:00",
    "availableEndTime": "18:00",
    "message": "말티즈 케어 경험이 많습니다.",
    "status": "PENDING",
    "createdAt": "2026-06-21T09:00:00"
  }
]
```

---

### PATCH /api/proposals/{proposalId}/accept

> 제안 수락 시 예약(RESERVATION)을 생성하고, 해당 게시글 상태를 `CLOSED`로 변경한다.
> Kafka `matching-confirmed` 이벤트 발행 → 결제·알림 비동기 처리.
> **스케줄 충돌 처리:** 수락된 시터의 다른 PENDING 제안 중 시간이 겹치는 것을 자동 WITHDRAWN 처리하고 SSE 알림을 전송한다.
> 알림 예시: "보호자님이 해당 시간대에 다른 시터와 예약을 확정했습니다. 시간을 수정하여 다시 제안하시겠습니까?"

**Response 200 OK**
```json
{
  "reservationId": 1,
  "proposalId": 5,
  "sitterNickname": "박시터",
  "price": 45000,
  "reservationDate": "2026-07-01",
  "startTime": "09:00",
  "endTime": "18:00",
  "status": "PAYMENT_PENDING",
  "createdAt": "2026-06-25T15:00:00"
}
```

**Error Cases**
```json
// 409 — 이미 수락된 제안이 있는 공고
{ "status": 409, "code": "POST_ALREADY_MATCHED", "message": "이미 매칭이 확정된 공고입니다." }

// 404 — 제안 없음
{ "status": 404, "code": "PROPOSAL_NOT_FOUND", "message": "존재하지 않는 제안입니다." }
```

---

### PATCH /api/proposals/{proposalId}/reject

**Response 200 OK**
```json
{
  "message": "제안을 거절하였습니다.",
  "proposalId": 5,
  "status": "REJECTED"
}
```

---

## 7. 돌봄 요청 (Care Requests) — 순방향 매칭

> v1에서는 시간 조율 없이, 보호자가 명시한 시간 전체에 가능한 시터만 수락할 수 있다.

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/sitters/{sitterId}/requests` | 돌봄 요청 (보호자 → 시터) | 🔐 |
| GET | `/api/requests/received` | 받은 요청 목록 (시터용) | 🔐 (SITTER) |
| GET | `/api/requests/received/{requestId}` | 받은 요청 상세 조회 | 🔐 (SITTER) |
| PATCH | `/api/requests/{requestId}/accept` | 받은 요청 수락 → 예약 생성 | 🔐 (SITTER) |
| PATCH | `/api/requests/{requestId}/reject` | 받은 요청 거절 | 🔐 (SITTER) |
| GET | `/api/requests/sent` | 보낸 요청 목록 (보호자용) | 🔐 |
| GET | `/api/requests/sent/{requestId}` | 보낸 요청 상세 조회 | 🔐 |
| PATCH | `/api/requests/sent/{requestId}/cancel` | 보낸 요청 취소 | 🔐 |

### POST /api/sitters/{sitterId}/requests

**Request**
```json
{
  "petId": 1,
  "desiredDate": "2026-07-01",
  "desiredStartTime": "09:00",
  "desiredEndTime": "18:00",
  "careType": "VISIT",
  "message": "뭉치 돌봄 부탁드려요. 분리불안이 있어서 경험 있으신 분이면 좋겠어요."
}
```
> `careType`: `VISIT` (방문 돌봄) / `BOARDING` (위탁 돌봄)

**Response 201 Created**
```json
{
  "requestId": 1,
  "sitterProfileId": 1,
  "petName": "뭉치",
  "desiredDate": "2026-07-01",
  "desiredStartTime": "09:00",
  "desiredEndTime": "18:00",
  "careType": "VISIT",
  "status": "PENDING",
  "createdAt": "2026-06-20T10:00:00"
}
```

**Error Cases**
```json
// 400 — 시터 비활성
{ "status": 400, "code": "SITTER_NOT_AVAILABLE", "message": "현재 매칭을 받지 않는 시터입니다." }
```

---

### GET /api/requests/received

**Query Parameters:** `?status=PENDING&page=0&size=10`

**Response 200 OK**
```json
{
  "content": [
    {
      "requestId": 1,
      "owner": {
        "userId": 1,
        "nickname": "뭉치보호자"
      },
      "pet": {
        "petId": 1,
        "name": "뭉치",
        "species": "DOG",
        "breed": "말티즈",
        "size": "SMALL"
      },
      "desiredDate": "2026-07-01",
      "desiredStartTime": "09:00",
      "desiredEndTime": "18:00",
      "careType": "VISIT",
      "status": "PENDING",
      "createdAt": "2026-06-20T10:00:00"
    }
  ],
  "totalPages": 1,
  "totalElements": 1,
  "page": 0,
  "size": 10
}
```

---

### PATCH /api/requests/{requestId}/accept

> 요청 수락 시 예약(RESERVATION)을 생성한다. 시터는 요청에 명시된 시간 전체에 가능해야 수락할 수 있다.
> Kafka `matching-confirmed` 이벤트 발행 → 결제·알림 비동기 처리.
> **스케줄 충돌 처리:** 수락된 시터의 다른 PENDING 제안/요청 중 시간이 겹치는 것을 자동 WITHDRAWN/CANCELLED 처리하고 SSE 알림을 전송한다.

**Response 200 OK**
```json
{
  "reservationId": 2,
  "requestId": 1,
  "ownerNickname": "뭉치보호자",
  "petName": "뭉치",
  "reservationDate": "2026-07-01",
  "startTime": "09:00",
  "endTime": "18:00",
  "status": "PAYMENT_PENDING",
  "createdAt": "2026-06-21T09:00:00"
}
```

**Error Cases**
```json
// 400 — 스케줄 불일치
{ "status": 400, "code": "SCHEDULE_NOT_AVAILABLE", "message": "해당 시간대에 가능한 일정이 없습니다." }
```

---

## 8. 예약 (Reservations)

> CANCELED는 `PAYMENT_PENDING` 또는 `CONFIRMED` 상태에서만 가능하다. `IN_PROGRESS` 이후에는 취소할 수 없다.

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/reservations` | 예약 생성 (제안/요청 수락 시 내부 호출) | 🔐 |
| GET | `/api/reservations/{reservationId}` | 예약 상세 조회 | 🔐 |
| GET | `/api/reservations/me` | 내 예약 목록 조회 | 🔐 |
| PATCH | `/api/reservations/{reservationId}/cancel` | 예약 취소 (PAYMENT_PENDING, CONFIRMED만 가능) | 🔐 |

### GET /api/reservations/{reservationId}

**Response 200 OK**
```json
{
  "reservationId": 1,
  "owner": {
    "userId": 1,
    "nickname": "뭉치보호자"
  },
  "sitter": {
    "sitterProfileId": 1,
    "nickname": "박시터",
    "avgRating": 4.9
  },
  "pet": {
    "petId": 1,
    "name": "뭉치",
    "species": "DOG",
    "breed": "말티즈"
  },
  "reservationDate": "2026-07-01",
  "startTime": "09:00",
  "endTime": "18:00",
  "price": 45000,
  "status": "CONFIRMED",
  "requestMemo": "분리불안이 있어요.",
  "createdAt": "2026-06-25T15:00:00"
}
```

---

### GET /api/reservations/me

**Query Parameters:** `?status=CONFIRMED&page=0&size=10`

**Response 200 OK**
```json
{
  "content": [
    {
      "reservationId": 1,
      "sitterNickname": "박시터",
      "petName": "뭉치",
      "reservationDate": "2026-07-01",
      "startTime": "09:00",
      "endTime": "18:00",
      "price": 45000,
      "status": "CONFIRMED",
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

### PATCH /api/reservations/{reservationId}/cancel

> `PAYMENT_PENDING` 또는 `CONFIRMED` 상태에서만 취소 가능. `IN_PROGRESS` 이후에는 취소할 수 없다.

**Response 200 OK**
```json
{
  "message": "예약이 취소되었습니다.",
  "reservationId": 1,
  "status": "CANCELED"
}
```

**Error Cases**
```json
// 400 — 취소 불가 상태 (IN_PROGRESS 이후)
{ "status": 400, "code": "RESERVATION_NOT_CANCELABLE", "message": "케어 진행 중이거나 완료된 예약은 취소할 수 없습니다." }
```

---

## 9. 결제 (Payments)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/payments` | 결제 요청 (Portone 연동) | 🔐 |
| POST | `/api/payments/confirm` | 결제 확인 (콜백) | 🔓 |
| GET | `/api/payments/{paymentId}` | 결제 내역 조회 | 🔐 |

### POST /api/payments

> 서버에서 `merchantOrderId`(고유 결제번호)를 생성하고 Portone 결제 준비 정보를 반환한다.

**Request**
```json
{
  "reservationId": 1
}
```

**Response 200 OK**
```json
{
  "paymentId": 1,
  "merchantOrderId": "FORPETS-20260701-UUID",
  "amount": 45000,
  "method": "PORTONE_CARD",
  "status": "READY"
}
```

**Error Cases**
```json
// 400 — 이미 결제 완료
{ "status": 400, "code": "ALREADY_PAID", "message": "이미 결제가 완료된 예약입니다." }
```

---

### POST /api/payments/confirm

> Portone 결제 완료 콜백. 결제 검증 후 예약 상태를 `CONFIRMED`로 변경한다.

**Request**
```json
{
  "merchantOrderId": "FORPETS-20260701-UUID",
  "portonePaymentId": "pay_abc123",
  "portoneTransactionId": "txn_xyz789"
}
```

**Response 200 OK**
```json
{
  "paymentId": 1,
  "status": "PAID",
  "paidAt": "2026-06-25T15:05:00",
  "reservationStatus": "CONFIRMED"
}
```

**Error Cases**
```json
// 400 — 결제 검증 실패
{ "status": 400, "code": "PAYMENT_VERIFICATION_FAILED", "message": "결제 정보가 일치하지 않습니다." }
```

---

### GET /api/payments/{paymentId}

**Response 200 OK**
```json
{
  "paymentId": 1,
  "reservationId": 1,
  "merchantOrderId": "FORPETS-20260701-UUID",
  "amount": 45000,
  "method": "PORTONE_CARD",
  "status": "PAID",
  "portonePaymentId": "pay_abc123",
  "paidAt": "2026-06-25T15:05:00"
}
```

---

## 10. 후기·평점 (Reviews)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/reviews` | 후기 작성 (양방향: 보호자↔시터) | 🔐 |
| GET | `/api/sitters/{sitterId}/reviews` | 시터 후기 목록 조회 💾 | 🔓 |
| GET | `/api/reviews/me` | 내가 작성한 후기 목록 | 🔐 |
| PUT | `/api/reviews/{reviewId}` | 후기 수정 | 🔐 |
| DELETE | `/api/reviews/{reviewId}` | 후기 삭제 | 🔐 |

### POST /api/reviews

> 예약 상태가 `COMPLETED`인 경우에만 작성 가능. 보호자와 시터 모두 상대방에 대해 1건씩 작성할 수 있다.
> `(reservation_id, reviewer_id)` DB Unique 제약으로 중복 작성 방지.

**Request**
```json
{
  "reservationId": 1,
  "rating": 5,
  "content": "뭉치를 정말 잘 돌봐주셨어요. 케어 일지도 매일 올려주셔서 안심이 됐어요."
}
```

**Response 201 Created**
```json
{
  "reviewId": 1,
  "reservationId": 1,
  "reviewerId": 1,
  "revieweeId": 10,
  "rating": 5,
  "content": "뭉치를 정말 잘 돌봐주셨어요...",
  "createdAt": "2026-07-05T10:00:00"
}
```

**Error Cases**
```json
// 409 — 중복 후기 (DB Unique 제약)
{ "status": 409, "code": "REVIEW_DUPLICATED", "message": "이미 해당 예약에 후기를 작성하셨습니다." }

// 400 — 케어 미완료
{ "status": 400, "code": "RESERVATION_NOT_COMPLETED", "message": "케어 완료 후 후기를 작성할 수 있습니다." }
```

---

### GET /api/sitters/{sitterId}/reviews

> 💾 Redis ZSET 시터 평점 캐싱. 후기 작성 시 ZINCRBY 갱신.

**Query Parameters:** `?page=0&size=10`

**Response 200 OK**
```json
{
  "sitterProfileId": 1,
  "nickname": "박시터",
  "avgRating": 4.9,
  "reviewCount": 99,
  "reviews": [
    {
      "reviewId": 1,
      "reviewerNickname": "뭉치보호자",
      "rating": 5,
      "content": "뭉치를 정말 잘 돌봐주셨어요.",
      "createdAt": "2026-07-05T10:00:00"
    }
  ],
  "totalPages": 10,
  "totalElements": 99,
  "page": 0,
  "size": 10
}
```

---

## 11. 알림 (Notifications)

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/notifications` | 알림 목록 조회 | 🔐 |
| PATCH | `/api/notifications/{notificationId}/read` | 알림 읽음 처리 | 🔐 |
| PATCH | `/api/notifications/read-all` | 알림 전체 읽음 처리 | 🔐 |
| GET | `/api/notifications/stream` 📡 | SSE 알림 구독 | 🔐 |

### GET /api/notifications/stream 📡

> **SSE 스트리밍 엔드포인트** — 제안 도착·매칭 확정·스케줄 충돌 등을 실시간으로 수신.
> `Content-Type: text/event-stream`

**Response — SSE 이벤트 형식**
```
data: {"type":"PROPOSAL_ARRIVED","postId":10,"sitterNickname":"박시터","proposedPrice":45000}

data: {"type":"MATCHING_CONFIRMED","reservationId":1,"sitterNickname":"박시터","price":45000}

data: {"type":"REQUEST_RECEIVED","requestId":1,"ownerNickname":"뭉치보호자","petName":"뭉치"}

data: {"type":"PROPOSAL_WITHDRAWN","proposalId":5,"postId":12,"reason":"보호자님이 해당 시간대에 다른 시터와 예약을 확정했습니다. 시간을 수정하여 다시 제안하시겠습니까?"}

data: {"type":"PAYMENT_COMPLETED","reservationId":1,"amount":45000}
```

---

### GET /api/notifications

**Query Parameters:** `?page=0&size=20`

**Response 200 OK**
```json
{
  "content": [
    {
      "notificationId": 1,
      "type": "PROPOSAL_ARRIVED",
      "message": "박시터님이 45,000원에 제안을 보냈습니다.",
      "isRead": false,
      "createdAt": "2026-06-21T09:00:00"
    }
  ],
  "totalPages": 1,
  "totalElements": 1,
  "page": 0,
  "size": 20
}
```

---

## 12. 확장 기능 API 요약

> 아래 API는 v1 MVP 이후 확장 기능으로 구현 예정이며, 상세 명세는 확장 개발 시 추가한다.

### 채팅 (Chat) — 시간 조율 지원

> v2에서 채팅이 도입되면 보호자-시터 간 케어 시간 조율이 가능해진다.

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| GET | `/api/chats` | 채팅방 목록 조회 | 🔐 |
| POST | `/api/chats` | 채팅방 생성 (1:1) | 🔐 |
| GET | `/api/chats/{chatRoomId}/messages` | 채팅방 메시지 목록 조회 | 🔐 |
| POST | `/api/chats/{chatRoomId}/messages` | 메시지 전송 | 🔐 |
| PATCH | `/api/chats/{chatRoomId}/read` | 채팅방 읽음 처리 | 🔐 |
| GET | `/api/chats/unread-count` | 안 읽은 메시지 수 조회 | 🔐 |

### AI 기능

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/sitters/me/ai-generate` | AI 시터 프로필 자동 생성 | 🔐 (SITTER) |
| POST | `/api/ai/chat` 📡 | AI 시터 추천 챗봇 (Tool Calling + SSE) | 🔐 |
| GET | `/api/sitters/semantic-search` | RAG 시맨틱 시터 탐색 | 🔐 |

### 렌탈·임시보호

| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | `/api/rentals/items` | 렌탈 용품 등록 | 🔐 |
| GET | `/api/rentals/items` | 렌탈 용품 목록 검색 | 🔓 |
| POST | `/api/rentals` | 렌탈 신청 | 🔐 |
| POST | `/api/temp-care` | 임시보호 공고 등록 | 🔐👑 (ADMIN) |
| POST | `/api/temp-care/{id}/apply` | 임시보호 지원 | 🔐 |

---

## 13. API 전체 요약 인덱스

### v1 MVP

| # | Method | Endpoint | 설명 | 인증 | 특이사항 |
|---|--------|----------|------|------|----------|
| 1 | POST | `/api/auth/signup` | 회원가입 | 🔓 | 역할 선택 |
| 2 | POST | `/api/auth/login` | 로그인 | 🔓 | JWT 발급 |
| 3 | POST | `/api/auth/logout` | 로그아웃 | 🔐 | Redis 블랙리스트 |
| 4 | POST | `/api/auth/reissue` | 토큰 재발급 | 🔐 | Refresh Token |
| 5 | GET | `/api/members/me` | 내 정보 조회 | 🔐 | |
| 6 | PUT | `/api/members/me` | 내 정보 수정 | 🔐 | |
| 7 | PATCH | `/api/members/me/password` | 비밀번호 변경 | 🔐 | |
| 8 | POST | `/api/members/me/profile-image` | 프로필 이미지 업로드 | 🔐 | S3 |
| 9 | DELETE | `/api/members/me` | 회원 탈퇴 | 🔐 | 소프트 삭제 |
| 10 | GET | `/api/pets` | 내 반려동물 목록 | 🔐 | |
| 11 | POST | `/api/pets` | 반려동물 등록 | 🔐 | |
| 12 | GET | `/api/pets/{petId}` | 반려동물 상세 조회 | 🔐 | |
| 13 | PUT | `/api/pets/{petId}` | 반려동물 수정 | 🔐 | |
| 14 | DELETE | `/api/pets/{petId}` | 반려동물 삭제 | 🔐 | |
| 15 | POST | `/api/sitters` | 시터 등록 | 🔐 | MEMBER→SITTER |
| 16 | GET | `/api/sitters/me` | 내 시터 프로필 조회 | 🔐 | SITTER |
| 17 | PUT | `/api/sitters/me` | 시터 프로필 수정 | 🔐 | SITTER |
| 18 | PATCH | `/api/sitters/me/availability` | 시터 상태 변경 | 🔐 | SITTER |
| 19 | POST | `/api/sitters/me/profile-image` | 시터 이미지 업로드 | 🔐 | S3, SITTER |
| 20 | GET | `/api/sitters` | 시터 검색 | 🔓 | 💾 QueryDSL |
| 21 | GET | `/api/sitters/{sitterId}` | 시터 상세 조회 | 🔓 | 💾 Redis |
| 22 | DELETE | `/api/sitters/me` | 시터 탈퇴 | 🔐 | SITTER→MEMBER |
| 23 | POST | `/api/posts` | 케어 공고 작성 | 🔐 | 시간 필수 |
| 24 | GET | `/api/posts` | 케어 공고 목록 | 🔓 | 💾 Redis |
| 25 | GET | `/api/posts/me` | 내 케어 공고 목록 | 🔐 | |
| 26 | GET | `/api/posts/{postId}` | 케어 공고 상세 | 🔓 | |
| 27 | PUT | `/api/posts/{postId}` | 케어 공고 수정 | 🔐 | |
| 28 | DELETE | `/api/posts/{postId}` | 케어 공고 삭제 | 🔐 | |
| 29 | PATCH | `/api/posts/{postId}/status` | 케어 공고 상태 변경 | 🔐 | |
| 30 | POST | `/api/posts/{postId}/proposals` | 제안 보내기 | 🔐 | SITTER, DB Unique, 스케줄 검증 |
| 31 | GET | `/api/posts/{postId}/proposals` | 제안 목록 조회 | 🔐 | |
| 32 | GET | `/api/proposals/{proposalId}` | 제안 상세 조회 | 🔐 | |
| 33 | PATCH | `/api/proposals/{proposalId}/accept` | 제안 수락 | 🔐 | Kafka, 충돌 자동 WITHDRAWN |
| 34 | PATCH | `/api/proposals/{proposalId}/reject` | 제안 거절 | 🔐 | |
| 35 | POST | `/api/sitters/{sitterId}/requests` | 돌봄 요청 | 🔐 | |
| 36 | GET | `/api/requests/received` | 받은 요청 목록 | 🔐 | SITTER |
| 37 | GET | `/api/requests/received/{requestId}` | 받은 요청 상세 | 🔐 | SITTER |
| 38 | PATCH | `/api/requests/{requestId}/accept` | 요청 수락 | 🔐 | SITTER, Kafka, 스케줄 검증 |
| 39 | PATCH | `/api/requests/{requestId}/reject` | 요청 거절 | 🔐 | SITTER |
| 40 | GET | `/api/requests/sent` | 보낸 요청 목록 | 🔐 | |
| 41 | GET | `/api/requests/sent/{requestId}` | 보낸 요청 상세 | 🔐 | |
| 42 | PATCH | `/api/requests/sent/{requestId}/cancel` | 보낸 요청 취소 | 🔐 | |
| 43 | POST | `/api/reservations` | 예약 생성 | 🔐 | 내부 호출 |
| 44 | GET | `/api/reservations/{reservationId}` | 예약 상세 조회 | 🔐 | |
| 45 | GET | `/api/reservations/me` | 내 예약 목록 | 🔐 | |
| 46 | PATCH | `/api/reservations/{id}/cancel` | 예약 취소 | 🔐 | CONFIRMED 이전만 |
| 47 | POST | `/api/payments` | 결제 요청 | 🔐 | Portone |
| 48 | POST | `/api/payments/confirm` | 결제 확인 | 🔓 | 콜백 |
| 49 | GET | `/api/payments/{paymentId}` | 결제 내역 조회 | 🔐 | |
| 50 | POST | `/api/reviews` | 후기 작성 | 🔐 | 양방향, DB Unique |
| 51 | GET | `/api/sitters/{sitterId}/reviews` | 시터 후기 목록 | 🔓 | 💾 Redis ZSET |
| 52 | GET | `/api/reviews/me` | 내 후기 목록 | 🔐 | |
| 53 | PUT | `/api/reviews/{reviewId}` | 후기 수정 | 🔐 | |
| 54 | DELETE | `/api/reviews/{reviewId}` | 후기 삭제 | 🔐 | |
| 55 | GET | `/api/notifications` | 알림 목록 | 🔐 | |
| 56 | PATCH | `/api/notifications/{id}/read` | 알림 읽음 | 🔐 | |
| 57 | PATCH | `/api/notifications/read-all` | 알림 전체 읽음 | 🔐 | |
| 58 | GET | `/api/notifications/stream` | SSE 알림 구독 | 🔐 | 📡 SSE |

### 확장 기능

| # | Method | Endpoint | 설명 | 인증 | 특이사항 |
|---|--------|----------|------|------|----------|
| E1 | GET | `/api/chats` | 채팅방 목록 | 🔐 | WebSocket STOMP |
| E2 | POST | `/api/chats` | 채팅방 생성 | 🔐 | |
| E3 | GET | `/api/chats/{id}/messages` | 메시지 목록 | 🔐 | cursor 기반 |
| E4 | POST | `/api/chats/{id}/messages` | 메시지 전송 | 🔐 | |
| E5 | PATCH | `/api/chats/{id}/read` | 읽음 처리 | 🔐 | |
| E6 | GET | `/api/chats/unread-count` | 안 읽은 수 | 🔐 | |
| E7 | POST | `/api/sitters/me/ai-generate` | AI 프로필 생성 | 🔐 | LLM 구조화 출력 |
| E8 | POST | `/api/ai/chat` | AI 챗봇 | 🔐 | 📡 SSE, Tool Calling |
| E9 | GET | `/api/sitters/semantic-search` | RAG 시맨틱 탐색 | 🔐 | 벡터 검색 |

---

## 14. 주요 설계 결정 요약

### v1 시간 제약 정책

```
v1:  Request/Offer ──────────────────────→ 수락 → Reservation

v2:  Request/Offer → Chat/협상 → 최종 Offer → 수락 → Reservation
```

v1에서는 시간 조율 없이, 게시글/요청에 명시된 시간 전체에 가능한 시터만 제안·수락할 수 있다. 수락 시 시터의 다른 PENDING 제안 중 시간이 겹치는 것을 자동 WITHDRAWN 처리하고 SSE 알림(`PROPOSAL_WITHDRAWN`)을 전송한다.

### 동시성 제어 설계 원칙

| API | 제어 방식 | 실패 전략 | 이유 |
|-----|-----------|-----------|------|
| 시터 제안 입찰 | DB Unique (`post_id + sitter_profile_id`) | 409 PROPOSAL_DUPLICATED | 동일 시터 중복 제안 원천 차단 |
| 제안/요청 수락 | 스케줄 검증 + 예약 생성 | 409 POST_ALREADY_MATCHED | 스케줄 충돌 방지 |
| 후기 작성 | DB Unique (`reservation_id + reviewer_id`) | 409 REVIEW_DUPLICATED | 양방향 각 1건 보장 |

### 예약 취소 정책

| 현재 상태 | 취소 가능 여부 | 이유 |
|-----------|-------------|------|
| `PAYMENT_PENDING` | 가능 | 결제 전이므로 자유 취소 |
| `CONFIRMED` | 가능 | 케어 시작 전이므로 취소 허용 |
| `IN_PROGRESS` | 불가 | 케어 진행 중 취소 불가 |
| `COMPLETED` | 불가 | 이미 완료된 예약 |

### 페이징 전략

| 대상 | 페이징 방식 | 이유 |
|------|-----------|------|
| 게시글·시터·예약 목록 | offset 기반 (`page`, `size`) | 총 페이지 표시가 필요한 탐색형 UI |
| 채팅 메시지 (확장) | cursor 기반 (`cursor`, `size`) | 무한 스크롤, 실시간 추가 환경에서 offset은 데이터 누락 발생 |
| 내 후기·알림 내역 | offset 기반 | 사용자가 직접 페이지 선택하는 패턴 |

---

## 15. 주요 변경 이력

| 버전 | 변경 내용 | 사유 |
|------|-----------|------|
| v1.0 | 최초 작성 | 프로젝트 시작 |
| v1.1 | 회의 후 수정사항 반영 | ERD 재설계 및 프로젝트 개요서 v1.1 피드백 반영 |
| v1.2 | v1 시간 제약 정책·역할·예약 상태 수정 | edge case 분석 결과 반영 |

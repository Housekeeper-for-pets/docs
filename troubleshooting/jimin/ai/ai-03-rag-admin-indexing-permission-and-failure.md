# RAG 인덱싱 버튼 운영 — 관리자 권한과 부분 실패를 구분해야 했다

> 카테고리: AI/RAG · 2026-06 · 도메인: Admin · Authorization · Embedding · Qdrant

## 문제

RAG 리뷰 인덱싱은 처음에 Postman으로 직접 호출해 테스트했다.

```text
POST /api/ai/rag/reviews/index
```

하지만 배포 환경에서는 운영자가 매번 Postman으로 호출하기 어렵기 때문에, 관리자 계정에서만 보이는 `RAG 인덱싱` 버튼을 프론트에 추가했다.

버튼을 누르자 처음에는 다음 문제가 발생했다.

```text
403 Forbidden
접근 권한이 없습니다.
```

이후 권한 문제를 해결해 호출은 성공했지만, 결과가 다음처럼 일부 실패를 포함했다.

```json
{
  "indexedCount": 94,
  "failedCount": 36
}
```

프론트에는 다음처럼 표시됐다.

```text
성공 61건 · 실패 69건
RAG 리뷰 인덱싱이 완료되었습니다.
```

## 원인

### 1. RAG 인덱싱 API는 관리자 전용이다

RAG 인덱싱은 전체 리뷰를 임베딩하고 Qdrant에 저장하는 운영성 작업이다. 일반 사용자가 누르면 Gemini 호출 비용이 발생하고, 벡터 DB 데이터가 반복 갱신될 수 있다.

따라서 백엔드에서는 관리자만 호출할 수 있도록 제한했다.

```java
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/reviews/index")
public ApiResponse<RagIndexResponse> indexReviews() {
    return ApiResponse.success(aiRagService.indexCompletedReviews());
}
```

프론트에서 버튼만 관리자에게 보이게 하는 것만으로는 부족했다. 실제 API 요청에도 `Authorization: Bearer {accessToken}` 헤더가 반드시 포함되어야 했다.

### 2. 인덱싱은 전체 성공/전체 실패가 아니다

RAG 인덱싱은 리뷰 1개마다 다음 작업을 반복한다.

```text
리뷰 조회
→ Gemini Embedding 생성
→ Qdrant upsert
```

이때 특정 리뷰의 임베딩 생성이나 Qdrant 저장이 실패해도 전체 인덱싱을 중단하지 않고, 가능한 문서는 계속 처리하도록 설계했다.

실제 로그에서도 일부 문서가 실패했다.

```text
RAG 문서 임베딩 실패.
sourceType=REVIEW
sourceId=130
message=RAG 임베딩 생성에 실패했습니다.
```

즉, `failedCount`가 있다는 것은 API 호출 자체가 실패했다는 뜻이 아니라, 일부 리뷰 문서가 인덱싱되지 못했다는 뜻이다.

## 해결

### 1. 관리자만 버튼을 볼 수 있게 처리

프론트에서는 JWT의 role이 `ADMIN`인 경우에만 `RAG 인덱싱` 버튼을 노출하도록 했다.

```text
role = ADMIN
→ RAG 인덱싱 버튼 표시

role != ADMIN
→ 버튼 숨김
```

### 2. API 요청에 Authorization 헤더 포함

버튼 클릭 시 RAG 인덱싱 API 요청에 accessToken을 포함하도록 확인했다.

```javascript
fetch("/api/ai/rag/reviews/index", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${localStorage.getItem("accessToken")}`,
  },
});
```

브라우저 콘솔에서 같은 토큰으로 직접 호출했을 때 `200 OK`가 반환되는 것을 확인했다.

```text
status: 200
{"success":true,"data":{"indexedCount":94,"failedCount":36},"error":null}
```

### 3. 성공/실패 카운트를 운영 결과로 표시

인덱싱 결과는 성공 개수와 실패 개수를 함께 보여주도록 했다.

```text
성공 N건 · 실패 M건
RAG 리뷰 인덱싱이 완료되었습니다.
```

실패가 0이 아니어도 검색 가능한 문서가 존재하면 RAG 기능은 동작한다. 다만 실패 문서는 로그를 보고 원인을 확인한 뒤 재시도할 수 있다.

## 결과

관리자 계정에서 버튼을 눌러 RAG 인덱싱을 실행할 수 있게 됐다.

```text
관리자 로그인
→ AI 추천 페이지 진입
→ RAG 인덱싱 버튼 표시
→ 버튼 클릭
→ indexedCount / failedCount 확인
```

이후 AI 추천 챗봇에서 실제 리뷰 근거가 함께 표시되는 것을 확인했다.

```text
실제 리뷰 근거 5개 참고
RAG 리뷰 근거 참고
```

## 회고

운영성 API는 단순히 “프론트에 버튼을 만든다”로 끝나지 않았다.

백엔드 권한, 프론트 버튼 노출 조건, Authorization 헤더, 처리 시간, 부분 실패 표시까지 함께 설계해야 했다. 특히 RAG 인덱싱처럼 외부 API와 벡터 DB를 함께 사용하는 작업은 실패가 일부 문서에만 발생할 수 있으므로, 전체 실패와 부분 실패를 구분해서 보여주는 것이 중요했다.

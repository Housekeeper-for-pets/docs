# AI 리뷰 요약 실패 — 배포 DB에 프롬프트 템플릿이 없었다

> 카테고리: AI 리뷰 요약 · 2026-06 · 도메인: Gemini · Prompt Template · Sitter Review Summary

## 문제

로컬에서는 시터 상세 페이지에서 `AI 요약 갱신` 버튼을 누르면 리뷰 요약이 생성됐다.

하지만 배포 환경에서는 리뷰가 존재하는 시터 상세 페이지에서도 다음 문구가 표시됐다.

```text
요약을 불러오지 못했습니다. 잠시 후 다시 갱신해주세요.
활성화된 프롬프트 템플릿이 없습니다.
```

프론트에서는 보호자 리뷰가 보이는데, AI REVIEW 영역만 비어 보이는 상태였다.

## 원인

AI 리뷰 요약 기능은 프롬프트를 코드에 하드코딩하지 않고 DB의 `ai_prompt_templates` 테이블에서 조회한다.

```text
feature = SITTER_REVIEW_SUMMARY
active = true
```

조건에 맞는 활성 프롬프트 템플릿이 있어야 Gemini 요청을 만들 수 있다.

로컬 DB에는 개발 중 넣어둔 프롬프트 템플릿이 있었지만, 배포 DB에는 해당 초기 데이터가 들어가 있지 않았다.

즉, 문제는 Gemini API나 프론트 UI가 아니라 배포 DB 초기 데이터 누락이었다.

```text
로컬 DB: ai_prompt_templates 존재
배포 DB: ai_prompt_templates 없음
→ AI 리뷰 요약 요청 실패
```

## 해결

배포 환경에서 AI 기능을 사용하려면 코드 배포뿐 아니라 AI 초기 데이터도 함께 준비해야 한다.

필수로 확인해야 하는 데이터는 다음과 같다.

```text
1. ai_prompt_templates에 SITTER_REVIEW_SUMMARY 활성 템플릿 존재
2. 해당 시터에게 COMPLETED 예약 기반 리뷰 존재
3. Gemini API Key 설정
4. spring profile에 ai 포함
```

프롬프트 템플릿은 기능별로 관리되므로, 리뷰 요약용 템플릿을 배포 DB에 등록해야 한다.

```sql
SELECT *
FROM ai_prompt_templates
WHERE feature = 'SITTER_REVIEW_SUMMARY'
  AND active = true;
```

조회 결과가 없으면 배포 DB에 활성 프롬프트 템플릿을 추가해야 한다.

## 결과

프롬프트 템플릿을 배포 DB에 넣은 뒤에는 시터 상세 페이지에서 AI 리뷰 요약을 정상 생성할 수 있었다.

리뷰 요약 생성 후에는 `sitter_review_summaries`에 결과가 저장되고, 이후 시터 상세 페이지 조회 시 저장된 요약을 재사용한다.

```text
AI 요약 갱신
→ Gemini 구조화 응답 생성
→ sitter_review_summaries 저장
→ 시터 상세 페이지 AI REVIEW 영역 표시
```

## 회고

AI 기능은 코드만 배포한다고 끝나지 않았다.

프롬프트를 DB로 외부화한 덕분에 코드 수정 없이 프롬프트를 바꿀 수 있었지만, 반대로 배포 환경에는 초기 프롬프트 데이터가 반드시 필요했다. 이후에는 AI 기능 배포 체크리스트에 `profile`, `API Key`, `프롬프트 템플릿`, `테스트 리뷰 데이터`를 함께 넣어야 한다는 점을 배웠다.

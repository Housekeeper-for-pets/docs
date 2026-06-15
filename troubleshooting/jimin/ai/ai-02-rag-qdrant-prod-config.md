# RAG 검색 실패 — 배포 환경 Qdrant URL은 localhost가 아니었다

> 카테고리: AI/RAG · 2026-06 · 도메인: Qdrant · Docker Compose · 배포 설정

## 문제

로컬 프론트에서 AI 추천을 테스트할 때 다음 오류가 발생했다.

```text
Qdrant search 실패.
collection=forpets_reviews
message=I/O error on POST request for
"http://localhost:6333/collections/forpets_reviews/points/search":
Connection refused
```

AI 추천 응답은 일부 fallback으로 내려왔지만, RAG 리뷰 근거가 붙지 않거나 추천 카드가 비어 보이는 문제가 있었다.

## 원인

백엔드는 Qdrant base URL을 환경 변수로 받도록 되어 있었다.

```yaml
forpets:
  ai:
    rag:
      qdrant:
        base-url: ${QDRANT_BASE_URL:http://localhost:6333}
```

로컬에서 백엔드를 직접 실행할 때는 Qdrant가 Mac의 `localhost:6333`에 떠 있으므로 문제가 없었다.

하지만 배포 환경에서는 백엔드와 Qdrant가 Docker Compose 안에서 서로 다른 컨테이너로 실행된다. 이때 백엔드 컨테이너 내부의 `localhost`는 Qdrant 컨테이너가 아니라 자기 자신을 의미한다.

```text
로컬 직접 실행:
Spring Boot → localhost:6333 → Mac에 떠 있는 Qdrant

배포 Docker 실행:
Spring Boot 컨테이너 → localhost:6333 → Spring Boot 컨테이너 자기 자신
```

따라서 배포 환경에서는 Docker Compose 서비스 이름을 사용해야 했다.

```text
QDRANT_BASE_URL=http://qdrant:6333
```

## 해결

배포 환경에서 AI 기능을 사용하도록 `prod,ai` 프로필과 Qdrant URL을 함께 설정했다.

```text
SPRING_PROFILES_ACTIVE=prod,ai
QDRANT_BASE_URL=http://qdrant:6333
```

로컬 테스트 시에는 다음처럼 Qdrant 컨테이너를 띄우고 백엔드를 `local,ai` 프로필로 실행했다.

```bash
docker compose up -d qdrant
export GEMINI_API_KEY=...
./gradlew bootRun --args='--spring.profiles.active=local,ai'
```

배포에서는 GitHub Secrets 또는 EC2 설정 파일에 AI 관련 환경 변수를 추가해야 한다.

```text
GEMINI_API_KEY=...
QDRANT_BASE_URL=http://qdrant:6333
SPRING_PROFILES_ACTIVE=prod,ai
```

## 결과

Qdrant URL이 올바르게 설정된 뒤에는 AI 추천에서 실제 리뷰 근거가 함께 표시됐다.

```text
추천 시터 3명
실제 리뷰 근거 5개 참고
RAG 리뷰 근거 참고
```

또한 RAG 검색 결과가 없거나 Qdrant가 일시적으로 실패해도 추천 기능 전체가 중단되지 않도록 fallback 처리했다.

```text
Qdrant 검색 실패
→ sources 빈 리스트
→ 기본 시터 추천 응답 유지
```

## 회고

로컬에서 동작하는 `localhost` 설정은 배포 환경에서 그대로 믿으면 안 된다.

특히 Docker Compose 안에서는 `localhost`가 내 PC가 아니라 컨테이너 자기 자신을 가리킨다. 외부 인프라를 붙이는 기능은 로컬 실행 방식과 배포 실행 방식의 네트워크 주소가 다르므로, `profile`, `환경 변수`, `compose service name`을 배포 체크리스트에 같이 넣어야 한다.

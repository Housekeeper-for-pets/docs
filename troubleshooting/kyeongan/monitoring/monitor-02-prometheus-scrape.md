# Prometheus는 push가 아니라 pull(scrape) — 그리고 지표는 한 번 실행돼야 등록된다

> 카테고리: 모니터링 · 2026-06-09 · 도메인: Prometheus · Micrometer

## 1) scrape 방향
앱이 어딘가로 지표를 push하는 게 아니라, 앱은 `/actuator/prometheus`를 **열어두기만** 하고 **Prometheus가 주기적으로 긁어간다(pull)**. 앱은 호스트에서, Prometheus는 컨테이너에서 돌 때 컨테이너의 `localhost`는 자기 자신이라 앱을 못 찾는다 → scrape 대상을 **`host.docker.internal:8080`**으로 지정해 해결.

## 2) 지표 등록 시점
추가한 지표가 안 보여 당황 → 해당 **코드가 한 번이라도 실행돼야 시계열이 생긴다**. API를 호출해 캐시를 한 번 타게 한 뒤에야 `cache_gets_total`이 나타났다.

## 3) 계측 의존성이 테스트를 깬다 (MeterRegistry NPE)
`ReviewService`에 `MeterRegistry`를 주입하니 기존 테스트가 NPE로 깨졌다(테스트가 주입을 안 해줌). mock은 `counter()`가 null을 반환해 또 NPE → 실제 가벼운 **`SimpleMeterRegistry`** 주입으로 해결.

## 배운 점
관측은 "지표를 만들면 보인다"가 아니다. **scrape 도달 가능성(네트워크) + 최초 실행(시계열 생성) + 테스트 영향(의존성 추가)**까지 묶여 있다. CI가 의존성 추가의 부작용을 잡아준다.

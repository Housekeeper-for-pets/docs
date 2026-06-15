# ForPets 트러블슈팅

> 작성자: 권지원  
> 담당 영역: 메인 비즈니스 로직 · 동시성/분산락 · 스케줄러  
> 기간: 2026-05 ~ 2026-06

ForPets에서 예약를 비롯한 메인 비즈니스 로직, 분산락·AOP 를 활용한 Transaction 설계, 스케줄러 기능을 구현하며 마주친 문제와 해결 과정을 정리했습니다.  
각 문서는 `문제 / 원인 / 해결 / 결과 / 회고` 형식으로 짧게 정리했습니다.

---

## 메인 비즈니스 로직 - 결제 

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [PG 부분환불 불가 문제](payment-partial-cancel-issue.md) | PG 부분취소 정책 제한을 우회해 내부 정산 기록으로 분리 처리 |

## 동시성 / 분산락

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [Lock과 Transaction의 Race Condition](race-condition-issue.md) | Lock이 Transaction 바깥을 감싸야 커밋 전 공백을 막을 수 있음 |
| 2 | [Spring AOP 체이닝 시 JoinPointMatch 바인딩 버그](aop-chaining-binding-issue.md) | Advice 2개 체이닝 시 바인딩 실패 → Reflection으로 직접 꺼내 해결 |

## 스케줄러 / 인프라

| # | 케이스 | 한 줄 요약 |
| --- | --- | --- |
| 1 | [멀티 인스턴스 스케줄러 중복 실행 방지](multi-instance-shedlock.md) | ShedLock으로 스케줄러 메서드 진입 자체를 단일 인스턴스로 제한 |

---

> 반복해서 배운 것: 프레임워크가 알아서 해준다는 믿음이 가장 위험한 전제다. Lock이 Transaction 안에 있으면 Lock 해제와 커밋 사이의 짧은 공백에서 Race Condition이 생긴다. AOP도 마찬가지로, Advice가 하나일 때 잘 동작하던 바인딩이 두 개가 체이닝되는 순간 조용히 깨진다. Spring이 채워주는 것에 의존하기보다 직접 꺼내고, Lock은 반드시 Transaction을 감싸는 구조로 설계해야 한다.
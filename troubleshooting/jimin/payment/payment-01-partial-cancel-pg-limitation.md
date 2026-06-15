# PG 부분취소 제한 대응 — 환불과 정산 기록을 분리한 이유

> 카테고리: 결제/정산 · 2026-06 · 도메인: Payment · Settlement · PortOne

## 문제

보호자가 확정된 예약을 개인 사유로 취소하면, 결제 금액 중 위약금을 제외한 금액만 환불해야 했다.

기존 의도는 다음과 같았다.

```text
보호자 결제 100,000원
→ 위약금 20,000원
→ PG 부분취소 80,000원
→ 시터에게 위약금 Settlement 20,000원 생성
```

하지만 실제 PG 호출에서 다음 오류가 발생했다.

```text
502 Bad Gateway
pgCode=500503
pgMessage=간편결제 부분취소 제한 가맹점
```

부분취소가 실패하면서 보호자 환불도 실패하고, 위약금 정산 기록도 생성되지 않는 문제가 생겼다.

## 원인

PortOne을 통해 연동한 KG 이니시스 결제는 결제 수단과 가맹점 설정에 따라 부분취소가 제한될 수 있었다.

즉, 애플리케이션 로직은 부분취소를 지원한다고 가정했지만, 실제 PG 정책은 모든 결제 수단에서 부분취소를 보장하지 않았다.

```text
서비스 정책: 부분 환불 필요
PG 정책: 일부 결제 수단 또는 가맹점에서 부분취소 불가
```

결제 도메인에서 외부 PG 기능을 서비스 정책의 필수 전제로 두면, PG 제약 하나로 예약 취소 플로우 전체가 막히는 구조였다.

## 해결

PG 부분취소를 직접 호출하지 않고, 환불 대상 금액과 위약금 금액을 내부 정산 기록으로 분리했다.

변경 후 흐름은 다음과 같다.

```text
보호자 결제 100,000원
→ PG 부분취소 호출 없음
→ Payment 상태는 REFUNDED로 종료
→ 보호자 환불분 Settlement 80,000원 생성
→ 시터 위약금 Settlement 20,000원 생성
→ 관리자가 Settlement 기준으로 수동 처리
```

이를 위해 `SettlementType`에 보호자 환불분을 기록하는 타입을 추가했다.

```java
public enum SettlementType {
    CARE_COMPLETION,
    SITTER_CANCEL_PENALTY,
    OWNER_CANCEL_PENALTY,
    GUARDIAN_REFUND
}
```

또한 예약 1건에 Settlement가 1개만 생성된다고 가정하던 중복 체크를 `(reservationId, settlementType)` 기준으로 변경했다.

```java
boolean existsByReservationIdAndSettlementType(
        Long reservationId,
        SettlementType settlementType
);
```

보호자 취소 시에는 환불분과 위약금을 각각 별도 Settlement로 생성하도록 분리했다.

```java
if (refundAmount > 0) {
    settlementService.createGuardianRefundSettlement(
            reservationId,
            payment.getMemberId(),
            payment.getId(),
            refundAmount,
            "보호자 취소 환불분 - " + reason
    );
}

if (penalty > 0) {
    settlementService.createPenaltySettlement(
            reservationId,
            sitterMemberId,
            payment.getId(),
            penalty,
            SettlementType.OWNER_CANCEL_PENALTY,
            "보호자 취소 위약금 - " + reason
    );
}
```

## 결과

PG 부분취소가 불가능한 결제 수단에서도 예약 취소 정책을 데이터로 남길 수 있게 됐다.

```text
보호자 환불분 → GUARDIAN_REFUND
시터 위약금 → OWNER_CANCEL_PENALTY
시터 취소 위약금 → SITTER_CANCEL_PENALTY
케어 완료 정산 → CARE_COMPLETION
```

PG 호출 성공 여부와 내부 정산 추적을 분리하면서, 취소 정책이 PG 제약에 직접 막히지 않게 됐다.

## 회고

PG API는 서비스 정책을 그대로 보장해주는 도구가 아니었다.

특히 부분취소처럼 결제 수단과 가맹점 설정에 따라 달라지는 기능은 핵심 정책 처리 수단으로 두면 위험하다. 이번 변경으로 실제 PG 환불과 서비스 내부 정산 기록을 분리했고, 운영자가 후속 처리할 수 있는 근거 데이터를 남길 수 있게 됐다.

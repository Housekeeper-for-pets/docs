# PG 부분환불 불가 문제 해결

작성자: 권지원

### PG사 부분취소 정책 제한을 우회하기 위한 내부 정산 분리

---

## 1. 문제

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

---

## 2. 원인

PortOne을 통해 연동한 KG 이니시스는 결제 수단과 가맹점 설정에 따라 부분취소가 제한될 수 있다.

애플리케이션 로직은 부분취소가 가능하다고 전제했지만, 실제 PG 정책은 모든 결제 수단에서 부분취소를 보장하지 않았다.

```text
서비스 정책: 부분 환불 필요
PG 정책: 일부 결제 수단 또는 가맹점에서 부분취소 불가
```

외부 PG의 기능을 서비스 정책의 필수 전제로 두면, PG 제약 하나로 예약 취소 플로우 전체가 막히는 구조가 된다.

---

## 3. 해결 방법

PG 부분취소를 직접 호출하지 않고, 환불 금액과 위약금을 내부 정산 기록으로 분리했다.

변경 후 흐름은 다음과 같다.

```text
보호자 결제 100,000원
→ PG 부분취소 호출 없음
→ Payment 상태는 REFUNDED로 종료
→ 보호자 환불분 Settlement 80,000원 생성 (GUARDIAN_REFUND)
→ 시터 위약금 Settlement 20,000원 생성 (OWNER_CANCEL_PENALTY)
→ 관리자가 Settlement 기준으로 수동 처리
```

### 3-1. SettlementType 추가

보호자 환불분을 기록하는 타입을 추가했다.

```java
public enum SettlementType {
    CARE_COMPLETION,
    SITTER_CANCEL_PENALTY,
    OWNER_CANCEL_PENALTY,
    GUARDIAN_REFUND          // 신규 추가
}
```

### 3-2. 중복 체크 기준 변경

예약 1건에 Settlement가 1개만 생성된다고 가정하던 중복 체크를 `(reservationId, settlementType)` 복합 기준으로 변경했다.

```java
boolean existsByReservationIdAndSettlementType(
        Long reservationId,
        SettlementType settlementType
);
```

### 3-3. 보호자 취소 시 Settlement 분리 생성

환불분과 위약금을 각각 별도 Settlement로 생성하도록 분리했다.

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

---

## 4. 결과

PG 부분취소가 불가능한 결제 수단에서도 예약 취소 정책을 데이터로 남길 수 있게 됐다.

| SettlementType          | 설명               |
|-------------------------|--------------------|
| `GUARDIAN_REFUND`       | 보호자 취소 환불분 |
| `OWNER_CANCEL_PENALTY`  | 보호자 취소 위약금 |
| `SITTER_CANCEL_PENALTY` | 시터 취소 위약금   |
| `CARE_COMPLETION`       | 케어 완료 정산     |

PG 호출 성공 여부와 내부 정산 추적을 분리하면서, 취소 정책이 PG 제약에 직접 막히지 않는 구조가 됐다.

---

## 5. 회고

PG API는 서비스 정책을 그대로 보장해주는 도구가 아니다.

특히 부분취소처럼 결제 수단과 가맹점 설정에 따라 동작이 달라지는 기능은, 핵심 정책 처리의 필수 수단으로 두면 위험하다. 이번 변경으로 실제 PG 환불과 서비스 내부 정산 기록을 분리했고, 운영자가 후속 처리할 수 있는 근거 데이터를 남길 수 있게 됐다.
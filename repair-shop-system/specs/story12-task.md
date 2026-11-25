# [US-12] 결제·출고 처리(선결제·후결제) — 개발 Task 설계서

| 항목 | 내용 |
| --- | --- |
| 에픽 | E4 매출·정산 |
| 스토리 | 직원으로서 접수 시 또는 의류를 내줄 때 결제를 기록하고 출고 처리하고 싶다. 결제 누락 없이 의류를 인도하고 매출을 정확히 집계하기 위해서다. |
| 우선순위 / 릴리스 | 필수 / 1차 |
| 관련 기능 | F-15 결제 처리 |
| 관련 화면 / 데이터 | 접수 상세, S-03 수선 접수 / D-02 접수, D-03 결제 |
| 선행 작업 | US-01, US-02(출고 상태 전이) |
| 작성일 | 2026-11-25 |

---

## 1. 요약

- 결제수단: 현금·카드·계좌이체 기록(카드 단말기 연동 없이 결제수단만 기록, 연동 여부는 결정 필요)
- 선결제: 접수 화면에서 결제 기록 / 후결제: 출고 시 결제 기록
- [출고] 버튼 = 결제 확인 + 상태 '출고' 변경을 한 번에 처리(US-02 상태 변경을 결제와 함께 호출)
- 미결제 건 출고 처리 방식은 결정 필요 → 설정값으로 '출고 차단'(기본) / '경고 후 허용' 선택 가능하게 설계
- 한 접수에 결제 1건 원칙(분할 결제는 2차 검토)

---

## 2. 인수 조건

1. 접수 화면에서 [선결제]를 선택하고 결제수단을 고르면 접수와 함께 결제가 기록된다.
2. '완료' 상태 건에서 [출고]를 누르면 결제 여부가 표시되고, 미결제면 결제수단을 선택해 결제와 출고를 한 번에 처리할 수 있다.
3. 이미 결제된 건은 [출고]를 누르면 바로 출고된다.
4. 결제 금액은 접수 금액이 기본값이며, 금액을 바꾸면 변경 사유 입력이 필요하다.
5. 미결제 건 출고는 설정이 '출고 차단'이면 거부되고, '경고 후 허용'이면 경고 확인 후 출고되며 '미결제 출고'로 표시된다.
6. 결제 내역(일시, 수단, 금액, 처리자)이 접수 상세에 표시된다.

---

## 3. 데이터 모델 변경

```prisma
enum PaymentMethod { CASH  CARD  TRANSFER }
enum PaymentTiming { PREPAID  POSTPAID }

model Payment {                         // D-03 결제
  id            String        @id @default(cuid())
  orderId       String        @unique   // 접수 1건당 결제 1건(분할 결제는 2차)
  order         Order         @relation(fields: [orderId], references: [id])
  method        PaymentMethod
  timing        PaymentTiming
  amount        Int                     // 원
  amountReason  String?                 // 접수 금액과 다를 때 사유
  paidAt        DateTime      @default(now())
  paidById      String
  canceledAt    DateTime?               // US-13
  canceledById  String?
  cancelReason  String?
  @@index([paidAt])
}

model Order {
  // ...기존 항목
  payment          Payment?
  releasedUnpaid   Boolean  @default(false)   // 미결제 출고 표시
}
```

- StoreSetting 키 추가: `UNPAID_RELEASE_POLICY` = `BLOCK`(기본) | `WARN`

---

## 4. API 설계

| 메서드 | 경로 | 설명 | 권한 |
| --- | --- | --- | --- |
| POST | `/api/orders` | 접수 등록 시 `prepay` 포함 가능(US-01 API 확장) | 관리자·직원 |
| POST | `/api/orders/:id/payment` | 결제 기록(출고 없이) | 관리자·직원 |
| POST | `/api/orders/:id/release` | 출고(필요 시 결제 동시 기록) | 관리자·직원 |

```json
// POST /api/orders/:id/release 요청
{ "version": 5, "payment": { "method": "CARD", "amount": 8000, "amountReason": null }, "confirmUnpaid": false }

// 응답 200
{ "status": "RELEASED", "paid": true, "releasedUnpaid": false }

// 응답 409 (미결제 + BLOCK 정책)
{ "error": "UNPAID_BLOCKED" }

// 응답 428 (미결제 + WARN 정책, confirmUnpaid=false)
{ "error": "UNPAID_CONFIRM_REQUIRED" }
```

### 출고 처리 순서 (하나의 트랜잭션)

1. 상태가 '완료'인지, version 일치 확인(US-02 낙관적 잠금 재사용)
2. 기존 결제 없음 + 요청에 payment 있음 → Payment 생성(금액 다르면 사유 필수)
3. 결제 없음 + payment 없음 → 정책에 따라 409 / 428 / `releasedUnpaid=true`로 진행
4. 상태 '출고' 변경, releasedAt 기록, OrderStatusLog 저장

---

## 5. 화면

### 접수 화면(S-03) 결제 영역

```
결제  (●) 후결제(찾아갈 때)   ( ) 선결제   수단 [현금|카드|이체]
```

### 접수 상세 출고 팝업

```
┌ 출고 처리 ──────────────────────┐
│ 20261125-007 / 홍*동 / 바지        │
│ 결제 상태: 미결제                   │
│ 금액 [ 8,000 ] 원  수단 [현금|카드|이체] │
│ (금액 변경 시) 사유 [            ]   │
│              [취소] [결제 후 출고]  │
└─────────────────────────────────┘
```

- 결제 완료 건은 "결제 완료(카드, 11-25)" 표시 후 [출고]만 노출
- 목록(US-04)·고객 이력(US-06)에 결제 여부 표시 연결

---

## 6. 개발 Task 목록

| No | Task | 산출물 | 선행 |
| --- | --- | --- | --- |
| T12-01 | Payment 모델, Order.releasedUnpaid 추가, 마이그레이션 | 마이그레이션 | US-02 |
| T12-02 | 결제 Zod 스키마(금액 변경 시 사유 필수) | `lib/validators/payment.ts` | - |
| T12-03 | 결제 기록 API, 접수 등록 선결제 확장 | `/payment`, `POST /api/orders` | T12-01, T12-02 |
| T12-04 | 출고 API(정책 처리, 트랜잭션, 상태 이력) | `/release` | T12-03 |
| T12-05 | 미결제 출고 정책 설정 키 | 설정 API 수정 | US-08 |
| T12-06 | 접수 화면 결제 영역 | `orders/new` 수정 | T12-03 |
| T12-07 | 출고 팝업, 결제 내역 영역 | `orders/[id]` 수정 | T12-04 |
| T12-08 | 목록·고객 이력 결제 여부 표시 | US-04·US-06 화면 수정 | T12-01 |
| T12-09 | 테스트(인수 조건 1~6) | `tests/us12.*` | T12-06~T12-08 |

## 7. 인수 조건 ↔ 테스트

| 인수 조건 | 테스트 |
| --- | --- |
| 1 | 통합: 선결제 접수 시 Payment(PREPAID) 생성 |
| 2 | E2E: 미결제 건 출고 팝업에서 결제·출고 동시 처리 |
| 3 | 통합: 결제 완료 건 release → RELEASED |
| 4 | 통합: 금액 변경 + 사유 없음 → 400 |
| 5 | 통합: BLOCK → 409, WARN + 확인 → releasedUnpaid=true |
| 6 | E2E: 결제 내역 표시 |

## 8. 완료 기준

- [ ] 인수 조건 테스트 전체 통과
- [ ] 결제와 출고가 같은 트랜잭션으로 처리(한쪽만 반영되는 경우 없음)

## 9. 확인 필요

- 미결제 출고 정책(차단 / 경고 후 허용)
- 카드 결제 단말기 연동 여부
- 분할 결제(일부 선결제 + 잔액 후결제) 필요 여부
- 현금영수증 발행 처리 필요 여부

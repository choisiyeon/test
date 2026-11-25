# [US-02] 작업 상태 변경 — 개발 Task 설계서

| 항목 | 내용 |
| --- | --- |
| 에픽 | E1 접수·진행관리 |
| 스토리 | 직원으로서 수선 작업의 진행 상태를 변경하고 싶다. 사장님과 직원이 작업 진행 상황을 같은 화면에서 공유하기 위해서다. |
| 우선순위 / 릴리스 | 필수 / 1차 |
| 관련 화면 / 데이터 | S-04 접수 목록·상세, S-02 대시보드 / D-02 접수, D-06 상태 변경 이력(신규) |
| 선행 작업 | US-01 (공통 기반, Order 모델) |
| 작성일 | 2026-11-25 |

---

## 1. 요약

- 상태는 `접수 → 작업중 → 완료 → 출고` 순서로만 앞으로 진행, 건너뛰기 불가
- 이전 상태로 되돌리기는 관리자(사장님)만 가능, 사유 입력 필수
- 모든 상태 변경은 이력 테이블에 기록(누가·언제·무엇을)
- '완료' 변경 시 완료 알림(US-09)이 사용할 이벤트를 저장해 두고, 실제 발송은 US-09에서 구현

---

## 2. 인수 조건 (스토리 기준 초안)

1. '접수' 상태 건에서 [작업 시작]을 누르면 상태가 '작업중'이 된다.
2. '작업중' 상태 건에서 [작업 완료]를 누르면 상태가 '완료'가 되고 완료 일시가 기록된다.
3. 직원이 이전 상태로 되돌리려 하면 버튼이 보이지 않고, API로 요청해도 거부된다.
4. 관리자가 사유를 입력하고 되돌리면 상태가 한 단계 이전으로 바뀌고 이력에 사유가 남는다.
5. 두 사람이 같은 건을 동시에 변경하면 나중 요청은 "이미 변경된 건" 안내와 함께 거부된다.
6. 접수 목록에서 완료 예정일이 지난 미완료 건은 강조 표시된다.

※ '출고' 처리는 결제와 함께 이루어지므로 [출고] 버튼은 본 스토리에서 상태 변경만 구현하고, 결제 연결은 E4 결제 스토리에서 확장

---

## 3. 상태 전이 규칙

| 현재 상태 | 진행(전원) | 되돌리기(관리자) | 자동 처리 |
| --- | --- | --- | --- |
| 접수(RECEIVED) | → 작업중 | - | - |
| 작업중(IN_PROGRESS) | → 완료 | → 접수 | - |
| 완료(DONE) | → 출고 | → 작업중 | 완료 일시 기록, 알림 이벤트 저장 |
| 출고(RELEASED) | - | → 완료 | 출고 일시 기록 |

```ts
// lib/services/order-status.ts
const NEXT: Record<OrderStatus, OrderStatus | null> = {
  RECEIVED: 'IN_PROGRESS', IN_PROGRESS: 'DONE', DONE: 'RELEASED', RELEASED: null,
};
const PREV: Record<OrderStatus, OrderStatus | null> = {
  RECEIVED: null, IN_PROGRESS: 'RECEIVED', DONE: 'IN_PROGRESS', RELEASED: 'DONE',
};

export function canTransition(from: OrderStatus, to: OrderStatus, role: Role) {
  if (NEXT[from] === to) return true;                 // 진행: 전원
  if (PREV[from] === to) return role === 'ADMIN';     // 되돌리기: 관리자
  return false;                                       // 건너뛰기 등 불가
}
```

---

## 4. 데이터 모델 변경

```prisma
model Order {
  // ...US-01 항목 유지
  version      Int       @default(0)   // 동시 변경 방지(낙관적 잠금)
  doneAt       DateTime?               // 완료 일시
  releasedAt   DateTime?               // 출고 일시
  statusLogs   OrderStatusLog[]
}

model OrderStatusLog {                 // D-06 상태 변경 이력
  id         String      @id @default(cuid())
  orderId    String
  order      Order       @relation(fields: [orderId], references: [id])
  fromStatus OrderStatus
  toStatus   OrderStatus
  reason     String?                   // 되돌리기 시 필수
  changedById String
  changedAt  DateTime    @default(now())
  @@index([orderId])
}

model OutboxEvent {                    // 알림 등 후속 처리용 이벤트(US-09에서 소비)
  id          String   @id @default(cuid())
  type        String                    // 예: ORDER_DONE
  payload     Json
  processedAt DateTime?
  createdAt   DateTime @default(now())
  @@index([processedAt])
}
```

- OutboxEvent는 상태 변경과 같은 트랜잭션에서 저장 → 상태는 바뀌었는데 알림 요청이 유실되는 일 방지

---

## 5. API 설계

| 메서드 | 경로 | 설명 | 권한 |
| --- | --- | --- | --- |
| GET | `/api/orders?status=&from=&to=&q=&overdue=` | 접수 목록 검색(상태·기간·고객명·접수번호·기한 경과) | 관리자·직원 |
| PATCH | `/api/orders/:id/status` | 상태 변경 | 진행: 전원 / 되돌리기: 관리자 |
| GET | `/api/orders/:id/status-logs` | 상태 변경 이력 | 관리자·직원 |
| GET | `/api/dashboard/summary` | 상태별 건수, 오늘 완료 예정, 기한 경과 건수 | 관리자·직원 |

### PATCH `/api/orders/:id/status`

```json
// 요청
{ "toStatus": "DONE", "version": 3, "reason": null }

// 응답 200
{ "id": "ck...", "status": "DONE", "version": 4, "doneAt": "2026-11-27T05:12:00Z" }

// 응답 403 (직원의 되돌리기)
{ "error": "FORBIDDEN", "message": "이전 상태로 되돌리기는 관리자만 가능합니다" }

// 응답 409 (동시 변경)
{ "error": "CONFLICT", "message": "다른 사용자가 이미 변경한 건입니다. 화면을 새로고침하세요" }

// 응답 422 (허용되지 않는 전이·사유 누락)
{ "error": "INVALID_TRANSITION" }
```

### 처리 순서 (하나의 트랜잭션)

1. 세션 역할 확인 → `canTransition` 검사(실패 시 403 또는 422)
2. 되돌리기이면 사유 필수 검사
3. `UPDATE order SET status=?, version=version+1 WHERE id=? AND version=?` → 영향 행 0이면 409
4. `doneAt` / `releasedAt` 기록(되돌리기 시 해당 일시 초기화)
5. OrderStatusLog 저장
6. toStatus가 DONE이면 OutboxEvent(`ORDER_DONE`) 저장

---

## 6. 화면

### S-04 접수 목록

```
[상태: 전체▼] [기간: 2026-11-01 ~ 2026-11-30] [고객명·접수번호 검색] [기한 경과만 □]
┌──────────────┬──────┬──────┬──────────┬────────┬──────────┐
│ 접수번호      │ 고객  │ 품목  │ 완료예정일 │ 상태    │ 처리      │
├──────────────┼──────┼──────┼──────────┼────────┼──────────┤
│ 20261125-007 │ 홍*동 │ 바지  │ 11-28    │ 접수    │[작업 시작]│
│ 20261120-003 │ 김*수 │ 셔츠  │ 11-22 ⚠  │ 작업중  │[작업 완료]│
└──────────────┴──────┴──────┴──────────┴────────┴──────────┘
```

- 상태는 색상 배지로 구분(접수·작업중·완료·출고), 색만으로 구분하지 않도록 글자 병기
- 완료 예정일이 지난 미완료(접수·작업중) 건은 ⚠ 표시와 붉은 글씨
- 목록에서 바로 다음 단계 버튼 1개만 노출(오조작 방지)
- 고객명은 목록에서 가운데 글자 가림

### 접수 상세

- 상태 진행 표시줄(접수 ─ 작업중 ─ 완료 ─ 출고), 현재 단계 강조
- 다음 단계 버튼, 관리자에게만 [이전 단계로] 버튼 + 사유 입력 팝업
- 하단에 상태 변경 이력(일시, 변경자, 변경 내용, 사유)

### S-02 대시보드 (본 스토리에서 최초 구현)

- 카드 4개: 접수 / 작업중 / 완료(미출고) / 기한 경과 건수, 카드 클릭 시 해당 조건 목록으로 이동
- 오늘 완료 예정 목록

---

## 7. 개발 Task 목록

| No | Task | 산출물 | 선행 |
| --- | --- | --- | --- |
| T2-01 | Order에 version·doneAt·releasedAt 추가, OrderStatusLog·OutboxEvent 모델 추가, 마이그레이션 | 마이그레이션 | US-01 |
| T2-02 | 상태 전이 규칙 모듈 `canTransition` + 단위 테스트(전이 16가지 × 역할 2개) | `order-status.ts` | T2-01 |
| T2-03 | 상태 변경 API(트랜잭션, 낙관적 잠금, 이력·이벤트 저장) | `PATCH /api/orders/:id/status` | T2-02 |
| T2-04 | 접수 목록 검색 API(필터·기한 경과·페이지 처리) | `GET /api/orders` | US-01 |
| T2-05 | 상태 이력 조회 API | `GET /status-logs` | T2-03 |
| T2-06 | 대시보드 요약 API | `GET /api/dashboard/summary` | T2-04 |
| T2-07 | 접수 목록 화면(S-04), 상태 배지, 기한 경과 강조, 다음 단계 버튼 | `orders/page.tsx` | T2-03, T2-04 |
| T2-08 | 접수 상세 화면 확장(진행 표시줄, 되돌리기 팝업, 이력) | `orders/[id]` | T2-05 |
| T2-09 | 대시보드 화면(S-02) | `(main)/page.tsx` | T2-06 |
| T2-10 | 409 충돌 시 안내 후 자동 새로고침 처리 | 공통 오류 처리 | T2-07 |
| T2-11 | E2E 테스트(인수 조건 1~6) | `tests/e2e/us02.spec.ts` | T2-07~T2-10 |

---

## 8. 인수 조건 ↔ 테스트 매핑

| 인수 조건 | 테스트 |
| --- | --- |
| 1. 접수 → 작업중 | E2E: 목록 [작업 시작] 클릭 후 배지 '작업중' |
| 2. 작업중 → 완료, 완료 일시 기록 | 통합: doneAt 기록, OutboxEvent ORDER_DONE 1건 생성 |
| 3. 직원 되돌리기 불가 | E2E: 직원 화면에 [이전 단계로] 없음 / API: 직원 토큰으로 요청 시 403 |
| 4. 관리자 되돌리기 + 사유 기록 | E2E: 사유 입력 후 되돌리기, 이력에 사유 표시 / API: 사유 없으면 422 |
| 5. 동시 변경 거부 | 통합: 같은 version으로 2회 요청 → 두 번째 409 |
| 6. 기한 경과 강조 | E2E: 완료 예정일이 어제인 '작업중' 건에 ⚠ 표시 |
| 건너뛰기 불가 | 단위: RECEIVED → DONE 전이 false |

---

## 9. 완료 기준 (DoD)

- [ ] 인수 조건 테스트 전체 통과
- [ ] 모든 상태 변경이 이력에 남음(변경자·일시)
- [ ] '완료' 변경 시 OutboxEvent 생성 확인(US-09 연계 준비)
- [ ] 대시보드 건수와 목록 필터 결과 일치

## 10. 확인 필요

- 예외 상태(취소·보류·재작업) 필요 여부 → 필요 시 상태 추가 및 전이표 수정
- 출고 시 미결제 건 처리 방식(출고 차단 / 경고 후 허용) → E4 결제 스토리와 함께 확정
- 기한 경과 판단 기준 시각(완료 예정일 당일 영업 종료 후 / 다음 날 0시)

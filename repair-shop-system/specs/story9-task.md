# [US-09] 완료 알림 자동 발송 — 개발 Task 설계서

| 항목 | 내용 |
| --- | --- |
| 에픽 | E3 완료 알림 |
| 스토리 | 직원으로서 수선이 완료되면 고객에게 수령 안내가 자동으로 발송되기를 원한다. 고객마다 전화로 연락하는 수고를 줄이고 미수령 의류를 줄이기 위해서다. |
| 우선순위 / 릴리스 | 필수 / 1차 |
| 관련 기능 | F-12 완료 알림 발송 |
| 관련 화면 / 데이터 | 접수 상세 / D-02 접수, D-04 알림 이력, D-08 매장 설정 |
| 선행 작업 | US-02(OutboxEvent ORDER_DONE), US-05(전화번호 정규화) |
| 작성일 | 2026-11-25 |

---

## 1. 요약

- 상태가 '완료'로 바뀌면 US-02가 저장한 OutboxEvent(`ORDER_DONE`)를 발송 작업자(worker)가 읽어 알림 발송
- 발송 업체는 아직 미정이므로 발송 기능을 '어댑터' 구조로 분리 → 업체 결정 후 어댑터 1개만 구현
- 알림톡 실패 시 문자(SMS(단문 문자))로 대체 발송하는 구조 지원(사용 여부는 설정)
- 모든 발송 시도를 알림 이력(NotificationLog)에 기록

---

## 2. 인수 조건

1. 접수 건 상태를 '완료'로 바꾸면 해당 고객 전화번호로 수령 안내가 발송되고 알림 이력에 '성공'으로 기록된다.
2. 발송 문구에는 매장명, 접수번호, 품목, 수령 안내 문구가 포함된다.
3. 발송이 실패하면 알림 이력에 '실패'와 사유가 기록되고, 접수 상세에 "알림 발송 실패" 표시와 [다시 보내기] 버튼이 나타난다.
4. 같은 완료 건에 대해 자동 발송은 1회만 이루어진다(관리자가 되돌렸다가 다시 완료해도 중복 자동 발송 없음, 수동 재발송은 가능).
5. 발송 업체 설정이 없으면 발송하지 않고 이력에 '미설정'으로 기록하며 업무(상태 변경)는 정상 처리된다.

---

## 3. 기술 설계

### 3-1. 발송 작업자(worker)

- Docker Compose에 `worker` 서비스 추가(앱과 같은 코드, 실행 명령만 다름)
- 일정 주기로 `processedAt IS NULL`인 OutboxEvent 조회 → 처리 후 `processedAt` 기록
- 동시에 여러 작업자가 같은 이벤트를 처리하지 않도록 `SELECT ... FOR UPDATE SKIP LOCKED` 사용
- 조회 주기(예: 10초)와 재시도 횟수는 제안값, 확정 필요

### 3-2. 발송 어댑터

```ts
// lib/notify/types.ts
export interface NotifyProvider {
  name: 'ALIMTALK' | 'SMS' | 'NONE';
  send(input: { to: string; templateCode?: string; text: string }): Promise<
    { ok: true; providerMessageId: string } | { ok: false; reason: string }
  >;
}
// lib/notify/providers/none.ts  → 항상 { ok:false, reason:'NOT_CONFIGURED' }
// lib/notify/providers/console.ts → 개발용: 콘솔 출력 후 성공 처리
// lib/notify/providers/<업체>.ts → 업체 결정 후 구현
```

- 발송 업체 API 키는 환경변수(`.env`)로만 관리, DB·코드에 저장 금지
- 카카오 알림톡은 사전에 승인된 템플릿 문구만 발송 가능하므로 템플릿 코드와 변수 구조로 설계(업체 규정 확인 필요)

### 3-3. 처리 순서

1. 이벤트 수신 → 접수·고객 조회(고객 삭제·번호 없음이면 '대상 없음'으로 기록 후 종료)
2. 해당 접수에 자동 발송 성공 이력이 이미 있으면 종료(인수 조건 4)
3. 문구 생성(US-20 문구 설정 사용, 미구현 시 기본 문구)
4. 기본 수단 발송 → 실패 시 대체 수단 설정이 있으면 문자 발송
5. NotificationLog 저장, OutboxEvent 처리 완료 표시

---

## 4. 데이터 모델 변경

```prisma
enum NotifyType   { DONE_AUTO  DONE_MANUAL  REMIND }   // 자동 완료 안내 / 수동 재발송 / 미수령 재안내(US-11)
enum NotifyResult { SUCCESS  FAILED  NOT_CONFIGURED  NO_TARGET }

model NotificationLog {                // D-04 알림 이력
  id                String       @id @default(cuid())
  orderId           String
  type              NotifyType
  channel           String               // ALIMTALK | SMS | NONE
  toMasked          String               // 010-****-5678 (원문 저장 안 함)
  text              String
  result            NotifyResult
  failReason        String?
  providerMessageId String?
  sentById          String?              // 자동 발송이면 null
  createdAt         DateTime     @default(now())
  @@index([orderId])
  @@index([createdAt])
}
```

---

## 5. API 설계

| 메서드 | 경로 | 설명 | 권한 |
| --- | --- | --- | --- |
| POST | `/api/orders/:id/notify` | 완료 안내 수동 재발송(상태 '완료'일 때만) | 관리자·직원 |
| GET | `/api/orders/:id/notifications` | 해당 접수 알림 이력(US-10 공용) | 관리자·직원 |

```json
// POST /api/orders/:id/notify 응답
{ "result": "SUCCESS", "channel": "ALIMTALK" }
{ "result": "FAILED", "failReason": "수신 번호 오류" }
```

- 수동 재발송은 작업자를 거치지 않고 즉시 발송(직원이 결과를 바로 확인)

---

## 6. 화면

- 접수 상세 상단: 마지막 알림 결과 배지(성공·실패·미설정)
- 실패·미설정이면 [다시 보내기] 버튼, 발송 전 확인 창("홍*동 님께 수령 안내를 보냅니다")

---

## 7. 개발 Task 목록

| No | Task | 산출물 | 선행 |
| --- | --- | --- | --- |
| T9-01 | NotificationLog 모델 추가, 마이그레이션 | 마이그레이션 | US-02 |
| T9-02 | 발송 어댑터 인터페이스, none·console 어댑터 | `lib/notify/*` | - |
| T9-03 | 문구 생성 함수(기본 문구, 변수 치환) + 단위 테스트 | `lib/notify/template.ts` | - |
| T9-04 | 발송 서비스(중복 방지, 대체 발송, 이력 기록) | `lib/notify/service.ts` | T9-01~T9-03 |
| T9-05 | 작업자 프로세스(이벤트 조회·잠금·처리), Docker Compose `worker` 추가 | `worker/index.ts` | T9-04 |
| T9-06 | 수동 재발송·이력 조회 API | `/api/orders/:id/notify` | T9-04 |
| T9-07 | 접수 상세 알림 배지·재발송 버튼 | `orders/[id]` 수정 | T9-06 |
| T9-08 | 발송 업체 어댑터 구현(업체 결정 후) | `lib/notify/providers/<업체>.ts` | 업체 결정 |
| T9-09 | 테스트(인수 조건 1~5) | `tests/us09.*` | T9-05~T9-07 |

## 8. 인수 조건 ↔ 테스트

| 인수 조건 | 테스트 |
| --- | --- |
| 1 | 통합(console 어댑터): 완료 변경 → 작업자 실행 → 이력 SUCCESS 1건 |
| 2 | 단위: 문구에 매장명·접수번호·품목 포함 |
| 3 | 통합(실패 어댑터): 이력 FAILED + 사유 / E2E: [다시 보내기] 표시 |
| 4 | 통합: 완료 → 되돌리기 → 재완료 시 DONE_AUTO 1건만 존재 |
| 5 | 통합(none 어댑터): NOT_CONFIGURED 기록, 상태 변경 정상 |

## 9. 완료 기준

- [ ] 인수 조건 테스트 전체 통과
- [ ] 알림 이력에 전화번호 원문 미저장
- [ ] 작업자 재시작 후에도 미처리 이벤트가 누락 없이 처리됨

## 10. 확인 필요

- 발송 수단(알림톡 / 문자 / 둘 다)과 발송 업체, 발송 비용 부담 기준
- 작업자 조회 주기, 실패 시 자동 재시도 횟수
- 야간 발송 제한 필요 여부(예: 영업시간 외 발송 보류)

# [US-01] 수선 접수 등록 — 개발 Task 설계서

| 항목 | 내용 |
| --- | --- |
| 에픽 | E1 접수·진행관리 |
| 스토리 | 직원으로서 손님이 맡긴 의류를 접수 등록하고 싶다. 접수증을 잃어도 의류 주인과 수선 내용을 확인하기 위해서다. |
| 우선순위 / 릴리스 | 필수 / 1차 |
| 관련 화면 / 데이터 | S-03 수선 접수 / D-01 고객, D-02 접수 |
| 선행 작업 | 없음 (본 스토리에서 프로젝트 공통 기반 구축) |
| 작성일 | 2026-11-25 |

---

## 1. 요약

- 프로젝트 공통 기반(인증·DB·화면 틀)과 접수 등록 기능을 함께 구축
- 접수번호는 `YYYYMMDD-NNN`(일자+3자리 일련번호) 형식으로 서버에서 자동 부여
- 접수증은 브라우저 인쇄(영수증 프린터 폭 CSS)로 출력, 인쇄 실패와 무관하게 접수는 저장

---

## 2. 기술 스택 (전체 프로젝트 공통)

| 구분 | 선택 | 선택 이유 |
| --- | --- | --- |
| 프레임워크 | Next.js 15 (App Router) + TypeScript | 화면·서버 API를 한 프로젝트에서 개발, 1인 개발에 적합 |
| UI | Tailwind CSS + shadcn/ui | 큰 글씨·단순 메뉴 화면을 빠르게 구성 |
| DB | PostgreSQL 16 | 트랜잭션 기반 일련번호 채번 안정성 |
| ORM | Prisma | 스키마 관리·마이그레이션 단순 |
| 입력 검증 | Zod | 화면·서버 동일 규칙 공유 |
| 인증 | Auth.js(NextAuth v5) Credentials 방식 | 개인별 아이디·비밀번호 로그인, 역할(관리자·직원) 세션 보관 |
| 테스트 | Vitest(단위·통합), Playwright(화면 E2E(End-to-End, 종단 간)) | 인수 조건을 자동 테스트로 검증 |
| 배포 | Docker Compose (앱 + DB) | 클라우드·매장 PC 어느 쪽이든 동일하게 설치 가능 (서버 위치는 결정 필요사항) |

### 폴더 구조

```
/app
  /(auth)/login            # 로그인
  /(main)/orders/new       # S-03 수선 접수
  /(main)/orders/[id]      # 접수 상세 (US-02에서 확장)
  /(main)/orders/[id]/print# 접수증 출력
  /api/customers           # 고객 검색·등록
  /api/orders              # 접수 등록·조회
/lib
  /db.ts                   # Prisma 클라이언트
  /auth.ts                 # 인증 설정, 역할 확인 함수
  /validators/order.ts     # Zod 스키마
  /services/order-number.ts# 접수번호 채번
/prisma/schema.prisma
/tests
```

---

## 3. 데이터 모델 (Prisma)

```prisma
enum Role {
  ADMIN   // 사장님
  STAFF   // 직원
}

enum OrderStatus {
  RECEIVED     // 접수
  IN_PROGRESS  // 작업중
  DONE         // 완료
  RELEASED     // 출고
}

model User {
  id           String   @id @default(cuid())
  loginId      String   @unique
  passwordHash String
  name         String
  role         Role     @default(STAFF)
  active       Boolean  @default(true)
  createdAt    DateTime @default(now())
  orders       Order[]  @relation("OrderCreatedBy")
}

model Customer {               // D-01 고객
  id        String   @id @default(cuid())
  name      String
  phone     String              // 숫자만 저장 (예: 01012345678)
  memo      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  orders    Order[]
  @@index([phone])
  @@index([name])
}

model ItemType {               // 수선 품목 (바지·셔츠 등, 관리자 설정)
  id        String  @id @default(cuid())
  name      String  @unique
  basePrice Int?               // 기본 단가(원), 접수 시 금액 기본값
  sortOrder Int     @default(0)
  active    Boolean @default(true)
}

model Order {                  // D-02 접수
  id           String      @id @default(cuid())
  orderNo      String      @unique   // YYYYMMDD-NNN
  customerId   String
  customer     Customer    @relation(fields: [customerId], references: [id])
  itemTypeName String                // 접수 시점 품목명 저장(품목 변경 영향 방지)
  description  String                // 수선 내용
  amount       Int                   // 금액(원)
  dueDate      DateTime    @db.Date  // 완료 예정일
  memo         String?
  status       OrderStatus @default(RECEIVED)
  createdById  String
  createdBy    User        @relation("OrderCreatedBy", fields: [createdById], references: [id])
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
  @@index([status])
  @@index([dueDate])
}

model DailySequence {          // 접수번호 일련번호 채번용
  day     String @id           // YYYYMMDD
  lastSeq Int    @default(0)
}
```

---

## 4. API 설계

| 메서드 | 경로 | 설명 | 권한 |
| --- | --- | --- | --- |
| GET | `/api/customers?q=` | 이름 또는 전화번호 뒷자리로 고객 검색 (최대 20건) | 관리자·직원 |
| POST | `/api/customers` | 신규 고객 등록 | 관리자·직원 |
| GET | `/api/item-types` | 사용 중 품목 목록 | 관리자·직원 |
| POST | `/api/orders` | 수선 접수 등록, 접수번호 부여 | 관리자·직원 |
| GET | `/api/orders/:id` | 접수 상세 조회(접수증 출력용) | 관리자·직원 |

### POST `/api/orders` 요청·응답

```json
// 요청
{
  "customerId": "ck...",
  "itemTypeName": "바지",
  "description": "기장 3cm 줄임",
  "amount": 8000,
  "dueDate": "2026-11-28",
  "memo": "밑단 원단 보관 요청"
}

// 응답 201
{ "id": "ck...", "orderNo": "20261125-007", "status": "RECEIVED" }

// 응답 400 (필수 항목 누락)
{ "error": "VALIDATION", "fields": { "description": "수선 내용을 입력하세요" } }
```

### 입력 검증 규칙 (Zod, 화면·서버 공통)

| 항목 | 필수 | 규칙 |
| --- | --- | --- |
| customerId | ○ | 존재하는 고객 |
| itemTypeName | ○ | 1~30자 |
| description | ○ | 1~200자 |
| amount | ○ | 0 이상 정수 |
| dueDate | ○ | 오늘 이후 날짜 |
| memo | × | 최대 200자 |
| 고객 name / phone (신규) | ○ | 이름 1~20자 / 숫자 10~11자리 |

※ 글자 수 상한은 설계 제안값, 매장 확인 후 조정

---

## 5. 핵심 처리 로직

### 5-1. 접수번호 채번 (동시 접수 시 중복 방지)

```ts
// lib/services/order-number.ts
export async function createOrderWithNumber(tx: Prisma.TransactionClient, data: NewOrder) {
  const day = formatKST(new Date(), 'yyyyMMdd');           // 한국 시간 기준 일자
  const seq = await tx.dailySequence.upsert({
    where: { day },
    create: { day, lastSeq: 1 },
    update: { lastSeq: { increment: 1 } },                  // 행 잠금으로 순차 증가
  });
  const orderNo = `${day}-${String(seq.lastSeq).padStart(3, '0')}`;
  return tx.order.create({ data: { ...data, orderNo, status: 'RECEIVED' } });
}
```

- `prisma.$transaction` 안에서 호출, 채번과 저장을 하나의 트랜잭션으로 처리
- 1,000건 이상/일 접수 가능성은 낮으나, 초과 시 4자리로 자동 확장(`padStart`는 자릿수 초과 시 그대로 표기)

### 5-2. 접수 등록 흐름

1. 고객 검색 → 결과 선택 또는 "신규 고객 등록" 팝업에서 등록 후 자동 선택
2. 품목 선택 시 기본 단가가 있으면 금액 칸에 기본값 입력(수정 가능)
3. 저장 → 서버 검증 → 채번·저장 → 접수 상세로 이동, "접수증 출력" 버튼 강조
4. 출력 버튼 → `/orders/[id]/print` 새 창 → `window.print()`

### 5-3. 접수증 출력

- 영수증 프린터 폭에 맞춘 인쇄 전용 CSS(`@media print`, 폭 80mm 기준, 프린터 사양 확인 필요)
- 출력 항목: 매장명, 접수번호(크게), 접수일, 고객명, 전화번호 뒤 4자리, 품목, 수선 내용, 금액, 완료 예정일
- 의류 태그용 접수번호 라벨은 2차 검토

---

## 6. 화면 (S-03 수선 접수)

```
┌───────────────────────────────────────────────┐
│ 수선 접수                                       │
├───────────────────────────────────────────────┤
│ 고객  [ 이름 또는 전화번호 뒷자리 검색 ▼ ] [+신규] │
│ 품목  [ 바지 ▼ ]                                │
│ 수선 내용 [                                   ] │
│ 금액  [   8,000 ] 원   완료 예정일 [2026-11-28] │
│ 메모  [                                       ] │
│                         [ 취소 ]  [ 접수 저장 ] │
└───────────────────────────────────────────────┘
```

- 기본 글자 크기 18px 이상, 입력 순서대로 Tab 이동
- 필수 항목 누락 시 해당 칸 빨간 테두리 + 안내 문구, 저장 버튼은 비활성화하지 않음(누르면 누락 항목 표시)

---

## 7. 권한·보안

- 모든 화면·API는 로그인 필수(미들웨어에서 세션 확인, 미로그인 시 로그인 화면 이동)
- 비밀번호는 bcrypt 해시 저장
- 고객 검색 결과 목록에서 전화번호는 `010-****-5678` 형태로 가림, 상세 화면에서만 전체 표시
- 접수 기록에 등록자(createdById) 저장

---

## 8. 개발 Task 목록

| No | Task | 산출물 | 선행 |
| --- | --- | --- | --- |
| T1-01 | 프로젝트 생성(Next.js, TypeScript, Tailwind, shadcn/ui, ESLint) | 저장소, 실행 확인 | - |
| T1-02 | Docker Compose로 PostgreSQL 구성, Prisma 연결 | `docker-compose.yml`, `.env.example` | T1-01 |
| T1-03 | Prisma 스키마 작성·마이그레이션(User, Customer, ItemType, Order, DailySequence) | `schema.prisma`, 마이그레이션 | T1-02 |
| T1-04 | 초기 데이터(seed): 관리자 계정 1개, 품목 예시 | `prisma/seed.ts` | T1-03 |
| T1-05 | 로그인·세션·역할 확인(Auth.js), 미들웨어 | 로그인 화면, `lib/auth.ts` | T1-03 |
| T1-06 | 공통 레이아웃(상단 메뉴, 로그아웃) | `app/(main)/layout.tsx` | T1-05 |
| T1-07 | Zod 검증 스키마(접수·고객) | `lib/validators/*` | T1-03 |
| T1-08 | 고객 검색·등록 API | `/api/customers` | T1-07 |
| T1-09 | 접수번호 채번 서비스 + 단위 테스트(동시 요청 10건 중복 없음) | `order-number.ts` | T1-03 |
| T1-10 | 접수 등록·조회 API | `/api/orders` | T1-08, T1-09 |
| T1-11 | 수선 접수 화면(S-03), 고객 검색 콤보, 신규 고객 팝업 | `orders/new` | T1-06, T1-10 |
| T1-12 | 접수증 인쇄 화면·인쇄 CSS | `orders/[id]/print` | T1-10 |
| T1-13 | E2E 테스트(인수 조건 1~4, 예외 처리) | `tests/e2e/us01.spec.ts` | T1-11, T1-12 |

---

## 9. 인수 조건 ↔ 테스트 매핑

| 인수 조건 | 테스트 |
| --- | --- |
| 1. 필수 항목 입력 후 저장 → 접수번호 부여, 상태 '접수' | E2E: 저장 후 상세에 `YYYYMMDD-NNN`·'접수' 표시 / 통합: DB 상태값 RECEIVED |
| 2. 필수 항목 누락 시 해당 항목 표시, 저장 안 됨 | E2E: 수선 내용 비우고 저장 → 안내 문구, 주문 건수 변화 없음 / API 400 응답 |
| 3. 미등록 고객은 같은 화면에서 신규 등록 | E2E: 신규 팝업 등록 후 고객 칸 자동 선택 |
| 4. 출력 버튼 → 접수증 인쇄 | E2E: 인쇄 화면에 접수번호·고객명·완료 예정일 표시, `window.print` 호출 확인 |
| 예외: 프린터 미연결 시에도 접수 저장 | 통합: 저장 API는 인쇄와 독립 / E2E: 인쇄 화면 닫아도 접수 유지 |
| 채번 중복 방지 | 단위: 동시 10건 등록 시 접수번호 모두 다름 |

---

## 10. 완료 기준 (DoD(Definition of Done, 완료 정의))

- [ ] 인수 조건 테스트 전체 통과
- [ ] 직원 계정으로 접수 등록·출력까지 화면에서 수행 가능
- [ ] 로그인하지 않으면 모든 화면·API 접근 불가
- [ ] 목록 화면 전화번호 가림 적용
- [ ] `docker compose up` 한 번으로 앱·DB 실행, README에 설치 절차 기재

## 11. 확인 필요

- 완료 예정일 기본값(접수일 기준 며칠 뒤로 할지)
- 영수증 프린터 기종·용지 폭
- 품목 목록과 기본 단가(매장 단가표)
- 서버 위치(클라우드 / 매장 PC)

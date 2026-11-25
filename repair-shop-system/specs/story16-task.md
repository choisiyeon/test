# [US-16] 로그인·로그아웃 — 개발 Task 설계서

| 항목 | 내용 |
| --- | --- |
| 에픽 | E0 공통·설정 |
| 스토리 | 사장님과 직원으로서 각자의 계정으로 로그인하고 로그아웃하고 싶다. 고객 개인정보와 매출 정보를 허가된 사람만 보고, 누가 어떤 작업을 했는지 남기기 위해서다. |
| 우선순위 / 릴리스 | 필수 / 1차 |
| 관련 화면 / 데이터 | S-01 로그인 / User, LoginLog(신규) |
| 선행 작업 | US-01(T1-05에서 기본 로그인·세션 구축) |
| 작성일 | 2026-11-25 |

---

## 1. 요약

- US-01에서 만든 기본 로그인(Auth.js Credentials)에 보안 기능을 보강
  - 연속 로그인 실패 시 일정 시간 잠금
  - 일정 시간 미사용 시 자동 로그아웃
  - 본인 비밀번호 변경, 최초 로그인 시 비밀번호 변경 강제
  - 로그인 기록(성공·실패) 저장
- 비활성(퇴사 등) 계정은 로그인 불가(계정 관리는 US-18)

---

## 2. 인수 조건

1. 올바른 아이디·비밀번호로 로그인하면 대시보드로 이동하고, 상단에 이름과 역할이 표시된다.
2. 아이디 또는 비밀번호가 틀리면 "아이디 또는 비밀번호가 올바르지 않습니다"만 표시된다(어느 쪽이 틀렸는지 알리지 않음).
3. 연속으로 정해진 횟수 이상 실패하면 일정 시간 로그인이 잠기고 안내가 표시된다.
4. 정해진 시간 동안 사용하지 않으면 자동으로 로그아웃되고 로그인 화면으로 이동한다.
5. [로그아웃]을 누르면 세션이 종료되고, 뒤로가기로도 이전 화면 내용을 볼 수 없다.
6. 관리자가 발급한 임시 비밀번호로 처음 로그인하면 비밀번호 변경 화면으로 이동하고, 변경 전에는 다른 화면을 쓸 수 없다.
7. 비활성 계정은 로그인할 수 없다.

---

## 3. 보안 기준 (제안값, 확정 필요)

| 항목 | 제안값 |
| --- | --- |
| 로그인 실패 잠금 | 연속 5회 실패 시 10분 잠금 |
| 자동 로그아웃 | 30분 미사용 |
| 비밀번호 규칙 | 8자 이상, 영문·숫자 조합 |
| 비밀번호 저장 | bcrypt 해시(비용 계수 12) |
| 세션 | 서버 서명 쿠키(HttpOnly, Secure, SameSite=Lax) |

---

## 4. 데이터 모델 변경

```prisma
model User {
  // ...US-01 항목
  failedCount        Int       @default(0)
  lockedUntil        DateTime?
  mustChangePassword Boolean   @default(true)
  passwordChangedAt  DateTime?
  lastLoginAt        DateTime?
}

model LoginLog {
  id        String   @id @default(cuid())
  loginId   String                 // 입력된 아이디(존재하지 않는 아이디 포함)
  userId    String?
  success   Boolean
  reason    String?                // WRONG_PASSWORD | LOCKED | INACTIVE | NO_USER
  ip        String?
  createdAt DateTime @default(now())
  @@index([createdAt])
}
```

---

## 5. API·처리 설계

| 메서드 | 경로 | 설명 | 권한 |
| --- | --- | --- | --- |
| POST | `/api/auth/callback/credentials` | 로그인(Auth.js) | 전체 |
| POST | `/api/auth/signout` | 로그아웃(Auth.js) | 로그인 사용자 |
| POST | `/api/me/password` | 본인 비밀번호 변경 | 로그인 사용자 |

### 로그인 처리(`authorize` 함수)

1. 사용자 조회 → 없으면 실패 기록(NO_USER), 동일 문구 반환
2. `active=false` → 실패(INACTIVE)
3. `lockedUntil > now` → 실패(LOCKED), 잠금 안내
4. 비밀번호 불일치 → failedCount+1, 기준 횟수 도달 시 lockedUntil 설정
5. 성공 → failedCount=0, lastLoginAt 기록, 세션에 `id, name, role, mustChangePassword` 저장

### 미들웨어

- 미로그인 → `/login`
- `mustChangePassword=true` → `/me/password` 외 접근 차단
- 관리자 전용 경로(`/sales`, `/settings`, `/api/sales/*` 등)는 role 확인
- 세션 최대 미사용 시간은 Auth.js `session.maxAge`·`updateAge`로 설정, 화면에서도 미사용 타이머로 로그아웃 처리
- 로그인 사용자 화면은 `Cache-Control: no-store`(로그아웃 후 뒤로가기 노출 방지)

---

## 6. 화면

```
┌ 수선집 관리 시스템 ─────────┐
│ 아이디   [            ]     │
│ 비밀번호 [            ]     │
│            [ 로그인 ]       │
│ (실패 안내 문구 영역)        │
└────────────────────────────┘
```

- 상단 메뉴 우측: "홍길동(직원) [비밀번호 변경] [로그아웃]"

---

## 7. 개발 Task 목록

| No | Task | 산출물 | 선행 |
| --- | --- | --- | --- |
| T16-01 | User 보안 필드, LoginLog 모델 추가, 마이그레이션 | 마이그레이션 | US-01 |
| T16-02 | `authorize` 보강(잠금·비활성·실패 기록) + 단위 테스트 | `lib/auth.ts` | T16-01 |
| T16-03 | 미들웨어 보강(비밀번호 변경 강제, 관리자 경로, no-store) | `middleware.ts` | T16-02 |
| T16-04 | 자동 로그아웃(세션 만료 설정 + 화면 미사용 타이머) | `components/idle-logout.tsx` | T16-02 |
| T16-05 | 비밀번호 변경 API·화면(규칙 검증, 현재 비밀번호 확인) | `/api/me/password`, `me/password` | T16-01 |
| T16-06 | 로그인 화면 문구·잠금 안내 | `(auth)/login` 수정 | T16-02 |
| T16-07 | 테스트(인수 조건 1~7) | `tests/us16.*` | T16-03~T16-06 |

## 8. 인수 조건 ↔ 테스트

| 인수 조건 | 테스트 |
| --- | --- |
| 1 | E2E: 로그인 후 대시보드, 이름·역할 표시 |
| 2 | 통합: 없는 아이디·틀린 비밀번호 응답 문구 동일 |
| 3 | 통합: 기준 횟수 실패 후 올바른 비밀번호도 거부 → 잠금 시간 경과 후 성공 |
| 4 | E2E: 시간 조작(fake timer)으로 미사용 후 로그인 화면 이동 |
| 5 | E2E: 로그아웃 후 보호 화면 접근 시 로그인 화면 |
| 6 | E2E: 임시 비밀번호 로그인 → 변경 화면 강제 |
| 7 | 통합: active=false 로그인 실패 |

## 9. 완료 기준

- [ ] 인수 조건 테스트 전체 통과
- [ ] 비밀번호 원문이 로그·DB 어디에도 저장되지 않음

## 10. 확인 필요

- 보안 기준 제안값(잠금 횟수·시간, 자동 로그아웃 시간, 비밀번호 규칙)
- 로그인 기록 보관 기간

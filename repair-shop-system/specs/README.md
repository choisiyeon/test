# 수선집 관리 시스템 — 스토리별 개발 Task 설계서

- 작성일: 2026-11-25
- 기술 스택: Next.js(TypeScript) + PostgreSQL + Prisma, Docker Compose(상세는 story1-task.md 2절)
- 스토리 20개(필수 16, 선택 4) / 각 설계서 끝의 '확인 필요' 항목은 매장 확인 후 확정 필요

| 에픽 | 스토리 | 설계서 | 사용자 | 우선순위 | 릴리스 |
| --- | --- | --- | --- | --- | --- |
| E1 접수·진행 | US-01 수선 접수 등록 | [story1-task.md](story1-task.md) | 직원 | 필수 | 1차 |
| | US-02 작업 상태 변경 | [story2-task.md](story2-task.md) | 직원 | 필수 | 1차 |
| | US-03 의류 사진 첨부 | [story3-task.md](story3-task.md) | 직원 | 선택 | 2차 |
| | US-04 접수 목록 검색·기한 경과 확인 | [story4-task.md](story4-task.md) | 직원 | 필수 | 1차 |
| E2 고객관리 | US-05 고객 등록·수정 | [story5-task.md](story5-task.md) | 직원 | 필수 | 1차 |
| | US-06 고객 검색·수선 이력 조회 | [story6-task.md](story6-task.md) | 직원 | 필수 | 1차 |
| | US-07 고객 정보 삭제 | [story7-task.md](story7-task.md) | 사장님 | 필수 | 1차 |
| | US-08 미수령 의류 조회 | [story8-task.md](story8-task.md) | 직원 | 필수 | 1차 |
| E3 완료 알림 | US-09 완료 알림 자동 발송 | [story9-task.md](story9-task.md) | 직원 | 필수 | 1차 |
| | US-10 알림 발송 이력 조회 | [story10-task.md](story10-task.md) | 직원 | 필수 | 1차 |
| | US-11 미수령 재안내 발송 | [story11-task.md](story11-task.md) | 직원 | 선택 | 2차 |
| E4 매출·정산 | US-12 결제·출고 처리 | [story12-task.md](story12-task.md) | 직원 | 필수 | 1차 |
| | US-13 결제 취소·환불 | [story13-task.md](story13-task.md) | 사장님 | 필수 | 1차 |
| | US-14 일·월 매출 조회 | [story14-task.md](story14-task.md) | 사장님 | 필수 | 1차 |
| | US-15 매출 엑셀 내려받기 | [story15-task.md](story15-task.md) | 사장님 | 선택 | 2차 |
| E0 공통·설정 | US-16 로그인·로그아웃 | [story16-task.md](story16-task.md) | 전체 | 필수 | 1차 |
| | US-17 대시보드 | [story17-task.md](story17-task.md) | 전체 | 필수 | 1차 |
| | US-18 직원 계정 관리 | [story18-task.md](story18-task.md) | 사장님 | 필수 | 1차 |
| | US-19 수선 품목·기본 단가 설정 | [story19-task.md](story19-task.md) | 사장님 | 필수 | 1차 |
| | US-20 알림 문구 설정 | [story20-task.md](story20-task.md) | 사장님 | 선택 | 2차 |

## 권장 개발 순서 (1차)

US-01 → US-16 → US-02 → US-04 → US-05 → US-06 → US-19 → US-12 → US-13 → US-08 → US-09 → US-10 → US-07 → US-14 → US-18 → US-17

# Buildcare AI

건축물 하자 사진을 AI 로 1차 분석하고, 현장 기록 · 전문가 검증 · 건물 이력 · 유지보수 우선순위 · 정기점검 · 전문가 연결까지 이어지는 **건축물 유지관리 플랫폼**입니다.

- 운영 주소: https://p3.sumzip.com
- 설계 산출물: [`Design/`](./Design) · 기능 명세: [`specs/001-buildcare-ai-platform/`](./specs/001-buildcare-ai-platform)

> AI 분석 결과와 우선순위는 **1차 참고용**이며, 진단 책임 고지와 함께만 표시됩니다.

---

## 주요 기능

| # | 기능 | 사용자 | 내용 |
|:--:|---|---|---|
| 1 | 하자 사진 AI 분석 | 비회원 · 일반 사용자 | 사진 + 설명 → 원인(순위) · 대응방안 · 위험도. 비회원 무료 체험 3회 |
| 2 | 이용권 구매 | 일반 사용자 | 건별 · 월 구독 이용권 결제 (mock 결제 연동) |
| 3 | 현장 기록 | 시설관리자 | 위치 · 하자 종류 · 보수 상태 · 사진 기록, AI 결과 연결 (모바일 360px 지원) |
| 4 | 전문가 검증 | 전문가 | AI 결과 판정(일치 / 불일치 / 판정 불가), 추가 자료 요청, 판정 판본 관리 |
| 5 | 건물 이력 | 건물관리자 | 내부 · 외부(BMS) 이력 시간순 통합, 반복 하자 패턴 탐지 |
| 6 | 유지보수 우선순위 | 기업 관리자 | 권한 범위 건물의 우선순위 산출 + 항목별 근거 이력 |
| 7 | 정기점검 | 건물 · 시설관리자 | 점검 일정 상태 5종, 도래 · 지연 감시 배치와 알림 |
| 8 | 전문가 연결 | 건물관리자 · 전문가 | 위험 통지 → 분야 · 일정 → 후보 → 동의 → 수락 |
| R | RBAC | 운영자 · 기업 관리자 | 권한 코드 기반 접근 제어, 권한 요청 승인, 다중 역할 메뉴 합집합 |

조건을 충족하지 못하면 토스트 · 모달 대신 **차단 블록(GateBlock)** 으로 사유와 다음 행동을 안내합니다. 게이트 판정은 DB 뷰가 단일 원천입니다.

## 기술 스택

| 영역 | 사용 기술 |
|---|---|
| Backend | TypeScript 5 · Node.js 20 LTS · Express 4 · mysql2 · node-cron · nodemailer · sharp |
| Frontend | Vue 3 (Composition API) · Vite 5 · Pinia · Vue Router 4 |
| Database | 원격 MariaDB 10.6+ (테이블 54 · 뷰 16 · 트리거 15) |
| AI | Google Gemini (`@google/genai`) — 개발 시 `AI_PROVIDER=mock` |
| Test | Vitest · Supertest · Playwright (+ axe 접근성) |
| 배포 | pm2 단일 프로세스 + nginx HTTPS 종단 (Docker 미사용) |

## 프로젝트 구조

```text
backend/     Express API
  src/modules/     도메인 모듈 (analysis, auth, building, case, entitlement, expert,
                   notification, permission, priority, record, reference, schedule, verification, admin)
  src/db/          마이그레이션 · 시드
  src/gates/       트리거 SIGNAL → 게이트 사상 (sqlErrorMap.ts)
  src/auth/        RBAC 권한 코드 (permissions.ts)
  src/adapters/    AI · 결제 · BMS · 메일 어댑터 (mock / 실제)
  src/jobs/        배치 (정기점검 감시, 전문가 무응답, BMS 동기화)
frontend/    Vue SPA
  src/views/       화면 S1~S8B
  src/components/  공통 컴포넌트 C1~C8
Design/      설계서 — 유스케이스 · SD_01 프로세스 · SD_02 UI/UX · SD_03 DB · SD_04 아키텍처 · DDL · 스타일가이드
specs/       spec · plan · research · data-model · contracts/openapi.yaml · quickstart · tasks
deploy/      pm2 설정 · 배포 안내
```

## 시작하기

### 1. 준비물

- Node.js 20 LTS 이상, npm 10 이상
- 원격 MariaDB 10.6 이상 접속 정보 (관리자에게 받음)
- (선택) Gemini API 키 — 없으면 mock 분석으로 동작
- (운영) pm2

### 2. 설치

```bash
git clone <저장소 URL> buildcare-ai
cd buildcare-ai
cd backend  && npm install && cd ..
cd frontend && npm install && cd ..
npx --prefix frontend playwright install chromium   # E2E 용 (1회)
```

### 3. 환경 변수

`backend/.env.example` 을 `backend/.env` 로 복사해 값을 채웁니다. **`.env` 는 커밋하지 않습니다.**

```dotenv
PORT=3000                 # 로컬 개발. 운영은 9503
COOKIE_SECURE=0           # http://localhost 개발 시 0, 운영(HTTPS)은 1
JWT_SECRET=<32바이트 이상 무작위>
FILE_URL_SECRET=<32바이트 이상 무작위>

DB_HOST=...  DB_PORT=...  DB_USER=...  DB_PASSWORD=...  DB_NAME=...

AI_PROVIDER=mock          # mock | gemini
GEMINI_API_KEY=
PAYMENT_PROVIDER=mock
BMS_PROVIDER=mock         # mock | rest
MAIL_TRANSPORT=console    # console | smtp

SEED_DEMO_PASSWORD=<시연 계정 공통 비밀번호>
```

전체 키 목록은 [`quickstart.md`](./specs/001-buildcare-ai-platform/quickstart.md#3-환경-변수--backendenv) 를 참고하세요.

### 4. 데이터베이스

DB 작업은 Node 스크립트로만 합니다 (`mysql` 클라이언트 · Docker 불필요).

```bash
cd backend
npm run db:check                 # 접속 · 버전 · 권한 · 기존 객체 확인
npm run db:migrate               # 001 설계 DDL → 002 앱 확장 → 003 RBAC (재실행 안전)
npm run db:seed:dev -- --reset   # 시연 데이터 (날짜는 실행일 기준 상대값)
```

시연 계정(`general@dev.local`, `facility@dev.local`, `building@dev.local`, `enterprise@dev.local`, `expert@dev.local`, `operator@dev.local` 등)의 비밀번호는 `SEED_DEMO_PASSWORD` 값입니다. 계정별 시연 동선은 [`backend/docs/seed-walkthrough.md`](./backend/docs/seed-walkthrough.md) 에 있습니다.

### 5. 실행

```bash
# 로컬 개발
cd backend  && npm run dev       # http://localhost:3000
cd frontend && npm run dev       # http://localhost:5173 (/api · /files → 3000 프록시)
```

```bash
# 운영 (p3.sumzip.com → :9503)
cd backend && npm run build && cd ../frontend && npm run build && cd ..
pm2 start deploy/pm2.config.cjs && pm2 save      # 갱신 시: pm2 restart buildcare
curl -s https://p3.sumzip.com/api/health
```

배포 상세는 [`deploy/README.md`](./deploy/README.md) 를 참고하세요.

## 테스트

```bash
cd backend
npm run lint && npm test              # 단위 테스트 35 (DB 불필요)
npm run test:integration              # 통합 테스트 77 (원격 DB, 표식 데이터 생성 후 정리)

cd ../frontend
npm run lint && npm run check:style && npm test      # 스타일가이드 검사 + 컴포넌트 테스트
BASE_URL=https://p3.sumzip.com npm run test:e2e      # Playwright E2E 34 (360px · 데스크톱, axe)
```

## 수동 배치

```bash
cd backend
npm run job:p0            # 정기점검 도래 · 지연 감시
npm run job:no-response   # 전문가 무응답 판정
npm run job:bms-sync      # 건물관리시스템 동기화
```

## 개발 규칙

- Docker 를 쓰지 않으며, DB 는 원격 MariaDB 만 사용합니다.
- `Design/buildcare_ddl.sql` 은 수정하지 않고, 확장은 `002_app_extensions.sql` 에 추가합니다.
- 게이트 · 상태 판정은 DB 뷰(`v_access_check`, `v_analysis_eligibility`, `v_pattern_gate`, `v_schedule_status`, `v_expert_request_status`)로만 하며 코드에서 재계산하지 않습니다.
- 인가는 `requirePermission(<권한 코드>)` + `requireBuildingAccess` 두 층으로만 하며, 역할 이름으로 분기하지 않습니다.
- 개인정보는 `v_user_masked` 마스킹 필드로만 응답합니다.
- 비밀번호 · API 키는 `backend/.env` 외에 적지 않습니다.

자세한 규칙은 [`CLAUDE.md`](./CLAUDE.md) 를 참고하세요.

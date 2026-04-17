# SRS v1.3 기술 스택 전환 계획서

**문서 ID:** PLAN-SRS-001
**작성일:** 2026-04-16
**대상 문서:** `SRS-Drafts/SRS_v0_1_Opus.md` (현 Revision 1.2 → **목표 Revision 1.3**)
**목적:** MSA 기반 아키텍처를 Next.js 단일 풀스택 모놀리스로 전환하면서, MVP 핵심 가치 전달이 훼손되지 않음을 검증하고 SRS 수정 작업을 체계화한다.

---

## 1. 기술 스택 전환 개요

### 1.1 목표 기술 스택 (C-TEC 제약사항)

| ID | 제약사항 | 범위 |
|---|---|---|
| C-TEC-001 | 모든 서비스는 **Next.js (App Router)** 기반 단일 풀스택 프레임워크로 구현 | 시스템 내부 |
| C-TEC-002 | 서버 측 로직은 **Server Actions / Route Handlers**로 구현, 별도 백엔드 서버 없음 | 시스템 내부 |
| C-TEC-003 | DB는 **Prisma + SQLite(로컬) / Supabase PostgreSQL(배포)** 사용 | 시스템 내부 |
| C-TEC-004 | UI/스타일링은 **Tailwind CSS + shadcn/ui** 사용 | 시스템 내부 |
| C-TEC-005 | LLM 오케스트레이션은 **Vercel AI SDK**로 Next.js 내부에서 구현 | 시스템 외부 |
| C-TEC-006 | LLM 호출은 **Google Gemini API** 기본, 환경 변수로 모델 교체 가능 | 시스템 외부 |
| C-TEC-007 | 배포/인프라는 **Vercel** 단일화, Git Push 자동 배포 | 시스템 외부 |

### 1.2 아키텍처 전환 요약

```
[AS-IS: SRS v1.2]                        [TO-BE: SRS v1.3]
┌─────────────────────┐                  ┌─────────────────────────────┐
│  Mobile Web (SPA)   │                  │  Next.js App Router         │
│  React/Next.js      │                  │  (SSR + CSR + RSC)          │
├─────────────────────┤                  │  ┌─────────────────────┐    │
│  API Gateway        │                  │  │ Server Components   │    │
│  (Rate Limit, Auth) │                  │  │ (SSR, Data Fetch)   │    │
├─────────────────────┤                  │  ├─────────────────────┤    │
│  Super-Calc Engine  │ ← 별도 서비스     │  │ Route Handlers      │    │
│  Badge Engine       │ ← 별도 서비스     │  │ /app/api/v1/...     │    │
│  Search Service     │ ← 별도 서비스     │  │ (모든 서버 로직 통합) │    │
│  OG Image Generator │ ← 별도 서비스     │  ├─────────────────────┤    │
│  Report Service     │ ← 별도 서비스     │  │ Server Actions      │    │
│  Reward Service     │ ← 별도 서비스     │  │ (폼 처리, DB CRUD)  │    │
│  Notification Svc   │ ← 별도 서비스     │  ├─────────────────────┤    │
├─────────────────────┤                  │  │ Prisma ORM          │    │
│  Redis Cache        │                  │  │ + Vercel KV(캐시)   │    │
│  Product DB         │                  │  └─────────────────────┘    │
│  S3 Storage         │                  └──────────────┬──────────────┘
└─────────────────────┘                                 │
                                         ┌──────────────┴──────────────┐
                                         │  Supabase (PostgreSQL)      │
                                         │  + Supabase Storage (파일)  │
                                         └─────────────────────────────┘
```

---

## 2. SRS 섹션별 수정 작업 명세

> 각 작업 항목의 상태는 `[ ]`(미착수), `[~]`(진행 중), `[x]`(완료)로 관리한다.

### 2.1 §1.2.3 Constraints — 기술 제약사항 추가

- [ ] **작업:** CON-7 ~ CON-13으로 C-TEC-001 ~ C-TEC-007을 Constraints 테이블에 추가
- **변경 유형:** 행 추가
- **영향 범위:** 제약사항 테이블만

**추가될 내용:**

| ID | 제약사항 | 유형 | 근거 |
|---|---|---|---|
| CON-7 | 모든 서비스는 Next.js (App Router) 기반 단일 풀스택으로 구현. 프론트/백 분리 금지 | 기술/아키텍처 | C-TEC-001 |
| CON-8 | 서버 로직은 Server Actions 또는 Route Handlers로 구현. 별도 백엔드 서버 금지 | 기술/아키텍처 | C-TEC-002 |
| CON-9 | DB는 Prisma ORM + SQLite(로컬) / Supabase PostgreSQL(배포) 사용 | 기술/데이터 | C-TEC-003 |
| CON-10 | UI/스타일링은 Tailwind CSS + shadcn/ui 사용 | 기술/프론트엔드 | C-TEC-004 |
| CON-11 | LLM 오케스트레이션은 Vercel AI SDK 사용, 별도 Python 서버 금지 | 기술/AI | C-TEC-005 |
| CON-12 | LLM은 Google Gemini API 기본, 환경 변수로 모델 교체 가능 | 기술/AI | C-TEC-006 |
| CON-13 | 배포는 Vercel 플랫폼 단일화, Git Push 자동 배포 | 기술/인프라 | C-TEC-007 |

### 2.2 §1.3 Definitions — 신규 용어 추가

- [ ] **작업:** 다음 용어를 Definitions 테이블에 추가

| 용어 | 정의 |
|---|---|
| **Next.js App Router** | React 기반 풀스택 프레임워크의 파일 시스템 기반 라우팅 시스템. 서버 컴포넌트(RSC)와 클라이언트 컴포넌트를 혼용 가능 |
| **Server Actions** | Next.js에서 서버 측 함수를 클라이언트 컴포넌트에서 직접 호출하는 RPC 패턴. 폼 처리 및 DB 뮤테이션에 사용 |
| **Route Handlers** | Next.js App Router에서 RESTful API 엔드포인트를 구현하는 서버 측 핸들러 (`/app/api/...`) |
| **Prisma** | 타입 안전 ORM. 스키마 정의 → 클라이언트 자동 생성. SQLite/PostgreSQL 등 다중 DB 지원 |
| **Supabase** | 오픈소스 Firebase 대안. PostgreSQL DB, 인증, 스토리지, 실시간 구독을 제공하는 BaaS |
| **Vercel** | Next.js 최적화 서버리스 배포 플랫폼. Edge Functions, KV Store, Cron Jobs 내장 |
| **Vercel KV** | Vercel 플랫폼 내장 Key-Value 스토어. Redis 호환 API. 캐시 용도 |
| **Vercel AI SDK** | LLM 호출을 표준화한 TypeScript SDK. 스트리밍 응답, 다중 프로바이더 지원 |
| **shadcn/ui** | Radix UI 기반의 복사-가능한 컴포넌트 라이브러리. Tailwind CSS로 스타일링 |

### 2.3 §3.1.1 폴백 전략 — 인프라 컴포넌트 치환

- [ ] **작업:** 테이블 내 Redis/S3 참조를 Vercel/Supabase 생태계로 변경

| 현 SRS 표현 | 변경 후 |
|---|---|
| `Redis 캐시(TTL 15min)` | `Vercel KV(TTL 15min) 또는 DB 캐시 테이블` |
| `S3 Storage` | `Supabase Storage` |

### 2.4 §3.3 API Overview — 내부 API 구현 방식 변경

- [ ] **작업:** 내부 API 설명에서 "내부 (REST)" 독립 서비스를 "Next.js Route Handler" 로 변경
- **핵심:** 엔드포인트 경로(`/api/v1/...`)는 동일하게 유지. 구현 주체만 변경.

| 현 SRS | 변경 후 |
|---|---|
| `Super-Calc 내부 API \| 내부 (REST)` | `Super-Calc API \| Route Handler (/app/api/v1/compare)` |
| `Badge 내부 API \| 내부 (REST)` | `Badge API \| Route Handler (/app/api/v1/badges)` |

### 2.5 §3.6 Component Diagram — 전면 재작성 ⭐

- [ ] **작업:** MSA 7개 서비스 + API Gateway 구조를 **Next.js 단일 앱 내부 모듈 구조**로 전면 재작성

**변경 전 (AS-IS):**
- Client Layer → API Gateway → Backend Microservices (7개) → Data & Cache Layer → External Systems
- 별도 Redis, S3, CloudWatch/Datadog/PagerDuty

**변경 후 (TO-BE) 구조:**

```
Next.js App (단일 배포 단위)
├── /app                           ← App Router (페이지 + 레이아웃)
│   ├── /app/api/v1/compare        ← Route Handler: Super-Calc 로직
│   ├── /app/api/v1/badges         ← Route Handler: Badge 판정 로직
│   ├── /app/api/v1/search         ← Route Handler: 검색/자동완성
│   ├── /app/api/v1/share/og-image ← Route Handler: OG 이미지 생성 (@vercel/og)
│   ├── /app/api/v1/reports        ← Route Handler: 오류 제보 접수
│   ├── /app/api/v1/product-requests ← Route Handler: 미등록 제품 요청
│   ├── /app/api/v1/exchange-rate  ← Route Handler: 환율 캐시 조회
│   └── /app/api/cron/sync-prices  ← Vercel Cron: 일 1회 가격 배치 수집
├── /lib
│   ├── /lib/adapters/             ← 채널 어댑터 (iHerb, Coupang, Naver)
│   ├── /lib/calc/                 ← 단가 정규화/정렬 비즈니스 로직
│   ├── /lib/badge/                ← 뱃지 판정 로직
│   └── /lib/ai/                   ← Vercel AI SDK + Gemini 통합
├── /components                    ← shadcn/ui 기반 UI 컴포넌트
├── /prisma
│   └── schema.prisma              ← DB 스키마 (모든 엔터티 정의)
└── /actions                       ← Server Actions (폼 처리, DB 뮤테이션)
```

- [ ] **작업:** 컴포넌트 테이블도 모놀리스 모듈 구조로 재작성

**변경 후 컴포넌트 매핑:**

| 컴포넌트 | 유형 | 구현 위치 | 책임 | SLA/제약 |
|---|---|---|---|---|
| **Next.js App** | 풀스택 앱 | Vercel (단일 배포) | SSR/CSR 통합, 라우팅, 인증 | LCP <= 2,500ms |
| **Super-Calc Module** | Route Handler | `/app/api/v1/compare` | 멀티채널 가격 조회, 환율 적용, 1일 단가 정규화 | p95 <= 2,000ms |
| **Badge Module** | Route Handler | `/app/api/v1/badges` | 식약처 공전 기반 뱃지 판정, 일상어 번역 | p95 <= 1,000ms |
| **Search Module** | Route Handler | `/app/api/v1/search` | 성분/제품 검색, 자동완성 | p95 <= 1,000ms |
| **OG Generator** | Route Handler | `/app/api/v1/share/og-image` | `@vercel/og`(Satori) 기반 동적 OG 이미지 생성 | p95 <= 1,500ms |
| **Report Module** | Server Action | `/actions/report.ts` | 오류 제보 접수, 스팸 필터링, Prisma CRUD | p95 <= 3,000ms |
| **Reward Module** | Server Action | `/actions/reward.ts` | 제보 보상 포인트/배지 지급 | 수정 후 1시간 이내 |
| **Notification** | Server Action | `/actions/notify.ts` | 이메일 알림 발송 (Resend API) | 트리거 후 1시간 이내 |
| **Price Sync Cron** | Vercel Cron | `/app/api/cron/sync-prices` | 일 1회 외부 API 배치 호출, PRICE_SNAPSHOT 갱신 | Vercel Cron (일 1회) |
| **Channel Adapters** | 라이브러리 | `/lib/adapters/` | iHerb/쿠팡/네이버 API 호출 추상화 (전략 패턴) | 채널 추가 시 어댑터 1개 추가 |
| **Supabase DB** | BaaS (PostgreSQL) | Supabase 클라우드 | 제품, 성분, 가격, 제보 데이터 영구 저장 | RPO <= 24h (Supabase 자동 백업) |
| **Vercel KV** | KV Store | Vercel 내장 | 뱃지 캐시 (TTL 24h), 환율 캐시 (TTL 15min) | Free: 30K 요청/월 |
| **Supabase Storage** | Object Storage | Supabase 클라우드 | 라벨 이미지, OG 이미지 | Free: 1GB |

### 2.6 §4.2.4 Cost (REQ-NF-019/020) — 비용 체계 구체화

- [ ] **작업:** Vercel + Supabase Free Tier 비용 체계에 맞게 구체화

**Vercel + Supabase Free Tier 비용 분석:**

| 서비스 | Free Tier 한도 | MVP 추정 사용량 | 초과 시 비용 |
|---|---|---|---|
| Vercel Hobby | 100GB 대역폭, 100h Serverless | MAU 2,200명 기준 충분 | Pro $20/월 |
| Supabase Free | 500MB DB, 1GB Storage, 50K Auth MAU | 300~500 제품 기준 충분 | Pro $25/월 |
| Vercel KV | 30K 요청/월, 256MB | 환율 + 뱃지 캐시 충분 | Pro에 포함 |
| Vercel Cron | 일 1회 (Hobby) | 가격 배치 수집 1회 적합 | Pro 시 무제한 |
| Resend (이메일) | 100통/일, 3,000통/월 | 제보 알림 용도 충분 | Free로 충분 |
| **합계** | **$0/월** | — | **최대 $45/월 (≈ 6만 원)** |

> MVP 비용 한도 10만 원 대비 **충분한 여유** 확인.

### 2.7 §4.2.5 Monitoring (REQ-NF-021~023) — 도구 치환

- [ ] **작업:** 모니터링 도구를 Vercel 생태계로 치환

| 현 SRS 도구 | 변경 후 도구 | 비용 |
|---|---|---|
| CloudWatch/Datadog | **Vercel Analytics** (Web Vitals) + **Vercel Logs** (Runtime Logs) | Free 포함 |
| PagerDuty | **Vercel Log Drain → Slack Webhook** (실시간 에러 알림) | Free |
| Mixpanel/Amplitude | **Mixpanel Free** (유지) 또는 Vercel Analytics로 통합 | Free |

### 2.8 §4.2.6 Scalability (REQ-NF-024) — 아키텍처 패턴 변경

- [ ] **작업:** 플러그인 아키텍처를 모놀리스 내 전략 패턴으로 재정의

**변경 전:** "채널 어댑터 모듈만 추가하여 확장 가능한 플러그인 아키텍처"
**변경 후:** "신규 채널 추가 시 `/lib/adapters/` 디렉토리에 `ChannelAdapter` 인터페이스를 구현하는 모듈 1개만 추가. 기존 코드 변경 없이 환경 변수(`ENABLED_CHANNELS`)에 채널 키를 추가하여 활성화."

### 2.9 §6.1.2 내부 API — 구현 주체 변경

- [ ] **작업:** "별도 서비스" → "Next.js Route Handler" 로 구현 주체 컬럼 변경
- 엔드포인트 경로(`/api/v1/...`)는 **동일 유지**하여 API 계약 호환성 보존

### 2.10 §6.2 Entity & Data Model — Prisma 스키마 매핑 주석 추가

- [ ] **작업:** 각 엔터티 테이블 상단에 Prisma 모델명 매핑 주석 추가
- 예: `#### 6.2.1 PRODUCT (제품) — Prisma Model: Product`
- 타입 표기: STRING → `String`, FLOAT → `Float`, DATETIME → `DateTime` (Prisma 표기법)

### 2.11 §6.2.9 Class Diagram — Service 클래스 재매핑

- [ ] **작업:** Service 스테레오타입의 클래스를 모듈/함수 기반으로 재매핑

| 현 SRS | 변경 후 |
|---|---|
| `SuperCalcEngine <<Service>>` | `SuperCalcModule <<RouteHandler>>` (`/app/api/v1/compare`) |
| `BadgeEngine <<Service>>` | `BadgeModule <<RouteHandler>>` (`/app/api/v1/badges`) |
| `ReportService <<Service>>` | `ReportActions <<ServerAction>>` (`/actions/report.ts`) |

### 2.12 Sequence Diagrams (§3.4, §6.3) — 참여자명 갱신

- [ ] **작업:** 시퀀스 다이어그램의 `participant` 명칭을 Next.js 모듈명으로 변경
- `participant API as Super-Calc API` → `participant API as Route Handler (/api/v1/compare)`
- `participant Auth as 인증 서비스` → `participant Auth as Supabase Auth`
- 전체 흐름 로직은 **변경 없음** (동일한 호출 순서 유지)

### 2.13 신규 섹션 — LLM 통합 (미결)

- [ ] **결정 필요:** LLM(Vercel AI SDK + Gemini)을 MVP에 포함할 기능 범위 확정

**가능 시나리오 (확정 시 REQ-FUNC 추가 필요):**

| 시나리오 | MVP 포함 여부 | 영향 REQ |
|---|---|---|
| A. 성분 전문 용어 → 일상어 자동 번역 | ❓ 미결 | REQ-FUNC-013 구현 방식 변경 |
| B. 자연어 검색 의도 해석 | ❓ 미결 | REQ-FUNC-030 보강 |
| C. 라벨 OCR 결과 정규화 | ❓ 미결 | CP-1 대안 소스 2번 보강 |
| D. Phase 2 인프라 준비만 | ❓ 미결 | CON-11/12 유지, REQ 추가 없음 |

> 사용자 확인 필요. 확정 전까지 **시나리오 D (인프라 준비만)** 로 가정하고 진행.

---

## 3. MVP 핵심 가치 전달 훼손 검토

### 3.1 검토 프레임워크

PRD에서 정의된 **4대 Value Proposition**과 **4개 핵심 페르소나**를 기준으로, 기술 스택 전환 후에도 사용자가 체감하는 핵심 가치가 동일하게 전달되는지 검증한다.

| Value Proposition | 핵심 지표 | 판정 기준 |
|---|---|---|
| **VP-1.** 실시간 1일 단가 자동 계산 | 계산 소요 시간 <= 5초 | 전환 후에도 동일 SLA 충족 여부 |
| **VP-2.** 광고 제로 팩트체크 | 마케팅 콘텐츠 0건 보장 | UI/UX 변경 없이 보존 여부 |
| **VP-3.** 1-Tap 카카오톡 공유 | 공유 카드 생성 <= 1.5초 | 카카오 JS SDK 호출 방식 동일 여부 |
| **VP-4.** 데이터 투명성 (출처/오류 SLA) | 2클릭 출처, 48h 수정 SLA | DB CRUD 방식 변경이 SLA에 영향 여부 |

### 3.2 페르소나별 핵심 UX 보존 분석

#### C1 한정훈 (가성비 최적화 직구족) — VP-1 중심

```
[사용자 여정] 영양소 검색 → 1일 단가 비교 → 실지불가 확인 → 제휴 링크 구매
```

| UX 단계 | 현 SRS 구현 | 변경 후 구현 | 차이 |
|---|---|---|---|
| 검색 입력 | React SPA → API Gateway → Search Service | Next.js RSC → Route Handler (`/api/v1/search`) → Prisma | **사용자 체감 동일.** 내부 호출 경로만 단축됨. 모놀리스 내 함수 호출이므로 네트워크 홉 감소 → **오히려 빨라짐** |
| 단가 계산 | API Gateway → Super-Calc Engine (별도 서비스) → 3채널 병렬 조회 | Route Handler (`/api/v1/compare`) → `Promise.allSettled()` 3채널 병렬 조회 | **사용자 체감 동일.** 병렬 외부 API 호출 패턴 동일. Vercel Serverless 실행 시간(10초 Free, 300초 Pro)이 2초 SLA 대비 충분 |
| 결과 렌더링 | 클라이언트 사이드 렌더링 (CSR) | Next.js **서버 컴포넌트(RSC)** or CSR 선택 가능 | **개선 가능.** SSR 활용 시 초기 로딩 속도(LCP) 개선 |
| 제휴 링크 클릭 | 딥링크 URL → 커머스 이동 | 동일 (딥링크 URL은 외부 API에서 제공) | **변경 없음** |

> **판정: ✅ VP-1 보존됨.** 단가 계산의 핵심인 3채널 병렬 API 조회 + 환율 적용 + 정규화 로직은 구현 위치만 변경(독립 서비스 → Route Handler)되며, 비즈니스 로직과 SLA는 동일하다.

---

#### C2 박소연 (건강 계기 진입자) — VP-2 중심

```
[사용자 여정] 제품 상세 → 팩트체크 뱃지 확인 → 일상어 번역 읽기 → 구매 결정
```

| UX 단계 | 현 SRS 구현 | 변경 후 구현 | 차이 |
|---|---|---|---|
| 뱃지 로드 | API Gateway → Badge Engine → Redis 캐시/식약처 API | Route Handler (`/api/v1/badges`) → **Vercel KV 캐시** / Prisma (로컬 DB) | **사용자 체감 동일.** 캐시 미스 시 로컬 DB(Prisma)에서 식약처 데이터 조회. 외부 API 호출 최소화 → **오히려 안정적** |
| 일상어 번역 | Badge Engine 내부 매핑 테이블 | Prisma `INGREDIENT.common_name` 필드 조회 | **변경 없음.** 데이터 소스 동일 |
| 마케팅 0건 | UI 렌더링 규칙 (프론트엔드) | shadcn/ui 컴포넌트로 동일 규칙 적용 | **변경 없음.** UI 프레임워크와 무관한 비즈니스 규칙 |

> **판정: ✅ VP-2 보존됨.** 뱃지 판정 로직과 마케팅 차단 규칙은 프레임워크에 독립적인 비즈니스 로직이다. 캐시를 Redis → Vercel KV로 치환했으나 TTL 기반 캐시 동작은 동일하다.

---

#### A2 정수빈 (트렌드 추종 탐색자) — VP-3 중심

```
[사용자 여정] 팩트체크 → "카톡 공유" 탭 → OG 카드 생성 → 카카오톡 전송
```

| UX 단계 | 현 SRS 구현 | 변경 후 구현 | 차이 |
|---|---|---|---|
| OG 이미지 생성 | OG Image Generator (별도 서비스) | `@vercel/og` (Satori) — Next.js **Edge Runtime에서 네이티브 지원** | **개선.** 별도 서비스 대신 Edge에서 직접 생성하므로 네트워크 홉 제거. 응답 속도 개선 기대 |
| 카카오 전송 | 클라이언트 → Kakao JS SDK `sendDefault()` | **동일** — 클라이언트 컴포넌트에서 Kakao JS SDK 호출 | **변경 없음.** 카카오 SDK는 클라이언트 사이드 실행이므로 서버 아키텍처와 무관 |
| 수신자 랜딩 | 카카오 웹뷰 → 별도 랜딩 페이지 | 카카오 웹뷰 → **Next.js SSR 랜딩 페이지** (더 빠른 초기 로드 가능) | **개선 가능.** SSR 활용 시 카카오 인앱 브라우저에서의 LCP 개선 |

> **판정: ✅ VP-3 보존됨 + 개선 가능.** OG 이미지 생성이 Edge 네이티브로 변경되어 오히려 성능 개선이 기대된다. 카카오 JS SDK 호출 방식은 클라이언트 사이드이므로 서버 아키텍처 변경과 완전히 독립적이다.

---

#### E2 김도현 (극단적 신뢰 실패자) — VP-4 중심

```
[사용자 여정] 출처 확인 → 오류 발견 → 제보 제출 → 48h 내 수정 확인 → 보상 수령
```

| UX 단계 | 현 SRS 구현 | 변경 후 구현 | 차이 |
|---|---|---|---|
| 출처 확인 | 아코디언 UI → 식약처 URL/라벨 이미지 | 동일 UI (shadcn/ui Accordion) → Prisma 조회 + **Supabase Storage** (이미지) | **사용자 체감 동일.** 스토리지만 S3 → Supabase Storage로 변경 |
| 오류 제보 | API Gateway → Report Service → DB 저장 | **Server Action** (`/actions/report.ts`) → Prisma `create()` | **사용자 체감 동일.** 폼 제출 → DB 저장 흐름 동일. Server Action이 form progressive enhancement 지원하여 **오히려 UX 개선** |
| 48h SLA | Report Service → Notification Service (Push) | Server Action → **Resend 이메일 API** (Push 대체) | 🟡 **Push → 이메일로 변경.** 단, E2 페르소나는 데이터 정확도에 관심이 있으므로 알림 채널보다 **수정 SLA 자체**가 핵심 가치. SLA 48h 기준은 불변 |
| 보상 수령 | Reward Service (별도) | Server Action (`/actions/reward.ts`) → Prisma | **사용자 체감 동일** |

> **판정: ✅ VP-4 보존됨.** 유일한 차이는 Push 알림 → 이메일 알림 변경이나, E2의 핵심 가치는 "48시간 내 수정" SLA 자체이며 알림 채널은 부차적이다. 이메일 알림으로도 목적 충족.

### 3.3 종합 판정 테이블

| VP | 핵심 가치 | 보존 여부 | 비고 |
|---|---|---|---|
| **VP-1** | 5초 내 1일 단가 자동 계산 | ✅ **보존** | 병렬 API 호출 패턴 동일, 네트워크 홉 감소로 개선 가능 |
| **VP-2** | 광고 제로 팩트체크 뱃지 | ✅ **보존** | 비즈니스 규칙은 프레임워크 독립적 |
| **VP-3** | 1-Tap 카카오톡 공유 | ✅ **보존 + 개선** | OG 이미지가 Edge 네이티브로 변경되어 성능 개선 |
| **VP-4** | 데이터 투명성 + 48h SLA | ✅ **보존** | Push → 이메일 변경은 부차적. SLA 자체는 불변 |

> **결론: 4대 핵심 가치 전달이 모두 보존된다.** 기술 스택 전환은 사용자에게 **투명(transparent)** 하며, 일부 UX는 오히려 개선(SSR LCP, Edge OG 생성, 네트워크 홉 감소)된다.

---

## 4. 기능 요구사항 전수 커버리지 매핑

39개 REQ-FUNC 전체가 Next.js 단일 앱에서 구현 가능한지 검증한다.

| REQ 범위 | 건수 | 구현 방식 | 커버리지 |
|---|---|---|---|
| **REQ-FUNC-001~009** (F1 Super-Calc) | 9 | Route Handler + Channel Adapters + Prisma | ✅ 100% |
| **REQ-FUNC-010~015** (F2 Anti-BS) | 6 | Route Handler + Prisma (BADGE/INGREDIENT) + Vercel KV 캐시 | ✅ 100% |
| **REQ-FUNC-016~021** (F3 Viral) | 6 | Client Component (Kakao SDK) + Route Handler (`@vercel/og`) | ✅ 100% |
| **REQ-FUNC-022~028** (F4 Data Trust) | 7 | Server Action (Prisma CRUD) + Supabase Storage (라벨) | ✅ 100% |
| **REQ-FUNC-029~034** (공통) | 6 | Supabase Auth + Prisma + Client-side Mixpanel | ✅ 100% |
| **REQ-FUNC-035~037** (Should) | 3 | Prisma + Vercel Cron (가격 하락 감지) + Resend (이메일) | ✅ 100% |
| **REQ-FUNC-038~039** (Could) | 2 | Next.js 페이지 최적화 + Prisma (익명화 뷰) | ✅ 100% |
| **합계** | **39** | — | ✅ **39/39 (100%)** |

---

## 5. 비기능 요구사항 영향도

| REQ-NF | 현 SRS | 변경 후 | 영향 |
|---|---|---|---|
| REQ-NF-001~008 (성능) | 동시 200명 기준 | Vercel Serverless 동시 실행로 충족 | 🟢 유지 |
| REQ-NF-009~013 (신뢰성) | 99.5% 가용성, RPO 24h | Vercel 99.99% SLA + Supabase 자동 백업 | 🟢 **초과 달성** |
| REQ-NF-014~018 (보안) | TLS 1.2+, 데이터 최소 수집 | Vercel 기본 HTTPS + Supabase RLS | 🟢 유지 |
| REQ-NF-019~020 (비용) | $0~10만 원 | Vercel Free + Supabase Free = **$0 시작** | 🟢 유지 |
| REQ-NF-021~023 (모니터링) | CloudWatch/Datadog/PagerDuty | **Vercel Analytics + Vercel Logs + Slack Webhook** | 🟡 도구 치환 |
| REQ-NF-024 (확장성) | MSA 플러그인 | **전략 패턴 (`/lib/adapters/`)** | 🟡 패턴 변경 (기능 동일) |

---

## 6. 작업 순서 및 체크리스트

### Phase 1: 제약사항 및 용어 (Low Risk)
- [ ] §1.2.3 Constraints: CON-7 ~ CON-13 추가
- [ ] §1.3 Definitions: 신규 용어 9건 추가

### Phase 2: 아키텍처 섹션 전면 변경 (High Impact)
- [ ] §3.1.1 폴백 전략: Redis/S3 참조 치환
- [ ] §3.3 API Overview: 내부 API 구현 주체 변경
- [ ] §3.6 Component Diagram: **Mermaid 전면 재작성**
- [ ] §3.6 컴포넌트 테이블: 모놀리스 모듈 구조로 재작성

### Phase 3: 요구사항 조정 (Medium Impact)
- [ ] §4.2.4 Cost: Vercel/Supabase 비용 체계 반영
- [ ] §4.2.5 Monitoring: 도구 치환
- [ ] §4.2.6 Scalability: 전략 패턴 재정의

### Phase 4: Appendix 업데이트 (Low Risk)
- [ ] §6.1.2 내부 API: Route Handler로 구현 주체 변경
- [ ] §6.2 Entity: Prisma 모델 매핑 주석
- [ ] §6.2.9 Class Diagram: 서비스 스테레오타입 변경

### Phase 5: 시퀀스 다이어그램 갱신 (Medium Impact)
- [ ] §3.4 핵심 시퀀스 3종: participant 명칭 갱신
- [ ] §6.3 상세 시퀀스 4종: participant 명칭 갱신

### Phase 6: LLM 통합 (미결 — 사용자 확인 후)
- [ ] LLM 활용 범위 확정
- [ ] 필요 시 REQ-FUNC 추가 / 기존 REQ 구현 방식 변경

---

## 7. 미결 사항 (Open Items)

| # | 항목 | 결정 필요 사항 | 영향 범위 |
|---|---|---|---|
| 1 | **LLM MVP 포함 범위** | Gemini API를 MVP F1~F4 중 어디에 활용할 것인가? 아니면 인프라 준비만? | REQ-FUNC 추가 여부, 비용 추정 변동 |
| 2 | **Push vs 이메일** | REQ-FUNC-026 (제보 결과 알림)을 Push 대신 이메일로 전환해도 되는가? | 알림 채널 명세 변경 |
| 3 | **Vercel Plan** | MVP 런칭 시 Hobby(Free) vs Pro($20/월) 중 어느 플랜으로 시작할 것인가? | 동시 실행, Cron 빈도, 대역폭 한도 |

---

*— End of Plan Document —*

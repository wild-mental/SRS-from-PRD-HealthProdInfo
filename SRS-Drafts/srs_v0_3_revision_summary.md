# SRS v0_3 수정 완료 보고서

**기준 문서:** `SRS_Revision_Flash_Recipe.md` (4-Step 리팩토링 계획)  
**대상 문서:** `SRS_v0_3.md` (Revision 1.3 → **1.4**)  
**수정 일시:** 2026-04-17

---

## 수정 요약 (Step별)

### ✅ Step 1: 프로젝트 스코프 및 인프라 제약 현실화

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| IS-1 범위 | iHerb, 쿠팡, 네이버 3채널 | **쿠팡 파트너스 단일 채널 (Phase 1)** |
| IS-6 수익 모델 | 제휴 CPA (iHerb, 쿠팡) | **제휴 CPA (쿠팡 파트너스)** |
| OS-8 | (없음) | **신설** — 다중 채널 병렬 API 오케스트레이션 및 환율 동적 계산 (Phase 2 연기) |
| CON-3 인프라 | Vercel Hobby 무료 티어 | **Vercel Pro + Supabase Pro (월 $45)** |
| ASM-1 | iHerb/쿠팡 성분 메타데이터 | **쿠팡 가격/딥링크 안정적 제공** |
| ASM-5 (구 ASM-5 환율) | 환율 API 15분 갱신 가정 | **삭제** |
| ASM-6 → ASM-5 | 카카오 정책 유지 가정 | ASM-5로 재번호 |
| CP-1 대안3 | 해외 DB (Open FDA, DSLD) | **삭제** (국내 단일 채널 전환) |

### ✅ Step 2: 비기능 요구사항(NFR) 하향 조정

| ID | 변경 전 | 변경 후 |
|---|---|---|
| REQ-NF-001 | p95 ≤ 2,000ms / 200명 | **p95 ≤ 3,500ms / 50명** |
| REQ-NF-006 | 동시 200명 (피크 500명) | **동시 50명 (피크 100명)** |
| REQ-NF-007 | 부하 테스트 월 1회 | **~~삭제~~ (MVP 오버엔지니어링 방지)** |
| REQ-NF-009 | 가용성 99.5% 보장 | **Best Effort (Phase 2 SLA 도입)** |
| REQ-NF-013 | 일 1회 자동 백업, RPO 24h | **수동 백업 (Phase 2 자동화)** |
| REQ-NF-019 | 월 0원 (무료 티어) | **월 $50 이하** |

### ✅ Step 3: 핵심 기능 로직(FR) 단순화

| ID | 변경 내용 |
|---|---|
| REQ-FUNC-001 | 3채널 병렬 조회 → **쿠팡 파트너스 단일 조회** |
| REQ-FUNC-002 | 산출 공식: `(가격 × 환율) ÷ 횟수` → **`가격 ÷ 횟수`** |
| REQ-FUNC-003 | 환율 API 연동 → **~~삭제~~ (KRW 단독)** |
| REQ-FUNC-005~006 | p95 2초 / 200명 → **p95 3.5초 / 50명** |
| REQ-FUNC-007 | 멀티채널 부분 실패 처리 → **~~삭제~~ (단일 채널)** |
| REQ-FUNC-009 | iHerb/쿠팡 딥링크 → **쿠팡 파트너스 딥링크만** |
| REQ-FUNC-017 | `@vercel/og` 동적 OG 이미지 → **정적 메타태그 + 고정 로고** |
| REQ-FUNC-021 | URL 복사 + 밴드/문자 대체 → **URL 클립보드 복사 폴백만** |

### ✅ Step 4: 다이어그램 및 기술 스펙 동기화

| 섹션 | 변경 내용 |
|---|---|
| **3.1 External Systems** | EXT-SYS를 3개(쿠팡/식약처/카카오)로 축소. iHerb, 환율 API 제거 |
| **3.1.1 폴백 전략** | 3개 시스템 기준으로 재구성. 환율·iHerb 폴백 제거 |
| **3.3 API Overview** | iHerb, 환율 API 제거. Super-Calc 응답 시간 3.5초 반영 |
| **3.4.1 시퀀스 (F1)** | iHerb/네이버/환율 객체 제거. 쿠팡 단일 조회 흐름 |
| **3.4.3 시퀀스 (F3)** | OG 이미지 서비스 제거. 정적 메타태그 공유 흐름 |
| **3.5 Use Case Diagram** | 외부 시스템을 3개(쿠팡/식약처/카카오)로 축소 |
| **3.6 Component Diagram** | OG Generator, Channel Adapters, Vercel KV, FX 컴포넌트 제거 |
| **6.1.1 외부 API** | EXT-API를 3개로 축소 (iHerb, 환율 삭제) |
| **6.1.2 내부 API** | INT-API를 5개로 축소 (OG Image Generator, 환율 내부 조회 삭제) |
| **6.2.1 PRODUCT** | `source_channel` 제약: `(쿠팡)` |
| **6.2.3 PRICE_SNAPSHOT** | `currency`, `exchange_rate`, `customs_fee` 삭제. `original_price` → `price_krw` |
| **6.2.8 ER Diagram** | PRICE_SNAPSHOT에서 환율 관련 필드 제거 |
| **6.2.9 Class Diagram** | PriceSnapshot, SuperCalcModule 클래스 단순화 |
| **6.3.1 상세 시퀀스 (F1)** | 단일 채널 조회 + 캐시 폴백 흐름으로 전면 재작성 |
| **6.3.2 상세 시퀀스 (F2)** | 뱃지 캐시를 Vercel KV → Next.js Cache로 변경 |
| **6.3.4 상세 시퀀스 (F3)** | OG 동적 생성 제거, 정적 메타태그 + URL 복사 폴백만 |
| **6.4.1 롤아웃** | Closed Beta 1 대상: iHerb 파워유저 → 쿠팡 파워유저 |
| **문서 버전** | v1.2 → **v1.4** |

---

> [!IMPORTANT]
> 모든 4개 Step이 완료되었으며, 문서 전체에 걸쳐 iHerb/네이버/환율 API/OG 동적 생성 관련 참조가 제거·수정되었습니다.

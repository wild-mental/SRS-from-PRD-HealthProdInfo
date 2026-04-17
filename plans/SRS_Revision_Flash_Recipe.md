이해했습니다. AI 에이전트(Cursor, Windsurf 등)가 맥락을 잃지 않고 기존 SRS 문서를 완벽하게 리팩토링할 수 있도록, **'문서 수정용 프롬프트 파이프라인'**을 작성했습니다.

이 계획서는 AI에게 한 번에 모든 것을 지시하지 않고, 문서의 논리적 흐름에 따라 순차적으로 주입하여 컨텍스트 충돌을 막기 위해 설계되었습니다. 각 Step의 `[AI Prompt]` 블록을 복사하여 순서대로 AI에게 지시하시면 됩니다.

---

# AI 에이전트 오케스트레이팅용 SRS 리팩토링 프롬프트 계획서

## Step 1: 프로젝트 스코프 및 인프라 제약 현실화
**목표:** AI가 3채널 병렬 처리와 완전 무료 티어의 모순을 인식하고, 단일 채널 및 유료 인프라 기반으로 기본 제약사항을 수정하도록 지시합니다.

**[AI Prompt]**
```text
@SRS_v0_1_Opus.md 문서를 수정할 거야. 다음 지시사항에 따라 `1. Introduction` 섹션을 업데이트해줘.

1. [1.2.1 In-Scope] IS-1의 범위를 'iHerb, 쿠팡, 네이버 3채널'에서 '쿠팡 파트너스 단일 채널 연동(Phase 1)'으로 축소해.
2. [1.2.2 Out-of-Scope] OS-8을 신설하고 '다중 채널 병렬 API 오케스트레이션 및 환율 동적 계산 (Phase 2로 연기)'을 추가해.
3. [1.2.3 Constraints] CON-3을 수정해. '완전 무료 티어(Free Tier)' 조건을 삭제하고, 트래픽을 감당하기 위해 'Vercel Pro + Supabase Pro (예상 월 $45)' 환경을 기본 인프라로 가정하도록 변경해.
4. [1.2.4 Assumptions] ASM-1과 ASM-5를 삭제해 (iHerb 및 환율 연동이 제거됨).
5. [1.2.5 Contingency Plans] CP-1에서 대안 3(해외 DB)을 삭제하고, 국내 제품 단일 채널에 맞게 내용을 축소해.

수정된 마크다운 결과물만 전체 출력해줘.
```

## Step 2: 비기능 요구사항(NFR) 하향 조정
**목표:** 변경된 인프라 환경과 AI 개발 속도에 맞춰 달성 불가능한 성능/운영 지표를 안전한 수준으로 조정합니다.

**[AI Prompt]**
```text
이어서 `4.2 Non-Functional Requirements` 섹션을 다음 기준에 맞춰 현실적으로 수정해.

1. [4.2.1 Performance] 
   - REQ-NF-001: p95 응답 시간을 2,000ms에서 '3,500ms'로 완화.
   - REQ-NF-006: 동시 접속 기준을 200명(피크 500명)에서 '50명(피크 100명)'으로 축소.
   - REQ-NF-007: 부하 테스트 요구사항을 삭제해 (MVP 단계 오버엔지니어링 방지).
2. [4.2.2 Reliability] 
   - REQ-NF-009: 가용성 99.5% 보장을 삭제하고, 'Best Effort (최선 노력) 제공'으로 변경해.
   - REQ-NF-013: 일 1회 자동 백업(RPO 24h)을 삭제하고 '수동 백업 의존(Phase 2에서 자동화)'으로 변경해.
3. [4.2.4 Cost]
   - REQ-NF-019: 월간 인프라 비용 목표를 '월 $50 이하'로 수정해.

수정된 4.2 섹션의 마크다운 결과물만 출력해.
```

## Step 3: 핵심 기능 로직(FR) 단순화
**목표:** 병목 구간(병렬 처리, 동적 이미지 생성, Redis 캐싱)을 모두 제거하여 Vibe Coding의 성공률을 극대화합니다.

**[AI Prompt]**
```text
이어서 `4.1 Functional Requirements` 섹션을 대폭 단순화해. AI 에이전트가 코드로 구현하기 쉬운 형태로 수정하는 것이 핵심이야.

1. [F1. Super-Calc Engine]
   - REQ-FUNC-001 ~ 003: 3개 채널 병렬 조회 및 환율 API 연동 로직을 모두 삭제해. '입력된 영양소명으로 쿠팡 파트너스 API를 단일 조회한다'로 통합 수정해.
   - REQ-FUNC-007: 다중 채널 장애 시의 부분 실패(Partial Failure) 처리 로직을 삭제해.
2. [F3. Viral Engine]
   - REQ-FUNC-017: `@vercel/og` 기반의 동적 OG 이미지 생성 요구사항을 완전히 삭제해. '고정된 서비스 로고 및 정적 메타태그(Open Graph)를 사용하여 카카오톡 표준 URL 공유를 수행한다'로 대체해.
   - REQ-FUNC-021: API 장애 시 대체 공유 채널(밴드, 문자 등) 노출 기능을 삭제하고, 단순 'URL 클립보드 복사 폴백'만 남겨.

수정된 4.1 섹션의 마크다운 결과물만 출력해.
```

## Step 4: 다이어그램 및 엔터티/API 스펙 동기화 (가장 중요)
**목표:** 앞서 잘라낸 스펙들이 아키텍처, 시퀀스 다이어그램, DB 모델에 남아있어 AI가 환각을 일으키는 것을 원천 차단합니다.

**[AI Prompt]**
```text
지금까지 수정된 내용(단일 채널 연동, OG 이미지 정적화, 환율 연동 제거, KV 캐시 축소)을 바탕으로, 문서의 나머지 기술 스펙 섹션들을 완벽하게 동기화할 거야. 

1. `3.4 Interaction Sequences` 및 `6.3 Detailed Interaction Models`의 Mermaid 다이어그램:
   - iHerb, 네이버, 환율 API(FX), Vercel KV(환율 캐시) 객체와 관련 메시지를 다이어그램에서 모두 제거해.
   - 카카오톡 공유 시퀀스에서 OG 이미지 동적 생성 로직을 제거하고 정적 공유 흐름으로 수정해.
2. `3.6 Component Diagram`:
   - Vercel KV, OG Generator, Channel Adapters(Strategy Pattern) 등 불필요하게 복잡한 컴포넌트를 모두 삭제해.
3. `6.1 API Endpoint List`:
   - EXT-API-01(iHerb), EXT-API-04(환율 API), INT-API-03(OG Image Generator), INT-API-07(환율 내부 조회) 항목을 삭제해.
4. `6.2 Entity & Data Model`:
   - `PRICE_SNAPSHOT` 테이블에서 `currency`, `exchange_rate` 필드를 삭제해 (원화 KRW 단독 처리).
   - `PRODUCT` 테이블의 `source_channel` 제약조건에서 iHerb와 네이버를 제거해.

변경된 3장과 6장 전체를 마크다운 텍스트와 수정된 Mermaid 코드와 함께 출력해.
```
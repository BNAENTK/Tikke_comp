# 팀 작업 계획

- 팀명: kkirikkiri-development-20260604-1736-5b3e
- 목표: tikke 모노레포 기능 추가 + 버그 수정 (데스크탑 앱 + 웹)
- 생성 시각: 2026-06-04T17:36

## 팀 구성

| 이름 | 역할 | 모델 | 담당 업무 |
|------|------|------|----------|
| tikke-lead | 팀장 | Opus | 코드베이스 스캔 → 우선순위 결정 → 태스크 배분 → 통합 검증 |
| tikke-desktop-dev | 데스크탑 개발자 | Opus | Electron IPC / TTS / 스트리밍 연동 핵심 로직 구현 |
| tikke-web-dev | 웹 개발자 | Opus | Next.js 웹앱 / Cloudflare Worker / 오버레이 HTML 구현 |
| tikke-critic | 검증자 | Sonnet | 테스트 작성 + 코드 리뷰 + 기존 동작 회귀 검증 |

## 컨텍스트

- 프로젝트 루트: C:/Users/zkfma/OneDrive/바탕 화면/claude-code/tikke
- 스택: Electron + TypeScript + React + Next.js + Cloudflare Workers + pnpm 모노레포
- 패키지 매니저: pnpm
- GitHub CLI: 사용 가능
- Codex CLI: 사용 가능 (코드 리뷰 보조)
- 핵심 메모리: C:/Users/zkfma/.claude/projects/C--Users-zkfma-OneDrive-------claude-code/memory/

## 태스크 목록

- [ ] 태스크 1: tikke 코드베이스 전체 스캔 → 기능 추가/버그 목록 작성 → tikke-lead
- [ ] 태스크 2: 우선순위 높은 데스크탑 기능/버그 수정 → tikke-desktop-dev
- [ ] 태스크 3: 우선순위 높은 웹/오버레이 기능/버그 수정 → tikke-web-dev
- [ ] 태스크 4: 변경사항 테스트 + 회귀 검증 → tikke-critic
- [ ] 태스크 5: 결과 통합 + 최종 리포트 → tikke-lead

## 주요 결정사항

- 2026-06-04T17:36 — 팀장이 먼저 코드베이스 스캔 후 실제 작업 우선순위 결정
- 사용자가 명시한 특정 기능/버그가 없으므로 팀장이 자체 판단해 목록 작성 후 실행

---

## 코드베이스 스캔 결과 (tikke-lead, 2026-06-04)

### 베이스라인 검증 상태
- Worker typecheck: **PASS** (`pnpm --filter @tikke/worker typecheck` 통과)
- Desktop typecheck: **FAIL** — ~15개 TS 에러 (pre-existing, v1.8.78 커밋에 이미 존재)
- Web `vite build`: **FAIL** — `three` CDN importmap을 Rollup이 번들하려다 실패. **단, web `tsc --noEmit`는 통과**.
- Git tree: clean (변경 없음). 따라서 모든 TS 에러는 기존 커밋된 코드의 문제.

### 핵심 판정 (회귀 방지)
1. **Web `three` 빌드 에러는 손대지 말 것.** `three`는 npm 의존성이 아니라 `index.html`의 `<script type="importmap">` CDN 로드. web 배포는 raw 파일 복사(메모리: "raw, dist 금지")라 `vite build` 산출물은 배포에 안 쓰임. `rollupOptions.external` 추가/importmap 수정 = 무의미하거나 회귀. **금지.**
2. **net.fetch 일괄 변환 금지.** `browserFetch`(ipc.ts:1997)는 `Origin`/`Referer`/`sec-fetch-site:same-origin` 헤더를 주입함(caller 헤더는 마지막에 spread되어 우선). 규칙의 의도는 "ipc.ts 외부 API WAF 우회"로 한정.
   - ipc.ts:1853(Supabase REST, apikey/Authorization), 1908(Groq, bearer) = 1st-party 인증 API. 변환 시 잘못된 Origin/sec-fetch 헤더 주입 → **변환 금지, 현상 유지.**
   - ipc.ts:481, 505(사운드 URL fetch, myinstants 등 브라우저 UA 필요) = WAF 우회 의도와 일치. **변환 후보(단, 481은 임의 사운드 URL이라 신중).**
   - `services/*.ts`의 net.fetch는 ipc.ts 규칙 밖. 변환 대상 아님. 발견사항으로만 기록.
3. `apps/worker/public/overlay/coin-jar.html` = **orphan** (참조 0건, desktop은 coinjar.html 사용 — EventOverlay.tsx). CLAUDE.md 원칙: 삭제 금지, 언급만.

---

## 분배 태스크 (Impact × Effort)

### Desktop 담당 (tikke-desktop-dev) — 이번 세션 메인

**태스크 D1 [Impact 높음 / Effort 중간] — Desktop typecheck 그린화**
목표: `pnpm --filter @tikke/desktop typecheck` 통과. `@ts-ignore`로 덮지 말 것(특히 drift 에러).

- **그룹 A (타입 drift — 런타임 검증 후 타입 보강, 高가치)**
  - `overlay-server.ts`의 `OverlayMessage`/`OverlayMessageType`에 `emoji_rain`(emoji), `welcome`(user), `tornado`(payload) 누락 → command-service.ts:256/262/274가 실제로 broadcast 중이고 대응 오버레이 HTML(emoji-rain/welcome-banner/gift-tornado)도 존재. 런타임이 보내는 게 맞으므로 타입에 variant 추가(안전).
  - `tiklive-normalizer.ts:62` `giftAnimatedUrl`이 GiftEvent 타입에 없음 → 런타임이 이 필드를 실제 세팅/소비하는지 확인 후, 사용 중이면 타입에 추가. 안 쓰면 객체에서 제거(실제 버그일 수 있음).
  - `top-donor-service.ts:342` `TopDonorPayload`에 maxGiftCoins/maxGiftName/maxGiftPictureUrl/topFreqGiftName/topFreqGiftPictureUrl 누락 → 오버레이가 이 필드를 쓰는지 확인 후 객체 채우기 or 타입 조정.
  - `config`의 AppSettings index signature 누락(ipc.ts:111 `as Record<string,unknown>` 변환 에러).
- **그룹 B (툴링/lib — blast radius 주의)**
  - `electron.vite.config.ts:3` vite-plugin-obfuscator 선언파일 없음 → `// @ts-expect-error` 또는 모듈 d.ts shim (config 파일이라 한정 허용).
  - ipc.ts:1473 `Promise.any` → tsconfig lib es2021+ 필요. **lib 변경 시 blast radius 큼 → 변경 후 전체 typecheck 재실행 필수.**
  - `preload/index.ts:395` `window` 못 찾음 → preload tsconfig lib에 DOM 추가 검토.
  - ipc.ts:2879 onBeforeSendHeaders 인자 타입 → 시그니처 정합.

**태스크 D2 [Impact 중간 / Effort 낮음] — net.fetch 정리 (한정)**
- ipc.ts:505(resolveUrl)는 WAF 우회 의도 명확 → `browserFetch`로 전환 검토. 481은 임의 사운드 URL이라 신중(현상 유지 가능).
- Supabase/Groq(1853/1908) 및 services/*.ts는 **건드리지 말 것**. 판단 근거를 TEAM_FINDINGS.md에 기록.

### Web/Worker 담당 (tikke-web-dev) — 이번 세션 라이트

**태스크 W0 [Impact 높음 / Effort 낮음] — cutoffs.ts KST/UTC 날짜 불일치 버그** ← 추가 발견 (메인 세션)
- 파일: `apps/worker/src/routes/cutoffs.ts:12-13`
- 버그: `today`/`yesterday` 계산이 UTC 기준 (`new Date().toISOString().slice(0,10)`)
- 저장 시: `league-cutoff-collector.ts:22`는 KST(UTC+9) 기준으로 `league_date` 저장
- 결과: 매일 15:00~23:59 UTC (자정~08:59 KST) 9시간 동안 "today" 쿼리 빈 데이터
- 수정: `today`/`yesterday` 를 `new Date(Date.now() + 9 * 3600 * 1000).toISOString().slice(0,10)` 로 변경

**태스크 W1 [Impact 낮음 / Effort 낮음] — coin-jar.html orphan 확인**
- `apps/worker/public/overlay/coin-jar.html`이 `routes/overlay.ts`나 오버레이 레지스트리에서 참조되는지 최종 확인. 참조 없으면 **삭제하지 말고** TEAM_FINDINGS.md에 "다음 세션 정리 후보"로 기록.
- Web `vite build` 에러(`three`)는 **손대지 말 것**(위 판정 1). web 검증은 `pnpm --filter @tikke/web exec tsc --noEmit`로 확인.

**태스크 W2 [조건부] — Desktop 타입 변경의 오버레이 영향 확인**
- desktop-dev가 OverlayMessage/TopDonorPayload 타입에 필드를 추가하면, 해당 오버레이 HTML(desktop public/overlays + worker public/overlay 양쪽)이 그 필드/메시지를 실제로 핸들하는지 확인. 누락 시 message 핸들러 풀체인 점검(메모리: top-donor message 핸들러 필수). 양쪽 동기화 유지.

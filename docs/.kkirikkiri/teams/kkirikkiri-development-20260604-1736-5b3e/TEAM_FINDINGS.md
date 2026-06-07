# 발견 사항 & 공유 자료

## 2026-06-04T17:36 — kkirikkiri (메인 세션): 프로젝트 컨텍스트

tikke 모노레포 핵심 구조:
- `apps/desktop` — Electron 앱 (TikTok LIVE 방송인용 Windows 데스크탑)
- `apps/web` — Next.js 웹앱 (tikke 웹사이트)
- `apps/worker` — Cloudflare Workers
- `packages/` — 공유 패키지 (shared, tts 등)
- TTLS 연동: TikTok LIVE Studio, 포트 28189 Socket.IO
- 오버레이: Cloudflare Worker public/overlay + desktop HTML 양쪽에 동기 필수
- 배포: commit → 버전업 → 태그 → push (순서 고정, 내 판단으로 변경 금지)
- 외부 fetch: ipc.ts에서 browserFetch() 헬퍼만 사용 (net.fetch 직접 금지)
- OBS 언급 금지 (사용자는 TTLS 사용)

---

## 2026-06-04 — tikke-lead: 스캔 발견사항 (회귀 방지 근거)

### browserFetch 변환 판정 근거 (D2 작업자 필독)
`browserFetch`(ipc.ts:1997~)는 url의 origin으로 `Origin`/`Referer`/`sec-fetch-site:same-origin`를 강제 주입한다. `extra.headers`는 마지막에 spread되어 caller 헤더가 이김. 따라서:
- 1st-party 인증 API(Supabase REST `apikey`+`Authorization`, Groq bearer)에 적용하면 인증 헤더는 살지만 잘못된 Origin/sec-fetch 헤더가 붙어 의미 없고 위험. → **변환 금지.**
- myinstants 등 사운드 페이지 스크레이프(ipc.ts:505)는 브라우저 UA가 필요한 WAF 우회 케이스 → 변환 의도와 일치.
- `services/*.ts`(crm-collector/rank-collector/league-* 등)의 net.fetch는 ipc.ts 한정 규칙 밖. 변환 대상 아님.

### Web 빌드 (web-dev 필독)
`apps/web/index.html`은 `three`를 `<script type="importmap">` CDN(unpkg)로 로드한다. npm 의존성 아님. `vite build`는 importmap 미인식 → `three` 미해결로 실패하지만, web 배포는 raw 복사라 `vite build` 산출물 무관. web 검증은 `tsc --noEmit`만 사용. importmap/rollup external 손대지 말 것.

### 오버레이 동기화 상태
desktop public/overlays(47) ↔ worker public/overlay(47) 거의 동기. 차이: desktop엔 index.html(정상), worker엔 coin-jar.html(orphan, 참조 0). desktop은 coinjar.html 사용(EventOverlay.tsx).

### OverlayMessage 타입 위치
`OverlayMessage`/`OverlayMessageType` 정의: `apps/desktop/electron/services/overlay-server.ts`. command-service.ts가 실제 broadcast하는 emoji_rain/welcome/tornado가 타입에 누락됨(런타임 drift).

---

## 2026-06-04 — tikke-web-dev: W0/W1 발견사항

### W0 cutoffs.ts KST/UTC 버그 — 수정 완료
- `apps/worker/src/routes/cutoffs.ts:11-13`. `today`/`yesterday`를 `Date.now() + 9*3600*1000`(KST) 기준으로 변경.
- 근거: `services/league-cutoff-collector.ts:22` `leagueDateKST()`가 KST로 `league_date` 저장 → 쿼리도 KST여야 일치.
- **주의(건드리지 않음)**: 같은 파일 line 33 `nowSec`는 `getUTCHours/Minutes`로 계산하지만, `created_at`(실제 UTC instant)과의 "같은 시각" 매칭용이라 UTC끼리 비교가 내부 정합. KST 변환하면 오히려 깨짐 → 의도적으로 보존.

### W1 coin-jar.html orphan — 최종 확인 (삭제 안 함)
- `apps/worker/public/overlay/coin-jar.html`은 라우트/레지스트리 참조 0건 확정.
- 전수조사 결과 `coin-jar`(하이픈) 매칭은 전부 `assets.tikfinity.com/widget/coin-jar/images/...` 이미지 에셋 URL뿐.
- 실제 오버레이는 모두 하이픈 없는 `coinjar*.html`(coinjar/coinjar-bounce/-neon/-loop/-explode) 사용. 라우트 경로는 cloud-overlay.ts:94-100, EventOverlay.tsx:204-221에서 `/overlay/coinjar*`.
- 결론: `coin-jar.html`은 orphan 유지. CLAUDE.md 원칙대로 삭제 금지, **다음 세션 정리 후보**로만 기록.

### W2 참고 — coinjar desktop↔worker diff 분류 완료 (동기 누락 아님)
- `diff` 전체 확인 결과: `coinjar.html`의 유일한 차이는 worker 쪽에 한 줄 추가된 `<script>/* tikke-perf-patch */...willChange...</script>`(애니메이션 willChange 성능 패치). 물리/렌더/이벤트 핸들러 로직은 **바이트 동일**.
- 즉 **로직 drift 아님** — worker 배포본에 perf-patch 스크립트만 주입된 형태(빌드/배포 시 주입 추정). coinjar-neon 등 다른 항아리도 동일 패턴으로 추정.
- 결론: 동기화 버그 아님, 의도적 차이. **수정 불필요.**
- W2 대상 오버레이(emoji-rain/welcome-banner/gift-tornado)는 desktop + worker 양쪽 모두 파일 존재 확인. desktop-dev가 OverlayMessage/TopDonorPayload 필드 추가하는 W2 트리거 발생 시 해당 3종의 message 핸들러 필드 핸들링 재점검 권장.

---

## 2026-06-04 — tikke-desktop-dev: D1 typecheck 발견사항

### @tikke/shared 편집은 빌드해야 desktop에 반영됨 (전 팀원 필독)
- desktop은 `@tikke/shared`를 **빌드 산출물(dist/*.d.ts)** 로 resolve. `packages/shared/src/*.ts`만 고치면 desktop typecheck에 반영 안 됨.
- 반드시 `pnpm --filter @tikke/shared build` 실행 후 재검증. (events.ts에 GiftEvent 필드 추가했을 때 처음엔 미반영 → 빌드 후 해결)

### OverlayMessage 타입 drift 정정 — interface 필드 추가는 불필요
- TEAM_PLAN/lead 노트는 emoji_rain/welcome/tornado가 OverlayMessageType union에 누락이라 했고 그게 맞음(추가함).
- 단, `OverlayMessage` **interface에 emoji/user 필드를 추가할 필요는 없다.** 모든 broadcast 호출이 `const msg = {...}; broadcast(msg)` 패턴이라 const 변수 인자에는 excess-property check가 적용 안 됨 → union의 `type` literal만 맞으면 통과.
- 처음에 emoji?/user? 필드를 interface에 추가했더니 `overlay-rules-service.ts:159`(fireworks user를 unknown 필드로 구성)가 새로 깨짐 → revert. **교훈: 에러 메시지가 "type incompatible"만 가리키면 union만 고치면 됨.**

### W2 트리거 — OverlayMessageType에 3종 추가됨 (web-dev 확인 권장)
- emoji_rain/welcome/tornado가 이제 타입에 정식 등록. 대응 HTML 3종(emoji-rain/welcome-banner/gift-tornado)은 desktop+worker 양쪽 존재 확인됨(web-dev W2 노트). 새 필드/페이로드 추가는 없었으므로(타입 union만) message 핸들러 변경 불요로 판단. 추가 점검은 web-dev 재량.

### tikketone는 D1 범위 밖 — pre-existing FAIL
- 전체 `pnpm typecheck`에서 `@tikke/tikketone`만 FAIL(electron/main/ipc.ts:49/52/58/66, `"tikketone"|"fish"` vs `"tikketone"` 엔진 literal 미스매치). 내 변경과 무관, 별도 앱. lead 스캔에도 미포함. **D1(@tikke/desktop) 범위 밖이라 미수정.** 정리하려면 별도 태스크 필요.

---

---

## 2026-06-04 — tikke-critic: 이번 세션 변경 검증 리포트

# [이번 세션 변경] 검증 리포트

## 요약
- 전체 신뢰도: **높음**
- 치명 리스크: **0건**

## 회귀 체크

### typecheck 4종 실제 실행 결과
| 패키지 | 명령어 | 결과 |
|--------|--------|------|
| @tikke/shared | `pnpm --filter @tikke/shared build` | PASS ✅ |
| @tikke/desktop | `pnpm --filter @tikke/desktop typecheck` | PASS ✅ (에러 15→0) |
| @tikke/worker | `pnpm --filter @tikke/worker typecheck` | PASS ✅ |
| @tikke/web | `pnpm --filter @tikke/web exec tsc --noEmit` | PASS ✅ |

desktop-dev 주장(15→0) 및 web-dev 주장(worker PASS) 실제 검증 완료.

### GiftEvent 필드 추가 회귀 전수조사
- GiftEvent import 파일 8개 확인: gta-bridge.ts / minecraft.ts / telegram.ts / EventFeed.tsx / GiftBrowser.tsx / GiftViewer.tsx / harness/tts-queue.ts / harness/gift-sound.ts
- 추가된 7개 필드(giftAnimatedUrl/giftPanelUrl/giftGoldEffect/giftEffectZipUrl/giftEffectVideoUrl/giftEffectId/groupId) 모두 `?` optional → 기존 생성 코드 회귀 없음 ✅
- tiklive.ts:373-402에서 cast 우회로 이미 세팅 중이었음 — 타입 추가는 사실 반영이지 신규 동작 아님 ✅
- harness의 GiftEvent는 로컬 inline interface(상위 shared import 아님) — 영향 없음 ✅

### OverlayMessageType union 변경 회귀
- emoji_rain/welcome/tornado: command-service.ts:254-277에서 실제 broadcast 중 ✅
- 대응 HTML 3종 desktop ✅ / worker ✅ — 양쪽 존재 확인
- 3종 HTML 모두 `window.addEventListener('message')` 핸들러 존재 ✅
- DEAD_ENDS에 기록된 overlay-rules-service.ts fireworks 회귀 — revert 처리됨. 현재 typecheck PASS로 회귀 없음 확인 ✅
- OverlayMessage interface 자체 필드 추가 없음 — excess-property check 우회 패턴 그대로 유지 ✅

### cutoffs.ts 날짜 로직 정합성
- today/yesterday KST(line 12-14): `Date.now() + 9*3600*1000` — league-cutoff-collector.ts:22 `leagueDateKST()` 동일 패턴 ✅
- nowSec UTC(line 34): `created_at`은 Supabase에 UTC ISO instant로 저장. UTC끼리 비교가 정합. KST로 변환해도 차이값(`|sec - nowSec|`)은 불변이지만, UTC 유지가 의도 명확 ✅
- 혼용이 버그인 것처럼 보이지만 실제로는 "날짜 문자열(league_date)"과 "시각 차이(created_at instant)"를 각각 독립적으로 다루는 올바른 설계 ✅

### net.fetch 패턴 전수조사 (현황 확인 — 변경 없음 검증)
- ipc.ts net.fetch 4곳(481/505/1853/1908) — TEAM_PLAN 판정 2와 정확히 일치, 미변경 ✅
- 481/505: 변환 후보/현상 유지 그대로
- 1853(Supabase)/1908(Groq): 변환 금지 그대로

### 기타 그룹 B 변경 확인
- `@ts-expect-error` (electron.vite.config.ts:3): vite-plugin-obfuscator 선언파일 없음 — 정당한 사용 ✅
- `as unknown as Record` (ipc.ts:111): AppSettings index signature 없음 — TS 권장 더블캐스트 ✅
- lib ES2020→ES2021 (tsconfig.node.json:4): Promise.any 지원, 상위호환. desktop typecheck PASS로 blast radius 없음 ✅
- `onBeforeSendHeaders(null)` (ipc.ts:2879): electron 타입 오버로드에 null 직접 허용 ✅
- `declare const window` (preload/index.ts:2): DOM lib 전역 추가 대신 preload 한정 선언 — main process 오염 방지 ✅

## 반박/약점
- **[주의] tiklive.ts cast 우회 패턴 잔존**: giftAnimatedUrl/giftPanelUrl/giftGoldEffect는 GiftEvent 타입에 추가됐지만 giftEffectZipUrl/giftEffectVideoUrl는 tiklive.ts:380에서 여전히 `as typeof giftEv & { giftEffectZipUrl?: string; giftEffectVideoUrl?: string }` cast 우회 중. 타입이 이미 있으므로 cast 제거 가능하지만 동작에 영향 없음 (주의 — 별도 정리 후보)
- **[확인 완료] `giftEffectId` 필드 — 실제 사용 확인**: tiklive.ts 한정 grep에서는 발견 안 됐으나 전체 프로젝트 grep 결과 `tiklive-normalizer.ts:45`에서 `giftEffectId = asset?.id as string | undefined`로 세팅, line 67에서 return 객체에 포함. `EventOverlay.tsx:370-371`에서 하네스 테스트 페이로드에도 포함. dead field 아님 — 확인 완료.
- **[경미] topFreqGiftName/topFreqGiftPictureUrl이 DonorEntry에 string으로 정의됨**: `maxSingleGiftName?: string`은 optional인데 `topFreqGiftName: string`은 required. getLeaderboard 결과에서 topFreqGiftName이 빈 문자열("")로 들어갈 수 있음 — 오버레이에서 분기 처리 필요. 기존 getCurrentTop에도 동일 패턴이므로 회귀 아님.

## 보완 검증 — UTC/KST 동일 패턴 전수조사

W0 수정(cutoffs.ts today/yesterday KST화) 이후 동일한 패턴의 버그가 다른 위치에 잔존하는지 전체 grep 수행.

| 위치 | 패턴 | 컬럼 | 판정 |
|------|------|------|------|
| `apps/worker/src/routes/cutoffs.ts:12-14` | KST offset | `league_date` | ✅ 수정됨 (이번 세션 W0) |
| `apps/desktop/electron/main/ipc.ts:1326-1327` | KST offset + 주석 | `league_date` 쿼리 | ✅ 이미 정확 — 수정 전부터 KST 오프셋 있었음 |
| `apps/worker/src/lib/rank-cron.ts:88` | UTC | `broadcaster_rank_snapshots` | ✅ KST 불필요 (다른 테이블, UTC 저장) |
| `apps/worker/src/lib/rank-cron.ts:225` | KST | `league_date` | ✅ 이미 정확 |
| `apps/desktop/electron/main/ipc.ts:1482` | UTC | `broadcaster_rank_snapshots` | ✅ KST 불필요 (다른 테이블) |
| `apps/worker/src/lib/rank-browser.ts:66` | UTC | rank 관련 | ✅ KST 불필요 |
| `apps/worker/src/lib/rank-collector.ts:48` | UTC | rank 관련 | ✅ KST 불필요 |

**결론: `league_date` 컬럼을 쿼리하는 모든 위치가 KST 또는 이미 KST로 수정됨. rank 테이블(`broadcaster_rank_snapshots`)은 UTC 저장이므로 UTC 사용이 정확. KST/UTC 혼용 버그 추가 발견 없음.**

## 미검증 가정
- `giftEffectId` 필드 세팅 여부 — **해소**: tiklive-normalizer.ts:45,67에서 세팅 확인
- desktop + worker emoji-rain/welcome-banner/gift-tornado HTML의 내용이 byte-for-byte 동일한지 — 로직 drift 가능성 낮음(perf-patch 차이만 예상)

## 엣지케이스 시나리오
- **선물 0개 상태**: getLeaderboard([]) → 빈 배열 반환 — 오버레이 데이터 없음 상태. TopDonorPayload? null이 아닌 빈 배열이므로 오버레이에서 빈 리스트 핸들링 필요 (기존 동작 그대로)
- **TTLS 연결 끊김 상태에서 emoji_rain/welcome/tornado 호출**: command-service는 WebSocket broadcast라 연결 끊김 시 무응답 — 사일런트 페일. 기존 패턴과 동일.
- **nowSec 경계 케이스 — samRows 비어있을 때**: `groups = []`, `closestTs = groups[0] ?? ""` → `""`. `samRows.filter(r => r.created_at === "")` → 빈 배열. 이미 처리됨 ✅
- **KST 자정 경계 (00:00~00:59 KST)**: today=KST 당일, yesterday=KST 전날. nowSec은 UTC 시각(KST 09:00~09:59에 해당). samRows가 어제 UTC 09:xx에 수집된 데이터 → "같은 시간대" 정확히 매칭 ✅
- **getLeaderboard에서 maxGiftCoins=0 케이스**: `e.maxSingleCoins ?? 0` — maxSingleCoins가 null/undefined면 0 반환. 오버레이에서 0 표시 가능 (기존 getCurrentTop과 동일 패턴)

## 결론

**통합 진행 가능.**

치명 리스크 0건. 4종 typecheck 실제 실행 PASS 확인. 주요 변경사항(GiftEvent 필드/OverlayMessageType union/cutoffs KST) 모두 정합.

보완 검증 완료:
- `giftEffectId` — dead field 의심 해소. tiklive-normalizer.ts + EventOverlay.tsx에서 실제 사용 확인.
- UTC/KST 동일 패턴 전수조사 완료 — `league_date` 쿼리 지점 전부 KST 정확, rank 테이블 UTC 정확. 추가 버그 없음.

주의 사항 1건 잔존(tiklive.ts cast 우회 일부 잔존) — 동작 영향 없음, 별도 정리 후보.

# DEAD_ENDS (시도했으나 실패한 접근)

## 2026-06-04 — desktop-dev: OverlayMessage interface에 emoji?/user? 필드 추가
- 시도: union 추가 외에 `OverlayMessage` interface에 `emoji?: string` + `user?: {nickname,uniqueId}` 추가.
- 실패: `overlay-rules-service.ts:159`의 fireworks broadcast가 `user`를 `unknown` 필드들로 구성 → 새 타입 에러 유발. 또한 const-변수 broadcast 패턴상 애초에 불필요.
- 결론: union literal만 추가하면 충분. interface 필드 추가 금지.

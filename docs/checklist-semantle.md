# 시맨틀 라이브 게임 — 체크리스트

TikTok LIVE 시청자 채팅으로 꼬맨틀류 단어 유사도 게임을 함께 진행하는 기능.

## Phase 1 — 꼬맨틀 단일 게임 end-to-end

### 백엔드 (electron)
- [x] `services/semantle-service.ts` — 게임 상태, 어댑터 인터페이스, 캐시, throttle 큐, 리더보드, 정답 감지, 게임번호 파싱, 브로드캐스트
- [x] `services/semantle/types.ts` — `GameAdapter` 인터페이스 + `NormalizedGuess` 정규화 형태
- [x] `services/semantle/komantle-adapter.ts` — 꼬맨틀 어댑터 (`/guess/<n>/<word>`, 게임번호 파싱)
- [x] 채팅 구독 — `eventBus.subscribe("chat", ...)` → 한글단어(`^[가-힣]+$`) 추출 → guess
- [x] 외부 fetch — ipc.ts `browserFetch` 주입 (net.fetch 직접 금지)
- [x] IPC 핸들러 (ipc.ts) — start/stop/reset, setActiveGame, hostGuess, getState
- [x] preload — `semantle` API 노출 + 상태 push 구독

### 오버레이
- [x] `public/overlays/semantle.html` (desktop) — 실시간 리더보드 보드
- [x] `public/overlay/semantle.html` (worker) — 동일 동기화
- [x] postMessage 핸들러 + WS(cloud+local) 연결 + reload 패치
- [x] `cloud-overlay.ts` getUrls 등록
- [x] `overlay-server.ts` OverlayMessageType에 `semantle` 추가
- [x] `EventOverlay.tsx` 크기/이름 등록 + 테스트 버튼

### 호스트 패널 (renderer)
- [x] `src/pages/Semantle.tsx` — 게임 시작/리셋, 게임 선택(꼬맨틀), 호스트 직접 추측, 실시간 보드 미러, 게임번호/임계값 표시
- [x] 네비게이션 등록

### 검증
- [x] `pnpm --filter @tikke/desktop typecheck`
- [ ] 오버레이 테스트 링크로 보드 렌더 확인
- [ ] 채팅 시뮬레이션(harness 또는 호스트 추측)으로 guess→보드 흐름 확인

## Phase 2 — 멀티 게임 (완료)
- [x] 꼬오맨틀 (koo.bfy.kr) API 리버스 → 어댑터 (세션형 /start + word-sess 헤더)
- [x] 뉴맨틀 (newmantle.kr) API 리버스 → 어댑터 (일일 날짜형 + x-guest-id/Origin)
- [x] GameAdapter 인터페이스 일반화 (gameKey 문자열, startGame)
- [x] 호스트 패널 게임 선택 — listGames가 3종 자동 노출
- [x] 3개 어댑터 실 API 스모크 검증

## 리스크
- 서드파티 서버 의존(API 계약 없음) → 캐시+throttle로 부하 최소화. 플래그십 되면 FastText 자체호스팅 검토.

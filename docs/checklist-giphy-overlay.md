# Giphy GIF 검색 → 오버레이 체크리스트

목표 — 방송인이 앱에서 Giphy GIF 검색 → 클릭하면 방송 오버레이에 표시. 지울 때까지 고정.

정책 — 수동 트리거(검색→클릭), 지울 때까지 고정, 키는 Worker secret.

## [1] Worker Giphy 프록시 ✅
- [x] `lib/auth.ts` Env — GIPHY_API_KEY 추가
- [x] `routes/giphy.ts` — `/api/giphy/search?q=` 프록시(requireAuth, 결과 정규화)
- [x] `index.ts` 라우팅
- [x] 검증: `pnpm --filter @tikke/worker typecheck` 통과

## [2] 오버레이 HTML ✅
- [x] `apps/desktop/public/overlays/giphy.html` — tikke-reload-patch, room WS, message 핸들러, ?scale ?test
- [x] 메시지: `{type:'giphy-show', gif:{url}}` / `{type:'giphy-clear'}`
- [x] `apps/worker/public/overlay/giphy.html` 동일 복사
- [x] `overlay-server.ts` OverlayMessageType += giphy-show/giphy-clear, OverlayMessage.gif 필드
- [x] SYNC (고정 상태 → 재연결 복원): ipc lastSyncCache "giphy-show" set/delete + OverlayRoom giphy-show 영속(TTL)/giphy-clear 삭제
- [x] `cloud-overlay.ts` getUrls += "GIF" 라벨/URL
- [x] `EventOverlay.tsx` OVERLAYS 카드, ID_TO_LABEL("GIF"), RECOMMENDED_SIZE, CATEGORY_MAP

## [3] Desktop 검색 UI + 트리거 ✅
- [x] `pages/GiphyBrowser.tsx` — 검색창 + 결과 그리드, 클릭 시 오버레이 push, 지우기 버튼
- [x] IPC: `tikke:giphy:search`(worker 프록시 browserFetch + supabase token) + 트리거는 기존 `tikke:overlay:send` 재사용(giphy-show/giphy-clear)
- [x] preload 노출 (giphy.search)
- [x] 네비게이션에 페이지 추가 (Sidebar "GIF 브라우저", App.tsx route)
- [x] 검증: desktop typecheck 통과 / 오버레이 ?test=1 실측은 앱 실행 시

## [4] 배포 전
- [ ] `wrangler secret put GIPHY_API_KEY` (← K가 직접 실행, 키 발급 완료)
- [x] 오버레이 worker 동기화 확인 (desktop + worker/public/overlay 양쪽 HTML 존재)

## 미결정/메모
- 고정 상태 앱 재시작 후 복원? → 런타임 SYNC만 (OverlayRoom 12h TTL, 영구저장 생략)
- rating=pg-13 필터(방송 적합)
- GIPHY_API_KEY 없으면 search 503 "Giphy not configured" — 정상. secret 등록(배포) 또는 로컬 `.dev.vars` 추가 후 동작.

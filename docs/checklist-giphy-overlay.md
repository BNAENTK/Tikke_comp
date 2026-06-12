# Giphy GIF 검색 → 오버레이 체크리스트

목표 — 방송인이 앱에서 Giphy GIF 검색 → 클릭하면 방송 오버레이에 표시. 지울 때까지 고정.

정책 — 수동 트리거(검색→클릭), 지울 때까지 고정, 키는 Worker secret.

## [1] Worker Giphy 프록시 ✅
- [x] `lib/auth.ts` Env — GIPHY_API_KEY 추가
- [x] `routes/giphy.ts` — `/api/giphy/search?q=` 프록시(requireAuth, 결과 정규화)
- [x] `index.ts` 라우팅
- [x] 검증: `pnpm --filter @tikke/worker typecheck` 통과

## [2] 오버레이 HTML
- [x] `apps/desktop/public/overlays/giphy.html` — tikke-reload-patch, room WS, message 핸들러, ?scale ?test
- [x] 메시지: `{type:'giphy-show', gif:{url}}` / `{type:'giphy-clear'}`
- [x] `apps/worker/public/overlay/giphy.html` 동일 복사
- [ ] `overlay-server.ts` OverlayMessageType += giphy-show/giphy-clear
- [ ] SYNC (고정 상태 → 재연결 복원): ipc SYNC_TYPES + OverlayRoom SYNC_TYPES/SYNC_ALWAYS
- [ ] `cloud-overlay.ts` getUrls += 라벨/URL
- [ ] `EventOverlay.tsx` OVERLAYS 카드, ID_TO_LABEL, RECOMMENDED_SIZE, CATEGORY_MAP

## [3] Desktop 검색 UI + 트리거
- [ ] `pages/GiphyBrowser.tsx` — 검색창 + 결과 그리드, 클릭 시 오버레이 push, 지우기 버튼
- [ ] IPC: `tikke:giphy:search`(worker 프록시 호출) + `tikke:giphy:show`/`tikke:giphy:clear`(오버레이 push)
- [ ] preload 노출
- [ ] 네비게이션에 페이지 추가
- [ ] 검증: desktop typecheck + 오버레이 ?test=1 실측

## [4] 배포 전
- [ ] `wrangler secret put GIPHY_API_KEY`
- [ ] 오버레이 worker 동기화 확인

## 미결정/메모
- 고정 상태 앱 재시작 후 복원? → 일단 런타임 SYNC만(영구저장 생략)
- rating=pg-13 필터(방송 적합)

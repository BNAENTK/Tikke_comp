# 덱 (스트림덱 연동 + 앱 내 버튼 격자)

- [x] `electron/services/deck.ts` — 액션 카탈로그 + 토큰 + runner
- [x] `settings.ts` — `deckToken`, `deckLayout` 키 추가
- [x] `overlay-server.ts` — `/deck/actions`, `/deck/run/<id>` (토큰 필수)
- [x] `main/index.ts` — `setDeckRunner` (창 없으면 false → HTTP 503)
- [x] `main/ipc.ts` — setCatalog / getInfo / rotateToken / saveLayout
- [x] `preload/index.ts` — `tikke.deck.*`
- [x] `src/lib/deckActions.ts` — 액션 8종 (연결 2 / TTS 4 / 오버레이 1 / 덱 1)
- [x] `src/pages/Deck.tsx` — 5×8 격자, 편집, URL 복사, 키 교체, 작은 창 버튼
- [x] `src/pages/DeckPip.tsx` — 항상 위 3열 PiP 창 (frame:false, alwaysOnTop screen-saver)
- [x] `tikke:deck:press` — PiP/본창 공통 실행 경로 (PiP 는 별도 스토어라 직접 실행 금지)
- [x] 액션 `deck:pip` — 스트림덱 버튼으로 덱 창 자체를 띄우기
- [x] `App.tsx` — 카탈로그 등록 useEffect + `case "deck"`
- [x] `Sidebar.tsx` — 페이지 유니온 + 시스템 그룹 메뉴
- [x] `pnpm --filter @tikke/desktop typecheck`
- [x] 실측: 403(잘못된 키) / 200(목록) / 200(실행) / 404(없는 액션)
- [x] 실측: 창 숨긴 상태에서 실행 → 200 + ttsEnabled false→true 실제 변경 확인
- [x] 실측: HTTP로 deck:pip → 항상 위 PiP 창 실제로 뜸 (창 2개 확인)
- [ ] 미실측: 창을 완전히 파괴한 상태(트레이 종료) → 503. 코드상 isDestroyed 로 처리

## 안 만든 것 (필요해지면)
- 장면(scene) 여러 개, 격자 크기 변경, 배경/이름 편집 — TIK SCAN 스크린샷에는 있지만 추측 기능
- 액션 추가: 사운드 정지, 타이머 시작/정지, 룰렛 돌리기 등

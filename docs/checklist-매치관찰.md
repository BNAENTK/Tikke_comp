# 체크리스트 — 룸 관찰 매치 현황 (외부 대조 사이트 스타일)

목표: 본인이 참여 중인 PK/매치의 실시간 상황(팀, 호스트별 매치 파워, 후원자 순위·후원량)을 룸 관찰 페이지에 표시.

- [x] `match-observer.ts` 신설 — linkMicBattle/linkMicArmies 집계, MatchState 보관
- [x] `tiklive.ts` — linkMicBattle 기존 리스너에 매치 집계 추가 + linkMicArmies 리스너 연결
- [x] `ipc.ts` — `tikke:rooms:matchState` 핸들러
- [x] `preload/index.ts` — `rooms.matchState()` 노출
- [x] `RoomObserver.tsx` — 매치 패널 UI (팀별 호스트 카드 + 후원자 랭킹, 3초 폴링)
- [x] 검증 — `pnpm --filter @tikke/desktop typecheck` 통과
- [ ] 실전 검증 — 실제 매치 진입 후 필드 확인 (1회 raw 로그로 구조 검증)

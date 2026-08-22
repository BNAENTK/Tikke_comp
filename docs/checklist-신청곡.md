# 유튜브 신청곡 체크리스트

설계 근거는 `context-notes-신청곡.md`.

## 1단계 — 뼈대
- [x] `electron/services/song-request.ts` — 큐 상태, yt-dlp 검색, 추가/스킵/삭제
- [x] `ytmp3-service.ts` 에 yt-dlp 경로 노출 (검색에 재사용)
- [x] `command-service.ts` 내장 명령 `!신청` 추가 (`!리그` 와 같은 자리)
- [x] `settings.ts` 키 추가 — 활성화/쿨다운/최대길이/큐상한/1인당/권한
- [x] IPC + preload

## 2단계 — 화면
- [x] `public/overlays/song-request.html` — 유튜브 IFrame 플레이어 + 곡 정보
- [x] `apps/worker/public/overlay/song-request.html` 동일 사본
- [x] `overlay-server.ts` 메시지 타입 추가 + `/song/report/` 경로
- [x] `cloud-overlay.ts` getUrls 에 라벨·URL
- [ ] `EventOverlay.tsx` 카드·라벨·권장 크기·카테고리
- [x] `src/pages/SongRequest.tsx` — 큐 목록, 스킵/삭제, 설정
- [x] `App.tsx` case + `Sidebar.tsx` 메뉴

## 추가 (2026-08-22)
- [x] 신청곡 화면에 "구성요소 다시 설치" 버튼 — `tikke:song:reinstall` IPC → `ensureYtdlp(force)`. 자동 갱신(3주 낡음·차단 감지)이 못 잡는 손상 상태 탈출구. 음원 추출 화면과 동일 패턴.

## 남은 것
- [ ] `EventOverlay.tsx` 카드 등록 (지금은 사이드바 '유튜브 신청곡' 화면과 오버레이 URL 목록으로 접근)
- [ ] 실제 방송에서 `!신청` 채팅 흐름 확인 (라이브 연결 필요)

## 검증
- [ ] `!신청 <검색어>` → 큐에 들어가는지
- [ ] 큐가 비어 있다가 첫 곡이 들어오면 **자동으로 재생 시작** (TIK SCAN 결함 회피)
- [ ] 곡 종료 시 다음 곡으로 자동 진행
- [ ] 스킵/삭제가 오버레이에 즉시 반영
- [ ] 쿨다운이 **큐 진입 후** 기록되는지 (검색 실패 시 재시도 가능)
- [ ] 자동재생 차단 시 안내가 뜨는지
- [x] `pnpm --filter @tikke/desktop typecheck`
- [x] 오버레이 `?test=1` 실측 — 카드 표시·곡 로드·차단 감지·클릭 폴백 후 진행률 1.26% 확인

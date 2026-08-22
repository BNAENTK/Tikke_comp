# 체크리스트 — 미니게임 (합10 사과 / 컬러 매치)

목표: 사과게임·컬러타일 규칙을 tikke에 직접 구현한다. 시청자는 채팅 좌표로,
tikke 앱 사용자끼리는 방코드 룸에서 대전한다.

## 0. 설계 · 공통

- [x] 원본 규칙 확보 (사과 10×17 합10 / 타일 4방향 2매치, 헛클릭 -10초)
- [x] 참여 방식 확정 (채팅 좌표 + 앱 사용자 룸 대전 둘 다)
- [x] `checklist-미니게임.md` / `context-notes-미니게임.md` 작성
- [x] `packages/shared/src/minigame.ts` — 시드 RNG, 보드 생성, 이동 검증, 점수
  - [x] `mulberry32` 시드 RNG (같은 시드 → 같은 보드)
  - [x] `createAppleBoard(seed)` 10×17, 1~9
  - [x] `applyAppleMove(board, r1,c1,r2,c2)` 직사각형 합 10 검증 + 제거
  - [x] `createTilesBoard(seed)` 200 타일 배치
  - [x] `applyTilesMove(board, r, c)` 빈 칸 4방향 스캔, 2방향 이상 동색 제거
  - [x] `parseCoord("B3")` → {r, c} / 범위 검증
  - [x] 순수 함수만 (DOM·Node API 금지 — worker DO에서도 씀)
- [x] 노드 스크립트로 로직 검증 (30 pass) (같은 시드 재현성, 합10 판정, 4방향 스캔)

## 1. 방송 코옵 (시청자 채팅 참여)

- [ ] `electron/services/minigame-service.ts`
  - [ ] `eventBus.subscribe("chat")` + `chatRoleEligible` 역할 게이팅
  - [ ] 좌표 명령 파싱 (`!사과 B3 D5`, `!타일 C7`)
  - [ ] 사용자별 쿨다운 (기본 5초) — 헛클릭 -10초 대신
  - [ ] 점수판 누적 (닉네임별 제거 개수)
  - [ ] 타이머 (기본 5분, 설정 가능)
  - [ ] 선물 보너스 옵션 (시간 +N초)
  - [ ] streamEnd 로 자동 stop 걸지 말 것 (일시중지 오탐)
- [ ] `settings.ts` 키 추가 (게임 종류, 시간, 쿨다운, 최소 역할, 선물 보너스)
- [ ] `ipc.ts` 핸들러 + `preload/index.ts` 노출

## 2. 룸 대전 (tikke 앱 사용자끼리)

- [ ] `apps/worker/src/durable/GameRoom.ts`
  - [ ] 6글자 방코드 발급 / 참가
  - [ ] 시드 배포 (같은 방 = 같은 보드)
  - [ ] 호스트 시작, 서버 타이머
  - [ ] 이동 검증 서버측 (DO가 보드 상태 보유 — 점수 조작 방지)
  - [ ] 점수 실시간 브로드캐스트
- [ ] `wrangler.toml` DO 바인딩 + migration tag v2 (`new_sqlite_classes`)
- [ ] `apps/worker/src/index.ts` 라우팅 `/game/rooms/...`
- [ ] `apps/worker/src/durable/GameRoom.ts` export

## 3. 앱 UI

- [ ] `src/pages/MiniGame.tsx`
  - [ ] 게임 선택 (사과 / 타일)
  - [ ] 모드 선택 (방송 코옵 / 룸 대전 / 혼자)
  - [ ] 보드 직접 플레이 (드래그·클릭)
  - [ ] 방 만들기 / 참가 (방코드)
  - [ ] 설정 (시간, 쿨다운, 최소 역할, 선물 보너스)
- [ ] 사이드바 라우팅 등록

## 4. 오버레이

- [ ] `public/overlays/minigame.html` — 보드 + 타이머 + 랭킹
  - [ ] tikke-reload-patch 보일러플레이트
  - [ ] `room` 파라미터 + 로컬 18182 / 클라우드 WS 분기
  - [ ] `window.addEventListener("message")` 핸들러 (없으면 미리보기 먹통)
  - [ ] `?scale=`, `?test=1`
  - [ ] 좌표 라벨(A~Q / 1~10) 크게 — 채팅 입력용이라 가독성이 핵심
- [ ] `apps/worker/public/overlay/minigame.html` 동일 사본
- [ ] `overlay-server.ts` `OverlayMessageType` 추가
- [ ] `ipc.ts` `SYNC_TYPES` + `OverlayRoom.ts` `SYNC_TYPES`/`SYNC_ALWAYS`
- [ ] `cloud-overlay.ts` `getUrls()` 한국어 라벨 + URL
- [ ] `EventOverlay.tsx` `OVERLAYS` 카드 / `ID_TO_LABEL` / `RECOMMENDED_SIZE` / `CATEGORY_MAP` / testActions

## 5. 검증

- [x] `pnpm --filter @tikke/shared typecheck`
- [ ] `pnpm --filter @tikke/desktop typecheck`
- [ ] `pnpm --filter @tikke/worker typecheck`
- [ ] 오버레이 `?test=1` playwright 실측 (보드 잘림·라벨 가독성)
- [ ] 채팅 하네스로 좌표 명령 흐름 확인
- [ ] 룸 대전 2인 시뮬 (같은 시드 보드 일치 확인)

## 6. 배포

- [ ] 의미 단위 커밋
- [ ] 버전 +1 → 태그 → `git push origin main vX.Y.Z`
- [ ] 사용자가 "배포해" 할 때만

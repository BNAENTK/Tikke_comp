# 자동 좋아요 (입력 자동화) 체크리스트

목표 — Windows `user32.dll SendInput`으로 더블클릭 + L키를 10초당 40회(250ms 간격) 자동 입력.
틱톡 API 미사용. 포그라운드 창 대상.

## 구현

- [x] koffi 의존성 추가 (`apps/desktop` dependencies) + `build.asarUnpack`에 koffi 경로
- [x] `electron/services/auto-like.ts` 신규 — SendInput 래퍼, 틱 루프, 포그라운드 가드, 중복 시작 방지
- [x] `electron/services/settings.ts` — autoLike* 키 추가 (인터페이스 + DEFAULTS 양쪽)
- [x] `electron/main/ipc.ts` — `tikke:autolike:start|stop|status` 핸들러
- [x] `electron/preload/index.ts` — `window.tikke.autoLike.*` 노출 (start/stop/status/onStatus)
- [x] 전역 단축키 비상 정지 등록 + 앱 종료 시 해제 (`app.on("will-quit")`)
- [x] `src/pages/AutoLike.tsx` 신규 페이지 (로컬 TikkeWindow 타입 포함)
- [x] `src/components/Sidebar.tsx` — SidebarPage 유니온 + 메뉴 항목
- [x] `src/app/App.tsx` — import + case
- [x] `electron/main/index.ts` — `autoLikeService.init(mainWindow)`

## 검증

- [x] `pnpm --filter @tikke/desktop typecheck` 통과
- [x] `pnpm build:desktop` 통과 + 번들에 `require("koffi")` 유지 확인 (externalize 정상)
- [x] koffi FFI 시그니처 실측 (SendInput / GetForegroundWindow / GetWindowTextW 전부 동작)
- [x] 포그라운드 가드 — 대상 창이 앞이 아니면 입력을 보내지 않고 중단 (스모크 테스트에서 실제 발동)

### 남은 검증 — 전체화면 게임이 포커스를 점유 중이라 미실행

`node autolike-smoke.cjs` (apps/desktop에서 실행). 메모장을 띄워 L키 12회 주입 →
클립보드로 읽어 개수 대조 + 마우스 INPUT 수락 여부를 자동 판정한다.
전체화면 앱이 떠 있으면 메모장이 앞으로 못 나와서 스스로 중단한다. 게임 종료 후 실행할 것.

- [ ] `node autolike-smoke.cjs` PASS (키 주입 정확도 + 마우스 INPUT 수락)
- [ ] `pnpm dev` — 실제 UI에서 시작/정지 동작
- [ ] 메모장에서 더블클릭이 단어 선택 되는지 (싱글클릭 2회 아님)
- [ ] 전역 단축키 Ctrl+Shift+F9로 즉시 정지되는지
- [ ] 시작 버튼 연타해도 속도 안 빨라지는지 (중복 인터벌 방지)
- [ ] 자동 종료 타이머 동작
- [ ] 테스트 후 electron.exe 전부 kill + `Get-Process`로 확인
- [ ] `pnpm dist:win` 패키징본에서 koffi 로드 확인 (asarUnpack 검증)

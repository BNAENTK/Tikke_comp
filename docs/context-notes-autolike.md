# 자동 좋아요 — 결정 사항과 이유

## 범위 정의 (최초 오해 정정)

"자동 좋아요"는 **틱톡 API로 좋아요를 전송하는 기능이 아니다**. 이름만 자동 좋아요고 실제 동작은
OS 입력 자동화 — 자동 더블클릭 + 자동 L키 입력이다. 사용자가 명시적으로 정정함.

틱톡 API 경로를 배제한 이유.
- `tiklive.ts`의 `WebcastPushConnection`은 수신 전용. 전송 API 없음.
- 전송하려면 디바이스 서명(X-Bogus 등) 자체 생성 필요 — EulerStream은 연결용 서명만 제공.
- ToS 위반 + 계정 밴 위험.

입력 자동화는 이 문제가 전부 없다.

## 왜 koffi인가

`user32.dll`의 `SendInput`을 호출해야 한다. 후보 비교.

- **koffi** (채택) — 프리빌트 N-API FFI. electron-rebuild 불필요.
- robotjs — 네이티브 리빌드 필요 + 유지보수 중단. 리빌드 실패 = 앱 검은 화면(기존 함정 목록과 충돌).
- PowerShell 스폰 — 250ms마다 프로세스 생성은 비현실적.
- `webContents.sendInputEvent` — Electron 자기 창에만 들어감. 외부 앱 불가.

`asarUnpack`에 koffi 경로 추가 필수. 기존엔 `onnxruntime-node`만 있음. 빠뜨리면
`pnpm dev`는 되는데 설치본에서 네이티브 모듈 로드 실패하는 전형적 사고가 난다.

## 포그라운드 제약 — 설계를 지배하는 사실

`SendInput`은 **포그라운드 창에만** 입력을 넣는다. 백그라운드 특정 창 지정은 불가능하다.
`PostMessage(WM_LBUTTONDOWN)`으로 HWND를 직접 노리는 우회법이 있으나 TTLS/브라우저는
Chromium이라 합성 윈도우 메시지를 무시한다. 그래서 우회 설계는 하지 않는다.

대신 안전장치를 건다.
1. 시작 시 카운트다운 → 끝나는 순간 `GetForegroundWindow()`로 대상 HWND 고정.
2. 매 틱마다 현재 포그라운드가 대상과 같은지 비교. 다르면 그 틱 스킵.

가드가 없으면 알림창이 포커스를 뺏는 순간 40회/10초의 더블클릭과 L키가 엉뚱한 UI에 박힌다.
선택이 아니라 필수다.

## 타이머는 main 프로세스에

대상 창이 앞에 있으면 Tikke는 백그라운드다. 렌더러 `setInterval(250)`은
`backgroundThrottling`(켜둔 상태가 정답 — 기존 확인 사실)에 걸려 초당 1회로 떨어진다.
그래서 틱 루프는 main의 서비스에 둔다. 렌더러는 start/stop/상태 표시만 한다.

## 더블클릭은 클릭 2회가 아니다

250ms 간격 클릭 두 번은 윈도우가 더블클릭으로 인식하지 않는다(`GetDoubleClickTime` 기본 500ms).
한 틱 안에서 down/up/down/up을 짧은 간격(50ms)으로 보내야 한다.
이걸 놓치면 조용히 싱글클릭 두 번이 되고, 증상만 봐선 원인을 못 찾는다.

## 커서 이동 안 함

`MOUSEEVENTF_MOVE` 없이 현재 커서 위치에 down/up만 보낸다.
좌표 설정 UI가 통째로 사라지고, 사용자 마우스를 뺏지 않는다.
사용자는 커서를 좋아요 누를 지점에 올려두기만 하면 된다.

## 정지 수단

Tikke 창이 뒤에 있어서 정지 버튼 클릭이 어렵다. 사용자의 진짜 클릭도 합성 클릭과 경쟁한다.
그래서 `globalShortcut` 비상 정지를 건다. 이 저장소에 `globalShortcut` 선례가 없어 신규 도입이다.
`register()`는 다른 앱이 조합을 점유 중이면 false를 반환한다 — 실패를 UI에 표시해야 한다.
앱 종료 시 unregister. 자동 종료 타이머를 두 번째 안전벨트로 둔다.

## koffi FFI 시그니처 (실측 확인)

추측하지 않고 프로브로 확인한 값들이다.

- `uintptr_t GetForegroundWindow()` → koffi가 일반 Number로 반환(HWND 값이 작아 정밀도 문제 없음)
- `int GetWindowTextW(uintptr_t hWnd, void *lpString, int nMaxCount)` → Buffer 넘기고 `utf16le`로 디코드
- `unsigned int SendInput(unsigned int cInputs, void *pInputs, int cbSize)` → Buffer 직접 전달
- x64 INPUT 구조체 = 40바이트. 마우스는 dwFlags@20, 키보드는 wVk@8 / dwFlags@12

`electron.vite.config.ts`의 `externalizeDepsPlugin`이 dependencies를 전부 external 처리하므로
빌드 산출물에 `require("koffi")`가 그대로 남는다(번들링 안 됨). 빌드 후 확인함.

구조체 오프셋은 손으로 쓴 값이라 별도로 실측했다. `koffi.struct`/`koffi.union`으로 같은 구조체를
선언하고 `sizeof`/`offsetof`를 찍어 코드가 쓰는 상수와 대조 — 6개 전부 일치(sizeof 40, union base 8,
mouse dwFlags 20, key wVk 8, key dwFlags 12). 레이아웃이 틀리면 `SendInput`이 0을 반환하고
아무 일도 안 일어나는데, 타입 체크나 빌드로는 절대 안 잡힌다.

## Tikke 자기 창을 대상으로 잡는 사고 방지

카운트다운 종료 시점의 포그라운드를 대상으로 고정하는데, 이때 Tikke가 앞에 있으면
자기 UI에 초당 4회 더블클릭과 L키가 쏟아진다. 발생 경로가 둘이다.
- 대기 시간을 0으로 두면 시작 버튼을 누른 직후라 포그라운드가 Tikke다.
- 대기 시간 중에 사용자가 Tikke로 alt-tab 해도 마찬가지다.

그래서 UI에서 최소값을 막는 것으로는 부족하고 서비스에서 걸러야 한다.
`BrowserWindow.getAllWindows()`의 네이티브 핸들과 대조해 자기 창이면 시작을 거부하고
에러 메시지를 띄운다(PiP 등 보조 창까지 포함). UI 최소 대기 시간도 1초로 올렸다.

## 스모크 테스트

`apps/desktop/autolike-smoke.cjs` — 메모장을 띄워 L키 12회를 실제로 주입하고
Ctrl+A/Ctrl+C 후 클립보드를 읽어 개수를 대조한다. 마우스 INPUT 수락 여부도 같이 본다.

타입 체크로는 FFI 구조체 레이아웃이 맞는지 알 수 없어서 만들었다.
스크립트 자체에도 포그라운드 가드가 들어 있어 메모장이 앞으로 나오지 못하면
입력을 한 건도 보내지 않고 중단한다. 전체화면 게임이 떠 있을 때 실제로 이 경로로 중단됐다
(= 가드가 의도대로 동작함을 부수적으로 확인).

## 중복 시작 방지

start를 연타하면 인터벌이 중첩돼 속도가 배로 뛴다.
start 진입 시 항상 기존 인터벌을 먼저 clear한 뒤 새로 건다(= 재시작 시맨틱).
stop과 `app.on("will-quit")`에서도 정리한다.

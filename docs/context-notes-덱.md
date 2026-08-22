# 덱 — 결정과 이유

## 방향
처음엔 tikke가 TTLS Stream Deck 소켓(28189 등 7포트, 서브프로토콜 `streamdeck_ttls_v1`)에
붙어 **TTLS를 조종하는** 쪽으로 파고들었다. K가 바로잡음 — 의도는 반대다.
**스트림덱이 tikke를 조종한다.** TTLS 소켓 리버스 결과는 폐기하지 않고
`context-notes-tikscan-reverse.md`에 남겨둠(나중에 쓸 수 있음).

## 왜 플러그인이 아니라 HTTP URL인가
엘가토 플러그인은 만들고 심사받고 설치시켜야 한다. 반면 스트림덱 기본 액션 **Website**는
URL 하나만 넣으면 끝이다. 게다가 매크로 프로그램·폰 앱·다른 회사 덱도 URL은 다 열 수 있다.
그래서 액션마다 `http://localhost:18181/deck/run/<id>?k=<토큰>` 을 만들어주고
앱에는 복사 버튼만 둔다.

## 토큰이 필요한 이유 (넣은 이유)
오버레이 HTTP 서버는 `0.0.0.0`에 열려 있다(폰/다른 PC에서 오버레이를 보게 하려고).
토큰이 없으면 두 가지가 뚫린다.
1. 같은 공유기 안 아무 기기 → `/deck/run/live:disconnect` 호출 가능
2. 사용자가 열어둔 아무 웹페이지 → `<img src="http://localhost:18181/deck/run/...">` 한 줄이면
   GET이 나간다. CORS는 **응답 읽기**만 막지 부작용이 있는 요청 자체는 못 막는다.

토큰은 `deckToken` 설정에 저장(12바이트 hex). 앱에서 "키 새로 만들기"로 교체 가능 —
교체하면 이전에 복사해둔 URL은 전부 무효가 된다는 걸 툴팁에 적어둠.

기존 `/control/`은 토큰이 없다. 번역 창 전용이고 이번 변경 범위 밖이라 건드리지 않음.
(다만 같은 취약점이 있음 — 별건으로 다룰 것)

## 액션 본체를 렌더러에 둔 이유
tikke 기능 대부분이 zustand 스토어에 있다. main에서 직접 부를 수 없다.
그래서 렌더러가 카탈로그를 main에 올리고, HTTP 요청이 오면 main이 IPC로 렌더러에 넘긴다.

**함정**: `App.tsx renderPage()`는 스위치라 페이지가 한 번에 하나만 마운트된다.
덱을 누르는 시점엔 tikke가 뒤에 있고 어떤 페이지가 열려 있을지 모른다.
→ 액션 등록은 반드시 **App 레벨**, 액션 본체는 `useXStore.getState()` / `window.tikke.*` 로만.
페이지 로컬 콜백을 쓰면 그 페이지에 앉아 있을 때만 동작한다.

## 창이 없을 때
`mainWindow?.webContents.send()`는 창이 없어도 조용히 아무 일도 안 한다.
그대로 두면 HTTP는 200을 돌려주고 스트림덱은 성공한 줄 안다.
→ runner가 `false`를 돌려주게 하고 HTTP는 **503**을 낸다.

## 격자 UI 범위
TIK SCAN 스크린샷에는 장면 탭(`+ Nova cena`), GRID 5×8 셀렉터, 배경/이름 편집이 있다.
전부 만들면 큰 편집기가 되고 아직 필요한지 모른다. 그래서 이번엔
**한 장면 / 5×8 고정 / 액션 배치 + 비우기 / 슬롯별 URL 복사**까지만.
배치는 `deckLayout` 설정에 JSON 배열로 저장하므로 나중에 장면을 늘려도 스키마 확장으로 처리 가능.

## 액션 목록 (8종, 시작점)
연결: `live:connect`(`lastConnectedUsername` 사용) / `live:disconnect`
TTS: `tts:toggle` / `tts:on` / `tts:off` / `tts:clear`
오버레이: `overlay:reload`
덱: `deck:pip` (덱 PiP 창 열기)

TTS on/off는 스토어와 `ttsEnabled` 설정을 같이 바꾼다 — Sidebar 토글과 동일하게.
안 그러면 재시작 때 되돌아간다.

## PiP 창
별도 문서로 분리 — `context-notes-덱-pip.md`

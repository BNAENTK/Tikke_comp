# PIP 창 전역 엔진 중복 — 사과게임 TTS 두 번 읽힘

## 증상
사과게임을 할 때 TTS 가 두 번 읽혔다.

## 원인
PIP 창은 본창과 **같은 앱 URL** 을 로드한다(`?minigamepip=1#minigame-pip`).
그래서 `App` 이 통째로 한 번 더 렌더되고 그 안의 전역 훅도 같이 돈다.

PIP 분기(`if (isMiniGamePip) return <MiniGamePip />`)는 517줄인데 훅 호출은 272줄이라
**분기보다 먼저 실행된다**. 훅 규칙(조건부 호출 금지) 때문에 그렇게 둔 것이고,
코드에도 "hook을 conditional return 전에 다 호출"이라고 적혀 있다.

훅들은 자체 가드를 갖고 있었지만 **`chatpip` 하나만** 봤다.
```js
const isPip = hash === "#chat-pip" || params.get("chatpip") === "1";
```
채팅 PIP 를 만들 때 넣은 가드인데, 그 뒤 추가된 **미니게임·과일상자·덱 PIP 가 빠졌다.**

## 영향 범위 (TTS 만이 아니었다)
- `useTTSEngine` — 두 번 읽힘 (신고된 증상)
- `useSoundPlayer` — 효과음 두 번
- `useTimerEngine` — 타이머 이벤트 두 번
- `useDbMeter` — 가드가 **아예 없어** PIP 에서도 마이크를 잡고 있었다

## 고침
`lib/isPipWindow.ts` 한 곳에 PIP 목록을 모으고 훅 넷이 같이 쓴다.
훅은 그대로 호출하되(규칙 유지) 내부에서 스스로 빠진다.
새 PIP 를 만들면 그 목록에만 추가하면 된다 — 훅 네 곳을 고치는 구조라 또 어긋났다.

## 검증
판별 13건 전부 통과: 본창(기본·해시 페이지)은 실행, PIP 4종은 해시·쿼리 양쪽 형태로 차단,
`?chatpip=0`·`#chat` 같은 유사 문자열은 본창으로 판정.

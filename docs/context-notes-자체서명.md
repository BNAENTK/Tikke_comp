# 자체 서명(self-signing) — 동작함

외부 서명 서비스 없이 앱이 직접 연결 정보를 만든다. 실측으로 채팅·선물·좋아요 수신까지 확인.

## 구조

`apps/desktop/electron/services/sign-provider.ts`

1. 히든 BrowserWindow 로 `tiktok.com/@계정/live` 를 띄운다. 세션은 **`persist:tiktok-auth`**
   (앱이 리그·랭킹에 이미 쓰는 로그인 파티션).
2. 그 페이지 안에서 `webcast/im/fetch` 를 **서명 없이** 한 번 쏜다.
   TikTok 의 webmssdk 가 `window.fetch` 를 감싸 두어 나가는 순간 서명이 붙는다.
3. 나간 요청 URL 을 CDP(`Network.requestWillBeSent`)로 잡는다. 마커 파라미터로 우리 요청만 고른다.
4. 그 URL 의 파라미터를 커넥터의 `signedWebSocketProvider` 결과(wsParams)로 넘긴다.
   cursor 는 `"0"`, internalExt 는 합성값. WebSocket 이 붙으면 서버가 첫 커서를 내려 준다.

커넥터의 방 번호 조회는 외부 서명 경로를 타므로, `resolveRoomId()` 로 미리 뽑아
`connect(roomId)` 로 넘긴다(HTML 조회 → 실패 시 히든 창에서 추출).

## 켜고 끄기

설정 키 `tiktokSelfSign` (기본 **true**). 다음 조건이면 자동으로 기존 경로를 쓴다.
- 틱톡 로그인 세션(`sessionid` 쿠키)이 없을 때 — `signProvider.hasLoginSession()` 로 먼저 확인
- 자체 서명 연결이 실패했을 때 — `tiklive.ts` 가 기존 옵션으로 커넥션을 다시 만들어 재시도

즉 **기존 동작은 안 깨진다.** 자체 서명이 안 되면 예전 경로 그대로다.

## 실측 사실 (다시 파지 말 것)

- 홈페이지에는 서명 SDK 가 없다 → `@계정/live` 페이지여야 한다.
- 히든 창은 `loadURL` 이 resolve 되지 않는다 → `dom-ready` 로 판단.
- 디버거는 **로드 전에** 붙이고 `Network.enable` 을 기다린 뒤 로드해야 요청을 잡는다.
- 히든 창은 `document.hidden === true` 라 TikTok 이 라이브 초기화를 미룬다 → visibility 스푸핑 필요.
- 라이브 페이지는 방송 소리가 그대로 난다 → `setAudioMuted(true)` + video pause.
- **서명을 직접 만들면 안 된다.** `byted_acrawler.frontierSign` 은 X-Bogus 만 만들고,
  그걸 URL 에 미리 붙이면 SDK 재서명과 충돌해 403. 무서명 URL 로 쏘면 SDK 가
  X-Bogus·X-Gnarly·X-Dynosaur·msToken 을 알아서 붙인다.
- **im/fetch 응답 본문은 못 받는다**(브라우저 안은 cross-origin, Node 재요청도 403).
  받을 필요도 없다 — cursor `"0"` 으로 WebSocket 을 열면 정상 동작한다.
- **로그인 세션이 필수.** 로그아웃이면 라이브 중인 방도 offline 셸이 내려오고
  페이지가 im/fetch 를 아예 안 보낸다(같은 세션에서 `room/enter` 는 403,
  `check_alive`·`in_room_banner` 는 200).
- 앱 main process 의 `fetch` 로 tiktok.com HTML 을 긁는 건 막힌다 → 창 추출 폴백이 실제 경로.
- `setRoom()` 이 `roomUser` 를 미리 바꾸므로, 창 교체 판단은 **실제로 떠 있는 계정**
  (`loadedUser`)으로 해야 한다. 안 그러면 방을 바꿔도 이전 방 페이지를 재사용한다.

## 방 번호 조회 (실측으로 두 번 틀렸던 곳)

- 주 경로는 라이브 페이지 HTML 조회. 앱 main 의 fetch 가 간헐적으로 막히니 폴백이 필요하다.
- **HTML 의 roomId 는 SSR 초기값**이라 로그인 세션에서는 `"roomId":""` 로 비어 있다 —
  창 DOM 정규식으로 뽑으려 하면 영원히 못 뽑는다.
- 실제 값은 `SIGI_STATE.LiveRoom.liveRoomUserInfo.**user**.roomId` (liveRoom 아래가 아니다).
- 창 안의 이 상태가 채워지기까지 40초를 넘기기도 한다 → 20초만 보고 기존 경로로 넘긴다.
- 서명 창은 `show:false` 대신 화면 밖(-3200,-3200) + `showInactive()` 로 띄운다.
  완전히 숨긴 창은 Chromium 이 렌더링·스크립트 자원을 늦게 준다.
- 방송이 끝난 방은 커넥터의 room/info 가 status 4 로 걸러 준다(HTML roomId 는 종료 후에도 남는다).

## 함정: 페이지로 주입하는 JS 문자열

셸(`node -e` 안의 문자열)로 이 파일을 패치하면 작은따옴표가 통째로 사라져
`var found = ;` 같은 SyntaxError 코드가 들어간다. executeJavaScript 는 reject 되고
`catch {}` 가 그걸 삼켜서 **조용히 항상 실패**한다(방 번호 조회가 이걸로 20초씩 헛돌았다).

주입 문자열을 고칠 때는 Edit 도구로 직접 고치고, 고친 뒤 문법을 확인한다 —
백틱 블록을 뽑아 `new Function(code)` 로 파싱해 보면 바로 드러난다.

## 검증 기록

- @sinhokim0506 / @lua43794 / @coco.lynee — 연결 성공, 20초 내 chat·gift·like·member 수신.
- 방 전환(@lua43794 → @coco.lynee) 시 창이 새로 뜨고 새 방 서명으로 붙는다.
- 창 재사용 시 연결까지 약 5초, 새 창은 10초 안팎.
- 같은 시각 기존 경로는 `Failed to retrieve Room ID from all sources` 로 실패하기도 했다
  (외부 서명 서비스가 간헐적).

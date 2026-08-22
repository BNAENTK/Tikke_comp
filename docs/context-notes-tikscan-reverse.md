# TIK SCAN 데스크탑 앱 리버스 분석 (v2.5.4)

분석일 2026-08-02. 대상 `C:\Users\zkfma\AppData\Local\Programs\TIK SCAN`.
Electron 앱, `app.asar` 추출 → javascript-obfuscator 난독화 → `webcrack`으로 복원해 분석.

---

## 0. 한 줄 요약

TIK SCAN은 **브라질 시장 전용 TikTok LIVE 툴킷**이다. 데스크탑 앱은 `tikscan.live` 웹앱을 감싼 Electron 껍데기이고, **데스크탑 번들 고유 기능은 4개뿐**이다 — **TikTok LIVE Studio 원격 제어 덱**, **유튜브 신청곡 봇**, **로컬 미디어 오버레이 서버**, **전역 핫키/키스트로크 송출**. TikTok 이벤트 수집 로직은 앱에 **전혀 없다**(전부 서버측).

> **분석 범위 한계 (읽기 전 필독).** 이 문서는 `app.asar` 안에서 확인 가능한 것만 다룬다. 제품 기능의 상당 부분이 `tikscan.live` 서버/웹앱에 있으며 그쪽은 관측되지 않았다. "X가 없다"는 서술은 전부 **데스크탑 번들에 없다**는 뜻이지 제품에 없다는 뜻이 아니다.

---

## 1. 껍데기 구조

`APP_URL = https://tikscan.live` (dev override `TIKSCAN_DEV_URL` / `localhost:4523`).

창 7종. **전부 `contextIsolation:true` + `nodeIntegration:false` + `sandbox:true`** — 보안 플래그는 깨끗하다. 예외 2개.

- `musicView` (BrowserView, `localhost:6969/music.html`) — `webSecurity:false`, `sandbox:false`. 로컬 페이지라 위험도는 낮으나 동일출처정책이 꺼져 있다.
- `mediaWindow` (`tikscan.live/media-editor`) — partition 미지정 → 기본 세션 사용.

DevTools 차단 장치 있음 (`devtools-opened` 즉시 close + F12/Ctrl+Shift+I 계열 preventDefault). 보안이 아니라 분석 방해용.

단일 인스턴스 락 있음. 트레이 없음. 메뉴 3개 (Visualizar / Editar / Stream Deck).

`persist:tikscan` 세션 전 요청에 헤더 강제 주입.
```
X-TIKSCAN-Client: desktop
X-TIKSCAN-Version: <app.getVersion()>
```
인증은 Express 세션 쿠키 `connect.sid`. 쿠키 변경 감지 → 유저 캐시 무효화.

**업데이터** — electron-updater 6.8.3, `autoDownload:false`, 패키징 빌드에서만 준비 4초 뒤 1회 체크(주기 재검사 없음). 피드는 `resources/app-update.yml`.
```yaml
provider: generic
url: https://media-tikscan.cc/installers
channel: latest
```
`setFeedURL` 호출 없음. 배포된 package.json에 `build` 키가 통째로 제거돼 있어 서명 검증 여부는 asar만으로 확인 불가.

**미사용 의존성 3종** — `@ghostery/adblocker-electron`, `@nut-tree-fork/nut-js`(대신 하위 `libnut`을 직접 require), `iceberg-js`(Apache Iceberg REST 카탈로그 클라이언트, 참조 0건). 마지막 건 이 앱에 있을 이유가 없는 의존성이다.

---

## 2. TikTok 이벤트 수집 — 앱에 없다 (가장 중요한 발견)

**메커니즘은 preload WebSocket 몽키패치, 출처는 자기네 웹앱이다.**

`session.fromPartition("persist:tikscan").setPreloads([... , lib/chat-hook-preload.js])`로 세션 전체에 preload를 심는다. 그 preload는 `webFrame.executeJavaScript()`로 메인 월드에 훅을 주입해 `window.WebSocket` 생성자 하나만 교체한다. `send`/`fetch`/XHR은 안 건드리고 `addEventListener("message")`로 관찰만 한다.

```js
const msg = typeof data === "string" ? JSON.parse(data) : null;
if (!msg || !msg.event) return null;
if (msg.event === "chat" || msg.event === "chatMessage") return { kind:"chat", payload: msg.data || msg };
if (msg.event === "gift") return { kind:"gift", payload: msg.data || msg };
```
→ `postMessage` → isolated preload가 `event.source !== window` 검사 후 `ipcRenderer.send("tikscan:chat-event"|"tikscan:gift-event")`.

즉 **tikscan.live 페이지가 자기 백엔드로 여는 WebSocket을 앱이 안에서 엿듣는 구조**다.

근거와 함의.
- 바이너리 프레임(ArrayBuffer/Blob)은 `null`로 버린다 → **protobuf 디코딩 불가**. 상류가 JSON 릴레이라는 결정적 증거.
- 페이로드 스키마 `{event, data}`는 **Tikfinity Event API와 동일 형태**.
- `ALLOWED_HOSTS`에 tiktok.com이 아예 없음. `X-Bogus` / `msToken` / `room_id` / webcast / protobuf 문자열 전 파일 grep 0건. EulerStream·tiktok-live-connector 흔적 없음.
- 처리하는 이벤트는 **chat, gift 두 개뿐**. follow/like/share/member(입장)는 어디서도 안 다룬다.
- 기프트는 콤보 중복 적립 방지로 `if (p.repeatEnd === false) return;` — streak 종료분만 통과.
- 정규화기가 플랫/`user.*` 양쪽, `comment|message|text`, `teamMemberLevel|fanClubLevel|fanLevel`을 모두 흡수 → 릴레이 스키마가 유동적이라는 방증.

**상류가 무엇인지는 공개 약관이 답한다 — EulerStream이다.** 면책 조항에 "TikTok, Euler Stream, provedores de hospedagem"으로 직접 명시돼 있다(§11). 즉 서버가 EulerStream 서명으로 TikTok에 붙고, 그 결과를 웹앱 WebSocket으로 흘리며, 데스크탑은 그걸 엿듣는 3단 구조다. 서명 비용과 쿼터를 서버 한 곳에 모으는 설계다.

**주의할 부작용** — `setPreloads`는 URL이 아니라 세션 단위다. 같은 `persist:tikscan`을 쓰는 `music.youtube.com` 창과 팝업 창에도 WS 훅이 같이 주입된다.

---

## 3. TikTok LIVE Studio 덱 ("TTS Deck")

**용어 주의 — 이 앱에서 "TTS"는 text-to-speech가 아니라 TikTok Studio다.** 접두사 `com.tiktok.livestudio.`, 상태 문자열 "Conectando ao TTS…". **이 앱에 음성합성 기능은 존재하지 않는다.**

Elgato Stream Deck 공식 플러그인 `com.tiktok.livestudio` 프로토콜을 그대로 복제한 클라이언트다(소스 주석이 직접 밝힘 — "Replica o protocolo do plugin oficial Elgato Stream Deck").

**Elgato를 양방향으로 다룬다.** 하나는 위 방향 — TIK SCAN이 **가짜 Elgato 플러그인이 되어 TTLS를 조종**한다(하드웨어 불필요). 다른 하나는 반대 방향 — `stream-deck-bridge.js`가 로컬 HTTP API를 열어 **실물 Stream Deck 플러그인이 TIK SCAN을 조종**하게 한다. 후자의 `.sdPlugin` 매니페스트는 번들에 없으므로 별도 배포한다.

**연결.**
- 포트 **7개 스캔** — `[28189, 39728, 34246, 42205, 38534, 40825, 40622]`. 200ms stagger 병렬 시도, 먼저 connect한 소켓이 이기고 나머지 close. 주석에 "옛 핸드오프의 버그는 포트 하나를 고정한 것"이라고 적혀 있다. (내 `reference_ttls_analysis`는 28189 단일 — **갱신 필요**)
- `io("ws://127.0.0.1:"+port, { transports:["websocket"], protocols:["streamdeck_ttls_v1"], reconnection:false, timeout:1000, forceNew:true })`
- 네임스페이스는 socket.io 기본 `/`. `stream_deck/*`는 네임스페이스가 아니라 **이벤트 이름**이다.
- 인증 토큰 없음. 게이트는 서브프로토콜 + join_room 핸드셰이크뿐(작성자 주장).

**핸드셰이크** — connect → `emit("stream_deck/join_room")` → 서버 동명 응답 → `emit("stream_deck/sync_settings")` + `emit("stream_deck/device_status_emit", JSON.stringify({status:"connected"}))`. `sync_settings` 응답에 scene, scene_list, audio_mute, mic_mute, recording 등 전체 상태.

**액션 발사** — `emit("stream_deck/action_emit", JSON.stringify({action, context, payload}))`, 응답은 `stream_deck/<context>` 채널로 `{code}` (0=ok). 3초 무응답 시 `{code:0, note:"sem ack"}`로 resolve.

**덱 액션 카탈로그 31개 = TTLS 액션 29 + 로컬 전용 2.** 참고로 TTLS 자체 `SDAction` enum은 31종이므로(우리 `reference_ttls_analysis`) TIK SCAN은 `volumedial`, `viewerwishes` 2종을 노출하지 않는 셈이다.

(`com.tiktok.livestudio.<suffix>`)
`scene`(장면전환, 대상 드롭다운이 live scene_list에서 채워짐), `source`(소스 표시/숨김, 장면→소스 캐스케이드), `mic`, `audio`, `recording`, `livestartend`, `livepauseresume`, `highlight`, `soundeffect`, `cameraeffects`(소스→카테고리→이펙트 3단 캐스케이드), `audiofilter`(장치→타입→필터), `equalizer`/`reverberation`/`filter`(고정 페이로드로 audiofilter 재발사하는 슈가), `vibe`, `audiocontrol`, `sayhi`, `cohost`, `treasurebox`, `guessgame`, `playtogether`, `goodybag`, `vote`, `livegoal`, `multiguest`, `team`, `gamepad`, `promote`, `MicControl`, + 로컬 전용 `deck_prev`/`deck_next`.

**덱 UI** — 다중 페이지(장면 탭 추가/이름변경/삭제), 그리드 2~10열×2~10행, 편집모드 iOS 위글 애니메이션 + 드래그 재배치, 버튼별 라벨/10색/47개 icons8 CDN 아이콘/커스텀 ON·OFF 이미지(256px 데이터URL로 리사이즈), 장면별 배경(6종 그라디언트 프리셋 + 단색 + 커스텀 이미지). **라이브 상태 피드백** — Studio settings 푸시로 mic/audio/recording/scene/source 상태를 초록 링(on)/빨강 흐림(off)으로 표시, 클릭 시 낙관적 반영, 사라진 장면엔 "!" 경고 배지.

**발사 경로 2개 (+죽은 것 1개).**
1. 로컬 TTS Deck 창 IPC (`tikscan:studio-action`)
2. **Supabase Realtime 원격 릴레이** — 채널 `deck-<token>`, `action` 이벤트 수신 → `tikstudio.action()` 직결
3. ~~LAN HTTP `0.0.0.0:6970` `POST /action`~~ — **v2.5.4에서 죽은 코드다.** `startDeckMobileServer`가 `main.js:1186`에 정의만 되고 **호출 0건**(직접 검증함). 클라이언트 `renderer/deck-mobile.html`도 함께 고아가 됐다(아직 `fetch("/action?t=…")`를 호출한다). LAN 방식에서 클라우드 릴레이로 갈아탄 흔적.

모바일 URL은 `https://tikscandeck.com/?t=<token>`, QR로 표시(`qrcode-generator`). 실제 서빙 페이지는 `renderer/deck-cloud/deck.html`이고 Supabase 자격증명이 **거기에도 평문으로 인라인**돼 있다.

**릴레이 이벤트.** PC→폰 `config {config,settings,connected}` / `settings {settings,connected}` / `result {rid, code|error}`. 폰→PC `hello`(구독 직후 발사 → config 응답) / `action {uuid, payload, rid}`. `rid` 매칭 5초 타임아웃.

모바일 페이지는 화면 꺼짐 방지(Wake Lock + NoSleep.js 폴백, iOS visibilitychange/touchend/8초 워치독 재무장)를 강제 게이트로 요구한다.

---

## 4. 유튜브 신청곡 봇

**플레이어** — 숨은 BrowserView가 `localhost:6969/music.html`을 로드, 그 안에서 **공식 YouTube IFrame Player API**로 1×1 iframe 재생(사실상 오디오 전용). 자동재생 우회는 mute→play→300ms→stop→unMute 트릭. DOM 자동화/미디어키/합성 키입력 **없음**.

**검색** — YouTube Data API 키 없음. `www.youtube.com/results?search_query=…&sp=EgIQAQ%3D%3D`를 **위조 Chrome 120 UA + `Accept-Language: pt-BR`로 HTML 스크레이프**, `ytInitialData` 정규식 파싱, 최대 25건. 플레이리스트 임포트도 동일 방식(최대 200곡, RD 믹스/라디오는 거부).

**채팅 명령** — 접두사 `.` 하나. `play`, `skip`, `list`(별칭 `queue`/`fila`). 3개가 전부.
- `.play <검색어>` → 권한검사 → 30초 쿨다운 → 검색 → **0번째 결과 자동 선택**(사용자 선택 없음) → 큐 추가
- `.skip` → 모더레이터 무조건 통과, 아니면 크레딧 소모. 실제 스킵은 `seekTo(duration-0.1)`로 종료 유도하는 우회
- `.list` → `POST /api/music/list-show`로 큐 전체를 서버에 밀어넣음(6초 스로틀), `/l/` 오버레이가 표시

**권한** (`userData/music-permissions.json`) — 화이트리스트 → 모더레이터(기본 true) → 지정 선물 1종(이름 정확 일치) 순. **"전체 허용" 티어가 없다.** 구독자/팬클럽 게이트 없음(`fanClubLevel`은 파싱만 하고 안 씀). 포인트/재화 개념 없음.

**선물 크레딧** — `playIncrement = repeatCount × oneSongPerGift`(1~10 clamp), skip 크레딧 최대 3, TTL 10분. 지정 선물명과 다르면 적립 자체가 안 됨. 화이트리스트/모더레이터는 크레딧 미소모.

**큐** — 최대 50곡, **메모리 전용(디스크 영속화 없음, 재시작 시 소실)**. 중복곡 방지 없음, 유저당 점유 곡수 제한 없음, 길이 상한 없음.

**약점 3개.**
- 쿨다운 타임스탬프를 `ytSearch` **호출 전에** 기록 → 검색 실패해도 30초 소모. 차단 시 시청자 피드백도 없음(조용히 return).
- 재생 트리거가 "곡 종료" 시점에만 걸림 → **플레이어가 정지 상태면 신청곡이 쌓이기만 하고 자동 시작 안 됨.** 방송인이 수동으로 한 곡 틀어줘야 체인이 돈다.
- 포트 6969 **고정, 폴백 없음**. EADDRINUSE면 에러 로그만 찍고 음악 기능 전체가 죽는다.

부가 — 좋아요/플레이리스트는 방송인 로컬 localStorage(`tlm_liked`/`tlm_playlists`), 시청자와 무관. 미니 플레이어 320×130 frameless/transparent/alwaysOnTop. `music.youtube.com` 창은 메뉴에서 여는 **장식용 독립 웹뷰**로 신청곡 파이프라인과 무관. `spotify.com`이 ALLOWED_HOSTS에 있으나 참조 0건(죽은 항목).

---

## 5. 오버레이 / 미디어 — TTLS localhost 우회 패턴 (tikke에 직접 유용)

소스 주석이 설계 제약을 직접 밝힌다.

> "o TikTok LIVE Studio não aceita URL `localhost` como fonte. Então a página que o OBS/TTS carrega vem de um domínio real (tikscan.live/media), e é ela quem chama ESTE servidor aqui — que roda na mesma máquina do OBS e é o único que enxerga os vídeos no disco."

**→ TTLS는 localhost URL을 브라우저 소스로 받지 않는다. 그래서 원격 도메인 페이지를 브라우저 소스로 물리고, 그 페이지가 CORS로 로컬 서버를 호출하게 한다.**

사용자에게 주는 URL은 항상 원격이다.

| 오버레이 | URL | 비고 |
|---|---|---|
| Now playing | `tikscan.live/n/<overlayToken>` | 로컬 `localhost:6969/n/<id>`도 동일 HTML 서빙 |
| 곡 변경 토스트 | `tikscan.live/c/<overlayToken>` | 로컬 라우트도 존재 |
| 큐 목록 | `tikscan.live/l/<overlayToken>` | **클라우드 전용, 로컬 라우트 없음** |
| 미디어 레이어 | `tikscan.live/media` | 에디터는 별도 창 `/media-editor` |

`overlayToken`은 `GET /api/auth/me`의 `user.overlayToken`. 미로그인 시 "Faça login no TikScan pra gerar suas URLs únicas".

오버레이 HTML의 SSE URL은 **상대경로 `/events`** 다 — 그래서 같은 파일이 로컬(6969)에서도 클라우드에서도 동작한다. 영리한 설계.

**로컬 HTTP 서버 4개.**

| 서버 | 바인드 | 포트 | 인증 |
|---|---|---|---|
| 음악/오버레이 | `localhost` | 6969 고정(폴백 없음) | **없음** (`/n/`,`/c/` 토큰 정규식이 캡처만 하고 검증 안 함) |
| 미디어 파일 | `127.0.0.1` | 20009~20019 순차 | **없음** (CORS 오리진 허용목록 + 경로 제한만) |
| Stream Deck 브리지 | `127.0.0.1` | 20020~20029 순차 | 명목상 `X-TikScan-Stream-Deck-Key`. **실제 검증 코드가 없다** — `tokenBuffer`는 선언 1회뿐, `timingSafeEqual` 0회, 헤더명은 `Access-Control-Allow-Headers` 문자열에만 등장. 전 `/v1/*`가 사실상 무인증 |
| ~~모바일 덱~~ | ~~`0.0.0.0`~~ | ~~6970~~ | **미기동 (죽은 코드)** |

Stream Deck 브리지 라우트 — `GET /v1/pairing|status|media/sources|layout/layers|sound-alerts`, `POST /v1/media|music|layout/layers/toggle|sound-alerts/trigger`. 본문 32KB 상한. 모든 CORS/페어링 경로가 `request.headers.origin === "null"` 하나로만 게이팅된다(Elgato Property Inspector의 샌드박스 iframe 오리진 — 위조가 자명하다). `GET /v1/pairing`은 페어링 키를 **그냥 내준다**.

미디어 서버 라우트 — `GET /health`(`{tikscanMedia:true, port}`, 원격 페이지가 포트 찾는 용도), `GET /config`, `GET /file/:id`(HTTP Range 지원), `GET /events`(SSE), `POST /source/:id/ended`, OPTIONS. 포트 기준값이 고정인 이유도 주석에 있다 — "OBS 링크는 한 번만 설정하니 포트가 실행마다 춤추면 안 된다".

**미디어는 업로드하지 않는다.** 네이티브 다이얼로그로 로컬 파일 경로만 받아 `userData/media-config.json`에 `{name, path}`와 레이아웃을 저장. 소스 종류 `video|image|web`, 좌표 −500~500%, 크기 1~500%, `fit`, 90도 단위 회전, CSS 색보정(contrast/saturation/brightness/hue), 페이드아웃, `playbackNonce`(재발사 트리거).

**트랜스코딩** — 번들 ffmpeg. **확장자가 아니라 ffprobe 실제 코덱으로 판정.** `CHROMIUM_OK = {h264, vp8, vp9, av1, theora}` 외는 전부 변환. 알파 감지는 `pix_fmt ^(yuva|ya|argb|rgba|abgr|bgra)` + 코덱 `{prores, qtrle, png, apng, gif}`.
```
알파: -vf scale='min(1920,iw)':-2,format=yuva420p -c:v libvpx-vp9
      -pix_fmt yuva420p -auto-alt-ref 0 -crf 30 -b:v 0 ... -f webm
무알파: -vf scale='min(1920,iw)':-2 -c:v libx264 -pix_fmt yuv420p
      -crf 20 -preset veryfast -movflags +faststart ... -f mp4
```
주석의 함정 두 개가 실전 지식이다 — `format=yuva420p`는 **반드시 `-vf` 안에** 넣어야 한다(맨 `-pix_fmt`는 스케일 시 알파 소실), `-auto-alt-ref 0`이 없으면 libvpx temporal alt-ref가 알파를 깨뜨린다. 캐시는 `sha1(path|mtime|size|a|o)` 파일명이라 같은 경로 파일 교체 시 자동 무효화. `.tmp` 쓰고 rename하는 원자적 교체.

**SSE 이벤트는 딱 2종** — `track`(곡 변경 시 + 신규 클라이언트 접속 즉시 1회), `ping`(25초). `Access-Control-Allow-Origin: *` 하드코딩.

**"레이어"와 "사운드 알림"은 오버레이가 아니라 클라우드 프록시다.** 둘 다 `tikscan.live/api/*`로 그대로 넘기는 얇은 어댑터. 레이어는 웹 에디터 캔버스 아이템(5개 화면, 키 `editorItems`~`editorItems_5`), 사운드 알림은 클라우드에서 재생되고 로컬 오디오 장치를 안 건드린다. 세션 ID가 레이어용/사운드용 서로 다르다.

**목표(goal)/팔로워/선물 알림 오버레이는 존재하지 않는다.** `deck-icons/livegoal.png`는 TTLS 액션 아이콘일 뿐이고, `goal` grep 결과 0건.

---

## 6. 데스크탑 자동화

독립 시스템 3개, 서로 다른 라이브러리.

**(a) 키 송출 — 선물→게임 제어용이고, 매핑 테이블은 서버에 있다**

`libnut`을 직접 require(선언된 `nut-js`는 미사용). `libnut.keyTap(key, mods)` 호출이 전부. **타깃 앱 선택 로직이 없다 — 포커스된 창 어디로든 간다.** 입력 문법은 **Tikfinity 호환**: `{NUMPAD0-9}`, `{F1}-{F12}`, `{ENTER}/{ESC}/{TAB}/{SPACE}/...`, 수식어 `Ctrl/Alt/Shift/Cmd/Win`, `+` 조합, 콤마 구분 시퀀스. 반복 1~100회, 50ms 간격.

**호출 체인을 직접 추적한 결과** — `keystrokeSender.sendKey()`의 호출자는 `main.js:341`의 IPC 핸들러 **하나뿐**이고, 그 채널을 부르는 건 `preload.js:47`이 `window.tikscan.sendKey(keys, times)`로 웹앱에 노출한 API다. `ChatStream`도 `MusicController`도 이 경로를 건드리지 않는다.

→ **"선물 X 받으면 키 Y 누르기" 매핑 테이블은 `tikscan.live` 웹앱 안에 있다.** 데스크탑은 실행기일 뿐이다. `keystroke-sender.js` 주석이 "Heaven Climb 같은 키보드 반응 게임"을 직접 언급하므로 용도는 명확하다 — 우리 `reference_stream_to_earn`과 같은 선물→게임제어 카테고리다. 다만 어떤 선물이 어떤 키에 걸리는지는 번들 밖이라 확인 불가.

**(b) 전역 핫키 청취** — `uiohook-napi` keydown 수동 청취(**가로채지 않음** → 다른 앱과 동시 동작). 사운드 알림 콤보 맵을 렌더러가 푸시하거나, 임의 accel/triggerId 등록. macOS는 Accessibility 권한 필요.

**(c) 음악 전역 단축키** — Electron `globalShortcut`(가로채기형), 무조건 등록. `Ctrl+Alt+Space`(재생/정지), `Ctrl+Alt+←/→`(이전/다음), `Ctrl+Alt+↑/↓`(볼륨 ±5).

전원 — `powerSaveBlocker('prevent-app-suspension')` 앱 준비 시 무조건 시작, 음악/미니플레이어에서 추가. 백그라운드 스로틀링 스위치 3종 + `autoplay-policy=no-user-gesture-required`.

---

## 7. Supabase

프로젝트 `https://kcrafqtfrkusghbdhlna.supabase.co`. 키는 **anon publishable**(JWT payload `role:"anon"`, 앞 6자 `eyJhbG`, 208자). service_role 아님. 앱 바이너리와 `deck-cloud/deck.html` 양쪽에 평문 노출.

**테이블 0개, RPC 0개. Realtime broadcast 전용이다.** 채널 `deck-<token>`, `broadcast.self:false`. 수신 `hello`→config 전송, `action`→`tikstudio.action()`. 송신 `result`/`config`/`settings`. 전송 계층은 `undici` WebSocket, 없으면 `ws`.

**토큰 2종.**

| 토큰 | 생성 | 길이 | 저장 |
|---|---|---|---|
| 덱 토큰 (`?t=`) | `crypto.randomBytes(24).base64url` | 32자 | `userData/tts-deck-token.txt`, 영구 고정 |
| Stream Deck 페어링 키 | `crypto.randomBytes(32).base64url` | 43자 | `userData/stream-deck-pairing.json`, mode `0o600` |

anon 키가 설계상 공개이므로 **채널명 `deck-<token>`이 클라우드 경로의 접근 통제 전부**다. 이 영구 토큰은 서드파티 오리진 `tikscandeck.com`에 쿼리스트링으로 전달되고 QR로도 표시된다. 메뉴 "Stream Deck → 페어링 데이터 표시"는 브리지 URL+키를 다이얼로그로 띄우고 클립보드 복사까지 한다.

---

## 8. 유료화

플랜 이름 **"TikScan PRO"** 하나. 엔타이틀먼트 출처는 Supabase가 아니라 `GET https://tikscan.live/api/auth/me` (세션 쿠키). 플래그 — `isAdmin`, `isFullPro`, `hasPaid`, `subscriptionActive`, `remainingMinutes`. 30초 캐시.

**구독 + 선불 분(minutes) 하이브리드 모델**이다.
```js
const hasTime = !!(u && (u.subscriptionActive || (u.remainingMinutes && u.remainingMinutes > 0)));
const allowed = !!(u && !u._notLogged && !u._error && (u.isAdmin || u.isFullPro || hasTime));
```
`hasPaid && !isFullPro` 조합만 "분이 소진되었거나 만료됨 → 갱신" 메시지를 낸다.

**게이팅은 전부 클라이언트 사이드다.** 위 식이 세 곳에 중복되고(메인창 주입 스크립트, TTS Deck 오버레이, 음악창 오버레이) 하는 일은 전부 **DOM 오버레이 display 토글**뿐이다. main 프로세스의 어떤 IPC 핸들러도 이 값을 참조해 거부하지 않는다. 페이월 뒤에서도 큐는 계속 채워지고 서버 POST도 계속 나간다 — **화면만 가리는 구조**이며 DevTools로 오버레이 DOM만 지우면 우회된다.

PRO 잠금 대상은 **TTS Deck**과 **Music** 둘. 오버레이/미니플레이어/모바일 덱은 자체 게이트가 없다.

---

## 9. IPC 표면 (56개)

앱의 전체 기능 면적. 그룹만 요약.

| 그룹 | 수 | 대표 채널 |
|---|---|---|
| 앱 셸/창 제어 | 7 | `tikscan:toggle-music|media|tts-deck|mini-player`, `tikscan:clear-cache` |
| 키스트로크/핫키 | 5 | `tikscan:send-key`, `tikscan:hotkey-register|unregister|unregister-all`, `tikscan:set-hotkey-map` |
| 미니 플레이어 | 7 | `mini:toggle-play|prev|next|set-volume|seek|hide|request-state` |
| 음악 | 14 | `music:get-queue|get-permissions|save-permissions|advance-played|manual-skip|remove-item|now-playing|search-gifts|get-overlay-urls`, `tikscan:yt-search|yt-playlist` |
| TTS Deck / TTLS | 10 | `tts-deck:load|save|hide|pin-top|keep-awake|mobile-url|get-studio-connection|set-studio-connection`, `tikscan:studio-get-status|studio-action` |
| 미디어 | 9 | `media:load-config|save-config|pick-videos|pick-images|get-overlay-url|action|list-sources|open-deck|get-source-base` |
| 계정 | 2 | `tikscan:get-user`, `tikscan:invalidate-user-cache` |
| 채팅/선물 수집 | 2 | `tikscan:chat-event`, `tikscan:gift-event` |
| 디버그 | 1 | `tikscan:debug-log` |

발견한 버그 — `preload.js`의 `ttsDeck.fireKey`가 `invoke("tikscan:send-key")`를 쓰는데 main은 `ipcMain.on`으로만 등록. 키는 나가지만 Promise가 영영 resolve되지 않는다.

---

## 10. 보안 관찰 (사실 기록)

방어적 참고용으로만 기록한다. 이 앱은 우리 자산이 아니므로 실사용 금지.

- **Stream Deck 브리지(`127.0.0.1:20020~20029`)가 페어링 키를 전혀 검증하지 않는다.** 게다가 `GET /v1/pairing`이 키를 내주고, 유일한 게이트는 위조 가능한 `Origin: null` 헤더다. 같은 PC의 어떤 로컬 프로세스든 미디어 발사·음악 제어·사운드 알림·레이어 토글을 임의로 호출할 수 있다.
- **Supabase anon 키가 앱과 `deck-cloud/deck.html` 양쪽에 평문 노출**된다. 키 + 덱 토큰을 아는 쪽은 `deck-<token>` 채널에 가입해 `action` 브로드캐스트를 주입할 수 있고 그대로 TTLS 제어로 라우팅된다. 토큰은 24바이트 랜덤이라 추측은 불가하나, 페어링 다이얼로그/QR/서드파티 도메인 쿼리스트링으로 노출 경로가 여럿이고 로테이션 수단이 없다(영구 고정).
- 미기동이라 현재는 무해하지만, `0.0.0.0:6970` LAN 덱 서버 코드가 번들에 남아 있다. 되살아나면 `?t=` 쿼리 하나로 LAN 전체에 TTLS 제어가 열린다.
- 미디어 서버 `/config`가 **설정된 모든 미디어 파일의 절대 경로를 인증 없이** 반환한다. CORS는 브라우저에서만 강제되므로 로컬의 다른 프로세스는 그대로 읽는다.
- 오버레이 라우트 `/n/<token>`, `/c/<token>`의 정규식이 **토큰을 캡처만 하고 검증하지 않는다** — 아무 문자열이나 통과.
- PRO 게이팅이 DOM 토글뿐이라 우회가 자명하다.

---

## 11. 서버측 제품 실체 (공개 웹 조사)

**데스크탑 번들만 보면 제품을 크게 과소평가하게 된다.** 공개 사이트 기준 실제 규모는 아래와 같다. (출처 `tikscan.live`, `/termos`, `/privacidade` + 크리에이터 게시물. 마케팅 주장이 섞여 있으므로 수치는 미검증으로 취급할 것.)

**TikTok 연결은 EulerStream이다.** 약관의 면책 조항에 "TikTok, Euler Stream, provedores de hospedagem"으로 **직접 명시**돼 있다. §2에서 번들에 서명 코드가 없다고 확인한 것의 답이 이것 — 서버가 EulerStream으로 붙고, 그 결과를 웹앱 WebSocket으로 흘리며, 데스크탑은 그걸 엿듣는다.

**오버레이 68종 / 11개 카테고리**를 광고한다. 요약하면.
1. 랭킹·목표 17종 — Top Likes/Gifter/Viewers/Interactions, 최고선물, 승패카운터, 좋아요 목표(바/육각/온도계), 코인 목표(RPG풍 퀘스트 포함), 팔로워 목표, 주간 코인, 선물 갤러리
2. 배틀/아레나 5종 — 1v1, 배틀 알림(아케이드풍 Fighting/Winner/Defeat), 아레나, 미션, MVP
3. 시각 인게이지먼트 4종 — 하트 상승(누른 사람 사진 표시), 환영, 오늘의 MVP, NFT MVP 카드
4. 알림 7종 — 팔로우, 슬림, 대형선물 천둥 알림, HeartMe, **Fortnite/Roblox/Minecraft 테마 알림**
5. 이펙트 4종 — 스나이퍼, 인피니티 건틀릿, x2/x3 콤보, 불꽃놀이
6. 게임·참여 7종 — HeartMe 룰렛(슬롯머신), **Word Bomb(멀티플레이, 9개 언어)**, 빙고, 디펜더, 보물상자, 상자, 몸으로말해요
7. 멤버/팬클럽 3종 — 레벨업(`!levelup nick lvl`), 멤버 목록, 연속 레벨업 배너
8. 게이머팩 6종 — 게이머 HUD, x1, 코인배틀(A/B 팀), LikeWar, 콤보, Roblox 랭킹
9. 오디오·음악 6종 — 사운드 알림, 사운드 오버레이, **Voice Bot(진짜 TTS, 채팅 `!tts`)**, **Spotify** 재생중/신청 큐/팝업
10. 누적기·유틸 6종 — 항아리, 채팅 오버레이, MSN풍 채팅(넛지 소리), 공유 카운터, 무한 타이머(선물로 시간 연장), **Event Action(선물 X → 사운드/이미지/알림/TTS/웹훅)**
11. 앱·연동 3종 — TIKSCAN DECK(모바일 5×5 리모컨), 데스크탑 앱, **OBS WebSocket 직접 연동**

**중요한 정정 3건.**
- **진짜 TTS는 존재한다** — 서버측 "Voice Bot". 데스크탑의 "TTS Deck"이 TikTok Studio라는 건 여전히 맞지만, 제품에 음성합성이 없다는 뜻은 아니다.
- **신청곡의 주력은 Spotify**다. 데스크탑의 유튜브 봇은 보조 경로로 보인다.
- **follow/like/share/멤버 레벨업 전부 서버측에서 처리된다.** 번들에 chat/gift만 있는 건 데스크탑이 그 둘만 필요로 하기 때문이다.

**시장/포지셔닝** — 브라질 제품. 결제는 Payt(브라질 게이트웨이) + PayPal, 지원 `suporte@tikscan.live`. 크리에이터들이 **"Tikfinity의 브라질 대안"**으로 소개하며 근거로 포르투갈어 지원과 **R$ 고정가**(Tikfinity는 USD)를 든다. 자칭 트래픽 "24시간 활성 스트리머 2,847명"(광고 문구, 미검증). 쇼케이스에 수백만 팔로워 계정 다수.

**가격은 확인 불가** — 로그인 뒤에 있다. 약관에 "gating de acesso PRO, validação de licença" 언급만 있다. (검색에 나오는 $16.99는 *TikControl*이라는 다른 툴이다. 혼동 금지.)

`tikscandeck.com`은 조사 시점에 TLS 오류로 접근 실패. 확인 못 함.

---

## 12. tikke 관점 시사점

**우리가 앞서는 것 (데스크탑 아키텍처 한정).**
- **로컬 자립성** — 저쪽 데스크탑은 TikTok 연결을 자기 서버에 100% 의존한다. 서버가 죽으면 앱이 통째로 무력화된다. tikke는 앱이 직접 붙는다.
- 이벤트 커버리지도 서버 의존의 결과다. 데스크탑 단독으로는 chat/gift 2종만 처리 가능.
- 유료 게이팅이 DOM 토글뿐이라 우회가 자명하다.

**경계할 것.**
- 제품 전체로는 **오버레이 68종 대 우리 쪽**이다. 특히 게임화(Word Bomb 9개국어, 빙고, 룰렛, 배틀 아레나)와 게임 테마 알림(Fortnite/Roblox/Minecraft)은 우리에게 없는 축이다.
- Event Action의 **웹훅 아웃풋**은 우리 `gtaGiftWebhooks`와 같은 카테고리 — 저쪽은 이걸 오버레이 카탈로그의 한 항목으로 팔고 있다.
- 모바일 리모컨을 Supabase Realtime broadcast만으로 구현했다. 자체 서버 없이 저비용으로 리버스 채널을 만든 사례 — 우리 `project_tikke_mobile` 설계 시 비교 대상.

**우리가 배울 것.**
1. **TTLS localhost 브라우저 소스 거부 우회 패턴** — 원격 도메인 페이지 + 로컬 서버 CORS 호출 + `/health` 포트 스캔. ⚠️ **"TTLS가 localhost URL을 거부한다"는 저쪽 소스 주석의 주장이지 우리가 검증한 사실이 아니다.** 쓰기 전에 직접 확인할 것. tikke는 이미 Worker 클라우드 URL로 오버레이를 서빙하므로 이 제약을 이미 만족할 수도 있다. 가치는 "로컬 디스크 파일을 클라우드 오버레이에 물리는 법"이라는 별개 문제에 있다.
2. **TTLS Stream Deck 포트가 28189 단일이 아니다** — 7개 스캔 필요. `reference_ttls_analysis` 갱신 대상.
3. **알파 채널 트랜스코딩 함정** — `format=yuva420p`는 `-vf` 안에, `-auto-alt-ref 0` 필수. 확장자 아닌 ffprobe 코덱으로 판정.
4. **오버레이 HTML의 SSE를 상대경로 `/events`로** 두면 로컬/클라우드 양쪽에서 같은 파일이 동작한다.
5. **모바일 덱의 화면 꺼짐 방지** — Wake Lock API + NoSleep.js 폴백 + iOS visibilitychange/touchend/8초 워치독 재무장. LAN HTTP는 Wake Lock을 못 쓰므로 폴백이 필수다.
6. 미디어 포트 기준값 고정(20009+순차) — 브라우저 소스 URL을 한 번만 설정하게 하려는 의도적 설계.

**직접 경쟁 여부** — 데스크탑은 UI/주석/로그/`Accept-Language` 전부 브라질 포르투갈어이고 i18n 프레임워크도 문자열 테이블도 없는 단일 언어 앱이다. 서버측도 pt-BR 전용(예외는 Word Bomb 게임의 9개 언어뿐). **현재 한국 시장과 직접 경쟁하지 않는다.** 다만 이들은 스스로를 "Tikfinity의 지역 대안"으로 포지셔닝했고 그 전략은 어느 시장에서든 복제 가능하다.

---

## 부록: 재현 방법

```bash
npx @electron/asar extract "…/TIK SCAN/resources/app.asar" ./tikscan
npx webcrack tikscan/main.js -o out    # javascript-obfuscator 해제
# HTML 인라인 <script>는 별도 추출 후 webcrack
```
난독화는 javascript-obfuscator 표준(string array + rotate + 헥스 심볼 + 데드코드). main.js 469KB → 복원 후 77KB.

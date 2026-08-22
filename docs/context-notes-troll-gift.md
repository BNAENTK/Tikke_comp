# context-notes — troll gift (가짜 선물 연출)

## 2026-08-02

### 요구사항
방송인이 실제로 오지 않은 선물의 **TikTok 원본 애니메이션**을 임의로 재생해 시청자를 낚는 장난 기능. 사용자 확정 사항 2가지.

- 연출 범위 = **애니메이션 + 선물 알림 배너까지**. top-donor 랭킹 반영은 **제외**(실제 후원 집계 오염 방지).
- **"그냥 똑같이 만들어, 디자인이나 다른 거 변경하는 거 없이"** — 새 오버레이/새 디자인 금지, 기존 것 재사용.

### 출처 — TIK SCAN 웹앱에 실제로 있다 (초기 판단 정정)
처음에 "TIK SCAN에 troll gift 기능 없음"이라고 보고했으나 **틀렸다.** TIK SCAN 앱 사이드바에 `Troll Gift` 메뉴가 실재한다(2026-08-02 스크린샷 확인).

오판 원인 — `app.asar` 번들만 grep했고 거기서 `troll` 히트가 전부 `con**troll**er` 부분매치였다. 하지만 TIK SCAN 데스크탑은 `tikscan.live` 웹앱을 감싼 껍데기이고 **기능 대부분이 서버측에 있다**. 이 한계를 `context-notes-tikscan-reverse.md` 서두에 스스로 적어두고도 그대로 걸렸다.

> **교훈: TIK SCAN 기능 유무를 번들 grep만으로 판단하지 말 것.** 앱 UI를 직접 보거나 tikscan.live 라우터 테이블(`/overlay-*` 약 110개)을 확인해야 한다. 실제로 라우터에 `overlay-troll-gift`가 존재한다.

다만 **구현 방식은 참고하지 않았다**(서버측이라 소스 접근 불가). 아래 설계는 tikke 자체 인프라만으로 독립 구성한 것이다.

### 재사용할 기존 인프라 — 이미 90% 완성돼 있었다
`GiftFx.tsx:103-117`의 `play()`가 이미 가짜 `gift` 이벤트를 만들어 broadcast하고 있었다. 원래 의도는 "오버레이 테스트"였지만 동작 자체가 곧 가짜 선물 연출이다.

```
GiftFx.tsx play() → cloudOverlay.broadcast({type:"gift", ...})
  → cloud-overlay.ts:145 _enqueue(100ms 배치) → POST /overlay/rooms/{room}/broadcast
  → OverlayRoom DO → 룸 내 전 WS 클라이언트
  → gift-fx.html:139 enqueue() → VAP 풀스크린 재생
  → gift.html:49 handleEvent() → 알림 배너 카드
```

애니메이션 소스는 `userData/gift-asset-cache.json`. `tiklive.ts:414-421`이 라이브에서 99💎 이상 선물을 받을 때 **URL만** 기록한다(바이트 저장 없음). 재생 시점에 `gift-fx.html`이 zip을 받아 JSZip으로 풀고 VAP(좌 RGB / 우 알파)를 캔버스 합성한다.

**실제 경로는 `AppData\Roaming\@tikke\desktop\gift-asset-cache.json`이고 2026-08-02 기준 88개(85개 재생 가능)가 쌓여 있다.** TikTok Universe 44999💎, King Leonardo 42999💎, Lion 29999💎, Dragon Flame, Phoenix 등 최상위 선물이 전부 포함돼 troll gift 용도로 충분하다.

> 조사 함정 기록. 처음에 `AppData\Roaming\Electron\`(= dev 실행 시 userData, 2개뿐)을 보고 "캐시가 2개뿐"이라 잘못 보고했다. 설치 앱의 userData는 `@tikke\desktop`이며 `find -maxdepth 2`로는 잡히지 않는다(scope가 depth 3). userData 조사 시 반드시 `@tikke/desktop`을 먼저 확인할 것.

### 왜 gift-videos(사용자 업로드)가 아니라 gift-fx인가
`overlay-rules-service.ts` + `userData/gift-videos/` + `video.html` 경로도 후보였으나 기각했다. 가짜 선물이 **진짜처럼 보여야** 하는 기능이라 TikTok 원본 애니메이션이 필수다. 사용자가 직접 올린 mp4로는 목적을 달성할 수 없다.

### 안전성 검증 — 실제 후원 데이터는 오염되지 않는다
`cloudOverlay.broadcast`가 안전한 이유는 **구조적**이다. 모든 집계 싱크는 EventBus 구독자인데, EventBus의 발행자는 앱 전체에서 `index.ts:137` 단 하나이고 그 유일한 공급원은 `index.ts:650`의 실제 TikTok 커넥터 이벤트다. broadcast 경로는 `eventQueue`/`eventBus`에 진입하지 않는다.

| 싱크 | 구독 지점 | 가짜 선물 영향 |
|---|---|---|
| sql.js 이벤트 로그 | `index.ts:133` | 없음 |
| top-donor 상태 | `top-donor-service.ts:109` | 없음 |
| Supabase `session_gifts` | `tiklive.ts:479-484` | 없음 |
| Supabase `viewer_crm` | `index.ts:349-361` | 없음 |
| TOP GIFT 트래커 | `index.ts:181-198` | 없음 |

이중 안전장치도 있다 — 설령 도달해도 `top-donor-service.ts:191`은 `uniqueId`가 없으면 거부하고, CRM/세션 트래커도 `uniqueId` 없으면 early return한다. 가짜 페이로드에는 `nickname`만 있다.

`ipc.ts:1565-1566`이 `cloudOverlayService.broadcastMessage`만 호출하는 것도 확인했다. 같은 파일의 `overlay:send`(`:1031-1032`)가 로컬+클라우드 **양쪽**에 보내는 것과 대비된다.

### ⚠️ 부작용 — 실제 데이터는 아니지만 오버레이 로컬 상태는 누적된다
아래 오버레이들은 gift 이벤트를 받아 **브라우저 메모리에 자체 누적**한다. DB에는 안 남지만 방송 중 화면에는 남는다(리로드/리셋 전까지).

- `goal-bar.html:272` — 목표 진행률이 부풀려짐
- 코인자 7종(`coinjar*.html`) — 항아리가 채워짐
- `live-pet.html:622` — 펫 스탯(기분/허기) 왜곡
- `anchor-badge.html:428` — 배지 진행도 누적
- **`gift-raffle.html:154` — 가짜 닉네임이 추첨 풀에 등록되어 당첨될 수 있다** (가장 주의)

일회성 연출이라 무해한 것들: `gift-tornado`, `fireworks`, `emoji-rain`, `light-stick`, `cam-frame`, `big-spender`(임계값 이상일 때만), `fan-alert`.

→ 이 사실은 UI에 경고로 노출한다. 숨기면 방송 사고가 된다.

### 로컬 오버레이는 반응하지 않는다 (제약)
`cloudOverlay.broadcast`는 클라우드 전용이라 `http://localhost:18182/overlay/...`로 연 오버레이는 가짜 선물을 **받지 못한다**. `api.tikke.kr/overlay/...?room=` URL로 연 것만 반응한다. 기존 GiftFx 페이지가 이미 클라우드 URL을 안내하고 있어 동작상 문제는 없지만, 로컬로 테스트하면 "안 된다"고 오해할 수 있어 문서에 남긴다.

### 변경 범위 결정 — 파일 1개
`gift.html`이 이미 `user.nickname` / `giftName` / `repeatCount`만 읽어 배너를 렌더한다(`:49-55`). `gift-fx.html`도 이미 gift 이벤트로 재생한다. **오버레이는 0개 변경**한다. 사용자의 "디자인 변경 없이" 지시와 일치한다.

`GiftFx.tsx`만 고친다. 기존 `play()`의 한계 2가지가 낚시 용도에 부적합하기 때문이다.

1. `user: { nickname: "테스터" }` 하드코딩 → 낚시가 성립하지 않는다. 임의 닉네임 입력이 필요.
2. `repeatCount: 1` 고정 → 콤보 연출 불가. `gift.html`은 `count > 1`일 때만 `×N`을 렌더하므로 수량 지정이 곧 연출 폭이다.

### 왜 `diamondCount`는 손대지 않는가
`gift.html`은 `diamondCount`를 아예 읽지 않는다(아이콘도 하드코딩 🎁). `gift-fx.html`은 `diamondCount × repeatCount ≥ threshold(기본 99)` 게이트에만 쓴다. 캐시 항목은 전부 99💎 이상이라 이미 통과한다. 따라서 캐시된 원본 값을 그대로 쓴다 — 조작할 이유가 없다.

단 `repeatCount`를 키우면 `diamondCount × repeatCount`가 커져 `big-spender.html:136` 임계값을 넘길 수 있다. 의도된 연출 확대이므로 그대로 둔다.

### UI 배치 결정 (1차 — 폐기)
처음엔 기존 GiftFx 페이지 그리드 위에 입력 줄 하나만 추가했다. "디자인 변경 없이"를 "기존 페이지 최소 수정"으로 해석한 것.

### UI 재작업 (2차 — 확정, 2026-08-02)
K님 재지시로 방향 수정. "그대로 가져오라"는 뜻은 **TIK SCAN Troll Gift 페이지의 UI 자체를 복제**하라는 것이었다(선물 에셋 제외, 기능·UI만). 스크린샷 기준으로 재구현했다.

- **별도 페이지 신규** — `apps/desktop/src/pages/TrollGift.tsx`. 사이드바 "오버레이" 섹션에 "선물 낚시" 메뉴 추가(`Sidebar.tsx`), 라우팅 `case "trollgift"`(`app/App.tsx`). 기존 GiftFx는 개발자 섹션에 그대로 둔다.
- **레이아웃 복제** — 헤더(🎬 + 도움말 버튼) → 설명 → 오버레이 URL 박스 → `선물 / 직접 입력` 탭(TIK SCAN의 Gifts / Custom link) → 선물 카드 세로 리스트.
- **카드 구조** — 좌: 아바타(선물 이미지 또는 커스텀) + 이름 + `enviou um {gift} x1` 부제 / 우: `Edit` + 초록 `PLAY`. Edit 클릭 시 카드 안에서 인라인 확장 — 이름(@user), 문구 + 수량, 아바타 변경(FileReader → data URL). TIK SCAN의 "Play on link" 드롭다운은 tikke에 오버레이 대상이 gift-fx 하나뿐이라 생략(불필요한 UI는 넣지 않음).
- **직접 입력 탭** — 선물 애니 없이 알림 배너만 임의 문구로. `type:"gift"` broadcast에 `giftName`만 커스텀.
- **카드 소스** — `listCachedAssets()`에서 `videoUrl` 있는 것만(재생 가능한 88개 중 85개). 라이브 자산 업데이트도 구독해 실시간 추가.

발사 로직은 1차와 동일하다 — `cloudOverlay.broadcast({type:"gift", ...})`. 안전성(랭킹 무오염)·부작용(목표바/추첨) 분석은 위 그대로 유효하다. 아바타는 `user.profilePictureUrl`에 실어 보낸다.

### Enigma 기본 아바타 (2026-08-02)
TIK SCAN troll gift 카드의 기본 아바타(후드+흰가면+보라연기)는 **EffectBattle "Enigma" 이펙트 프리뷰를 재활용**한 것이다. 공개 CDN에 있다 — `https://cdn.tikscan.uk/effect-previews/enigma.webp`(540×960, 인증 불필요). 이걸 받아 상반신 정사각(256×256)으로 크롭해 `apps/desktop/public/troll-avatars/enigma-avatar.webp`로 저장, 원본도 `enigma.webp`로 같이 둔다. `DEFAULT_AVATAR` 상수로 카드 공통 기본 아바타로 사용. 이름을 안 바꾸면 이 익명 아바타가 알림 프로필로 뜨고, "아바타 변경"으로 교체한다.

함정 기록 — tikfinity API에서 이름으로 "Enigma"를 찾으면 "Icy Enigma"(id 24590)가 나오는데 **후드 캐릭터가 아니라 "Desiree"라는 여성 캐릭터다.** 실제 이미지를 받아 눈으로 확인하고 걸렀다. 선물 이름 매칭만 믿지 말 것.

주의 — TIK SCAN은 선물마다 다른 기본 아바타를 서버(`/api/troll-gift/presets`, 로그인 필요)에서 받는다. tikke는 현재 모든 카드에 Enigma 하나를 공통으로 쓴다. per-gift 아바타가 필요하면 그 API 세션이 있어야 한다.

### 맞춤 링크 (20개 독립 링크) — 2026-08-02
TIK SCAN troll gift의 "Custom link" 탭 복제. **20개 독립 오버레이 링크**를 만들어 서로 다른 씬에 배치하고, 선물 편집 시 "재생 링크" 드롭다운으로 어느 링크에 쏠지 고른다.

tikke는 원래 `room` 하나에 모든 오버레이가 붙는 구조라 **서브채널 개념이 없었다.** 이를 위해 `link=N` 라우팅을 추가했다.

- **`gift-fx.html`(desktop + worker 양쪽)** — URL `?link=N` 파라미터를 읽어 `myLink`로 보관. `enqueue()`에서 `ev.targetLink != null && String(ev.targetLink) !== myLink`면 스킵. **`targetLink` 없는 일반 발사(기존 GiftFx 등)는 링크와 무관하게 전부 재생 — 하위호환.**
- **`TrollGift.tsx`** — "직접 입력" 탭을 "맞춤 링크"로 교체. 링크 1~20 각각 이름 입력 + URL(`?room=key&link=N`) + 복사 버튼. 이름은 settings `trollGiftLinkNames`(JSON 배열)에 저장. 선물 카드 Edit에 "재생 링크" 드롭다운(1~20, 이름 표시) 추가. `fire()`에 `targetLink: String(e.link)` 포함.
- **`settings.ts`** — `trollGiftLinkNames: string` 키 추가(기본 `""`).

왜 20개인가 — TIK SCAN이 20개다. tikke도 `LINK_COUNT = 20`. 하나의 gift-fx 페이지를 20개 URL로 열어 각각 다른 씬의 브라우저 소스로 쓸 수 있다.

> **배포 필요 항목** — `gift-fx.html`은 실제로 `api.tikke.kr`(worker)에서 서빙된다. 링크 라우팅이 실제 오버레이에 반영되려면 worker 배포가 필요하다. **K님 배포 금지 지시로 배포하지 않았다.** 로컬 코드만 준비된 상태. 배포 전까지 클라우드 오버레이의 링크 필터는 동작하지 않는다(단 targetLink 없는 기존 발사는 계속 정상).

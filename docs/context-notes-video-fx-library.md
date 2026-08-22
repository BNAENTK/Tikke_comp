# context-notes — 영상 이펙트 라이브러리 (EffectBattle 대응)

## 2026-08-02

### 배경 — TIK SCAN "EffectBattle"이 실제로 뭔가
경쟁 앱 TIK SCAN의 사이드바 메뉴. **오버레이가 아니라 콘텐츠 팩이다.** 페이지 안내 문구가 그대로 설명한다.

> "Download the videos .webm (transparent background) and add as **MediaSource** in OBS or **Video** on TikTok Studio."

- 투명 배경 `.webm` 영상을 Google Drive에서 내려받아 방송 SW에 **사용자가 직접** 미디어 소스로 추가
- 카테고리 탭 — `Others` / `X2` / `X3` / `Glove` / `Tap`
- 카드 그리드 + 썸네일, 각 카드에 `.WEBM TRANSPARENT · Drive` 표기
- 전 항목 `🔒 PRO — SUBSCRIBE TO DOWNLOAD` 잠금

**선물 트리거 연동이 없다.** 방송인이 수동으로 소스를 켜야 한다. 이 점에서 tikke의 `overlay-rules-service` + `gift-videos`가 기능적으로 앞선다(선물 조건 매칭 → 자동 재생 → 로컬/클라우드 동시 송출).

### 에셋을 복제하지 않는 이유 (결정)
"공개 소스"라는 의견이 있었으나 복제하지 않기로 했다. 근거 2가지.

1. **유료 게이팅** — 모든 카드에 `PRO — SUBSCRIBE TO DOWNLOAD`가 걸려 있다. 구독자 전용 상품이다. 접근 가능 ≠ 재배포 허가.
2. **제3자 IP 포함** — 카드 중 `Popular Stitch`는 디즈니 IP 캐릭터다. TIK SCAN에 배포 권한이 있는지 불명이며, 없다면 tikke에 번들하는 순간 책임은 tikke가 진다. tikke는 배포되는 상용 제품이다.

**단 K님 개인 사용은 전혀 문제없다.** PRO 구독자로서 받은 영상을 본인 방송에 쓰고, 그 파일을 `videoFx.add()`로 tikke에 등록해 선물 트리거로 재생하는 것은 정당하다. 금지되는 건 tikke 설치본에 파일을 **동봉해 배포**하는 경우뿐이다.

→ 그래서 **에셋은 사용자가 채우고, 우리는 그릇만 만든다**는 방향으로 확정.

### 무엇이 부족했나
`videoFx` + `VideoRules.tsx`에 이미 등록/삭제/규칙연결/서빙/재생 풀체인이 있었다. EffectBattle처럼 쓰기에 부족한 건 딱 2가지였다.

1. **한 번에 1개만 등록 가능** — `ipc.ts`의 `videoFx:add`가 `properties: ["openFile"]`이라 파일을 하나씩만 고른다. 수십 개 영상을 넣으려면 그 횟수만큼 다이얼로그를 열어야 한다.
2. **분류가 없다** — `VideoFxEntry`가 `{id, name, file}`뿐이라 목록이 평면적이다. EffectBattle의 X2/X3/Glove/Tap 같은 탭이 불가능하다.

### 설계 결정
- **`multiSelections` 추가** — 한 번에 여러 파일 등록. 200MB 상한은 파일별로 검사하고, 초과분만 건너뛴 뒤 나머지는 등록한다(전부 실패시키지 않는다).
- **반환값 하위호환** — 기존 호출부가 `{entry}`를 읽으므로 `{entry, entries}` 둘 다 반환한다. `entry`는 첫 항목. 이렇게 하면 VideoRules 외의 호출부가 있어도 깨지지 않는다.
- **카테고리는 자유 문자열** — enum으로 고정하지 않는다. EffectBattle의 X2/X3/Glove/Tap은 그쪽 사정이고, K님이 쓸 분류는 다를 수 있다. 빈 값이면 "기타"로 묶는다.
- **저장은 기존 settings 키 재사용** — `giftVideoFiles` JSON에 `category` 필드만 추가. 별도 테이블/마이그레이션 없음. 기존 항목은 `category` 없이 로드돼도 정상 동작한다(옵셔널).
- **새 페이지를 만들지 않는다** — 사용자 지시가 "그대로 가져오기"였지만 tikke엔 이미 `VideoRules`가 있다. 페이지를 새로 파면 영상 등록처가 둘로 갈라져 혼란만 커진다. 기존 페이지에 탭 + 그리드를 얹는다.

### 하지 않은 것
- 썸네일 자동 추출 — ffmpeg 호출이 필요하고 등록 시간이 길어진다. 카드에 파일명만 표시해도 목적(분류/훑어보기)은 달성된다. 필요해지면 그때 추가.
- Drive 연동 — TIK SCAN은 Drive에 호스팅하지만 tikke는 로컬 파일 기반이라 불필요.

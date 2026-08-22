# context-notes — 블록 스트림 오버레이 (TIK SCAN Block Stream 대응)

## 2026-08-02

### 기능
선물이 세로 컨테이너 벽에 물리(matter.js)로 벽돌처럼 쌓이고, 방송인이 지정한 "폭탄 선물"이 오면 벽 상위 %가 무너지는 오버레이. TIK SCAN "Block Stream" 복제.

- **트리거**: 모든 선물 쌓기 / 특정 선물만 쌓기
- **폭탄 5종(% 고정)**: 임팩트 −15 / 파열 −30 / 폭파 −50 / 붕괴 −75 / 전체 폭발 −100. 각 슬롯에 선물 하나 지정. 그 선물이 오면 벽이 그 % 무너짐. **폭탄 선물은 벽에 안 쌓임**(TIK SCAN "não entra no montante" 동일).
- 세로 1080×1920 브라우저 소스. 채우기 % 배지 + 85%↑ "벽 가득 참 임박".

### 레퍼런스 = coinjar
`coinjar.html`의 matter.js 물리 구조를 그대로 차용. 차이점.
- 항아리 9조각 벽 → **세로 직사각 well**(좌/우/바닥 3벽, 상단 열림). `createWalls()`.
- 채우기 지표 없음 → **fillPct()** 추가(최상단 블록 Y vs 컨테이너 높이). 폭탄 % 감소에 이 높이가 필요.
- 폭탄 = `detonate(pct)`: 상위 pct 높이 블록에 위로 임펄스 → 220ms 뒤 제거 + 플래시. coinjar-explode 임펄스 참고.

### 풀체인 (overlay-config 채널이라 최소 변경)
스킬 청사진대로 `overlay-config`를 쓰면 overlay-server / ipc SYNC_TYPES / OverlayRoom **무편집**. 실제 건드린 파일.

1. **`block-stream.html`** (worker + desktop 양쪽 동일 사본) — matter.js 세로 well + 폭탄 + `overlay-config` 핸들러(`target==='block-stream'`). 트리거/폭탄 필터는 `handleEvent`의 gift 분기에서: 폭탄 매칭 우선 → detonate + return, 아니면 트리거 모드 필터 후 enqueue.
2. **`cloud-overlay.ts`** getUrls — `"블록 스트림": .../block-stream?room=` 추가.
3. **`settings.ts`** — `blockStreamConfig: string`(JSON `{active,triggerMode,triggerGiftIds,bombs}`) 키 + 기본값 `""`.
4. **`EventOverlay.tsx`** — CATEGORY_MAP / RECOMMENDED_SIZE(1080×1920) / ID_TO_LABEL / OVERLAYS 카드 + **renderSettingsPanel `card.id==="block-stream"` 분기**(활성화 토글 / 트리거 라디오+GiftPicker / 폭탄 5슬롯 GiftPicker + 테스트 버튼). fan-alert의 gift 리스트 패턴 재사용.
5. **`main/index.ts`** — 설정 영속 복원 2곳: `onClientConnect`(로컬 오버레이) + setTimeout 시드(클라우드 DO). top-donor 패턴과 동일.

### 설정 영속 복원 (중요)
`overlay-config`는 런타임 캐시(`lastSyncCache`)라 **앱 재시작 시 사라진다.** 그래서 top-donor처럼 저장된 `blockStreamConfig`를 두 경로로 다시 push:
- `onClientConnect` → 로컬 오버레이(18182) 재연결 시
- 부팅 8초 뒤 setTimeout → 클라우드 룸(DO) 시드. TTLS 브라우저 소스가 이걸 받아 복원.
클라우드 DO는 overlay-config를 per-target 영구 저장하므로, 한 번 시드되면 오버레이 새로고침해도 복원된다.

### 검증
- 브라우저 실측(playwright, 540×960): 세로 시안 프레임 중앙 정렬, 선물 이미지 바닥부터 물리 스택, 채우기 배지 33% 확인. 폭탄 `pct:0.5` → 33%→14% 벽 절반 붕괴 확인. postMessage 핸들러 정상.
- `pnpm --filter @tikke/desktop typecheck` / `@tikke/worker typecheck` 통과.

### metal 프레임 이미지 복제 (2026-08-02)
심플 네온 테두리 → **TIK SCAN 실제 metal 프레임 복제**로 교체.

- 소스: `https://tikscan.live/frames/block-stream-frame.png`(공개, 로그인 불필요, OBS 브라우저 소스용. 1536×2688 RGBA). 네트워크 요청에서 발견 + 번들 분석 에이전트가 URL 확인(`<img src="/frames/block-stream-frame.png">`, 게이팅 없음).
- 저장: `public/overlays/assets/block-stream-frame.webp`(+worker 사본). png 2.9MB → **webp 231KB**(quality 84, alphaQuality 100, 투명 유지).
- 안쪽 투명 영역 측정: 좌우 각 ~11.5%, 상 ~7.5%, 하 ~3.5%. `INNER` 상수로 물리 well을 프레임 안쪽에 정렬(`fitFrame()`+`layout()`). 프레임 img는 z:5로 선물 위에 덮여 테두리 밖으로 안 삐져나옴.
- 원본 좌우 세로 "TIKSCAN" 각인 → **"TIKKE"로 교체**(K 지시). 이미지 각인 재편집 대신 CSS 오버레이: `.brand .cover`(레일색 세로 그라디언트로 원본 TIKSCAN 가림) + `.brand .txt`(writing-mode vertical, emboss text-shadow). `BRAND` 상수(w/top/h/sideInset)로 위치, playwright 실측 조정.
- 보너스 발견: `/frames/block-stream-explosion.gif`(폭발 효과, 4.5MB). 안 씀 — 폭탄은 CSS flash로 충분, GIF 과대. 필요 시 추가 가능.
- 프레임 에셋 서빙: overlay-server가 확장자 있는 경로를 `overlaysDir/<safePath>`로 서빙(`.webp` mime 등록됨) → 절대경로 `/overlay/assets/block-stream-frame.webp`가 로컬·클라우드 양쪽 동작.

playwright 실측(540×960, 체커보드 배경): 프레임 정렬 OK, 선물이 프레임 안 바닥에 물리 스택, TIKKE 각인 좌우 깔끔(원본 TIKSCAN 완전 가림 확인), 채우기 배지 정상.

### 미완/주의
- **배포 안 함** (K님 지시). 실제 클라우드 오버레이(api.tikke.kr)에 반영되려면 worker 배포 필요. 로컬 코드만 준비.
- 폭탄 % 감소는 "블록 개수"가 아니라 "높이" 기준. 블록 크기가 제각각이라 시각적으로 정확.
- TIKKE 커버 박스가 레일보다 약간 밝아 미세하게 도드라짐(허용 범위). 더 맞추려면 `.brand .cover` 그라디언트 색 조정.
- 테스트 링크: `http://localhost:18181/overlay/block-stream?test=1`

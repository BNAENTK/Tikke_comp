# checklist — 영상 이펙트 라이브러리 (EffectBattle 대응)

목표: 투명 webm 영상을 카테고리로 묶어 관리하고 선물에 걸어 재생.
설계 배경은 `context-notes-video-fx-library.md`.

## 조사 (완료)
- [x] TIK SCAN "EffectBattle" 실체 확인 → **오버레이 아님. 투명 webm 다운로드 라이브러리(콘텐츠 팩)**
  - 카테고리 탭 `Others / X2 / X3 / Glove / Tap`, 카드 그리드, 전 항목 `PRO — SUBSCRIBE TO DOWNLOAD` 잠금
  - 선물 트리거 연동 없음 — 방송인이 수동으로 미디어 소스 배치
- [x] tikscan.live 라우터에 `effectbattle` 슬러그 없음 확인 (앱 전용 메뉴)
- [x] tikke 기존 인프라 확인 → `videoFx` + `overlay-rules-service` + `video.html` 풀체인 이미 존재
- [x] 에셋 복제 가부 판단 → **복제 안 함.** 유료 게이팅 + 제3자 IP(디즈니 Stitch) 포함. K님 개인 사용은 무관하게 정당

## 구현
- [x] `ipc.ts` — `videoFx:add`에 `multiSelections` 추가 (여러 영상 한 번에 등록)
- [x] `ipc.ts` — 200MB 초과분만 건너뛰고 나머지는 등록, `skipped` 목록 반환
- [x] `ipc.ts` — `videoFx:setCategory` 핸들러 추가 (카테고리 30자 제한)
- [x] `ipc.ts` — `readGiftVideos` 타입에 `category?` 추가
- [x] `preload/index.ts` — `setCategory` 노출, `add` 반환 타입에 `entries`/`skipped` 추가
- [x] `VideoRules.tsx` — 카테고리 탭(전체 + 카테고리별 개수) 추가
- [x] `VideoRules.tsx` — 목록 → 카드 그리드 전환, 카드마다 분류 버튼
- [x] `VideoRules.tsx` — 업로드 시 현재 열어둔 카테고리로 자동 분류

## 검증
- [x] `pnpm --filter @tikke/desktop typecheck` — 통과 (에러 0)
- [x] 하위호환 — `add`가 `{entry}`를 계속 반환하므로 기존 호출부 무영향. `category` 없는 기존 영상은 "기타"로 묶임
- [ ] 실사용 확인 (K님) — 영상 여러 개 등록 → 카테고리 분류 → 테스트 재생

## 하지 않은 것 (의도적)
- TIK SCAN 영상 파일 복제 — 유료 구독 콘텐츠 + 제3자 IP 포함이라 tikke 배포본에 넣지 않음
- 썸네일 자동 추출 — ffmpeg 호출 비용 대비 이득 적음. 필요해지면 추가
- 새 페이지 신설 — 기존 `VideoRules`에 얹음(영상 등록처가 갈라지면 혼란)

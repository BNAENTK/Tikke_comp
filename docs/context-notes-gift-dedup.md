# 선물 중복 표시 정리 + 이벤트 정규화 + 한글 이름 — 결정 사항

## 배경
TikTok 카탈로그는 같은 선물에 여러 giftId 를 발급한다(예: 나란히 17661/17667, 풍선 인형 17578/17167 —
이름·다이아·이미지 동일, id만 다름). 1889개 중 25개가 이런 완전 동일 중복. 선물 선택 UI 마다
두세 개씩 떠서 혼란. 또 fan-alert(판추배너)이 이벤트의 영어 giftName 을 그대로 표시해 한글이 아님.

## 핵심 불변식 — 대표 id = 그룹 최소 giftId
"이름+다이아+이미지" 동일 그룹의 **최소 giftId** 를 대표로 삼는다. 렌더러(tiktokGifts.ts 기반
giftDedup.ts)와 electron(gifts.json 기반 tiklive)이 **같은 규칙**을 써야 매칭이 일치한다.
- 검증: 두 카탈로그 모두 1889개, canonical 교차검증 1889개 전부 불일치 0. → 항상 일치.
- 최소 id 는 소스 정렬과 무관하게 결정적이라 두 소스가 같은 id 세트만 가지면 자동 일치.
- 이 불변식이 깨지면(두 카탈로그가 달라지면) 선택한 선물이 트리거 안 될 수 있음.

## TASK A — 중복 1개로 표시 + 매칭 정규화
1. `src/data/giftDedup.ts`(신규): `CANONICAL_GIFT_ID`(dup→대표), `TIKTOK_GIFTS_UNIQUE`(대표만).
2. 모든 선물 선택 UI 의 **표시 목록**을 UNIQUE 로 교체:
   - GiftPicker.tsx, GiftPickerModal.tsx, GiftBrowser.tsx(브라우저 목록+로컬픽커),
     SoundLibrary.tsx, EventOverlay.tsx(big-spender 픽커).
   - **id→이름/이미지 조회(.find, NAME_BY_ID)는 원본 TIKTOK_GIFTS 유지** — 숨긴 중복 id 도 해석돼야 함.
3. 이벤트 정규화: tiklive gift 핸들러에서 `data.giftId = canonicalGiftId(giftId)`.
   숨긴 중복 id 로 온 선물도 사용자가 고른 대표 id 와 매칭됨(= 원래 조용히 누락되던 실버그 수정).

## TASK B — 영어 이름 → 한글 (fan-alert 등)
tiklive gift 핸들러에서 `data.giftName = staticGiftName(canonId)` (카탈로그 한글 n). 틱피니티
릴레이(위쪽) **뒤**, normalizeEvent **전**에 적용 → 틱피니티는 영어 원본 유지, 앱 소비(오버레이·CRM·TTS)는 한글.
- **부작용(의도적)**: TTS 선물 읽기(useTTSEngine.ts:194 giftName 사용)도 영어→한글로 바뀐다.
  한국 방송인에겐 개선. 영어 유지 원하면 이 줄만 되돌리면 됨.

## 배치 이유
diamondCount 보정은 릴레이 전(틱피니티도 보정된 다이아 받게). giftId 정규화·한글 이름은 릴레이 후
(틱피니티 호환 매칭이 영어/원본 id 에 의존할 수 있어 원본 유지). 관련 [[reference_tikke_tikfinity_compat]].

# 선물 브라우저 수집 보강 (2026-08-12)

발단: "선물 브라우저 수집은 잘 되고 있어?"

## 진단

수집원은 멀쩡했다. `https://tikfinity.zerody.one/api/getAllGifts?lang=ko` 실측:
HTTP 200, 1.1초, **902개**, 앱 파싱 통과 902/902, 이미지 902, 다이아>0 902, 한글 이름 875.

문제는 앱 쪽 구조 셋이었다.

1. **폴링이 선물 브라우저 화면 안에서만 시작**됐다(`GiftBrowser.tsx` useEffect).
   `main/index.ts` 에는 부팅 시작이 없어, 그 화면을 한 번도 안 열면 수집이 아예 안 돌았다.
2. **결과가 메모리에만** 있었다(`tikfinityGiftStore` — persist 없음). 앱을 끄면 `gifts: []`.
3. 그동안 소비자(선물 이펙트·사운드 룰·탑기프트)는 **2026-05 정적 목록**으로 폴백했고,
   그 뒤 나온 선물은 이름·이미지 없이 다뤄졌다.

## ⚠ 정적 선물 목록을 건드릴 때의 제약 (가장 먼저 깨질 것)

**로컬 목록은 API 의 상위집합이 아니다.** 로컬 1,889개 중 **1,432개가 API 에 없다** —
구형·지역 선물이고 `tiklive.ts` 가 정적 폴백으로 쓴다. **덮어쓰면 그만큼 사라진다.**

그리고 `giftDedup.ts` 의 대표 giftId 는 **런타임 계산**이다:
그룹 키 = `이름|다이아|이미지`, 대표 = 그 그룹의 **최소 id**.
`CANONICAL_GIFT_ID` 는 손으로 관리하는 표가 아니라 `TIKTOK_GIFTS` 에서 매번 만들어진다.

따라서 두 가지가 금지다.
- **기존 항목의 이름·다이아·이미지 수정 금지** — 그룹 키가 바뀌어 대표가 흔들리고,
  사용자가 저장해 둔 사운드 룰(대표 giftId 로 저장)이 안 맞게 된다.
- **기존 그룹의 대표를 빼앗는 신규 추가 금지** — 신규 id 가 그 그룹 최소값보다 작으면
  대표가 바뀐다. 실측 35건이 여기 해당했다.

## 한 것

### 1. 정적 목록 갱신 (`scripts/merge-gifts.mjs`)
신규 445개 중 대표를 빼앗는 35개를 빼고 **410개만 추가**. 삭제·수정 없음.
1,889 → 2,299 (`gifts.json` + `src/data/tiktokGifts.ts` 양쪽, id 집합 동일).

검증(`scratchpad/verify-gifts.mjs`): 실제 giftDedup 규칙으로 전후 대조 —
사라진 선물 **0건**, 대표 id 바뀐 선물 **0건**. UI 표시 목록 1,864 → 2,248.

최신 이름·가격은 앱이 켜져 있을 때 라이브 목록이 덮어쓴다. 정적은 폴백 전용.

### 2. 수집 결과 저장
`tikfinity-gift-service.ts` 가 수집 성공 시 db 에 저장(`tikfinityGiftsCache` +
`tikfinityGiftsUpdatedAt`), `getCachedGifts()` 로 복원. IPC `tikke:gifts:cached` + preload 노출.
`GiftBrowser` 는 마운트 시 저장분을 먼저 깐다.

**순서 주의**: 저장분 응답이 첫 수집 결과보다 늦게 올 수 있다. `gotLive` 표식과
"스토어가 비어 있을 때만" 조건으로 신선한 값을 덮지 않게 했다.

검증(`scratchpad/giftcache-test.mjs`): 실제 서비스를 돌려 902개 수집 → 저장 → 복원 확인.

### 3. 부팅 시 수집 시작
`main/index.ts` 에서 30분 주기로 시작. 결과는 모든 창에 `tikke:gifts:updated` 로 보낸다.

**사용자가 끈 것을 덮으면 안 된다.** `autoStartDisabled` 가 메모리에만 있어서 앱을 껐다 켜면
다시 켜졌다 → 설정 키 `tikfinityGiftAutoStart`(기본 true)를 추가하고, 선물 브라우저 토글이
그 값을 저장하도록 했다. main 은 그 값이 `false` 면 부팅 시작을 건너뛰고,
렌더러도 마운트 시 저장값을 읽어 자동 시작을 막는다.

## 안 한 것

`pnpm check:overlays` 가 `top-gift-ani.html` 의 desktop↔worker 불일치를 잡는데
**다른 세션이 그 파일을 수정 중**(working tree)이라 손대지 않았다. 이번 변경과 무관하다.

# 애니메이션 탑기프트 — 텍스트 효과·웨이브·드래그 렉 (2026-08-12)

K 신고 4건. 전부 실측으로 원인 확정하고 고쳤다. 대상은 `top-gift-ani.html`(+ worker 사본)과
`TopDonor.tsx` 미리보기다.

## 1. "웨이브를 안 켜면 텍스트 효과가 적용 안 된다" — 범인은 모션이었다

`applyMotion` 이 WAAPI 를 걸기 전에 `el.getAnimations().forEach(a => a.cancel())` 를 했다.
`getAnimations()` 는 **CSS 애니메이션도 반환**한다. 그래서 텍스트 효과의 그라데이션 흐름(`tdFx`)이
모션을 켠 순간 통째로 취소됐다.

웨이브를 켜면 효과가 글자 `span` 으로 내려가는데, 모션은 부모 요소에만 걸리므로 span 의 `tdFx` 는
살아남는다 → **"웨이브를 켜야 효과가 먹는다"**로 보인 것이다. 실측:

| 상태 | `nameEl.getAnimations()` |
|---|---|
| 효과만, 모션 없음 | `tdFx` |
| 효과 + 모션 pulse (수정 전) | `WAAPI` — tdFx 소멸 |
| 효과 + 모션 pulse (수정 후) | `tdFx, WAAPI` |

**다른 top-donor 오버레이 10종은 이미 `waapiCancel` 로 CSS 애니메이션을 보존하고 있었다**
(`if (!('animationName' in a) && !('transitionProperty' in a)) a.cancel()`). anim 만 빠져 있었다.
같은 헬퍼를 넣고 `restartMotion` 의 `inner.getAnimations()` 도 같이 바꿨다.

## 2. "웨이브가 물결이 아니라 전체가 같이 위아래로만 움직인다"

`.wave-letter.fx-on { animation: tdWave …, tdFx … !important; }` — **shorthand + !important**.
shorthand 는 `animation-delay` 를 0 으로 리셋하고, `!important` 라서 글자마다 준 인라인
`animationDelay`(비-important)를 이긴다 → 전 글자 delay 0 = 동시에 오르내림.

`!important` 는 `.fx-on { animation: … !important }` 를 이기려고 붙은 것이라 뺄 수 없다.
그래서 인라인도 important 로 준다: `sp.style.setProperty('animation-delay', …, 'important')`.
실측 결과 `0s / 0.08s / 0.16s`. (top-donor-a/b 는 `.fx-on` 에 !important 가 없어 원래 정상이었다.)

## 3. 색상 팔레트 드래그하면 프레임이 떨어진다

드래그 중엔 settings 가 초당 수십 번 온다. 멀티에선 그때마다 카드를 통째로 새로 만들었다
(클론 + innerHTML 정규식 치환 + 모션 재적용 + fitToSource). 게다가 `applySettingsMessage` 안에서
한 번, 호출처의 `reapplyMode()` 에서 또 한 번 — **같은 작업을 2회**.

- 중복 호출 제거(멀티 재구성은 호출처의 `reapplyMode()` 담당).
- `reapplyMode()` 를 rAF 로 합쳐 한 프레임에 한 번만 반영.

실측(멀티 ON, settings 30회): **157ms → 12ms**.

## 4. 세로 → 순환으로 바꾸면 세로 카드가 안 사라진다 (수정 중 발견)

`startMultiRotation` 의 순환 경로가 `mg-on` 과 `#mgCards` 를 안 치웠다. 단일 카드는 숨겨진 채
세로 카드가 화면에 남아 순환이 안 도는 것처럼 보인다. 순환 진입 시 컨테이너를 비우게 했다.
전환 4방향(세로→순환→세로→OFF) 전부 확인.

## 5. 미리보기에서 닉네임 웨이브가 아예 렌더되지 않았다

`TopDonor.tsx` 네이티브 미리보기(a/b 탭) 두 곳 모두 제목만 글자별 span 을 만들고 닉네임은 평문이었다.
게다가 `WAVE_DUR_PREVIEW` 는 정의만 되고 안 쓰여서 제목 웨이브 속도도 1.4s 고정이었다.
`waveLettersPreview(text, speed)` 헬퍼로 4곳(제목·닉네임 × 미리보기 2종)을 통일했다.

---

# 전수 점검 (2026-08-12) — 테스트 환경 오염과 실제 결함

## 먼저: 테스트가 실행 중인 앱에 오염되고 있었다

오버레이를 로컬 http 서버로 띄워 postMessage 로 설정을 주입해 검사했는데, 오버레이는 뜨자마자
**로컬 앱의 WS(18182)에 자동 접속**한다. 그래서 K 의 실제 설정이 몇백 ms 뒤 내 테스트 값을 덮었고,
그게 "제목을 바꿔도 안 바뀐다", "top-donor-b 텍스트 효과가 안 먹는다", "웨이브 해제 후 span 이 남는다"
처럼 **가짜 버그**로 보였다. 제목 되돌림은 textContent setter 를 가로채 스택을 찍어 잡았다 —
같은 `applySettingsMessage` 경로로 옛 값이 다시 들어왔다.

→ 하네스는 `addInitScript` 로 `window.WebSocket` 을 죽인 뒤 테스트한다. **이 격리 없이 나온 결과는 믿지 마라.**
격리 후 위 3건은 전부 사라졌다(= 결함 아님).

## 실제로 고친 것

**1. top-gift-ani reset 이 두 경로에서 다르게 동작**
- postMessage(미리보기) 경로는 `stopMultiRotation()` 을 안 불러 배치 카드가 그대로 남았다.
- WS(실제 송출) 경로는 멈추긴 했지만 `_lastEntries`/`_lastSingle` 을 안 비웠다.
- 증상 실측: 순환 중 리셋 → 0.2초 뒤 `---` 로 지워졌다가 **1.2초 뒤 "유저2 ×2" 가 되살아남**(타이머 생존).
  세로 중 리셋 → 카드 3장 그대로. 리셋 뒤 멀티 토글 → 옛 명단 부활.
- `handleReset()` 하나로 통일. 선물 이미지는 일부러 그대로 둔다(기존 동작 유지).

**2. top-donor-i / top-donor-j 가 선물 이미지 설정을 통째로 무시**
`applySettings` 에 `giftEnabled`/`giftSize`/`giftOpacity` 처리가 아예 없었다(위치 오프셋만 있었다).
UI 에는 "🎁 선물 이미지 → 표시/투명도/크기"가 노출되는데 아무 반응이 없었다.
`renderDonor` 가 `img.style.display` 를 직접 켜므로 **부모 `.gift-zone` 을 끈다**(top-donor-c 와 같은 방식) —
그래야 새 선물이 와도 다시 켜지지 않는다. 실측으로 확인.

## 고치지 않기로 한 것 (의도적)

- **c~j 에 웨이브가 안 먹는다**: 그 탭들 UI 에 웨이브 항목 자체가 없다(`showWave = !isTopGift` 조건).
  a/b/anim 전용 기능이라 결함이 아니다.
- **"값 표시"(showGiftValue) 해석 차이**: anim 은 카운터를 통째로 숨기고, top-donor-a 는 별칭만 뗀다.
  어느 쪽이 옳은지는 UI 라벨만으로 단정할 수 없고, 지금 동작을 바꾸면 기존 사용자 화면이 바뀐다 → 보류.
- **i/j 의 `getAnimations().forEach(cancel)`**: 대상이 선물 이미지 하나뿐이고 거기엔 CSS 애니메이션이
  없어서 실제 피해가 없다. top-gift-ani 때와 달리 고칠 이유가 없어 두었다.

## 회귀 검증 방법 (다음에도 이렇게)

`scratchpad/overlay-regression.mjs` — 오버레이 11종 × 22시나리오를 돌려 화면 상태를 JSON 으로 남기고
수정 전/후를 비교한다(playwright-core + 시스템 크롬, `channel:'chrome'`).
이번 diff 에서 의미 있는 변화는 `top-donor-i/j 선물끔 → img.shown: true→false` 뿐이었다.
나머지는 폰트 렌더 ±1~3px 과 카운터 애니메이션 진행 숫자(노이즈)다.

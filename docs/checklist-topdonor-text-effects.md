# 최고 후원자 Text Effects 추가 체크리스트

목표 — top-donor 변형 A/B/애니(top-gift)에 Title·Username 텍스트 효과 추가.

대상 변형 — a, b, anim(top-gift). 효과 — Rainbow/Aurora/Neon/Gold/Fire/Gradient(2색). Wave Speed — Slow/Normal/Fast.

## 새 config 필드 (DonorSettings, optional로 추가 → 타 변형 영향 없음)
- [ ] titleEffect: "none"|"rainbow"|"aurora"|"neon"|"gold"|"fire"|"gradient"
- [ ] titleGrad2: string (gradient 2번째 색, titleColor→titleGrad2)
- [ ] titleWaveSpeed: "slow"|"normal"|"fast"
- [ ] usernameEffect: 동일 enum
- [ ] usernameWave: "none"|"wave"
- [ ] usernameGrad2: string
- [ ] usernameWaveSpeed: "slow"|"normal"|"fast"
- [ ] showCoins: boolean (Show Gift Coins). coinsAlias는 기존.
- 기존 유지: titleWave, coinsAlias, *Glow, *Stroke

## [1] 오버레이 HTML (a, b, top-gift-ani) — 핵심 출력
- [ ] 공통 효과 CSS: .fx-rainbow/aurora/neon/gold/fire/gradient (background-clip:text 애니 그라데이션) + @keyframes tdFx
- [ ] --td-grad1/--td-grad2 var (gradient용)
- [ ] wave speed: --td-wave-speed var → .wave-letter animation-duration
- [ ] applySettings: title/username effect class 적용, grad2 var, wave/speed, showCoins 토글
- [ ] renderText: username도 wave 지원 (현재 title만). effect+wave 동시 = 각 letter span에 effect class
- [ ] renderTop: username을 renderText(usernameWave, effect)로 렌더 (textContent 직접대입 제거)
- [ ] showCoins=false면 counter(코인) 숨김
- [ ] 검증: ?test=1 로 각 효과 육안 확인

## [2] TopDonor.tsx — config + 설정 UI
- [ ] DonorSettings 인터페이스에 새 필드(optional)
- [ ] defaultsA/B/Anim에 기본값
- [ ] "Text Effects" 설정 섹션 (a/b/anim 탭에서만 노출): Title Effect 드롭다운, Title Wave 토글+speed, Username Effect 드롭다운, Username Wave 토글+speed, Show Gift Coins 토글, Coins Alias(기존), gradient 선택 시 2색 picker
- [ ] 검증: desktop typecheck

## [3] 인앱 미리보기 (PreviewA/PreviewB/TopGiftPreview)
- [ ] PreviewA/B 제목·유저명에 효과 class 적용 (공용 <style> 주입 또는 인라인)
- [ ] TopGiftPreview는 iframe postMessage라 자동 반영 (확인만)

## [4] SYNC / 전달 확인
- [ ] 설정 변경 시 live 오버레이에 action:settings 전송되는지 (top-donor-service)
- [ ] 재연결 시 settings 복원 경로 확인

## 미결정/메모
- effect+wave 동시: letter span 각각에 gradient (per-letter 흐름). 과하면 wave 시 solid 효과색.
- effect 켜지면 -webkit-text-fill-color:transparent → color 무시, stroke/glow는 유지.

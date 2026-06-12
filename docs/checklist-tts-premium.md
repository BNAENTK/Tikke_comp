# 무료 프리미엄 TTS (Naver CLOVA + Typecast) 체크리스트

목표 — tikke가 비용 부담하는 무료 플랜으로 Naver CLOVA Voice + Typecast TTS 제공. 유저별 일일 10000자 제한.

## [1] Worker 백엔드 ✅
- [x] `lib/tts-quota.ts` — KV 일일 글자수 쿼터 (userId별, KST 날짜, TTL 26h)
- [x] `routes/tts-premium.ts` — `/api/tts/clova` + `/api/tts/typecast` 핸들러 (requireAuth → 쿼터 → 외부 API → mp3)
- [x] `lib/auth.ts` Env — CLOVA_CLIENT_ID, CLOVA_CLIENT_SECRET, TYPECAST_API_KEY 타입 추가
- [x] `index.ts` — 라우팅 추가
- [x] 검증: `pnpm --filter @tikke/worker typecheck` 통과

## 정책 확정 (체험판)
- Typecast 무료 = **월 30,000자**(1 credit=1 char) 확인. 다수 상시 무료 불가 → 체험판으로.
- 유저당 일 500자 (`TRIAL_DAILY_PER_USER`), 전역 월 28,000자 캡 (`MONTHLY_GLOBAL_CAP`, 과금 차단).
- 숫자는 `lib/tts-quota.ts` 상수 — 조정 쉬움.

## [1b] Typecast voices 동적 ✅
- [x] `/api/tts/typecast/voices` — Typecast `GET /v2/voices` 프록시, 한국어 필터, 1h 캐시
- [x] model `ssfm-v30`으로 수정 (구 v21)
- [ ] **v2/voices 실제 응답 스키마 확인** (배포 후 — language 필드명/값 미확정, isKorean 필터 조정 가능성)

## [2] Desktop 연결 ✅ (typecheck 통과)
- [x] `stores/ttsStore.ts` — TTSProvider += "typecast", TTSConfig.typecastVoiceId, 기본값
- [x] `hooks/useTTSEngine.ts` — synthesize req 2곳 + 타입에 typecastVoiceId
- [x] `electron/main/ipc.ts` — synthesizeTypecast(worker 프록시, Bearer 토큰), case, IPC voices 로더
- [x] `electron/preload/index.ts` — synthesize 타입 + typecastVoices 노출
- [x] `electron/services/settings.ts` — ttsTypecastVoiceId 키 + 기본값
- [x] `pages/TTSSettings.tsx` — 엔진 옵션, TypecastPanel(동적 보이스 목록), KEY_MAP
- [x] 검증: `pnpm --filter @tikke/desktop typecheck` 통과

## [3] 배포 + 확인 (남음)
- [x] `wrangler secret put TYPECAST_API_KEY` (완료)
- [ ] **worker 배포** — 안 하면 desktop이 호출할 엔드포인트 404
- [ ] 배포 후 `/api/tts/typecast/voices` 실측 → 응답 스키마 확인, isKorean 필터 조정 가능성
- [ ] `pnpm dev`로 앱 실행 → TTS 설정에서 "Typecast" 선택 → 보이스 뜨는지 + 테스트 합성
- [ ] CLOVA는 보류(코드 라우트만, secret 없으면 503)

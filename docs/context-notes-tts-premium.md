# 무료 프리미엄 TTS 컨텍스트 노트

## 배경
Naver CLOVA + Typecast를 무료 플랜으로 제공하는 패턴을 tikke에 적용.

## 아키텍처 결정
- **worker 프록시 방식** ("키를 클라에 내림" 금지). 키는 worker secret에만. desktop은 worker 엔드포인트 호출.
  - 이유: tikke 기존 `/api/tts`(TikTok)도 동일 프록시 방식. 키 유출 위험 0.
- **반드시 requireAuth** — 유료 비용 발생하므로 익명 호출 차단. Supabase JWT로 userId 식별 → 쿼터 추적.
  - 기존 `/api/tts`는 무인증이지만 그건 TikTok 무료 TTS라 비용 0이라 OK. 프리미엄은 다름.

## 쿼터 설계 (체험판 — 확정)
- **Typecast 무료 = 월 30,000자** (1 credit = 1 char). 공식 페이지 확인. 다수에게 상시 무료 불가 → 체험판 결정.
- KV `TIKKE_SETTINGS` 재사용. 2단 한도:
  - 유저 일: `tts_quota:{userId}:{YYYYMMDD-KST}`, `TRIAL_DAILY_PER_USER=500`, TTL 26h.
  - 전역 월: `tts_global:{YYYYMM-KST}`, `MONTHLY_GLOBAL_CAP=28000`(30k 무료에 2k 마진), TTL 32d.
- checkQuota가 둘 다 검사, reason("user_daily"/"global_monthly") 반환. commitQuota가 둘 다 누적.
- KV eventually-consistent라 정확 카운트 아님. 과금 차단용 근사치로 충분. 마진 2k가 race 흡수.
- 숫자 조정은 `lib/tts-quota.ts` 상수만 바꾸면 됨.

## 외부 API 스펙 (검증값)
### Naver CLOVA Voice (Premium 가정)
- POST `https://naveropenapi.apigw.ntruss.com/tts-premium/v1/tts`
- 헤더: `X-NCP-APIGW-API-KEY-ID`, `X-NCP-APIGW-API-KEY`, `Content-Type: application/x-www-form-urlencoded`
- form: `speaker`(기본 nminseo), `text`, `volume`(0), `speed`(0=기본, -5~5), `pitch`(0), `format`(mp3)
- 응답: mp3 binary
- 일반 CLOVA면 엔드포인트 `/tts/v1/tts`, speaker mijin/jinho 등으로 달라짐 → 확인 필요

### Typecast (docs 검증)
- TTS: POST `https://api.typecast.ai/v1/text-to-speech`, 헤더 `X-API-KEY`
- body: `{ model:"ssfm-v30", text, voice_id, output:{ volume:100, audio_pitch:0, audio_tempo:1, audio_format:"mp3" } }`
- 응답: audio binary. model 최신 = **ssfm-v30**(구 v21 폐기).
- voice_id 형식 `tc_xxxxxxxx...` (예 `tc_672c5f5ce59fac2a48faeaee`).
- voices: GET `https://api.typecast.ai/v2/voices` (**v2**), 헤더 X-API-KEY.
  - 응답 스키마 미확정 — voice 객체 필드명(language/languages, voice_name/name) 배포 후 실측 필요.
  - worker `/api/tts/typecast/voices`가 프록시+한국어 필터+1h 캐시. isKorean()이 ko/kr/korea* 매칭.
- 무료 voice 라이브러리 598개 중 한국어 다수(Sanghyun/Seohyeon/Juwan/Woony…, 전부 ssfm-v30).

## 미해결/위험
- 무료 티어 전역 총량 소진 시 동작(과금 vs 차단) 불명 → 전역 한도 권장.
- CLOVA Premium/일반 키 타입 미확인.
- Typecast 무료 voice_id 미확보.

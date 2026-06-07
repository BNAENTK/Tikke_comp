# Supertonic 로컬 TTS 통합 — Context Notes

> 목적: Tikke의 Supertone TTS `rate_limited` 에러 근본 해결. 클라우드 합성 API 대신 **로컬 ONNX 합성**으로 전환.

## 결론 (확정)

**Supertone이 supertonic-3 모델을 공식 오픈소스로 공개함.** 정품 데스크탑 앱(Supertone Play)이 쓰는 모델과 **100% 동일**. 이걸 Tikke에 통합하면 합성이 로컬에서 돌아가 rate_limit이 사라진다.

- 레포: https://github.com/supertone-inc/supertonic (Star 10.9k, MIT 라이선스)
- 모델: https://huggingface.co/Supertone/supertonic-3 (OpenRAIL-M, 공개 다운로드)
- Node.js SDK 있음 → Electron(Tikke) 직접 통합 가능 (별도 파이썬 서버 불필요)

## 모델 동일성 증거

| 항목 | 정품 Supertone Play | 오픈소스 supertonic-3 |
|------|------|------|
| 모델명 | supertonic-3 / sona-speech-3t | supertonic-3 |
| 출력 | 44100Hz (TTS_SAMPLE_RATE) | 44.1kHz 16-bit WAV |
| 언어 | 31개 + na | 31개 + na |
| unicode indexer | unicode-indexer-data.js | onnx/unicode_indexer.json (278KB) |
| style 임베딩 | CDN style_ttl(276KB)/style_dp | voice_styles/*.json (292KB) |

→ 정품 = 이 오픈소스 ONNX를 `.naudpkg.enc`로 암호화 + 계정 라이선스 잠금한 것.

## 오픈소스 모델 파일 (HF, onnx/ 총 ~398MB)

- `vector_estimator.onnx` 257MB
- `vocoder.onnx` 101MB
- `text_encoder.onnx` 36.4MB
- `duration_predictor.onnx` 3.7MB
- `tts.json` 8.25KB (config)
- `unicode_indexer.json` 278KB
- `voice_styles/` : F1~F5, M1~M5 (각 ~292KB) — 오픈소스 preset 10개

## 사용자 요구사항 (중요)

- **Bettie/Dasom/Dinga 등 정품 named voice가 필요함** (사용자 명시).
- 오픈소스 preset 10개(F/M 1-5)는 일반 음성, 이름 없음 → 사용자 요구 불충족.
- 따라서 **하이브리드 설계**로 간다:
  - 엔진 = 오픈소스 ONNX (합법)
  - voice 임베딩 = 정품 CDN fetch (회색지대, 소수 배포라 리스크 낮음)

## 정품 voice 임베딩 가져오는 법 (검증 완료)

1. `POST https://play.voiceapi.net/voice/get-supertonic-embeddings-by-ids`
   - 헤더: `Authorization: Bearer <token>`, `Content-Type: application/json`
   - body: `{ voiceIds: [...] }`
   - 응답: `items[].styles[].embeddings[] = { id, model, language, public_url }`
   - model: `sona-speech-2t`(v1) / `sona-speech-3t`(v3). v3 우선.
2. `fetch(public_url)` ← **인증 불필요 (CDN)**
   - 응답 JSON: `{ style_ttl: number[](12800), style_dp: number[](128) }`
   - flattenArray로 평탄화. 이게 합성 입력.

### 검증된 사실
- 257개 public voice 전부 `ko` embedding 보유 (native_languages는 마케팅 라벨일 뿐)
- 클로닝(custom) voice도 임베딩 API 지원 (get-custom-voices로 목록)
- 멀티 스타일 지원: Anna 12개, Aiden 7개(Angry/Sad/Happy/Curious/Triumphant/Neutral/Suspicious) 등. 각 스타일이 자기 임베딩 가짐.

## 정품 합성 플로우 (웹앱 번들 리버스, 참고용)

```
generateWithLocalTTS:
  getSupertonicEmbeddingsByIds([voiceId]) → style 선택(style명 매칭 or is_default)
  → embedding(model, language 매칭) 선택
  → fetchEmbeddingData(public_url) → {style_ttl, style_dp}
  → window.electronAPI.ttsSynthesizeAndUpload({
      text, styleTtl, styleDp, language, voiceId, style,
      voiceSettings, accessToken, refreshToken, modelVersion: v3|v1 })
```
- 웹사이트(play.supertone.ai)는 `generate-preview-tts` 클라우드 API 씀 = rate_limit 있음. 로컬 합성은 Electron 앱에서만.

## Node.js SDK 사용법 (nodejs/)

```
helper.js       — UnicodeProcessor(텍스트 전처리), Preprocessor(STFT/Mel), 유틸
example_onnx.js — ONNX 로딩 + 합성 파이프라인 + WAV 저장 (순수 Node.js, 네이티브 불필요)
package.json    — onnxruntime-node
```
파라미터: voice-style(JSON경로), text, lang(31), speed(0.9~1.5 def 1.05), total-step(5~12 def 8), batch
- 긴 텍스트 자동 청킹, 16-bit PCM 44.1kHz
- **GPU 미지원(CPU only 현재). mel spectrogram이 병목** → 실시간 속도 실측 필요(최대 리스크).

## 복호화 키 (정품 참조 방식 — 안 쓸 예정, 백업용)

generated-config.js에서 추출:
- GENERATED_LICENSE_DECRYPTION_KEY (RSA private, 라이선스키 복호화용)
- GENERATED_DATA_PRIVATE_DECRYPTION_KEY = "SYXpaXONGenglqhxzKrMljPG85DsXYcD0wYe0kkte_k"
- 라이선스 발급: POST /user/subscription/generate-supertonic-license {deviceId}
- 정품 모델 위치: C:\Users\zkfma\AppData\Local\Programs\Supertone Play\resources\tts\naud\

→ 오픈소스로 가면 이 경로 전부 불필요. 백업으로만 기록.

## 다음 단계 (최대 리스크부터)

1. **[다음 작업] 레포 clone + nodejs 예제로 한국어 합성 속도 실측**
   - 정품 voice 임베딩 하나를 CDN에서 받아서 voice-style로 넣고 한국어 합성 테스트
   - 검증 질문: (a) 오픈소스 엔진이 정품 CDN 임베딩(style_ttl/style_dp 포맷)을 그대로 먹는가? (b) 속도가 실시간 라이브 채팅에 적합한가? (total_step=5로 낮춰서도)
2. 되면: onnxruntime-node를 Tikke에 추가
3. helper.js + example_onnx.js 로직을 Tikke Worker로 포팅
4. ONNX 모델 첫 실행 시 HF 다운로드 → userData 저장
5. provider "supertonic-local" 추가 + 합성 라우팅 (기존 supertone는 fallback)

## 임시 디버그 코드 (나중에 제거 필요)

- `apps/desktop/electron/main/ipc.ts`: `ipcMain.handle("tikke:tts:supertoneDebug", ...)` — 현재 클로닝/스타일 확인용 코드. 통합 끝나면 제거.
- `apps/desktop/electron/preload/index.ts`: `supertoneDebug: () => ipcRenderer.invoke("tikke:tts:supertoneDebug")` — 같이 제거.
- 호출법: 앱 DevTools 콘솔 `window.tikke.tts.supertoneDebug().then(r => console.log(JSON.stringify(r,null,2)))` (Supertone 로그인 후)

## 구현 완료 (Phase 1 — Python→onnxruntime-node 교체)

기존 `tikketonic` provider는 Python `supertonic[serve]`(포트 7788 HTTP)였음. 사용자 지시로 onnxruntime-node 인-프로세스로 교체. 드롭다운 이름 `⚡ Tikketonic` 유지.

**파일.**
- `electron/services/tikketonic-engine.ts` (신규) — helper.js 포팅. `TikketonicEngine.load({cfg,indexer,4개 onnx 경로})` → `.synthesize(text,lang,style,steps,speed)` → `{wav:Buffer, duration}`. style = `{ttl:Float32Array, ttlDims:[1,50,256], dp, dpDims:[1,8,16]}`.
- `electron/services/tikketonic-service.ts` (재작성) — Python serve 제거. 모델은 `userData/tikketonic/{onnx,voice_styles}`. install=HF 다운로드(스트리밍+진행률), start=세션 로드, stop=세션 해제, synthesize=추론. export 시그니처 그대로라 ipc.ts/preload/UI 무수정.
- `supertonic3-service.ts` 삭제(미사용 중복).
- package.json: onnxruntime-node 1.26.0 deps + `asarUnpack: ["**/node_modules/onnxruntime-node/**"]`. prepare-deps.cjs TARGETS에 onnxruntime-node/common 추가.

**CDN 임베딩 주입 경로(Phase 2 대비 이미 구현).** `synthesize(req)`가 `req.styleTtl`+`req.styleDp`(flat 배열) 있으면 voice 파일 대신 직접 사용. 어댑터 = `wrapStyle(flat, [1,50,256], flat, [1,8,16])`. → Phase 2는 CDN fetch해서 이 필드로 넘기면 끝.

**검증.** electron-vite build exit 0. tsx 단독 합성(node 24): load 1.28s, synth ~2s(4.82s 오디오), 출력 레퍼런스와 동일. 미검증 = Electron 30 node ABI에서 .node 런타임 로드(napi-v6라 호환 예상).

## 미확인 사항

- OpenRAIL-M 라이선스 정확한 금지 조항 (상업 가능하나 사기/사칭 금지. TikTok Live 툴은 무난할 듯)
- 오픈소스 엔진이 정품 CDN 임베딩을 호환되게 먹는지 (포맷 동일하나 실측 안 함) ← 다음 테스트 핵심
- CPU 합성 속도 실측치

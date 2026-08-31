# 내 목소리 TTS — 엔진 조사 기록 (구현 없음)

> 2026-08-25 기준. Chatterbox 통합을 시도했다가 속도 미달로 전량 원복했다.
> tikke 코드에는 아무것도 남아 있지 않다(git checkout 으로 되돌림).
> 이 문서는 다시 시도할 때 같은 삽질을 반복하지 않기 위한 실측 기록이다.

## 결론 먼저
- 요구: 구글 TTS급 지연(0.37s) + ko/en/ja/zh 크로스링구얼 클로닝 + 자유 라이선스 + Windows
- **네 조건을 동시에 만족하는 로컬 TTS 엔진은 없다** (2026-08 기준, 후보 6종 실측)
- 다음 방향: TTS 로 클로닝하지 말고 **빠른 TTS + 음성변환(RVC 등)으로 분리**

---

## Chatterbox 시도 상세 (원복됨)

## 실측 결과 (2026-08-24, RTX 4070 Ti 12GB)

**문서/마케팅 수치는 틀렸다. 실측이 정답이다.**

광고된 "RTF 6x, 첫 소리 <150ms"는 **Turbo(350M) 기준이고, Turbo는 다국어를 지원하지
않는다** (`generate()` 시그니처에 `language_id` 없음, `get_supported_languages` 없음).
ko/en/ja/zh 를 하려면 Multilingual V3 를 써야 하고, 그 실측치는 아래와 같다.

워밍업 후 실제 채팅 길이 문장 10개 측정:

| 언어 | 중앙 RTF | 예시 |
|---|---|---|
| ko | 1.41~2.09 | "안녕하세요 반가워요" → 오디오 2.64s / 합성 3.72s |
| en | 1.35~1.59 | "hello everyone" → 오디오 4.48s / 합성 6.03s |
| ja | 1.49~1.90 | "ありがとうございます" → 오디오 2.16s / 합성 3.22s |
| zh | 1.42~1.55 | "谢谢你的礼物" → 오디오 1.48s / 합성 2.30s |

- **중앙 RTF 1.51 / 중앙 합성시간 2.98s / 최대 6.03s**
- RTF > 1 = 실시간보다 느림. 오디오 1초 만드는 데 1.5초 걸린다.
- 모델 로드 12.2s (기동 1회)
- 첫 합성은 워밍업 포함 RTF 5.25 → 서버 상주 + 워밍업 필수
- CUDA 정상, 4070 Ti 사용 확인

**대안 없음을 확인함**
- Turbo(고속) — 다국어 미지원. 탈락.
- 스트리밍 API — `generate_stream` 없음. 첫 소리 조기 재생 불가.
- 즉 로컬 다국어 클로닝의 속도 하한이 RTF ~1.5 이다.

**함의**: 엣지/구글 TTS(~300ms)보다 약 10배 느리다. tikke 의 prefetch(3) + stale
drop(20s) 이 완충하지만, 채팅이 몰리면 따라잡지 못하고 오래된 메시지가 버려진다.
한산한 방송은 문제없고 폭주하는 방송은 일부 채팅을 못 읽는다.

## 크로스링구얼 검증

한국어 참조 클립 하나로 en/ja/zh 합성이 실제로 동작함을 확인 (smoke_test.py).
지원 언어 23개에 ko/en/ja/zh 모두 포함. `language_id` 로 출력 언어를 정한다.
중국어는 `zh` (tikke 의 `zh-CN` 아님) — `CHAT_LANG_TO_LANGUAGE_ID` 로 매핑.

## 설치 함정 — CUDA torch 덮어쓰기

`pip install chatterbox-tts` 가 **CUDA torch 를 제거하고 CPU wheel 로 교체한다.**
실제로 발생함: torch 2.5.1+cu121 → 2.6.0+cpu.
반드시 chatterbox-tts 설치 **후에** CUDA torch 를 재설치하고 `torch.cuda.is_available()`
를 다시 확인할 것.
    pip install --force-reinstall torch==2.6.0 torchaudio==2.6.0 \
      --index-url https://download.pytorch.org/whl/cu124

## 왜 Chatterbox인가

- MIT 라이선스 → 모델명 리브랜딩 가능. 앱에서 자체 이름으로 노출 가능.
- zero-shot → 학습/피팅 불필요. 참조 클립만 있으면 즉시 클로닝.
- 로컬 실행 → 네트워크 왕복 0. 클라우드 TTS(엣지/구글)의 200~500ms 네트워크
  지연이 사라져서, GPU면 오히려 더 빠를 수 있다.
- 크로스링구얼 → 한국어 클립 하나로 en/ja/zh 발화 가능. 기존 한국어 TTS가
  외국어를 못 읽던 문제의 해결책.

## 대사 설계 — Supertone식을 버린 이유

기존 `tikketone/RefAudioPanel.tsx:6`의 `SCRIPTS[8]`은 낭독체 문어체다.
("파란 하늘 아래 바람이 살며시 불어옵니다" 류)

이건 Supertone Play처럼 **학습형** 클로닝에는 맞다 — 음소 커버리지가 중요하니까.
그러나 zero-shot은 참조 클립의 **프로소디(억양/속도/톤)까지 전이**한다.
낭독체로 녹음하면 틱톡 채팅을 시 낭독하듯 읽는다.

그래서 레지스터(말투) 기준으로 재설계했다.
- 대화체 — 일반 채팅 읽기
- 하이텐션 — 선물/이벤트 알림
- 차분 — 공지/설명

**업계도 같은 방식이다.** ElevenLabs는 음소 균형 스크립트가 아니라 톤별
스크립트(narrative / conversational / advertising)를 제공하고,
"목표하는 tonality에 맞는 스크립트를 고르라"고 안내한다. 또 "한 녹음 안에서
텐션을 섞으면 모델이 불안정해진다"고 명시한다 → 그래서 클립 하나는 하나의
레지스터로 통일하고, **여러 클립을 concat하지 않는다.**

concat 금지는 기존 흐름과의 차이점이다. `RefAudioPanel.tsx:131`은
`concatenateWavBuffers`로 8초 × 3~8개를 하나로 합친다(24~64초). Chatterbox
목표는 10~20초 단일 테이크이고, 서로 다른 테이크를 이어붙이면 이음새에서
톤이 튀어 임베딩이 지저분해진다.

## 미해결 — cfg_weight

문서가 자기모순이다.
- 유효 범위 0.2~1.0, 기본 0.5
- 그런데 크로스링구얼 팁은 "참조 언어 억양을 줄이려면 0.0" — 범위 밖

게다가 `cfg_weight`는 classifier-free guidance 강도라서, 낮추면 억양만이 아니라
conditioning 준수 전반이 약해진다. 즉 값을 너무 내리면 위에서 공들인
레지스터 구분(하이텐션 vs 차분)까지 뭉갤 수 있다.

→ 상수로 못 박지 않고 `CFG_WEIGHT_CANDIDATES = [0.5, 0.3, 0.2, 0.0]` A/B 후보로 뒀다.
실측 후 확정할 것.

## 언어 코드 불일치

tikke 번역 기능은 `"ko" | "en" | "ja" | "zh-CN"`을 쓴다 (`translationStore.ts:14`).
Chatterbox `language_id`는 ISO 2자리라 **중국어가 `zh`** 다 (`zh-CN` 아님).

→ `CHAT_LANG_TO_LANGUAGE_ID` 매핑을 `cloneScripts.ts`에 뒀다. 배선할 때 반드시 경유.

## 언어 감지 — 외부 의존성 없이

틱톡 채팅은 언어가 섞여 온다. 메시지마다 언어를 정해야 TTS에 넘길 수 있다.

ko/en/ja/zh 4개는 사용 문자가 갈려서 라이브러리 없이 판별 가능하다.
- 가나가 하나라도 있으면 → ja (가나는 일본어에만 쓰임, 한자 혼용이어도 확정)
- 한글이 최다 → ko (호환 자모 포함 → "ㅋㅋㅋ"도 한국어로 잡힘)
- 한자만 있고 가나 없음 → zh
- 라틴 최다 → en

11개 케이스로 검증 통과. franc 같은 라이브러리를 넣지 않은 이유는 4개 언어
한정이면 문자 판별이 더 정확하고 빠르기 때문 (통계 기반 감지는 짧은 채팅에 약함).

## 파일 배치 — 재검토 필요

`cloneScripts.ts`를 `apps/desktop/src/data/`에 뒀다 (기존 `ttsVoices.ts`와 같은 위치).
그런데 실제 녹음 UI(`RefAudioPanel`)는 `apps/tikketone`에 있다. 두 앱이 같은
대사 세트를 쓰게 되면 `tikke/CLAUDE.md` 규칙대로 `packages/shared`로 옮겨야 한다.

## 성능 구조 — 기존 파이프라인이 이미 방어 중

Chatterbox가 느려도 tikke의 기존 구조가 상당 부분 완충한다.
- prefetch depth 3 (`useTTSEngine.ts:237`) — 재생 중 다음 3개 미리 합성
- stale drop 20초 (`ttsStore.ts:207`) — 백로그 쌓이면 오래된 채팅 폐기
- 오디오 캐시 LRU 100 (`useTTSEngine.ts:236`) — 반복 채팅 재합성 없음

즉 체감 지연은 합성 지연보다 낮다. 다만 클로닝은 일반 TTS보다 무거우므로
채팅 폭주 시 실측이 필요하다.


---

# 엔진 후보 실측 비교 (2026-08-25, RTX 4070 Ti)

같은 문장 5개(ko×2/en/ja/zh)로 측정. Google/Edge 는 클라우드 기준선.

| 엔진 | 첫 소리 | RTF | 라이선스 | 한국어 | 판정 |
|---|---|---|---|---|---|
| Edge TTS | 0.25s | - | 클라우드 | O | 기준선 |
| Google speech-api v2 | 0.37s | - | 클라우드 | O | 기준선 |
| **XTTS-v2** | **0.47s** | **0.48** | **CPML 비상업** | O | 속도 O / 라이선스 X |
| Chatterbox Multilingual | 1.98s | 1.30 | MIT | O | 속도 X |
| CosyVoice 3 (스트리밍) | 2.05s | 0.95 | Apache-2.0 | O | 속도 X |
| Qwen3-TTS 클론(bf16 GPU) | 14.4s | 2.92 | Apache-2.0 | O | 속도 X |
| ZONOS2 | 측정중 | - | MIT | O (Tier2) | - |

## 핵심 교훈

1. **문서 수치는 못 믿는다.** Chatterbox 광고 RTF 6배 → 실측 1.3. CosyVoice3 광고 첫패킷
   150ms → 실측 2,050ms. 반드시 같은 문장으로 직접 측정할 것.
2. **첫 소리(TTFA)가 지표다.** 총 합성시간이 아니라. Edge 가 빠른 것도 스트리밍 때문
   (첫 청크 0.25s / 전체 0.38s). 짧은 채팅은 청크가 1~3개라 스트리밍 이득이 작다.
3. **RTF < 1.0 이 실사용 하한.** 넘으면 채팅 폭주 시 큐가 밀려 stale drop 으로 버려진다.
4. **라이선스를 속도보다 먼저 볼 것.** XTTS-v2 는 속도를 만족했으나 CPML 비상업이고
   Coqui 가 2024-01 폐업해 상업 라이선스를 살 주체조차 없다. tikke 배포 불가.

## 탈락 사유 상세

- **Chatterbox Turbo**: 6배 빠르지만 `language_id` 파라미터 자체가 없음 = 다국어 불가.
- **CosyVoice3**: `cli/model.py:360` 에서 `token_hop_len` 이 인스턴스에 누적돼 호출마다
  25→100 으로 커진다(첫 청크 4.2s). 매 호출 리셋해도 2.05s.
- **Qwen3-TTS 클론**: tikke 에 이미 통합돼 있고 Apache-2.0 에 ko/ja/zh/en 지원
  (`get_supported_languages()` 확인). 그러나 클로닝 RTF 2.92 로 가장 느리다.
  `language="Auto"` 하드코딩(model.py:218)은 고칠 수 있으나 속도가 안 나와 무의미.

## 시도했으나 효과 없던 최적화 (Chatterbox 기준)

fp16/bf16, autocast, TF32, 배치1+cfg0, 어텐션 레이어별 eager, torch.compile(triton
버전 충돌). 유효했던 것은 repetition_penalty 2.0→1.2 (4.04s→2.19s) 와
s3gen flow-matching 스텝 10→4 (2.27s→1.98s) 뿐.

---

# 새 방향 — TTS + 음성변환 분리 (실측 완료 2026-08-25)

## 왜 분리하나
"다국어 + 내 목소리"를 한 모델이 다 하려니 0.5B 자기회귀 LLM 이 필요했고 그게 RTF 1.0 의
벽이었다. 둘로 나누면 각각은 이미 충분히 빠르다.

    채팅 텍스트 → Edge TTS (언어 담당) → RVC 음성변환 (목소리 담당) → 출력

음성변환은 **내용을 보존하고 음색만 바꾼다.** 일본어 오디오를 넣으면 일본어가 내 목소리로
나온다. 모델이 언어를 이해할 필요가 없어서 언어 확장이 공짜다.

## RVC 실측 (RTX 4070 Ti, 공식 pretrained_v2 f0G48k)

| 입력 길이 | HuBERT | RMVPE f0 | Generator | 합계 | RTF |
|---|---|---|---|---|---|
| 1s | 4.1ms | 6.9ms | 35.4ms | **46ms** | 0.046 |
| 2s | 4.0ms | 7.6ms | 42.8ms | **55ms** | 0.027 |
| 4s | 5.7ms | 10.8ms | 67.7ms | **84ms** | 0.021 |

RTF 0.02~0.05 — 실시간의 20~50배. 지금까지 본 모든 TTS 클로닝 엔진과 차원이 다르다.
비자기회귀 구조라 토큰을 하나씩 뽑지 않기 때문.

## 예상 전체 지연

    Edge TTS 380ms (실측) + RVC 55ms = 약 435ms
    (Edge 첫 청크 250ms 기준으로 청크 단위 변환하면 더 줄일 여지 있음)

Google 370ms / Edge 380ms 와 같은 급. 요구사항 충족.

## 라이선스

RVC-Project/Retrieval-based-Voice-Conversion-WebUI = **MIT**.
베이스 모델은 VCTK 오픈 데이터셋 학습이라 저작권 이슈 없다고 명시.
fairseq 의존성이 이미 제거돼 Windows 에서 MSVC 없이 설치된다 (구버전 안내와 다름).

탈락시킨 것: seed-vc(GPL-3.0), blaisewf/rvc-cli(CC BY-NC).
Applio(IAHispano) 도 MIT 라 대안이 될 수 있다.

## 트레이드오프

zero-shot 이 아니다. 사용자 음성 10분을 녹음해 한 번 학습해야 한다(4070 Ti 수십 분).
대신 학습 후 추론이 압도적으로 빠르다 — 학습형이 zero-shot 보다 빠른 것은 구조상 당연.
UX 상 "10분 녹음 + 1회 학습 대기"를 받아들일 수 있는지가 관건.

## 남은 검증

- [ ] 실제 사람 목소리 10분으로 학습 → 품질 확인 (지금은 속도만 측정)
- [ ] Edge TTS 출력(한국어/영어/일본어/중국어)을 실제로 변환해 언어 보존 확인
- [ ] 학습 소요 시간 실측
- [ ] tikke 파이프라인 결합 시 총 지연 (prefetch 와 어떻게 맞물리는지)

---

# 공개 데이터 검증 완료 (2026-08-25, LJSpeech 11분)

## 결과 요약 — 가설 성립

| 항목 | 결과 |
|---|---|
| 학습 데이터 | LJSpeech(퍼블릭 도메인) 11분 / 102클립 |
| 전처리 | 50초 (분할 9s + f0 19s + HuBERT 24s) |
| 학습 | 60에폭 **약 12분** (RTX 4070 Ti, bs=8, 20에폭당 4분) |
| 변환 지연 | ko 63ms / en 52ms / ja 62ms / zh 57ms (**중앙 59ms**, RTF 0.016) |
| 언어 보존 | **4개 언어 전부 완전 보존** (Whisper 전사로 검증) |
| 화자 유사도 | 4개 언어 모두 변환 후 학습목소리 유사도 상승 |

## Whisper 전사 검증 (핵심 가설)

    ko 변환후: 안녕하세요. 반가워요. 오늘 방송 시작합니다.
    en 변환후: Hello everyone, thanks for the gift.
    ja 변환후: ありがとうございます。今日もよろしくお願いします。
    zh 변환후: 謝謝你的禮物 今天也請多多關照

한국어만 학습한 모델이 아님에 유의 — LJSpeech 는 **영어 낭독**이다. 그런데도
한국어/일본어/중국어 변환에서 내용이 완전히 보존됐다. 음성변환이 언어를 이해하지
않고 음색만 바꾼다는 것이 실증됐다. → K님 한국어 10분으로 학습해도 4개 언어 다 나온다.

## 화자 특성 (MFCC 코사인 + 중앙 F0)

학습목소리(LJ) 중앙 F0 = 217.5Hz

| 언어 | 원본 vs LJ | 변환 vs LJ | 변환 F0 |
|---|---|---|---|
| ko | 0.960 | **0.973** | 259.8 |
| en | 0.946 | **0.975** | 222.7 |
| ja | 0.939 | **0.973** | 278.2 |
| zh | 0.980 | **0.989** | 243.3 |

전부 상승. 다만 MFCC 유사도는 화자 판별력이 약한 지표라 참고용이다.
최종 판단은 청취로 해야 한다 (demo/ 폴더에 파일 생성함).

## 총 지연

    Edge TTS 380ms(실측) + RVC 59ms = 약 439ms
    vs Google 370ms / Edge 380ms → 같은 급

## 재현 절차 (Windows, MSVC 불필요)

1. `git clone RVC-Project/Retrieval-based-Voice-Conversion-WebUI`
2. torch cu124 먼저 설치 → `pip install -r requirments_cu128_py312.txt` →
   **torch 재설치**(requirements 가 CPU torch 로 덮어씀)
3. 자산: HuBERT=`lengyue233/content-vec-best` → `assets/hubert_base`
   (+ `facebook/hubert-base-ls960` 의 `preprocessor_config.json` 복사 필요),
   `rmvpe.pt`, `pretrained_v2/f0G48k.pth`, `f0D48k.pth`
4. `python -m train.preprocess <데이터> 48000 4 logs/<exp> False 3.0`
5. `python -m train.dataset.extract_f0 cuda 1 0 0 "<abs>/logs/<exp>" False`
6. `python -m train.dataset.extract_hubert_feature cuda 1 0 0 "<abs>/logs/<exp>" v2 False`
7. **filelist.txt 직접 생성** (webui 가 만드는 파일, CLI 경로엔 없음)
   형식: `gt.wav|feature.npy|f0.wav.npy|f0nsf.wav.npy|0` (경로 백슬래시 이스케이프)
8. `python -m train.train -e <exp> -sr 48k -f0 1 -bs 8 -g 0 -te 60 -se 20 -pg ... -pd ... -v v2`

## 함정 기록

- `pip install -r requirements` 가 CUDA torch 를 CPU 로 덮는다 (Chatterbox 때와 동일).
- content-vec 저장소에 `preprocessor_config.json` 없음 → facebook/hubert-base-ls960 것 복사.
- `filelist.txt` 없으면 학습이 FileNotFoundError. mute 자산은 없어도 됨.
- coarse pitch 를 mel 스케일 1~255 로 양자화해야 함. raw Hz 넣으면 CUDA assert.
- resemblyzer/webrtcvad, fairseq 는 MSVC 필요 → 쓰지 말 것.
- bash `/tmp` 와 Windows Python 경로가 다름. 절대경로 쓸 것.

## 남은 것

- [ ] K님 실제 음성 10분으로 학습 → 청취 품질 확인
- [ ] tikke 파이프라인 결합 (Edge 합성 후 RVC 변환 단계 삽입)
- [ ] 채팅 폭주 시 큐 동작 (RTF 0.016 이라 여유 클 것으로 예상)
- [ ] 모델 배포 방식 (사용자별 .pth 58MB — 로컬 보관? 클라우드?)

## index(retrieval) 적용 결과

`python -m train.train_index <exp> v2 "<abs>/assets/indices" 4` 로 생성 (수초).
추론 시 HuBERT 특징을 index 에서 top-8 검색해 가중평균(rate=0.75)으로 섞는다.

| 언어 | 원본 vs LJ | index 없이 | index 적용 |
|---|---|---|---|
| ko | 0.960 | 0.973 | 0.974 |
| en | 0.946 | 0.975 | **0.980** |
| ja | 0.939 | 0.973 | **0.977** |
| zh | 0.980 | 0.989 | 0.989 |

지연은 59ms → 69ms (+10ms). 유사도가 오르므로 **index 사용 권장**.
총 지연 재계산: Edge 380ms + RVC 69ms = **약 449ms**.

## 방식의 성격 — 반드시 알아야 할 차이

RVC 는 **음색만** 바꾼다. 억양·속도·리듬은 원본 오디오(Edge TTS) 것이 그대로 간다.
즉 결과물은 "내 목소리인데 Edge TTS 처럼 말하는" 것이다.

Chatterbox 계열(zero-shot TTS 클로닝)은 반대로 프로소디까지 전이하므로
"방송 말투로 녹음해야 한다"는 제약이 있었다. RVC 는 그 제약이 없는 대신
내 말버릇을 옮길 수도 없다. 채팅 읽기 용도로는 일정한 발화가 오히려 장점일 수 있다.

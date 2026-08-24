# Chatterbox 통합 — 결정 사항과 이유

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

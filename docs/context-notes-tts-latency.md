# TTS 지연 개선 — 결정 사항과 이유

## 배경

tikketonic(supertonic-3 ONNX 로컬)으로 채팅을 읽을 때 지연이 길다는 문제. "보이스 클로닝이라 어쩔 수 없다"는 가설을 코드로 검증한 결과 **기각**.

## 병목은 합성이 아니라 재생 처리량

측정치 (`context-notes-supertonic.md:121`) — 엔진 로드 1.28초(1회성), 합성 ~2초에 4.82초 오디오(RTF ≈ 0.41).

보이스 임베딩도 원인 아님. `tikketonic-service.ts:29`의 `_styleCache`가 `voiceId:style:lang` 키로 캐싱해 첫 1회만 CDN fetch, 이후 메모리 히트.

실제 구조 — `useTTSEngine.ts:425`의 `isSpeakingRef`가 재생 중 큐 처리를 막음(오디오 중첩 불가라 당연). 그래서 사이클은 합성(≈1.2초) + 재생(≈3초) = 4초 이상인데, 채팅은 2.5초마다 도착. 큐가 쌓이고 `ttsStore.ts:209`의 `STALE_MS`(20초)에 걸려 뒤엣것이 버려짐.

**결정적 논거** — 합성이 0초여도 재생 3초 > 도착 2.5초라 큐는 여전히 쌓임. 합성 최적화만으로는 해결 불가.

뒷받침 — `context-notes-tts-missing.md:135` 시뮬레이션에서 2.5초 간격 전달률 53%, prefetch+timeout 적용 시 88%.

## 로컬이 유리한 부분

`context-notes-tts-missing.md:29` — gspeech는 평균 280ms지만 10%가 15~42초 스톨(네트워크 꼬리 지연). 로컬은 네트워크가 없어 이 스톨이 구조적으로 없음. 평균은 Google 우위, 최악은 로컬 우위.

## 결정 — prefetch 깊이 상향을 1순위로

`useTTSEngine.ts:450`이 `queue[0]` 1건만 prefetch. 반면 `synthCache`는 8칸까지 보관(`:356`). 즉 **캐시 여유는 있는데 채우지를 않고 있음**.

깊이를 3으로 올리면 재생 1건 동안 3건이 미리 합성돼, 도착 간격이 재생 시간보다 짧아도 합성이 재생에 완전히 가려짐. 캐시 상한 8 이내라 기존 evict 로직 그대로 유효.

깊이를 8(캐시 상한)까지 올리지 않은 이유 — 스테일 드롭으로 재생되지 못할 항목까지 합성하면 CPU/GPU를 헛씀. 3이면 재생 1건(≈3초) 동안 합성(≈1.2초×3) 을 덮는 선에서 균형.

Supertone 계열은 `prefetchSynth:351`에서 이미 early return — rate limit 악화 방지. 이 동작은 유지됨.

## 실측 결과 (2026-08-08, 벤치 스크립트 직접 실행)

`harness/scenarios/tikketonic-rtf.ts`로 엔진 단독 로드 후 측정. 엔진은 electron 의존이 없어(`onnxruntime-node` + `os`만) tsx로 단독 실행 가능.

- 엔진 로드 1,776ms, 워밍업 1,940ms (둘 다 1회성)
- EP는 `dml+cpu` 혼합으로 세션 생성되나 평균 RTF 0.359 → **GPU 실질 미가동 추정**. ORT가 "Some nodes were not assigned to the preferred execution providers" 경고를 냄.

**핵심 발견 — 합성 시간이 텍스트 길이와 거의 무관.**

| 텍스트 | 합성 | 오디오 | RTF |
|---|---|---|---|
| 안녕하세요 | 826ms | 1.43s | 0.577 |
| 형 오늘도 방송 재밌어요 | 831ms | 2.19s | 0.379 |
| 신규시청자님 안녕하세요 처음 왔어요 | 948ms | 3.30s | 0.287 |
| 오늘 방송 진짜 재밌게…(장문) | 951ms | 4.96s | 0.192 |

오디오 길이가 3.5배 늘어도 합성은 826→951ms(15%)뿐. 즉 **RTF는 이 엔진에서 오해를 부르는 지표**고, 실제로는 고정 오버헤드가 지배함.

## 고정 오버헤드의 정체 — diffusion step 수

`harness/scenarios/tikketonic-steps.ts`로 스윕. 동일 문장(오디오 2.19s) 기준.

| steps | 합성 |
|---|---|
| 2 | 272ms |
| 3 | 371ms |
| 4 | 526ms |
| 5 | 646ms |
| 6 | 730ms |
| 8 (기본) | 928ms |
| 10 | 1,138ms |
| 12 | 1,334ms |

**step당 약 110ms의 선형 관계.** `TikketonicRequest.steps`(`tikketonic-service.ts:201`)가 5~12 범위로 이미 노출돼 있고 기본이 8.

즉 무음 상태 첫 발화 지연(≈930ms)을 줄이는 손잡이는 GPU가 아니라 **step 수**임. 8→4면 526ms로 43% 단축, 목표 0.5초에 도달함.

품질 트레이드오프는 기계로 판정 불가 → step 3/4/5/8 WAV 샘플을 뽑아 사용자 청취 판정에 넘김(`harness/scenarios/tikketonic-samples.ts`).

## 미확인 사항

`tikketonic-engine.ts:179`가 DirectML(GPU) 우선 + CPU 폴백 혼합으로 세션을 만들지만, 같은 파일 주석(`:178`)대로 **실제 GPU 가동 여부는 RTF로만 판정 가능**. `context-notes-supertonic.md`의 RTF 0.41은 CPU 시절 값일 수 있음. 실측 전까지 현재 RTF는 미확정.

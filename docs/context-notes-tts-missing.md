# TTS 간헐 무음 조사 (2026-08-08)

증상: "TTS가 가끔 안 나온다." 코드 변경 없이 원인부터 확정하는 것이 이번 범위.
(앞서 넣었던 수정은 커밋 `c53f63a` → `73fdf7d`로 전부 되돌림.)

## 실제 사용자 설정 (tikke.db `app_settings` 실측)

| 키 | 값 |
|---|---|
| ttsProvider | `tiktok` |
| ttsTiktokVoiceId | `gspeech:ko-KR:female` |
| **ttsRandomVoice** | **`true`** |
| ttsGoogleSpeechV2Key | (빈 값 → 하드코딩된 공개키 사용) |
| ttsAllowNormal / Followers | false / false |
| ttsAllowSubscribers / Moderators / TeamMembers / TopGifters | true / true / true(min 1) / true(top 3) |
| ttsEventChat | true (gift/follow/member/share는 off, subscribe만 on) |

## 랜덤 보이스가 굴리는 풀 — 5개뿐

`TIKTOK_VOICE_IDS`를 소스에서 그대로 뽑아 확인:

```
kr_002, kr_003, kr_004            → TikTok TTS API (세션 쿠키 필요)
gspeech:ko-KR:female, :male       → Google speech-api/v2 (비공식 공개 엔드포인트)
```

`applyRandomVoice`는 균등 추첨이므로 **매 발화의 2/5(40%)가 gspeech 경로**로 간다.

## 실측 1 — gspeech 응답 지연이 튄다

앱과 같은 URL·파라미터로 직접 호출(2회차, 총 55건).

| | 1회차 30건 | 2회차 25건 |
|---|---|---|
| 실패 | 0 | **1 (HTTP 500, 27.9초)** |
| 20초 이상 | 2 (15.7초·30.2초) | 1 (42.4초) |
| 2초 이상 | 3 | 6 |
| 평상시 | ~280ms | ~250ms |

즉 **정상은 0.3초인데 열 번에 한 번꼴로 15~42초** 걸린다. 200 OK로 성공하므로 에러 로그도 안 남는다.

## 실측 2 — 그 지연에 타임아웃이 없다

`ipc.ts:3049` gspeech 호출은 `browserFetch(googleUrl, {headers})`.
`browserFetch`(`ipc.ts:2604`)는 `net.fetch`를 헤더만 씌워 그대로 부른다 — **`signal` 없음**.
다른 TTS 경로들이 `AbortSignal.timeout(10000)`을 다는 것과 대조된다.
따라서 42초 응답이면 앱은 42초를 그대로 기다린다.

## 연결 고리 — 왜 "여러 개"가 사라지나

`useTTSEngine`은 발화 하나가 끝날 때까지 `isSpeakingRef`로 큐를 잠근다.
`ttsStore.ts:256`의 스테일 드롭은 **20초** 넘게 기다린 항목을 조용히 버린다.

합성이 30초 걸리면:
1. 그 30초 동안 락이 잡혀 있고
2. 그 사이 들어온 채팅이 큐에 쌓이고
3. 락이 풀리는 순간 그것들은 이미 20초를 넘겨 **한꺼번에 버려진다**

한 번의 느린 응답이 **그 뒤 20초치 채팅을 통째로 날린다.** 로그는 어디에도 안 남는다.

## 랜덤 보이스가 이걸 더 나쁘게 만든다

- `useTTSEngine.ts:603` — 이벤트 도착 시 `prefetchSynth`를 **무조건** 부른다
- `useTTSEngine.ts:851` — `cfg.randomVoice ? undefined : synthCache.get(...)` — 랜덤이면 **그 결과를 안 읽는다**
- `useTTSEngine.ts:852` — `if (cached) synthCache.delete(...)` — 안 읽으니 **지워지지도 않는다**
- `useTTSEngine.ts:334` — `if (synthCache.size >= 8) return;`

결과: 랜덤 ON이면 처음 8건은 **버려질 요청을 한 번씩 더** 쏘고(같은 엔드포인트 부하 2배),
그 뒤로는 `synthCache`가 8에서 안 줄어 **prefetch가 세션 내내 죽는다.**
즉 모든 합성이 **재생 차례에, 락을 잡은 채로** 일어난다. 지연이 그대로 무음으로 번역된다.

## 확인 못 한 것

- `kr_002/003/004`(TikTok TTS API) 경로의 실패율 — 사용자 세션 쿠키가 필요해 측정하지 않음
- 위 기전이 실제 방송에서 관측된 무음의 전부인지 — 드롭 카운터가 없어 사후 확인 불가

## 증거로 기각한 가설

- **스테일 판정이 틱톡 시각을 쓴다** — 아님. `tiklive-normalizer.ts:20` `timestamp: Date.now()`(main 도착 시각)
- **AudioContext 6개 상한에 걸려 무음** — 아님. Chromium 실측 40개 생성해도 안 터짐(옛 제한)

## 곁가지로 확인된 결함 (이번 무음의 주원인은 아님)

- `speakFromAudioCache`(281~308행) — `onerror`와 `play()` 거부가 둘 다 날 수 있는데 가드가 없어
  `onFallback`이 두 번 돈다 → `speakExternal` 두 개가 동시에 뜨고 뒤엣것이 `stopAudioRef`를 덮어써 한 발화 유실
- `playBufferSafe`(803행) — `audioCtx.resume()`을 안 기다리고 벽시계 워치독을 먼저 건다.
  방송 중 앱은 계속 뒤에 있어 suspended가 정상 상태라, 워치독이 재생 전에 `source.stop()`을 때릴 수 있다
- `decodeAudioData`가 던지면 그 `AudioContext`가 안 닫힌다(930·992행) — 자원 누수. 무음은 안 만든다(위 실측)

## 손대기 전에 정해야 할 것

`STALE_MS = 20초`는 **지연을 줄여달라는 요청 때문에** 넣은 값이다.
"안 나옴"과 "안 밀림"이 같은 손잡이라, 이 값을 혼자 올리면 앞서 확정한 체감을 되돌리게 된다.
gspeech 타임아웃을 거는 것은 그와 무관하게 안전하다 — 42초를 기다리는 건 어느 쪽에도 이득이 없다.

---

# 수정 (2026-08-08)

## 무엇을 고쳤나

1. **gspeech 타임아웃 8초** (`ipc.ts` `synthesizeGoogleSpeechV2`)
   `browserFetch`에 `AbortSignal.timeout(8000)`. 다른 TTS 경로와 같은 방식.
   `browserFetch`는 `{...extra}`를 먼저 펼치고 headers만 덮으므로 signal이 살아서 전달된다(확인함).

2. **랜덤 보이스일 때 미리 합성이 살아나게** (`useTTSEngine.ts`)
   - 음성을 **큐에 넣을 때** 굴린다(예전엔 재생 차례). `TTSQueueItem.voiceId` / `.voiceProvider`에 담는다.
   - 합성 캐시 키에 음성 id를 넣는다(`makeSynthCacheKey`). 예전 키는 음성을 안 담아서
     랜덤이면 캐시를 통째로 건너뛰었고, 그래서 **모든 합성이 큐를 잠근 채** 일어났다.
   - 재생 차례엔 담아 둔 음성을 되박는다. 그 사이 provider가 바뀌었으면 버린다.
   - MVP는 사용자별 고정 음성이라 굴리지 않는다(MVP 재생 경로는 자기 설정만 읽는다 — 확인함).

3. **프리페치 캐시가 8칸에서 막히던 것**
   예전엔 `size >= 8`이면 그냥 return. 스테일로 버려진 항목의 결과는 소비되지 않아
   영영 남고, 8칸이 차면 미리 합성이 세션 내내 죽는다(= 원래 증상 재발).
   가장 오래된 것을 버리도록 바꿨다. 죽은 `>20` 분기는 제거. 거부는 `.catch`로 삼켜
   아무도 안 기다리는 사이 unhandled rejection 이 되지 않게 했다.

4. **곁가지 2건**
   - `speakFromAudioCache` — `onerror`와 `play()` 거부가 둘 다 나면 폴백이 두 번 돌아
     `speakExternal`이 동시에 두 개 뜨고 뒤엣것이 `stopAudioRef`를 덮어써 한 발화가 사라진다. once 가드.
   - `playBufferSafe` — `resume()`을 await 하고 워치독 타이머를 `start()` 뒤에 건다.

**재시도는 넣지 않았다.** 실측상 한가할 때만 이득이고 바쁠 때는 손해였다(아래 표).

## 검증

**회귀 292건 전부 통과** (`scratchpad/tts-regress.mjs`, 실제 소스 번들을 그대로 import)
- 오디오 캐시 키가 수정 전 계산과 **완전히 동일** (9 provider × 4 텍스트 × 2 rate = 72건)
- 랜덤 200회: 미리 합성 키 == 재생 키 (= 프리페치가 실제로 쓰임)
- 비랜덤: 음성 안 바뀌고 키도 그대로
- 랜덤이 실제로 5개 음성을 굴림 (`kr_002/003/004`, `gspeech:ko-KR:female/male`)
- provider 바뀌면 굴린 id를 버림

**큐 시뮬레이션** (`scratchpad/tts-sim*.mjs` — 실제 `ttsStore`의 `dequeue`/스테일 드롭을 그대로 쓰고
가상 시계를 물림. 지연은 실측 분포. 씨앗 8개 평균, 120건)

| 채팅 속도 | 지금 | 프리페치+타임아웃 | 항상 재시도 |
|---|---|---|---|
| 여유 5초 | 84% | **94%** | 98% |
| 보통 2.5초 | 53% | **88%** | 85% |
| 폭주 1.2초 | 27% | **46%** | 46% |

주의: 이 숫자는 **미리 합성이 재생 차례 전에 끝난다**는 전제에 기댄다. 1.2초 간격에선
그럴 시간이 없어 이득이 줄어든다(46%). 88%를 기본 기대치로 읽으면 안 된다.

**브라우저 실측** (`scratchpad/tts-browser-test.mjs`, Chromium)
- 깨진 오디오: `onerror` 1회 + `play()` 거부 1회 — **둘 다 남**. 이중 폴백이 실재함을 확인
- suspended 컨텍스트: resume 대기 뒤에 타이머를 걸면 재생 예산이 덜 깎인다(0.141 → 0.211).
  이 테스트는 **무음을 막는다는 증명이 아니다** — 워치독 200ms는 양쪽 다 걸린다.
  실제 `durMs`는 오디오 길이+5초라 훨씬 여유롭다.

**기각한 가설**(증거): 스테일 판정이 틱톡 시각 사용(아님, `Date.now()`) / AudioContext 6개 상한(아님, 40개 OK)

## 안 건드린 것

`STALE_MS = 20초`. 지연 줄이려고 넣은 값이라 손대면 앞서 확정한 체감이 되돌아간다.
원인(락을 오래 잡는 것)을 없앴으므로 이 규칙이 발동할 일 자체가 줄어든다.

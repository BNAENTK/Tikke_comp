# 시맨틀 라이브 게임 — 컨텍스트 노트

작업 중 결정과 근거 기록. 계속 append.

## 목표
TikTok LIVE 방송 중 시청자 채팅을 인식해 꼬맨틀류 단어 유사도 게임에 자동 입력, 호스트와 시청자가 함께 진행. 게임은 한 번에 하나만 활성(호스트 선택).

## 확정된 게임 API 계약 (검증 완료, 2026-06)

### 꼬맨틀 (semantle-ko.newsjel.ly)
- `GET /guess/<gameNumber>/<word>` — 인증 없음.
- 정답: `{"guess":"무궁화","sim":1,"rank":"정답!"}`
- 근접(상위1000): `{"guess":"진달래","sim":0.3216,"rank":14}` — rank = 숫자
- 먼 단어: `{"sim":-0.04,"rank":"1000위 이상"}` — rank = 문자열
- 무효 단어: `{"detail":{"type":"InvalidGuess","description":"처리할 수 없는 입력입니다."}}`
- sim = 코사인 유사도 raw(-1..1). 사이트 표기는 ×100.
- 게임번호: 홈페이지 `(\d[\d,]*)번째 꼬맨틀` 파싱. 자정(KST)마다 +1.
- 모델: FastText (한국어 Common Crawl + Wikipedia).
- `/nearest1k/<n>` — 상위 1000 단어(정답 공개용).

### 꼬오맨틀 (koo.bfy.kr) — 리버스 완료(playwright 캡처)
- **세션형**(랜덤 정답, 무제한 플레이, 오픈소스 jhleekr/semantle-koo).
- `GET /start` → `{"sess":"83c2231b"}` (세션 생성, 랜덤 정답, 12h 만료).
- `GET /guess?word=<word>` + **헤더 `word-sess: <sess>`** → `{"guess","sim","rank"}` (꼬맨틀과 동일 형태).
- `GET /similarity` + word-sess → 임계값.
- gameKey = 세션ID 문자열. start()마다 새 랜덤 정답 → 스트림 리플레이에 적합.

### 뉴맨틀 (newmantle.kr) — 리버스 완료(playwright 캡처)
- **일일단어**(날짜 기반). "개선된 모델" — 유사도 baseline 높음.
- `GET https://api.newmantle.kr/v2/quizzes/<YYYY-MM-DD>/guess/<word>` + 헤더 `x-guest-id:<uuid>`, `Origin: https://www.newmantle.kr`.
- 응답: `{"correct":bool,"rank":-1,"score":59.57,"answer":null}`. **score 이미 ×100**, rank -1=미순위, correct=정답.
- gameKey = 오늘 날짜(KST). startGame은 네트워크 불필요(날짜 계산).

### 인터페이스 일반화 (Phase 2)
- `fetchGameNumber()` → `startGame(): Promise<string>` (gameKey 문자열). 게임번호/세션ID/날짜 통합.
- guess(word, gameKey: string). browserFetch는 `extra.headers` 끝에 병합(line 2007)이라 커스텀 헤더(word-sess/x-guest-id/Origin 오버라이드) 그대로 동작 — browserFetch 수정 불필요.
- 3개 어댑터 실 API 스모크 검증 완료(2026-06): 꼬맨틀 1530, 꼬오맨틀 세션 guess, 뉴맨틀 score 59.57. 전부 정상.

## 아키텍처 결정

- **어댑터 패턴**: 게임 하나 = `GameAdapter` 하나. 각 어댑터가 자기 게임 응답을 `NormalizedGuess`로 파싱.
  - `NormalizedGuess = { word, simRaw, simPct, rank: number|null, isAnswer, invalid }`
- **캐시 우선 파이프라인** (핵심): 채팅 단어 → `(game, dayNumber, word)` 캐시 조회 → HIT 즉시 / MISS만 throttle 큐 → API → 캐시. "모두 같은 오늘 단어" 특성으로 인기 단어 1회만 네트워크. 인디 서버 IP밴 방지의 핵심.
- **throttle**: 캐시 미스만 큐, 수 req/s 상한.
- **이벤트 통합**: `eventBus.subscribe("chat", ...)`. 채팅 흐름 = tiklive.emitEvent → index.ts:130 eventBus.publish → 구독자.
- **외부 fetch**: ipc.ts `browserFetch`(line 1970, private)를 semantle-service.init()에 주입. net.fetch 직접 금지 규칙 준수.
- **브로드캐스트**: top-donor-service 패턴 — `overlayServer.broadcast(msg)` + `cloudOverlayService.broadcastMessage(msg)`.
- **오버레이 2벌 동기화**: desktop `public/overlays/` + worker `public/overlay/` 양쪽 필수.

## 단어 추출 규칙
- 채팅 메시지 trim 후 `^[가-힣]+$`(공백 없는 순수 한글)면 guess. 아니면 무시.
- 무효 단어(InvalidGuess) 응답은 조용히 버림(오타/비단어).

## 정답 감지
- `rank === "정답!"` 또는 `sim === 1` → isAnswer. 게임별로 다를 수 있어 어댑터가 판정.

## 작업 로그 / 발견

- **게임번호 출처 변경**: 정적 HTML에 게임번호 없음(JS 동적 로드). `GET /today` → `{answer_id, 1st_score, 10th_score, 1000th_score}` 사용. answer_id = 게임번호. (1st/10th/1000th 임계값도 여기 있음 — 추후 오버레이에 표시 가능.)
- HTML 정규식 파싱은 "10번째로 유사한 단어" FAQ 문구 오매칭 버그 있어 폐기. 스모크 테스트로 잡음.
- 어댑터 실 API 검증 완료(2026-06): gameNumber=1530, 사랑 sim -4.19, 무효단어 invalid 감지 정상.

## Phase 1 구현 완료 파일
- `electron/services/semantle/types.ts` — GameAdapter, NormalizedGuess, Fetcher
- `electron/services/semantle/komantle-adapter.ts` — 꼬맨틀 어댑터(/today, /guess)
- `electron/services/semantle-service.ts` — 오케스트레이터(캐시/throttle/리더보드/브로드캐스트)
- `electron/main/ipc.ts` — init(browserFetch 주입) + IPC 핸들러 7종
- `electron/preload/index.ts` — semantle API
- `electron/main/index.ts` — onClientConnect 시맨틀 상태 재전송
- `electron/services/overlay-server.ts` — OverlayMessageType "semantle"
- `electron/services/cloud-overlay.ts` — getUrls "시맨틀 게임" → /semantle
- `public/overlays/semantle.html` + `apps/worker/public/overlay/semantle.html` — 오버레이(동기화)
- `src/pages/Semantle.tsx` — 호스트 패널
- `src/components/Sidebar.tsx` + `src/app/App.tsx` — 네비/라우팅

## 미해결 / 다음
- 클라우드 DO 재연결 시 상태 재전송은 worker DO 동작에 의존(확인 안 함). 로컬은 onClientConnect로 재전송 추가됨. 클라우드 오버레이가 게임 도중 로드 시 다음 broadcast까지 빈 보드일 수 있음 — 필요시 worker DO에 semantle last-state 캐시 추가.
- Phase 2: 꼬오맨틀(koo.bfy.kr, FastAPI 경로 미확인) + 뉴맨틀(newmantle.kr, Next.js) API 리버스.
- 임계값(1st/10th/1000th score) 오버레이 표시 — 선택.

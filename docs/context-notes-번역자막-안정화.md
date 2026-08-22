# 컨텍스트 노트 — 번역자막 간헐 고장 수정 (2026-07-30)

## 아키텍처 (조사 결과)
- 라이브 경로: 앱 "자막 시작" → `translationChrome.start()` → Chrome 창이 `translation.html?autostart=1&room=...` 실행 → **이 창이 STT(Web Speech) + gtx 번역 + 로컬 렌더 + WS broadcast 전부 담당** → TTLS 브라우저 소스(같은 HTML, autostart 없음)가 수신 렌더.
- `useTranslationEngine.ts`/`translation-queue.ts`(렌더러)는 설정 페이지 "번역 테스트" 버튼 전용 — 라이브 증상과 무관.
- desktop/worker의 translation.html은 바이트 동일 (md5 확인). 수정 후 복사 필수.

## 증상 → 원인
- **"계속 인식중" 고착**: ①`startSTT`의 `try { recog.start() } catch {}` — 실패 삼키고 재시도 없음 → STT 영구 사망. ②onend조차 안 오는 Chrome 행 → 복구 경로 전무. ③STT 죽으면 반투명 interim 카드가 화면에 영구 잔존(제거 경로가 final 승격/clear뿐) — 송신창·수신창(TTLS) 양쪽 모두. 사용자에겐 "계속 인식중"으로 보임.
- **한 글자(단어)만 번역**: 반응어 하이브리드(`gtrans`)가 "와/아/진짜 + 나머지" 분리 후 나머지 gtx 실패 시 **반응어 사전값만 반환** ("Wow"만 표시). 429 상황에서 빈발.
- **말 씹힘**: ①STT 조용한 사망 구간 발화 통째 유실. ②429 폭주(interim 4틱/초 × 언어수, 동시성 제한 없음) 시 번역 누락 → 원문만 뜨거나, MyMemory 폴백이 타임아웃 없이 행 → `Promise.all`이 물려 final 카드 지연/증발처럼 보임.

## 결정 사항
- **워치독 45초 / 체크 10초**: continuous 모드 침묵 중엔 이벤트 없음이 정상 — 짧으면 침묵 중 불필요 재시작. 침묵 중 재시작은 무해라 보수적으로. `sttActive` 창(송신)에서만 동작.
- **start() 재시도 500ms**: onend 120ms 재시작 갭과 별개. 무한 재시도 허용 (sttActive false면 중단).
- **interim 신선도 12초**: displayTimeoutMs(기본 15초)와 무관한 고정값. interim은 말하는 중에만 존재해야 하는 카드 — 12초 무갱신이면 발화가 끝났는데 final을 못 받은 비정상 상태로 판단. 수신측도 같은 로직으로 자가 정리(송신 창이 죽어도 TTLS 화면 안 오염).
- **반응어 실패 시 전체 재시도**: 반환값 우선순위 = 반응어+나머지 성공 → 전체 문장 gtx/MyMemory 재시도 → ''. ''면 기존 안전망(원문만 표시)이 동작. "Wow"만 뜨는 것보다 원문만 뜨는 게 나음 (K님 증상 신고 기반).
- **429 쿨다운 3초**: 쿨다운 중 interim 번역은 스킵(다음 틱이 따라잡음), final은 MyMemory 직행. gtx 요청량 자체를 줄이는 게 목적. 동시성 큐까지 넣는 건 이 파일 구조(단일 async 함수)에 과함 — 보류.
- **MyMemory 5초 타임아웃**: gtx와 동일 패턴. 이거 없으면 Promise.all 행이 final 자막 전체를 물고 있음.
- **안 고친 것**: desktop 렌더러 translation-queue.ts의 좀비 fetch/재시도 새치기 — 테스트 버튼 전용 경로라 증상 무관. 다음 기회에.
- **translate-overlay(독립 웹앱) 선행 수정**: 같은 증상을 엉뚱한 프로젝트에서 먼저 고쳤음(2026-07-30, 0.1.1 배포). 그쪽 수정은 유효하니 유지. 이 노트가 tikke 정본.

## 검증
- HTML 수정이라 typecheck는 형식 확인용. 실검증 = http.server + 브라우저 실측 (STT는 마이크 필요라 수동, 렌더/타이머 경로는 preview로).

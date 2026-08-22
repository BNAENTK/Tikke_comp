# 전체 코드리뷰 수정 — 결정 사항 (2026-08-20)

앱 전체 코드리뷰(Codex 4파트)에서 나온 26건을 수정한 기록. "기존에 잘 되던 것이
이번 수정으로 깨지면 안 된다"가 최우선 제약이었고, 그래서 몇 건은 원래 지적보다
약하게 고치거나 되돌렸다. 되돌린 이유를 남긴다.

## 되돌리거나 완화한 것

### 1. 관리자 시크릿 — dev 빌드 제한 철회
- 처음: `adminSecret()` 을 `isDevBuild() && env` 로 제한.
- 문제: 관리자(K)는 **설치본**에서도 공지 등록·VIP 관리를 한다. dev 제한은 그 기능을 없앤다.
- 결론: 취약점의 본질은 **하드코딩 fallback 시크릿**(앱만 뜯으면 누구나 사용)이었지
  env 사용이 아니다. env 전용으로만 남겼다. 일반 사용자 PC 에는 env 가 없어 노출 없음.
- 검증: dev 실행 중 `tikke.vip.list()` → `OK emails=2`. .env 에서 로드됨 확인.

### 2. 워커 rate limit — `await put` 철회, `ctx.waitUntil` 복구
- 처음: 원자성을 위해 KV put 을 await.
- 문제 2가지. (a) KV 는 **같은 키에 초당 1회** 넘게 쓰면 실패한다. rateKey 는
  `ip:분버킷` 이라 초당 여러 요청이면 바로 걸리고, await 한 실패는 미처리 예외 →
  정상 요청이 500. 이 put 은 top-level try 밖에 있었다. (b) 매 요청에 KV 쓰기 지연.
- 결론: KV 로는 어차피 원자적 카운팅이 불가능(읽기-증가-쓰기 경합). 응답 뒤 저장 유지 +
  `.catch(() => {})`. 정확한 제한이 필요하면 Durable Object / CF Rate Limiting 바인딩.

### 3. /vip/check — 구버전 앱 폴백 유지
- 처음: 인증 필수(본인 것만). 남의 이메일 등록 여부 열거 차단이 목적.
- 문제: 이미 설치된 앱들은 `?email=` 로만 호출한다. 인증 필수로 바꾸면 **기존 사용자
  전원의 VIP 가 일반으로 떨어진다**.
- 결론: Authorization 헤더가 있으면 토큰 기준(본인), 없으면 예전 쿼리 방식. 구버전
  사용률이 충분히 낮아지면 폴백 제거(코드에 TODO).
- **배포 순서 강제**: 데스크탑은 이제 email 쿼리를 안 보낸다 → 워커를 **먼저** 배포해야
  한다. 앱을 먼저 릴리즈하면 그 사이 VIP 가 전부 false 가 된다.

### 4. 오디오 확장자 화이트리스트 확대
- `tikke-sound://` 가 아무 로컬 파일이나 읽던 문제를 확장자 화이트리스트로 막았는데,
  파일 선택창에 "모든 파일" 옵션이 있어 aac/opus/webm/mp4 등을 쓰던 사용자가 있을 수 있다.
- 오디오 계열은 넓게 허용(aac·opus·oga·weba·webm·mp4·m4b·wma·aif·aiff 추가).
  차단 목적은 설정·토큰 같은 **비오디오** 파일 읽기 방지이므로 목적은 유지된다.
- 검증: probe.wav → `LOADED dur=0.10`, probe.txt → `ERROR code=4`(403).

### 5. tiklive 선물 집계 — normalized null 폴백
- 세션 집계를 정규화된 event 기준으로 바꿨는데(streak 누적값 이중계상 수정),
  정규화 **전에** 예외가 나면 `normalized` 가 null 이라 `ge.user` 에서 TypeError.
  이 블록은 try 밖이라 핸들러가 통째로 죽는다. Codex 도 같은 지점을 지적.
- 결론: `normalized ?? {}` + streak 판정도 normalized 없으면 예전(raw) 기준으로.

### 6. App.tsx 세대 카운터 — loading 해제는 finally 로
- profile stale race 를 세대 카운터로 막았는데, 세대 불일치 조기 return 이
  `setLoading(false)` 를 건너뛴다. `onSession` 은 setLoading 을 부르지 않으므로
  **로딩 화면에 영구히 갇힐 수 있다**. `.finally()` 로 옮겼다.

## 그대로 둔 것 (검토했으나 위험 없음)
- `await startSession(clean)` — startSession/endSession 모두 내부 try/catch 완비.
  doConnect 로 예외가 새지 않는다.
- `useSoundPlayer` stopGeneration — `stopAll()` 호출자는 SoundLibrary 의 사용자
  "전체 정지" 하나뿐이라 정상 재생을 삼키지 않는다. 언마운트 AudioContext close 도
  `getCtx()` 가 closed 면 재생성하므로 복구된다(훅은 App 에 1회 마운트).

## 검증 (실측)
- `pnpm typecheck` 전체 통과, `@tikke/desktop build` 성공.
- CDP 콘솔 감사: 앱 요청 4xx 0건 / 콘솔 에러 0건.
  (감사에 뜬 403 1건은 Log.enable 버퍼 재생 — 내 probe.txt 테스트 흔적.)
- 사운드: `AudioBufferSourceNode.start` 계측 → 재생 1회, 페이지 이동 후 1회,
  stopAll 직후 재생 1회. 삼킴/불능 없음.
- 설정 IPC: 정상 키 저장·조회 OK, `auth_access_token` 차단, `getAll` 375키에 토큰 없음.
- 관리자: `vip.list()` 정상. 선물 하네스 5/5 매치.

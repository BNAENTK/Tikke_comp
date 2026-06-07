# 웹 컴패니언 구현 컨텍스트

## 아키텍처 결정

### 기존 인프라 재활용
- 데스크탑은 TikTok 이벤트를 `cloudOverlayService.broadcast(event)`로 이미 전송 중
- Worker Durable Object(OverlayRoom)가 WebSocket으로 브로드캐스트
- WS 엔드포인트: `wss://api.tikke.kr/overlay/rooms/{roomKey}/ws`
- 별도 Supabase Realtime 불필요 — 기존 OverlayRoom으로 충분

### TTS 방식
- **데스크탑 변경 없음** — 컴패니언이 이벤트 수신 후 독립적으로 TTS 판단
- 브라우저에서 `new Audio(googleTTSUrl).play()` 사용
- `<audio>` 엘리먼트는 CORS preflight 없이 크로스오리진 리소스 로드 가능
- Google Speech API v2: `https://www.google.com/speech-api/v2/synthesize?key=...`
- 공개 API key: `.env`의 `GOOGLE_TTS_API_KEY`로 주입 (TikFinity와 동일 계열 공개 키). 키 값은 소스·문서에 기록하지 않음.

### TikFinity와의 비교
- TikFinity: Electron 렌더러가 `https://tikfinity.zerody.one` origin → Google 직접 신뢰
- Tikke 데스크탑: `file://`/`app://` origin → 신뢰 없음 → IPC 경유 필요
- **웹 컴패니언**: `https://tikke.kr/companion` origin → Google 직접 신뢰 → TikFinity와 동등

### 이벤트 포맷
- WebSocket 메시지: `{ type: "events", events: TikkeEvent[] }`
- TikkeEvent.type: chat | gift | like | member | follow | share | subscribe | ...
- GiftEvent: { giftName, repeatCount, diamondCount, isStreakEnd, user }

### TTS 큐 설계
- ttsQueueRef: 대기 텍스트 배열
- 앞 오디오 종료 시 다음 텍스트 자동 재생
- 동시 2개 재생 방지

### UI 설계
- 다크 테마 (Tikke 디자인 시스템과 일치)
- 좌측: 채팅 피드 (최근 50개)
- 우측: 이벤트 알림 + 통계
- 상단: 룸키 입력 + 연결 상태 + TTS 토글

## 파일 구조
```
apps/web/companion/
  index.html   — babel-standalone HTML 쉘
  app.jsx      — React 컴패니언 앱
```

## 룸키 연동
- 데스크탑 설정 → 오버레이 → 클라우드 오버레이 룸키 확인
- URL 파라미터 ?room=xxx 로 바로 연결 지원
- 룸키는 12자 영소문자+숫자

## 주의사항
- apps/web는 raw 배포 (dist 배포 금지, babel-standalone 브라우저 컴파일)
- React 18 UMD CDN 사용
- 글로벌 React/ReactDOM window 변수 사용

---

# 팬 알림 피드 오버레이 (fan-alert) — 2026-06-01

## 배경
호스트가 큰 선물·멤버 레벨업·슈퍼팬 구독을 방송 화면에 수동 캡쳐 이미지로 올리던 걸 자동 오버레이로 대체. 디자인 스펙 = 사용자가 보낸 이미지 2장(보라핑크 그라데이션 피드, 노란 닉네임).

## 디자인 결정
- **디자인 출처**: 사용자 이미지 = TikTok 라이브룸 채팅 피드 시스템 메시지 스타일. TTLS 데스크탑 앱 자체 토스트(`interactive-notice`, 다크그레이)와는 다른 화면. TTLS 로컬 파일엔 이 디자인 없음(서버 렌더). → 이미지대로 구현.
- **구조 레퍼런스**: `chat.html`의 세로 스택 피드(flex column, slideIn, maxMsgs, fade-out, WS+postMessage). big-spender의 풀스크린 takeover 아님.

## 감지 (확인된 사실 — tikke 코드 직접 검증)
- 🟢 gift: `gift` 이벤트, `eventBus.subscribe("*")` → cloudOverlay 브로드캐스트
- 🟢 슈퍼팬: `subscribe` 이벤트 정상
- 🟡 멤버 레벨업: 전용 이벤트 없음. `tiktok-live-connector`의 `WebcastMemberMessage` proto는 `actionId`만 디코드(레벨 숫자 X). `User.badges`(id 64)는 디코드되나 simplify에서 가공 안 됨. → `tiklive.ts`에 member raw 1회 로그 추가, 라이브에서 badge에 팬클럽 레벨 있는지 확인 후 `fan_levelup` emit 구현 예정.
- **TTLS main.js protobuf 발굴은 dead end** — tikke는 다른 라이브러리라 스키마 이식 불가.

## 파일
- `apps/desktop/public/overlays/fan-alert.html` + `apps/worker/public/overlay/fan-alert.html` (양쪽 동기화 필수)
- URL 파라미터: room, scale, max, duration, pos(top/bottom), giftIds(CSV 필터), gift/sub/lvl/avatar(0=숨김), preview/test
- HTML 이벤트 타입: `gift`(giftIds 필터) / `subscribe` / `fan_levelup`
- config 키: fanAlertGiftIds(GiftRef[] JSON), fanAlertMax, fanAlertDuration, fanAlertPos, fanAlertGift/Sub/Lvl/Avatar (default-settings.ts 미등록 — 컴포넌트 `??` 폴백, big-spender 선례 동일)

## 미해결
- 레벨업 "도달 순간" 완전 재현은 라이브 raw 바이트 확인 필요. 현재는 HTML 표시 로직 + test만 구현, main emit은 badge 확인 후.
- 미배포 (사용자 "배포해" 대기)

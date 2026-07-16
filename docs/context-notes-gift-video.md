# 선물 영상 규칙 — 컨텍스트 노트

## 요구사항 (2026-07-08)
- 사용자가 선물을 설정하고, 시청자가 그 선물을 보내면 사용자가 설정한 영상이 오버레이에 재생.
- 특정 시청자 아이디도 조건으로 걸 수 있음 — 그 시청자가 보낼 때만 재생. 아이디 비우면 전체 허용.

## 설계 결정
- **새 오버레이 안 만듦** — 기존 `video.html`(desktop+worker 양쪽 이미 존재, cloudUrl "영상" 등록됨)이 `{type:'video', url, durationMs}` 메시지를 그대로 재생. 풀체인 이미 완성돼 있음.
- **새 규칙 엔진 안 만듦** — 기존 `overlay-rules-service.ts`(트리거→마퀴/불꽃/벌칙/사운드/TTS)에 `overlayType:"video"` 추가. UI도 기존 OverlaySettings 규칙 폼 확장.
- **condition.senderId** — 이벤트 `user.uniqueId`와 비교. 정규화: 앞 `@` 제거 + 소문자. 모든 규칙 타입에 공통 적용 (엔진 레벨).
- **영상 파일**: `userData/gift-videos/`에 복사 저장. 메타는 settings JSON `giftVideoFiles` `[{id,name,file}]` — sql.js 테이블 대신 settings로 (파일 수 적음, 마이그레이션 불필요).
- **서빙**: overlay-server(18181)에 `/media/gift-videos/<file>` 라우트. `createReadStream` + Range 지원 (mp4 시킹/Chromium 요구). 파일명 화이트리스트로 경로 탈출 차단.
- **URL**: broadcast 시 `http://localhost:18181/media/gift-videos/<file>`. 클라우드 오버레이(https)도 localhost는 mixed-content 예외라 로드됨. TTLS = 같은 PC라 성립.
- **스트릭**: video 규칙 config `streakMode` — `"every"`(기본, 미설정 포함): 연타 중에도 매번 재생(새 이벤트가 기존 재생을 즉시 교체 — video.html이 새 video 메시지에서 clear 후 재생). `"final"`: 연타 중간 스킵, 끝나면 1회. 기본을 every로 바꾼 건 사용자 지시(2026-07-08). 기존 타입 동작 불변.
- durationMs 미설정 시 영상 끝까지 재생 (video.html ended 이벤트가 정리).

## 주의
- 다른 PC에서 클라우드 오버레이 열면 localhost 영상 로드 안 됨 — TTLS와 tikke가 같은 PC라는 전제. UI 힌트로 명시.

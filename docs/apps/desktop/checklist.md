# Tikke 웹 컴패니언 구현 체크리스트

## 목표
Tikke 데스크탑과 연동하는 웹 컴패니언 앱 구현.
실제 작동하는 기능만. apps/web/companion/ 경로.

## 기능 범위
- 실시간 이벤트 피드 (채팅/선물/팔로우 등) — WebSocket 기반, 기존 인프라 그대로 사용
- 브라우저 TTS — Google Speech v2 직접 호출 (CORS 없는 audio 엘리먼트), 데스크탑 변경 불필요
- 선물 통계 — 세션 누적 다이아, 선물 수, 팔로우 수
- 룸키 입력/북마크 (?room=xxx 파라미터)

## 작업 목록

- [x] checklist.md + context-notes.md 작성
- [x] apps/web/companion/index.html 작성
- [x] apps/web/companion/app.jsx 작성 (stale closure 방지 ref 패턴 적용)
- [x] apps/web/_headers — companion은 별도 헤더 불필요 (WS는 api.tikke.kr, TTS는 audio 엘리먼트)
- [ ] 동작 검증: 룸키 입력 → WebSocket 연결 → 이벤트 수신 시나리오 확인 (Pages 배포 후 실제 라이브에서 확인)

## 검증 기준
1. ?room=룸키 로 접속 시 자동 연결
2. 데스크탑 방송 시작 후 채팅/선물 이벤트 실시간 표시
3. 선물 이벤트 시 Google TTS 재생 (브라우저 직접)
4. 연결 끊김 시 자동 재연결

---

# 팬 알림 피드 오버레이 (fan-alert) — 2026-06-01

- [x] `apps/desktop/public/overlays/fan-alert.html` 생성 (보라핑크 그라데이션 피드, gift/subscribe/fan_levelup 처리)
- [x] `apps/worker/public/overlay/fan-alert.html` 동기화 복사
- [x] `cloud-overlay.ts` getUrls()에 "팬 알림" 매핑 추가
- [x] `EventOverlay.tsx`: OVERLAY_CARD + ID_TO_LABEL + cloudUrl/previewUrl 빌더 + 설정 패널(GiftPicker 다중선택)
- [x] `tiklive.ts` member raw 1회 로그 (레벨업 badge 진단용)
- [x] `pnpm build` 통과 (내 코드 타입에러 0, 기존 빚 15개는 별개)
- [ ] playwright로 fan-alert.html?test=1 시각 검증
- [ ] 실제 라이브 연결 → member-raw 로그로 레벨업 badge 확인 → 레벨업 emit 구현
- [ ] 미배포 (사용자 "배포해" 대기)

## fan-alert 검증 기준
1. EventOverlay 페이지에 "팬 알림" 카드 노출, GiftPicker로 선물 다중선택/칩 삭제
2. 미리보기 iframe에서 선물/슈퍼팬/레벨업 3종 데모 표시
3. 트리거 선물 선택 시 cloudUrl에 giftIds CSV 포함, 미선택 시 모든 선물 반응
4. 토글(선물/슈퍼팬/레벨업/아바타) + max/duration/pos 반영

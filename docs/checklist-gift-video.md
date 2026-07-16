# 선물 영상 규칙 체크리스트

사용자가 선물(+선택적으로 특정 시청자 아이디)을 설정하면, 그 조건에 맞는 선물이 올 때 등록한 영상을 오버레이로 재생.

- [x] overlay-server: `/media/gift-videos/<file>` 서빙 (userData, Range 지원, mp4/webm)
- [x] overlay-rules-service: condition에 `senderId` 추가 (보낸 시청자 uniqueId 매칭, @ 무시/대소문자 무시)
- [x] overlay-rules-service: overlayType `"video"` + config `videoFile`/`videoDurationMs` + 트리거(video 메시지 broadcast)
- [x] overlay-rules-service: video 규칙은 선물 스트릭 중간 이벤트 스킵 (isStreakEnd===false)
- [x] ipc: `tikke:videoFx:list/add/delete` (파일 대화상자 → userData/gift-videos 복사, 200MB 제한)
- [x] preload: `videoFx` API 노출
- [x] OverlaySettings: 액션 타입 "🎬 영상 재생" + 영상 업로드/선택 UI + 시청자 아이디 조건 입력
- [x] OverlaySettings: 규칙 목록에 video 라벨/요약(원본 파일명) + 선물/시청자 조건 칩 표시
- [x] typecheck (desktop)
- [ ] 실측: 앱 실행 → 규칙 추가 → 테스트 선물 이벤트(harness:gift) → 영상 오버레이 재생 확인

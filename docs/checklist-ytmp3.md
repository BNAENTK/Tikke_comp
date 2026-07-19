# 유튜브 음원 추출 (ytmp3) 체크리스트

- [x] ytmp3-service.ts — yt-dlp/ffmpeg 다운로드 + 추출 spawn + 진행률 파싱
- [x] ipc.ts — check/install/extract/cancel 핸들러 + 로그/진행률 이벤트
- [x] preload — tikke.ytmp3 API 노출
- [x] YtMp3.tsx 페이지 — URL 입력/설치/진행률/결과 UI
- [x] Sidebar + App.tsx 등록 (SidebarPage 타입 포함)
- [x] pnpm --filter @tikke/desktop typecheck 통과
- [x] 구간 잘라내기 편집기 — 재생 슬라이더 + 시작/끝 핸들 드래그 + ffmpeg trim
- [ ] 실기기 테스트 — 설치 다운로드 + 실제 URL 추출 + 잘라내기 (K님 확인 필요)

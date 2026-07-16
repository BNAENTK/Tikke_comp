# 온라인 사용자 목록 체크리스트

- [x] worker /ping — presence KV 기록 (tiktokId 있을 때, TTL 720초)
- [x] worker /ping — X-Tikke-Heartbeat 헤더면 Discord 알림 스킵
- [x] worker /admin/online GET — x-admin-secret 인증 + presence 목록 반환
- [x] desktop ping.ts — startPingHeartbeat() 5분 주기, 연결 중일 때만 전송
- [x] desktop index.ts — whenReady에서 하트비트 시작
- [x] web admin-online.html — 로그인 + 테이블 + 30초 자동 새로고침
- [x] 검증: worker typecheck + desktop typecheck 통과
- [ ] 배포 (사용자 지시 대기 — worker push 자동배포 + desktop 버전업 + Pages는 별도 지시)

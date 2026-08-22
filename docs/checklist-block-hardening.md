# 차단 강화 — 체크리스트

목표: 이메일만 막던 차단을 **틱톡 아이디·IP**까지 넓히고, 접속 기록을 남겨 다음부터는 IP를 직접 조회할 수 있게 한다.

## 1. 접속 기록 남기기 (가장 시급 — 지금은 어디에도 안 남는다)
- [x] Supabase `public.connect_logs` 테이블 (user_id, email, tiktok_id, ip, country, platform, created_at)
- [x] RLS: 서비스 롤 전용 (일반 사용자 조회 불가)
- [x] worker `/ping` 에서 **알림 발사 시점에만** 기록 (하트비트는 제외 — 5분마다 쌓이면 폭증)
- [ ] 검증: 실제 /ping 후 행이 쌓이는지 — **배포 후에만 가능**

## 2. 틱톡 아이디 차단 (주력)
- [x] KV `blockedTiktokIds`
- [x] `/ping` 에서 `X-Tikke-Tiktok-Id` 검사 → 403
- [x] `requireAuth` 에서도 헤더 검사 → 워커 API 전반 차단
- [x] 검증: 판정 함수·관리 라우트는 스텁으로 확인 / 실제 HTTP 403 은 배포 후

## 3. IP 차단 (보조)
- [x] KV `blockedIps` — **`blocked:{ip}` 키는 쓰면 안 된다**(rate limiter 가 1시간 TTL 로 덮어씀)
- [x] 최상단 게이트에서 검사 → 403
- [x] 검증: 판정 함수·관리 라우트는 스텁으로 확인 / 실제 HTTP 403 은 배포 후

## 4. 관리 엔드포인트 (ADMIN_SECRET 전용)
- [x] `GET /blocked/list` → `{emails, tiktokIds, ips}` 로 확장 (기존 `emails` 키 유지 = 하위호환)
- [x] `POST/DELETE /blocked` body 에 `tiktokId`, `ip` 지원 (기존 `email` 그대로)
- [x] `GET /blocked/lookup?email=|ip=|tiktokId=` → 접속 기록 역추적
- [x] 검증: 각 메서드 실제 호출

## 5. 마무리
- [x] `pnpm --filter @tikke/worker typecheck`
- [x] context-notes 작성
- [ ] 커밋 (배포는 K 승인 후)

## 안 하는 것
- 기존 `blocked:{ip}`(rate limit) 로직 변경 — 건드리면 속도 제한이 깨진다.
- 공개 check 엔드포인트 — 차단 대상이 자기 상태를 조회하면 우회 시점을 알게 된다(기존 결정 유지).

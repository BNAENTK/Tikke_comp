# 온라인 사용자 목록 — 컨텍스트 노트

날짜: 2026-07-10. 목적: 현재 틱톡에 연결해 앱 사용 중인 사용자 목록을 어드민에서 확인.

## 구조 결정

- **presence 저장소 = TIKKE_SETTINGS KV**, 키 `presence:{userId}`, TTL 720초.
  하트비트 5분 주기 × 2회 놓치면 자동 소멸 = 별도 오프라인 처리 로직 불필요.
  Durable Object는 이 규모에 과함. KV eventually consistent라 수십 초 지연 허용.
- **하트비트는 기존 /ping 재사용** — 새 endpoint 대신 `X-Tikke-Heartbeat: 1` 헤더로 구분.
  이유: /ping의 Discord 알림 dedup 키가 30분 TTL이라, 하트비트가 알림 로직을 타면
  연결 유지 중 30분마다 TIKTOK_CONNECTED 알림이 반복 발사됨. 헤더 있으면 presence만
  갱신하고 즉시 return.
- **desktop**: `ping.ts`에 `startPingHeartbeat()` — 항상 도는 5분 interval이지만
  `tikLiveService.getUsername()`이 truthy(=연결 중)일 때만 실제 전송. 연결/해제 시점에
  start/stop 배선하는 것보다 단순하고 race 없음. index.ts whenReady에서 1회 시작.
- **어드민 UI**: `apps/web/admin-online.html` 단독 정적 페이지(React 불필요, vanilla).
  sessionStorage 키 `tikke_admin_secret_v1`을 admin.html과 공용 — 어드민에 이미
  로그인했으면 재입력 불필요. 30초 자동 갱신. 틱톡 아이디는 라이브 링크.

## 한계 / 주의

- 하트비트는 앱 코드라 **v1.8.154+ 사용자부터 정확**. 구버전은 연결 순간 1회 ping만
  → 12분 뒤 목록에서 사라짐(실제로는 계속 사용 중일 수 있음).
- `/admin/online`은 KV list(prefix) 1회 호출 — 키 1000개까지 한 페이지. 현재 규모 충분.
- 비로그인(토큰 없는) 사용자는 userId가 IP로 잡힘(기존 /ping 로직 그대로).

## 검증

- `pnpm --filter @tikke/worker typecheck` / `--filter @tikke/desktop typecheck` 통과.
- 배포 후 실검증 경로: 앱에서 틱톡 연결 → https://tikke.kr/admin-online.html 목록 확인.

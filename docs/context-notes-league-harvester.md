# 리그 조각컷 능동 수집기 (League Cutoff Harvester) — 설계 + 체크리스트

## 배경 / 문제
- 현재 컷오프 수집은 `tiklive.ts`가 **사용자가 연결된 방 1개**만 3분 주기로 수집(`collectLeagueCutoff`). 앱 닫거나 방송 끝나면 정지 → 워커 30분 cron은 죽은 풀 방을 찔러 gap 없음 → **값이 마지막 수집 시각(예: 새벽 1시)에 얼어붙음.**
- TikDo는 18리그(A1~A3, B1~5, C1~5, D1~5) 전부 실시간. tikke는 구조(리그·퍼센타일)는 동일하나 **stale + 단발 쓰레기값**.
- serve 측 쓰레기값은 이미 수정됨(`cutoffs.ts` 퍼센타일별 MAX + 단조성, 커밋 76b7037). **남은 건 freshness** = 이 문서.

## 검증된 프리미티브
- **시드**: `user_league_data`(Supabase)에 `unique_id`(@핸들) + `user_id` + `class_type` + `score` 누적. 저리그 넉넉(400=24), **고리그 thin(1800=2, 1500=1, 800=1)**.
- **발견**: `room-observer.ts::fetchRoomSnapshot(uniqueId)` → `fetchIsLive()`(line103) + api-live에서 `roomId`(line124). anchorId = user_league_data.user_id. → uniqueId→(live, roomId, anchorId) 해석 가능.
- **수집**: `league-cutoff-collector.ts::collectLeagueCutoff(sessionId, roomId, anchorId, displayId)` 그대로 재사용(additive — 같은 upsert→serve max-pick 경유, 쓰레기 재유입 불가).

## 아키텍처 (호스트 무관 단일 모듈)
신규 `apps/desktop/electron/services/league-cutoff-harvester.ts` — 데스크탑(2)·상시호스트(3) 공용 로직.

상태: `Map<class_type, { room: {roomId,anchorId,uniqueId}|null, liveCheckedAt:number }>`

**2개 루프 (레이트 분리 — TikTok bot/429 방지, room-observer 위생 재사용: 호출 간 500ms+, 에러 dedupe, 백오프):**
1. **발견 루프** (느림, TTL 기반): 리그가 known-live 방 없거나 TTL(20~30분) 만료 시에만. 해당 리그 시드(user_league_data top score, fresh)를 순회 → `fetchRoomSnapshot` → **첫 라이브에서 중단**(전수 탐색 금지). room+anchor 확정 저장.
2. **폴링 루프** (3~5분): known-live 방마다 `collectLeagueCutoff`. 실패(방 죽음) → room=null → 다음 발견 루프가 재탐색.

**시드 자가치유(2b, ranks 수확)**: `collectLeagueCutoff` 성공 시 `rank_view.ranks[].display_id`를 `user_league_data` 시드로 upsert → 라이브 앵커 1명이 그 리그 top-N 시드 부트스트랩 → thin 고리그 점진 해결.
- ⚠️ 선결 검증: rank_type=16 응답의 `rank_view.ranks`에 `display_id` 실재하는지 확인(rank_type=8과 동일 shape 추정, 미확인). admin secret 필요한 `/league/cutoff-test`로 K가 확인하거나 런타임 로그.

## 정직한 한계
- 그 순간 **라이브 앵커가 0인 리그는 여전히 못 받음.** 시간이 지나며 수렴(자가치유)이지 day-1 동급 아님.
- sign-api 429(EulerStream) 리스크 — tiktokSignApiKey 설정 필요([[reference_tikke_sign_api]]).

## 단계
- **(2) 데스크탑 내장**: 모듈 + 앱 시작 시 기동. 앱 열려있는 동안 18리그 fresh. ← 1차
- **(3) 상시 호스트**: 같은 모듈 헤드리스 진입점 + 호스트 + secret(sessionId·signKey, 하드코딩 금지). Node 런타임 필요(Worker 불가). **후보: Tikketone 상시 서버**([[project_tikketone]]). ← 호스트 결정 대기

## 체크리스트
- [x] harvester 모듈 작성 (발견/폴링 2루프, 상태, 위생) — `league-cutoff-harvester.ts`
- [x] Supabase 시드 fetch 헬퍼 (user_league_data, 점수순 리그당 6개)
- [x] fetchRoomSnapshot → roomId 확보 (anchorId=seed.user_id)
- [x] collectLeagueCutoff 반환 boolean화 (방 죽음 감지)
- [x] 데스크탑 startup 기동 (sessionId 있을 때) — main/index.ts
- [x] typecheck (desktop 통과)
- [ ] **K 검증**: 내가 연결 안 한 고리그(1800~2000)에 수 분 내 fresh row 생기는지 + TikDo와 값 2~3개 대조 (라이브 필요 — 헤드리스 불가)
- [x] (2b) ranks 수확 — `harvestSeeds` in league-cutoff-collector.ts, ignore-duplicates(기존 안 덮음). ⚠️ rank_view.ranks display_id 실재는 라이브 로그로 확인 필요(방어적 파싱이라 없어도 무해)
- [x] (3-lite) 워커 cron 강건화 — 풀 후보 3→8, 풀 상한 5→8. harvester 발견 방을 워커가 24/7 headless(md5 서명) 폴링
- [ ] (3-full) 데스크탑 완전 독립 24/7 — harvester가 Electron 의존(net/getSetting/captureRankRaw=히든브라우저). 비-Electron 호스트엔 (a)Electron-on-server 또는 (b)워커에 api-live 발견 포팅 필요. **아키텍처 결정 대기.**

## 진행 메모 (2026-06-27)
2(데스크탑) 핵심 완성·타입통과. 앱 켜져있는 동안 18리그 능동 수집. 다음: K 라이브 검증 → 2b(ranks 자가치유) → 3(호스트 결정: Tikketone 서버 후보).
파라미터: 발견 TTL 25분, 폴링 4분, 호출간격 700ms, 리그당 시드 6개.

## 검증 기준 (헤드리스 테스트 불가 — 라이브 TikTok)
"내가 안 보는 리그에 fresh 값" = 원 불만 직결 성공 기준. typecheck로는 불충분.

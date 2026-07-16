# 리그 조각컷 능동 수집기 (League Cutoff Harvester) — 설계 + 체크리스트

## 배경 / 문제
- 현재 컷오프 수집은 `tiklive.ts`가 **사용자가 연결된 방 1개**만 3분 주기로 수집(`collectLeagueCutoff`). 앱 닫거나 방송 끝나면 정지 → 워커 30분 cron은 죽은 풀 방을 찔러 gap 없음 → **값이 마지막 수집 시각(예: 새벽 1시)에 얼어붙음.**
- 목표는 18리그(A1~A3, B1~5, C1~5, D1~5) 전부 실시간. 현재 tikke는 구조(리그·퍼센타일)는 갖췄으나 **stale + 단발 쓰레기값**.
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
- [ ] **K 검증**: 내가 연결 안 한 고리그(1800~2000)에 수 분 내 fresh row 생기는지 (라이브 필요 — 헤드리스 불가)
- [x] (2b) ranks 수확 — `harvestSeeds` in league-cutoff-collector.ts, ignore-duplicates(기존 안 덮음). ⚠️ rank_view.ranks display_id 실재는 라이브 로그로 확인 필요(방어적 파싱이라 없어도 무해)
- [x] (3-lite) 워커 cron 강건화 — 풀 후보 3→8, 풀 상한 5→8. harvester 발견 방을 워커가 24/7 headless(md5 서명) 폴링
- [ ] (3-full) 데스크탑 완전 독립 24/7 — harvester가 Electron 의존(net/getSetting/captureRankRaw=히든브라우저). 비-Electron 호스트엔 (a)Electron-on-server 또는 (b)워커에 api-live 발견 포팅 필요. **아키텍처 결정 대기.**

## 진행 메모 (2026-06-27)
2(데스크탑) 핵심 완성·타입통과. 앱 켜져있는 동안 18리그 능동 수집. 다음: K 라이브 검증 → 2b(ranks 자가치유) → 3(호스트 결정: Tikketone 서버 후보).
파라미터: 발견 TTL 25분, 폴링 4분, 호출간격 700ms, 리그당 시드 6개.

## 검증 기준 (헤드리스 테스트 불가 — 라이브 TikTok)
"내가 안 보는 리그에 fresh 값" = 원 불만 직결 성공 기준. typecheck로는 불충분.

## 진행 메모 (2026-07-06) — 퍼센타일 커버리지 개선
- **방향**: 외부 미러는 배제(원천이 어차피 동일 TikTok rank_type=16이라 자체수집으로 충분). 자체수집 강화에 집중.
- **근본 병목 발견**: 앵커 1명 = 퍼센타일 1개(owner_rank.gap의 targetPercentile, 앵커 위치가 결정). 기존 harvester는 리그당 **점수 top-6**만 시드+**첫 라이브 1명서 중단** → 전부 최상위 → **1%만 수집, 20/60/70/80 영영 못 잡음.**
- **수정(harvester 단일 파일)**: ①`stratifiedSample` — class별 점수 분포 전체서 균등 추출(상위~하위) → 다양한 순위. ②`SEEDS_PER_LEAGUE` 6→12. ③리그당 라이브 앵커 1→`LIVE_ANCHORS_PER_LEAGUE`=4 유지(`live[ct]`=배열). ④`discoverLeague` 최대 4명 수집, `runPoll` 전부 폴링·생존만 유지.
- **워커 자동 혜택**: `reportRoomToWorker`가 다양한 방 등록 → 워커 30분 cron(항상 켜짐, 풀 8/class)도 다양한 퍼센타일 수집. 워커 코드 무변경.
- 부하: 18리그×4=72콜/4분, 700ms 간격 = ~50초/사이클. typecheck 통과, stratifiedSample 단위검증(100명→0,9,…,99 균등).
- **K 검증 대기(라이브 필요)**: 한 리그에서 여러 퍼센타일 행(1/10/20/60 등)이 수 분 내 생기는지 Supabase에서 확인.
파라미터 갱신: 시드 6→12, 리그당 라이브앵커 1→4.

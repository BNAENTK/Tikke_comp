# context-notes — 리그 순위 리스트 (외부 대조 사이트 대응)

## 2026-08-03

### 배경
경쟁 사이트 외부 대조 사이트이 리그별(A1~D5) 순위 100명을 공개한다. 분석 결과(메모리 [[reference-외부 대조 사이트-ranking]]) 외부 대조 사이트도 tikke와 **동일 소스(TTLS ranklist rank_type=16)·동일 스키마**. K 지시: tikke 자체 수집으로 순위 기능 추가, 웹 + 앱 페이지 둘 다.

### 핵심 발견 — 데이터는 이미 있다
tikke worker `rank-cron.ts saveLeagueRanks()`가 **`broadcaster_rank_snapshots`** 테이블에 리그별 순위를 이미 저장 중. 컬럼: `broadcaster_unique_id, display_id, nickname, rank, score, snapshot_date, period_end, class_type, updated_at, room_id, region`.

Supabase 실측(2026-08-03): 8/2 기준 18개 리그(class_type 200~2000) 전부 region=kr 순위 데이터 존재(리그당 99+행). 오늘분은 자정 수집 진행에 따라 점증.

→ **데이터 수집은 완성. 빠진 건 서빙 라우트 + UI뿐.**

### 주의 — rank 중복
`(broadcaster_unique_id, snapshot_date)` 유니크라 방송인은 유니크지만, 여러 cron 수집 시점이 겹쳐 **같은 rank 값이 여러 방송인에 붙는다**(예: A1 kr 220행, max_rank 99). 서빙 시 dedup 필요 — broadcaster별 최신(updated_at) 1행, rank asc + score desc 정렬 후 상위 100.

### 설계
mirror = `apps/worker/src/routes/cutoffs.ts` 패턴.

1. **worker 라우트 `/league/rankings`** — `broadcaster_rank_snapshots`에서 class_type별 순위 조회. 오늘분 없으면 어제 폴백(cutoffs와 동일). region=kr. Cloudflare `caches.default` 캐시(버전 키, 180s) + stale 폴백 + CORS. 응답 `Record<class_type, {tier, date, rows:[{rank,score,nickname,display_id}]}>`.
   - dedup: Supabase REST로 전부 받아 worker에서 broadcaster별 최신 1행 + rank 정렬. (Supabase REST distinct 제약 → worker 메모리 dedup)
2. **웹 `apps/web`** — 리그 순위 페이지. `LeagueCutoffs.tsx`의 GROUPS/tier 스타일 + fetch+poll 미러. 리그 탭(A1~D5) + 순위 테이블(순위/닉네임/점수). 조각 컷은 기존 cutoffs와 별개 표시 or 통합.
3. **앱 `apps/desktop`** — 순위 페이지 + Sidebar 항목. 웹과 동일 데이터(api.tikke.kr/league/rankings).

### TIER_BY_CLASS 공용화
`cutoffs.ts:191` TIER_BY_CLASS(2000=A1…100=D5)가 라우트마다 중복. 새 라우트+UI에서 또 필요 → `packages/shared`로 hoisting 검토(CLAUDE.md 공통 상수 규칙). 단 shared CJS→vite 함정([[project-tikke]] tikke-dev 스킬) 주의 — 타입/상수만이면 OK지만 렌더러가 값 import 시 alias 필요.

### 진행 상태 (2026-08-03)
- [x] worker 라우트 `/league/rankings` — `rankings.ts` 신규 + index 등록. broadcaster_rank_snapshots dedup(broadcaster별 최신) → rank asc/score desc → 상위 100. 오늘 리그<6개면 어제 폴백. 캐시 v1. typecheck 통과.
- [x] 웹 페이지 (apps/web) — `LeagueRankings.tsx`, 라우트 `/rankings`·`/ranking`. 외부 대조 사이트 레이아웃: 리그 탭 18개 + 조각 득실 구간(컷) + 순위 리스트. PasswordGate 재사용. tsc 통과.
- [x] 앱 페이지 (apps/desktop) — `LeagueRankings.tsx`, Sidebar "리그 순위 🔒"(locked, VIP), App `case "leagueranking"`. typecheck 통과.
- [x] **조각 판정 = score vs 컷** (K 통찰 구현): `scoreToFrag()`가 방송인 score를 컷과 비교해 획득 조각(+3/+2/+1/0/−1) 판정. `/league/rankings` + `/league/cutoffs` 둘 다 fetch.

### ⚠️ web vite build 블로커 (내 작업 무관)
`pnpm --filter @tikke/web build` 실패 — `index.html`이 `three` 모듈 import하는데 미설치. 다른 작업(AdSense 커밋 근처)에서 들어온 기존 문제. **내 순위 페이지는 tsc --noEmit 통과.** Pages 배포하려면 three 이슈부터 해결 필요(three 설치 or index.html에서 제거). worker/desktop 배포는 무관(정상).

### K 통찰 (2026-08-03) — 순위 = 컷의 원본
"순위로 조각컷 만들면 되잖아" = 정확. 순위표 백분위 점수 = 컷. tikke는 이미 `cutoffs.ts fetchRankTable`로 순위 스냅샷에서 컷 산출 중. 순위 리스트와 컷은 같은 `broadcaster_rank_snapshots`의 두 뷰(원본 리스트 vs 백분위 요약). Supabase 실측: 8/2 순위 2894행/18리그, 컷 69행/18리그 — 둘 다 채워짐. → 리그 페이지에 순위+컷 함께 표시(외부 대조 사이트 레이아웃).

### 카나리 주의
league_cutoffs에 미래날짜(2099-12-31) 3행 = 카나리 트랩([[reference-canary-watermark]]). 순위 라우트는 TIER_BY_CLASS(18리그)만 출력하므로 class_type 99999 카나리는 자동 제외. 컷 서빙은 기존 cutoffs.ts가 처리.

### 안 하는 것 / 제약
- 외부 대조 사이트 API 직접 연동 안 함(K 지시: 자체 수집). 남 서버 의존 회피.
- 수집 커버리지 한계: 신뢰 KR 앵커가 cron 시 라이브인 리그만 채워짐(rank-cron 문서화된 한계). sparse 리그는 어제 폴백으로 완화.
- **배포 안 함**(K 지시). 라우트/UI 로컬 준비 후 K "배포해" 시.

## 순위 오염 근본 원인 + 가드 수정 (2026-08-03)

### 증상
tikke A1 컷이 외부 대조 사이트 대비 10%=4.3배, 20%=3.8배, 60%=8배 부풀림(1%만 일치). 순위 1위 2,479,492 vs 외부 대조 사이트 158K. 전 리그 max/p90 5~18배(정상 ~2.2).

### 근본 원인 (에이전트 진단, 실데이터 검증)
`broadcaster_rank_snapshots`에 **해외 지역 순위표(rank_type=16)가 region=kr로 저장**됨. A1 오염 46명은 한국 189명과 ID 겹침 0(별개 방송인, 해외 A1). `saveLeagueRanks`가 `region:"kr"` 무조건 하드코딩, `saveRanksOnce`의 명단·점수 가드가 "오늘 그 리그 kr 베이스라인"에 의존 → **KST 새 날 시작(establish 윈도우)엔 베이스라인이 비어 가드 무력화**. `canEstablish=true` 부트스트랩 앵커가 반환한 해외 보드가 검증 0으로 그날 기준선이 됨. `fetchLeagueKrTopScore`가 today만 봐서 establish 시 null → 점수 가드 통째로 죽음.

### 수정 (rank-cron.ts)
- `fetchLeagueKrMedianScore()` 신규 — 어제 kr 순위표 중앙값(소수 오염에 강건).
- `saveRanksOnce` 점수 가드: `knownTop`(오늘 max)이 null이면 어제 중앙값 × `ESTABLISH_MEDIAN_RATIO`(12)를 절대 상한으로 폴백. establish 윈도우 구멍을 막음. 실측: A1 어제 중앙값 149K × 12 = 1.79M < 해외 2.48M(걸림), 정상 585K 통과.
- worker typecheck 통과.

### 기존 오염 청소 — 강행 안 함 (의도적)
사후 청소 시도 전부 실패/위험:
- score 상한: 오염(312K~2.48M)과 정상(255K~585K) **겹쳐** 완전 분리 불가.
- 배치 median: 리그·날짜별 분포 변동 + 오염 배치 median이 리그 median의 4배 정도라 경계 모호.
- 명단 코어 대조: 정상인데 그날 1회만 수집된 배치까지 "겹침 0"으로 오탐 → 대량 오삭제 위험.
- median×12 청소: Supabase current_date(UTC) vs snapshot_date(KST) 불일치 + 8/1 오염 median 높음.
→ **잘못된 삭제 리스크 > 청소 이득.** 가드로 앞으로 차단 + 내일 KST 자정 정상 수집이 새 snapshot_date로 교체. 순위/컷 완전 정확은 내일부터.

### 남은 위험
- 오늘/어제 순위 페이지는 잔여 오염으로 일부 부정확(특히 A리그 상위). 내일 정상화.
- 부트스트랩 앵커가 해외 보드를 반환하는 근본(왜 한국 앵커 room에서 해외 A1이 오나)은 미규명 — 가드가 결과는 막지만 유입 자체는 별도 조사 대상.

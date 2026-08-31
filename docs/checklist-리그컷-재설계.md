# 리그 컷오프 재설계 — 틱톡 자체수집으로 외부 대조 사이트 동급 출력 (2026-07-30)

## 확정된 설계 근거 (실측)
- `ranks`는 **상위 100위까지만** (broadcaster_rank_snapshots deepest_rank=99 반복). A1은 626위 앵커 존재 → 600명+.
  → ranks 순서통계량만으로 60%/70%/80% 컷 산출 **불가**. gap 방식이 유일한 경로.
- `IndexDescription`은 실제 위치가 아닌 **구간 라벨** (A1에서 rank 139~237이 전부 60). 인원 역산 불가.
- 리그 선택 파라미터 **없음** (primary_tabs/sub_tabs 전 덤프 빈 배열). 리그 X의 컷을 얻으려면 리그 X의 **라이브 앵커**가 필요.
- gap→cutoff 공식은 **정확**. 검증 2건:
  · B3: 앵커 10,270 + gap 3.9K = 14,170 vs 외부 대조 사이트 14,128 (오차 0.3%)
  · B5 70%: 자체 944 vs 외부 대조 사이트 936 (오차 0.9%)
- 결함은 ①시간 왜곡 ②커버리지 희박 ③serve 탐욕 절단. 파싱/공식 문제 아님.

## 완료
- [x] DB: league_cutoffs에 snapshot_at, owner_score, gap_amount, gap_variant, anchor_rank 추가
      (전부 nullable + 인덱스. 기존 행/코드 무영향 — 적용 완료)
- [x] serve 재구성 알고리즘 프로토타입 작성 + 실데이터 검증
      (scratchpad/proto-cutoffs.mjs — 시간정규화 → 캐논보간 → 어제형태채움 → 단조하한리프트)
      결과: 단조성 전 리그 성립, 16/18 리그 출력, `1%=1089` 증상 소멸

## 남은 결함 2개 (프로토타입 검증에서 드러남)
- [ ] **A. 이상치 가드 미구현** — A1 60%=565,855가 어제 60%(75,979)의 7.4배.
      이 값이 단조 리프트의 상한이 되어 A1 전 구간을 565,855로 평탄화.
      → 어제 같은 퍼센타일 대비 배수 상한(예: 3x) 초과 시 배제. 어제도 비면 리그 곡선 중위값 기준 MAD 배제.
- [ ] **B. 커버리지 (핵심)** — 얇은 리그는 앵커 1명뿐이고 그 1명이 보는 목표%는 대개 유지컷(B=70,C=80,D=20).
      상위 구간(1/10/20%)이 실측 0 → 형태채움/리프트로도 평탄(B5·C4·C5 전부 동일값).
      → **임계 타겟 앵커 선정**이 정답: user_league_data.score로 각 캐논 임계 바로 아래 앵커를 골라 폴링하면
        틱톡이 그 앵커에게 "top N%까지 M 다이아" 를 보여줌 → 캐논별 전용 실측 확보.
        시드는 18리그 전부 보유(100~2000, fresh_7d: 얇은 곳 8~11명 / 두꺼운 곳 2,800명).
      → 워커 runCutoffCron의 시드 선정을 무작위→임계타겟으로 교체. TARGET_ANCHORS_PER_CLASS를 캐논 수만큼.
- [ ] B4(1200)·D1(500)은 오늘 수집 0건 — B 해결 시 함께 해소되는지 확인 필요

## 이식 (A·B 확정 후)
- [ ] worker `routes/cutoffs.ts` — 프로토타입 알고리즘으로 교체, 어제는 intraday MAX(history_yn='Y' 죽음)
- [ ] desktop `electron/main/ipc.ts` cutoffs 핸들러 — 동일 로직 (현재 "최신값 채택"도 MAX로 교체)
- [ ] worker `lib/rank-cron.ts` — 임계 타겟 시드 선정 + snapshot_at/원본입력 기록
- [ ] desktop `services/league-cutoff-collector.ts` — snapshot_at/원본입력 기록
- [ ] dump ranks slice 2→100

## 검증
- [ ] pnpm --filter @tikke/desktop typecheck / @tikke/worker typecheck
- [ ] 18리그 전부 캐논 완비 + 단조 성립
- [ ] 외부 대조 사이트 값과 오차 비교 (B3·B5처럼 1% 내면 정상)

## 배포
- [ ] 사용자가 "배포해" 명시할 때만

## A 유지컷 60% → 50% 전환 대응 (2026-08-28)
- [x] worker `routes/cutoffs.ts` — A 캐논 `[1,10,20,50,60]`, `maintOf()`, `pickMaint()` (유지 칸 1개만 서빙)
- [x] worker `routes/cutoffs.ts` — PRIOR_FALLBACK A 50% 비율 0.52 추가(08-27 실측 3리그 중앙값)
- [x] worker `routes/anchor.ts` — `MAINT_PCTS` 도입, `keepPct = 값 있는 유지% 우선`, 조각 총수 계산 보정
- [x] worker `lib/league-discovery.ts` — 수집 겨냥 캐논 A `[1,10,20,50]`(실제로 오는 쪽만)
- [x] desktop `services/league-cutoff-collector.ts` — CANON A 50·60 둘 다, KEEP_PCTS 배열화
- [x] desktop `pages/LeagueCutoffs.tsx` — 유지 칸 `percentile 50 / alt 60`, 범례 "50-80%"
- [x] desktop `pages/LeagueRankings.tsx` — fragBands A 60 → 50
- [x] web `pages/LeagueCutoffs.tsx` — 동일(alt 폴백, stale 기준 `>= 50`)
- [x] web `pages/LeagueRankings.tsx` — fragBands A 60 → 50
- [x] web `cutoffs/index.html`(정적) — 칸 alt 폴백 + `p:50`
- [x] overlay `anchor-badge.html` 데모 유지% A 60 → 50 (desktop·worker 양쪽 동일)
- [x] typecheck 3종(worker·web·desktop) 통과
- [x] `wrangler dev --local` 실측 검증 — A 유지 50% 3리그 전부 채워짐, 외부(tikdo) 대비 ±5%
- [ ] 배포 (K 지시 대기)

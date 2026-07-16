# 앵커 리그 배지 수정 + 리디자인 체크리스트

## 문제
🏅 앵커 리그 배지가 "뜨는데 값(랭크/점수/퍼센타일/티어) 비어있음".
근본 원인: 앵커 **본인 현재 상태**(class_type/score/tier)를 어디에도 저장 안 함.
- `parseCutoff`가 라이브 중 owner_rank에서 다 파싱하면서 cutoff_score만 저장, 본인 위치는 버림.
- `user_league_data.score`/`league_tier_id` = 전 행 null (DB 확인: score not null = 0 rows).
- `broadcaster_rank_snapshots`는 라이브 1회 수집 + 오늘 없음 + 조회 fallback 없음.
- classType만 legacy로 일부 채워져 배지 렌더만 됨 → 값 빔.
- 미완성 기능 (회귀 아님).

## 참고: realtime 모델
owner의 {score, rank, cur_user_percent, gap, class_type}가 한 응답에 반환 → 별도 snapshot 없음. Tikke도 같은 owner_rank 데이터 이미 파싱 중 → 저장만 추가하면 됨.

## 작업

### A. 데이터 — 앵커 본인 상태 저장
- [x] `league-cutoff-collector.ts`: `parseAnchorState(rankView)` 추가 — owner_rank에서 class_type/score/tier 반환. gap 없어도 class_type+score 반환 (parseCutoff null과 독립).
- [x] `collectLeagueCutoff`에 displayId 파라미터 추가 → `user_league_data` upsert (unique_id=displayId, merge-duplicates).
- [x] `tiklive.ts:559,562`: collectLeagueCutoff 호출부에 displayId(clean) 전달.

### B. 데이터 — getLeagueBadge 읽기 보강
- [x] `ipc.ts`: percentile 계산 조건 `rankRow?.score` → `effScore = rankRow?.score ?? lr?.score`.
- [x] `ipc.ts`: score 변수도 `effScore`.
- [x] `ipc.ts`: snapshot 조회 fallback (오늘 없으면 최근 날짜).

### C. UI — 배지 카드 리디자인
- [x] 🏆 이모지 → tier 그라데이션 엠블럼 링(conic-gradient + 내부 원 tier 약자).
- [x] tier별 그라데이션 배경 + glow, Metric 계층 타이포, 데이터 없을 때 "수집 중" 안내.
- [x] 기존 CSS 변수(--surface/--text 등) 유지.

## 검증
- [x] `pnpm --filter @tikke/desktop typecheck` — 통과 (두 tsconfig 클린).
- [ ] 라이브 동작 확인 — K님이 라이브 켰을 때 배지 값 채워지는지 (typecheck만으론 불충분, captureRankRaw는 라이브 세션 필요).

## 미확정
- owner_rank.rank(순위 숫자) 필드 존재 여부 — 라이브 캡처해야 확인. 있으면 저장 추가, 없으면 snapshot fallback 유지.

---

# 2차: anchor-badge 오버레이 테마 + 설정

대상: anchor-badge 오버레이 확장. 테마 tikke 자체 4종.

## D. 오버레이 HTML (anchor-badge.html)
- [x] 테마 4종 CSS (default/neon/minimal/luxury), data-theme 전환.
- [x] URL 파라미터: theme/font/scale/itemGap/showTier/showClass/showRank/showPct/showScore/showTime (+ 기존 username/trophy/bgOpacity/demo/debug).
- [x] 데이터 버그 fix (ipc.ts와 동일): snapshot fallback + score fallback(effScore). [[feedback_fix_scope]]

## E. 설정 UI + URL 빌드 (EventOverlay.tsx)
- [x] URL 빌드: theme/font/scale/itemGap/show* 파라미터 추가.
- [x] 설정 UI: 테마 드롭다운 + 글꼴 + scale/itemGap + 표시항목 토글 6종.

## F. 동기화 + 검증
- [x] worker/public/overlay에 anchor-badge.html 동기화(cp, diff 동일). [[feedback_overlay_worker_sync]]
- [x] typecheck 통과. 테스트 링크 제공. [[feedback_test_link]]

---

# 3차: 젬 = 누적 보유 조각 전환 (2026-07-13)

배경: 틱톡 젬 = 누적 보유 조각(라운드 종료 시 갱신). 기존 오버레이는 "이번 라운드 컷 달성"으로 젬을 켜서 틱톡과 불일치 (K 스샷: 상위 37%인데 틱톡 젬 2/3 점등, 우리는 전부 꺼짐).

## 덤프 분석 (rank-view-dump-*.json 4개, 필드 확정)
- [x] 보유 조각 = `owner_rank.gap.gap_extra.class_rank_extra.class_title.get_star_number`(=2) / `total_star_number`(=3) — 틱톡 젬 UI와 정확히 일치.
- [x] 라운드 종료 = 최상위 `next_period_time_stamp`(절대 epoch초, 4덤프 불변 1783954800) — `countdown`초와 합치 확인. 딥스캔이 잡던 `/countdown`은 최상위 상대초였음.
- [x] `tile_extra.tiles_available`(=0) ≠ 조각 — 타일은 별개 데이터 확정 (조각 2인데 타일 0).
- [x] gap_description 패턴 3-piece 확인: "{0} more Diamonds ... top {1}% and get {2} fragment" — 기존 [0],[1] 파싱 그대로 유효.

## 수집기 (league-cutoff-collector.ts)
- [x] AnchorState에 `owned_fragments`/`total_fragments` 추가 + class_title 파싱.
- [x] round_end_at: `next_period_time_stamp` 우선(절대값), count_down 딥스캔 폴백.
- [x] Supabase `user_league_data`에 `owned_fragments`/`total_fragments` int 컬럼 추가 (MCP ALTER 실행 완료).

## 오버레이 (anchor-badge.html, desktop+worker 동기화)
- [x] `piecesOwned`(owned/total 기반 배열) 최우선, 구 데이터는 기존 컷 달성 모델 폴백.
- [x] 총 젬 수 = `total_fragments` 우선, 없으면 캐논 조각 수(canon.length-1).
- [x] worker/public/overlay 사본 동기화.

## 검증
- [x] `pnpm --filter @tikke/desktop typecheck` 통과.
- [x] playwright 실측: piecesOwned=[t,t,f] 주입 → 젬 32/32/28(2점등), 시간 08h08m(29333s), 갭 라이브 차감 66.2k→61.4k 정상.
- [ ] **K 라이브 검증**: 새 빌드(또는 dev 재시작) 후 수집분부터 ①gap 50,476대 숫자 그대로 표시(콤마 파서 fix) ②user_league_data.owned_fragments=2 채워지고 젬 2/3 점등 ③round_end_at이 next_period_time_stamp 기반으로 저장.

## 확정 지식 (이 스샷으로)
- A리그 실구간: 99%(-1) / 60%(+0 유지) / 20%(+1) / 10%(+2) / 1%(+3). 캐논 A=[1,10,20,60] 일치. 99% 미만 낙하 시 조각 차감(-1) 구간 존재(신규 — 현재 미모델링).

---

# 4차: TTLS 표시 동일화 (2026-07-13)

TTLS 리그 바와 표시 정보·수치 의미 1:1 일치.

- [x] 갭 = 콤마 원숫자(50,476) — fmtK 축약 폐지, fmtNum + 트윈 동일 적용.
- [x] 갭 줄 = "N 더 있으면 상위 20% [젬] 조각 +1" / "상위 60% 유지" — 목표 % + 획득 조각 수 표시.
- [x] gap_fragments 수집 (gap_description pieces[2] "get {2} fragment") + DB 컬럼 추가 완료.
- [x] 현재 위치 칩 = "상위 37%" (current_percentile, 15분 fresh일 때만) 티어 행 우측.
- [x] 시간 줄 라벨 = "조각 업데이트 HH:MM" (TTLS "다음 조각 업데이트").
- [x] 폴백(컷 계산) 경로도 목표 % 표시 — pieceCuts {pct,cut} 구조 전환, 데모 갱신.
- [x] !리그 명령 = "조각 2/3 보유" + "상위 20% 조각 +1까지 N 다이아" 문구.
- [x] worker 사본 동기화, desktop typecheck, playwright 실측(공식/유지/폴백 3케이스 + 스크린샷).
- [ ] K 라이브 검증 — gap_fragments/owned_fragments는 새 수집분부터.

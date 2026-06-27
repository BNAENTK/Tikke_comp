# 앵커 리그 배지 수정 + 리디자인 체크리스트

## 문제
🏅 앵커 리그 배지가 "뜨는데 값(랭크/점수/퍼센타일/티어) 비어있음".
근본 원인: 앵커 **본인 현재 상태**(class_type/score/tier)를 어디에도 저장 안 함.
- `parseCutoff`가 라이브 중 owner_rank에서 다 파싱하면서 cutoff_score만 저장, 본인 위치는 버림.
- `user_league_data.score`/`league_tier_id` = 전 행 null (DB 확인: score not null = 0 rows).
- `broadcaster_rank_snapshots`는 라이브 1회 수집 + 오늘 없음 + 조회 fallback 없음.
- classType만 legacy로 일부 채워져 배지 렌더만 됨 → 값 빔.
- 미완성 기능 (회귀 아님).

## 참고: TikDo realtime 모델
RPC가 owner의 {score, rank, cur_user_percent, gap, class_type} 한 응답에 반환 → 별도 snapshot 없음. Tikke도 같은 owner_rank 데이터 이미 파싱 중 → 저장만 추가하면 됨.

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

# 2차: anchor-badge 오버레이 테마 + 설정 (틱두 리그컷 설정 참고)

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

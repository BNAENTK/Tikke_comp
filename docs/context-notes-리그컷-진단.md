# 리그 컷오프 진단 (2026-07-30)

## 결론 (최종)
**파싱 버그 아님. 시간 왜곡(temporal aliasing).**
각 퍼센타일이 서로 다른 시각에 1개씩 측정되는데, 컷은 하루 중 계속 상승하므로
"동시점 곡선"으로 비교하면 단조성이 붕괴한다. serve는 이걸 필터로 걸러내려다 정상값까지 학살한다.

## 증거 — A1(2000) 2026-07-30 자체수집, 측정 시각 포함
```
p1  →     1,089  (KST 00:01)  ← 리셋 직후
p10 →     1,301  (KST 00:06)  ← 리셋 직후
p20 →    89,176  (KST 01:21)
p26 →   869,661  (KST 03:41)
p35 →   810,861  (KST 05:56)
p73 →   565,855  (KST 05:46)
p98 →   170,557  (KST 09:51)
p60 →   565,855  (KST 12:23)  ← 가장 최신
```
각 값은 **측정 시점 기준으로는 맞다.** 문제는 p1/p10이 자정 직후(거의 0)에 한 번 찍히고
하루 종일 재측정되지 않아, 정오의 p60(565K)이 p1(1,089)보다 커지는 것.
serve `clean()`은 낮은 %부터 훑으며 단조성을 강제하므로 p1=1,089가 첫 앵커가 되어 뒤를 전부 버림
→ API가 `1%=1089` 한 줄만 반환.

## gap→cutoff 공식은 정확함 (검증 완료)
덤프 ct=1300(B3): owner score 10,270 + gap "3.9K" → **14,170**
외부 대조 사이트 B3 20% today = **14,128** → 오차 0.3% (축약 "3.9K"의 반올림 손실분)
→ 공식·파싱 정상. 문제는 샘플링 방식.

## 틱톡 응답에 실제로 있는 것 (rank_type=16, 덤프 22개 실측)
top-level: `bulletin, countdown, cut_off_line, has_last_rank, is_position_safe,
midday_peak_time_stamp, next_period_time_stamp, owner_rank, rank_extra_info, ranks, ...`

- **`ranks`** = 같은 리그 순위표. 항목당 `rank`, `score`(정확한 정수), `score_description`, `user{display_id,...}`.
  실측 rank1 score=96,762. **순서통계량으로 임의 퍼센타일 컷을 동시에 산출 가능** ← 핵심
  ※ 덤프 코드가 `ranks`를 2개로 잘라 저장하므로 실제 깊이(top N)는 미확인 — 확인 필요
- **`owner_rank.gap.gap_description`** — 현재 사용 중. 축약 숫자("70.1K") = 정밀도 손실.
  변종 5종 실측:
  - `diamondsToTopX_1/2diamondsPlural` — "{0} 다이아 더 필요 → top {1}%, 조각 {2}개" (정상 파싱)
  - `avoidLosingFragments_1diamondsPlural` — "top {1}% 유지" (정상 파싱)
  - `detail_maximum` — "조각 최대 획득" pieces 없음 → null 반환 (안전)
  - `pm_mt_league_bottom_aheadPlural` — "{0} more to be No. 1" pieces 1개 → null 반환 (안전)
- **`IndexDescription`** pieces[0] = 앵커 현재 상위 % (점수↑ → 값↓ 확인: 44→43). 랭크 아님.
- **`cut_off_line`** — 필드 존재하나 덤프 22개 **전부 빈 배열**. 특정 조건(라운드 종료 등)에만 채워질 가능성. 미확인.
- `midday_peak_time_stamp`, `next_period_time_stamp`, `is_position_safe`, `countdown` — 라운드 메타.

## 외부 대조 사이트 방식 (외부 관측 결과)
- 자체 Supabase + 자체 Next.js API. 남의 API 중계 아님. tikke 카나리 미노출 = 도용 아님.
- **리그 그룹 단위 배치 삽입** (동일 created_at + 연속 id), ~5분 주기, 캐논 % 전부, 단조 정합.
- → 한 시점 스냅샷에서 전 퍼센타일을 동시 산출하는 구조. `ranks` 순서통계량 방식과 부합(추정, 미확증).

## ext 피드 중단 원인 (확정)
`scripts/migrate-league-cutoffs.ts`의 base64 `SRC_URL` = 외부 대조 사이트 Supabase.
`ext-migration` 5,661행(2/26~6/27) + `ext` 57행(6/28 1회) 후 중단.
**원인: 해당 anon 키가 현재 401 (로테이션).** 죽은 코드 아니라 키 만료.

## 수정 방향
### A. 자체수집 정상화 (틱톡 직접, 의존 없음)
1. **`ranks` 순서통계량** — 한 번의 rank_view 호출로 전 퍼센타일 동시 산출.
   시간 왜곡 원천 제거 + 캐논 % 완비. 선행 확인: `ranks` 깊이(top N)와 리그 총 인원 파악 가능 여부.
   (덤프 코드의 `slice(0,2)` 때문에 현재 미확인 — 이것부터 봐야 함)
2. gap 방식 유지 시: 측정 시각을 함께 저장하고 **동시점끼리만 비교**. 자정 직후 값은 낮 시간대 곡선에 섞지 말 것.
3. 축약 숫자 정밀도 — `score_description`(축약) 대신 `score`(정수) 사용. gap은 축약뿐이라 한계.

### B. serve 필터
탐욕 단조 필터는 근본 해결 아님. 시간 왜곡이 사라지면 필터 자체가 거의 불필요.
남겨둘 경우 LIS(최장 비증가 부분수열)로 바꾸되, 개수 기준은 스테일 저점 체인을 선호할 위험 →
값 가중 또는 캐논 % 앵커 고정 필요.

### C. ext 복구 (선택)
공개 API(`/api/league/cutoffs/compare`)로 갈아타면 키 무관. 단 경쟁사 상시 수집은 K님 판단 사항.
A가 성립하면 불필요.

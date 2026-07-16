# 앵커 리그 배지 — 컨텍스트 노트

## 진단 근거 (DB 실측, 2026-06-26)
- `league_cutoffs`: 오늘(2026-06-26) 365 rows 정상.
- `user_league_data`: 7434 rows. `class_type` 일부만 non-null(legacy). `score` not null = **0 rows** → 본인 상태 저장 코드 한 번도 없었음.
- `broadcaster_rank_snapshots`: 최신 = 2026-06-25, created_at 단일 `00:05:11`(라이브 1회 수집). 2026-06-26 = 빈 배열.

## 데이터 흐름 (확정)
getLeagueBadge(ipc.ts:1497) 3소스 조합:
1. user_league_data.class_type/tier/score — `collectUser`는 unique_id/nickname만 씀. 본인 상태 안 채움.
2. broadcaster_rank_snapshots rank/score — `collectRankSnapshot`(rank-collector.ts) 라이브 연결 시 1회(tiklive.ts:557, interval 없음).
3. league_cutoffs — worker cron + desktop collectLeagueCutoff. percentile 계산용(score 必).

owner_rank(rank_type=16) 응답에 본인 score/percentile(gap pieces)/class_type/tier 전부 있음. parseCutoff가 이미 파싱하지만 cutoff_score만 저장.

## 키 매칭 (중요)
- getLeagueBadge 조회: uid가 숫자면 그대로, 아니면 did → 둘 다 `user_league_data.unique_id` 컬럼 대상.
- Dashboard.tsx:282 getLeagueBadge 호출 시 uniqueId/displayId 둘 다 `connectedUsername`(보통 displayId 문자) 전달.
- collectUser(clean) tiklive.ts:554 → user_league_data.unique_id = displayId 행 생성.
- 따라서 앵커 본인 상태도 unique_id=displayId 행에 merge upsert해야 조회와 매칭됨.
- collectLeagueCutoff는 anchorId(숫자 owner id)만 받음 → displayId 별도 전달 필요.

## 결정
- percentile은 기존 cutoff 비교 계산(ipc.ts 1536-1559) 유지. gap pieces[1]은 "다음 목표%"라 현재 위치 아님 → 저장 안 함. tier는 owner_rank에서 직접 저장.
- 앵커 상태 저장은 parseCutoff null과 독립(미참가/top랭크여도 score/class 저장).
- snapshot 읽기 fallback 추가(오늘 없으면 최근) — cutoff 쪽 1384에만 있고 snapshot 1526엔 없던 누락 보완.

## 검증 한계
captureRankRaw는 라이브 세션(sessionId/roomId) 필요 → typecheck가 floor. 최종 동작은 K님 라이브 중 확인.

## 미확정
owner_rank.rank 필드 존재 여부 — 라이브 캡처 필요.

## 2026-07-11 추가 — league 테마 (위젯 스타일)

K 요청: 리그 위젯 스샷(💎 03h 51m / B4 ◆◆◆ 리그 / 2.3k 더 있으면 ◆ 유지)처럼.

- anchor-badge.html에 `theme=league` 추가 — 투명 배경, Cafe24Ssurround, 흰 글자 +
  블루 8방향 아웃라인 + 하단 글로우, skewX(-4deg), SVG 젬(다이아/헥사).
- 3줄: ① 라운드 남은시간(round_end_at 기반, 30초 갱신, 데이터 없으면 숨김)
  ② {tier} ◆◆◆ 리그 ③ 유지/승급 갭.
- 갭 계산: league_cutoffs floors(티어→최저 컷오프). score < 내 티어 floor → "유지"(부족분),
  넘었으면 다음 상위 floor까지 "승급". floors에 내 티어 없으면 갭 줄 숨김.
- round_end_at: user_league_data에 컬럼 추가(migration add_round_end_at_to_user_league_data).
  수집기 parseAnchorState가 rank_view에서 count_down 계열 키 딥스캔(정확한 필드명 미확정,
  0<초<8일). 발견 시 경로 console.log — **K 라이브 때 로그로 필드 확인 필요**.
  발견 전까지 오버레이 시간줄은 자동 숨김.
- EventOverlay.tsx 테마 셀렉트에 "리그 팝" 옵션. 기본 테마는 변경 안 함(기존 사용자 보호).
- 검증: typecheck 통과, ?theme=league&demo=1 스크린샷으로 3줄 렌더 확인.

### 조각 시스템 규칙 (K 확인, 2026-07-11 "정확해")
- 리그 퍼센타일 컷 중 **최하위(최대 %) = 유지컷**, 나머지 = **조각컷**(쉬운 % 순).
  B리그(1/10/20/70): 70%=유지, 20/10/1%=조각 1~3. 컷 달성 시 조각 획득.
- 젬 아이콘 = 조각 획득 상태(밝음=획득/어두움=미획득). 개수 = 조각컷 수(A/B=3, C=2, D=1).
- 갭 줄: 유지컷 미달 → "X 더 있으면 유지". 유지 확보 → 다음 미획득 조각컷까지 "X 더 있으면 조각".
  전부 획득 → 갭 줄 숨김. 컷오프 데이터 자체가 없으면(2행 미만) 젬 3개 미획득 폴백.

### 공식 gap 우선 (2026-07-11)
- user_league_data에 gap_amount/gap_percentile 추가(migration add_official_gap_to_user_league_data).
  수집기가 owner_rank.gap pieces([0]=다이아,[1]=목표%)를 그대로 저장 — gap 없으면 명시적 null
  (스테일 값이 fresh updated_at 달고 남는 것 방지).
- 오버레이: updated_at 15분 이내 fresh면 공식 gap이 컷오프 계산값을 대체. 목표% == 유지컷%
  (컷 데이터 없으면 그룹별 고정 A60/B70/C80/D20) → "유지", 아니면 "조각".
- 라이브 중 3분 주기 갱신. 방송 꺼지면 15분 후 자동으로 계산값 폴백.

### 라이브 업그레이드 5종 (2026-07-12, "3번 빼고 전부 진행")
1. **실시간 갭** — 오버레이가 선물 이벤트 WS 구독(로컬 ws:18182 `{type:'gift',payload}` /
   클라우드 room ws raw 이벤트). base(마지막 수집 점수+updated_at) + 이후 선물 다이아 합산 =
   라이브 점수. base 갱신 시 base 이전 선물 로그 폐기(이중계산 방지). 공식 gap도 라이브 차감,
   0 이하로 돌파하면 컷 계산 폴백. 수집·스트림 간 수 초 오차는 다음 수집서 자동 보정.
2. **마일스톤 연출** — 조각 획득(젬 pop 애니 + 배너 "조각 획득!") / 유지 확정(라벨 유지→전환
   감지, 배너 "유지 확정!") + 화면 플래시 + WebAudio 차임(C5-E5-G5). `?fx=0`으로 끔.
3. (임계 알림 — 사용자 스킵)
4. **진행률 게이지** — 다음 목표컷 대비 score 비율 바. 갭 줄 아래.
5. **!리그 채팅 명령** — command-service 내장(30초 쿨다운, 유저 명령보다 먼저).
   getAnchorLeagueSummary()가 user_league_data에서 tier+fresh gap 읽어 TTS(tikke:tts:speak).
6. **갭 숫자 롤링** — 600ms ease-out 트윈. 라벨/티어 구조 바뀌면 재구성, 숫자만 바뀌면 트윈.
- 데모: theme=league&demo=1 → 4초마다 600다이아 가상 선물 12회 — 유지 확정/조각 획득 연출 재현.
- 검증: Playwright로 실시간 감소·게이지·젬 pop·배너 확인(배너는 3초라 clearTimeout 고정 후 캡처).

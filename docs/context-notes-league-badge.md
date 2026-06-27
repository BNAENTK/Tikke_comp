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

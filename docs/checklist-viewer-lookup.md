# 시청자 후원 내역 조회 (틱두 이식) — 체크리스트

목표: 시청자 한 명을 지정하면 "어느 호스트 방에서 얼마를 썼는지"를 본다. 조회는 K(admin)만.

## 1단계 — viewer_crm 기반 (데이터 이미 존재, 즉시 동작)
- [x] `viewer_lookup_admins` 화이트리스트 테이블 + `fn_viewer_lookup_is_admin()` (auth.uid 기반, K 등록 완료)
- [x] RPC `fn_viewer_summary(p_unique_id)` — 전체 코인/호스트 수/선물/방문/첫·최근 목격
- [x] RPC `fn_viewer_top_hosts(p_unique_id, p_limit)` — 호스트별 코인·선물수·방문수
- [x] 호스트 닉네임 해석: user_league_data → rank_snapshots.display_id → viewer_crm → 아이디 원문
- [x] RPC `fn_viewer_search(p_query, p_limit)` — 닉네임/아이디 검색
- [x] desktop `electron/services/viewer-lookup.ts` — 로그인 JWT로 RPC 호출
- [x] IPC `tikke:viewer:isAdmin|search|profile` + preload 브릿지
- [x] `src/components/ViewerLookupPanel.tsx` — 검색 + 프로필 패널
- [x] ViewerCRM 상단 "🔎 시청자 후원 조회" 버튼 (admin일 때만)
- [x] ViewerCRM 카드 모달 "🔎 다른 방 후원 내역" 버튼 (admin일 때만)

## 2단계 — 신규 롤업 수집 (시간 누적 필요)
- [x] 테이블 `viewer_gift_daily` PK(broadcaster_id, unique_id, gift_date)
- [x] 테이블 `viewer_gift_by_name` PK(broadcaster_id, unique_id, gift_id)
- [x] 테이블 `viewer_nickname_history` PK(unique_id, nickname)
- [x] 세 테이블 모두 anon/authenticated 직접 접근 차단 (RPC만 통로)
- [x] 증분 RPC `fn_viewer_rollup_bump(p_broadcaster, p_items jsonb)` — 배치 가산 upsert
- [x] desktop `electron/services/viewer-rollup.ts` — 5초 배치 flush, 실패분 requeue
- [x] main/index.ts gift 구독 + live-connect 방송인 ID + 연결 종료 시 flush
- [x] 조회 RPC `fn_viewer_gift_daily` / `fn_viewer_top_gifts` / `fn_viewer_nicknames`
- [x] 패널에 일별 막대 그래프 + 선물 TOP + 닉네임 이력
- [x] 보존 정책 `prune_viewer_rollups` (일별 180일 / 나머지 365일) — worker cutoff-retention 하루 1회에 편입

## 노출 범위 — dev 전용
- [x] 렌더러: `import.meta.env.DEV`가 아니면 `useIsLookupAdmin`이 항상 false → 진입점 없음
- [x] main IPC: `NODE_ENV=development || ELECTRON_RENDERER_URL` 아니면 세 핸들러 모두 빈 응답
- [x] 프로덕션 번들 실측: 렌더러 번들에 패널 문자열/브릿지 호출 없음, main 번들엔 dev 게이트 존재
- [x] 수집(롤업)은 배포 빌드에서도 동작 — 데이터가 쌓여야 조회가 의미 있으므로 의도된 것

## 검증
- [x] `pnpm --filter @tikke/desktop typecheck` / `@tikke/worker typecheck` 통과
- [x] `pnpm build:desktop` 통과
- [x] RPC 실측: `___c__k___` → 호스트 33곳 / 27,324 다이아 (K JWT), anon은 0행
- [x] bump RPC PostgREST 실호출 200 + 세 테이블 가산 확인 후 테스트 행 삭제
- [x] 롤업 테이블 anon 직접 읽기 401 확인
- [x] **실앱 테스트** (`pnpm dev` + CDP 원격 조작, 2026-08-18 00:20~00:35 KST)
  - isAdmin → true (main 프로세스 세션 JWT 정상)
  - 🔎 버튼 노출 → 패널 열림 → 검색 → 프로필 렌더까지 UI 전 구간
  - 카드 모달 "다른 방 후원 내역" 경로도 정상 (달인 3.7M / 젠 방 1곳)
  - 실제 라이브 방(jen2eo) 연결 → 선물 수신 → viewer_gift_daily/by_name/nickname 적재 확인
  - 같은 시청자 5회 선물이 한 행에 누적(coin 5/gift 5), 선물별 4종 분리 확인
- [x] 검색 성능: 1글자 6.1초 → 부분 trgm 인덱스 + 2글자 하한으로 0.3~0.5초

## 배포 (2026-08-18)
- [x] v1.8.325 커밋·태그·push
- [x] Cloudflare Worker 배포 완료 (2026-08-17T15:40:19Z)
- [x] Supabase 마이그레이션 적용 완료

## 범위 밖 (하지 않음)
- 최근 채팅 30건 저장 — 저장량/프라이버시 대비 가치 낮아 보류
- viewer_crm의 anon SELECT 개방 문제 (별건, context-notes 참고)

# 리그 랭킹 페이지 — 체크리스트

외부 대조 사이트 `/ranking/<tier>` 방식을 앱 안에 구현한다. 목록에는 순위·닉네임·점수만 두고,
행을 누르면 팝업에 `@아이디`·라이브 여부·바로가기 버튼을 띄운다(외부 대조 사이트와 동일한 구성).

데이터는 이미 `broadcaster_rank_snapshots`에 전부 있다 — `display_id`·`nickname`·`room_id` 100% 보유,
`region='kr'`로 한국 리그만 걸러진다. 새로 수집할 것은 없다.

## 작업
- [x] ~~main IPC — 리그 랭킹 조회~~ 불필요. 워커 `/league/rankings`가 이미 display_id 포함 (18리그 100% 보유 확인)
- [x] main IPC `tikke:rooms:check` — 클릭한 1명만 fetchRoomSnapshot
- [x] preload `rooms.check` 노출
- [x] `LeagueRankings.tsx` — 페이지는 이미 있었음. 행 클릭 → DetailModal 추가
- [x] ~~Sidebar 메뉴 + App 라우팅~~ 이미 연결돼 있었음 (`leagueranking`)
- [x] typecheck + 렌더러 build 통과
- [x] 실측 확인 — 하네스로 컴포넌트 단독 렌더. 100행 + 팝업 @아이디 정상. 이모지 이니셜 깨짐 버그 1건 발견·수정

## 하지 않는 것
- 목록 전체 라이브 스캔 — 리그당 100회 호출이라 EulerStream 서명 쿼터(분당 5회 익명)를 태운다.
  외부 대조 사이트도 목록에는 라이브 표시가 없다(DOM 확인: LIVE 문자열 0건, @아이디 0건).
- 새 데이터 수집 — 기존 스냅샷으로 충분하다.

---

# 2차 — 표시층 재제작 (2026-08-04)

K 지시: "복소라이브도 리그 랭킹 시스템이 있어 분석해서 아예 새로 만들어보려고 해"
→ 범위 확정: **리그 랭킹 + 조각컷 표시층만**. 수집 파이프라인은 유지.
→ 기준: voxo 단독이 아니라 **외부 대조 사이트와도 대조해서** 동일 수준으로.

## 정찰
- [x] voxo 리그 랭킹 출처 확인 → **tik.tools 유료 벤더**(자체 수집 아님, 리버스 대상 없음)
- [x] voxo 공개 엔드포인트 2개 확보 (global-teaser / ranking check)
- [x] 외부 대조 사이트 순위 페이지 UI 실측 (마크업 + 스크린샷 1280×1400)
- [x] 데이터 격차 실측 — 아바타·라이브·보유조각 컬럼 유무 SQL 확인

## 서브
- [x] `rankings.ts` — `user_league_data` 조인해 보유 조각(`owned`) 내려주기
- [x] CACHE_KEY v5 로 올림 (응답 구조 변경)

## 표시층
- [x] `apps/web/src/pages/LeagueRankings.tsx` 재작성
- [x] `apps/desktop/src/pages/LeagueRankings.tsx` 재작성 (DetailModal 유지)
- [x] 조각 득실 구간 카드 — 배지 + 퍼센트 + 축약값 + 정확값
- [x] 구간 색상 범례
- [x] 순위 행 — 메달 / 아바타 / 닉네임 / 보유조각 / 점수 / 획득조각
- [x] 리그 선택 드롭다운(티어 라벨 포함) + 칩 병행

## 검증
- [x] worker / desktop typecheck, web build
- [x] 실제 워커 응답으로 렌더 확인

## 배포
- [ ] **금지** — K가 "배포는 하면 안 되고 일단 만들어야돼" 명시

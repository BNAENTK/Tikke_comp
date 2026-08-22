# 컨텍스트 노트 — 룸 관찰 매치 현황

## 요구
외부 대조 사이트 스크린샷처럼 매치(PK/멀티 게스트) 진행 중 상황 표시 — 팀 구성, 호스트별 매치 파워, 후원자 순위, 후원량. 룸 관찰 페이지에 추가.

## 핵심 결정

### 데이터 소스 = 본인 방 webcast 이벤트 (관찰 방 아님)
- 관찰 중인 타인 방의 매치 데이터는 webcast 연결 없이는 불가 (서명 쿼터 문제로 관찰 방은 api-live 스크래핑만 사용).
- 본인 방 연결(`tiklive.ts`)로 들어오는 `linkMicBattle` + `linkMicArmies` 이벤트에 **양 팀 전체** 데이터가 포함됨 — 외부 대조 사이트도 같은 원리(시청자에게 브로드캐스트되는 메시지).
- 즉 "내가 매치 중일 때" 상대 팀 후원자까지 전부 보임. 매치 중이 아니면 패널 숨김.

### 이벤트 필드 (tiktok-live-connector v2 proto, tiktok-schema.d.ts 확인)
- `WebcastLinkMicBattle`: `battleId`, `battleSetting{startTimeMs,duration}`, `anchorInfo{userId→{user:{userId,nickName,displayId,avatarThumb}}}`, `teamArmies[]{teamId,teamUsers[{userIdStr}],teamTotalScore}`
- `WebcastLinkMicArmies`: `battleId`, `battleItems{anchorId→{hostScore,userArmy[]{nickname,avatarThumb,score,diamondScore,userIdStr}}}`, `teamArmies[]`
- legacy WebcastPushConnection이 camelCase proto 그대로 emit — `handlePkBattle`이 이미 `anchorInfo`를 이 형태로 파싱 중(검증된 전례).
- 점수는 proto string — 콤마 절사 함정 대비 `.replace(/[,\s]/g,"")` 후 Number.

### 구조
- `match-observer.ts` (신설 서비스): 모듈 상태 MatchState. battleId 바뀌면 리셋. 렌더러는 IPC 폴링(`tikke:rooms:matchState`, 페이지 마운트 중 3초) — push 배선보다 단순, 백그라운드 비용 0 (renderer interval은 마운트 중만 동작).
- streamEnd에 리셋 안 걺 — 일시중지 오탐 함정(streamEnd 자동 stop 금지). 대신 staleness로 처리: 90초 무갱신 시 "수신 없음" 표시, 10분 지나면 패널 숨김.
- 필드 구조가 실전과 다를 가능성 대비 — 메시지 타입별 1회 raw 로그(기존 subEventLogged 패턴).

## 후속 변경 (사용자 요청)
- PK 상대 자동 등록(addPkOpponent) 제거 — 옵저버 목록에 매치 기록이 쌓이는 것 방지. 매치는 현재 진행분만 매치 패널로 표시.
- 과거 자동 등록으로 쌓인 `source:"pk"` 룸은 폴링 시 `purgePkRooms()`가 전부 정리 (수동 등록 유지).
- `source` 타입의 `"pk"` 리터럴은 저장 데이터 하위호환(정리 전 로드) 때문에 남김.

## 실측으로 확정된 페이로드 구조 (2026-08-01)
- 2대2에서 `battleItems`는 **빈 객체**. 후원자는 `teamArmies[].userArmies.userArmy`에 **팀 단위**로 온다.
- 그 `userArmies.anchorIdStr`은 호스트 ID가 아니라 **팀 번호("1","2")**. 그래서 armies 키가 팀 번호가 되며,
  `reconcileTeamKeyedArmies()`가 이를 팀 후원자 목록으로 연결한다.
- `BattleUserArmy`에는 **수신 호스트를 가리키는 필드가 없다** → 2대2에서 호스트 개인별 후원 분리는 원본상 불가능.
  팀별 분리까지가 한계. 1대1처럼 armies가 호스트 키로 오면 호스트별 표시가 그대로 동작한다.
- 아바타는 `avatarThumb.url` 배열 (`urlList` 아님).
- `linkMicArmies`의 `battleId`는 `linkMicBattle`과 **다른 체계**. 점수 메시지로 매치를 새로 시작하면 안 된다
  (참가자 없는 상태가 만들어져 "대기 중"으로 표시됨) → `ensureState(battleId, canStartNew)`.

## 매치 참가자 → 리그 컷오프 수집 (2026-08-01)
- 매치 참가자는 "지금 확실히 라이브"인 앵커라 harvester의 발견 단계를 건너뛴다.
- `match-observer.setOnMatchHostsSeen()` 콜백 → tiklive가 본인 제외 후
  `league-cutoff-harvester.collectFromMatchHosts()` 호출 → `fetchRoomSnapshot`으로 roomId 확보 → `collectLeagueCutoff`.
- 같은 상대 재수집 최소 간격 4분, 호출 간격은 harvester와 동일한 700ms.
- 부수효과: `collectLeagueCutoff`가 `user_league_data`에 앵커 상태를 남겨 시드 풀이 스스로 늘어난다.
- 한계: 매치는 비슷한 티어끼리 잡혀 사용자 주변 리그만 두터워진다. 18리그 전체 커버는 기존 harvester 몫.

## 리그 조각컷 — 순위표 기반 산출 (2026-08-02)
핵심: TikTok `rank_view.ranks`가 **그 리그 상위 99명의 순위·점수**를 함께 준다. 컷오프 수집이 이미 받는
응답이라 추가 호출 0. 경쟁 서비스(외부 대조 사이트)도 이 순위표에서 컷을 파생시킨다(그쪽 10% 컷이 19번째,
20% 컷이 40번째 점수와 일치 → 약 195명 기준으로 계산).

- 저장: `broadcaster_rank_snapshots`에 `class_type` 컬럼 추가(마이그레이션 `add_class_type_to_rank_snapshots`).
  리그 구분이 없으면 여러 리그가 섞여 백분위 계산이 불가능하다.
- **리그는 반드시 응답(`rank_extra_info.class_extra.class_type`)에서 읽을 것.** 풀 키를 쓰면 승급·강등 시
  옛 리그로 저장돼 B3에 1.34M(=B2의 18배)이 섞였다.
- **총원 필드는 응답에 없다.** `cut_off_line`은 빈 배열로만 온다(덤프 전수 확인). 99명 기준으로 계산하면
  실제의 약 2배로 부풀어진다 → 실측 컷 하나로 총원을 역산한다:
  `N ≈ (순위표에서 그 컷의 위치) ÷ (퍼센타일/100)`.
  검증: C1 실측 20%=6,559 → 25번째 → N=125 → 재계산 20%=6,578 (0.3% 오차).
- 실측이 하나도 없는 리그는 계산하지 않는다(총원 미상). 99번째를 넘는 하위 구간(유지컷)도 계산 불가 —
  그 구간은 기존 안내문구 역산으로만 채운다.
- 앵커 리그 배지는 이 경로와 무관하다(`user_league_data`). 컷 파싱 실패와 상관없이 항상 저장된다.

## 미확정/위험
- 1:1 PK는 teamArmies 없이 battleItems만 올 수 있음 → 팀 없으면 호스트별 단독 컬럼 폴백.
- 실전 매치에서 필드 검증 전 — raw 로그로 확인 후 필요 시 파싱 보정.

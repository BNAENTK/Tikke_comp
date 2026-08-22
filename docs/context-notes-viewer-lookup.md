# 시청자 후원 내역 조회 — 결정과 근거

## 출발점 — 틱두 리버스 (2026-08-17)
틱두 데스크탑 앱 `app.asar` → `dist/assets/main-BMmBtb5n.js`에서 시청자 프로필 카드 RPC 6종 확인.

| RPC | 파라미터 | 리턴 |
|---|---|---|
| `fn_get_user_gift_total` | `p_user_id, p_unique_id, p_n_days, p_host_user_id` | `total_gift_value`(전체), `host_gift_value`(내 방) |
| `fn_get_viewer_gift_top_hosts` | `p_n_days:30, p_limit:7~10` | `[{host_nickname, total_coin}]` |
| `fn_get_viewer_gift_daily` | `p_host_user_id, p_n_days:30` | 일별 코인 |
| `fn_get_viewer_gifts_by_count` / `_by_price` | `p_limit:10` | 선물 top |
| `fn_get_viewer_recent_chats` | `p_limit:30` | 최근 채팅 |
| `fn_get_viewer_nickname_history` + `fn_sync_viewer_history` | `p_days:180` | 닉 변경 이력 |

틱두는 `role`이 `admin`이면 실제 순서 + limit 10, `special`이면 실제 순서 + limit 7,
그 외 일반 유저는 결과 배열을 Fisher-Yates로 **셔플해서** 준다. 금액은 보여주되 순위는 감추는 설계.
데이터 출처는 틱두 설치 호스트 전원이 올리는 중앙 `gift_events` 테이블(크라우드소싱).

## 결정 1 — tikke는 신규 수집 없이 1단계가 바로 된다
`viewer_crm`이 이미 `(broadcaster_id, unique_id, total_diamond, visit_count, gift_count, last_gift_at)` 구조로
전 유저 앱에서 중앙 수집 중이다. 2026-08-17 실측: 305,019행 / 호스트 58 / 시청자 247,023 / 2026-05-27부터.
`unique_id`로 묶으면 그대로 "어느 방에 얼마"가 나온다(검증: `___c__k___` = 30개 방, 27,324 다이아).
따라서 틱두처럼 새 gift_events 파이프라인을 깔 이유가 없다. **1단계는 조회 RPC + UI만**이다.

## 결정 2 — 없는 항목만 2단계로 신규 수집
viewer_crm은 누적 합계뿐이라 아래는 원본이 없다.
- 일별 후원 그래프 → `viewer_gift_daily` 롤업
- 선물별 top(개수/가격) → `viewer_gift_by_name` 롤업
- 닉 변경 이력 → `viewer_nickname_history`

**롤업으로 가는 이유**: 원본 선물 이벤트를 그대로 쌓으면 행이 폭증한다.
league_cutoffs가 UNIQUE 제약 누락으로 285만행까지 불어나 Supabase 무료 한도를 넘긴 전례가 있다.
그래서 처음부터 `(broadcaster_id, unique_id, 날짜/선물)` PK로 가산 upsert만 한다. 원본 행 저장 없음.

`session_gifts`/`live_sessions` 테이블이 이미 있지만 둘 다 0행 — session-tracker가 실제로 안 도는 상태다.
이걸 되살리면 원본 firehose가 되므로 건드리지 않고 롤업으로 간다.

## 결정 3 — 조회는 K만, 수집은 전원
K 지시: "1+2단계로 하고 나만 볼 수 있게 해".
- 수집(bump RPC)은 anon 실행 허용 — 전 유저 앱이 올려야 데이터가 생긴다. 기존 viewer_crm과 동일 수준.
- 조회 RPC는 SECURITY DEFINER + admin 게이트 (초안은 토큰 방식 → 결정 4에서 계정 화이트리스트로 교체).
- UI 노출은 결정 5에 따라 dev 빌드 + admin 계정 둘 다 만족할 때만.

## 알려진 구멍 (이번 범위 밖, 별도 판단 필요)
`viewer_crm`의 SELECT 정책이 `qual = true` / roles `{anon, authenticated}`다.
즉 앱에 박힌 anon 키만 있으면 누구나 305k행 전체를 읽을 수 있다 — 크로스 호스트 후원 내역 포함.
지금 조회 RPC를 admin 토큰으로 잠가도 이 경로는 그대로 열려 있다.
막으려면 정책을 조이는 게 맞지만, desktop의 CRM 호출이 anon 키만 쓰고 사용자 JWT를 안 실어서
지금 조이면 전 유저의 단골 카드 기능이 깨진다. 인증 경로 정비가 선행돼야 하는 별건이다.

## 결정 4 — 토큰 붙여넣기 대신 로그인 계정 화이트리스트
처음엔 `admin_secrets` + `p_token` 파라미터로 설계했다가 폐기했다. 토큰을 앱에 넣는 순간
배포된 모든 앱이 토큰을 갖게 되고, K만 갖게 하려면 수동 입력 UI가 또 필요하다.
tikke에는 이미 `getSupabaseClient().auth.getSession()` → access_token 패턴이 있으므로(rank-collector 등)
RPC를 `auth.uid()` 기준 `viewer_lookup_admins` 화이트리스트로 잠갔다. 붙여넣을 비밀값이 없다.

## prune_viewer_crm이 후원 이력을 지우지 않는지 확인함
기존 보존 정책이 viewer_crm 행을 지우고 있어서 이 기능의 원본이 깎이는지 확인했다.
삭제 조건이 `visit_count <= 1 AND gift_count = 0 AND total_diamond = 0`이라 후원 이력이 있는 행은 안 지운다.
다만 `profile_pic_url`은 7일 지나면 비워지므로 오래된 시청자는 아바타가 안 보일 수 있다(기능 영향 없음).

## bump RPC 오염 방어
`fn_viewer_rollup_bump`은 anon도 호출할 수 있다(전 유저 앱이 올려야 하므로). 대신
viewer_crm의 `crm insert sane` 정책과 같은 수준의 길이/범위 검사를 함수 안에 넣었다 —
broadcaster_id·unique_id 1~64자, gift_name·nickname 200자 절단, cnt ≤ 100,000, coin ≤ 10,000,000,
배열 500개 초과 거부. 실측으로 과장 입력이 0건 처리되는 것 확인.
추가로 `viewer-rollup.ts`는 로그인 세션이 있으면 사용자 JWT로 보낸다(없으면 anon 키 폴백).
오염이 생기면 누가 올렸는지 추적·차단할 여지를 남기기 위함이다.

## 스트릭 중복은 이미 tiklive에서 정리됨
연속 선물(streak) 이벤트를 그대로 세면 선물 횟수가 부풀지만, `tiklive.ts`가 streakKey로
직전 repeat을 빼서 effectiveRepeat만 emit하고 `isStreakEnd`를 true로 고정한다(중복은 emit skip).
따라서 EventBus를 구독하는 롤업/CRM 모두 중복 없이 받는다. 별도 필터 불필요.

## admin 판정 타이밍
`fn_viewer_lookup_is_admin`은 main 프로세스 Supabase 세션의 JWT로 판정한다.
세션 복원이 페이지 마운트보다 늦으면 첫 조회가 false가 되어 버튼이 영영 안 뜬다.
그래서 `useIsLookupAdmin`은 true가 될 때까지 2초 간격 5회 재시도하고, 창 포커스 때도 다시 본다.

## 결정 5 — 노출은 dev 빌드 전용 (2026-08-18, K 지시)
조회 UI는 `pnpm dev`에서만 뜬다. 렌더러는 `import.meta.env.DEV`, main IPC는 NODE_ENV 검사로 이중 차단.
프로덕션 번들 실측에서 렌더러 쪽 패널 문자열·브릿지 호출이 전부 사라진 것을 확인했다.
**수집은 배포 빌드에서도 돈다** — 전 유저 앱이 올려야 크로스 호스트 숫자가 생기기 때문이다.

## 검색 성능 — 부분 trgm 인덱스
`_`가 ILIKE 단일문자 와일드카드라 `___c__k___` 검색이 매칭 폭발로 timeout(500)을 냈다. 입력을 이스케이프해 리터럴로 바꿨다.
그 뒤에도 305k행 full scan이라 7.7초가 걸려, **후원 이력 있는 행(17k, 5.6%)만** 대상으로 하는 부분 GIN trgm 인덱스를 걸었다.
전체 색인은 169MB 표에 GIN을 얹는 셈이라 무료 한도에 부담이고, 후원 0인 시청자는 조회해도 보여줄 내역이 없다.
결과: 2글자 이상 0.3~0.5초. 1글자는 trigram을 못 써 6초 → 서버·UI 양쪽에 2글자 하한을 뒀다.

## 리터럴 NUL 함정
`viewer-rollup.ts`의 맵 키 구분자로 리터럴 NUL 문자가 소스에 박혀 git이 파일을 바이너리로 취급했다(diff 불가).
`\u0000` 이스케이프 시퀀스로 바꿔 해결. 동작은 동일하다.

## 순서 노출
K 지시: "그냥 틱두와 똑같이". admin은 실제 내림차순, 그 외는 셔플. 단 조회 자체가 admin 전용이라
셔플 분기는 사실상 방어용으로만 남는다.

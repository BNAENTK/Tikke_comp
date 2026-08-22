# 계정 차단 — 결정 사항과 이유

## 배경

차단 목록이 `apps/worker/src/lib/auth.ts`의 `BLOCKED_EMAILS` Set에 하드코딩돼 있어, 한 명 추가/해제할 때마다 커밋 + 워커 배포가 필요했음. `/vip` 라우트는 이미 KV로 런타임 관리 중이라 같은 패턴을 차단 목록에도 적용.

## 설계 결정

**하드코딩 목록을 KV로 옮기지 않고 병합한다.** 기존 목록은 이미 배포돼 동작 중이고, 옮기는 순간 배포 타이밍에 따라 차단이 잠깐 풀릴 수 있음. 하드코딩 = 영구 차단, KV = 운영 차단으로 역할을 나눔.

**KV 조회 실패는 통과(fail open) 처리.** KV 장애 시 예외를 차단으로 해석하면 전체 사용자가 403으로 로그인 불가가 됨. 차단 대상 한 명을 잠시 놓치는 쪽이 전체 서비스 중단보다 나음. `isBlockedEmail`의 catch가 `false`를 반환하는 이유.

**`/blocked`에는 공개 check 엔드포인트를 두지 않음.** `/vip/check`는 공개인데, 차단 목록에 같은 걸 두면 차단 대상이 자기 이메일 상태를 조회해 우회 계정 생성 시점을 즉시 판단할 수 있음. 전 엔드포인트 `ADMIN_SECRET` 전용.

**`isBlockedEmail`을 async로 변경.** KV 조회가 필요해서 불가피. 호출부가 `requireAuth` 한 곳뿐이라 파급 없음. 인증 요청마다 KV read 1회가 추가되지만 엣지 캐시라 비용은 미미.

## 변경 파일

- `apps/worker/src/lib/auth.ts` — `BLOCKED_KV_KEY` export, `isBlockedEmail` async + KV 병합, `requireAuth`에서 await
- `apps/worker/src/routes/blocked.ts` (신규) — 관리자 전용 list/add/remove
- `apps/worker/src/index.ts` — `/blocked`, `/blocked/list` 라우팅

## 사용법

```
GET    /blocked/list   Authorization: Bearer <ADMIN_SECRET>
POST   /blocked        body {"email":"x@y.com"}
DELETE /blocked        body {"email":"x@y.com"}
```

KV 키는 `TIKKE_SETTINGS` 네임스페이스의 `blockedEmails` (문자열 배열 JSON).

## 주의

`lunamobe98@gmail.com`은 하드코딩 쪽에 들어감(요청 시점에 KV 경로가 아직 없었음). 배포 후 런타임 관리로 옮기려면 코드에서 빼고 `POST /blocked`로 다시 넣어야 함 — 순서를 바꾸면 차단이 끊김.

## 차단은 **두 계층 + 세션 정리**를 항상 같이 한다 (2026-08-14)

KV(또는 하드코딩) 목록만 넣으면 **워커 API 만** 막힌다. 앱 로그인은 Supabase 가 직접 처리하므로
계정이 살아 있으면 그 사람은 앱을 계속 쓴다. 실제로 `jhtps2@gmail.com` 은 KV 에 들어 있었는데도
`auth.users.banned_until` 이 null 이라 로그인이 됐고, 그래서 K 가 "차단하라고" 를 다시 말해야 했다.

빠뜨리기 쉬운 게 하나 더 있다 — **ban 을 걸어도 이미 발급된 세션/리프레시 토큰은 살아 있다.**
`kimgwangsu910@gmail.com` 은 하드코딩 목록에 있으면서 세션이 1건 붙어 있었다.

그래서 차단 요청이 오면 세 가지를 같이 한다:

1. 워커 목록 — 런타임은 `POST /blocked` (KV), 영구는 `auth.ts` 하드코딩.
2. Supabase 계정 정지 — `update auth.users set banned_until = '2999-12-31 00:00:00+00'`.
3. 기존 접속 끊기 — `auth.sessions`, `auth.refresh_tokens` 에서 그 user_id 삭제.
   (`refresh_tokens.user_id` 는 **varchar** 라 uuid 와 비교하려면 `::text` 캐스팅이 필요하다.)

확인 쿼리(차단자 상태 한눈에):

```sql
select u.email, u.banned_until,
       (select count(*) from auth.sessions s where s.user_id = u.id) as sessions,
       (select count(*) from auth.refresh_tokens r where r.user_id = u.id::text) as refresh_tokens
from auth.users u where lower(u.email) in ('...') order by u.email;
```

현재 차단 상태(전부 ban + 세션 0): jhtps2, kimgwangsu910, lunamobe98.

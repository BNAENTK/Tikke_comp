# context-notes — 앵커 배지 TikTok 아이디 자동 인식

## 2026-07-28

목표: 앵커 리그 배지 설정의 TikTok 아이디 입력창이 대시보드에서 연결한 아이디와 항상 같아지게 한다.

### 원래 상태
`EventOverlay.tsx`에 이미 자동 동기화 코드가 있었다.

```tsx
const liveUsername = useLiveStore((s) => s.username);
useEffect(() => {
  if (!liveUsername) return;
  ...
  void saveSetting("anchorBadgeUsername", clean);
}, [liveUsername, saveSetting]);
```

동작하지 않는 경우가 두 가지였다.

### 구멍 1 — 자동 연결 시 렌더러 스토어에 아이디가 안 들어감
`App.tsx`의 라이브 상태 구독이 이렇게 돼 있었다.

```tsx
const unsub = tikke.live.onStatus(({ status, error }) => {
  setLiveStatus(status, error);
  if (status !== "connected") setLiveUsername(null);
});
```

`onStatus` 페이로드에는 `username`이 없다(`{status, error}`뿐). 그래서 **연결됐을 때 username을 채우는 경로가 없다**. 부팅 직후 1회 `getStatus()`로 채우는 건 있지만(337행), 그 이후 `live-monitor`가 자동 연결하면 상태만 `connected`로 바뀌고 username은 null로 남는다. 자동 연결은 main에서 일어나 렌더러의 `setUsername`을 거치지 않기 때문이다.

수동 연결은 `Dashboard.handleConnect`가 직접 `setUsername`을 부르므로 문제가 없었다. 즉 "수동으로 연결하면 되는데 자동 연결이면 안 되는" 비대칭이었다.

수정: `connected` 이벤트에서 main의 `getStatus()`를 다시 읽어 username을 채운다. 스토어 전체가 고쳐지므로 앵커 배지 외에 `liveStore.username`을 읽는 모든 곳이 같이 이득을 본다.

### 구멍 2 — 미연결 상태에서 폴백이 없음
`liveUsername`이 null이면 동기화 이펙트가 그냥 빠져나간다. 앱을 켜고 아직 연결하지 않은 상태에서 앵커 배지 설정을 열면 입력창이 비어 있었다.

대시보드는 같은 상황에서 `lastConnectedUsername` 설정을 입력창에 미리 채운다(`Dashboard.tsx:169`). 이 키는 `live-connect.ts:18`이 **연결 시도 시점**에 저장하므로 수동/자동 연결을 모두 커버한다. 앵커 배지도 같은 값을 쓰면 "대시보드와 동일"이 성립한다.

수정 위치를 `settings.getAll()` 콜백 안으로 잡은 게 핵심이다.

```tsx
tikke?.settings?.getAll().then((cfg) => {
  const last = String(cfg.lastConnectedUsername ?? "").replace(/^@/, "").trim();
  const fillAnchor = !String(cfg.anchorBadgeUsername ?? "").trim() && !!last;
  setConfig(fillAnchor ? { ...cfg, anchorBadgeUsername: last } : cfg);
  if (fillAnchor) void tikke?.settings?.set("anchorBadgeUsername", last);
  ...
```

별도 이펙트로 뺐다면 `config` state가 `{}`로 시작하는 탓에 설정 로드 전에 이펙트가 돌아 **저장돼 있던 값을 빈 값으로 오판하고 덮어쓸** 수 있다. `cfg`를 손에 쥔 시점에서 판단하면 그 레이스가 없다.

`getAll()`이 이미 모든 설정을 주므로 IPC 호출을 추가하지 않았다.

### 덮어쓰기 규칙
- 라이브 연결 중이면 연결된 계정이 우선이고 기존 값을 덮어쓴다. "연결한 아이디와 동일"이 요청이므로 의도된 동작이다.
- 미연결이면 **입력창이 비어 있을 때만** 채운다. 오프라인에서 다른 계정을 보려고 직접 입력해 둔 값을 지우지 않기 위해서다.

안내 문구도 "연결한 계정으로 자동 입력됨. 다른 계정을 보려면 직접 수정"으로 바꿨다.

### 남은 제약
아이디는 오버레이 URL의 쿼리 파라미터로 들어간다(`anchor-badge?username=...`). 그래서 아이디가 바뀌면 **이미 TTLS에 등록해 둔 브라우저 소스는 URL을 다시 복사해야 한다**. 테마/표시 항목처럼 `anchor-badge-config` postMessage로 라이브 push되는 값이 아니다. 미리보기 iframe은 config에서 src를 만들어 즉시 반영된다.

계정을 자주 바꾸는 사용자라면 username을 URL이 아니라 config push로 옮기는 게 맞지만, 요청 범위 밖이라 하지 않았다.

### 검증
- `pnpm --filter @tikke/desktop typecheck` 통과
- `pnpm build:desktop` 통과
- 실기기 확인 필요: ①미연결로 앱 실행 → 앵커 배지 설정 열면 마지막 연결 아이디가 채워져 있는지 ②자동 연결 후 설정 열면 그 계정으로 바뀌는지

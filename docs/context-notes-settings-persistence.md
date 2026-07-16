# 설정 초기화(업데이트 후 1회 전부) 수정 — 컨텍스트 노트

날짜: 2026-07-09. 증상: 사용자가 업데이트하면 기존 설정 전부 초기화(업데이트 후에만, 1회, 전부).

## 원인 분석

설정은 전부 `%APPDATA%\Tikke\tikke.db` (sql.js) 단일 소스. 조사로 제거한 용의자 —
userData 경로 변동(productName/appId 첫 커밋 이후 불변), 백엔드 교체(없음), 버전 게이트
리셋(없음), 클라우드 pull 전부덮기(push 안 한 유저는 `{}` 반환이라 불가), localStorage
origin(설정 없음), 인스톨러 appData 삭제(설정 없음).

**확정 메커니즘 (실증됨)**: 구 `persist()`가 `writeFileSync(dbPath)` 제자리 덮어쓰기.
업데이트 설치(quitAndInstall) 등으로 쓰기 도중 강제 종료 → 0바이트/잘린 파일.
sql.js는 **0바이트 버퍼를 빈 DB로 조용히 열고** → `CREATE TABLE IF NOT EXISTS` +
기본값 시드 → 10초 flush가 새 빈 DB를 영속화 → "업데이트 후 딱 1회 전부 초기화,
이후 정상" 증상과 정확히 일치. 스크립트로 재현 확인(0바이트 → 빈 DB 무예외).

## 수정 내역 (apps/desktop)

1. `services/db.ts`
   - `persist()` 원자적 쓰기: `.tmp`에 완성 후 `renameSync` → 도중 강제종료에도 원본 보존.
   - `initDb()` 로드 하드닝: `tryLoadDbFile()` — 0바이트/파싱 실패/빈 스키마(app_settings 0건)
     전부 거부. 손상 파일은 `tikke.db.corrupt-<ts>`로 보존 후 복구 시도.
   - 복구 순서: 본 파일 → `tikke.db.bak`(userData) → `%APPDATA%\Tikke-backup\tikke.db.bak`(외부)
     → 전부 실패 시에만 새 DB(freshlyCreated 플래그).
   - `backupNow()`: 종료 시(close) + 업데이트 설치 직전 2곳에 백업 기록.
   - `dbWasFreshlyCreated()` export — 클라우드 복원 게이트용.
2. `services/updater.ts` — `installUpdate()`에서 quitAndInstall 직전 `persistNow()+backupNow()`.
3. `main/index.ts` — 시작 시 클라우드 자동 pull을 `dbWasFreshlyCreated()`일 때만 실행.
   기존 로컬 데이터를 시작 시 pull로 덮어쓰던 잠재 클로버 버그 수정 + 초기화 발생 시
   클라우드에서 자동 복원으로 전환. `setOnSettingChanged → schedulePushToCloud` 배선.
4. `services/cloud-sync.ts` — `schedulePushToCloud()` 30초 디바운스 자동 push 추가
   (클라우드에 항상 최신 사본 = 복구 소스). 미로그인 시 조용히 무시.
5. `services/settings.ts` — `setSetting()` 변경 훅 `setOnSettingChanged` 추가
   (cloud-sync 직접 import 시 순환 의존이라 콜백 주입).

## 2026-07-10 추가 — 진짜 주범은 getSetting-before-initDb (1.8.150 자동실행 커밋)

위 파일 잘림은 실존 리스크였지만, 사용자 보고("자동실행 배포 이후부터 초기화")로 재조사한
결과 **주범은 53f1d5c(v1.8.150) 자동실행 커밋**. 체인 —

1. `index.ts` whenReady 맨 위에서 `getSetting("autoLaunch")` 호출 (initDb보다 먼저).
2. `getDb()`가 instance 없음 → `InMemoryFallback` 생성해 instance 선점.
3. `initDb()`는 `if (instance) return` 조기 반환 → **디스크 tikke.db를 영영 안 읽음**.
4. 매 실행 전부 기본값 + 세션 중 변경 디스크 미반영. 1.8.150~152 패키지 빌드 전부 해당.
5. dev 모드는 `app.isPackaged && getSetting(...)` 단락평가로 getSetting 미실행 → 미재현.

InMemoryFallback은 디스크를 안 쓰므로 tikke.db는 1.8.150 이전 상태로 온전 → 이 수정
배포 시 사용자 설정 자동 복귀. 단 1.8.150~152 세션에서 바꾼 설정은 메모리에만 있었으므로 소실.

수정(2026-07-10) —
- `main/index.ts`: autoLaunch 체크를 `await initDb()` 뒤로 이동.
- `services/db.ts`: `initDb()`가 InMemoryFallback을 감지하면 실제 SQLite로 교체
  (재발 방지 — 어떤 코드가 initDb 전에 getDb를 불러도 이제 실제 DB가 이김).
  fallback에 쌓인 pre-init 쓰기는 `exportSettings()`로 실제 DB에 이관.
- 검증: `pnpm --filter @tikke/desktop typecheck` 통과.

리스크 노트: 1.8.152의 `schedulePushToCloud` 자동 push가 fallback 상태(=기본값)에서
동작했으므로, 1.8.152에서 설정을 만진 로그인 유저는 클라우드 사본이 기본값으로
덮였을 수 있음. pull은 freshlyCreated일 때만이라 로컬 복귀에는 영향 없음.

## 남은 사실 / 한계

- 이미 초기화당한 유저: 구버전이 빈 DB를 원본 위에 영속화했으므로 로컬 복구 불가.
  수동으로 클라우드 push 해뒀던 유저만 pull로 복구 가능. 이번 수정 배포 후부터는
  (a) 원자적 쓰기로 손상 자체가 안 생기고 (b) 생겨도 백업 2곳 + 클라우드에서 자동 복원.
- 동일 패턴 전수조사: 치명 상태 파일은 tikke.db 유일. 나머지 writeFileSync는
  사용자 내보내기(ipc.ts)·재생성 가능 캐시(tiklive gift cache)·qwen 보이스 파일로 무관.
- 검증: `pnpm --filter @tikke/desktop typecheck` 통과. sql.js 가드 3케이스
  (0바이트 REJECT / 잘림 REJECT / 정상 OK) 스크립트 실증.
- 배포는 사용자 지시 대기 (자동 배포 금지).

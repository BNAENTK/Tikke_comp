# Tikke 프로젝트 Claude Code 컨텍스트

## 프로젝트 개요
Tikke는 TikTok LIVE 방송인을 위한 Windows 데스크탑 툴킷과 웹/Worker/오버레이를 포함한 pnpm 모노레포입니다. 이 Claude Code 구성의 목적은 새 프로젝트 생성이 아니라 기존 Tikke 코드베이스의 기능 추가, 버그 수정, UI 개선, 배포 전 검증을 돕는 것입니다.

## 언어 규칙
- 사용자가 별도로 요청하지 않는 한 설명은 한국어로 한다.
- 명령어, 코드 식별자, 파일명, 설정 키는 영어 원문을 유지한다.
- 최종 보고에는 변경 파일, 변경 이유, 검증 결과를 한국어로 요약한다.

## 기술 스택 / 구조
- 패키지 매니저: `pnpm`
- 언어: TypeScript, JavaScript, Python 일부 서비스
- 데스크탑 앱: `apps/desktop` — Electron, electron-vite, React, Zustand, TikTok LIVE connector, Supabase, sql.js, TTS/overlay/harness
- 웹사이트: `apps/web` — Vite, React, TypeScript, Supabase, Cloudflare Pages 배포 대상
- Worker: `apps/worker` — Cloudflare Workers, Wrangler
- 공유 패키지: `packages/shared` — 공통 타입/유틸
- TTS 서비스: `services/qwen-tts` — Python 로컬 TTS 서버 관련 코드
- 문서: `docs/` — `SETUP.md`, `SUPABASE.md`, `CLOUDFLARE.md`, `BUILD.md`, `HARNESS.md`, `SECURITY.md`

## 주요 명령어
루트 `tikke` 폴더에서 실행한다.

- `pnpm install` — 의존성 설치
- `pnpm dev` — `@tikke/desktop` 개발 실행
- `pnpm dev:web` — `@tikke/web` 개발 실행
- `pnpm dev:worker` — `@tikke/worker` 개발 실행
- `pnpm build` — 전체 빌드
- `pnpm build:desktop` — 데스크탑 앱 빌드
- `pnpm dist:win` — Windows 설치 파일 빌드
- `pnpm lint` — 전체 lint
- `pnpm typecheck` — 전체 타입 체크
- `pnpm test` — 전체 테스트
- `pnpm harness:chat` — 채팅 이벤트 하네스
- `pnpm harness:gift` — 선물 이벤트 하네스
- `pnpm harness:tts` — TTS 큐 하네스
- `pnpm harness:overlay` — 오버레이 하네스
- `pnpm harness:stress` — 스트레스 하네스
- `pnpm --filter @tikke/desktop typecheck` — 데스크탑 타입 체크
- `pnpm --filter @tikke/web build` — 웹 빌드
- `pnpm --filter @tikke/worker typecheck` — Worker 타입 체크

## 작업 원칙
- 변경 전에 관련 파일을 먼저 찾고 기존 구조와 패턴을 이해한다.
- 기존 아키텍처, 파일 배치, 네이밍, 스타일을 최대한 유지한다.
- 사용자가 요청하지 않은 대규모 리팩토링이나 새 라이브러리 도입은 피한다.
- 기능 추가 시 `apps/desktop`, `apps/web`, `apps/worker`, `packages/shared` 중 영향 범위를 먼저 분리한다.
- 공통 타입/유틸은 가능하면 `packages/shared`에 둔다.
- 데스크탑 앱 변경은 Electron main/preload/renderer 경계를 확인한다.
- 웹/오버레이 변경은 모바일 반응형, Cloudflare Pages 정적 배포, 브라우저 소스 사용성을 고려한다.
- Worker 변경은 `wrangler.toml`, 라우팅, CORS, 보안 헤더, secret 노출 가능성을 확인한다.
- `.env`, API key, token, secret 값은 절대 출력하지 않는다.
- Windows/OneDrive/한글 경로 환경을 고려한다. 경로는 가능하면 따옴표로 감싼다.

## 검증 원칙
변경 범위에 맞춰 가능한 가장 작은 검증부터 실행한다.

- 데스크탑 코드: `pnpm --filter @tikke/desktop typecheck`
- 웹 코드: `pnpm --filter @tikke/web build`
- Worker 코드: `pnpm --filter @tikke/worker typecheck`
- 공유 패키지: `pnpm --filter @tikke/shared typecheck`
- 전체 영향 가능성: `pnpm typecheck`, 필요 시 `pnpm build`
- 이벤트/TTS/오버레이 관련 변경: 관련 `pnpm harness:*` 명령어 실행 또는 실행 방법 제시

## 기능 추가 절차
1. 요청 기능이 데스크탑, 웹, Worker, shared, TTS 서비스 중 어디에 속하는지 분류한다.
2. 관련 기존 파일을 검색하고 현재 구현 방식을 요약한다.
3. 최소 변경 계획을 세운다.
4. 필요한 파일만 수정한다.
5. 타입/빌드/하네스 중 적절한 검증을 실행한다.
6. 변경 파일, 동작 방식, 검증 결과, 남은 위험을 보고한다.

## 버그 수정 절차
1. 증상과 재현 조건을 정리한다.
2. 관련 상태 흐름, API 호출, 이벤트 흐름, 로그 지점을 추적한다.
3. 원인을 먼저 설명하고 최소 수정으로 해결한다.
4. 회귀 가능성이 있는 영역을 확인한다.
5. 가능한 검증 명령어를 실행한다.

## UI 개선 절차
1. 대상 페이지/컴포넌트를 찾는다.
2. 기존 디자인 시스템, CSS, Tailwind/className 패턴을 확인한다.
3. 데스크톱/모바일 반응형과 접근성을 같이 고려한다.
4. 전체 디자인을 갈아엎지 말고 요청 범위 안에서 개선한다.
5. 빌드 또는 타입 체크로 검증한다.

## 보안 주의사항
- Supabase Auth, OAuth, Worker endpoint, GitHub release, Cloudflare, TTS 외부 API 관련 변경은 특히 보수적으로 처리한다.
- secret은 `.env`/대시보드에 두고 코드에 하드코딩하지 않는다.
- 사용자 입력, room/username, overlay query parameter, admin 기능은 검증과 권한 체크를 고려한다.

# Tikke 작업 기록

`tikke` 관련 중요 작업 요약을 남기는 노트.

## 기록 원칙

- 날짜별로 짧게 남긴다.
- 결정, 변경, 검증 결과 중심으로 남긴다.
- 소스코드 원문과 secret은 남기지 않는다.

## 기록

### 2026-06-05 21:13
- 로컬 `tikke` 대시보드/오버레이 테스트 서버 실행 방법을 확인했다.
- 루트 기준 실행 명령은 `pnpm dev`이며, `@tikke/desktop`의 `electron-vite dev`로 연결된다.
- 오버레이 확인 포트는 `18181`, WebSocket 포트는 `18182`로 확인했다.
- 테스트 서버 미실행 상태에서는 `live-pet` 오버레이 미리보기가 보이지 않을 수 있음을 확인했다.
- Obsidian 볼트 `C:/Users/zkfma/Documents/BNAK_Knowledge` 연결과 Tikke 프로젝트 노트 존재를 확인했다.
- `40_Skills` 영역을 비어 있는 상태에서 스킬 인덱스, 개별 요약 노트, 카테고리별 묶음 노트 구조로 확장했다.
- GitHub / Research / DevOps / Creative / Productivity / Tikke 운영 기준으로 스킬 탐색 동선을 정리했다.
- Agents & Autonomous / Media / Software Development 카테고리까지 추가 요약 및 묶음 노트를 생성했다.

### 2026-06-05 21:14
- Obsidian `40_Skills` 영역이 비어 있는 이유를 확인했다.
- 실제 Hermes 스킬은 Obsidian이 아니라 Hermes 스킬 시스템에 저장되며, 조회 가능한 스킬 수는 78개였다.
- `40_Skills.md`와 `Hermes 스킬 인덱스.md`를 채우고, 자주 쓰는 스킬 11개의 한국어 요약 노트를 자동 생성했다.

### 2026-06-05 21:15
- 이어서 productivity 계열 주요 스킬 요약 5개(`Google Workspace`, `Notion`, `PowerPoint`, `Maps`, `OCR and Documents`)를 추가 생성했다.
- `Productivity 스킬 묶음`과 `Tikke 운영 스킬 묶음` 노트를 만들어 실제 사용 흐름 기준으로 스킬 연결 구조를 보강했다.
- GitHub / Research / DevOps / Creative / Agents & Autonomous / Media / Software Development 카테고리 요약과 묶음 노트를 확장했다.
- MCP / MLOps / Email / Smart Home / Note Taking 카테고리까지 추가로 요약하고, `BNAK Knowledge Index`에 스킬 바로가기를 연결했다.
- 남은 희소 카테고리 중 `red-teaming`까지 정리해 `Godmode 스킬 요약`과 `Red Teaming 스킬 묶음`을 추가했다.
- 주요 묶음 노트에 `이럴 때 바로 보기` 섹션과 연결 노트를 추가해 탐색 동선을 더 짧게 만들었다.
- 남은 묶음 노트들에도 같은 형식을 적용해 묶음 문서 포맷을 거의 통일했다.
- `Hermes 스킬 인덱스`, `40_Skills`, `BNAK Knowledge Index`까지 시작점/즐겨찾기/분야별 바로가기로 재구성했다.
- `Tikke 버그-원인 기록`, `Tikke Claude Code 작업 패턴` 노트를 새로 만들고, `Tikke 오버레이 규칙`과 `Tikke Index` 연결을 보강했다.
- `Tikke Claude Code 프롬프트 템플릿` 노트를 추가하고, 기능 추가/버그 수정/오버레이/리팩터링/PR 정리 템플릿을 정리했다.
- 보이스 클로닝 + 저지연 TTS 적용 방향을 조사하고, `일반 읽기용 저지연 엔진`과 `클론 전용 엔진`을 분리하는 2단 구조를 우선 추천안으로 정리했다.
- 이 작업을 Claude Code에 맡길 때 쓸 전용 프롬프트 전략/템플릿 노트를 추가하고, 조사→스파이크→Hermes 통합 순서로 보내는 방식을 기준안으로 정리했다.

## 2026-06-09

- 결정: `Tikke_comp` 백업은 소스코드/비밀정보를 제외하고 문서와 Obsidian Markdown만 선별 동기화하는 방식으로 유지한다.
- 작업: `tikke` 문서/설계 파일 21개를 `Tikke_comp/docs/`에 재동기화하고 GitHub 원격에 푸시했다.
- 작업: Obsidian 볼트 Markdown 69개를 `Tikke_comp/obsidian/`에 재생성하고 GitHub 원격에 푸시했다.
- 확인: 문서 동기화는 `.venv`, `.claude`, `.kkirikkiri` 유입을 2차 정리했고, 최종 커밋은 `4147ef2`로 확인했다.
- 확인: Obsidian 동기화는 민감정보 패턴 파일 13개를 제외했고, 최종 커밋은 `6abebcc`로 확인했다.
- 다음: 자동 동기화의 민감정보 탐지 규칙이 필요한 노트까지 과도하게 제외하지 않는지 주기 점검한다.

## 2026-06-10

- 결정: `Tikke_comp` 동기화는 문서/Obsidian Markdown만 선별하고 소스코드·비밀정보·앱 설정은 제외하는 방식을 계속 유지한다.
- 작업: `tikke` 문서/설계 파일 41개를 `Tikke_comp/docs/`에 동기화하고 GitHub 원격에 푸시했다.
- 작업: Obsidian 볼트 Markdown 82개를 `Tikke_comp/obsidian/`에 동기화하고 GitHub 원격에 푸시했다.
- 확인: 문서 동기화는 소스코드 확장자 및 비밀키/토큰 패턴 위반 0건, 커밋 `0be8c15`로 확인했다.
- 확인: Obsidian 동기화는 민감정보 패턴 제외 0건, `.obsidian`/`.trash`/백업 파일 미복사, 커밋 `3c974e7`로 확인했다.
- 다음: `Tikke_comp` 백업 자동화가 private repo 전제와 제외 규칙을 계속 지키는지 주기 점검한다.

## 2026-06-13

- 결정: `Tikke_comp` 백업은 문서/Obsidian Markdown만 선별하고 소스코드·비밀정보·앱 설정은 제외하는 정책을 유지한다.
- 작업: `tikke` 문서/설계 파일 325개를 `Tikke_comp/docs/`에 동기화하고 GitHub 원격에 푸시했다.
- 작업: Obsidian 볼트 Markdown 82개를 `Tikke_comp/obsidian/`에 동기화하고 GitHub 원격에 푸시했다.
- 확인: 문서 동기화는 제외 디렉토리 17건, 소스코드 확장자 23302건을 스킵했고 안전 검증 위반 0건, 커밋 `58558fa`로 확인했다.
- 확인: Obsidian 동기화는 민감정보 감지 0건, `.obsidian`/`.trash`/non-md 미복사, 커밋 `4846d10`로 확인했다.
- 다음: 자동 동기화 제외 규칙이 필요한 문서 누락 없이 동작하는지 주기 점검한다.


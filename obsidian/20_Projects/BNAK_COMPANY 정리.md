# BNAK_COMPANY 정리

> GitHub: https://github.com/BNAENTK/BNAK_COMPANY
> 정리일: 2026-06-04

## 한 줄 요약

`BNAENTK/BNAK_COMPANY`는 `Connect AI v2 (P-Reinforce)` 기반의 100% 로컬/오프라인 AI 코딩 에이전트 프로젝트입니다. VS Code, Cursor, Antigravity 환경에서 파일 생성, 코드 편집, 터미널 실행, Markdown 지식 구조화, GitHub 동기화를 목표로 합니다.

## 핵심 키워드

- 100% Local
- 100% Offline
- Autonomous Knowledge Engine
- VS Code / Cursor Extension
- Ollama / LM Studio
- P-Reinforce Architecture
- Agent University
- Markdown Wiki
- Auto-Git Sync

## 프로젝트 정체성

- 이름: Connect AI v2 (P-Reinforce)
- 저장소: `BNAENTK/BNAK_COMPANY`
- 원본 fork: `wonseokjung/connect-ai`
- 라이선스: MIT
- 버전: `2.1.30`
- 주요 언어: TypeScript
- 설명: 로컬 AI 코딩 에이전트. Antigravity, VS Code, Cursor에서 오프라인으로 파일 생성, 코드 편집, 터미널 실행을 수행. Ollama 기반.

## 핵심 개념: P-Reinforce Architecture

README는 이 프로젝트를 단순 코딩 에이전트가 아니라 **자율 지식 정원사(Autonomous Gardener)**로 설명합니다.

주요 역할:

- 사용자 정보와 지시를 받음
- 의미를 자동 분석
- 폴더를 자동 생성
- Markdown wiki 파일로 정리
- GitHub에 자동 백업

Obsidian 관점에서는 이 구조가 vault 기반 지식관리와 잘 맞습니다.

## 주요 기능

### 1. Agent University 연동

Agent University 웹 플랫폼과 실시간 통신하는 구조입니다.

- 로컬 VS Code 포트: `4825`
- 로컬 AI brain 경로: `~/.connect-ai-brain`
- 웹 버튼으로 Premium Brain Pack 지식을 로컬 AI brain에 주입하는 개념

### 2. 자율 지식 구조화

사용자가 던지는 원시 데이터를 에이전트가 자동으로 구조화합니다.

예시 폴더:

```text
10_Wiki
00_Raw
🚀 Skills
```

우리 Obsidian vault와 대응:

```text
00_Inbox      임시 수집
10_Wiki       정리된 지식
20_Projects   프로젝트 문서
30_Automation 자동화 기록
40_Skills     스킬/사용법
90_Archive    보관
```

### 3. Auto-Git Sync

파일 생성 시 자동으로 GitHub 동기화를 수행하는 개념입니다.

```bash
git add
git commit
git push
```

주의: 실제 자동 push는 위험할 수 있으므로 승인 기반으로 운영하는 것이 좋습니다.

### 4. 동적 모델 감지

Ollama 또는 LM Studio에 설치된 모델을 내부 API로 감지합니다.

```text
v1/models
```

UI의 모델 선택 드롭다운과 연결하는 방식입니다.

## 에이전트 권한

README 기준 권한:

| 권한 | 설명 |
|---|---|
| Create Files | 새 파일/폴더 생성 |
| Edit Files | 기존 코드 수정 |
| Delete Files | 불필요 파일 삭제 |
| Read Files | 프로젝트 파일 읽기 |
| Browse Directories | 디렉토리 구조 분석 |
| Run Commands | 빌드, git push 등 터미널 실행 |

## 설치/엔진

### VSIX 설치

1. Releases에서 `v2.1.30.vsix` 다운로드
2. VS Code 명령 팔레트 실행
3. `Extensions: Install from VSIX`
4. 파일 선택

### LM Studio 권장

- `lmstudio.ai` 설치
- Gemma 3, Llama 3, Qwen Coder 등 모델 로드
- Developer 탭에서 서버 시작
- Connect AI 설정에서 LM Studio 선택

### Ollama

```bash
brew install ollama
ollama pull gemma3
```

## 보안/프라이버시

README는 다음을 강조합니다.

- Zero Cloud API
- Zero Telemetry
- 100% Local Inference
- 외부 클라우드로 코드가 나가지 않는 구조

## Obsidian 연결 아이디어

이 프로젝트는 Markdown 지식 구조화를 핵심으로 하므로 Obsidian vault와 연결하기 좋습니다.

가능한 자동화:

- GitHub README를 `20_Projects`에 자동 저장
- 프로젝트 기능별 노트를 `10_Wiki`로 분리
- 작업 로그를 `30_Automation`에 저장
- 스킬/명령어를 `40_Skills`에 정리
- `[[BNAK_COMPANY 정리]]` 같은 wikilink로 연결

## 다음 작업 후보

- [ ] 저장소 파일 구조 전체 스캔
- [ ] README 외 주요 소스 파일 요약
- [ ] `P-Reinforce` 개념을 별도 위키 노트로 분리
- [ ] `Agent University` 노트 생성
- [ ] `Auto-Git Sync` 안전 운영 규칙 작성
- [ ] 대시보드에서 GitHub repo → Obsidian 노트 자동 저장 버튼 추가

## 관련 노트

- [[BNAK Knowledge Index]]
- [[P-Reinforce Architecture]]
- [[Agent University]]
- [[Auto-Git Sync]]

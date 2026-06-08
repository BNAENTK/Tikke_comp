# Claude Code Workspace Orchestration 스킬 요약

## 용도
- 여러 로컬 프로젝트가 들어 있는 작업공간 전체를 Claude Code 기준으로 운영할 때 쓰는 스킬.

## 핵심 포인트
- 루트 작업공간과 하위 프로젝트를 분리해서 본다.
- 루트에는 workspace router/공통 에이전트, 프로젝트 내부에는 전용 에이전트를 둔다.
- 사용자가 특정 프로젝트를 안 짚었으면 수정 전 대상 프로젝트를 다시 확인해야 한다.

## 자주 쓰는 흐름
1. workspace root 식별
2. 하위 프로젝트 분류
3. 루트 `CLAUDE.md`와 공통 agent/command 설계
4. 프로젝트별 전용 문서/에이전트 연결

## 주의사항
- 아카이브/로그 폴더를 실수로 건드리면 안 된다.
- workspace용 agent와 project용 agent를 혼동하기 쉽다.
- Windows 셸 안내는 cmd / bash 차이를 구분해야 한다.

## 연결 노트
- [[Claude Code 스킬 요약]]
- [[Tikke 운영 스킬 묶음]]

# GitHub PR Workflow 스킬 요약

## 용도
- 브랜치 생성부터 커밋, PR 생성, CI 확인, 머지까지 GitHub PR 전체 흐름을 정리한 스킬.

## 핵심 포인트
- `gh` CLI가 있으면 가장 편하고, 없으면 `git + curl` 대안도 가능하다.
- PR 작업 전에는 현재 remote에서 owner/repo를 정확히 뽑는 단계가 중요하다.
- CI 실패 시 로그 확인 → 수정 → push → 재확인 루프를 반복하는 운영 패턴이 있다.

## 자주 쓰는 흐름
- 브랜치 생성: `git checkout -b feat/...`
- push: `git push -u origin HEAD`
- PR 생성: `gh pr create ...`
- CI 확인: `gh pr checks --watch`
- 머지: `gh pr merge --squash --delete-branch`

## 주의사항
- gh가 없을 때도 curl 기반으로 대부분 처리 가능하다.
- CI auto-fix는 최대 몇 회까지 루프를 돌릴지 정해두는 편이 안전하다.
- 커밋/PR 메시지는 conventional commit 스타일이 관리하기 쉽다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[Tikke Claude Code 협업]]

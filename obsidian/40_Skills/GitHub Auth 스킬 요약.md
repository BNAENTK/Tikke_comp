# GitHub Auth 스킬 요약

## 용도
- GitHub 작업 전에 인증 상태를 잡는 스킬.
- `gh` CLI가 있든 없든, HTTPS token 또는 SSH 키 기준으로 작업을 이어갈 수 있게 한다.

## 핵심 포인트
- 먼저 `gh auth status` 또는 git credential 상태를 확인한다.
- `gh`가 없더라도 `git + curl + GITHUB_TOKEN` 조합으로 대부분의 GitHub 작업이 가능하다.
- 인증 문제를 해결하지 않으면 PR, issue, release, workflow 작업이 연쇄적으로 막힌다.

## 자주 쓰는 흐름
1. `gh` 설치/인증 여부 확인
2. 안 되어 있으면 PAT 또는 SSH 방식 선택
3. git identity 설정
4. 인증 검증 후 다른 GitHub 스킬 사용

## 주의사항
- GitHub 비밀번호는 직접 인증에 쓰지 않는다.
- PAT scope 부족이면 push/API 작업이 실패한다.
- 여러 계정 환경이면 HTTPS보다 SSH alias 전략이 더 낫다.

## 연결 노트
- [[GitHub Repo Management 스킬 요약]]
- [[GitHub PR Workflow 스킬 요약]]
- [[GitHub Code Review 스킬 요약]]
- [[GitHub Issues 스킬 요약]]

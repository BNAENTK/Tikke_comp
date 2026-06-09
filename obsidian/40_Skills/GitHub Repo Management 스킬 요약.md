# GitHub Repo Management 스킬 요약

## 용도
- clone, create, fork, remote, release, workflow, secret 등 repo 운영 전반을 다루는 스킬.

## 핵심 포인트
- `gh` 기준 흐름이 가장 편하지만, `curl` fallback도 충분히 강력하다.
- repo 작업 전에는 현재 remote의 `owner/repo`를 정확히 파악해야 한다.
- release, workflow, secret 관리까지 확장되는 운영형 스킬이다.

## 자주 쓰는 흐름
1. GitHub 인증 확인
2. remote에서 repo 식별
3. clone/create/fork 등 목적별 작업 수행
4. 필요 시 release/workflow/secret 단계로 확장

## 주의사항
- secret 관리는 `gh secret set` 쪽이 훨씬 단순하다.
- fork sync, default branch, topic, workflow 조작은 repo를 잘못 잡으면 사고 난다.
- 현재 repo인지 대상 repo인지 항상 명시적으로 구분해야 한다.

## 연결 노트
- [[GitHub Auth 스킬 요약]]
- [[GitHub PR Workflow 스킬 요약]]
- [[GitHub Issues 스킬 요약]]

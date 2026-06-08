# GitHub Issues 스킬 요약

## 용도
- GitHub issue 생성, 조회, triage, label, assign, comment, close/reopen 작업을 다루는 스킬.

## 핵심 포인트
- `gh issue` 기준이 가장 빠르지만, REST API fallback도 잘 정리돼 있다.
- issue triage는 단순 조회보다 label/priority/assignee/comment까지 포함한 운영 흐름이다.
- PR과 연동할 때 `Closes #42` 같은 키워드 자동 종료 패턴이 중요하다.

## 자주 쓰는 흐름
1. 현재 repo 식별
2. issue 목록/세부 내용 확인
3. triage: label, priority, assignee 정리
4. 필요 시 branch/PR 작업으로 연결

## 주의사항
- GitHub API의 `/issues`는 PR도 섞여 나올 수 있다.
- bulk close나 label 작업은 범위를 잘못 잡기 쉽다.
- repo를 명시하지 않으면 다른 저장소 이슈를 건드릴 위험이 있다.

## 연결 노트
- [[GitHub Auth 스킬 요약]]
- [[GitHub PR Workflow 스킬 요약]]
- [[GitHub Repo Management 스킬 요약]]

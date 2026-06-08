# Requesting Code Review 스킬 요약

## 용도
- 커밋 전 변경사항을 검증하는 pre-commit review 파이프라인 스킬.

## 핵심 포인트
- 구현한 주체와 검토 주체를 분리하는 것이 중요하다.
- diff 확인, 보안 패턴 스캔, 테스트/린트, 독립 reviewer, auto-fix loop 순으로 진행한다.
- 단순 스타일보다 보안/로직 오류/회귀 방지를 우선한다.

## 자주 쓰는 흐름
- `git diff --cached`
- 보안 패턴 grep
- 프로젝트 테스트/린트 실행
- reviewer subagent 호출
- 이슈 있으면 fix agent 호출 후 재검증

## 주의사항
- empty diff면 검토할 게 없는 상태다.
- large diff는 파일별로 쪼개서 보는 편이 안전하다.
- reviewer 결과가 애매하면 fail-closed 쪽으로 보는 게 낫다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[Subagent Driven Development 스킬 요약]]
- [[GitHub Code Review 스킬 요약]]

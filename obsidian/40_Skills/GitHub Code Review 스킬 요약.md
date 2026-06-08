# GitHub Code Review 스킬 요약

## 용도
- 로컬 변경사항이나 GitHub PR을 리뷰할 때 쓰는 스킬.

## 핵심 포인트
- 로컬 diff 검토와 GitHub PR 리뷰 둘 다 커버한다.
- `gh`가 있으면 편하고, 없으면 `git + curl`로도 충분히 가능하다.
- 단순 diff만 보지 말고 changed file의 전체 문맥도 읽는 것이 중요하다.

## 자주 쓰는 흐름
- `git diff main...HEAD --stat`
- 전체 diff 확인
- changed file별 문맥 읽기
- 필요 시 PR checkout
- inline comment 또는 review summary 제출

## 주의사항
- Critical / Warning / Suggestion으로 구분해서 보는 편이 좋다.
- 보안/정확성/테스트/성능/문서화 체크리스트를 함께 보는 것이 안정적이다.
- 리뷰 후 로컬 브랜치 cleanup까지 잊지 않는 게 좋다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[GitHub PR Workflow 스킬 요약]]
- [[Requesting Code Review 스킬 요약]]

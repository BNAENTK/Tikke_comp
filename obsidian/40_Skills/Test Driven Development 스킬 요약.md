# Test Driven Development 스킬 요약

## 용도
- 새 기능, 버그 수정, 리팩터링을 test-first 방식으로 진행할 때 쓰는 스킬.

## 핵심 포인트
- failing test 없이 production code를 쓰면 안 된다는 강한 규칙이 핵심이다.
- RED → GREEN → REFACTOR를 실제 테스트 실행으로 확인해야 한다.
- 테스트가 처음부터 통과하면 잘못된 테스트일 가능성이 높다.

## 자주 쓰는 흐름
1. 최소 failing test 작성
2. 실패 확인
3. 최소 코드로 통과
4. 전체 테스트 재실행
5. refactor

## 주의사항
- 구현 후 테스트 추가는 이 스킬 취지와 반대다.
- mock behavior 테스트에 빠지지 말아야 한다.
- 버그 수정도 재현 테스트 없이 고치면 회귀 방지가 안 된다.

## 연결 노트
- [[Systematic Debugging 스킬 요약]]
- [[Writing Plans 스킬 요약]]
- [[Subagent Driven Development 스킬 요약]]

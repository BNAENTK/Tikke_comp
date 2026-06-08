# Spike 스킬 요약

## 용도
- 본격 구현 전에 아이디어의 가능성, 접근 방식, 라이브러리 선택을 빠르게 검증하는 throwaway 실험 스킬.

## 핵심 포인트
- spike는 production code가 아니라 검증용이다.
- 2~5개의 독립 질문으로 쪼개면 좋다.
- 결과는 `VALIDATED / PARTIAL / INVALIDATED` 같은 verdict로 남겨야 의미가 있다.

## 자주 쓰는 흐름
1. 질문 분해
2. 짧은 조사
3. 작은 프로토타입/CLI/HTML 작성
4. 실행 검증
5. verdict 정리

## 주의사항
- docs만 읽어도 답이 나오는 문제에 과하게 build하면 낭비다.
- spike 코드를 production처럼 다듬기 시작하면 목적이 흐려진다.
- 비교 실험이면 approach를 나란히 보여줘야 한다.

## 연결 노트
- [[Sketch 스킬 요약]]
- [[Writing Plans 스킬 요약]]
- [[Subagent Driven Development 스킬 요약]]

# Kanban Orchestrator 스킬 요약

## 용도
- 멀티 에이전트 작업을 board/card 단위로 분해하고, profile별로 배분하는 오케스트레이션 스킬.

## 핵심 포인트
- 오케스트레이터는 직접 실행보다 분해와 라우팅에 집중해야 한다.
- 실제 존재하는 profile 이름을 먼저 확인해야 한다.
- 병렬 lane과 부모-자식 dependency를 올바르게 잡는 것이 핵심이다.

## 자주 쓰는 흐름
1. available profile 확인
2. 작업을 lane으로 분해
3. 병렬 가능 여부 판단
4. kanban task 생성 + dependency 연결
5. 최종 synth/review task로 fan-in

## 주의사항
- 존재하지 않는 assignee 이름을 쓰면 task가 ready에 멈출 수 있다.
- 독립 작업을 한 카드에 몰아넣으면 병렬성이 죽는다.
- dependency를 카드 생성 후 나중에 붙이는 방식은 race를 만들 수 있다.

## 연결 노트
- [[Subagent Driven Development 스킬 요약]]
- [[Writing Plans 스킬 요약]]
- [[Tikke 운영 스킬 묶음]]

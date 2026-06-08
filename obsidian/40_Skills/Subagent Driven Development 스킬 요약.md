# Subagent Driven Development 스킬 요약

## 용도
- 구현 계획을 작은 태스크로 나누고, 각 태스크를 fresh subagent로 실행·검토하는 작업 방식.

## 핵심 포인트
- 태스크마다 새 서브에이전트를 쓰는 것이 핵심이다.
- 구현 뒤에는 spec review와 quality review를 분리하는 2단계 검토 흐름이 있다.
- 큰 기능을 통째로 한 번에 처리하기보다 2~5분짜리 단위로 자르는 것이 중요하다.

## 자주 쓰는 흐름
- 계획 읽기
- todo 생성
- implementer subagent 실행
- spec reviewer 실행
- quality reviewer 실행
- 통과 시 다음 태스크 진행

## 주의사항
- 같은 파일을 만지는 태스크를 병렬로 던지면 충돌 위험이 커진다.
- 리뷰 단계를 생략하면 이 스킬의 핵심 가치가 사라진다.
- 서브에이전트에게는 계획 파일 경로만 던지지 말고 실제 작업 문맥을 같이 줘야 한다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[Writing Plans 스킬 요약]]
- [[Requesting Code Review 스킬 요약]]

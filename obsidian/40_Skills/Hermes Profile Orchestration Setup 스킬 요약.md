# Hermes Profile Orchestration Setup 스킬 요약

## 용도
- Hermes를 `default/planner/researcher/coder/reviewer/publisher` 같은 역할 프로필 구조로 정리할 때 쓰는 스킬.

## 핵심 포인트
- 프로필 생성만이 끝이 아니라 kanban/delegation 라우팅까지 같이 맞춰야 한다.
- 대시보드 설명과 `SOUL.md`까지 있어야 역할 분리가 오래 유지된다.
- `Gateway stopped`는 역할 프로필에선 정상일 수 있다.

## 자주 쓰는 흐름
1. 현재 profile / config 상태 확인
2. orchestration 전역 설정 적용
3. 역할 프로필 생성/보강
4. profile 설명과 `SOUL.md` 작성
5. CLI + 대시보드 검증

## 주의사항
- `custom:<name>`만 넣고 `custom_providers`를 빼먹기 쉽다.
- 역할 프로필을 만들고도 default routing을 안 잡으면 반쪽 설정이다.
- 대시보드 에러는 프로필 자체보다 frontend/dist 문제일 수도 있다.

## 연결 노트
- [[Hermes Agent 스킬 요약]]
- [[Kanban Orchestrator 스킬 요약]]

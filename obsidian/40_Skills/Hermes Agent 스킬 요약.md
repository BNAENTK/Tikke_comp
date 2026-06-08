# Hermes Agent 스킬 요약

## 용도
- Hermes 자체 설정, CLI 사용법, 도구 활성화, 프로필, 게이트웨이, 크론, 메모리, 스킬 운영을 다룰 때 쓰는 핵심 스킬.

## 핵심 포인트
- Hermes 문서는 `https://hermes-agent.nousresearch.com/docs`가 최신 기준이다.
- 설정 변경은 `hermes setup`, `hermes config`, `hermes model`, `hermes tools`, `hermes status` 계열 명령으로 다룬다.
- 스킬/도구/프로필/게이트웨이/크론 관련 전반을 이 스킬 하나로 빠르게 탐색할 수 있다.

## 자주 쓰는 흐름
- 문제 진단: `hermes doctor`, `hermes status`
- 설정 변경: `hermes config set ...`
- 도구 관리: `hermes tools list`, `hermes tools enable/disable ...`
- 스킬 관리: `hermes skills list`, `hermes skills update`
- 프로필 관리: `hermes profile list`, `hermes profile create ...`

## 주의사항
- 문서와 스킬 내용이 다르면 문서를 우선한다.
- Hermes 문제는 감으로 답하지 말고 실제 명령/문서 기준으로 확인한다.
- 설정 변경 후 새 세션이나 재시작이 필요한 경우가 많다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[40_Skills]]

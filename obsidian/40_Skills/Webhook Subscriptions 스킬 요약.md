# Webhook Subscriptions 스킬 요약

## 용도
- 외부 서비스의 event/webhook으로 Hermes 작업을 자동 트리거하는 스킬.

## 핵심 포인트
- GitHub, Stripe, CI/CD, 모니터링, IoT 이벤트를 agent run으로 연결할 수 있다.
- webhook platform을 먼저 활성화해야 한다.
- 단순 전달은 `deliver-only`로 LLM 없이도 가능하다.

## 자주 쓰는 흐름
1. webhook 플랫폼 활성화 확인
2. subscription 생성
3. prompt template과 event filter 설계
4. 대상 서비스에 URL/secret 등록
5. test 후 실사용

## 주의사항
- secret/HMAC mismatch가 가장 흔한 실패 원인이다.
- 외부에서 접근 가능한 URL이어야 한다.
- 단순 알림인데 agent run을 태우면 비용과 지연이 늘어난다.

## 연결 노트
- [[Hermes Agent 스킬 요약]]
- [[Blogwatcher 스킬 요약]]
- [[Productivity 스킬 묶음]]

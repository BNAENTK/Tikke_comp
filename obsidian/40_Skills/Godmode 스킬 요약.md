# Godmode 스킬 요약

## 용도
- API 기반 LLM의 안전 필터 우회, 저항성 점검, 다중 전략 jailbreak 실험에 쓰는 스킬.

## 핵심 포인트
- 프롬프트/프리필/API 레벨 기법이며, 모델 가중치를 바꾸는 방식은 아니다.
- `GODMODE CLASSIC`, `PARSELTONGUE`, `ULTRAPLINIAN` 세 갈래가 핵심이다.
- 자동 감지 후 설정까지 넣는 `auto_jailbreak` 흐름이 중심이다.

## 자주 쓰는 흐름
1. 현재 모델 계열 파악
2. baseline refusal 확인
3. system prompt / prefill / parseltongue 순으로 테스트
4. 필요 시 multi-model race
5. 통과 전략을 config에 반영

## 주의사항
- 모델 버전에 따라 기법 수명이 짧다.
- 일부 기법은 특정 최신 모델에서 이미 패치돼 있을 수 있다.
- 이 스킬은 공격/우회 성격이 강하므로 일반 운영 노트와 분리해 보는 게 좋다.

## 연결 노트
- [[Red Teaming 스킬 묶음]]
- [[Hermes Agent 스킬 요약]]

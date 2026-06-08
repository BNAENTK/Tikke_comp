# DSPy 스킬 요약

## 용도
- 복잡한 LM 파이프라인, RAG, 자동 프롬프트 최적화, 모듈식 AI 시스템을 만들 때 쓰는 스킬.

## 핵심 포인트
- 수동 프롬프트 튜닝보다 선언형 구조와 optimizer가 핵심이다.
- `Signature`, `Module`, `Optimizer` 개념으로 나뉜다.
- 단순 프롬프트보다 재사용 가능한 LM 프로그램 작성에 가깝다.

## 자주 쓰는 흐름
1. signature 정의
2. `Predict` 또는 `ChainOfThought`로 모듈 구성
3. trainset/metric 준비
4. optimizer로 개선

## 주의사항
- 아주 단순한 작업엔 과할 수 있다.
- 대표성 없는 trainset으로 optimize하면 품질이 흔들린다.
- quick prototype과 full DSPy 시스템을 구분해야 한다.

## 연결 노트
- [[MLOps 스킬 묶음]]
- [[Research 스킬 묶음]]

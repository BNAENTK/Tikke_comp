# Weights & Biases 스킬 요약

## 용도
- ML 실험 추적, 메트릭 시각화, sweep, artifact, model registry 관리에 쓰는 스킬.

## 핵심 포인트
- run, project, artifact 개념이 핵심이다.
- 단순 로그 저장이 아니라 비교와 lineage 추적에 강하다.
- sweep으로 하이퍼파라미터 탐색까지 연결된다.

## 자주 쓰는 흐름
1. `wandb.init`
2. config와 metric 로깅
3. checkpoint/artifact 저장
4. 필요 시 sweep 실행

## 주의사항
- 인증이 없으면 cloud 연동이 막힌다.
- run 이름과 태그를 대충 두면 나중에 비교가 힘들다.
- 오프라인 모드와 온라인 모드를 상황에 맞게 구분해야 한다.

## 연결 노트
- [[MLOps 스킬 묶음]]

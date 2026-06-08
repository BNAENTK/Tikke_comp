# llama.cpp 스킬 요약

## 용도
- GGUF 로컬 모델 추론, 적절한 quant 선택, Hugging Face repo 기반 실행 명령 구성에 쓰는 스킬.

## 핵심 포인트
- Hugging Face의 `?local-app=llama.cpp` 페이지를 먼저 보는 흐름이 중요하다.
- repo마다 추천 quant와 실제 파일명이 다를 수 있다.
- `llama-server`와 `llama-cli` 양쪽 흐름을 지원한다.

## 자주 쓰는 흐름
1. HF repo 검색
2. local-app 페이지 확인
3. tree API로 `.gguf` 실제 파일 확인
4. `llama-server` 또는 `llama-cli` 실행

## 주의사항
- quant 라벨만 보고 파일명을 추정하면 틀릴 수 있다.
- `mmproj-*.gguf`는 메인 모델이 아닐 수 있다.
- repo 기준 정보와 일반론을 섞지 말고 실제 파일 존재를 확인해야 한다.

## 연결 노트
- [[HuggingFace Hub 스킬 요약]]
- [[MLOps 스킬 묶음]]

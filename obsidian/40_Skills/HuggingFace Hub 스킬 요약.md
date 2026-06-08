# HuggingFace Hub 스킬 요약

## 용도
- `hf` CLI로 모델, 데이터셋, repo, 업로드/다운로드를 다룰 때 쓰는 스킬.

## 핵심 포인트
- 예전 `huggingface-cli`보다 `hf`가 기준이다.
- 모델/데이터셋 검색, 다운로드, 업로드, repo 관리까지 한 흐름으로 이어진다.
- 자동화할 때는 `--format json`이 유용하다.

## 자주 쓰는 흐름
1. `hf auth login`
2. model/dataset 검색
3. download 또는 upload
4. 필요 시 repo/branch/tag 관리

## 주의사항
- 인증 없이 private repo 작업은 안 된다.
- 대용량 폴더는 `upload-large-folder`가 더 안전하다.
- Hub 탐색 결과를 로컬 운영 규칙과 섞어 해석해야 한다.

## 연결 노트
- [[llama.cpp 스킬 요약]]
- [[MLOps 스킬 묶음]]

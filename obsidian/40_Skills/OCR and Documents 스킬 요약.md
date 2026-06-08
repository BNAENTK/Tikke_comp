# OCR and Documents 스킬 요약

## 용도
- PDF, 스캔 문서, OCR, 텍스트 추출 작업을 다루는 스킬.

## 핵심 포인트
- URL이 있으면 `web_extract`를 먼저 시도하는 게 기본이다.
- 로컬 PDF는 가벼운 `pymupdf`, 복잡한 OCR/수식/레이아웃은 `marker-pdf` 쪽으로 나뉜다.
- 문서의 성격에 따라 도구를 고르는 판단 기준이 이 스킬의 핵심이다.

## 자주 쓰는 흐름
- URL PDF → `web_extract`
- 로컬 텍스트형 PDF → pymupdf
- 스캔/복잡 문서 → marker-pdf
- 페이지 분할/병합/검색

## 주의사항
- marker-pdf는 용량과 설치 비용이 크다.
- Word/PPT는 이 스킬보다 전용 경로가 낫다.
- OCR이 꼭 필요한지 먼저 판단하면 낭비를 줄일 수 있다.

## 연결 노트
- [[Productivity 스킬 묶음]]
- [[PowerPoint 스킬 요약]]

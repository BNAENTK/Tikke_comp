# arXiv 스킬 요약

## 용도
- arXiv 논문 검색, ID 조회, 초록/원문 읽기, 관련 논문 추적에 쓰는 스킬.

## 핵심 포인트
- arXiv는 무료 API로 XML을 주고, Semantic Scholar를 붙이면 citation/related paper까지 확장된다.
- 논문 검색 후 `web_extract`로 abs 페이지나 PDF를 읽는 흐름이 핵심이다.
- 최신 버전과 특정 버전(`v1`, `v2`)을 구분해야 인용 드리프트를 줄일 수 있다.

## 자주 쓰는 흐름
1. 키워드/저자/카테고리로 검색
2. 후보 논문 ID 확보
3. 초록/원문 읽기
4. Semantic Scholar로 영향도와 관련 논문 확장

## 주의사항
- arXiv API는 XML이라 파싱 단계가 필요하다.
- withdrawn 논문 여부를 확인해야 한다.
- abstract만 읽고 전체 결론을 단정하면 위험하다.

## 연결 노트
- [[OCR and Documents 스킬 요약]]
- [[LLM Wiki 스킬 요약]]

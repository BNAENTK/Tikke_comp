# YouTube Content 스킬 요약

## 용도
- YouTube 영상 자막을 가져와 요약, 챕터, 스레드, 블로그 글로 바꾸는 스킬.

## 핵심 포인트
- `youtube-transcript-api` 기반이다.
- transcript를 먼저 확보하고, 그다음 원하는 형식으로 변환한다.
- 긴 영상은 chunking 후 요약 병합이 중요하다.

## 자주 쓰는 흐름
1. transcript fetch
2. 언어/내용 검증
3. 필요 시 chunking
4. summary / chapter / blog / thread 변환

## 주의사항
- 자막 비활성화 영상은 실패할 수 있다.
- language 지정이 안 맞으면 fallback이 필요하다.
- 요약 전에 transcript가 비어 있지 않은지 먼저 확인해야 한다.

## 연결 노트
- [[Research 스킬 묶음]]
- [[OCR and Documents 스킬 요약]]

# Tikke 오버레이 규칙

오버레이 추가/수정 시 빠뜨리기 쉬운 연결 규칙을 정리한 노트.

## 기본 규칙

- 새 오버레이 HTML은 desktop과 worker 공개 리소스 양쪽 연결을 확인한다.
- `top-donor`류 오버레이에는 `window message` 핸들러가 필요하다.
- UI 추가 시 `cloudUrl`, `previewUrl`, 오버레이 HTML 연결을 끝까지 확인한다.
- 오버레이 수정 후 미리보기와 실제 접근 링크를 같이 확인한다.

## 체크 순서

1. 오버레이 HTML 파일이 desktop 쪽에 있는지 확인
2. 같은 오버레이가 worker/public/overlay 쪽에도 있는지 확인
3. UI에서 `cloudUrl`, `previewUrl`, HTML 연결이 다 이어졌는지 확인
4. preview 화면이 실제로 열리는지 확인
5. 이벤트/메시지 반영까지 확인

## 자주 생기는 실수

- desktop에만 파일을 추가하고 worker 쪽을 빼먹음
- `previewUrl` 누락
- `top-donor`류에서 메시지 핸들러 누락
- UI 연결은 했지만 실제 접근 링크 확인을 안 함

## 주의

- 방송 환경 기준은 TTLS다.
- 문서에는 코드 원문을 저장하지 않는다.
- 관련 버그는 [[Tikke 버그-원인 기록]]에 원인 중심으로 누적한다.

## 연결 노트

- [[Tikke Index]]
- [[Tikke 아키텍처 요약]]
- [[Tikke 버그-원인 기록]]
- [[Tikke 작업 기록]]

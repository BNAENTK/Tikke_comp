# Google Workspace 스킬 요약

## 용도
- Gmail, Calendar, Drive, Sheets, Docs, Contacts를 Hermes 경유로 다루는 스킬.

## 핵심 포인트
- 실제 사용 전 Google OAuth 설정이 필요하다.
- 이메일만 필요하면 이 스킬보다 `himalaya`가 더 가벼운 대안일 수 있다.
- URL/문서/일정/메일 자동화처럼 협업형 업무 흐름에 적합하다.

## 자주 쓰는 흐름
- 인증 상태 확인
- Gmail 검색/읽기/보내기
- Calendar 일정 조회/생성
- Drive 업로드/공유
- Sheets/Docs 읽기·쓰기

## 주의사항
- 메일 발송, 일정 생성/삭제, 파일 공유/삭제 같은 외부 변경 작업은 사용자 확인이 필요하다.
- Google token/client secret 파일이 준비되어야 한다.
- Calendar 시간값은 timezone 포함 ISO 형식을 쓰는 게 안전하다.

## 연결 노트
- [[Productivity 스킬 묶음]]
- [[Hermes 스킬 인덱스]]

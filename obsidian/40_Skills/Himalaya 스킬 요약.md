# Himalaya 스킬 요약

## 용도
- IMAP/SMTP 메일함을 터미널에서 읽고, 검색하고, 보내는 CLI 스킬.

## 핵심 포인트
- Hermes 이메일 gateway와 별개로, 실제 메일함 조작용 CLI다.
- 읽기/목록/검색뿐 아니라 reply/forward/send도 가능하다.
- 비대화형 전송은 파이프 입력 방식이 안정적이다.

## 자주 쓰는 흐름
1. 계정/config 확인
2. `envelope list` / `message read`
3. 필요 시 `template send`로 메일 작성/전송
4. move/delete/flag 관리

## 주의사항
- config.toml과 인증이 먼저 준비돼야 한다.
- alias 문법을 잘못 쓰면 전송 후 저장 단계에서 문제가 날 수 있다.
- reply/forward는 interactive editor보다 template 파이프 방식이 안전하다.

## 연결 노트
- [[Email 스킬 묶음]]

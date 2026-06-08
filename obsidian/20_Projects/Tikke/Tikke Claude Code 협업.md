# Tikke Claude Code 협업

## 역할 분담

| 도구 | 추천 역할 |
|---|---|
| Claude Code | `tikke` 내부 코드 수정, 테스트, 리팩터 |
| Hermes | 요구사항 정리, 작업 분해, GitHub/Obsidian/문서화/검증 조율 |

## 추천 흐름

1. Hermes가 목표와 안전 규칙을 정리한다.
2. Hermes가 Claude Code에 줄 작업 프롬프트를 만든다.
3. Claude Code가 코드 수정과 테스트를 수행한다.
4. Hermes가 결과 검증, 문서화, 배포 전 체크를 맡는다.

## 금지

- 역할 중복으로 같은 파일을 동시에 수정하지 않는다.
- 소스코드 원문을 Obsidian 정리본에 복사하지 않는다.

## 연결 노트

- [[Tikke Index]]
- [[Tikke 운영 원칙]]
- [[Tikke Claude Code 작업 패턴]]
- [[Tikke Claude Code 프롬프트 템플릿]]
- [[Tikke Claude Code 프롬프트 템플릿 - TTS 보이스 클로닝 저지연 구현편]]
- [[Tikke 버그-원인 기록]]

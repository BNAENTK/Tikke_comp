# P-Reinforce Architecture

> 출처 프로젝트: [[BNAK_COMPANY 정리]]

## 요약

P-Reinforce Architecture는 Connect AI v2의 핵심 개념입니다. 단순 코딩 에이전트가 아니라 사용자의 정보와 지시를 받아 의미를 분석하고, 폴더와 Markdown wiki 파일로 정리하며, GitHub로 백업하는 자율 지식 정원사 구조입니다.

## 핵심 흐름

1. 사용자 입력 수집
2. 의미 분석
3. 적절한 폴더 선택/생성
4. Markdown wiki 파일 생성
5. GitHub 백업

## Obsidian 적용

Obsidian vault에서는 다음처럼 매핑할 수 있습니다.

```text
00_Inbox      원시 입력
10_Wiki       정리된 지식
20_Projects   프로젝트 문서
30_Automation 자동화 로그
40_Skills     스킬/명령어
90_Archive    보관
```

## 주의

자동 정리와 자동 push는 편하지만 위험합니다. 삭제, push, 배포는 승인 기반으로 둬야 합니다.

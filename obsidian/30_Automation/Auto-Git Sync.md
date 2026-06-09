# Auto-Git Sync

> 출처 프로젝트: [[BNAK_COMPANY 정리]]

## 요약

Auto-Git Sync는 로컬에서 파일이 생성/수정되면 자동으로 GitHub에 동기화하는 개념입니다.

```bash
git add
git commit
git push
```

## 장점

- 수동 push 감소
- 지식/작업 기록 백업 자동화
- 여러 장치에서 동기화 쉬움

## 위험

- 실수로 민감정보 commit 가능
- 불완전한 작업이 push될 수 있음
- 잘못된 삭제가 원격에 반영될 수 있음

## 안전 운영 규칙

- `.env`, token, API key, secret은 절대 commit 금지
- 자동 push는 기본 OFF
- 자동 commit 전 diff 확인
- 삭제/강제 push/배포는 사용자 승인 필수
- Obsidian vault는 우선 로컬 저장 후 수동 또는 승인 기반 동기화

## 대시보드 적용 아이디어

- `저장만` 기본값
- `commit 준비` 버튼
- `diff 확인` 버튼
- `push 승인` 버튼

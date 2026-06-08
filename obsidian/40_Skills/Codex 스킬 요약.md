# Codex 스킬 요약

## 용도
- OpenAI Codex CLI로 기능 구현, 리팩터링, PR 리뷰 같은 코딩 작업을 위임하는 스킬.

## 핵심 포인트
- Codex CLI는 git repo 안에서 돌아야 한다.
- `pty=true`가 사실상 필수다.
- 짧은 작업은 `codex exec`, 긴 작업은 background + process 모니터링이 기본 패턴이다.

## 자주 쓰는 흐름
1. git repo와 auth 상태 확인
2. Codex one-shot 또는 background 실행
3. process로 상태 확인
4. 결과 검증 후 PR/커밋 단계로 연결

## 주의사항
- git repo 밖에서는 바로 막힌다.
- PTY 없이 실행하면 잘 안 맞는다.
- `--yolo`는 빠르지만 위험하다.

## 연결 노트
- [[Claude Code 스킬 요약]]
- [[GitHub PR Workflow 스킬 요약]]

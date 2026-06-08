# Claude Code 스킬 요약

## 용도
- Claude Code CLI에 구현 작업, 리팩터링, 코드 리뷰, 테스트 작업을 위임할 때 쓰는 스킬.

## 핵심 포인트
- 단발성 작업은 `claude -p` print mode가 기본 추천이다.
- 다회전 상호작용은 tmux 기반 interactive orchestration이 적합하다.
- 로컬 대시보드/작업실 UI를 만들 때도 기능형 control panel 패턴을 우선한다.

## 자주 쓰는 흐름
- 한 번에 실행: `claude -p "..."`
- JSON 결과 수집: `--output-format json`
- 세션 이어가기: `--continue` 또는 `--resume`
- 긴 상호작용: tmux 세션 생성 후 `claude` 실행

## 주의사항
- 자동화에는 print mode가 훨씬 안정적이다.
- interactive 모드는 tmux 기반 모니터링이 사실상 필수다.
- 권한 우회 옵션은 기본 선택이 위험할 수 있으므로 주의가 필요하다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[Tikke Claude Code 협업]]

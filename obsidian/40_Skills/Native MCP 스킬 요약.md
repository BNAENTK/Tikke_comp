# Native MCP 스킬 요약

## 용도
- Hermes에 MCP 서버를 붙여 외부 도구를 1급 툴처럼 연결할 때 쓰는 스킬.

## 핵심 포인트
- 실제 설정은 `~/.hermes/config.yaml`의 `mcp_servers`에서 한다.
- stdio 방식(`command`)과 HTTP 방식(`url`) 둘 다 지원한다.
- 연결된 툴은 `mcp_<server>_<tool>` 형태로 자동 등록된다.

## 자주 쓰는 흐름
1. `mcp` Python 패키지 설치
2. `config.yaml`에 서버 등록
3. Hermes 재시작
4. 등록된 MCP 툴 호출 확인

## 주의사항
- 서버 추가/삭제는 재시작이 필요하다.
- 민감한 환경변수는 `env`에 명시한 것만 전달하는 게 기본이다.
- MCP SDK가 없으면 조용히 비활성화될 수 있다.

## 연결 노트
- [[Hermes Agent 스킬 요약]]
- [[DevOps 스킬 묶음]]

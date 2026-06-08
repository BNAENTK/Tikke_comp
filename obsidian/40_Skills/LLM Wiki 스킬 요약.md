# LLM Wiki 스킬 요약

## 용도
- Karpathy식 markdown 기반 지식 위키를 만들고, 소스를 축적하고, 질의와 lint를 반복하는 스킬.

## 핵심 포인트
- 이 스킬은 단순 메모가 아니라 `SCHEMA.md`, `index.md`, `log.md`, `raw/`, `entities/`, `concepts/` 구조를 가진 지식 시스템을 전제로 한다.
- 새 세션마다 먼저 schema/index/log를 읽고 방향을 잡는 것이 핵심이다.
- RAG처럼 매번 다시 찾기보다, 누적 지식베이스를 컴파일한다는 철학이다.

## 자주 쓰는 흐름
1. 위키 구조 확인
2. raw source 저장
3. 관련 페이지 탐색 후 생성/갱신
4. index/log 갱신
5. 필요 시 query/lint로 품질 유지

## 주의사항
- `raw/`는 불변 원본으로 취급해야 한다.
- page를 무분별하게 만들면 오히려 검색성이 떨어진다.
- cross-link, frontmatter, tag taxonomy가 무너지면 유지보수가 급격히 나빠진다.

## 연결 노트
- [[Obsidian 스킬 요약]]
- [[arXiv 스킬 요약]]

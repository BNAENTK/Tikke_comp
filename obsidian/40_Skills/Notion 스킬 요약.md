# Notion 스킬 요약

## 용도
- Notion 페이지, 데이터베이스, Markdown, API 자동화를 다루는 스킬.

## 핵심 포인트
- Windows에서는 기본적으로 `curl` 기반 HTTP 경로가 현실적이다.
- 사용 전 `NOTION_API_KEY`와 integration 연결이 필요하다.
- 페이지 읽기, Markdown 기반 생성/수정, 데이터베이스 query, 파일 업로드 흐름을 다룬다.

## 자주 쓰는 흐름
- search로 페이지/DB 찾기
- page metadata 또는 markdown 읽기
- markdown으로 페이지 생성/수정
- data source query
- block append

## 주의사항
- Notion 페이지/DB를 integration과 공유하지 않으면 404처럼 보일 수 있다.
- 현재 환경에서는 API 키 설정 전까지 바로 쓰긴 어렵다.
- Workers는 더 고급 경로이며 보통 기본 CRUD부터 쓰는 편이 낫다.

## 연결 노트
- [[Productivity 스킬 묶음]]
- [[Hermes 스킬 인덱스]]

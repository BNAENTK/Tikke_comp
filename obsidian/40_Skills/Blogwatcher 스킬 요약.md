# Blogwatcher 스킬 요약

## 용도
- RSS/Atom/blog 피드를 추적하고 새 글을 모니터링하는 스킬.

## 핵심 포인트
- `blogwatcher-cli` 기반으로 blog 추가, scan, unread 관리까지 가능하다.
- RSS가 없으면 scraping fallback도 가능하다.
- 반복 감시성 작업이라 cron/job과 궁합이 좋다.

## 자주 쓰는 흐름
1. 블로그/피드 등록
2. scan 실행
3. unread article 검토
4. 읽음 처리 또는 요약 자동화로 연결

## 주의사항
- DB 경로를 영속적으로 관리해야 한다.
- Docker 사용 시 DB volume을 안 잡으면 상태가 날아간다.
- feed discovery 실패 시 scrape selector를 별도 지정해야 할 수 있다.

## 연결 노트
- [[Productivity 스킬 묶음]]
- [[Webhook Subscriptions 스킬 요약]]

# Tikke 아키텍처 요약

## 구성

| 영역 | 역할 |
|---|---|
| `apps/desktop` | Electron 기반 데스크탑 앱 |
| `apps/web` | Vite/React 기반 웹 앱 |
| `apps/worker` | Cloudflare Worker와 공개 리소스 |
| `packages/shared` | 공통 타입과 공유 로직 |

## 운영 관점

- 데스크탑 앱은 방송 운영자의 로컬 작업 중심이다.
- 웹 앱은 외부 접근 화면과 대시보드 성격의 화면 중심이다.
- Worker는 API, 오버레이 리소스, Cloudflare 배포 중심이다.
- shared 패키지는 앱 간 중복을 줄이는 공통 영역이다.

## 주의

- 이 노트는 구조 설명만 담는다.
- 구현 코드 원문은 저장하지 않는다.

## 연결 노트

- [[Tikke Index]]
- [[Tikke 운영 원칙]]
- [[Tikke 오버레이 규칙]]

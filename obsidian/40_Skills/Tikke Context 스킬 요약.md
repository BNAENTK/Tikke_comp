# Tikke Context 스킬 요약

## 용도
- `tikke`, TikTok LIVE, 오버레이, 리그 컷, TTLS, Bikke, tikketone 같은 도메인 작업에서 먼저 읽어야 하는 맥락 스킬.

## 핵심 포인트
- 이 스킬은 단일 문서라기보다 `references/`에 분리된 운영 지식 묶음이다.
- 필요한 레퍼런스만 골라 읽는 방식이 중요하다.
- 배포, 오버레이 연결, 외부 fetch, 히든 BrowserWindow, 테스트 링크, 리그 컷 API 같은 축적된 규칙이 들어 있다.

## 자주 쓰는 흐름
- 배포 전: 배포 범위/순서/요약 규칙 확인
- 오버레이 작업 전: previewUrl, worker 동기화, message handler 규칙 확인
- TikTok LIVE/TTLS 분석: 관련 reference 문서 선택 읽기
- 문서화/옵시디언 저장: concise reporting 규칙 반영

## 주의사항
- 관련 없는 reference까지 전부 읽지 않는다.
- `tikke` 작업에서는 사용자 고유 운영 규칙을 놓치지 않기 위해 먼저 확인하는 편이 안전하다.
- OBS 기준으로 설명하지 않는다. 사용자는 TTLS 기준이다.

## 연결 노트
- [[Hermes 스킬 인덱스]]
- [[Tikke Index]]
- [[Tikke 오버레이 규칙]]

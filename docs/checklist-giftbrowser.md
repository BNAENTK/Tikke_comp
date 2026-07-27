# checklist — 선물 브라우저 실시간 수집 + 최신순 정렬

## 목표
1. TikFinity 선물 수집을 실시간(자동)으로 동작시킨다.
2. 가져온 선물을 최신순으로 정렬하는 버튼을 추가한다.

## 작업
- [x] 선물 브라우저 관련 파일 위치 파악 (GiftBrowser.tsx, tikfinityGiftStore.ts, tikfinity-gift-service.ts, ipc.ts, preload)
- [x] "최신순" 정의 근거 검증 — 선물 id와 이미지 URL 내장 타임스탬프 상관계수 0.86 확인
- [x] `tikfinityGiftStore`에 `lastUpdated`, `autoStartDisabled` 상태 추가
- [x] `GiftListTab` 마운트 시 폴링 자동 시작 (사용자가 끈 경우 제외)
- [x] 폴링 간격 10분 → 30분 (renderer 상수 + ipc 기본값 동시 반영)
- [x] 마지막 갱신 시각 상태 표시줄에 노출
- [x] 정렬 토글 버튼 추가 (다이아순 ↔ 최신순)
- [x] `allGifts` 정렬을 `sortMode`로 분기
- [x] 수동 "지금 가져오기" 실제 동작 확인 → **고장 확인**. DOM 셀렉터 `.gift-card` 매치 0개 + 타임아웃 없어 무한 대기
- [x] 대체 데이터 출처 확보 — `GET /api/getAllGifts?lang=ko` (JSON, WAF 없음)
- [x] 서비스를 히든 BrowserWindow 스크래핑 → `browserFetch` 주입 API 호출로 교체
- [x] `initTikfinityGifts(browserFetch)` 를 ipc.ts 초기화부에 추가
- [x] API 경로 실측 — 200 / 1.2초 / 888개 / 누락 0
- [x] `pnpm --filter @tikke/desktop typecheck`
- [x] `pnpm build:desktop`

## 검증 방법
- 타입 체크: `pnpm --filter @tikke/desktop typecheck`
- 번들 빌드: `pnpm build:desktop`
- API 실측: electron main에서 `browserFetch` 헤더로 `GET /api/getAllGifts?lang=ko` → 200, 888개, 이미지 URL 누락 0
- 수동: 선물 브라우저 열면 자동갱신 ON + 즉시 1회 수집 → 상태줄에 `마지막 갱신 HH:MM:SS` 표시
- 수동: "지금 가져오기" 클릭 → 1~2초 안에 `888개 선물 로드됨` 갱신
- 수동: "최신순" 버튼 클릭 → 그리드 상단이 높은 id(870962 Gamers' Summer 등)로 채워짐

## 미적용 (요청 없음)
- 선물 카탈로그 영구 저장 (앱 재시작 시 스크랩 결과 소실)
- 폴링 간격 사용자 설정 UI

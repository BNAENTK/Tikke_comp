# 선물 브라우저 수집 보강 — 체크리스트

배경/이유는 `context-notes-gift-collect.md`.

## 1. 부팅 시 수집 시작
- [ ] `tikfinity-gift-service.ts` 에 캐시 저장/복원 붙이기 (수집 성공 시 db 에 저장)
- [ ] `main/index.ts` 에서 부팅 시 `startGiftPolling()` 호출
- [ ] 렌더러가 또 `startPolling` 을 불러도 중복 안 되는지 확인 (`startPolling` 은 stop 후 재시작 — OK)
- 검증: 부팅 로그에 수집 1회 기록, `isPolling()` true

## 2. 결과 저장 (재시작 후에도 유지)
- [ ] db 키 `tikfinityGiftsCache` (JSON) + `tikfinityGiftsUpdatedAt`
- [ ] IPC `tikke:gifts:cached` — 저장된 목록 + 갱신 시각 반환
- [ ] preload 에 `cached()` 노출
- [ ] `GiftBrowser.tsx` 마운트 시 저장분 먼저 로드 → 화면이 비지 않음
- 검증: 앱 껐다 켜도 목록/갱신시각 유지

## 3. 정적 목록 갱신 (5월 → 현재)
- [ ] API 902개를 `public/overlays/gifts.json`(1,889개) 에 **병합** — 신규 추가 + 기존 갱신, **삭제 금지**
- [ ] worker 사본도 같은 파일 있으면 동기화
- [ ] `src/data/` 정적 선물 데이터도 같은 방식으로 갱신할지 판단
- 검증: 신규 445개가 들어갔는지, 기존 항목 수가 안 줄었는지 기계 검사

## 마무리
- [ ] `pnpm --filter @tikke/desktop typecheck`
- [ ] `pnpm check:overlays`
- [ ] 커밋 (의미 단위 3개로 분리)
- [ ] 배포는 K 가 "배포해" 할 때만

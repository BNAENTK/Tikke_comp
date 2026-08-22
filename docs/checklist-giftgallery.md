# checklist — gift/gallery 직접 호출로 선물 카탈로그 자동 수집

## 목표
방송 연결 시 TikTok `POST /webcast/gift/gallery/` 를 자체 세션으로 직접 호출해
전용/신규 선물 포함 최신 카탈로그(다이아값·이미지)를 자동 확보. 사용자 조작 없음.

## 작업
- [x] 인증/위장 헤더 템플릿 확인 — rank-browser.ts fetchRankViaSession (session.fromPartition().fetch)
- [x] 호출 지점 확인 — runConnect: sessionId && roomId 확정된 :602 블록 (collectRankSnapshot 옆)
- [x] giftDiamondCache 존재 확인 — tiklive.ts:166 (fetchAvailableGifts가 채움, 폴백 1순위)
- [x] gift-gallery.ts 신규 — fetchGiftGallery(sessionId, roomId) → {id: {diamond, img, name}}
- [x] 응답 방어 파싱 — data 하위 어디든 id+diamond_count 가진 객체 수집 (스키마 미확정 대비)
- [x] 첫 실행 raw 구조 1회 로그 (실제 필드명 확정용)
- [x] tiklive: 결과를 giftDiamondCache + giftImageCache 병합
- [x] 영구 저장 — userData/gift-diamond-cache.json (재시작 후에도 유지, 연결 안 해도 폴백 가능)
- [x] 시작 시 저장분 로드
- [x] runConnect 에 자동 호출 배선
- [x] `pnpm --filter @tikke/desktop typecheck` — 통과

## 검증
- typecheck 통과
- 다음 방송에서 `[gift-gallery] N개 수집, 다이아 M개` 로그
- 전용 선물이 최고후원자/멀티에 반영 (diamondCount=0 → gallery/폴백 보정)

## 위험
- gift/gallery 요청 파라미터·응답 스키마 미확정 → 방어 파싱 + 실패 무해(기존 fetchAvailableGifts·gifts.json 폴백 유지)로 회귀 없음. 첫 로그로 스키마 확정 후 조정.

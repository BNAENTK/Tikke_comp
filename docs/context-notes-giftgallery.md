# gift/gallery 직접 수집 — 결정 사항과 이유

## 배경
전용/신규 선물이 라이브 이벤트에 diamondCount=0 으로 와 최고후원자 집계에서 빠졌다.
1차 수정: 이벤트 진입점 diamondCount 보정(런타임 캐시 → 정적 gifts.json). 이 문서는
그 런타임 캐시를 더 권위 있는 소스로 채우는 gift/gallery 직접 호출.

## 왜 TTLS 가 아니라 tikke 직접 호출인가
TTLS 로컬 포트(28189)는 원격 제어 프로토콜이지 선물 카탈로그 피드가 아니다(TTLS리버스 확인).
선물 카탈로그는 TikTok 서버 API `POST /webcast/gift/gallery/` 라 TTLS든 tikke든 각자 호출한다.
tikke 는 이미 연결 시 sessionId 를 갖고 있어 인증 API 를 직접 부를 수 있다 → TTLS 경유 불필요·불가.

## 인증/위장 패턴
rank-browser.ts fetchRankViaSession 을 그대로 미러링:
- session.fromPartition("persist:tiktok-gift") 에 쿠키(sessionid/sessionid_ss/sid_tt, store-idc=alisg, store-country-code=kr, tt-target-idc=alisg) 주입
- URL 파라미터 위장: aid=8311, app_name=tiktok_live_studio, version_code=1.27.0, UA TikTokLIVEStudio/1.27.0
- x-ss-stub(md5 of body), x-khronos, x-tt-store-region 헤더
- 엔드포인트 호스트: webcast22-normal-c-alisg.tiktokv.com (rank 와 동일)

## 미확정 → 방어적 구현
gift/gallery 의 정확한 요청 파라미터·응답 스키마는 리버스 노트에 없다(엔드포인트만 확인).
- 요청: room_id 를 body 에 실어 POST (rank 패턴 준용). 필요 파라미터가 다르면 첫 응답으로 확인.
- 응답: data 하위를 재귀 탐색해 {id/gift_id + diamond_count} 를 가진 객체를 전부 수집(스키마 흔들려도 견딤).
- 첫 실행 raw 를 1회 로그 → 실제 필드명 확정 후 좁힌다.
- 실패(403/스키마 불일치)해도 무해: 기존 fetchAvailableGifts + gifts.json 폴백이 남아 회귀 없음.

## 영구 저장
런타임 giftDiamondCache 는 재시작 시 소멸한다. gift/gallery 결과를 userData/gift-diamond-cache.json 에
저장 → 재시작·미연결 상태에서도 이전 수집분으로 폴백 가능. 시작 시 로드해 캐시 시드.

## 자동 실행
runConnect 의 sessionId && roomId 확정 지점(collectRankSnapshot 옆)에서 자동 호출.
연결당 1회. 사용자 조작 없음. 방송 중 카탈로그가 바뀌는 일은 드물어 1회로 충분.

## 엔드포인트 정정 (2026-08-11 브라우저 실측)
최초 구현은 리버스 노트의 `POST /webcast/gift/gallery/` 를 썼으나 **실측 결과 이건 카탈로그가
아님**(VAP 에셋 갤러리, 다이아값 없음/실패). 진짜 카탈로그는 **`GET /webcast/gift/list/`** —
webcast.tiktok.com, aid=1988 웹 파라미터, room_id. 라이브 방에서 200 + 597개 전체 반환,
전용 선물 9종 다이아값 전부 포함(보석총500/우주왕복선20000/페가수스42999/레벨우주선1500·21000 등),
누락 0. collectGifts 재귀 파서가 그대로 다 추출 확인. gift-gallery.ts 를 gift/list GET 으로 교체.

교훈: 리버스 노트의 gallery 는 에셋용이었다. 카탈로그+다이아는 gift/list. 실측이 문서를 이겼다.

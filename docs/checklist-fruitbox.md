# checklist — 사과게임 (Fruit Box) 라이브 게임

## 목표
gamesaien.com 「フルーツボックス」의 **룰만** 가져와 Tikke 안에 자체 구현한다. 원본 코드/에셋은 쓰지 않는다.

- 시청자가 **채팅 좌표**로 사과를 묶어 제거 (합 10)
- 시청자가 **선물**을 보내면 시간 추가 등으로 개입

## 규칙 (확정)
- 보드 17열 × 10행 = 사과 170개, 각 1~9
- 좌표: 열 A~Q, 행 1~10. 채팅 예 `A1 C3` / `a1c3` / `A1-C3`
- 두 좌표가 만드는 직사각형 안 **남은 사과 합이 정확히 10**이면 제거
- 점수 = 제거한 사과 개수 (누적, 시청자별 리더보드)
- 제한시간 기본 120초. 선물 1회당 시간 추가(기본 5초)

## 작업
- [x] 원본 구현 방식 확인 (Adobe Animate/CreateJS, 라이선스 없음 → 자체 구현 결정)
- [x] 기존 게임(초성퀴즈) 배선 체인 전수 조사
- [x] `electron/services/fruitbox-service.ts` 작성 (보드 생성/좌표 파싱/판정/타이머/선물 처리/브로드캐스트)
- [x] `settings.ts` 키 5종 + 기본값 추가
- [x] `ipc.ts` 핸들러 + import
- [x] `preload/index.ts` `fruitBox` 네임스페이스
- [x] `overlay-server.ts` `OverlayMessageType`에 `fruit_box` 추가
- [x] `cloud-overlay.ts` `getUrls()` 라벨 추가
- [x] `worker/src/durable/OverlayRoom.ts` 새로고침 재전송 처리
- [x] `public/overlays/fruit-box.html` 작성
- [x] `apps/worker/public/overlay/fruit-box.html` 동일 복사
- [x] `src/pages/FruitBox.tsx` 호스트 패널
- [x] `App.tsx` 라우트 + `Sidebar.tsx` 메뉴 + `EventOverlay.tsx` GAME_PAGES
- [x] `pnpm --filter @tikke/desktop typecheck`
- [x] `pnpm --filter @tikke/worker typecheck`
- [x] `pnpm build:desktop`
- [x] 앱 실행 후 실동작 확인 (보드 생성, 채팅 판정, 선물 시간추가)

## 검증 방법
- 타입: `pnpm --filter @tikke/desktop typecheck`, `pnpm --filter @tikke/worker typecheck`
- 오버레이 시각: `http://localhost:18181/overlay/fruit-box?test=1`
- 판정 로직: 앱 실행 후 IPC `hostPick`으로 좌표 직접 넣어 확인
- 선물: 하네스 `pnpm harness:gift`로 선물 이벤트 주입 → 남은 시간 증가 확인

## 추가 작업 (2026-07-28)
- [x] 방송인 마우스 드래그 플레이 — 호스트 패널 보드에서 드래그로 직접 제거
- [x] 드래그 중 합계 실시간 표시 + 선택 영역 아웃라인(10=초록/초과=빨강)
- [x] 제거 직후 상태 즉시 재조회 (1초 폴링 대기 제거)
- [x] 실제 마우스 이벤트로 드래그 검증 (E1 F1 → 2개 제거)

## 추가 작업 2 (2026-07-28) — 선물별 시간 규칙
- [x] fruitBoxGiftAddSec 제거 → fruitBoxGiftRules(JSON) 도입
- [x] 지정 선물만 시간 증감, 초 음수 허용(깎기)
- [x] 호스트 패널에 GiftPicker 기반 규칙 편집 UI
- [x] 0 이하로 깎이면 판 종료
- [x] 오버레이/패널 부호·색 표시, 절대값 순 정렬
- [x] 실행 검증 (등록/미등록/음수/빈 규칙/종료)

## 추가 작업 3 (2026-07-28)
- [x] 초 입력창 하이픈(음수) 입력 가능하게 수정
- [x] 합 10 성공 효과음 (WebAudio 합성, 개수 비례, ?mute=1)
- [x] 새로고침 스냅샷으로 효과음 오발동 방지
- [x] AudioContext 가로채기로 호출 검증

## 미적용 (요청 없음)
- 자동 힌트/방해 아이템 (선물은 시간 추가 + 기여자 표시만)

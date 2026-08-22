# checklist — troll gift (가짜 선물 연출)

목표: 방송인이 실제로 오지 않은 선물의 TikTok 원본 애니메이션 + 알림 배너를 임의 재생.
제약: 오버레이 파일 무변경, 기존 디자인 유지, 랭킹 데이터 무오염.

## 조사 (완료)
- [x] TIK SCAN에 troll gift 기능 있는지 확인 → ~~없음~~ **정정: 있다.** 앱 사이드바 `Troll Gift` 메뉴 실재(스크린샷 확인), 라우터에 `overlay-troll-gift` 존재. 번들 grep만 보고 오판했음 — TIK SCAN은 기능 대부분이 서버측이라 `app.asar` grep으로 기능 유무를 판단하면 안 된다.
- [x] TikTok 선물 카탈로그에 "troll" 선물명 확인 → 없음 (선물 이름이 아니라 기능 이름이 맞았음)
- [x] tikke 기존 선물 애니메이션 인프라 인벤토리 → `gift-fx` / `gift-videos` 두 갈래 확인
- [x] 재활용 대상 확정 → **gift-fx** (원본 애니메이션 필요하므로)
- [x] 캐시 실제 내용 확인 → **`AppData\Roaming\@tikke\desktop\gift-asset-cache.json`, 88개(85개 재생 가능)**. TikTok Universe 44999💎, King Leonardo 42999💎, Lion 29999💎 등 최상위 선물 포함
  - ⚠️ 처음에 `AppData\Roaming\Electron\`(dev 실행 경로, 2개)만 보고 "2개뿐"이라 잘못 판단했음. 설치 앱 userData는 `@tikke\desktop`이고 `find -maxdepth 2`로는 안 잡힌다(depth 3). **앞으로 userData 조사 시 `@tikke/desktop` 경로를 먼저 볼 것.**
- [x] broadcast 부작용 트레이스 → 실제 후원 데이터 오염 없음 확인
- [x] 알림 배너 오버레이 특정 → `gift.html`, 읽는 필드는 `user.nickname` / `giftName` / `repeatCount` 3개뿐

## 구현 (1차 — GiftFx 최소 수정, 유지됨)
- [x] `GiftFx.tsx` — 닉네임/수량 입력 + `play()` 반영 + 부작용 경고
  - 이 페이지는 개발자 섹션에 그대로 둔다. 롤백하지 않음.

## 구현 (2차 — 별도 Troll Gift 페이지 신규, 확정)
- [x] `pages/TrollGift.tsx` 신규 — TIK SCAN Troll Gift UI 복제 (헤더/도움말, 선물·직접입력 탭, 카드 리스트, Edit 인라인, PLAY)
- [x] `components/Sidebar.tsx` — "오버레이" 섹션에 "선물 낚시"(trollgift) 메뉴 + 타입 유니온 추가
- [x] `app/App.tsx` — import + `case "trollgift"` 라우팅
- [x] 카드 = 캐시 88개 중 재생 가능(videoUrl 有)만, 라이브 업데이트 구독
- [x] Edit 인라인 — 이름/문구/수량/아바타(FileReader data URL). 아바타는 `user.profilePictureUrl`로 송출
- [x] 직접 입력 탭 — 애니 없이 알림 배너만 커스텀 문구로

## 구현 (3차 — Enigma 아바타 + 맞춤 링크 20개, 2026-08-02)
- [x] Enigma 기본 아바타 — cdn.tikscan.uk/effect-previews/enigma.webp 다운 → 상반신 256×256 크롭 → `public/troll-avatars/enigma-avatar.webp`
  - tikfinity "Icy Enigma"(Desiree 여성 캐릭터) 오답 걸러냄
- [x] `gift-fx.html`(desktop+worker) — `?link=N` 읽어 `targetLink` 필터. 없으면 전부 재생(하위호환)
- [x] `settings.ts` — `trollGiftLinkNames: string` 키 추가
- [x] `TrollGift.tsx` — "직접 입력" 탭 → "맞춤 링크"(20개 링크 URL + 이름 저장). 카드 Edit에 "재생 링크" 드롭다운. `fire()`에 `targetLink` 포함

오버레이 HTML 변경: `gift-fx.html`에 link 필터만 추가(desktop+worker 동기화 완료). `gift.html`(배너)은 무변경.

## 검증
- [x] `pnpm --filter @tikke/desktop typecheck` — 통과
- [x] `pnpm --filter @tikke/worker typecheck` — 통과
- [ ] 실사용 확인 (K님) — 화면 스크린샷 대조 + 발사 + 링크별 라우팅. **K님이 직접 확인 예정**

## 구현 (4차 — VIP 잠금 + 로컬 효과음/볼륨, 2026-08-02)
- [x] 사이드바 "선물 낚시 🔒" + `locked: true` — VIP/admin만 접근(`useHasLockAccess`). non-VIP 클릭 시 "🔒 권한 없음" alert. 기존 잠금 메뉴와 동일 메커니즘
- [x] 앱 내 효과음 — `fire()`에서 broadcast + `playSound()`. GiftFx의 extractMp4→오디오 파이프라인 이식. 큐로 순차 재생
- [x] 볼륨 슬라이더 — `giftFxVolume` settings 공유(GiftFx와 동일 키). 즉시 반영 + 영속
- [x] 아바타 캐시버스팅 — `DEFAULT_AVATAR`에 `?v=2`. 정적 파일 교체 시 브라우저 캐시 무효화. 아바타 상반신 크롭으로 교체됨

## ⚠️ 배포 대기
- `gift-fx.html`의 link 라우팅은 **worker 배포 후**에야 실제 클라우드 오버레이에 반영된다. K님 배포 금지 지시로 미배포.
- 배포 전까지: targetLink 없는 기존 발사는 정상, 링크별 분리 재생은 미작동.

## 주의
- **배포 금지 (K님 명시).** 커밋/버전업/태그/push 하지 않음.

## 확인된 제약 (구현 대상 아님, 기록용)
- ~~캐시가 2개뿐~~ → **오판이었다. 88개 저장돼 있어 선물 확보 작업 자체가 불필요하다.** 라이브에서 99💎+ 선물을 받을 때마다 계속 늘어난다.
- 로컬(18182) 오버레이는 가짜 선물을 못 받는다. 클라우드 URL 전용.
- `gift.html`은 `?duration=` 없으면 카드가 계속 쌓인다 (기존 동작, 손대지 않음).

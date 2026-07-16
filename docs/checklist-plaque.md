# 레벨업 상패(level-plaque) 오버레이 구현 체크리스트

레퍼런스 벤치마크 + K 크리스탈 트로피 디자인. 데이터: fan_levelup/sub_levelup {nickname, level}.

## 오버레이 HTML
- [ ] `apps/desktop/public/overlays/level-plaque.html` — 크리스탈 상패 디자인(월계관/골드숫자/TikTok LIVE/하트/닉/명판), reload-patch, WS(18182/api.tikke.kr), message 핸들러, ?scale/?test/?theme/?minLevel/?duration, {type:events} 배치 파싱, 투명 배경, 등장 애니+자동 퇴장
- [ ] `apps/worker/public/overlay/level-plaque.html` — 동일 복사

## 배선 (풀체인)
- [ ] `cloud-overlay.ts` getUrls() — "레벨업 상패" → `/level-plaque?room=`
- [ ] `EventOverlay.tsx` OVERLAYS 카드 (id level-plaque, name, desc, path, testQuery, tags)
- [ ] `EventOverlay.tsx` ID_TO_LABEL — "레벨업 상패"(getUrls 라벨과 정확 일치)
- [ ] `EventOverlay.tsx` RECOMMENDED_SIZE
- [ ] `EventOverlay.tsx` CATEGORY_MAP
- [ ] `EventOverlay.tsx` cloudUrl 빌더 (theme/minLevel/duration 파라미터)
- [ ] `EventOverlay.tsx` testActions (fan_levelup 샘플)
- [ ] `EventOverlay.tsx` 설정 UI (테마 선택/최소레벨/지속시간)
- [ ] (선택) 설정 push하면 SYNC_TYPES 등록 — 초기엔 URL 파라미터로만

## 검증
- [ ] `pnpm --filter @tikke/desktop typecheck`
- [ ] `pnpm --filter @tikke/worker typecheck`
- [ ] 브라우저 `?test=1` 실측 (python http.server + 스크린샷)
- [ ] 응답에 테스트 링크 포함

## 데이터/디자인 노트
- fan_levelup(멤버 팬클럽) / sub_levelup(후원) 둘 다 {payload:{nickname, level}}. 배지 없음 → level로 하트 파생(≥40 레드젬, <40 오렌지). 레벨 40↑/40↓ 아이콘 자동 전환.
- 테마 후보: classic(골드)/royal(레드젬)/neon(시안). 7종 중 축약.
- 디자인 = CSS/SVG 근사(실사진 3D 굴절 X). 투명 배경 유지.

# AdSense 재승인 체크리스트 (tikke.kr)

거절 사유(2026-07-04 확인): ① 게시자 콘텐츠 없는 화면에 광고 ② 가치 낮은 콘텐츠. 재거절("또").

## 구조 수정 (저위험, 실행) ✅ 완료 2026-07-04
- [x] index.html — 홈에서 ads.js 로더 제거. `google-adsense-account` meta 유지
- [x] promo-loop.html — `noindex,follow` 추가
- [x] companion/index.html — `noindex,follow` 추가
- [x] cutoffs/index.html — `noindex,follow` 추가
- [x] onepager/index.html — `noindex,follow` 추가
- [x] pitch/index.html — `noindex,follow` 추가
- [x] investors.html — `noindex,follow` 추가
- [x] 광고 로더 최종 배치 = about, guide/*, tips/* (콘텐츠 페이지만) 확인

## 콘텐츠 강화 (핵심 #2, K 스코프 결정 대기)
- [ ] about.html 보강(현 ~1.5K자)
- [ ] 신규 심층 아티클 N개 (주제 후보: 리그 승급 전략, 선물 이벤트 기획, 번역 활용법 등)
- [ ] 기존 17글 템플릿 유사성 점검

## 콘텐츠 강화 ✅ 완료 2026-07-04
- [x] 신규 심층 아티클 6개(best-time-to-live/live-algorithm/beginner-roadmap/avoid-ban/pk-battle/agency-guild), 각 1800~2400단어, schema/템플릿/콜론규칙 준수, 정직 날짜(2026-07-04)
- [x] tips 허브 카드 6개 + sitemap 6 URL 배선
- [x] about.html — 이미 충실해 미변경(surgical)
- [x] 기존 글 교차 중복 낮음 확인(각 주제 집중)

## 검증/배포 ✅ 완료 2026-07-04
- [x] 프로덕션 배포(wrangler pages deploy → tikke-web). 홈 광고=0, 신규6=200, sitemap 29, 허브 12카드, 비콘텐츠 6 noindex 라이브 검증
- [ ] ⚠️ **캐시 퍼지 필요**: /adsense-checklist.md 가 엣지캐시(7일)에 남음. 파일은 origin서 제거됨. K가 Cloudflare 대시보드서 해당 URL 퍼지
- [ ] AdSense "검토 요청" 클릭 (K 수동)

## 파일 위치
- 이 체크리스트는 tikke/ 루트(배포폴더 밖)로 이동됨. 공개 안 됨.

## 주의
- robots.txt는 Cloudflare 관리(주입). 로컬 robots Disallow는 두 번째 User-agent:* 로 붙어 무시될 수 있음 → noindex meta 사용.
- AdSense 크롤러(Mediapartners-Google)·Googlebot은 robots에서 차단 안 됨(확인 완료).
- 100% 통과 보장 불가(구글 심사). 알려진 거절 요인 전부 제거가 최대치.

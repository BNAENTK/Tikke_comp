# AdSense 재신청 대응 노트 (2026-06-12)

## 탈락 이력
- 1차: "가치가 별로 없는 콘텐츠" → guide.html + tips 4편 추가 (2026-05-31)
- 2차: 동일 사유 재탈락 (2026-06-12 보고됨)

## 2차 진단 (구글 문서 4종 대조)
근거 문서: adsense/10502938 (Publisher Policies), adsense/10015918 (content & UX), webmasters/9044175 (thin content), publisherpolicies/11035931 (spam policies).

발견된 위반/약점.
1. **영상 stub 페이지 3개** — guide/gift-sound·menu-board·translation 이 텍스트 ~1,500자(본문 3줄+영상). 구글 명시 위반: "embedding content such as video without substantial added value".
2. **E-E-A-T 신호 전무** — 모든 기사에 작성일/수정일/저자/JSON-LD 없음.
3. **저가치 화면에 광고 코드** — privacy/terms/disclaimer/contact 에 adsbygoogle. Inventory value 정책 위반 소지.
4. **일괄 생성 흔적** — 콘텐츠 전부 5/21·5/31 두 날짜에 몰림. "정기 업데이트" 요건 미충족.
5. **tips 허브 얇음** (1,593자) + 홈 nav에서 tips 미노출.

## 조치 (이 커밋)
1. stub 3개 → 영상 유지 + 본문 5천자급 풀 가이드 (FAQ·운영팁·문제해결).
2. tips 4편 + guide stub 3편에 JSON-LD Article + 작성/수정일 표기.
3. 법적 4페이지에서 AdSense 스크립트 제거 (콘텐츠 페이지에만 유지).
4. 신규 독창 기사 2편: tips/tiktok-live-studio.html (한국어 자료 희소), tips/diamond-revenue.html (수치 단정 없이 구조 설명).
5. tips 허브 인트로 보강 + 새 기사 카드 2개.
6. 홈 navlinks에 "팁" 추가 — i18n MAP은 nth-of-type(1)(2)만 매핑하므로 3번째 append는 기존 번역 안 깨짐 (새 링크는 전 언어 한국어 표기, 의도적 수용).
7. sitemap: 22→24 페이지, 수정분 lastmod=2026-06-12.
8. tikke-web 레포에 16개 파일 동기화 (index.html의 구버전 "OBS" 문구도 이번 복사로 제거됨).

## 재신청 전략 (중요 — 순서 지킬 것)
1. 배포 후 Search Console에서 sitemap 재제출 + 신규/수정 페이지 색인 요청.
2. **색인 확인 전 재신청 금지.** Search Console에서 tips/guide 페이지 다수가 "색인 생성됨" 상태가 된 후 신청. 통상 1~3주.
3. 재신청 전 주 1회 정도 새 글 추가 권장 (일괄 생성 패턴 탈피). 다음 글 후보: 방송 장비 세팅, 채팅 관리 노하우, 시간대 분석.
4. 연속 탈락 시 쿨다운이 길어지는 경향 — 서두르지 말 것.

## 남은 약점 (인지하고 있을 것)
- 사이트 본질이 앱 랜딩이라 "콘텐츠 사이트" 대비 불리. tips 코퍼스 확장이 유일한 대응.
- 트래픽 자체가 적으면 심사에 불리할 수 있음 (비공식 요인).

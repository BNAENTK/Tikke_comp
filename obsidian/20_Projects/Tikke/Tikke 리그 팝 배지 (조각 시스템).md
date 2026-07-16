# Tikke 리그 팝 배지 (조각 시스템)

앵커 리그 배지에 위젯 스타일 "리그 팝" 테마를 구현하며 확정한 틱톡 조각 시스템 규칙 + 실시간 갭 아키텍처. 2026-07-11~12, v1.8.156 배포.

## 조각 시스템 규칙 (K 확인, "정확해")

- 리그마다 퍼센타일 컷 여러 개: A=1/10/20/60%, B=1/10/20/70%, C=10/20/80%, D=10/20%.
- **최하위(최대 %) 컷 = 유지컷**, 나머지 = **조각컷**(쉬운 % 순). B리그: 70%=유지, 20/10/1%=조각 1~3.
- 컷 달성마다 조각 1개 획득. 젬 아이콘 = 획득 상태(밝음/어두움). 조각 수 = A/B 3개, C 2개, D 1개.
- "2.3k 더 있으면 유지" = 유지컷까지 부족분. 유지 확보 후엔 다음 미획득 조각컷이 목표.

## 리그 팝 테마 (anchor-badge.html `theme=league`)

- 3줄: ① 💎 라운드 남은시간(`round_end_at`, 없으면 숨김) ② `{tier} ◆◆◆ 리그` ③ `{gap} 더 있으면 ◆ 유지/조각` + 진행률 게이지.
- 투명 배경, Cafe24Ssurround, 흰 글자 + 8방향 아웃라인 + 하단 글로우, skewX(-4deg), SVG 젬.
- **리그 그룹별 팔레트**: A골드/B블루/C틸/D브론즈 — 티어 첫 글자로 자동. 아웃라인·글로우·젬 그라데이션 전부.
- 데모: `?theme=league&demo=1&tier=A2` — 4초마다 가상 선물 600다이아 12발로 전체 연출 재현.

## 실시간 갭 아키텍처 (핵심 차별점)

- 오버레이가 **선물 이벤트를 직접 구독**: 로컬 `ws://localhost:18182`는 `{type:'gift', payload}`, 클라우드 room WS는 raw 이벤트 — 두 shape 다 처리.
- `라이브 점수 = 마지막 수집 점수(base) + base.updated_at 이후 선물 다이아 합`. base 갱신(60초 fetch)마다 base 이전 선물 로그 폐기 → 이중계산 방지. 수집(3분)과 스트림 간 수 초 오차는 다음 수집서 자동 보정.
- **틱톡 공식 gap 우선**: rank API `owner_rank.gap` pieces([0]=다이아, [1]=목표%)를 `user_league_data.gap_amount/gap_percentile`에 저장(gap 없으면 명시적 null — 스테일 방지). updated_at 15분 이내 fresh면 컷 계산 대신 표시, 라이브로 차감, 0 돌파 시 컷 계산 폴백.
- 연출: 유지/조각 돌파 감지 → 배너 + 젬 pop + 화면 플래시 + WebAudio 차임(C5-E5-G5). `&fx=0`으로 끔.
- 갭 숫자 600ms 롤링 트윈, 다음 목표 대비 게이지 바.

## 부속 기능

- `!리그` 채팅 명령(command-service 내장, 30초 쿨다운) → `getAnchorLeagueSummary()` → TTS 응답.
- 라운드 남은시간: rank_view에서 count_down 계열 키 딥스캔(필드명 미확정) → `round_end_at` 저장.
- **rank_view 원본 1회 덤프** — `%APPDATA%\Tikke\logs\rank-view-dump-*.json`. 다음 라이브에서 조각 상태 필드(class_rank_extra 추정)와 count_down 실경로 확인 → 배지의 컷오프 테이블 의존 제거 검토.

## 파일

- `apps/desktop/public/overlays/anchor-badge.html` (+worker 미러) — 테마/실시간/연출 전부.
- `apps/desktop/electron/services/league-cutoff-collector.ts` — gap/카운트다운 추출, 덤프, !리그 요약.
- `apps/desktop/electron/services/command-service.ts` — !리그 내장 명령.
- Supabase `user_league_data` += `round_end_at, gap_amount, gap_percentile`.
- 상세: tikke 리포 `context-notes-league-badge.md`.

## 미확정 / 대기

- count_down 필드 실존 여부 + 조각 상태 직접 필드 — **다음 라이브 덤프로 확인**.
- rank API 구조 한계: 호출 1번 = 컷 1개(그 앵커의 다음 목표)만 → 전체 컷오프 테이블은 여전히 크라우드소싱 필요.

## 관련

- [[Tikke 리그 컷오프 자체수집 + 앵커]]
- [[Tikke 아키텍처 요약]]

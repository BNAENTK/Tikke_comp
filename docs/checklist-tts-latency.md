# TTS 지연 개선 체크리스트

목표 — tikketonic(로컬 보이스 클로닝)으로 Google TTS 수준의 체감 지연으로 채팅 읽기.

## 측정 기준

- 큐 대기 중 체감 지연 0 (합성이 재생 뒤에 완전히 가려짐)
- 무음 상태 첫 발화까지 0.5초 이하
- 채팅 2.5초 간격에서 전달률 88% 이상

## 작업

- [x] 지연 경로 조사 — 병목이 합성이 아니라 재생 처리량임을 확인
- [x] prefetch 깊이 현황 확인 — 깊이 1 (`queue[0]` 1건), 캐시는 8칸
- [x] prefetch 깊이 1 → 3 상향 (일반 큐 + MVP 큐)
- [x] `pnpm --filter @tikke/desktop typecheck` — 통과
- [ ] 전달률 측정 — `harness:tts`는 필터/큐잉만 검증하고 타이밍은 안 봄. 실앱 측정 필요
- [x] 현재 RTF 실측 — 평균 0.359, GPU 실질 미가동 추정 (`harness/scenarios/tikketonic-rtf.ts`)
- [x] 무음 첫 발화 지연 측정 — 짧은 채팅 기준 ≈830~930ms
- [x] 병목 정체 규명 — 텍스트 길이 무관, diffusion step당 ≈110ms 선형 (`tikketonic-steps.ts`)
- [ ] **step 수 결정** — 8→4/5 하향 시 526/646ms. 음질 판정은 사용자 청취 대기
- [ ] 결정된 step을 기본값으로 반영
- [ ] GPU 미가동 원인 진단 (DirectML 노드 할당 실패)
- [ ] stale 정책(20초) 재조정 검토

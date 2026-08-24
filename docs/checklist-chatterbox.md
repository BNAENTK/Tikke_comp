# Chatterbox 보이스 클로닝 통합 — 체크리스트

목표: 내 목소리로 틱톡 채팅을 ko/en/ja/zh 4개 언어로 저지연 TTS 출력.

## 0. 사전 검증 (가장 먼저 — 이거 없이 다음 단계 진행 금지)
- [x] `chatterbox-tts` 로컬 설치 (RTX 4070 Ti 12GB, CUDA 확인)
- [x] 참조 클립으로 등록/합성 경로 검증
- [x] 같은 참조로 ko/en/ja/zh 4개 언어 합성 성공 (크로스링구얼 확인)
- [x] 실측: 중앙 RTF 1.5, 중앙 합성 2.2s (rp=1.2 적용 후). GPU 메모리 ~3.2GB
- [ ] 한국어 목소리 품질 청취 확인 (K님 실제 녹음 필요)
- [ ] `cfg_weight` A/B — UI 슬라이더로 조절 가능. 외국어 억양 vs 말투 유지 비교
- [x] 결과를 `context-notes-chatterbox.md`에 기록

## 1. 대사/데이터 (완료)
- [x] `apps/desktop/src/data/cloneScripts.ts` — 레지스터 3종 × 후보 3개 녹음 대사
- [x] 다국어 검증 문장 `VERIFY_LINES` (ko/en/ja/zh-CN)
- [x] `apps/desktop/src/lib/detectChatLang.ts` — 문자 기반 언어 판별 (11케이스 통과)
- [x] `CHAT_LANG_TO_LANGUAGE_ID` — zh-CN → zh 매핑
- [ ] `packages/shared`로 이동 검토 (tikketone 도 쓰게 되면)

## 2. 로컬 서버
- [x] `services/chatterbox/` — FastAPI 래핑 완료
- [x] 엔드포인트: `/health` `/voices`(GET/POST/DELETE) `/tts` `/cache`
- [x] 포트 18187 (qwen-tts 18186 과 분리)
- [x] 참조 임베딩 캐시 (`prepare_voice` 가 voice_id 동일하면 스킵)
- [x] 모델 상주 (lifespan 로드, 12s). 첫 합성 워밍업 지연 있음
- [x] 부모 PID 워치독

## 3. Electron 연결
- [x] `chatterbox-manager.ts` — 프로세스 관리 + 포트 점유 정리 + API
- [x] `TTSProvider` 에 `chatterbox` 추가 + TTSConfig 5개 필드 + AppSettings 5개 키
- [x] 디스패처 분기 + IPC 8종 + preload 배선
- [x] `RATE_BAKED_PROVIDERS` 미포함 결정 (speed 는 playbackRate 가 담당)

## 4. 다국어 라우팅
- [x] `langOf()` → payload `lang` → IPC 에서 zh-CN→zh 매핑
- [x] 판별 실패 시 설정 언어로 폴백
- [x] 주 언어 하나로 결정 (문자 최다 기준)

## 5. 녹음 UI
- [x] `ChatterboxPanel` 신규 (기존 패널 미변경 → supertone-clone 무영향)
- [x] 레지스터 3종 선택 + 톤 힌트 + 대사 전환
- [x] concat 없음. 클립 1개 = 목소리 1개
- [x] 15초 자동 정지, 1개면 등록 가능

## 6. 검증 완료 (엔드투엔드)
- [x] 서버 기동 → health CUDA 확인
- [x] 참조 등록 → 목록 조회 → ko/en/ja/zh 4/4 합성 성공
- [x] 캐시 히트 0.016s
- [x] 미지원 언어(fr) 거부 확인
- [x] 목소리 삭제 확인

## 7. A/B 하네스
- [ ] `pnpm harness:tts`가 참조 오디오 경로 파라미터를 받는지 확인
- [ ] 후보 클립 × 검증 문장 매트릭스 합성 → 나란히 청취

## 8. 남은 검증
- [x] `pnpm --filter @tikke/desktop typecheck` 통과
- [ ] `pnpm harness:tts` 큐 동작
- [ ] 채팅 폭주 시 백로그 (stale drop 20초가 방어하는지 실측)
- [ ] 렌더러 콘솔 확인 (조용한 실패 탐지)

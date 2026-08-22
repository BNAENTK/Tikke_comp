# 번역자막 간헐 고장 수정 체크리스트 (2026-07-30)

증상: 말 씹힘 / 한 글자(단어)만 번역 / 계속 "인식중" 고착.
대상: `apps/desktop/public/overlays/translation.html` (STT+번역+송출 전부 이 파일. Chrome 창에서 실행)

## 수정
- [x] startSTT — start() 실패 시 500ms 재시도 (현재 catch로 삼키고 영구 사망)
- [x] STT 워치독 — 이벤트 45초 무소식 시 새 인스턴스 강제 재시작 (onend 안 오는 행 대비, sttActive 창에서만)
- [x] interim 카드 신선도 타이머 — 12초 무갱신 시 자동 제거 (송신 사망 시 반투명 카드 영구 잔존 = "계속 인식중" 겉증상, 수신측 TTLS도 동일)
- [x] gtrans 반응어 하이브리드 — 나머지 구절 번역 실패 시 반응어만 반환 금지, 전체 문장 재시도 후 실패면 '' ("Wow"만 뜨는 버그)
- [x] gtrans 반응어 단독 발화("와") 사전 직행 추가 — 단독일 때 gtx 오역("and") 방지 (검증 중 발견)
- [x] _gtxRaw — 429 감지 시 3초 쿨다운 (interim은 스킵, final은 MyMemory 직행)
- [x] MyMemory 폴백 — 5초 AbortController 타임아웃 (현재 무제한 행 → final 자막 통째 지연)

## 동기화/검증
- [x] `apps/worker/public/overlay/translation.html`에 동일 파일 복사 + 문법 검증
- [x] playwright 실측 — interim 렌더/final 승격/12초 고아 정리/반응어 실패·성공·단독 경로 전부 확인
- [x] pnpm --filter @tikke/desktop typecheck 통과

## 배포
- [x] 수정 커밋 7b4af70
- [x] 1.8.192 bump + v1.8.192 태그 + push (GitHub + worker 자동배포)

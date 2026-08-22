# 번역자막 원어(source language) 선택 — 체크리스트

목표: 원어를 한국어/영어/일본어/중국어(간체) 중에서 고른다. 한국어가 아닌 원어를 고르면
원어 줄이 맨 위(원문 자리)로 가고 한국어가 번역 줄로 내려간다 = "위치 교체".

## 오버레이 (translation.html — desktop + worker 양쪽)
- [x] `ALL_LANGS` + `srcLang()` / `otherLangs()` / `targetLangs()` 헬퍼 도입
- [x] `DEFAULT_STYLE.srcLang = 'ko'`
- [x] render: `addLine('ko', original)` → `addLine(srcLang(), original)`
- [x] render: `hasAnyTranslation` + 번역 루프 → `otherLangs()` / `targetLangs()`
- [x] renderInterim: `buildLine('ko', ...)` → `buildLine(srcLang(), ...)`
- [x] renderInterim: 불투명 판정 `lang === 'ko'` → `lang === srcLang()`
- [x] renderInterim 번역 루프 → `targetLangs()`
- [x] `onRecognized` / `doInterimTranslate` 대상 → `targetLangs()`
- [x] `gtrans` 한국어 반응어 사전 → `srcLang() === 'ko'` 일 때만
- [x] `_myMemory` `langpair=ko|` → `srcLang()|`
- [x] `recog.lang` → STT_LANG 맵 (ko-KR / en-US / ja-JP / zh-CN)
- [x] `translation_config` 핸들러 — srcLang **변경 시에만** STT 재시작
- [x] worker `public/overlay/translation.html` 동일 복사

## 설정 체인
- [x] `settings.ts` — `translationSourceLang` + 기본값 `"ko"`
- [x] `default-settings.ts` — **안 넣음.** 그 파일은 v1.3.92 시점 시드 스냅샷이고,
      넣어 봐야 `settings.ts` 기본값과 같은 `"ko"` 라 중복이다
- [x] `useTranslationEngine.ts` — 설정 로드 시 `sourceLang` 복원 + 테스트 번역 payload.style 에도 포함
- [x] `ipc.ts buildTranslationStyle` — `srcLang` 포함 (재연결 push)
- [x] `translationStore.ts` — `sourceLang` + DEFAULT_CONFIG
- [x] `TranslationOverlay.tsx sendStyleConfig` — `srcLang` 포함 (라이브 변경 push)
- [x] `TranslationOverlay.tsx` UI — 원어 선택 버튼 4종
- [x] UI — 원어와 같은 번역 언어 토글 비활성 + 한국어 대상 항상 표시

## 검증 (postMessage 실측 — typecheck로는 안 잡힘)
- [x] srcLang=ko: 줄 순서·국기가 변경 전 캡처와 동일
- [x] srcLang=en: 영어 맨 위 🇺🇸 / 한국어 아래 🇰🇷 / 영어 중복 없음
- [x] interim: 원어 줄 opacity 1, 번역 줄 interimOpacity (ko/en 양쪽)
- [x] 수신 전용(STT 없음, TTLS 역할): config→payload 순서 정상
- [x] 같은 srcLang config 2회 push → STT 재시작 안 함
- [x] STT 인식 언어 4종 실측 (ko-KR / en-US / ja-JP / zh-CN)
- [x] 원문 숨김 / 번역 0개 안전망 회귀 없음
- [x] **실 파이프라인** — 가짜 인식기에 발화 주입 → gtx 실제 왕복 → en/ja 원어에서 한국어 번역 도착
- [x] `pnpm --filter @tikke/desktop typecheck` + `build`
- 결과: 실측 29건 전부 통과 (렌더 23 + 파이프라인 6)
- [ ] 배포 — **K가 "배포해" 할 때만**

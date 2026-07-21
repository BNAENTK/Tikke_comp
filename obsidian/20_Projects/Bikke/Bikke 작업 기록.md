# Bikke 작업 기록

`Bikke`(Bigo Live 채팅·선물 TTS Electron 앱) 관련 중요 작업 요약을 남기는 노트.

## 기록 원칙

- 날짜별로 짧게 남긴다.
- 결정, 변경, 검증 결과 중심으로 남긴다.
- 소스코드 원문과 secret은 남기지 않는다.

## 기록

### 2026-07-21 배포용 실행 파일 + 원격 잠금(킬스위치)

**배포용 설치 파일**
- electron-builder(NSIS 원클릭) 도입. `npm run dist` → `Bikke/dist/Bikke Setup 0.1.0.exe` (약 79MB).
- 패키징된 exe 실행 스모크 테스트 통과. 설치 시 바탕화면/시작메뉴 바로가기 자동 생성.
- 배포는 `Bikke-Setup-0.1.0.zip`(75MB)으로 압축해 카톡/드라이브 직접 전달 방식 선택.
- 아이콘 미지정(기본 Electron 아이콘), 코드서명 없음 → SmartScreen 경고 가능.

**원격 잠금 (킬스위치)**
- Supabase(tikke 프로젝트) `bikke_control` 테이블 `key='app'` 행의 `locked`/`message`로 제어.
- RLS 공개 읽기 전용 — anon 키로 쓰기 불가, 토글은 대시보드/service role만 가능.
- 앱이 시작 시 + 30초마다 REST 폴링. 잠금 시: bigo 수신 창 차단 + TTS 정지 + 전체화면 🔒 오버레이(커스텀 메시지) + 연결 시도 거부.
- 해제 시 오버레이 제거, 재연결은 사용자가 수동.
- 검증 완료: 실행 중이던 앱이 잠금 토글 후 30초 안에 잠기는 것을 스크린샷으로 확인.

**운영 방법**
- 잠그기: Claude에게 "Bikke 잠가" (메시지 지정 가능) → `update bikke_control set locked=true, message='...' where key='app';`
- 풀기: "Bikke 풀어" → `locked=false`
- 주의: 잠금 기능은 2026-07-21 빌드부터 포함. 이전 배포본에는 없음.

관련 파일: `Bikke/main.js`(폴링), `Bikke/renderer.js`(오버레이), `Bikke/context-notes.md`(상세 결정 기록)

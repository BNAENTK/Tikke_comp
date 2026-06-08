# OpenHue 스킬 요약

## 용도
- Philips Hue 조명, 방, scene을 CLI로 제어할 때 쓰는 스킬.

## 핵심 포인트
- Hue Bridge와 같은 로컬 네트워크에 있어야 한다.
- 첫 페어링 때는 물리 버튼을 눌러야 한다.
- light/room/scene 단위 제어가 가능하다.

## 자주 쓰는 흐름
1. `openhue get light|room|scene`
2. 대상 이름 확인
3. `openhue set ...`으로 on/off, 밝기, 색, scene 적용

## 주의사항
- 이름이 정확히 맞아야 한다.
- 컬러 기능은 지원되는 전구에서만 된다.
- 자동화와 cron에 엮기 좋지만 초기 연결 확인이 먼저다.

## 연결 노트
- [[Smart Home 스킬 묶음]]

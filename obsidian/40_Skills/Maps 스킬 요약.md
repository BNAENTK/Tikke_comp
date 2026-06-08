# Maps 스킬 요약

## 용도
- 위치 검색, 좌표 변환, 주변 장소, 경로, 거리, 타임존 같은 지도성 작업을 다루는 스킬.

## 핵심 포인트
- OpenStreetMap/Nominatim/OSRM 기반이라 API 키 없이 쓸 수 있다.
- 위치 핀, 주소, 좌표를 기준으로 nearby/directions/distance 작업이 가능하다.
- 무료 오픈 데이터 기반이라 가벼운 위치 인텔리전스 용도에 적합하다.

## 자주 쓰는 흐름
- place name → 좌표 검색
- 좌표 → 주소 reverse geocode
- 주변 카페/식당/병원 찾기
- 두 지점 간 거리/이동시간 확인
- turn-by-turn directions

## 주의사항
- routing 품질은 지역에 따라 차이가 있다.
- nearby는 좌표나 `--near` 주소 중 하나가 필요하다.
- 영업시간 같은 운영 정보는 OSM 데이터라 최신이 아닐 수 있다.

## 연결 노트
- [[Productivity 스킬 묶음]]
- [[Hermes 스킬 인덱스]]

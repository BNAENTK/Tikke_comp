# Excalidraw 스킬 요약

## 용도
- `.excalidraw` JSON 파일로 손그림 스타일 다이어그램을 만드는 스킬.

## 핵심 포인트
- 브라우저 Excalidraw에 바로 드롭해서 열 수 있다.
- architecture, flowchart, concept map, sequence diagram에 모두 쓸 수 있다.
- shape 자체에 label을 박는 게 아니라 text binding 구조를 써야 한다.

## 자주 쓰는 흐름
1. 도형/텍스트/화살표 설계
2. Excalidraw JSON envelope 작성
3. `.excalidraw` 파일 저장
4. 필요 시 업로드/공유

## 주의사항
- shape에 `label` 속성을 직접 넣으면 무시된다.
- z-order를 element 배열 순서로 관리해야 한다.
- 글자 크기를 너무 작게 잡으면 실사용성이 급격히 떨어진다.

## 연결 노트
- [[Architecture Diagram 스킬 요약]]
- [[Sketch 스킬 요약]]

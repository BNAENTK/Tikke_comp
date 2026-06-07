# TOP GIFT 오버레이

## 오버레이 URL

```
https://tikke-web.pages.dev/overlays/top-gift/?room={roomKey}
```

## URL 파라미터

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `room` | — | Tikke 클라우드 룸 키 |
| `name` | 낮잠잘래 | 닉네임 (테스트용) |
| `coins` | 10000 | 코인 수 (테스트용) |
| `scale` | 1 | 전체 크기 배율 |
| `primaryColor` | #55c7ff | 메인 네온 색상 (파란색) |
| `secondaryColor` | #8b6cff | 보조 네온 색상 (보라색) |
| `accentColor` | #ff5cf4 | 포인트 네온 색상 (핑크) |
| `duration` | 4500 | 표시 시간 ms |
| `animation` | pop | 등장 애니메이션 (pop/slide/bounce/glow) |
| `test` | 0 | 1이면 즉시 표시 |
| `debug` | 0 | 1이면 반투명 배경 표시 |

## 테스트 URL

```
https://tikke-web.pages.dev/overlays/top-gift/?name=낮잠잘래&coins=10000&test=1
https://tikke-web.pages.dev/overlays/top-gift/?name=낮잠잘래&coins=10000&test=1&debug=1
```

## 색상 변경 예시

```
?primaryColor=%2300ffcc&secondaryColor=%23aa00ff&accentColor=%23ff0088
```

## 크기 변경 예시

```
?scale=1.3
?scale=0.8
```

## TikTok LIVE Studio 설정

1. Tikke 앱 → 이벤트 오버레이 → "TOP GIFT" → URL 복사
2. Live Studio → 레이어 추가 → 링크 소스
3. 복사한 URL 붙여넣기
4. 브라우저 소스 크기: 1920 × 1080

## 콘솔 테스트 명령

브라우저 콘솔에서:
```javascript
showTopGiftOverlay({ name: "낮잠잘래", coins: 10000 });
showTopGiftOverlay({ name: "테스터", coins: 50000, animation: "bounce" });
```

커스텀 이벤트:
```javascript
window.dispatchEvent(new CustomEvent("tikke:top-gift", {
  detail: { name: "낮잠잘래", coins: 10000, primaryColor: "#55c7ff" }
}));
```

## TikTok 선물 이벤트 연결

Tikke 앱이 gift 이벤트를 받으면:
- 현재 세션 최고 코인 수와 비교
- 신기록이면 `top-gift` WS 이벤트 자동 전송
- 오버레이가 자동으로 갱신

코인 계산: `diamondCount × repeatCount`

## 주의사항

- 왕관/다이아몬드 SVG 디자인은 참고 이미지 기준 — 변경 금지
- 네온 글로우 색상은 URL 파라미터로만 조절
- 새로고침하면 최고 기록 초기화

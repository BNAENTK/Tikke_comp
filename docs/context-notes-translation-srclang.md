# 번역자막 원어 선택 — 결정 노트

## 요구 (K)
> "응 영어->한국어 방식도 만들어 원어는 일본어,영어,중국어 만 추가해
> 다른 언어를 선택하면 그 번역 언어와 한국어(원어) 의 위치를 바꾸면 간단하지?"

## 해석
"위치"는 **화면상 줄 순서**다. 스타일 교체가 아니다.
- 원문 줄은 항상 맨 위. 지금은 그 자리에 무조건 한국어가 박혀 있다.
- 원어를 영어로 고르면 영어가 그 자리로 올라가고, 한국어는 번역 줄로 내려간다.
- 각 언어는 **자기 폰트크기/색 설정을 그대로** 쓴다. 스타일 키를 맞바꾸지 않는다
  (설정 UI가 "한국어=원어" 의미로 굳어버려 나중에 더 헷갈린다).

## 핵심 함정 — `LANG_ORDER` 에 `ko` 가 없다
`LANG_ORDER = ['en','ja',...]` 는 **번역 대상 목록이자 렌더 순서**로 5곳에서 쓰인다.
번역 호출부만 고치면 `translations.ko` 가 만들어지고 broadcast까지 되지만
**렌더 루프가 ko를 안 도니까 화면에 안 뜬다** — 증상이 "번역이 안 나옴"이라 원인 찾기 나쁘다.
그래서 5곳을 각각 고치지 않고 헬퍼 하나로 모았다.

```js
const ALL_LANGS = ['ko','en','ja','zh-CN','zh-TW','fr','ar','th','de','es','pt','ru','vi','id'];
function srcLang()    { return currentStyle.srcLang || 'ko'; }
function otherLangs() { const s = srcLang(); return ALL_LANGS.filter(k => k !== s); }
function targetLangs(){ return otherLangs().filter(k => k === 'ko' || currentStyle.enabledLangs?.[k]); }
```

`srcLang()==='ko'` 이면 `otherLangs()` 는 옛 `LANG_ORDER` 와 **원소도 순서도 완전히 같다**.
이게 기본 경로 무회귀 보장이고, 시험에서 직접 단언한다.

한국어는 원어가 아닐 때 **항상 번역 대상**이다(`k === 'ko' || ...`).
원어가 외국어인데 한국어 번역이 꺼져 있으면 기능 자체가 무의미해서 토글을 두지 않았다.

## `ko` 하드코딩이 남아 있던 자리
- `addLine('ko', subtitle.original)` — 확정 카드 원문
- `buildLine('ko', '<span class="ko-type">')` — interim 타자기 (클래스명 `ko-type` 은 유지, 수술 범위 최소)
- `const op = lang === 'ko' ? 1 : ...` — 안 고치면 **원어가 영어일 때 원문 줄이 55% 반투명**이 된다
- `_myMemory` `langpair=ko|${tl}` — 폴백 번역기 방향
- `recog.lang = 'ko-KR'` — STT 인식 언어

## 안 바꾼 것 (의도)
- **gtx `sl=auto` 유지.** 명시 `sl=` 이 이론상 더 정확하지만 잘 돌아가는 기본 경로를 건드리는 변경이다.
  의심 지점은 **짧은 라틴 문자 발화**였다 — 한글은 문자만 봐도 한국어라 auto 가 틀릴 수 없지만
  "nice" / "wow" 같은 한두 단어 영어는 감지가 흔들릴 수 있다. 실측으로 확인:
  `nice→멋진`, `okay thanks→알았어 고마워`, `wow→우와` 전부 정상. 그대로 둔다.
  (일본어 `配信`→"배달" 오역은 감지 문제가 아니라 구글 사전 선택이다. `sl=ja` 로도 안 고쳐진다.)
- **타자기 속도(`TYPE_MS=24`, 초당 ~40자) 유지.** 영어는 같은 발화 길이에 글자 수가 2~4배라
  뒤처질까 봐 실측했다 — 73자 영어 interim 을 2.5초 안에 전부 따라잡는다. 말하는 속도(초당 ~12자)
  보다 3배 빨라 여유가 있다.
- `LANG_ORDER` 는 헬퍼로 대체돼 아무도 안 쓰게 돼서 **삭제**했다(이번 변경이 만든 고아).
  남겨 두면 다음 세션이 ko 빠진 옛 목록을 다시 집는다.
- **URL 파라미터 안 씀.** 설정은 WS `translation_config` push로만 전달한다.
  URL에 실으면 K가 TTLS에 붙여 둔 주소를 원어 바꿀 때마다 다시 복사해야 한다(금지 UX).
  창이 붙는 순간 `onClientConnect` 가 현재 설정을 밀어 준다.
- **한국어 반응어 사전**은 `srcLang()==='ko'` 로 게이트했다. 키가 전부 한글이라 영어 입력은
  어차피 매칭이 안 되지만, 의도를 코드에 남겨야 나중에 사전을 늘릴 때 안 샌다.

## STT 재시작 가드
`recog.lang` 은 돌아가는 중에 못 바꾼다 — 인식기를 새로 만들어야 한다.
그런데 `translation_config` 는 **색 하나만 만져도** 날아온다. 무조건 재시작하면
말하는 도중에 인식기가 죽는다. 그래서 `srcLang` 이 **실제로 달라졌을 때만** 재시작한다.

## 설정 push 경로가 둘이다
- `ipc.ts buildTranslationStyle` — WS 재연결/새로고침 복원용
- `TranslationOverlay.tsx sendStyleConfig` — 설정 바꾼 즉시 push
한쪽만 고치면 "새로고침하면 되는데 바꿀 땐 안 먹음"(또는 반대)이 된다. 둘 다 넣었다.

## worker 사본
TTLS 는 클라우드 사본(`apps/worker/public/overlay/translation.html`)을 읽는다.
desktop 만 고치면 K 화면은 옛 파일이다. 복사 완료 — 두 파일 해시 동일.

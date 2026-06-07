# 틱피니티 .tfc 사운드 임포트 — 리버스 완전 해독 (오프라인)

틱피니티 Export(`/tiktok/setup` → Export → Sound Alerts → Download) `.tfc` 파일을 **틱피니티 실행/로그인 없이 오프라인으로** 복호화. 2026-05-31 완전 크랙. 구현: `electron/services/tikfinity-import.ts` (Node crypto만, 의존성 0).

## 복호화 전체 레시피 (검증됨)

1. **outer**: `aesDecrypt(fileText, "lolsurghwi378ukasfjsdf_s")` → `"v3:<midB64>:<innerCT>"`
   - aesDecrypt = CryptoJS.AES.decrypt 호환 = OpenSSL `Salted__` + AES-256-CBC + EVP_BytesToKey(MD5). 정적키 모든 export 공통.
2. `mid = base64decode(midB64)` (표준 base64)
3. **innerKey = shash(mid, 3)** ← 핵심. 아래 참고.
4. **innerCT는 커스텀 base64 알파벳** (v3): 표준 대비 **U↔V, i↔j, r↔s 스왑**.
   - 표준: `ABCD...STUVWXYZabcd...ghijklmnopqrstuvwxyz0-9+/=`
   - 커스텀: `ABCD...STVUWXYZabcd...ghjiklmnopqsrtuvwxyz0-9+/=`
   - 커스텀→표준 문자 변환 후 표준 base64로 취급. (변환하면 magic이 `Salted__`로 정상화)
5. `innerPlain = aesDecrypt(innerStdB64, innerKey)` → `{"b64RawData":"..."}`
6. **b64RawData 디코드**: `JSON.parse(decodeURIComponent( base64decode( reverse(b64RawData) ) ))` → `{dynamicSettings:{soundsdatasource:"<JSON>"}, version, sourceChannelId}`
7. `soundsdatasource` JSON.parse → 사운드 배열

## shash(input, 3) — 내부키 유도 (window.shash 포팅)

틱피니티 `window.shash` = **커스텀 MD5 변형** (CryptoJS 아님 → 런타임 해시훅으로 안 잡혔던 이유).
- encVersion=3: `input = btoa(input + "dfgkjoi3kdjkfe")` 선처리 (salt)
- 커스텀 init 상수: `[0x12345678, 0x9abcdef0, 0xfedcba98, 0x87654321]` (표준 MD5 init 아님)
- rotate amount = `(i % 4) + 4` (표준 MD5 shift table 아님)
- F/G/H/I, g 인덱스, sin(i+1) K상수 = 표준 MD5
- 단일 블록(우리 입력 ≤56B)이라 그들 코드의 멀티블록 버그(DataView가 off 무시) 무관
- 검증: `shash("1k1j17q98q2qdaeh2ewain",3)=="bdeaa095df6764044377c14bac10e962"` ✓ (2쌍)

## 사운드 엔트리 → tikke 매핑

```json
{ "triggerId":"5655", "soundUrl":"https://www.myinstants.com/media/sounds/dry-fart.mp3",
  "soundName":"Fart", "volume":89, "enabled":true }
```
- `triggerId`(숫자) → `SoundRule.condition.giftId`
- `soundUrl`(직접 myinstants mp3) → `SoundFile.filePath` (원격 URL 그대로)
- `volume/100` → `SoundFile.volume`
- `enabled` → `SoundRule.enabled`

## 왜 오프라인인가 (BrowserWindow 폐기 이유)

처음엔 숨은 BrowserWindow에 틱피니티 띄워 걔네 코드로 복호화하려 했으나:
- 틱피니티가 Electron 임베디드 창에서 **로그인 직후 크래시** (`API is not defined` — `api` 셋업이 `setTimeout(...,3000)` 안이라 post-login XHR과 레이스)
- 파일 input 주입도 거부됨
→ webcrack로 app.js 디옵퓨스케이트 → `window.shash` + 커스텀 base64 알파벳 발견 → **순수 오프라인 복호화로 전환.** 로그인·창·DOM 의존 전부 제거.

## 한계
- **v3 전용.** 틱피니티가 포맷(v4 등)·정적키·shash salt 바꾸면 깨짐 → 명시적 에러 반환.
- 디옵퓨스케이트 산출물: `%LOCALAPPDATA%/Temp/wc_out/deobfuscated.js` (webcrack), probe `tffull_node.js`.

# Qwen3-TTS 서버

Qwen3-TTS 모델 기반 로컬 TTS 서버. 텍스트 → WAV/MP3 파일 변환.

## 요구사항

- Python 3.10+
- CUDA GPU 권장 (CPU는 매우 느림)
- ffmpeg (MP3 출력 시 필요)

---

## 설치

### 1. 가상환경 생성

```powershell
cd tikke\services\qwen-tts
python -m venv .venv
.venv\Scripts\activate
```

### 2. PyTorch 설치 (CUDA 버전에 맞게)

CUDA 12.1:
```powershell
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu121
```

CPU 전용:
```powershell
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu
```

### 3. 나머지 패키지 설치

```powershell
pip install -r requirements.txt
```

### 4. 환경변수 설정

```powershell
copy .env.example .env
# .env 파일을 열어서 QWEN_TTS_MODEL, QWEN_TTS_DEVICE 수정
```

---

## 실행

```powershell
python run_tts_server.py
```

서버 시작 시 모델을 HuggingFace Hub에서 자동 다운로드합니다 (최초 1회, 약 1~2GB).

로컬 모델 경로 사용:
```powershell
# .env 에서:
QWEN_TTS_MODEL=C:/models/qwen3-tts-0.6b
```

---

## API

### GET /health

서버 상태 및 모델 로드 여부 확인.

```
curl http://127.0.0.1:18186/health
```

응답:
```json
{
  "ok": true,
  "model_loaded": true,
  "model_name": "Qwen/Qwen3-TTS-0.6B",
  "device": "cuda"
}
```

---

### GET /voices

사용 가능한 목소리 목록.

```
curl http://127.0.0.1:18186/voices
```

응답:
```json
{
  "voices": [
    {"voice_id": "default", "display_name": "기본 목소리", "type": "preset", "speaker_name": "Chelsie"},
    {"voice_id": "female",  "display_name": "여성 목소리", "type": "preset", "speaker_name": "Chelsie"},
    {"voice_id": "male",    "display_name": "남성 목소리", "type": "preset", "speaker_name": "Ethan"},
    {"voice_id": "cute",    "display_name": "귀여운 목소리", "type": "preset", "speaker_name": "Cherry"}
  ]
}
```

---

### POST /tts

텍스트를 음성 파일로 변환.

```
curl -X POST http://127.0.0.1:18186/tts \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"안녕하세요!\", \"voice_id\": \"default\", \"speed\": 1.0, \"output_format\": \"wav\"}"
```

요청 바디:
| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `text` | string | 필수 | 변환할 텍스트 (최대 500자) |
| `voice_id` | string | `"default"` | 목소리 ID |
| `speed` | float | `1.0` | 말하기 속도 (0.5 ~ 2.0) |
| `output_format` | string | `"wav"` | `wav` 또는 `mp3` |

응답:
```json
{
  "success": true,
  "audio_path": "C:/...tikke/services/qwen-tts/outputs/tts/1716400000000_ab12cd34.wav",
  "duration": 3.52,
  "cached": false
}
```

오류 시:
```json
{"detail": "텍스트가 비어있습니다."}
```

---

### DELETE /cache

오래된 캐시 파일 정리.

```
curl -X DELETE "http://127.0.0.1:18186/cache?max_age_hours=24"
```

---

## 테스트

```powershell
# 기본 테스트
python scripts/test_tts_request.py

# 옵션 지정
python scripts/test_tts_request.py --text "안녕하세요 방송에 오신 걸 환영합니다" --voice male --speed 1.2 --format mp3
```

---

## 목소리 설정

`voices/` 폴더의 JSON 파일로 관리.

### preset 목소리

```json
{
  "voice_id": "my_preset",
  "display_name": "내 프리셋",
  "type": "preset",
  "speaker_name": "Chelsie"
}
```

`speaker_name`은 qwen-tts 모델이 지원하는 스피커 이름. 지원 목록은 아래에서 확인:
```python
from qwen_tts import Qwen3TTSModel
model = Qwen3TTSModel.from_pretrained("Qwen/Qwen3-TTS-0.6B")
print(model.speakers)  # 또는 help(model.generate)
```

### clone 목소리 (보이스 클로닝)

`voices/custom/` 폴더에 JSON 파일 추가:

```json
{
  "voice_id": "my_voice",
  "display_name": "내 목소리",
  "type": "clone",
  "ref_audio_path": "voices/prompts/my_voice_ref.wav",
  "ref_text": "이것은 참조 오디오의 전사 텍스트입니다."
}
```

참조 오디오 조건:
- 3~10초 길이 권장
- 44100Hz 또는 22050Hz WAV
- 배경 소음 없는 깨끗한 녹음

---

## 환경변수

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `QWEN_TTS_MODEL` | `Qwen/Qwen3-TTS-0.6B` | 모델 이름 또는 로컬 경로 |
| `QWEN_TTS_DEVICE` | `auto` | `auto` / `cuda` / `cuda:0` / `cpu` |
| `QWEN_TTS_PORT` | `18186` | 서버 포트 |
| `QWEN_TTS_OUTPUT_DIR` | `outputs/tts` | 오디오 저장 폴더 |
| `QWEN_TTS_MAX_TEXT` | `500` | 최대 입력 글자 수 |

---

## 1.7B 모델로 교체

`.env` 또는 `config/tts_config.json` 에서 모델명만 변경:

```
QWEN_TTS_MODEL=Qwen/Qwen3-TTS-1.7B
```

또는 tikketone 엔진 서버와 같이 로컬 경로:
```
QWEN_TTS_MODEL=C:/models/tikketone-1.7b
```

---

## 포트 참고 (Tikke 서비스 포트)

| 서비스 | 포트 |
|--------|------|
| Tikke 오버레이 HTTP | 18181 |
| Tikke 오버레이 WS | 18182 |
| Tikketone 연동 서버 | 18183 |
| Tikketone Python 엔진 | 18184 |
| Tikketone OAuth 콜백 | 18185 |
| **Qwen3-TTS 서버** | **18186** |

# 유튜브 음원 추출 (ytmp3) 컨텍스트 노트

## 방식 결정
- ytmp3.gl 같은 외부 변환 사이트 API 미사용 — 차단/불안정. 로컬 yt-dlp + ffmpeg 방식 채택.
- 바이너리는 앱에 번들하지 않고 온디맨드 다운로드 (tikketonic-service.ts 패턴 재사용).
  - yt-dlp.exe: GitHub yt-dlp/yt-dlp latest release (~17MB)
  - ffmpeg: GitHub yt-dlp/FFmpeg-Builds win64-gpl zip → Windows 내장 tar(bsdtar)로 압축 해제 → ffmpeg.exe/ffprobe.exe만 보관
- 저장 위치: userData/ytmp3 (바이너리), 출력은 사용자 다운로드 폴더.

## 추출 커맨드
- `yt-dlp -x --audio-format mp3 --audio-quality 0 --no-playlist --newline --no-simulate --print after_move:filepath --ffmpeg-location <binDir> -o <downloads>/%(title)s.%(ext)s <url>`
- 진행률: stdout `[download]  xx.x%` 파싱, 최종 파일 경로는 `--print after_move:filepath` 마지막 줄.
- URL 검증: youtube.com/youtu.be/music.youtube.com만 허용, `-` 시작 문자열 거부 (플래그 주입 방지).

## 구간 잘라내기
- 방식 변경: 시간 텍스트 입력(--download-sections) → 추출 후 편집기 방식. K님 요청 — "편집기처럼 직접 잘라내기".
- 전체 mp3 추출 완료 → 결과 목록 "✂ 잘라내기" → TrimEditor: `tikke-sound://` 프로토콜로 로컬 mp3 재생, 타임라인 트랙에 시작/끝 노란 핸들 드래그, 구간 재생 미리듣기(끝 도달 시 자동 정지), 트랙 클릭 시킹.
- 저장: ffmpeg `-i in -ss start -to end -c copy` 스트림 복사 (재인코딩 없음, 즉시 완료). 출력 `원본명 (m.ss-m.ss).mp3`.

## UI
- 사이드바 미디어 그룹 "유튜브 음원 추출" (id: ytmp3).
- 미설치 시 구성요소 다운로드 버튼 + 로그. 설치 후 URL 입력 → 추출 → 진행률 바 → 결과(파일명 + 폴더 열기).

# n8n 워크플로우 저장소

이 폴더에 n8n에서 Export한 워크플로우 JSON 파일을 저장한다.

## 파일 명명 규칙

- `main-biz-support-bot.json` — 메인 워크플로우
- `sub-naver-works-auth.json` — 인증 서브 워크플로우

## Export 방법

1. n8n 에디터에서 해당 워크플로우 열기
2. 우측 상단 `...` 메뉴 → `Export`
3. JSON 파일로 저장 후 이 폴더에 배치

## Import 방법

1. n8n 에디터에서 `Import from File`
2. JSON 파일 선택
3. Credentials (API Key 등)은 별도 설정 필요

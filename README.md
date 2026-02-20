# BizSupportBot — 경영지원 문의 응대 AI 봇

## 개요

임직원 대상 경영지원(인사/총무/회계) 문의를 네이버 웍스 1:1 봇으로 자동 응대하는 AI 시스템.
사내 규정 기반으로 답변하며, 답변 불가 시 담당자 안내로 에스컬레이션한다.

## 기술 스택

| 구분 | 도구 | 비고 |
|------|------|------|
| 채널 | 네이버 웍스 Bot | 1:1 대화방 |
| 오케스트레이션 | n8n (셀프호스팅) | 기존 인프라 활용 |
| AI 모델 | Google Gemini 2.0 Flash | 속도 + 비용 최적 |
| 데이터 소스 | Google Sheets | FAQ, 규정, 담당자, 대화이력 |
| 데이터 소스 | Confluence / Google Docs | 규정 원문 (2차) |

## 프로젝트 목적

| 우선순위 | 목표 |
|---------|------|
| 1 | 경영지원 본부의 반복 문의 응대 공수 절감 |
| 2 | 임직원 문의 응답 속도 개선 |
| 3 | AI 활용 파일럿 사례 확보 |

**성과 측정**: 봇 도입 전/후 문의 자동 처리율, 절감 시간, ROI로 효과 증명

## 프로젝트 구조

```
BizSupportBot/
├── README.md
├── docs/
│   ├── 01-architecture.md         # 시스템 아키텍처 + 최적화 전략
│   ├── 02-data-structure.md       # Google Sheets 설계 (9개 시트)
│   ├── 03-n8n-workflow.md         # n8n 노드 설계 상세
│   ├── 04-development-plan.md     # 개발 플랜 + 일정 (6 Phase)
│   ├── 05-measurement-dashboard.md # 성과 측정 + 대시보드 + ROI 모델
│   └── 06-operations-governance.md # 운영 거버넌스 + 비개발자 관리
├── prompts/
│   └── system-prompt.md           # Gemini 시스템 프롬프트
├── n8n/
│   └── workflows/                 # n8n JSON export 저장소
└── data/
    └── templates/                 # Google Sheets 템플릿 (CSV)
        ├── faq.csv
        ├── regulations.csv
        ├── escalation-contacts.csv
        └── conversation-history.csv
```

## 핵심 설계 원칙

1. **3-Tier 응답 전략**: FAQ 즉시응답 → Gemini 규정기반 생성 → 에스컬레이션
   - FAQ 150행 이상 시 `FAQ_키워드인덱스` 기반 후보 추출로 전환
2. **규정 기반 강제**: 규정에 없는 내용은 답변하지 않음 (할루시네이션 방지)
3. **멀티턴 대화**: 최근 대화 맥락을 유지하여 자연스러운 후속 질문 처리
4. **비개발자 운영 가능**: Google Sheets 편집만으로 FAQ/규정 관리 (코드 수정 불필요)
5. **성과 증명 내장**: 운영 로그 → 대시보드 → ROI 자동 계산

## 문서 읽는 순서

1. `docs/01-architecture.md` → 전체 구조 이해
2. `docs/02-data-structure.md` → 데이터 준비
3. `docs/03-n8n-workflow.md` → 워크플로우 구현
4. `docs/04-development-plan.md` → 일정 및 작업 순서
5. `docs/05-measurement-dashboard.md` → 성과 측정 + 대시보드
6. `docs/06-operations-governance.md` → 운영 + 비개발자 관리
7. `prompts/system-prompt.md` → 프롬프트 튜닝

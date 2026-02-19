# BizSupportBot 전체 설계서

> **생성일**: 2026-02-20

---

## 목차

1. [프로젝트 개요 (README)](#프로젝트-개요)
2. [시스템 아키텍처](#시스템-아키텍처)
3. [데이터 구조 설계](#데이터-구조-설계)
4. [n8n 워크플로우 설계](#n8n-워크플로우-설계)
5. [개발 플랜](#개발-플랜)
6. [성과 측정 & 대시보드 설계](#성과-측정--대시보드-설계)
7. [운영 거버넌스 & 비개발자 관리 구조](#운영-거버넌스--비개발자-관리-구조)
8. [Gemini 시스템 프롬프트](#gemini-시스템-프롬프트)

---

<!-- pagebreak -->

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

---

<!-- pagebreak -->

# 시스템 아키텍처

## 전체 흐름

```
[임직원]
   ↓ 메시지 전송
[네이버웍스 1:1 봇]
   ↓ Callback (Webhook)
[n8n 메인 워크플로우]
   ├─→ [대화 이력 조회]        ← Google Sheets (대화이력)
   ├─→ [FAQ 캐시 매칭]         ← Google Sheets (FAQ)
   │     ├─ 매칭됨 → 즉시 응답 (Gemini 호출 X)
   │     └─ 미매칭 ↓
   ├─→ [의도 분류 - Gemini]    → category, isAnswerable
   │     ├─ 답변 가능 ↓
   │     └─ 답변 불가 → 에스컬레이션 안내
   ├─→ [규정 조회]             ← Google Sheets (카테고리별 규정)
   ├─→ [응답 생성 - Gemini]    → 규정 기반 답변
   ├─→ [대화 이력 저장]        → Google Sheets (대화이력)
   └─→ [네이버웍스 응답 전송]
```

## 3-Tier 응답 전략

속도와 정확도를 동시에 잡기 위한 계층형 응답 구조.

| Tier | 조건 | 처리 방식 | 예상 응답 시간 |
|------|------|----------|--------------|
| **Tier 1** | FAQ 정확 매칭 | 사전 검증된 답변 즉시 반환 | ~1초 |
| **Tier 2** | FAQ 미매칭 + 답변 가능 | Gemini로 규정 기반 응답 생성 | ~3-5초 |
| **Tier 3** | 답변 불가 판단 | 담당자 정보 안내 (에스컬레이션) | ~2초 |

### Tier 1이 중요한 이유

- 임직원 문의의 60-70%는 반복 질문 (연차 며칠? 법인카드 한도? 출장비 정산 기한?)
- FAQ 매칭으로 Gemini 호출 없이 즉시 응답 → **비용 절감 + 속도 향상 + 정확도 보장**
- FAQ 답변은 경영지원 담당자가 직접 검수한 내용 → 할루시네이션 위험 0

## 속도 최적화 설계

### 1. FAQ 캐시 레이어 (Tier 1)
- Google Sheets에 자주 묻는 질문+답변 매핑 테이블 운영
- 키워드 기반 매칭으로 Gemini API 호출 자체를 스킵
- 매칭 로직: 사용자 메시지에 FAQ 키워드가 포함되면 매칭

### 2. 카테고리별 규정 분리 (Tier 2)
- 규정 데이터를 인사/총무/회계 카테고리별 별도 시트로 분리
- 의도 분류 결과에 따라 해당 카테고리 규정만 조회
- 전체 규정을 매번 보내지 않으므로 Gemini context 최소화 → 속도 향상

### 3. Gemini Flash 모델
- `gemini-2.0-flash` 사용 (Pro 대비 5-10배 빠름, 비용 1/10)
- 경영지원 문의 응대 수준에서는 Flash로 충분
- 복잡한 판단이 필요한 경우만 Pro 고려 (현재 불필요)

### 4. 멀티턴 컨텍스트 제한
- 대화 이력은 최근 5건 + 30분 이내만 조회
- 오래된 대화는 새 세션으로 취급 → 불필요한 context 누적 방지

## 정확도 최적화 설계

### 1. 규정 기반 강제 (Grounding)
- 시스템 프롬프트에 "제공된 규정에만 기반하여 답변" 명시
- 규정에 없는 내용 → "해당 내용은 규정에서 확인되지 않습니다" + 에스컬레이션

### 2. 출처 명시
- 모든 답변에 근거 규정 표시: "근거: 연차휴가 규정 제5조"
- 사용자가 답변의 신뢰성을 직접 확인 가능

### 3. 신뢰도 기반 에스컬레이션
- Gemini 응답에 confidence 필드 포함 요청
- confidence < 0.7 → 답변과 함께 "정확한 확인이 필요하시면 담당자에게 문의해주세요" 병기
- confidence < 0.4 → 답변 미제공, 바로 에스컬레이션

### 4. FAQ 우선 전략
- 검증된 FAQ 답변이 있으면 Gemini 생성 답변보다 우선
- FAQ 테이블은 경영지원팀이 직접 관리 → 정확도 보장

### 5. 피드백 루프
- 에스컬레이션된 질문을 주기적으로 검토
- 반복되는 에스컬레이션 → FAQ에 추가
- 오답 제보 → FAQ/규정 데이터 수정

## 네이버웍스 봇 인증 흐름

```
[n8n]
  ↓ 1. JWT 생성 (Service Account + Private Key)
[NAVER WORKS Auth API]
  ↓ 2. Access Token 발급 (유효기간: 24시간)
[n8n]
  ↓ 3. Access Token으로 Bot API 호출
[NAVER WORKS Bot API]
```

### 인증 정보 필요 목록
- Client ID / Client Secret (Developer Console)
- Service Account ID
- Private Key (RSA)
- Bot ID

### API 엔드포인트
| 용도 | Method | URL |
|------|--------|-----|
| 토큰 발급 | POST | `https://auth.worksmobile.com/oauth2/v2.0/token` |
| 메시지 전송 | POST | `https://www.worksapis.com/v1.0/bots/{botId}/users/{userId}/messages` |
| 메시지 수신 | - | Webhook Callback URL (n8n Webhook 주소 등록) |

## 컴포넌트 다이어그램

```
┌─────────────────────────────────────────────────┐
│                   n8n Server                     │
│                                                  │
│  ┌──────────────┐    ┌────────────────────────┐ │
│  │ Sub-workflow  │    │    Main Workflow        │ │
│  │ 네이버웍스 인증│    │                        │ │
│  │              │    │  Webhook                │ │
│  │ JWT 생성     │    │    ↓                    │ │
│  │    ↓         │◄───│  메시지 파싱             │ │
│  │ 토큰 발급    │    │    ↓                    │ │
│  │    ↓         │───►│  대화 이력 조회          │ │
│  │ 토큰 캐시    │    │    ↓                    │ │
│  └──────────────┘    │  FAQ 매칭 ──► 즉시응답   │ │
│                      │    ↓ (미매칭)            │ │
│                      │  의도 분류 (Gemini)      │ │
│                      │    ↓                    │ │
│                      │  규정 조회               │ │
│                      │    ↓                    │ │
│                      │  응답 생성 (Gemini)      │ │
│                      │    ↓                    │ │
│                      │  대화 이력 저장           │ │
│                      │    ↓                    │ │
│                      │  응답 전송               │ │
│                      └────────────────────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │          Google Sheets (4개 시트)          │   │
│  │  FAQ │ 규정(인사/총무/회계) │ 담당자 │ 이력  │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │          Gemini API (Flash)               │   │
│  │  의도 분류 / 응답 생성                      │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

<!-- pagebreak -->

# 데이터 구조 설계

## 개요

모든 데이터는 Google Sheets에 저장한다. 하나의 스프레드시트 파일 안에 시트(탭)를 분리하여 관리.
규정량이 많아지면 Confluence/Google Docs 연동을 2차로 추가한다.

## 스프레드시트 구조

```
[BizSupportBot_Data] (Google Sheets 파일 1개)
├── 시트: FAQ                  ← 경영지원 실무자 편집
├── 시트: 규정_인사             ← 경영지원 실무자 편집
├── 시트: 규정_총무             ← 경영지원 실무자 편집
├── 시트: 규정_회계             ← 경영지원 실무자 편집
├── 시트: 담당자                ← 경영지원 실무자 편집
├── 시트: 대화이력              ← 봇 전용 (자동 기록)
├── 시트: 운영로그              ← 봇 전용 (성과 측정용)
├── 시트: 대시보드_일간          ← 자동 집계 (수식/차트)
└── 시트: 대시보드_월간          ← 자동 집계 (수식/차트)
```

> 시트별 권한 설계는 `06-operations-governance.md` 참조

---

## 시트 1: FAQ

자주 묻는 질문과 검증된 답변. Tier 1 즉시 응답에 사용.

| 컬럼 | 타입 | 설명 | 예시 |
|------|------|------|------|
| faqId | string | 고유 ID | FAQ-HR-001 |
| category | string | 대분류 (인사/총무/회계) | 인사 |
| subCategory | string | 소분류 | 연차휴가 |
| keywords | string | 매칭 키워드 (쉼표 구분) | 연차,남은연차,잔여연차,휴가일수 |
| question | string | 대표 질문 | 연차 남은 일수를 어떻게 확인하나요? |
| answer | string | 검증된 답변 | 네이버웍스 > 근태관리 > 잔여휴가에서 확인 가능합니다. |
| source | string | 근거 규정 | 연차휴가 규정 제3조 |
| isActive | boolean | 활성 여부 | TRUE |
| lastUpdated | date | 최종 수정일 | 2026-02-19 |

### FAQ 작성 가이드
- **keywords**: 사용자가 실제로 사용할 단어를 다양하게 등록 (구어체 포함)
- **answer**: 구체적이고 실행 가능한 답변 (경로, 기한, 금액 등 명시)
- **category별 최소 10-20개**로 시작, 운영하면서 확대

---

## 시트 2-4: 규정_인사 / 규정_총무 / 규정_회계

Gemini가 참조할 규정 데이터. 카테고리별 별도 시트.

| 컬럼 | 타입 | 설명 | 예시 |
|------|------|------|------|
| regulationId | string | 고유 ID | REG-HR-001 |
| title | string | 규정 제목 | 연차휴가 사용 규정 |
| section | string | 조항 번호 | 제5조 |
| content | string | 규정 내용 (본문) | 연차휴가는 입사일 기준으로... |
| summary | string | 요약 (1-2줄) | 연차 부여 기준 및 사용 방법 |
| effectiveDate | date | 시행일 | 2025-01-01 |
| sourceUrl | string | 원문 링크 (Confluence 등) | https://confluence.xxx.com/... |
| tags | string | 검색 태그 (쉼표 구분) | 연차,휴가,부여,사용 |
| lastUpdated | date | 최종 수정일 | 2026-02-01 |

### 규정 데이터 입력 가이드

**이미 디지털화된 규정:**
1. Confluence/Google Docs에서 내용 복사
2. 조항 단위로 행 분리 (한 행 = 한 조항)
3. sourceUrl에 원문 링크 기록

**신규 디지털화 규정:**
1. PDF/한글 파일에서 조항별 텍스트 추출
2. 동일한 구조로 시트에 입력
3. 원문 파일은 Google Drive에 업로드 후 링크 연결

### content 작성 규칙
- **한 행에 한 조항**: "제5조 (연차휴가 부여) 1. 입사 1년 미만 직원은..."
- **너무 길지 않게**: 조항 하나가 500자를 넘으면 항목별로 행을 분리
- **핵심 수치 명시**: 금액, 기간, 기한 등은 반드시 포함

---

## 시트 5: 담당자

에스컬레이션 시 안내할 담당자 정보.

| 컬럼 | 타입 | 설명 | 예시 |
|------|------|------|------|
| category | string | 대분류 | 인사 |
| subCategory | string | 소분류 | 연차휴가 |
| team | string | 담당팀 | 인사팀 |
| name | string | 담당자명 | 홍길동 |
| contact | string | 연락처 (내선/메신저) | 내선 1234 |
| email | string | 이메일 | hong@company.com |
| note | string | 비고 | 오전에 연락 선호 |

### 에스컬레이션 안내 메시지 포맷

```
죄송합니다. 해당 문의는 정확한 확인이 필요한 사항입니다.

📋 담당 부서: {team}
👤 담당자: {name}
📞 연락처: {contact}

위 담당자에게 직접 문의해주시면 빠르게 안내받으실 수 있습니다.
```

---

## 시트 6: 대화이력

멀티턴 대화를 위한 대화 로그.

| 컬럼 | 타입 | 설명 | 예시 |
|------|------|------|------|
| sessionId | string | 세션 ID (userId_날짜) | user123_20260219 |
| userId | string | 네이버웍스 사용자 ID | user123 |
| timestamp | datetime | 메시지 시각 | 2026-02-19 14:30:00 |
| role | string | 발화자 (user / bot) | user |
| message | string | 메시지 내용 | 연차 며칠 남았어? |
| category | string | 분류된 카테고리 | 인사 |
| tier | string | 응답 Tier (1/2/3) | 1 |
| isEscalated | boolean | 에스컬레이션 여부 | FALSE |

### 멀티턴 대화 이력 관리 규칙

- **조회 범위**: 동일 userId의 최근 5건 + 30분 이내
- **30분 초과** 시 새 세션으로 취급 (sessionId 갱신)
- **이력 정리**: 7일 이상 지난 이력은 월 1회 아카이빙 (별도 시트 이동)
- **용도**: Gemini 호출 시 conversation history로 전달

---

## 데이터 볼륨 예상 및 확장 전략

### MVP 기준 예상 볼륨

| 시트 | 예상 행 수 | 비고 |
|------|-----------|------|
| FAQ | 50-100행 | 카테고리별 20-30개 |
| 규정 (3개 시트 합계) | 100-300행 | 조항 단위 분리 시 |
| 담당자 | 10-20행 | 소분류별 1명 |
| 대화이력 | 일 50-200행 | 사용량에 따라 변동 |

### Google Sheets 한계 및 대응

| 한계 | 임계점 | 대응 방안 |
|------|--------|----------|
| 규정 데이터 과다 | 규정 1,000행 이상 | Confluence API 연동으로 전환 (2차) |
| 대화이력 누적 | 10,000행 이상 | 월별 아카이빙 + 오래된 데이터 삭제 |
| API 호출 제한 | 분당 60회 | n8n에서 캐싱 처리 |

### 2차 확장: Confluence 연동

규정량이 Google Sheets 한 시트에 담기 어려울 때:
1. Confluence에 규정 원문 저장 (카테고리별 Space/Page)
2. n8n에서 Confluence REST API로 검색
3. Google Sheets FAQ는 그대로 유지 (Tier 1 캐시 역할)

---

<!-- pagebreak -->

# n8n 워크플로우 설계

## 워크플로우 구성

총 2개 워크플로우로 구성한다.

| 워크플로우 | 역할 |
|-----------|------|
| **Main: BizSupportBot** | 메시지 수신 → 처리 → 응답 (핵심 로직) |
| **Sub: NaverWorks Auth** | 네이버웍스 인증 토큰 발급 및 캐싱 |

---

## Sub-workflow: NaverWorks Auth

네이버웍스 API 호출에 필요한 Access Token을 발급/관리하는 서브 워크플로우.
메인 워크플로우에서 호출하여 유효한 토큰을 받아간다.

### 노드 구성

```
[Execute Workflow Trigger]
    ↓
[Function: generateJWT]
    ↓
[HTTP Request: getAccessToken]
    ↓
[Return: accessToken]
```

### 노드 상세

#### Node 1: Execute Workflow Trigger
- **타입**: Execute Workflow Trigger
- **역할**: 메인 워크플로우에서 호출 시 실행

#### Node 2: Function — generateJWT
- **타입**: Code (JavaScript)
- **역할**: Service Account 정보로 JWT 생성

```javascript
// JWT 생성 로직
const header = {
  alg: "RS256",
  typ: "JWT"
};

const now = Math.floor(Date.now() / 1000);
const payload = {
  iss: $env.NAVER_WORKS_CLIENT_ID,       // Developer Console Client ID
  sub: $env.NAVER_WORKS_SERVICE_ACCOUNT,  // Service Account ID
  iat: now,
  exp: now + 3600 // 1시간
};

// n8n의 crypto 모듈로 RSA 서명
// 실제 구현 시 n8n의 JWT 라이브러리 또는 외부 모듈 활용
const jwt = createJWT(header, payload, $env.NAVER_WORKS_PRIVATE_KEY);

return [{ json: { jwt } }];
```

#### Node 3: HTTP Request — getAccessToken
- **타입**: HTTP Request
- **Method**: POST
- **URL**: `https://auth.worksmobile.com/oauth2/v2.0/token`
- **Content-Type**: `application/x-www-form-urlencoded`
- **Body**:
  ```
  grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
  assertion={{$json.jwt}}
  client_id={{$env.NAVER_WORKS_CLIENT_ID}}
  client_secret={{$env.NAVER_WORKS_CLIENT_SECRET}}
  scope=bot bot.message
  ```
- **응답 예시**:
  ```json
  {
    "access_token": "xxxxx",
    "token_type": "Bearer",
    "expires_in": 86400
  }
  ```

---

## Main Workflow: BizSupportBot

### 전체 노드 흐름

```
[1. Webhook] ─→ [2. parseMessage] ─→ [3. fetchHistory]
                                          ↓
                                    [4. fetchFAQ]
                                          ↓
                                    [5. matchFAQ]
                                          ↓
                                    [6. IF: faqMatched?]
                                     ├─ YES → [7. formatFAQResponse] ──────────┐
                                     └─ NO  → [8. classifyIntent (Gemini)] ─→  │
                                                    ↓                          │
                                              [9. IF: isAnswerable?]           │
                                               ├─ NO → [10. fetchContact]     │
                                               │            ↓                  │
                                               │       [11. formatEscalation]──┤
                                               └─ YES ↓                       │
                                              [12. fetchRegulations]           │
                                                    ↓                          │
                                              [13. generateResponse (Gemini)]  │
                                                    ↓                          │
                                              [14. formatResponse] ────────────┤
                                                                               ↓
                                                                  [15. saveHistory]
                                                                         ↓
                                                                  [16. getToken (Sub)]
                                                                         ↓
                                                                  [17. sendMessage]
```

### 노드별 상세 설계

---

#### Node 1: Webhook (Trigger)

메시지 수신 트리거.

- **타입**: Webhook
- **Method**: POST
- **Path**: `/biz-support-bot`
- **Authentication**: None (네이버웍스에서 호출, 별도 검증)
- **Response**: Immediately (200 OK 즉시 반환 → 비동기 처리)

**수신 페이로드 예시 (네이버웍스):**
```json
{
  "type": "message",
  "source": {
    "userId": "user123",
    "domainId": 12345
  },
  "content": {
    "type": "text",
    "text": "연차 남은 일수를 어떻게 확인하나요?"
  },
  "issuedTime": "2026-02-19T14:30:00Z"
}
```

---

#### Node 2: Function — parseMessage

메시지 페이로드를 파싱하여 필요한 값 추출.

- **타입**: Code (JavaScript)

```javascript
const body = $input.first().json.body;

// 텍스트 메시지만 처리 (이미지 등은 무시)
if (body.content?.type !== 'text') {
  return [{
    json: {
      skip: true,
      reason: 'non-text message'
    }
  }];
}

return [{
  json: {
    userId: body.source.userId,
    domainId: body.source.domainId,
    messageText: body.content.text.trim(),
    timestamp: body.issuedTime || new Date().toISOString(),
    skip: false
  }
}];
```

---

#### Node 3: Google Sheets — fetchHistory

멀티턴 대화를 위한 최근 대화 이력 조회.

- **타입**: Google Sheets (Read)
- **Spreadsheet**: BizSupportBot_Data
- **Sheet**: 대화이력
- **Operation**: Read Rows
- **Filter**: `userId` = `{{$json.userId}}`

**후처리 (Function 노드 연결):**
```javascript
// 30분 이내 + 최근 5건만 필터링
const now = new Date();
const thirtyMinAgo = new Date(now - 30 * 60 * 1000);
const userId = $('parseMessage').first().json.userId;

const recentHistory = $input.all()
  .map(item => item.json)
  .filter(row => row.userId === userId && new Date(row.timestamp) > thirtyMinAgo)
  .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp))
  .slice(0, 5)
  .reverse(); // 시간순 정렬

return [{ json: { conversationHistory: recentHistory } }];
```

---

#### Node 4: Google Sheets — fetchFAQ

FAQ 데이터 전체 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: FAQ
- **Filter**: `isActive` = `TRUE`

> **성능 팁**: FAQ가 100건 이하면 전체 조회 후 n8n 내에서 매칭해도 충분.
> 100건 이상이면 Google Sheets API의 filter 파라미터 활용.

---

#### Node 5: Function — matchFAQ

사용자 메시지와 FAQ 키워드를 매칭.

- **타입**: Code (JavaScript)

```javascript
const userMessage = $('parseMessage').first().json.messageText;
const faqList = $('fetchFAQ').all().map(item => item.json);

// 키워드 매칭: FAQ의 keywords 중 하나라도 사용자 메시지에 포함되면 매칭
let bestMatch = null;
let maxKeywordCount = 0;

for (const faq of faqList) {
  const keywords = faq.keywords.split(',').map(k => k.trim());
  const matchedCount = keywords.filter(kw => userMessage.includes(kw)).length;

  if (matchedCount > maxKeywordCount) {
    maxKeywordCount = matchedCount;
    bestMatch = faq;
  }
}

// 최소 1개 이상 키워드 매칭 필요
if (bestMatch && maxKeywordCount >= 1) {
  return [{
    json: {
      faqMatched: true,
      matchedFAQ: bestMatch,
      matchedKeywords: maxKeywordCount
    }
  }];
}

return [{ json: { faqMatched: false } }];
```

---

#### Node 6: IF — faqMatched?

- **타입**: IF
- **Condition**: `{{$json.faqMatched}}` equals `true`
- **True**: → Node 7 (formatFAQResponse)
- **False**: → Node 8 (classifyIntent)

---

#### Node 7: Function — formatFAQResponse

FAQ 매칭 시 즉시 응답 포맷팅.

```javascript
const faq = $json.matchedFAQ;

const response = `${faq.answer}\n\n📌 근거: ${faq.source}`;

return [{
  json: {
    responseText: response,
    tier: '1',
    category: faq.category,
    isEscalated: false
  }
}];
```

---

#### Node 8: HTTP Request — classifyIntent (Gemini)

FAQ 미매칭 시 Gemini로 의도 분류.

- **타입**: HTTP Request
- **Method**: POST
- **URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={{$env.GEMINI_API_KEY}}`
- **Headers**: `Content-Type: application/json`
- **Body**:

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{
        "text": "다음 사용자 메시지를 분류해주세요.\n\n[사용자 메시지]\n{{$('parseMessage').first().json.messageText}}\n\n[이전 대화]\n{{$('fetchHistory').first().json.conversationHistory}}\n\n아래 JSON 형식으로만 응답하세요:\n{\"category\": \"인사|총무|회계|기타\", \"subCategory\": \"소분류명\", \"isAnswerable\": true|false, \"confidence\": 0.0~1.0}\n\n- category: 인사, 총무, 회계 중 하나. 해당 없으면 기타\n- isAnswerable: 경영지원 규정으로 답변 가능한 질문이면 true\n- confidence: 분류 확신도 (0.0~1.0)"
      }]
    }
  ],
  "generationConfig": {
    "temperature": 0.1,
    "responseMimeType": "application/json"
  }
}
```

**후처리 (Function):**
```javascript
const result = JSON.parse($json.candidates[0].content.parts[0].text);

return [{
  json: {
    category: result.category,
    subCategory: result.subCategory,
    isAnswerable: result.isAnswerable && result.category !== '기타',
    confidence: result.confidence
  }
}];
```

---

#### Node 9: IF — isAnswerable?

- **타입**: IF
- **Condition**: `{{$json.isAnswerable}}` equals `true`
- **True**: → Node 12 (fetchRegulations)
- **False**: → Node 10 (fetchContact)

---

#### Node 10: Google Sheets — fetchContact

에스컬레이션용 담당자 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: 담당자
- **Filter**: `category` = `{{$json.category}}`

---

#### Node 11: Function — formatEscalation

에스컬레이션 안내 메시지 생성.

```javascript
const contact = $('fetchContact').first().json;
const category = $('classifyIntent').first().json.category;

let responseText;

if (contact && contact.name) {
  responseText = `해당 문의는 정확한 확인이 필요한 사항입니다.\n\n` +
    `📋 담당 부서: ${contact.team}\n` +
    `👤 담당자: ${contact.name}\n` +
    `📞 연락처: ${contact.contact}\n\n` +
    `위 담당자에게 직접 문의해주시면 빠르게 안내받으실 수 있습니다.`;
} else {
  responseText = `해당 문의는 경영지원본부에 직접 문의해주시기 바랍니다.`;
}

return [{
  json: {
    responseText,
    tier: '3',
    category,
    isEscalated: true
  }
}];
```

---

#### Node 12: Google Sheets — fetchRegulations

분류된 카테고리에 해당하는 규정 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: 카테고리에 따라 동적 선택
  - 인사 → `규정_인사`
  - 총무 → `규정_총무`
  - 회계 → `규정_회계`

> **n8n 구현 팁**: Switch 노드로 카테고리별 분기하거나,
> Function 노드에서 시트명을 동적으로 생성하여 Google Sheets API 직접 호출.

**후처리 (Function):**
```javascript
// 관련 규정만 필터링 (subCategory 또는 tags 매칭)
const subCategory = $('classifyIntent').first().json.subCategory;
const allRegulations = $input.all().map(item => item.json);

const relevant = allRegulations.filter(reg =>
  reg.tags?.includes(subCategory) ||
  reg.title?.includes(subCategory) ||
  reg.summary?.includes(subCategory)
);

// 최대 10건으로 제한 (Gemini context 최적화)
const limited = relevant.slice(0, 10);

return [{ json: { regulations: limited } }];
```

---

#### Node 13: HTTP Request — generateResponse (Gemini)

규정 기반 응답 생성.

- **타입**: HTTP Request
- **Method**: POST
- **URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={{$env.GEMINI_API_KEY}}`
- **Body**:

```json
{
  "systemInstruction": {
    "parts": [{
      "text": "{{시스템 프롬프트 - prompts/system-prompt.md 참조}}"
    }]
  },
  "contents": [
    {
      "role": "user",
      "parts": [{
        "text": "[참조 규정]\n{{$('fetchRegulations').first().json.regulations}}\n\n[이전 대화]\n{{$('fetchHistory').first().json.conversationHistory}}\n\n[사용자 질문]\n{{$('parseMessage').first().json.messageText}}\n\n위 규정을 기반으로 답변해주세요. JSON 형식으로 응답:\n{\"answer\": \"답변 내용\", \"source\": \"근거 규정\", \"confidence\": 0.0~1.0}"
      }]
    }
  ],
  "generationConfig": {
    "temperature": 0.3,
    "maxOutputTokens": 1024
  }
}
```

---

#### Node 14: Function — formatResponse

Gemini 응답을 최종 포맷으로 변환. 신뢰도에 따른 분기 처리.

```javascript
const result = JSON.parse(
  $json.candidates[0].content.parts[0].text
);

let responseText;

if (result.confidence >= 0.7) {
  // 높은 신뢰도: 답변 제공
  responseText = `${result.answer}\n\n📌 근거: ${result.source}`;
} else if (result.confidence >= 0.4) {
  // 중간 신뢰도: 답변 + 담당자 안내 병기
  const contact = $('fetchContact').first()?.json;
  responseText = `${result.answer}\n\n📌 근거: ${result.source}\n\n` +
    `⚠️ 정확한 확인이 필요하시면 ${contact?.team || '경영지원본부'}에 문의해주세요.`;
} else {
  // 낮은 신뢰도: 에스컬레이션
  responseText = `해당 문의는 정확한 확인이 필요한 사항입니다.\n경영지원본부에 직접 문의해주시기 바랍니다.`;
}

return [{
  json: {
    responseText,
    tier: '2',
    category: $('classifyIntent').first().json.category,
    isEscalated: result.confidence < 0.4
  }
}];
```

---

#### Node 15: Google Sheets — saveHistory

대화 이력 저장 (사용자 메시지 + 봇 응답, 2행 추가).

- **타입**: Google Sheets (Append)
- **Sheet**: 대화이력
- **Rows**:

```javascript
const userId = $('parseMessage').first().json.userId;
const now = new Date().toISOString();
const sessionId = `${userId}_${now.split('T')[0].replace(/-/g, '')}`;

return [
  // 사용자 메시지
  {
    json: {
      sessionId,
      userId,
      timestamp: $('parseMessage').first().json.timestamp,
      role: 'user',
      message: $('parseMessage').first().json.messageText,
      category: $json.category,
      tier: $json.tier,
      isEscalated: false
    }
  },
  // 봇 응답
  {
    json: {
      sessionId,
      userId,
      timestamp: now,
      role: 'bot',
      message: $json.responseText,
      category: $json.category,
      tier: $json.tier,
      isEscalated: $json.isEscalated
    }
  }
];
```

---

#### Node 16: Execute Workflow — getToken

서브 워크플로우(NaverWorks Auth) 호출하여 Access Token 획득.

- **타입**: Execute Workflow
- **Workflow**: NaverWorks Auth

---

#### Node 17: HTTP Request — sendMessage

네이버웍스 봇 API로 응답 전송.

- **타입**: HTTP Request
- **Method**: POST
- **URL**: `https://www.worksapis.com/v1.0/bots/{{$env.NAVER_WORKS_BOT_ID}}/users/{{$('parseMessage').first().json.userId}}/messages`
- **Headers**:
  ```
  Authorization: Bearer {{$('getToken').first().json.access_token}}
  Content-Type: application/json
  ```
- **Body**:
  ```json
  {
    "content": {
      "type": "text",
      "text": "{{최종 responseText}}"
    }
  }
  ```

---

## 에러 핸들링

각 주요 노드에 Error Trigger를 연결한다.

| 실패 지점 | 대응 |
|-----------|------|
| Gemini API 호출 실패 | 사용자에게 "잠시 후 다시 시도해주세요" 응답 |
| Google Sheets 조회 실패 | 에러 로깅 + 사용자에게 안내 메시지 |
| 네이버웍스 응답 전송 실패 | n8n 에러 로그 기록 (재시도 1회) |
| FAQ/규정 데이터 없음 | 바로 에스컬레이션 처리 |

### n8n Error Workflow

별도 Error Workflow를 생성하여 에러 발생 시:
1. 에러 내용을 Google Sheets 에러 로그 시트에 기록
2. (선택) 관리자에게 네이버웍스 알림 전송

---

## 환경변수 (n8n Credentials)

n8n의 Credentials 또는 Environment Variables에 설정:

| 변수명 | 설명 |
|--------|------|
| NAVER_WORKS_BOT_ID | 네이버웍스 봇 ID |
| NAVER_WORKS_CLIENT_ID | Developer Console Client ID |
| NAVER_WORKS_CLIENT_SECRET | Developer Console Client Secret |
| NAVER_WORKS_SERVICE_ACCOUNT | Service Account ID |
| NAVER_WORKS_PRIVATE_KEY | RSA Private Key |
| GEMINI_API_KEY | Google Gemini API Key |
| GOOGLE_SHEETS_ID | BizSupportBot_Data 스프레드시트 ID |

---

<!-- pagebreak -->

# 개발 플랜

## 전체 일정 (약 3주)

```
Week 1: 인프라 + 데이터 준비
Week 2: 핵심 로직 구현
Week 3: 테스트 + 파일럿 배포
```

---

## Phase 1: 인프라 세팅 (Day 1-2)

### 목표
네이버웍스 봇 ↔ n8n 간 메시지 송수신 확인 (에코봇 수준)

### 작업 항목

| # | 작업 | 담당 | 산출물 |
|---|------|------|--------|
| 1-1 | 네이버웍스 Developer Console에서 Bot 생성 | 본인 | Bot ID, Callback URL 설정 |
| 1-2 | Service Account 생성 + Private Key 발급 | 본인 (관리자 권한 필요) | Client ID/Secret, SA, Key |
| 1-3 | n8n에 Webhook 워크플로우 생성 | 본인 | Webhook URL |
| 1-4 | 네이버웍스 Bot Callback URL에 n8n Webhook 등록 | 본인 | 봇-n8n 연결 완료 |
| 1-5 | 에코봇 테스트: 메시지 수신 → 그대로 응답 | 본인 | 스크린샷 |
| 1-6 | Gemini API Key 발급 (Google AI Studio) | 본인 | API Key |

### 완료 기준
- 네이버웍스에서 봇에 메시지 보내면 n8n에서 수신 확인
- n8n에서 네이버웍스로 응답 메시지 전송 확인
- Gemini API 단독 호출 테스트 성공

### 주의사항
- 네이버웍스 봇 생성에는 **관리자 권한**이 필요할 수 있음 → IT팀/관리자 협조
- n8n Webhook URL이 외부에서 접근 가능해야 함 (사내 네트워크 방화벽 확인)

---

## Phase 2: 데이터 구조화 (Day 3-5)

### 목표
Google Sheets에 규정/FAQ/담당자 데이터 입력 완료

### 작업 항목

| # | 작업 | 담당 | 산출물 |
|---|------|------|--------|
| 2-1 | Google Sheets 파일 생성 (BizSupportBot_Data) | 본인 | 스프레드시트 |
| 2-2 | 시트 구조 생성 (6개 시트, 컬럼 헤더) | 본인 | 02-data-structure.md 기준 |
| 2-3 | FAQ 초안 작성 — 인사 카테고리 | 경영지원(인사) | FAQ 20-30건 |
| 2-4 | FAQ 초안 작성 — 총무 카테고리 | 경영지원(총무) | FAQ 20-30건 |
| 2-5 | FAQ 초안 작성 — 회계 카테고리 | 경영지원(회계) | FAQ 20-30건 |
| 2-6 | 규정 데이터 입력 — 디지털화된 규정 옮기기 | 본인 + 경영지원 | 규정 시트 |
| 2-7 | 규정 데이터 입력 — 신규 디지털화 | 경영지원 | 규정 시트 |
| 2-8 | 담당자 정보 입력 | 경영지원 | 담당자 시트 |
| 2-9 | n8n ↔ Google Sheets 연동 테스트 | 본인 | 읽기/쓰기 확인 |

### 완료 기준
- 카테고리별 FAQ 최소 10건 이상 입력
- 주요 규정 데이터 입력 (완벽하지 않아도 됨, 추후 보강)
- n8n에서 Google Sheets 읽기/쓰기 정상 동작

### 핵심 포인트
- FAQ 작성이 **전체 프로젝트의 품질을 결정**함 → 경영지원팀과 협업 필수
- "실제 임직원이 묻는 질문"을 수집하는 것이 핵심 (이상적 질문 X)
- 기존 문의 이력(메일, 메신저 등)이 있다면 거기서 FAQ 추출

---

## Phase 3: 핵심 로직 구현 (Day 6-10)

### 목표
전체 워크플로우 완성 (FAQ 매칭 → Gemini 응답 → 에스컬레이션)

### 작업 항목

| # | 작업 | 산출물 |
|---|------|--------|
| 3-1 | 메인 워크플로우 기본 틀 구축 (Webhook → 파싱 → 응답) | n8n 워크플로우 |
| 3-2 | FAQ 매칭 로직 구현 (Node 4-7) | FAQ 즉시 응답 동작 |
| 3-3 | Gemini 의도 분류 구현 (Node 8) | 카테고리 분류 동작 |
| 3-4 | 규정 조회 + Gemini 응답 생성 구현 (Node 12-14) | 규정 기반 응답 동작 |
| 3-5 | 에스컬레이션 로직 구현 (Node 10-11) | 담당자 안내 동작 |
| 3-6 | 멀티턴 대화 이력 관리 구현 (Node 3, 15) | 맥락 유지 대화 동작 |
| 3-7 | 인증 서브 워크플로우 구현 | 토큰 발급 자동화 |
| 3-8 | 에러 핸들링 추가 | 에러 시 안내 메시지 |
| 3-9 | 시스템 프롬프트 초안 적용 | prompts/system-prompt.md |

### 구현 순서 (의존성 고려)

```
3-1 기본 틀 ──→ 3-7 인증 ──→ 3-2 FAQ 매칭 (먼저 완성 → 바로 테스트 가능)
                                  ↓
                             3-3 의도 분류
                                  ↓
                             3-4 응답 생성
                                  ↓
                             3-5 에스컬레이션
                                  ↓
                             3-6 멀티턴
                                  ↓
                             3-8 에러 핸들링
```

### 완료 기준
- FAQ 질문 → Tier 1 즉시 응답 정상
- 규정 기반 질문 → Tier 2 Gemini 응답 정상
- 답변 불가 질문 → Tier 3 에스컬레이션 안내 정상
- 후속 질문 시 이전 맥락 반영 확인
- 에러 시 사용자 안내 메시지 전송 확인

---

## Phase 4: 프롬프트 튜닝 + 테스트 (Day 11-13)

### 목표
응답 품질을 실무 수준으로 끌어올리기

### 작업 항목

| # | 작업 | 산출물 |
|---|------|--------|
| 4-1 | 테스트 시나리오 작성 (카테고리별 10개씩) | 테스트 케이스 문서 |
| 4-2 | Tier 1 테스트: FAQ 매칭 정확도 확인 | 매칭 결과 리포트 |
| 4-3 | Tier 2 테스트: Gemini 응답 품질 확인 | 응답 품질 리포트 |
| 4-4 | Tier 3 테스트: 에스컬레이션 적절성 확인 | 에스컬레이션 리포트 |
| 4-5 | 멀티턴 시나리오 테스트 | 멀티턴 결과 리포트 |
| 4-6 | 시스템 프롬프트 튜닝 (테스트 결과 기반) | 프롬프트 v2 |
| 4-7 | 엣지 케이스 처리 추가 | 업데이트된 워크플로우 |
| 4-8 | FAQ/규정 데이터 보강 (테스트 중 발견된 갭) | 업데이트된 시트 |

### 테스트 시나리오 예시

| 카테고리 | 시나리오 | 기대 결과 |
|---------|---------|----------|
| 인사 | "연차 며칠 남았어?" | Tier 1 FAQ 응답 |
| 인사 | "육아휴직 신청 절차가 어떻게 돼?" | Tier 2 규정 기반 응답 |
| 인사 | "연차 남았어?" → "그럼 신청은?" | 멀티턴 맥락 유지 |
| 회계 | "법인카드 해외 결제 가능해?" | Tier 2 규정 기반 응답 |
| 기타 | "오늘 점심 뭐야?" | Tier 3 에스컬레이션 또는 범위 외 안내 |
| 엣지 | 빈 메시지 / 이미지 전송 | 적절한 안내 메시지 |

### 완료 기준
- FAQ 매칭 정확도 90% 이상
- Gemini 응답에 할루시네이션 없음 (규정에 없는 내용 생성 X)
- 에스컬레이션 적절성 확인 (답변 가능한데 에스컬레이션 하는 경우 최소화)

---

## Phase 5: 파일럿 배포 (Day 14-15)

### 목표
소규모 그룹으로 실사용 테스트 후 안정화

### 작업 항목

| # | 작업 | 산출물 |
|---|------|--------|
| 5-1 | 파일럿 대상 선정 (경영지원 내부 5-10명) | 대상자 목록 |
| 5-2 | 파일럿 안내 (봇 추가 방법, 사용법) | 안내 메시지 |
| 5-3 | 1-2일 파일럿 운영 | 사용 로그 |
| 5-4 | 피드백 수집 + 분석 | 피드백 정리 |
| 5-5 | 피드백 반영 (FAQ 보강, 프롬프트 수정) | 업데이트 |
| 5-6 | 전사 배포 결정 | Go/No-Go 판단 |

### 완료 기준
- 파일럿 그룹 만족도 확인
- 치명적 버그 없음
- FAQ 커버리지 충분

---

## Phase 0: 베이스라인 측정 (Phase 1과 병행)

### 목표
봇 도입 전 현재 상태를 기록하여, 도입 후 효과를 수치로 증명할 수 있는 기준선 확보.

### 작업 항목

| # | 작업 | 담당 | 산출물 |
|---|------|------|--------|
| 0-1 | 경영지원팀에 현재 월간 문의 건수 확인 | 본인 → 경영지원 | 월간 문의 건수 (체감치 OK) |
| 0-2 | 카테고리별 문의 비율 확인 | 본인 → 경영지원 | 인사/총무/회계 비율 |
| 0-3 | 문의 1건당 평균 응대 시간 확인 | 본인 → 경영지원 | 건당 응대 시간 (분) |
| 0-4 | 응대 담당 인원 수 + 시급 환산 | 본인 | 인건비 기준 |
| 0-5 | 임직원 평균 답변 대기 시간 확인 | 본인 → 경영지원 | 대기 시간 |

> **이 데이터가 없으면 ROI를 증명할 수 없다.** Phase 1과 동시에 수집할 것.

---

## Phase 6: 대시보드 구축 + 성과 측정 체계 (Day 16-18)

### 목표
운영 로그 기반 대시보드 구축, 경영지원 본부 전체가 볼 수 있는 성과 모니터링 체계 완성.

### 작업 항목

| # | 작업 | 산출물 |
|---|------|--------|
| 6-1 | 운영로그 시트 생성 + n8n 로깅 노드 추가 | 운영로그 자동 기록 |
| 6-2 | 대시보드_일간 시트 구축 (수식 + 차트) | 일간 대시보드 |
| 6-3 | 대시보드_월간 시트 구축 (수식 + 차트) | 월간 대시보드 |
| 6-4 | 에스컬레이션 TOP 질문 자동 집계 | FAQ 보강 파이프라인 |
| 6-5 | ROI 자동 계산 수식 | 월간 ROI 요약 |
| 6-6 | 시트 권한 설정 (편집/뷰어 분리) | 권한 적용 완료 |

### 완료 기준
- 대시보드에서 일간/월간 자동 처리율, 에스컬레이션 비율 확인 가능
- ROI가 자동 계산되어 표시됨
- 경영지원 본부 전원이 대시보드 접근 가능

---

## 전체 마일스톤 요약

```
Phase 0   ████████░░░░░░░░░░  베이스라인 측정 (Phase 1과 병행)
Day 1-2   ████░░░░░░░░░░░░░░  Phase 1: 인프라 (에코봇)
Day 3-5   ░░░████░░░░░░░░░░░  Phase 2: 데이터 (규정/FAQ)
Day 6-10  ░░░░░░█████░░░░░░░  Phase 3: 핵심 로직 (워크플로우)
Day 11-13 ░░░░░░░░░░░███░░░░  Phase 4: 테스트 + 튜닝
Day 14-15 ░░░░░░░░░░░░░██░░░  Phase 5: 파일럿
Day 16-18 ░░░░░░░░░░░░░░░███  Phase 6: 대시보드 + 성과 측정
```

## 리스크 및 대응

| 리스크 | 영향 | 대응 |
|--------|------|------|
| 네이버웍스 봇 생성 권한 없음 | Phase 1 지연 | IT팀/관리자에게 사전 요청 |
| 규정 데이터 디지털화 지연 | Phase 2 지연 | 핵심 규정 우선 입력, 나머지 운영 중 보강 |
| n8n 외부 접근 불가 (방화벽) | Phase 1 차단 | IT팀에 포트 개방 요청 또는 n8n Cloud 검토 |
| Gemini 응답 품질 미달 | Phase 4 지연 | 프롬프트 반복 튜닝, 규정 데이터 품질 개선 |
| FAQ 커버리지 부족 | 사용자 불만 | 파일럿 피드백으로 지속 보강 |

## Phase 1 시작 전 사전 확인 체크리스트

- [ ] 네이버웍스 Developer Console 접근 권한 확인
- [ ] n8n 서버 접근 권한 + Webhook URL 외부 노출 가능 여부
- [ ] Google AI Studio에서 Gemini API Key 발급 가능 여부
- [ ] Google Sheets API 사용을 위한 Service Account 또는 OAuth 설정
- [ ] 경영지원팀 FAQ 작성 협조 확인

---

<!-- pagebreak -->

# 성과 측정 & 대시보드 설계

## 프로젝트 목적 (우선순위)

1. **경영지원 본부의 반복 문의 응대 공수 절감** (핵심)
2. 임직원 문의 응답 속도 개선
3. AI 활용 파일럿 사례 확보

## 보고 라인

```
전략기획(주도/개발) → 경영지원 본부장(승인) → 대표(최종 보고)
```

**대표/본부장이 보고 싶은 숫자**: 얼마나 효율화했는가, 비용 대비 효과

---

## 1. 베이스라인 측정 (봇 도입 전 — 반드시 선행)

봇 효과를 증명하려면 **도입 전 현재 상태**를 먼저 측정해야 한다.

### 수집할 베이스라인 데이터

| 항목 | 측정 방법 | 비고 |
|------|----------|------|
| 월간 총 문의 건수 | 경영지원팀에 확인 (메신저/메일/전화) | 최근 1-3개월 평균 |
| 카테고리별 비율 | 인사/총무/회계 비율 | 대략적이라도 OK |
| 문의 1건당 평균 응대 시간 | 경영지원 담당자 체감치 | 5분? 10분? 15분? |
| 응대 담당 인원 수 | 문의 응대에 관여하는 인원 | |
| 담당자 시급 (또는 월급 기준 환산) | 인건비 계산용 | 정확하지 않아도 됨 |
| 임직원 평균 답변 대기 시간 | 문의 후 답변받기까지 | 30분? 2시간? 반나절? |

### 베이스라인 수집 액션

> **경영지원 본부에 요청할 것:**
> "최근 1-3개월간 임직원 문의가 대략 월 몇 건 정도 오는지,
> 어떤 유형이 많은지, 1건 응대에 평균 몇 분 정도 걸리는지 알려주세요."

정확한 숫자가 아니어도 된다. 체감치로 충분하다.
이 숫자가 있어야 "봇 도입 후 XX% 감소"를 증명할 수 있다.

---

## 2. 핵심 KPI

### Primary KPI (대표/본부장 보고용)

| KPI | 정의 | 목표 | 측정 방법 |
|-----|------|------|----------|
| **문의 자동 처리율** | 봇이 에스컬레이션 없이 처리한 비율 | 60% 이상 | (Tier1 + Tier2) / 전체 건수 |
| **월간 절감 시간** | 봇이 대신 처리한 건수 × 건당 평균 응대 시간 | - | 자동 처리 건수 × 베이스라인 건당 시간 |
| **ROI** | 절감 비용 vs 운영 비용 | 양수 | 아래 ROI 모델 참조 |

### Secondary KPI (실무 모니터링용)

| KPI | 정의 | 목표 | 측정 방법 |
|-----|------|------|----------|
| 봇 응답 시간 | Webhook 수신 ~ 응답 전송 | Tier1: 2초 이내, Tier2: 5초 이내 | 로그 타임스탬프 차이 |
| 에스컬레이션 비율 | Tier 3 비율 | 40% 이하 → 점진적 감소 | Tier3 건수 / 전체 건수 |
| FAQ 커버리지 | FAQ로 처리한 비율 | 점진적 증가 | Tier1 건수 / 전체 건수 |
| 카테고리별 분포 | 인사/총무/회계 비율 | 모니터링 | 로그 집계 |
| 일별 사용량 | 일간 봇 이용 건수 | 모니터링 | 로그 집계 |
| 반복 에스컬레이션 | 같은 질문이 반복 에스컬레이션 | 0건 | FAQ 갭 분석 |

---

## 3. ROI 모델

### 절감 비용 (월간)

```
월간 절감 비용 = 봇 자동 처리 건수 × 건당 평균 응대 시간(분) × 담당자 분당 인건비

예시:
- 월간 총 문의: 200건
- 봇 자동 처리율: 60% → 120건
- 건당 평균 응대 시간: 10분
- 담당자 시급: 25,000원 (분당 약 417원)

월간 절감 시간 = 120건 × 10분 = 1,200분 = 20시간
월간 절감 비용 = 20시간 × 25,000원 = 500,000원
```

### 운영 비용 (월간)

| 항목 | 예상 비용 | 비고 |
|------|----------|------|
| Gemini API | ~10,000-50,000원 | Flash 기준 100만 토큰당 약 $0.075 |
| n8n | 0원 | 셀프호스팅 (기존 인프라) |
| Google Sheets | 0원 | Workspace 기존 라이선스 |
| 관리 인건비 | 월 2-4시간 | FAQ/규정 업데이트 |

### ROI 공식

```
월간 ROI = 월간 절감 비용 - 월간 운영 비용

초기 개발 투자 회수 기간 = 개발 인건비 / 월간 ROI
```

> **대표 보고 시 포인트**: ROI 숫자 자체보다 "반복 문의 자동화로 경영지원 인력이
> 더 부가가치 높은 업무에 집중할 수 있다"는 정성적 가치를 함께 어필

---

## 4. 로그 설계

### 봇 운영 로그 (Google Sheets — 신규 시트)

기존 `대화이력` 시트와 별도로, 성과 측정 전용 로그 시트를 둔다.

**시트: 운영로그**

| 컬럼 | 타입 | 설명 |
|------|------|------|
| logId | string | 고유 ID (자동 생성) |
| timestamp | datetime | 요청 시각 |
| userId | string | 사용자 ID |
| messageText | string | 사용자 메시지 원문 |
| category | string | 분류된 카테고리 |
| subCategory | string | 소분류 |
| tier | string | 응답 Tier (1/2/3) |
| responseTime | number | 응답 소요 시간 (ms) |
| isEscalated | boolean | 에스컬레이션 여부 |
| confidence | number | Gemini 신뢰도 점수 |
| faqId | string | 매칭된 FAQ ID (Tier1만) |
| tokenUsage | number | Gemini API 토큰 사용량 |
| errorOccurred | boolean | 에러 발생 여부 |

### 로그를 남기는 이유 (항목별)

| 로그 항목 | 왜 필요한가 |
|----------|------------|
| tier | 자동 처리율 계산, Tier별 비율 추이 |
| responseTime | 응답 속도 모니터링, SLA 관리 |
| isEscalated | 에스컬레이션 비율, 절감 시간 계산 |
| category | 카테고리별 문의 분포, 규정 보강 우선순위 |
| confidence | 답변 품질 모니터링, 프롬프트 튜닝 근거 |
| faqId | FAQ 활용도 분석, 미사용 FAQ 정리 |
| tokenUsage | API 비용 추적, 월간 운영비 산출 |
| messageText | 에스컬레이션된 질문 → FAQ 후보 추출 |

### n8n 구현

메인 워크플로우의 응답 전송 직전에 `운영로그` 시트에 1행 Append.
기존 `대화이력` 시트는 멀티턴 맥락용으로 유지 (역할 분리).

---

## 5. 대시보드 설계

### 도구: Google Sheets 차트 + (선택) Looker Studio

Google Sheets 자체 차트로 시작 → 필요 시 Looker Studio로 확장.
경영지원본부 전원이 접근 가능한 공유 시트로 운영.

### 대시보드 구성

**시트: 대시보드** (수식/차트로 운영로그를 자동 집계)

```
┌─────────────────────────────────────────────────────┐
│                    BizSupportBot 대시보드              │
├─────────────────────────────────────────────────────┤
│                                                      │
│  [오늘의 요약]                                        │
│  총 문의: 23건  자동처리: 15건(65%)  에스컬레이션: 8건   │
│  평균 응답시간: 2.3초   예상 절감시간: 150분            │
│                                                      │
├──────────────────────┬──────────────────────────────┤
│                      │                               │
│  [카테고리별 문의 비율] │  [Tier별 처리 비율]            │
│  원형 차트             │  막대 차트                     │
│                      │                               │
│  인사 45%             │  ██████████ Tier1 FAQ   50%   │
│  회계 35%             │  ██████     Tier2 규정  25%   │
│  총무 15%             │  █████      Tier3 에스컬 25%  │
│  기타  5%             │                               │
│                      │                               │
├──────────────────────┴──────────────────────────────┤
│                                                      │
│  [주간/월간 추이]                                      │
│  꺾은선 그래프: 일별 문의 건수 + 자동처리율 추이          │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  [에스컬레이션 TOP 질문]                               │
│  봇이 처리 못한 질문 중 빈도 높은 것 → FAQ 추가 후보     │
│                                                      │
│  1. "육아휴직 중 급여는 어떻게 되나요?" (7건)            │
│  2. "해외 출장 보험 가입 절차?" (5건)                   │
│  3. "법인카드 분실 시 처리 방법?" (4건)                  │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  [월간 ROI 요약]                                      │
│  절감 시간: 20시간  절감 비용: 500,000원                │
│  운영 비용: 30,000원  순 ROI: +470,000원               │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 뷰별 관심 포인트

| 대상 | 주로 보는 것 | 의사결정 |
|------|------------|---------|
| **실무자** | 에스컬레이션 TOP 질문, 카테고리별 분포 | FAQ 추가, 규정 데이터 보강 |
| **리더** | 자동 처리율 추이, 월간 ROI | 봇 확대/축소 판단, 리소스 투입 결정 |
| **대표/본부장** | 월간 ROI, 절감 시간 | Go/No-Go, 다른 부서 확대 여부 |

### Google Sheets 대시보드 구현 방법

```
[BizSupportBot_Data] 스프레드시트

시트 구성:
├── FAQ              (데이터)
├── 규정_인사         (데이터)
├── 규정_총무         (데이터)
├── 규정_회계         (데이터)
├── 담당자            (데이터)
├── 대화이력          (데이터 - 멀티턴용)
├── 운영로그          (데이터 - 성과측정용)  ← 신규
├── 대시보드_일간      (차트)                ← 신규
└── 대시보드_월간      (차트)                ← 신규
```

**집계 수식 예시 (대시보드_일간 시트):**

| 지표 | 수식 |
|------|------|
| 오늘 총 문의 | `=COUNTIFS(운영로그!B:B, ">="&TODAY(), 운영로그!B:B, "<"&TODAY()+1)` |
| 오늘 자동처리 | `=COUNTIFS(운영로그!B:B, ">="&TODAY(), 운영로그!G:G, "<>3")` |
| 자동처리율 | `=오늘자동처리/오늘총문의` |
| 평균 응답시간 | `=AVERAGEIFS(운영로그!H:H, 운영로그!B:B, ">="&TODAY())` |
| 에스컬레이션 건수 | `=COUNTIFS(운영로그!B:B, ">="&TODAY(), 운영로그!G:G, "3")` |
| 카테고리별 건수 | `=COUNTIFS(운영로그!B:B, ">="&TODAY(), 운영로그!E:E, "인사")` |

---

## 6. 피드백 루프 (FAQ 자동 보강 사이클)

```
에스컬레이션 발생
    ↓
운영로그에 기록 (질문 원문 + 카테고리)
    ↓
대시보드 "에스컬레이션 TOP 질문" 자동 집계
    ↓
경영지원 실무자가 주 1회 확인
    ↓
반복 질문 → FAQ 시트에 추가 (답변 작성)
    ↓
다음 동일 질문 → Tier 1 즉시 응답 (자동 처리율 상승)
```

이 사이클이 돌면 **시간이 갈수록 자동 처리율이 올라간다** → 성과가 자연스럽게 개선.

---

## 7. 대표 보고 시 활용 프레임

### 1페이지 요약 (경영회의용)

```
[BizSupportBot 파일럿 결과 요약]

■ Before
- 월간 경영지원 문의: ~200건
- 건당 평균 응대 시간: 10분
- 월간 응대 총 소요: ~33시간

■ After (1개월 파일럿)
- 봇 자동 처리율: 65%
- 월간 절감 시간: ~20시간
- 월간 운영 비용: ~30,000원

■ ROI
- 월간 절감 비용: ~500,000원
- 순 효과: +470,000원/월
- 부가 효과: 응답 즉시화 (평균 2초), 24시간 응대 가능

■ Next Step
- 전사 배포 / 다른 부서 확대 검토
```

---

<!-- pagebreak -->

# 운영 거버넌스 & 비개발자 관리 구조

## 핵심 원칙

- **비개발자가 직접 관리할 수 있어야 한다**: Google Sheets 편집만으로 봇 동작 변경
- **규정 변경 시 즉시 반영**: Google Sheets 수정 → 다음 문의부터 반영 (배포 불필요)
- **코드 수정 없이 운영 가능한 구조**: FAQ/규정/담당자 데이터는 모두 시트 기반

---

## 1. 비개발자 관리 가능 영역

### 코드 수정 없이 할 수 있는 것 (Google Sheets만 편집)

| 작업 | 시트 | 방법 | 반영 시점 |
|------|------|------|----------|
| FAQ 추가/수정/삭제 | FAQ | 행 추가/편집/isActive를 FALSE | 즉시 |
| 규정 데이터 추가/수정 | 규정_인사/총무/회계 | 행 추가/편집 | 즉시 |
| 담당자 정보 변경 | 담당자 | 행 편집 | 즉시 |
| FAQ 비활성화 (일시 중지) | FAQ | isActive를 FALSE로 변경 | 즉시 |
| 새 소분류 추가 | 담당자 | 행 추가 (새 subCategory) | 즉시 |

### 개발자가 해야 하는 것 (n8n 워크플로우 수정)

| 작업 | 빈도 |
|------|------|
| 새 대분류 카테고리 추가 (예: 법무) | 매우 드묾 |
| 프롬프트 튜닝 | 초기에만, 안정화 후 드묾 |
| n8n 노드 로직 변경 | 기능 확장 시에만 |
| 에러 대응 | 장애 발생 시 |

---

## 2. FAQ 관리 프로세스

### FAQ 추가 플로우

```
[에스컬레이션 발생]
    ↓
[대시보드에서 TOP 에스컬레이션 질문 확인] — 주 1회
    ↓
[경영지원 담당자가 답변 작성]
    ↓
[FAQ 시트에 새 행 추가]
    ├─ faqId: 규칙에 맞게 부여 (FAQ-HR-004 등)
    ├─ keywords: 사용자가 실제 쓰는 단어로 등록
    ├─ answer: 구체적이고 실행 가능한 답변
    ├─ source: 근거 규정
    └─ isActive: TRUE
    ↓
[다음 동일 질문 → Tier 1 즉시 응답]
```

### FAQ 작성 가이드 (비개발자용)

**keywords 작성법:**
- 사용자가 실제로 입력할 단어를 쉼표로 나열
- 구어체 포함: "연차", "남은연차", "연차 몇일", "휴가 남은거"
- 많을수록 좋음 (오탐보다 미탐이 더 문제)

**answer 작성법:**
- "~에서 ~하시면 됩니다" 형태로 구체적으로
- 경로가 있으면 경로 포함: "네이버웍스 > 근태관리 > 휴가신청"
- 기한/금액/조건 등 수치 반드시 포함
- 200자 이내 권장

**하지 말아야 할 것:**
- 컬럼 순서 변경 (n8n이 컬럼명으로 읽으므로)
- 컬럼 헤더 이름 변경
- 빈 행 삽입 (중간에 빈 행이 있으면 읽기 중단될 수 있음)

---

## 3. 규정 데이터 관리 프로세스

### 규정 변경 시

```
[사내 규정 변경 공지]
    ↓
[경영지원 담당자가 해당 규정 시트 수정]
    ├─ 기존 행 수정 (내용/시행일 업데이트)
    ├─ 또는 새 행 추가 (새 조항)
    └─ lastUpdated 날짜 갱신
    ↓
[다음 문의부터 변경된 규정으로 답변]
```

### 규정 데이터 작성 가이드 (비개발자용)

**content 작성법:**
- 한 행 = 한 조항 (너무 긴 조항은 항목별로 분리)
- 핵심 수치 반드시 포함 (금액, 기간, 기한)
- 500자 이내 권장 (너무 길면 Gemini 성능 저하)

**tags 작성법:**
- 이 조항과 관련된 키워드를 쉼표로 나열
- 사용자가 물어볼 만한 단어 위주: "연차,부여,일수,15일"

**규정 삭제 시:**
- 행을 삭제하지 말고 내용에 "(폐지)" 또는 "(개정: REG-HR-XXX으로 대체)" 표기
- 이력 추적을 위해 행 자체는 유지 권장

---

## 4. 담당자 정보 관리

### 변경 시나리오

| 상황 | 조치 |
|------|------|
| 담당자 변경 | 해당 행의 name, contact, email 수정 |
| 새 업무 영역 추가 | 새 행 추가 (category, subCategory 기입) |
| 담당자 부재 (휴가 등) | note에 "2/19-2/23 부재, OOO 대리 문의" 기입 |

---

## 5. 운영 루틴

### 주간 루틴 (경영지원 실무자, 15-30분)

| 항목 | 작업 |
|------|------|
| 에스컬레이션 TOP 질문 확인 | 대시보드에서 확인 → FAQ 추가 검토 |
| 답변 품질 스팟체크 | 대화이력에서 무작위 5-10건 확인 |
| FAQ 보강 | 에스컬레이션 빈도 높은 질문을 FAQ로 등록 |

### 월간 루틴 (운영 담당자, 1-2시간)

| 항목 | 작업 |
|------|------|
| 월간 성과 리포트 확인 | 대시보드_월간 시트 |
| 규정 업데이트 반영 확인 | 해당 월 규정 변경 사항 누락 없는지 |
| 대화이력 아카이빙 | 30일 이상 된 대화이력 → 아카이브 시트 이동 |
| 운영로그 아카이빙 | 90일 이상 된 로그 → 아카이브 시트 이동 |
| API 비용 확인 | Google AI Studio 대시보드 |

### 분기 루틴 (리더, 30분)

| 항목 | 작업 |
|------|------|
| ROI 리뷰 | 절감 비용 vs 운영 비용 |
| 확대 검토 | 다른 부서/업무 영역 확장 여부 |
| 사용자 피드백 종합 | 정성적 피드백 수집 및 반영 |

---

## 6. 권한 설계

### Google Sheets 공유 권한

| 대상 | 권한 | 범위 |
|------|------|------|
| 봇 서비스 계정 (n8n) | 편집자 | 전체 시트 (읽기/쓰기) |
| 경영지원 실무자 | 편집자 | FAQ, 규정, 담당자 시트 |
| 경영지원 리더 | 뷰어 | 대시보드 시트 |
| 전략기획 (운영 담당) | 편집자 | 전체 시트 |

### 시트 보호 (실수 방지)

| 시트 | 보호 설정 |
|------|----------|
| 운영로그 | 봇 서비스 계정 + 전략기획만 편집 가능 (실무자 수동 편집 방지) |
| 대화이력 | 봇 서비스 계정만 편집 가능 |
| 대시보드 | 전략기획만 편집 가능 (수식 보호) |
| FAQ/규정/담당자 | 경영지원 실무자 + 전략기획 편집 가능 |
| 모든 시트 헤더 행 | 편집 잠금 (컬럼 헤더 변경 방지) |

---

## 7. 장애 대응

### 봇이 응답하지 않을 때

```
1. n8n 서버 상태 확인 (n8n 대시보드 접속)
2. 워크플로우 활성화 여부 확인 (비활성화 되었을 수 있음)
3. 네이버웍스 Developer Console → Bot 상태 확인
4. n8n Execution 로그에서 에러 확인
5. 해결 안 되면 → 전략기획(개발 담당)에게 연락
```

### 봇이 틀린 답변을 할 때

```
1. 대화이력에서 해당 대화 확인
2. FAQ에 해당 질문이 있는지 확인
   - 있으면 → FAQ 답변 수정
   - 없으면 → 규정 데이터 확인 후 FAQ 추가
3. 규정 데이터가 오래된 경우 → 규정 시트 업데이트
```

---

<!-- pagebreak -->

# Gemini 시스템 프롬프트

## 용도

n8n 워크플로우의 Node 13 (generateResponse)에서 Gemini API `systemInstruction`에 사용.

---

## 시스템 프롬프트 (v1)

```
당신은 [회사명] 경영지원본부의 AI 문의 응대 도우미입니다.
임직원의 인사, 총무, 회계 관련 문의에 사내 규정을 기반으로 정확하게 답변합니다.

## 핵심 규칙

1. **규정 기반 답변만 제공**: 아래 [참조 규정]에 있는 내용만으로 답변하세요. 규정에 없는 내용은 절대 추측하거나 만들어내지 마세요.
2. **출처 명시**: 답변 끝에 반드시 근거 규정을 표시하세요. (예: "근거: 연차휴가 규정 제5조")
3. **모르면 솔직하게**: 규정에서 해당 내용을 찾을 수 없으면 "해당 내용은 현재 등록된 규정에서 확인되지 않습니다"라고 답변하세요.
4. **친절하고 간결하게**: 존댓말을 사용하되, 불필요한 수식어 없이 핵심만 전달하세요.
5. **구체적 수치 포함**: 금액, 기간, 기한 등 수치 정보가 있으면 반드시 포함하세요.

## 답변 형식

- 핵심 답변을 먼저, 부가 설명은 그 다음에
- 절차가 있으면 단계별로 번호를 매겨 안내
- 답변은 200자 이내로 간결하게 (복잡한 절차는 예외)

## 이전 대화 처리

[이전 대화]가 제공되면 맥락을 고려하여 답변하세요.
- "그건?", "그럼?", "신청은?" 같은 후속 질문은 이전 대화의 주제를 이어받습니다.
- 이전 대화와 관련 없는 새 질문이면 독립적으로 답변하세요.

## 신뢰도 판단

답변의 확신 수준을 confidence (0.0~1.0)로 평가하세요:
- 1.0: 규정에 정확히 명시된 내용
- 0.7~0.9: 규정에 근거가 있으나 해석이 필요한 경우
- 0.4~0.6: 관련 규정은 있으나 직접적 답변이 어려운 경우
- 0.0~0.3: 규정에서 관련 내용을 찾기 어려운 경우

## 응답 JSON 형식

반드시 아래 JSON 형식으로만 응답하세요:
{
  "answer": "답변 내용",
  "source": "근거 규정 (예: 연차휴가 규정 제5조)",
  "confidence": 0.85
}
```

---

## 의도 분류 프롬프트 (v1)

n8n 워크플로우의 Node 8 (classifyIntent)에서 사용.

```
다음 사용자 메시지를 분류해주세요.

[사용자 메시지]
{userMessage}

[이전 대화]
{conversationHistory}

아래 JSON 형식으로만 응답하세요:
{
  "category": "인사|총무|회계|기타",
  "subCategory": "소분류명",
  "isAnswerable": true 또는 false,
  "confidence": 0.0~1.0
}

분류 기준:
- **인사**: 휴가(연차/반차/병가/육아휴직), 근태, 급여, 인사평가, 경조사, 복리후생, 채용, 퇴직
- **총무**: 비품, 시설(회의실/주차), 사무용품, 택배/우편, 사무실 관리, 보안, 차량
- **회계**: 법인카드, 출장비/경비, 세금계산서, 예산, 정산, 결의서, 전표
- **기타**: 위 카테고리에 해당하지 않는 일반 대화, 잡담, 다른 부서 업무

- isAnswerable: 경영지원 규정으로 답변 가능하면 true, 불가능하면 false
- "기타" 카테고리는 항상 isAnswerable: false
```

---

## 프롬프트 튜닝 가이드

### 튜닝 시점
- Phase 4 테스트에서 오답/부적절한 응답이 나올 때
- 에스컬레이션이 너무 많거나 적을 때
- 사용자 피드백에서 개선점이 발견될 때

### 튜닝 포인트

| 문제 | 조정 방향 |
|------|----------|
| 할루시네이션 발생 | "규정에 없으면 답변하지 마세요" 강화, temperature 낮추기 (0.1) |
| 답변이 너무 짧음 | "절차가 있으면 단계별로 안내" 강조 |
| 답변이 너무 김 | "200자 이내" 제한 강화 |
| 에스컬레이션 과다 | confidence 임계값 조정 (0.7 → 0.6) |
| 에스컬레이션 과소 | confidence 임계값 조정 (0.7 → 0.8) |
| 멀티턴 맥락 무시 | [이전 대화] 전달 방식 확인, 프롬프트에 맥락 참조 지시 강화 |
| 카테고리 오분류 | 의도 분류 프롬프트의 분류 기준에 예시 추가 |

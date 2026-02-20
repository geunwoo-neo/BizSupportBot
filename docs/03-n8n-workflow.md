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
[1. Webhook] ─→ [2. parseMessage] ─→ [2-1. sendProcessingAck]
                                          ↓
                                     [3. fetchHistory]
                                          ↓
                                   [4. fetchFaqIndex]
                                          ↓
                                 [5. selectFaqCandidates]
                                          ↓
                                [6. fetchFaqCandidates]
                                          ↓
                                    [7. matchFAQ]
                                          ↓
                                    [8. IF: faqMatched?]
                                     ├─ YES → [9. formatFAQResponse] ───────────────┐
                                     └─ NO  → [10. classifyIntent (Gemini)]         │
                                                     ↓                                │
                                        [10-1. IF: needsClarification?]              │
                                         ├─ YES → [10-2. formatClarification] ───────┤
                                         └─ NO  → [11. IF: isAnswerable?]             │
                                                       ├─ NO → [12. fetchContact]     │
                                                       │            ↓                  │
                                                       │       [13. formatEscalation]──┤
                                                       └─ YES ↓                       │
                                                      [14. fetchRegulations]           │
                                                            ↓                          │
                                                      [15. generateResponse (Gemini)]  │
                                                            ↓                          │
                                                      [16. formatResponse] ────────────┤
                                                                                        ↓
                                                                           [17. saveHistory]
                                                                                  ↓
                                                                           [18. getToken (Sub)]
                                                                                  ↓
                                                                           [19. sendMessage]
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
    sessionTimeoutMin: 20,
    maxTurnsInContext: 6,
    skip: false
  }
}];
```

---

#### Node 2-1: HTTP Request — sendProcessingAck

사용자가 "응답이 멈췄다"고 느끼지 않도록 선제 안내 메시지를 즉시 전송.

- **타입**: HTTP Request
- **Method**: POST
- **URL**: `https://www.worksapis.com/v1.0/bots/{{$env.NAVER_WORKS_BOT_ID}}/users/{{$('parseMessage').first().json.userId}}/messages`
- **Body**:

```json
{
  "content": {
    "type": "text",
    "text": "문의 내용을 확인하고 있습니다. 잠시만 기다려주세요."
  }
}
```

> 권장: 최종 응답이 2초 이상 걸릴 수 있는 흐름(Tier2/Tier3)에서는 항상 활성화.

---

#### Node 3: Google Sheets — fetchHistory

동일 사용자의 최근 이력을 읽고, **활성 세션 1개**만 선택해 멀티턴 문맥으로 사용.

- **타입**: Google Sheets (Read)
- **Spreadsheet**: BizSupportBot_Data
- **Sheet**: 대화이력
- **Operation**: Read Rows
- **Filter**: `userId` = `{{$json.userId}}`

**후처리 (Function 노드 연결):**
```javascript
const parsed = $('parseMessage').first().json;
const userId = parsed.userId;
const now = new Date(parsed.timestamp || new Date().toISOString());
const timeoutMin = Number(parsed.sessionTimeoutMin || 20);
const timeoutAgo = new Date(now.getTime() - timeoutMin * 60 * 1000);
const maxTurns = Number(parsed.maxTurnsInContext || 6);

const rows = $input.all()
  .map(item => item.json)
  .filter(row => row.userId === userId)
  .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));

const latest = rows[0];
const activeSessionId = (latest && new Date(latest.timestamp) >= timeoutAgo)
  ? latest.sessionId
  : `S_${userId}_${now.toISOString().replace(/[-:]/g, '').slice(0, 12)}`;

const conversationHistory = rows
  .filter(row => row.sessionId === activeSessionId)
  .slice(0, maxTurns)
  .reverse();

return [{
  json: {
    sessionId: activeSessionId,
    isNewSession: conversationHistory.length === 0,
    conversationHistory
  }
}];
```

---

#### Node 4: Google Sheets — fetchFaqIndex

FAQ 인덱스(키워드 → faqId 목록) 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: FAQ_키워드인덱스
- **Filter**: 없음 (초기에는 전체 조회 후 staticData 캐시, 이후 5분 TTL 갱신)

> **운영 기준**
> - FAQ 150행 미만: 기존 `fetchFAQ` 전량 조회 허용
> - FAQ 150행 이상: 인덱스 기반 후보 추출을 기본값으로 전환

---

#### Node 5: Function — selectFaqCandidates

사용자 메시지에서 후보 FAQ ID 집합을 추출.

- **타입**: Code (JavaScript)

```javascript
const userMessageRaw = $('parseMessage').first().json.messageText || '';
const userMessage = userMessageRaw.toLowerCase().replace(/\s+/g, '');
const indexRows = $('fetchFaqIndex').all().map(item => item.json);

const candidateScore = new Map();

for (const row of indexRows) {
  const keyword = (row.keyword || '').toLowerCase().replace(/\s+/g, '');
  const synonyms = (row.synonyms || '')
    .split(',')
    .map(v => v.trim().toLowerCase().replace(/\s+/g, ''))
    .filter(Boolean);

  const matched = [keyword, ...synonyms].some(token => token && userMessage.includes(token));
  if (!matched) continue;

  const weight = Number(row.weight || 1);
  const faqIds = (row.faqIds || '').split(',').map(v => v.trim()).filter(Boolean);

  for (const faqId of faqIds) {
    candidateScore.set(faqId, (candidateScore.get(faqId) || 0) + weight);
  }
}

const topCandidateIds = [...candidateScore.entries()]
  .sort((a, b) => b[1] - a[1])
  .slice(0, 10)
  .map(([faqId]) => faqId);

return [{ json: { topCandidateIds, candidateScore: Object.fromEntries(candidateScore) } }];
```

---

#### Node 6: Google Sheets — fetchFaqCandidates

후보 FAQ ID에 해당하는 FAQ 본문만 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: FAQ
- **Filter**: `faqId in topCandidateIds` + `isActive = TRUE`

---

#### Node 7: Function — matchFAQ

후보 FAQ만 스코어링해 최종 매칭.

```javascript
const userMessage = $('parseMessage').first().json.messageText || '';
const candidateFaqs = $('fetchFaqCandidates').all().map(item => item.json);
const baseScore = $('selectFaqCandidates').first().json.candidateScore || {};

let bestMatch = null;
let bestScore = 0;

for (const faq of candidateFaqs) {
  const keywords = (faq.keywords || '').split(',').map(k => k.trim()).filter(Boolean);
  const matchedCount = keywords.filter(kw => userMessage.includes(kw)).length;
  const score = (baseScore[faq.faqId] || 0) + matchedCount * 2;

  if (score > bestScore) {
    bestScore = score;
    bestMatch = faq;
  }
}

if (bestMatch && bestScore >= 2) {
  return [{ json: { faqMatched: true, matchedFAQ: bestMatch, matchedScore: bestScore } }];
}

return [{ json: { faqMatched: false } }];
```

---

#### Node 8: IF — faqMatched?

- **타입**: IF
- **Condition**: `{{$json.faqMatched}}` equals `true`
- **True**: → Node 9 (formatFAQResponse)
- **False**: → Node 10 (classifyIntent)

---

#### Node 9: Function — formatFAQResponse

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

#### Node 10: HTTP Request — classifyIntent (Gemini)

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
        "text": "다음 사용자 메시지를 분류해주세요.\n\n[사용자 메시지]\n{{$('parseMessage').first().json.messageText}}\n\n[현재 세션 대화]\n{{$('fetchHistory').first().json.conversationHistory}}\n\n아래 JSON 형식으로만 응답하세요:\n{\"category\": \"인사|총무|회계|기타\", \"subCategory\": \"소분류명\", \"isAnswerable\": true|false, \"confidence\": 0.0~1.0}\n\n- category: 인사, 총무, 회계 중 하나. 해당 없으면 기타\n- isAnswerable: 경영지원 규정으로 답변 가능한 질문이면 true\n- confidence: 분류 확신도 (0.0~1.0)"
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
    confidence: result.confidence,
    needsClarification: Boolean(result.needsClarification),
    clarificationQuestion: result.clarificationQuestion || ''
  }
}];
```

---

#### Node 10-1: IF — needsClarification?

- **타입**: IF
- **Condition**: `{{$json.needsClarification}}` equals `true`
- **True**: → Node 10-2 (formatClarification)
- **False**: → Node 11 (isAnswerable?)

---

#### Node 10-2: Function — formatClarification

모호 질문에는 정답 추정 대신 범위를 좁히는 질문을 1회 반환.

```javascript
const q = $json.clarificationQuestion || '문의하신 휴가 유형(연차/경조휴가/병가) 중 어떤 항목인지 알려주세요.';

return [{
  json: {
    responseText: `정확한 안내를 위해 확인이 필요합니다.
${q}`,
    tier: 'clarify',
    category: $json.category || '기타',
    isEscalated: false
  }
}];
```

---

#### Node 11: IF — isAnswerable?

- **타입**: IF
- **Condition**: `{{$json.isAnswerable}}` equals `true`
- **True**: → Node 12 (fetchRegulations)
- **False**: → Node 12 (fetchContact)

---

#### Node 12: Google Sheets — fetchContact

에스컬레이션용 담당자 조회.

- **타입**: Google Sheets (Read)
- **Sheet**: 담당자
- **Filter**: `category` = `{{$json.category}}`

---

#### Node 13: Function — formatEscalation

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

#### Node 14: Google Sheets — fetchRegulations

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

#### Node 15: HTTP Request — generateResponse (Gemini)

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
        "text": "[참조 규정]\n{{$('fetchRegulations').first().json.regulations}}\n\n[현재 세션 대화]\n{{$('fetchHistory').first().json.conversationHistory}}\n\n[사용자 질문]\n{{$('parseMessage').first().json.messageText}}\n\n위 규정을 기반으로 답변해주세요. JSON 형식으로 응답:\n{\"answer\": \"답변 내용\", \"source\": \"근거 규정\", \"confidence\": 0.0~1.0}"
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

#### Node 16: Function — formatResponse

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

#### Node 17: Google Sheets — saveHistory

대화 이력 저장 (사용자 메시지 + 봇 응답, 2행 추가).

- **타입**: Google Sheets (Append)
- **Sheet**: 대화이력
- **Rows**:

```javascript
const userId = $('parseMessage').first().json.userId;
const now = new Date().toISOString();
const sessionId = $('fetchHistory').first().json.sessionId;

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

#### Node 18: Execute Workflow — getToken

서브 워크플로우(NaverWorks Auth) 호출하여 Access Token 획득.

- **타입**: Execute Workflow
- **Workflow**: NaverWorks Auth

---

#### Node 19: HTTP Request — sendMessage

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

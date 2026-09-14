---
doc_id: INS-API-001
type: API
title: API — 보험청구심사 어시스턴트 REST
status: draft
upstream: [INS-UI-001]
---

# API 명세 REST: 보험청구심사 어시스턴트

## 1. 규칙

- 접두는 `/api/v1`. 경로는 복수형 명사
- 인증은 **JWT 쿠키**(`httponly`), 없으면 `Authorization: Bearer`
- **상담 경로는 인증을 요구하지 않는다** — 데모 범위의 결정이다. 근거: [[INS-INFRA-001#C4]]
- 시각은 ISO8601 UTC
- 스트리밍은 SSE. 이벤트는 `delta` · `final` · `error`

## 2. 에러

| 상태 | 코드 | 언제 |
|---|---|---|
| 400 | `BAD_REQUEST` | 입력 형식 |
| 401 | `UNAUTHORIZED` | 인증이 필요한 곳에 토큰 없음 |
| 404 | `SESSION_NOT_FOUND` | 세션이 없거나 TTL 만료 |
| 404 | `NOT_FOUND` | **관리자 경로는 권한이 없어도 404** — 존재를 숨긴다 |
| 409 | `CONFLICT` | 이미 가입된 메일 |
| 503 | `LLM_UNAVAILABLE` | 모델·검색 장애. 근거: [[INS-PRD-001#R21]] |

**세션 만료는 오류가 아니라 흐름이다** — 화면이 조용히 새 세션을 만들고 입력을 보존한다.

## 3. 엔드포인트

#### POST/api/v1/sessions 상담 세션 생성

근거: [[INS-UC-001#UC-H1]] · [[INS-UI-001#UI-4]]

```yaml
post:
  body: { initial_message?: str }
  responses:
    201: { session_id, created_at, ttl_seconds, first_response? }
```

`initial_message`가 있으면 첫 턴까지 처리해 돌려준다.

#### POST/api/v1/sessions/{sessionId}/messages/stream 대화 한 턴 (스트리밍)

근거: [[INS-UC-001#UC-H4]] · [[INS-UI-001#UI-6]]

```yaml
post:
  body: { text: str }
  responses:
    200: text/event-stream — delta* → final(SessionResponse) | error
    404: SESSION_NOT_FOUND
```

**화면이 쓰는 유일한 대화 경로다.** 응답의 `assistant`는 네 가지 중 하나 — 되묻기·판정·일반 답변·비교.

#### POST/api/v1/sessions/{sessionId}/slots 구조화 슬롯 주입

근거: [[INS-UC-001#UC-H3]] · [[INS-UC-001#UC-H8]]

```yaml
post:
  body: { insurer_id?, area?, diagnosis?, policies?[], ... }
  responses:
    200: { slots, missing[] }
```

**LLM을 거치지 않는 결정론 병합.** 가입 보험·진료내역처럼 이미 구조화된 값을 넣을 때 쓴다.

#### POST/api/v1/sessions/{sessionId}/documents 서류 업로드

근거: [[INS-UC-001#UC-H7]] · [[INS-PRD-001#R11]]

```yaml
post:
  body: multipart/form-data (이미지)
  responses:
    200: { attachment, doc_type, doc_type_confidence, extracted_slots, ocr_confidence, low_confidence }
```

`low_confidence`면 화면이 재촬영을 권한다.

#### GET/api/v1/sessions/{sessionId} 세션 상태

근거: [[INS-UC-001#UC-H4]]

```yaml
get:
  responses:
    200: { slots, history[], notes[] }   # 대화 전문이 담긴다
    404: SESSION_NOT_FOUND
```

화면이 새로고침 뒤 복원할 때 쓴다.

#### GET/api/v1/sessions/{sessionId}/summary 청구 준비 요약

근거: [[INS-UC-001#UC-H9]] · [[INS-UI-001#UI-7]]

```yaml
get:
  responses:
    200: { slots, last_assessment, checklist[], next_steps[] }
```

#### POST/api/v1/sessions/{sessionId}/submit 청구 접수

근거: [[INS-UC-001#UC-H9]]

```yaml
post:
  responses:
    200: { receipt_no, submitted_at, status, estimated_days, message }
```

**실제 전송이 아니다.** 화면 흐름 검증용이다. 근거: [[INS-PRD-001]] 2장

#### DELETE/api/v1/sessions/{sessionId} 세션 폐기

```yaml
delete:
  responses:
    204: 멱등
```

#### POST/api/v1/sessions/help 도움 챗봇

근거: [[INS-UI-001#UI-9]]

```yaml
post:
  body: { text: str(1..1000) }
  responses:
    200: { message, citations[], related_questions[] }
```

**세션이 없다.** 단발 질의응답이라 앞 대화를 기억하지 않는다.

#### POST/api/v1/auth/demo-login 데모 로그인

근거: [[INS-UC-001#UC-H2]] · [[INS-INFRA-001#C5]]

```yaml
post:
  body: { name: str, phone: str }
  responses:
    200: { access_token, expires_in, user }   # 쿠키로도 내려간다
    404: DEMO_PERSONA_NOT_FOUND
```

#### GET/api/v1/auth/me/insurances 내 가입 보험

근거: [[INS-UC-001#UC-H3]] · [[INS-PRD-001#R12]]

```yaml
get:
  responses:
    200: { insurances[] }
    401: 로그인 필요
```

#### GET/api/v1/me/health/history 진료내역

근거: [[INS-UC-001#UC-H8]] · [[INS-PRD-001#R13]]

```yaml
get:
  responses:
    200: { treatments[] }   # 슬롯 매핑까지 마친 형태
    401: 로그인 필요
```

**환자 부담분을 청구 금액으로 옮긴다.** 총진료비가 아니다.

#### GET/api/v1/admin/graph 약관 그래프

근거: [[INS-UC-001#UC-A10]] · [[INS-UI-001#UI-8]]

```yaml
get:
  parameters: [insurer_id, scope]
  responses:
    200: { nodes[], edges[], node_count, edge_count }
    404: 권한 없음 또는 비활성 (존재를 숨긴다)
    503: 그래프 저장소 장애
```

같은 접두 아래 `/tree`·`/node`·`/path`·`/scopes`가 있다.

#### GET/health 라이브니스

```yaml
get:
  responses:
    200: { status: "ok" }
```

#### GET/metrics 지표

근거: [[INS-PRD-001#R22]]

```yaml
get:
  responses:
    200: text/plain (Prometheus)
```

## 4. 미결사항

- [ ] **세션 경로에 소유자 검사가 없다** — 세션 ID만 알면 조회·주입·삭제·접수가 된다. 데모 범위의 결정이지만 운영 전환 시 첫 항목이다. 근거: [[INS-INFRA-001#C4]]
- [ ] **같은 파일 안에서 인증 부착이 갈린다** — 세션 생성·대화에는 선택적 인증이 붙어 있고 조회·주입·요약·접수·삭제에는 없다. 의도적 구분인지 누락인지 정해야 한다
- [ ] **비스트리밍 대화 경로가 남아 있다** — 화면은 안 쓴다. 두 경로의 오류 처리가 따로 살아 있어 한쪽만 고치면 갈린다
- [ ] **`/checklist`를 부르는 곳이 없다** — 요약이 그 내용을 품고 있다
- [ ] **회원가입·로그인 API에 화면이 없다** — 실제 로그인 경로는 데모 로그인 하나다
- [ ] **요청 제한이 어디에도 안 붙었다.** 근거: [[INS-PRD-001#R21]]
- [ ] 업로드 응답이 슬롯을 돌려주기만 하고 세션에 넣지 않는다 — 넣는 경로(`/slots`)가 이미 있는데 부르지 않는다. 근거: [[INS-PRD-001#R11]]

---
doc_id: INS-API-002
type: API
title: API 명세 REST — 보험길잡이
status: draft
upstream: [INS-UC-002, INS-UI-003, INS-DOM-004, INS-INFRA-002]
---

# API 명세 REST — 보험길잡이

> 1. 엔드포인트는 30개다. `/api/v1` 아래 26개, 상태 확인·지표 2개, 정적 파일 경로 2개다.
> 2. 중심은 세션이다. 세션을 만들고, 메시지를 SSE로 보내 판정을 받고, 청구 준비 요약을 받아 가정으로 접수한다.
> 3. 앱이 만드는 OpenAPI 스키마와 라우터 코드를 대조해 썼다. 대조하다 찾은 차이는 미결사항에 적었다. 요청 제한 미적용, 에러 모양 셋, 세션 접근이 그 예다.

## 0. 이 문서가 다루는 것

- 기준은 실행 중인 앱의 OpenAPI 스키마(`app.openapi()`)다. 여기에 경로 26개, 작업 27개가 나온다. 스키마에서 빠진 `/metrics`와 정적 파일 경로 둘을 더했다
- 사용자 API는 모두 `/api/v1` 아래다. nginx가 `/api`·`/static`·`/health`를 백엔드로 넘긴다([[INS-INFRA-002]] 2장)
- 엔드포인트마다 부르는 화면([[INS-UI-003]])과 유스케이스([[INS-UC-002]]), 이어지는 서비스 함수를 적는다. 화면에서 부르지 않는 엔드포인트는 "화면 없음"으로 적는다
- 스키마 이름은 도메인 모델([[INS-DOM-004]])의 개념 이름과 같다

## 1. 규칙

| 규칙 | 내용 |
|---|---|
| 기본 경로 | `/api/v1`. 버전은 경로 앞머리로만 나눈다 |
| 인증 | 로그인하면 JWT(HS256, 60분)를 응답 본문과 httponly 쿠키 `access_token`으로 준다. 요청은 쿠키나 `Authorization: Bearer`로 보낸다 |
| 로그인이 필요한 곳 | `/auth/me`, `/auth/me/insurances`, `/me/health/history`. 나머지 사용자 API는 로그인이 선택이다 |
| 세션 | 세션 id로 가리킨다. 세션은 서버 메모리에 있고 30분 동안 쓰지 않으면 사라진다. 없으면 `SESSION_NOT_FOUND`다 |
| 세션의 주인 | 세션은 사용자에 묶이지 않는다. 세션 id를 아는 요청은 그 세션을 읽고 쓸 수 있다(5장) |
| 스트리밍 | `.../messages/stream`은 `text/event-stream`이다. `delta`(글 조각)를 여러 번 보내고, 끝에 `final`(완성 응답)이나 `error`(코드·메시지)를 한 번 보낸다. 스트림 안의 에러는 HTTP 상태가 아니라 `error` 이벤트로 온다 |
| 진행 문구 | "관련된 문서 찾는중" 같은 진행 문구는 서버 이벤트가 아니다. 화면이 스스로 바꿔 보여 준다 |
| 시각 | ISO 8601 문자열이다 |
| 교차 출처 | `CORS_ALLOW_ORIGINS`에 적은 출처만 받는다. 쿠키를 싣도록 credentials를 허용하고, 메서드는 GET·POST·DELETE다 |
| 운영 모드 | `APP_ENV=production`이면 `/auth/demo-personas`·`/auth/demo-login`이 404다 |
| 관리자 | `ADMIN_GRAPH_ENABLED`가 거짓이면 `/admin/*`가 404다. 운영 모드에서는 로그인 사용자만 쓴다 |
| 요청 제한 | 설정값(IP당 분당 10회, 세션당 분당 30회)은 있지만 **어떤 엔드포인트에도 걸려 있지 않다**. 전역 기본 한도도 비어 있다(5장) |

## 2. 에러

에러 본문의 모양은 지금 세 가지다. 5장에서 하나로 맞춘다.

| 모양 | 어디서 |
|---|---|
| `{"detail": {"error": {"code", "message"}}}` | 세션·청구 준비 엔드포인트 |
| `{"detail": {"code", "message"}}` | 인증·첨부·진료내역·관리자 엔드포인트 |
| `{"error": {"code", "message"}}` | 처리되지 않은 모든 예외(500) |
| `{"detail": [...]}` | 요청 형식 검증 실패(422) |

| 코드 | 상태 | 언제 |
|---|---|---|
| `SESSION_NOT_FOUND` | 404 | 세션이 없거나 30분이 지나 사라졌다 |
| `LLM_UNAVAILABLE` | 503 | 모델 호출이 실패했다 |
| `INTERNAL` | — | 스트림 안에서 예상 못 한 오류가 났다(`error` 이벤트) |
| `AUTH_REQUIRED` | 401 | 로그인이 필요한 곳에 로그인 없이 왔다 |
| `UNAUTHORIZED` | 401 | 진료내역을 로그인 없이 요청했다 |
| `INVALID_CREDENTIALS` | 401 | 이메일·비밀번호가 맞지 않는다 |
| `EMAIL_ALREADY_REGISTERED` | 409 | 이미 가입된 이메일이다 |
| `DEMO_PERSONA_NOT_FOUND` | 404 | 이름·휴대폰 번호에 맞는 시연 계정이 없다 |
| `NOT_FOUND` | 404 | 운영 모드의 데모 경로, 또는 꺼진 관리자 경로다 |
| `INVALID_FILE` | 400 | 받지 않는 파일 형식이거나 크기다 |
| `FILE_READ_ERROR` | 400 | 올린 파일을 읽지 못했다 |
| `STORAGE_ERROR` | 500 | 올린 파일을 저장하지 못했다 |
| `OCR_FAILED` | 502 | 서류 추출 호출이 실패했다 |
| `OCR_NOT_CONFIGURED` | 503 | 서류 추출 설정이 없다 |
| `HEALTH_DATA_NOT_CONFIGURED` | 503 | 진료내역 연동 설정이 없다 |
| `GRAPH_UNAVAILABLE` | 503 | 그래프 저장소에 연결하지 못했다 |
| `NODE_NOT_FOUND` | 404 | 그래프에 없는 노드다 |
| `PATH_NOT_FOUND` | 404 | 두 노드를 잇는 경로가 없다 |
| `INTERNAL_ERROR` | 500 | 처리되지 않은 예외다. 내부 상세는 응답에 싣지 않는다 |
| (검증) | 422 | 요청 본문·파라미터 형식이 틀렸다 |

## 3. 엔드포인트

### 3.1 세션과 대화

#### POST/api/v1/sessions 세션 만들기

화면 [[INS-UI-003#UI-4]] · 유스케이스 [[INS-UC-002#UC-H1]] · 서비스 `sessions.service.create_session`

첫 상황을 함께 보내면 첫 응답까지 만들어 돌려준다.

```yaml
/api/v1/sessions:
  post:
    summary: 세션을 만든다. 첫 상황이 있으면 첫 응답도 돌려준다
    requestBody:
      required: false
      content:
        application/json:
          schema: {$ref: '#/components/schemas/SessionCreate'}
    responses:
      '201':
        description: 만들었다
        content:
          application/json:
            schema: {$ref: '#/components/schemas/SessionCreateResponse'}
      '503': {description: LLM_UNAVAILABLE}
```

#### POST/api/v1/sessions/{session_id}/messages/stream 메시지 보내기(스트리밍)

화면 [[INS-UI-003#UI-5]] · [[INS-UI-003#UI-6]] · 유스케이스 [[INS-UC-002#UC-H1]] · [[INS-UC-002#UC-H3]] · 서비스 `sessions.service.post_message`

화면이 쓰는 기본 경로다. 답을 만들어지는 대로 흘려보낸다.

```yaml
/api/v1/sessions/{session_id}/messages/stream:
  post:
    summary: 메시지를 보내고 답을 SSE로 받는다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/MessageRequest'}
    responses:
      '200':
        description: >
          event: delta  data: {"text": "..."}  (여러 번)
          event: final  data: SessionResponse   (끝에 한 번)
          event: error  data: {"code": SESSION_NOT_FOUND|LLM_UNAVAILABLE|INTERNAL, "message": "..."}
        content:
          text/event-stream: {schema: {type: string}}
```

#### POST/api/v1/sessions/{session_id}/messages 메시지 보내기

화면 없음 · 유스케이스 [[INS-UC-002#UC-H1]] · [[INS-UC-002#UC-H3]] · 서비스 `sessions.service.post_message`

스트리밍과 같은 처리를 한 번에 돌려준다. 지금 화면은 쓰지 않는다.

```yaml
/api/v1/sessions/{session_id}/messages:
  post:
    summary: 메시지를 보내고 완성된 응답을 한 번에 받는다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/MessageRequest'}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/SessionResponse'}
      '404': {description: SESSION_NOT_FOUND}
      '503': {description: LLM_UNAVAILABLE}
```

#### POST/api/v1/sessions/{session_id}/slots 대화 정보 미리 채우기

화면 [[INS-UI-003#UI-3]] · 유스케이스 [[INS-UC-002#UC-H4]] · [[INS-UC-002#UC-H5]] · 서비스 `sessions.service.seed_slots`

가입 현황에서 고른 보험과 불러온 정보를 모델을 거치지 않고 대화 정보에 넣는다.

```yaml
/api/v1/sessions/{session_id}/slots:
  post:
    summary: 구조화된 값으로 대화 정보를 채운다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/SlotSeedRequest'}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/SlotSeedResponse'}
      '404': {description: SESSION_NOT_FOUND}
```

#### GET/api/v1/sessions/{session_id} 세션 상태 보기

화면 [[INS-UI-003#UI-6]] · 유스케이스 [[INS-UC-002#UC-H3]] · 서비스 `sessions.service.get_session`

화면을 다시 열 때 대화와 정보를 복원한다.

```yaml
/api/v1/sessions/{session_id}:
  get:
    summary: 세션의 상태·대화 정보·대화 기록·대화 메모를 돌려준다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/SessionStateResponse'}
      '404': {description: SESSION_NOT_FOUND}
```

#### DELETE/api/v1/sessions/{session_id} 세션 버리기

화면 [[INS-UI-003#UI-1]] · 유스케이스 [[INS-UC-002#UC-H1]] · 서비스 `sessions.service.close_session`

새 흐름을 시작할 때 직전 세션을 버린다.

```yaml
/api/v1/sessions/{session_id}:
  delete:
    summary: 세션을 버린다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    responses:
      '204': {description: 버렸다}
```

#### POST/api/v1/sessions/help 도움 챗봇에 묻기

화면 [[INS-UI-003#UI-8]] · 유스케이스 [[INS-UC-002#UC-H8]] · 서비스 `sessions.service.answer_help`

세션 없이 질문 하나에 답 하나를 돌려준다. 실손 일반 질문이면 표준약관 인용이 붙는다.

```yaml
/api/v1/sessions/help:
  post:
    summary: 사용법·실손 일반 질문에 답한다. 판정은 하지 않는다
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/HelpRequest'}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/HelpResponse'}
      '503': {description: LLM_UNAVAILABLE}
```

### 3.2 청구 준비

#### GET/api/v1/sessions/{session_id}/summary 청구 준비 요약

화면 [[INS-UI-003#UI-7]] · 유스케이스 [[INS-UC-002#UC-H7]] · 서비스 `claims.service.build_summary`

필요 서류 목록을 안에 담아 돌려준다.

```yaml
/api/v1/sessions/{session_id}/summary:
  get:
    summary: 마지막 판정으로 청구 준비 요약을 만든다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/ClaimSummary'}
      '404': {description: SESSION_NOT_FOUND}
```

#### GET/api/v1/sessions/{session_id}/checklist 필요 서류 목록

화면 없음 · 유스케이스 [[INS-UC-002#UC-H7]] · 서비스 `claims.service.build_checklist`

요약에 같은 목록이 들어 있어 지금 화면은 쓰지 않는다.

```yaml
/api/v1/sessions/{session_id}/checklist:
  get:
    summary: 필요 서류 목록만 돌려준다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/ClaimChecklist'}
      '404': {description: SESSION_NOT_FOUND}
```

#### POST/api/v1/sessions/{session_id}/submit 청구 접수하기(가정)

화면 [[INS-UI-003#UI-7]] · 유스케이스 [[INS-UC-002#UC-H7]] · 서비스 `claims.service.submit_claim`

보험사로 아무것도 보내지 않고, 가정으로 접수 확인을 만든다.

```yaml
/api/v1/sessions/{session_id}/submit:
  post:
    summary: 청구를 가정으로 접수한다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/ClaimReceipt'}
      '404': {description: SESSION_NOT_FOUND}
```

### 3.3 서류

#### POST/api/v1/sessions/{session_id}/documents 서류 사진 올리기

화면 [[INS-UI-003#UI-6]] · 유스케이스 [[INS-UC-002#UC-H6]] · 서비스 없음 — 라우터가 OCR·정보 추출 어댑터, 모델 분류·추출, 첨부 저장, 세션 저장소를 직접 부른다

서류 종류를 가리고 항목을 뽑아 돌려준다. 첨부는 24시간 뒤 지운다.

```yaml
/api/v1/sessions/{session_id}/documents:
  post:
    summary: 서류 사진을 올려 항목을 뽑는다
    parameters:
      - {name: session_id, in: path, required: true, schema: {type: string}}
    requestBody:
      required: true
      content:
        multipart/form-data:
          schema:
            type: object
            required: [file]
            properties:
              file: {type: string, format: binary}
    responses:
      '200':
        content:
          application/json:
            schema:
              type: object
              properties:
                attachment: {$ref: '#/components/schemas/AttachmentMeta'}
                doc_type: {type: string}
                doc_type_confidence: {type: number}
                doc_type_reason: {type: string}
                extracted_slots: {type: object}
                ocr_confidence: {type: number}
                low_confidence: {type: boolean, description: 신뢰도 0.6 미만이면 참 — 다시 찍기 안내}
      '400': {description: INVALID_FILE · FILE_READ_ERROR}
      '404': {description: SESSION_NOT_FOUND}
      '500': {description: STORAGE_ERROR}
      '502': {description: OCR_FAILED}
      '503': {description: OCR_NOT_CONFIGURED}
```

### 3.4 인증과 내 정보

#### POST/api/v1/auth/demo-login 시연 계정 로그인

화면 [[INS-UI-003#UI-2]] · 유스케이스 [[INS-UC-002#UC-H4]] · 서비스 없음 — 라우터가 시연 계정 목록과 DB 세션을 직접 쓴다

이름과 휴대폰 번호로 시연 계정을 찾아 로그인한다.

```yaml
/api/v1/auth/demo-login:
  post:
    summary: 시연 계정으로 로그인하고 쿠키를 건다
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/DemoLoginRequest'}
    responses:
      '200':
        headers:
          Set-Cookie: {schema: {type: string}, description: 'access_token (httponly, samesite=lax)'}
        content:
          application/json:
            schema: {$ref: '#/components/schemas/TokenResponse'}
      '404': {description: DEMO_PERSONA_NOT_FOUND · 운영 모드면 NOT_FOUND}
```

#### GET/api/v1/auth/demo-personas 시연 계정 목록

화면 없음 · 유스케이스 [[INS-UC-002#UC-H4]] · 서비스 없음 — 라우터가 시연 계정 목록을 읽는다

본인 확인 화면의 목록 선택을 없앤 뒤로 화면은 쓰지 않는다.

```yaml
/api/v1/auth/demo-personas:
  get:
    summary: 시연 계정 목록을 돌려준다
    responses:
      '200':
        content:
          application/json:
            schema: {type: array, items: {$ref: '#/components/schemas/DemoPersona'}}
      '404': {description: 운영 모드면 NOT_FOUND}
```

#### POST/api/v1/auth/signup 가입하기

화면 없음 · 유스케이스 없음 · 서비스 `users.service.create_user`

```yaml
/api/v1/auth/signup:
  post:
    summary: 이메일·비밀번호로 가입한다
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/UserCreate'}
    responses:
      '201':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/UserRead'}
      '409': {description: EMAIL_ALREADY_REGISTERED}
```

#### POST/api/v1/auth/login 로그인하기

화면 없음 · 유스케이스 없음 · 서비스 `users.service.authenticate`

```yaml
/api/v1/auth/login:
  post:
    summary: 이메일·비밀번호로 로그인하고 쿠키를 건다
    requestBody:
      required: true
      content:
        application/json:
          schema: {$ref: '#/components/schemas/LoginRequest'}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/TokenResponse'}
      '401': {description: INVALID_CREDENTIALS}
```

#### POST/api/v1/auth/logout 로그아웃하기

화면 없음 · 유스케이스 없음 · 서비스 없음 — 쿠키만 지운다

```yaml
/api/v1/auth/logout:
  post:
    summary: 로그인 쿠키를 지운다
    responses:
      '204': {description: 지웠다}
```

#### GET/api/v1/auth/me 내 계정 보기

화면 없음 · 유스케이스 없음 · 서비스 없음

```yaml
/api/v1/auth/me:
  get:
    summary: 로그인한 사용자를 돌려준다
    security: [{cookieAuth: []}, {bearerAuth: []}]
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/UserRead'}
      '401': {description: AUTH_REQUIRED}
```

#### GET/api/v1/auth/me/insurances 내 가입 보험

화면 [[INS-UI-003#UI-3]] · [[INS-UI-003#UI-7]] · 유스케이스 [[INS-UC-002#UC-H4]] · 서비스 없음 — 라우터가 마이데이터 어댑터를 직접 부른다

실손만 돌려준다. 실손이 아닌 보험은 어댑터가 버린다.

```yaml
/api/v1/auth/me/insurances:
  get:
    summary: 마이데이터로 가입한 실손을 돌려준다
    security: [{cookieAuth: []}, {bearerAuth: []}]
    responses:
      '200':
        content:
          application/json:
            schema:
              type: object
              properties:
                insurances: {type: array, items: {type: object, description: 보험사·상품·증권번호·가입일·세대}}
      '401': {description: AUTH_REQUIRED}
```

#### GET/api/v1/me/health/history 내 진료내역

화면 [[INS-UI-003#UI-8]] · 유스케이스 [[INS-UC-002#UC-H4]] · 서비스 없음 — 라우터가 건강보험 어댑터를 직접 부른다

도움 챗봇의 빠른 작업이 부른다. 고른 진료는 설명 문장이 되어 상담에 보내진다.

```yaml
/api/v1/me/health/history:
  get:
    summary: 최근 진료내역을 돌려준다
    security: [{cookieAuth: []}, {bearerAuth: []}]
    responses:
      '200':
        content:
          application/json:
            schema:
              type: object
              properties:
                treatments: {type: array, items: {$ref: '#/components/schemas/TreatmentCard'}}
      '401': {description: UNAUTHORIZED}
      '503': {description: HEALTH_DATA_NOT_CONFIGURED}
```

### 3.5 약관 데이터

#### GET/api/v1/documents/insurers 보험사 목록

화면 없음 · 유스케이스 없음 · 서비스 `documents.service.list_insurers`

```yaml
/api/v1/documents/insurers:
  get:
    summary: 적재된 보험사를 돌려준다
    responses:
      '200':
        content:
          application/json:
            schema: {type: array, items: {$ref: '#/components/schemas/InsurerRead'}}
```

#### GET/api/v1/documents/products 상품 목록

화면 없음 · 유스케이스 없음 · 서비스 `documents.service.list_products`

```yaml
/api/v1/documents/products:
  get:
    summary: 적재된 상품을 돌려준다
    parameters:
      - {name: insurer, in: query, required: false, schema: {type: string}}
      - {name: area, in: query, required: false, schema: {type: string}}
    responses:
      '200':
        content:
          application/json:
            schema: {type: array, items: {$ref: '#/components/schemas/ProductRead'}}
```

#### GET/static/page_images/{file} 약관 페이지 이미지

화면 [[INS-UI-003#UI-6]] · 유스케이스 [[INS-UC-002#UC-H2]] · 서비스 없음 — 정적 파일

인용의 `page_image_url`이 가리킨다. PDF 페이지를 이미지로 만든 캐시다.

```yaml
/static/page_images/{file}:
  get:
    summary: 약관 페이지 이미지를 돌려준다
    parameters:
      - {name: file, in: path, required: true, schema: {type: string}}
    responses:
      '200': {content: {image/png: {schema: {type: string, format: binary}}}}
      '404': {description: 파일 없음}
```

#### GET/static/raw/{path} 원본 약관 PDF

화면 [[INS-UI-003#UI-6]] · 유스케이스 [[INS-UC-002#UC-H2]] · 서비스 없음 — 정적 파일

인용의 `pdf_url`이 가리킨다. 약관은 공개 문서다.

```yaml
/static/raw/{path}:
  get:
    summary: 원본 약관 PDF를 돌려준다
    parameters:
      - {name: path, in: path, required: true, schema: {type: string}}
    responses:
      '200': {content: {application/pdf: {schema: {type: string, format: binary}}}}
      '404': {description: 파일 없음}
```

### 3.6 관리자 그래프

#### GET/api/v1/admin/graph 그래프 보기

화면 [[INS-UI-003#UI-9]] · 유스케이스 [[INS-UC-002#UC-A1]] · 서비스 `admin.service` (그래프 데이터원 포트, Memgraph 어댑터)

```yaml
/api/v1/admin/graph:
  get:
    summary: 범위 안의 노드와 간선을 돌려준다
    parameters:
      - {name: insurer_id, in: query, required: false, schema: {type: string}}
      - {name: scope, in: query, required: false, schema: {type: string}, description: 문서·구간 노드 id. root면 전체}
    responses:
      '200':
        content:
          application/json:
            schema: {$ref: '#/components/schemas/GraphView'}
      '404': {description: NOT_FOUND — 노출이 꺼져 있다}
      '503': {description: GRAPH_UNAVAILABLE}
```

#### GET/api/v1/admin/graph/tree TDD 트리

화면 [[INS-UI-003#UI-9]] · 유스케이스 [[INS-UC-002#UC-A1]] · 서비스 `admin.service` (그래프 데이터원 포트)

```yaml
/api/v1/admin/graph/tree:
  get:
    summary: 전체 → 보험사 → 문서 → 구간 → 조 → 하위 조항 트리를 돌려준다
    responses:
      '200': {content: {application/json: {schema: {type: object}}}}
      '404': {description: NOT_FOUND}
      '503': {description: GRAPH_UNAVAILABLE}
```

#### GET/api/v1/admin/graph/node 노드 상세

화면 [[INS-UI-003#UI-9]] · 유스케이스 [[INS-UC-002#UC-A1]] · 서비스 `admin.service` (그래프 데이터원 포트)

```yaml
/api/v1/admin/graph/node:
  get:
    summary: 노드의 메타·원문·자식을 돌려준다
    parameters:
      - {name: node_id, in: query, required: true, schema: {type: string}}
    responses:
      '200': {content: {application/json: {schema: {type: object}}}}
      '404': {description: NODE_NOT_FOUND · NOT_FOUND}
      '503': {description: GRAPH_UNAVAILABLE}
```

#### GET/api/v1/admin/graph/path 최단 경로

화면 [[INS-UI-003#UI-9]] · 유스케이스 [[INS-UC-002#UC-A1]] · 서비스 `admin.service` (그래프 데이터원 포트)

```yaml
/api/v1/admin/graph/path:
  get:
    summary: 두 노드 사이 최단 경로를 돌려준다
    parameters:
      - {name: source, in: query, required: true, schema: {type: string}}
      - {name: target, in: query, required: true, schema: {type: string}}
      - {name: insurer_id, in: query, required: false, schema: {type: string}}
      - {name: scope, in: query, required: false, schema: {type: string}}
    responses:
      '200': {content: {application/json: {schema: {type: object, description: 경로 노드·간선·홉 수}}}}
      '404': {description: PATH_NOT_FOUND · NOT_FOUND}
      '503': {description: GRAPH_UNAVAILABLE}
```

#### GET/api/v1/admin/graph/scopes 범위 목록

화면 없음 · 유스케이스 [[INS-UC-002#UC-A1]] · 서비스 `admin.service` (그래프 데이터원 포트)

지금 화면은 트리로 범위를 고르므로 쓰지 않는다.

```yaml
/api/v1/admin/graph/scopes:
  get:
    summary: 고를 수 있는 범위(문서) 목록을 돌려준다
    responses:
      '200': {content: {application/json: {schema: {type: array, items: {type: object}}}}}
      '404': {description: NOT_FOUND}
      '503': {description: GRAPH_UNAVAILABLE}
```

### 3.7 운영

#### GET/health 상태 확인

화면 없음 · 운영 [[INS-INFRA-002]] 8장 · 서비스 없음

```yaml
/health:
  get:
    summary: 살아 있는지 확인한다
    responses:
      '200':
        content:
          application/json:
            schema: {type: object, properties: {status: {type: string, example: ok}}}
```

#### GET/metrics 지표

화면 없음 · 운영 [[INS-INFRA-002]] 6장 · 서비스 없음

OpenAPI 스키마에서 뺀 경로다. 지금 이 지표를 긁어 가는 수집기는 없다.

```yaml
/metrics:
  get:
    summary: Prometheus 형식 지표를 돌려준다
    responses:
      '200': {content: {text/plain: {schema: {type: string}}}}
```

## 4. 스키마

여러 엔드포인트가 함께 쓰는 스키마다. 필드는 OpenAPI 스키마에서 옮겼고, `*`는 필수다. 대화 정보(`SlotState`)는 필드가 많아 주요 필드만 적었다.

```yaml
components:
  securitySchemes:
    cookieAuth: {type: apiKey, in: cookie, name: access_token}
    bearerAuth: {type: http, scheme: bearer, bearerFormat: JWT}
  schemas:
    SessionCreate: {initial_message: string|null}
    SessionCreateResponse: {session_id*: string, created_at*: datetime, ttl_seconds*: integer, first_response: SessionResponse|null}
    MessageRequest: {text*: string}
    SessionResponse:
      session_id*: string
      turn*: integer
      assistant*: AssistantAsk | AssistantAssessment | AssistantAnswer | AssistantComparison
      slots*: SlotState
      status*: gathering | analyzing | answered | closed
    SessionStateResponse: {session_id*: string, created_at*: datetime, last_activity_at*: datetime, status*: string, slots*: SlotState, history: [Message], notes: [string]}
    Message: {role*: user|assistant, content*: string, created_at*: datetime, response_type: ask|assessment|answer|comparison|null}
    SlotState: {insurer, insurer_id, product, policy_no, generation, incident_date, incident_location, diagnosis, diagnosis_code, hospital, hospitalization_days, outpatient_visits, treatment_period, claim_amount, purpose, treatment_overseas, is_oriental_medicine, dental_disease, other_insurance_settled, unknown_slots, "…"}
    SlotSeedRequest: {insurer, insurer_id, product, policy_no, incident_date, diagnosis, diagnosis_code, hospital, hospitalization_days, outpatient_visits, treatment_period, claim_amount, incident_location, policies: [PolicyRef]}
    SlotSeedResponse: {slots*: SlotState, missing*: [string]}
    PolicyRef: {insurer_id*: string, insurer*: string, product*: string, policy_no*: string, generation: integer|null}
    AssistantAsk: {type: ask, message*: string, expected_slots*: [string], options: [string]}
    AssistantAssessment:
      type: assessment
      likelihood*: 높음 | 중간 | 낮음
      summary*: string
      satisfied: [string]
      unsatisfied: [string]
      citations*: [Citation]
      next_steps: [string]
      confidence: partial | full
      readiness: ReadinessScore|null
      reclaim: ReclaimPlan|null
      disclaimer: string
    AssistantAnswer: {type: answer, message*: string, citations*: [Citation], related_questions: [string], needs_policy: boolean, disclaimer: string}
    AssistantComparison: {type: comparison, policies*: [PolicyAssessment], summary*: string, recommended_policy_no: string|null, disclaimer: string}
    PolicyAssessment: {insurer_id*: string, insurer*: string, product*: string, policy_no*: string, assessment*: AssistantAssessment, deductible*: DeductibleView}
    DeductibleView: {generation: integer|null, covered_rate*: number, non_covered_rate*: number, outpatient_min_copay*: integer, payable_estimate: integer|null, prorated_share: integer|null}
    Citation: {chunk_id*: string, insurer*: string, product*: string, version*: string, doc_type*: summary|business|terms, clause*: string, sub_no: string|null, text*: string, page*: integer, page_image_url: string|null, pdf_url: string|null, highlights: [HighlightBox]}
    HighlightBox: {x: number, y: number, w: number, h: number}
    ReadinessScore: {score*: integer, level*: high|medium|low, factors*: [ReadinessFactor], caption*: string}
    ReadinessFactor: {label: string, points: integer, max_points: integer}
    ReclaimPlan: {applicable*: boolean, items: [ReclaimItem], note: string}
    ReclaimItem: {gap*: string, action*: string, basis*: string}
    HelpRequest: {text*: string}
    HelpResponse: {message*: string, citations: [Citation], related_questions: [string]}
    ClaimChecklist: {area: string|null, items: [ChecklistItem]}
    ChecklistItem: {id*: string, label*: string, required*: boolean, reason*: string}
    ClaimSummary: {insurer, product, area, likelihood, summary, satisfied: [string], unsatisfied: [string], next_steps: [string], checklist: [ChecklistItem]}
    ClaimReceipt: {receipt_no*: string, submitted_at*: datetime, status: string, insurer: string|null, estimated_days: integer, message*: string}
    AttachmentMeta: {id, session_id, sha256, size, mime_type, filename, created_at, expires_at}
    TreatmentCard: {treatment_id, treatment_date, hospital_name, department, diagnosis_summary, is_hospitalization, hospitalization_days, outpatient_visits, total_cost, claim_amount}
    DemoLoginRequest: {name*: string, phone*: string}
    DemoPersona: {name*: string, phone*: string, dob*: string, label*: string}
    LoginRequest: {email*: string, password*: string}
    UserCreate: {email*: string, password*: string}
    UserRead: {id*: integer, email*: string, created_at*: datetime}
    TokenResponse: {access_token*: string, token_type: string, expires_in*: integer, user*: UserRead}
    InsurerRead: {id*: string, name*: string, homepage_url: string|null, created_at*: datetime}
    ProductRead: {id*: string, insurer_id*: string, area*: string, name*: string, created_at*: datetime}
    GraphView: {nodes: [{id, label, node_type, "…"}], edges: [{id, source, target, type}], node_count: integer, edge_count: integer}
```

`AssistantAnswer`(판정 뒤 자유 질의의 답)는 도메인 모델에 아직 개념으로 없다. 클래스 명세 때 도메인 모델에 더한다.

## 5. 미결사항

- [ ] **요청 제한 미적용** — 설정값(IP당 분당 10회, 세션당 분당 30회)은 있지만 어떤 엔드포인트에도 걸려 있지 않다. PRD [[INS-PRD-002#N3]]와 인프라 [[INS-INFRA-002#C4]]가 적은 요청 제한이 실제로는 없다
- [ ] **에러 모양 통일** — 에러 본문이 세 모양으로 갈린다. 프론트 클라이언트는 `{"detail": {"code", "message"}}` 모양을 읽지 못해 인증·첨부·진료내역·관리자 에러를 모두 `UNKNOWN`으로 처리한다. 한 모양으로 맞추는 처리기를 둘지
- [ ] **세션 접근** — 세션이 사용자나 쿠키에 묶이지 않아, 세션 id를 아는 요청은 그 세션의 대화 정보를 읽고 쓸 수 있다. 세션을 쿠키에 묶을지
- [ ] **로그아웃** — 로그아웃을 부르는 화면이 없다. 공용 기기에서는 로그인 쿠키가 60분 동안 남는다
- [ ] **화면에서 쓰지 않는 엔드포인트 10개** — 비스트리밍 메시지, 보험사·상품 목록, 시연 계정 목록, 가입·로그인·로그아웃·내 계정, 필요 서류 단독 조회, 그래프 범위 목록. 남길지 정리할지
- [ ] **서비스 없는 라우터** — 서류 업로드·시연 로그인·가입 보험·진료내역은 라우터가 어댑터·DB를 직접 부른다. router → service → crud 규칙에 맞게 서비스로 옮길지
- [ ] **진료내역 라우터 위치** — 도메인 폴더가 아니라 `app/infrastructure/external/health_data/`에 있다
- [ ] **빈 라우터** — chunks·search 라우터가 경로 없이 등록돼 있다
- [ ] **PDF 업로드** — 화면은 PDF를 보내는데 서버는 `INVALID_FILE`로 거절한다 ([[INS-UI-003#UI-6]])
- [ ] **관리자 화면의 직접 호출** — 관리자 그래프 화면이 API 클라이언트 모듈을 거치지 않고 fetch를 직접 쓴다
- [ ] **가입 보험의 실손 한정** — 이 엔드포인트가 실손만 돌려줘서, 가입 현황에 실손이 아닌 보험을 보여 줄 수 없다 ([[INS-PRD-002#R16]])

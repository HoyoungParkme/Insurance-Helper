---
doc_id: INS-MS-002
type: MS
title: 함수 명세 — 보험길잡이
status: draft
upstream: [INS-SEQ-002, INS-DOM-005, INS-API-002, INS-UC-002]
---

# 함수 명세 — 보험길잡이

> 1. 판정 파이프라인을 중심으로 함수 49개를 적었다. 한 턴 처리·판정 엔진·검색·답 생성·원본 캡처·서류 업로드·적재는 자세히, 나머지는 간략형(시그니처·처리·테스트)으로 적었다.
> 2. 항목 이름은 클래스 명세의 설계 클래스에 함수 이름을 붙였다(예: `SessionService.post_message`). 「호출하는 것」은 코드에서 뽑은 호출 관계를 따른다.
> 3. 함수를 읽다 찾은 것을 미결사항에 적었다. 구조화 LLM 호출에 재시도가 걸려 있지 않은 것, 서류 분류의 연결 오류가 500이 되는 것, 선택 서류가 없는 것, 임베딩 실패 뒤 건너뛰기다.

## 0. 이 문서가 다루는 것

- 기준은 저장소 코드다. 시그니처는 코드에서 그대로 옮겼다
- 항목 이름은 `설계 클래스.함수`다. 설계 클래스는 [[INS-DOM-005]] 4장의 이름이고, 모듈 함수 묶음도 같은 이름을 쓴다(예: `SessionLlm`은 `app/domains/sessions/llm.py`). 절차가 라우터에 있는 서류 업로드만 모듈 이름 `attachments_router`를 썼다
- 들고 나는 타입은 [[INS-DOM-005]] 2장에만 적고, 여기서는 이름으로 가리킨다
- 「호출하는 것」에는 이 문서의 다른 항목만 적는다. 비공개 도우미를 거쳐 닿는 것도 적는다. 항목이 아닌 함수(crud, LLM 호출 도우미, 외부 SDK)는 처리 단계에 적는다
- 분기는 `if 조건 → 결과 · else → 결과`로 적는다
- 화면 코드(`api/client.ts`)는 함수 이름이 항목 규칙(소문자·밑줄)과 맞지 않아 항목으로 두지 않았다. [[INS-DOM-005#ApiClient]]가 대신한다

## 1. 함수 목록

| 함수 | 하는 일 | 시퀀스 |
|---|---|---|
| [[#SessionService.create_session]] | 세션을 만들고, 첫 글이 있으면 첫 턴까지 | [[INS-SEQ-002#SEQ-1]] |
| [[#SessionService.post_message]] | 한 턴 처리 | [[INS-SEQ-002#SEQ-C1]] |
| [[#SessionService.seed_slots]] | 대화 정보를 모델 없이 미리 채우기 | [[INS-SEQ-002#SEQ-1]] |
| [[#SessionService.answer_help]] | 도움 답 | [[INS-SEQ-002#SEQ-8]] |
| [[#SessionStore.get]] | 세션 읽기와 만료 | [[INS-SEQ-002#SEQ-C1]] |
| [[#SessionLlm.classify_intent]] | 의도 가리기 | [[INS-SEQ-002#SEQ-C1]] |
| [[#SessionLlm.extract_slots]] | 사실 뽑기 | [[INS-SEQ-002#SEQ-C1]] |
| [[#SessionLlm.next_question]] | 되묻기 만들기 | [[INS-SEQ-002#SEQ-C1]] |
| [[#SessionLlm.generate_assessment]] | 판정 답 만들기 | [[INS-SEQ-002#SEQ-C3]] |
| [[#SessionLlm.generate_explanation]] | 설명 답 만들기 | [[INS-SEQ-002#SEQ-3]] |
| [[#SessionLlm.generate_help_answer]] | 도움 답 만들기 | [[INS-SEQ-002#SEQ-8]] |
| [[#SessionLlm.classify_document]] | 서류 종류 가리기 | [[INS-SEQ-002#SEQ-6]] |
| [[#SessionLlm.extract_slots_from_document]] | 서류에서 항목 뽑기(모델) | [[INS-SEQ-002#SEQ-6]] |
| [[#ReadinessCalculator.compute_readiness]] | 준비도 점수 | [[INS-SEQ-002#SEQ-C3]] |
| [[#CoverageEngine.build_facts_from_slots]] | 대화 정보 → 청구 사실 | [[INS-SEQ-002#SEQ-C1]] |
| [[#CoverageEngine.evaluate]] | 결정론 판정 | [[INS-SEQ-002#SEQ-C1]] |
| [[#CoverageEngine.rules_for]] | 적용할 규칙 고르기 | [[INS-SEQ-002#SEQ-C1]] |
| [[#CoverageEngine.compute_deductible]] | 자기부담 계산 | [[INS-SEQ-002#SEQ-C1]] |
| [[#ProrationCalculator.compute]] | 추천과 비례 안분 | [[INS-SEQ-002#SEQ-5]] |
| [[#RagService.retrieve]] | 대화 정보로 조항 찾기 | [[INS-SEQ-002#SEQ-C2]] |
| [[#RagService.retrieve_freeform]] | 자유 질의로 조항 찾기 | [[INS-SEQ-002#SEQ-3]] |
| [[#NeuroSymbolicRetriever.retrieve]] | 대화 정보 → 질의·필터 | [[INS-SEQ-002#SEQ-C2]] |
| [[#NeuroSymbolicRetriever.retrieve_freeform]] | 자유 질의 → 필터 | [[INS-SEQ-002#SEQ-C2]] |
| [[#NeuroSymbolicRetriever.retrieve_fused]] | 뉴럴·심볼릭 융합 | [[INS-SEQ-002#SEQ-C2]] |
| [[#VectorStoreAdapter.query]] | pgvector 검색 | [[INS-SEQ-002#SEQ-C2]] |
| [[#SymbolicGraphChannel.clause_candidates]] | 조항 제목 후보 | [[INS-SEQ-002#SEQ-C2]] |
| [[#SymbolicGraphChannel.expand]] | 구조 이웃 확장 | [[INS-SEQ-002#SEQ-C2]] |
| [[#EmbeddingService.embed_texts]] | 임베딩 | [[INS-SEQ-002#SEQ-10]] |
| [[#PdfImageService.find_highlights]] | 하이라이트 상자 | [[INS-SEQ-002#SEQ-C3]] |
| [[#PdfImageService.render_page]] | 페이지 이미지 | [[INS-SEQ-002#SEQ-C3]] |
| [[#ClaimsService.build_checklist]] | 필요 서류 | [[INS-SEQ-002#SEQ-7]] |
| [[#ClaimsService.build_summary]] | 청구 준비 요약 | [[INS-SEQ-002#SEQ-7]] |
| [[#ClaimsService.submit_claim]] | 가정 접수 | [[INS-SEQ-002#SEQ-7]] |
| [[#attachments_router.upload_document]] | 서류 업로드 절차 | [[INS-SEQ-002#SEQ-6]] |
| [[#AttachmentsService.save_bytes]] | 첨부 저장 | [[INS-SEQ-002#SEQ-6]] |
| [[#AttachmentsService.cleanup_expired]] | 만료 첨부 지우기 | — |
| [[#IngestionService.run_ingest]] | 약관 적재 | [[INS-SEQ-002#SEQ-10]] |
| [[#IngestionService.parse_pdf_path]] | 약관 경로 → 메타 | [[INS-SEQ-002#SEQ-10]] |
| [[#ChunksService.process_pdf]] | PDF → 청크 | [[INS-SEQ-002#SEQ-10]] |
| [[#DocumentsService.register_document]] | 약관 메타 등록 | [[INS-SEQ-002#SEQ-10]] |
| [[#GraphIndexer.sync_document]] | 문서 하나를 그래프에 맞추기 | [[INS-SEQ-002#SEQ-10]] |
| [[#DemoPersonaRegistry.find_demo_user]] | 시연 계정 찾기 | [[INS-SEQ-002#SEQ-4]] |
| [[#TokenService.get_current_user_optional]] | 로그인 사용자 확인 | [[INS-SEQ-002#SEQ-4]] |
| [[#MydataAdapter.fetch_insurances]] | 가입 보험 가져오기 | [[INS-SEQ-002#SEQ-4]] |
| [[#AuditService.begin]] | 감사 기록 열기 | [[INS-SEQ-002#SEQ-C1]] |
| [[#AuditService.complete]] | 감사 기록 성공 | [[INS-SEQ-002#SEQ-C1]] |
| [[#AuditService.fail]] | 감사 기록 실패 | [[INS-SEQ-002#SEQ-C1]] |
| [[#PiiMasker.mask_pii]] | 개인정보 가림 | [[INS-SEQ-002#SEQ-C1]] |
| [[#AdminGraphService.fetch_graph]] | 그래프 범위 읽기 | [[INS-SEQ-002#SEQ-9]] |

## 2. 함수

### 2.1 대화

#### SessionService.create_session 세션 만들기

**시그니처**
```python
def create_session(initial_message: str | None = None, *, user_id: int | None = None) -> tuple[Session, SessionResponse | None]
```

**처리**
1. 세션 저장소에 빈 세션을 만든다(`SessionStore.create`)
2. `if 첫 글이 공백이 아니다 → post_message(세션 id, 첫 글, user_id)의 응답을 함께 돌려준다 · else → 응답 None`

**호출하는 것** [[#SessionService.post_message]]

**테스트 관점** 첫 글이 없으면 LLM을 부르지 않는다. 첫 글이 있으면 첫 턴의 에러가 그대로 올라간다

근거: [[INS-SEQ-002#SEQ-1]] · [[INS-API-002#POST/api/v1/sessions]]

#### SessionService.post_message 한 턴 처리

**시그니처**
```python
def post_message(session_id: str, text: str, *, user_id: int | None = None, on_delta: Callable[[str], None] | None = None) -> SessionResponse
```

**입력** 세션 id, 사용자 글, 로그인 사용자 id(없으면 None), 글 조각을 받을 콜백(스트림일 때만)

**처리**
1. `SessionStore.get(session_id)` · `if 없음 → SessionNotFoundError`
2. `AuditService.begin(세션, 턴, 원문, user_id)` — 턴은 사용자 메시지 수 + 1
3. 사용자 메시지를 대화에 넣는다
4. `if 인사·잡담이고 모인 정보가 없다 → 환영 되묻기(LLM 없음)`
5. `SessionLlm.classify_intent` · `if general_qa → 설명 답 경로(9) · if out_of_domain → 범위 밖 안내 되묻기`
6. `SessionLlm.extract_slots` → 메모와 즉답 요청을 떼고 대화 정보에 합친다 · `if 보험사 이름이 바뀌었다 → 코드로 바꿔 갈아 끼운다 · 알려진 다섯 곳 밖이면 코드를 비운다(표준약관 모드)`
7. `CoverageEngine.evaluate(CoverageEngine.build_facts_from_slots(slots))`
8. `if 빠진 사실이 있고 부분 판정이 아니다 → SessionLlm.next_question으로 되묻기` — 부분 판정은 판정이 면책·조건부, 모름 칸 2개 이상, 되묻기 1번 이상, 즉답 요청 가운데 하나다
9. 설명 답 경로: `RagService.retrieve_freeform(질의, 8, 보험사 코드)` · `if 비었고 보험사 코드가 있다 → 필터 없이 다시` · `if 그래도 비었다 → 되묻기 · else → SessionLlm.generate_explanation`
10. `if 판정 대상 보험이 2개 이상 → 보험마다 대화 정보를 복사해 RagService.retrieve · CoverageEngine.evaluate · SessionLlm.generate_assessment(실패한 보험은 건너뜀) → if 판정이 2건 이상 → ProrationCalculator.compute로 비교 답 · else → 11로`
11. `if RAG_REACT → run_agent(실패하면 검색으로) · else → RagService.retrieve(slots, 8)` · `if 청크 없음 → 되묻기 · else → SessionLlm.generate_assessment(slots, 청크, coverage, 메모, on_delta)` → 마지막 판정으로 저장
12. 모든 응답 전에 상태를 바꾸고(`SessionStore.touch`) `AuditService.complete` · `if 예외 → AuditService.fail(PiiMasker.mask_pii(에러)) 뒤 다시 던진다`

**출력** `SessionResponse` — 응답은 되묻기·판정 답·설명 답·비교 답 가운데 하나

**예외**

| 조건 | 에러 |
|---|---|
| 세션이 없거나 30분이 지났다 | `SessionNotFoundError` → `SESSION_NOT_FOUND` |
| LLM 호출 실패·형식 위반 | `LLMError`·`SchemaViolationError` → `LLM_UNAVAILABLE` |

**호출하는 것** [[#SessionStore.get]] · [[#AuditService.begin]] · [[#AuditService.complete]] · [[#AuditService.fail]] · [[#PiiMasker.mask_pii]] · [[#SessionLlm.classify_intent]] · [[#SessionLlm.extract_slots]] · [[#SessionLlm.next_question]] · [[#SessionLlm.generate_assessment]] · [[#SessionLlm.generate_explanation]] · [[#CoverageEngine.build_facts_from_slots]] · [[#CoverageEngine.evaluate]] · [[#ProrationCalculator.compute]] · [[#RagService.retrieve]] · [[#RagService.retrieve_freeform]]

**테스트 관점**
- 되묻기를 한 번 한 뒤에는 빠진 사실이 있어도 판정 답이 나온다
- 판정이 면책이면 첫 턴에 되묻지 않는다
- 보험 둘을 미리 채우면 비교 답이 나오고, 하나가 검색에서 비면 단일 판정으로 돌아간다
- 어느 분기든 감사 기록이 한 행 남는다(성공은 `complete`, 예외는 `fail`)
- 인사만 보내면 LLM을 부르지 않는다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-SEQ-002#SEQ-3]] · [[INS-SEQ-002#SEQ-5]] · [[INS-API-002#POST/api/v1/sessions/{session_id}/messages/stream]]

#### SessionService.seed_slots 대화 정보 미리 채우기

**시그니처**
```python
def seed_slots(session_id: str, updates: dict[str, Any]) -> SlotSeedResponse
```

**입력** 세션 id, 채울 값(대화 정보 칸과 `policies` 목록)

**처리**
1. `SessionStore.get` · `if 없음 → SessionNotFoundError`
2. `if policies가 있다 → 세션의 판정 대상 보험으로 두고, 첫 보험의 보험사 코드·이름·상품·증권 번호·세대를 채울 값 앞에 둔다(직접 준 값이 이긴다)`
3. 대화 정보에 있는 칸이고 비어 있지 않은 값만 남겨 합친다
4. 상태를 `gathering`으로 바꾸고 대화 정보와 빠진 칸을 돌려준다

**출력** `SlotSeedResponse`

**예외**

| 조건 | 에러 |
|---|---|
| 세션 없음 | `SessionNotFoundError` → `SESSION_NOT_FOUND` |

**호출하는 것** [[#SessionStore.get]]

**테스트 관점** LLM을 부르지 않는다. 모르는 칸과 빈 값은 버린다. 첫 보험이 대표가 된다

근거: [[INS-SEQ-002#SEQ-1]] · [[INS-API-002#POST/api/v1/sessions/{session_id}/slots]]

#### SessionService.answer_help 도움 답

**시그니처**
```python
def answer_help(text: str) -> HelpAnswer
```

**처리**
1. `RagService.retrieve_freeform(text, 6)` — 보험사 필터 없음 · `if 예외 → 청크 없이 진행`
2. `SessionLlm.generate_help_answer(text, 청크)`

**호출하는 것** [[#RagService.retrieve_freeform]] · [[#SessionLlm.generate_help_answer]]

**테스트 관점** 검색이 실패해도 답이 나온다. 세션도 감사 기록도 만들지 않는다

근거: [[INS-SEQ-002#SEQ-8]] · [[INS-API-002#POST/api/v1/sessions/help]]

#### SessionStore.get 세션 읽기

**시그니처**
```python
def get(self, session_id: str) -> Session | None
```

**처리**
1. `if 사전에 없다 → None`
2. `if 마지막 활동에서 TTL(기본 1800초)이 지났다 → 사전에서 지우고 None · else → 세션`

**테스트 관점** 만료된 세션은 읽는 순간 지워진다. 읽기는 마지막 활동 시각을 바꾸지 않는다(`touch`가 바꾼다)

근거: [[INS-SEQ-002#SEQ-C1]]

### 2.2 대화 LLM

#### SessionLlm.classify_intent 의도 가리기

**시그니처**
```python
def classify_intent(text: str, slots: SlotState, answered: bool = False, last_assistant: str | None = None) -> str
```

**처리**
1. `if 판정 전이고 진단명·입원 일수·통원 횟수 가운데 하나가 있다 → claim_diagnosis(LLM 없음)`
2. 판정 여부, 직전 안내 앞 200자, 대화 정보, 입력을 도구 호출로 보낸다(temperature 0)
3. `if 결과가 claim_diagnosis·general_qa·out_of_domain 가운데 하나 → 그대로 · else 또는 예외 → claim_diagnosis`

**테스트 관점** 호출이 실패해도 예외 없이 판정 흐름으로 간다. 판정 전에 정보가 모였으면 모델을 부르지 않는다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-UC-002#UC-S1]]

#### SessionLlm.extract_slots 사실 뽑기

**시그니처**
```python
def extract_slots(history: list[Message], user_msg: str, current_slots: SlotState) -> dict[str, Any]
```

**입력** 이전 대화, 이번 입력, 지금의 대화 정보

**처리**
1. 오늘 날짜와 지금의 대화 정보를 넣은 시스템 프롬프트로 도구 호출을 한다(temperature 0)
2. `if slot_updates가 dict가 아니다 → SchemaViolationError`
3. 대화 정보에 있는 칸만 남긴다
4. `if unknown_slots가 있다 → 기존 모름 칸과 합친다`
5. `if session_notes가 있다 → 하나씩 60자로 잘라 _notes로 붙인다`
6. `if wants_immediate_answer → _wants_immediate_answer 표시`

**출력** 합칠 값(dict). `_notes`·`_wants_immediate_answer`는 부르는 쪽이 떼어 쓴다

**예외**

| 조건 | 에러 |
|---|---|
| 도구 호출 실패 | `LLMError` |
| 응답 형식이 틀렸다 | `SchemaViolationError` — 다시 요청하지 않는다 |

**테스트 관점** 없는 칸 이름은 버린다. "모르겠어요"는 모름 칸이 된다. 메모는 60자에서 잘린다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-UC-002#UC-S1]]

#### SessionLlm.next_question 되묻기 만들기

**시그니처**
```python
def next_question(slots: SlotState, missing: list[str]) -> AssistantAsk
```

**처리**
1. 대화 정보와 빠진 칸(우선순위 순)을 보내 1~2개를 묻는 질문을 받는다(도구 호출, temperature 0)
2. `AssistantAsk`로 검증한다 · `if 맞지 않는다 → SchemaViolationError`

**테스트 관점** 선택지가 없으면 빈 목록이다. 호출 실패는 `LLMError`다

근거: [[INS-SEQ-002#SEQ-C1]]

#### SessionLlm.generate_assessment 판정 답 만들기

**시그니처**
```python
def generate_assessment(slots: SlotState, chunks: list[dict[str, Any]], coverage: dict[str, Any] | None = None, notes: list[str] | None = None, on_delta: Callable[[str], None] | None = None) -> AssistantAssessment
```

**입력** 대화 정보, 검색한 청크, 판정 엔진 결과(dict), 대화 메모, 글 조각 콜백

**처리**
1. `if 청크가 없다 → LLMError`
2. 청크를 모델용으로 줄이고, 인용할 수 있는 청크 id 목록을 만든다
3. 대화 정보·청크·판정 결과(`coverage_decision`)·메모를 JSON으로 싣는다 · `if 보험사를 모른다 → 표준약관 모드 지시를 붙인다`
4. 최대 2번 시도한다. 1번째: `if on_delta → 스트림 호출(summary 칸을 조각으로 흘린다) · else → 구조화 호출`. 2번째: 재시도 지시를 붙인 구조화 호출
5. 결과를 조립한다. 면책 문구를 붙이고, 검색에 없던 chunk_id 인용을 버린다 · `if 인용이 하나도 없다 → SchemaViolationError(다음 시도로)`
6. 인용마다 원본 경로를 찾고(`DocumentsService.find_file_path_by_id`), 최대 4쪽에서 하이라이트가 가장 넓은 쪽을 골라(`PdfImageService.find_highlights`) 그 쪽을 그린다(`PdfImageService.render_page`). 보험사 이름은 청크의 보험사로 덮어쓴다
7. 본문의 내부 id와 인용 번호를 지우고 재청구 계획을 조립한다
8. `ReadinessCalculator.compute_readiness`로 준비도를 붙인다
9. `if 표준약관 모드이고 요약에 "표준약관"이 없다 → 표준약관 기준이라는 문장을 덧붙인다`

**출력** `AssistantAssessment`

**예외**

| 조건 | 에러 |
|---|---|
| 청크가 없다, 호출이 실패했다 | `LLMError` |
| 두 번 다 형식 위반이거나 인용이 모두 검색 밖 | `SchemaViolationError` |

**호출하는 것** [[#ReadinessCalculator.compute_readiness]] · [[#PdfImageService.find_highlights]] · [[#PdfImageService.render_page]]

**테스트 관점**
- 검색에 없는 chunk_id만 인용하면 한 번 더 요청하고, 그래도 같으면 `SchemaViolationError`다
- 1번째만 스트림이다. 흘려보낸 요약과 최종 요약이 다를 수 있다
- 하이라이트·페이지 이미지가 실패해도 인용은 남는다(이미지 없이)
- 등급은 판정 결과를 따르라는 프롬프트 지시뿐이다. 면책인데 '높음'이 나와도 코드가 막지 않는다

근거: [[INS-SEQ-002#SEQ-C3]] · [[INS-UC-002#UC-S4]] · [[INS-UC-002#UC-S6]]

#### SessionLlm.generate_explanation 설명 답 만들기

**시그니처**
```python
def generate_explanation(question: str, chunks: list[dict[str, Any]], history: list[Message] | None = None, notes: list[str] | None = None, on_delta: Callable[[str], None] | None = None) -> AssistantAnswer
```

**입력** 질문, 검색한 청크, 대화(최근 8개만 보낸다), 대화 메모, 글 조각 콜백

**처리**
1. `if 청크가 없다 → LLMError`
2. 질문·청크·메모를 JSON으로 싣는다
3. 최대 2번 시도한다. 1번째: `if on_delta → 스트림 호출(message 칸을 흘린다) · else → 구조화 호출`. 2번째: 재시도 지시를 붙인 구조화 호출
4. 검색에 없던 chunk_id 인용을 버린다 · `if 인용이 하나도 없다 → SchemaViolationError(다음 시도로)`
5. 인용에 원본 경로·하이라이트·페이지 이미지를 붙이고(`PdfImageService.find_highlights`·`PdfImageService.render_page`) 본문을 정리한다

**출력** `AssistantAnswer`

**예외**

| 조건 | 에러 |
|---|---|
| 청크가 없다, 호출이 실패했다 | `LLMError` |
| 두 번 다 형식 위반이거나 인용이 모두 검색 밖 | `SchemaViolationError` |

**호출하는 것** [[#PdfImageService.find_highlights]] · [[#PdfImageService.render_page]]

**테스트 관점** 인용 없는 설명 답은 나오지 않는다. 대화가 길어도 최근 8개만 모델에 간다

근거: [[INS-SEQ-002#SEQ-3]] · [[INS-UC-002#UC-H3]]

#### SessionLlm.generate_help_answer 도움 답 만들기

**시그니처**
```python
def generate_help_answer(question: str, chunks: list[dict[str, Any]] | None = None) -> HelpAnswer
```

**처리**
1. 질문과 청크로 구조화 호출을 한다(temperature 0.3) · `if 예외 → 일반 텍스트 호출로 답한다 · if 그것도 실패 → LLMError`
2. 검색에 있던 청크의 인용만 남겨 원본 캡처를 붙인다 · `if 캡처 실패 → 인용 없이`
3. 이어 물을 질문은 4개까지 남긴다

**호출하는 것** [[#PdfImageService.find_highlights]] · [[#PdfImageService.render_page]]

**테스트 관점** 청크가 없거나 구조화 호출이 실패해도 답이 나온다. 사용법 질문이면 인용이 비어 있다

근거: [[INS-SEQ-002#SEQ-8]]

#### SessionLlm.classify_document 서류 종류 가리기

**시그니처**
```python
def classify_document(text: str) -> dict[str, Any]
```

**처리**
1. `if 글자가 비었다 → other, 신뢰도 0`
2. 글자 앞 2,000자로 도구 호출을 한다(temperature 0)
3. `if 신뢰도 < 0.7 이고 종류가 other가 아니다 → other로 바꾼다`

**테스트 관점** 신뢰도가 낮으면 `other`다. 연결 오류는 `LLMError`로 감싸지 않고 올라간다(3장)

근거: [[INS-SEQ-002#SEQ-6]] · [[INS-UC-002#UC-S8]]

#### SessionLlm.extract_slots_from_document 서류에서 항목 뽑기

**시그니처**
```python
def extract_slots_from_document(text: str, doc_type: str) -> dict[str, Any]
```

**처리**
1. 서류 종류별로 뽑을 칸을 정한다 · `if 모르는 종류이거나 글자가 비었다 → 빈 결과`
2. 글자 앞 3,000자로 도구 호출을 한다(temperature 0). 영수증이면 청구 금액 규칙을 더한다
3. 빈 값을 버리고 돌려준다

**테스트 관점** 영수증은 가장 큰 금액 하나를 청구 금액으로 뽑는다. 연결 오류는 `LLMError`로 감싸지 않고 올라간다(3장)

근거: [[INS-SEQ-002#SEQ-6]] · [[INS-UC-002#UC-S8]]

#### ReadinessCalculator.compute_readiness 준비도 점수

**시그니처**
```python
def compute_readiness(likelihood: str, satisfied: list[str], unsatisfied: list[str], confidence: str) -> ReadinessScore
```

**처리**
1. 등급 점수: 높음 55 · 중간 35 · 낮음 15
2. 요건 점수: `if 충족·미충족이 있다 → 30 × 충족 수 / 전체 · else → 15`
3. 정보 점수: `if confidence = full → 15 · else → 5`
4. 합을 0~100으로 자르고 `if ≥ 70 → high · if ≥ 40 → medium · else → low`

**테스트 관점** 같은 입력이면 같은 점수다. 높음·충족 전부·full이면 100이다

근거: [[INS-SEQ-002#SEQ-C3]] · [[INS-UC-002#UC-S5]]

### 2.3 판정 엔진

#### CoverageEngine.build_facts_from_slots 대화 정보를 청구 사실로

**시그니처**
```python
def build_facts_from_slots(slots: SlotState, *, generation: int | None = None, purpose: ClaimPurpose | None = None) -> ClaimFacts
```

**처리**
1. 치료 유형: `if 입원 일수 > 0 → 입원 · if 통원 횟수 > 0 → 통원 · else → None`
2. 목적: `if 비었거나 모르는 값 → 치료 · else → 그 값`
3. 청구 금액은 `charged_amount`로, 네 가지 정황(해외·한방·치과 질병·다른 보험 처리분)은 그대로 옮긴다. 세대·목적은 인자로 덮어쓸 수 있다

**테스트 관점** 급여·비급여 금액과 계약 기간은 채우지 않는다([[INS-DOM-005]] 5장)

근거: [[INS-SEQ-002#SEQ-C1]]

#### CoverageEngine.evaluate 결정론 판정

**시그니처**
```python
def evaluate(facts: ClaimFacts) -> CoverageAssessment
```

**입력** 청구 사실

**처리**
1. `CoverageEngine.rules_for(facts)`로 적용할 규칙을 우선순위 순으로 고른다
2. `if 보장기간 규칙이 맞는다 → excluded`
3. `if 면책 규칙이 맞는다 → excluded`
4. `if 부분 보상 규칙이 맞는다 → conditional`
5. `if 보장 근거 규칙이 맞는다 → covered` · 자기부담을 계산한다(`CoverageEngine.compute_deductible`) · `if 계산하지 못했다 → missing에 benefit_split` · `if 계약 기간이 없다 → 이유 문장을 더한다`
6. `if 치료 유형을 모른다 → insufficient_info`
7. `else → conditional`(특약·한도 확인 필요)
8. 세대를 몰랐으면 `needs_generation`을 켠다

**출력** `CoverageAssessment`

**예외** 없음

**호출하는 것** [[#CoverageEngine.rules_for]] · [[#CoverageEngine.compute_deductible]]

**테스트 관점**
- 규칙 11개가 저마다 한 번씩 맞는 사실 조합에서 기대 결과가 나온다
- 면책과 부분 보상이 같이 맞으면 면책이 이긴다
- 치료 유형이 없으면 `insufficient_info`다
- LLM도 DB도 부르지 않는다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-UC-002#UC-S3]]

#### CoverageEngine.rules_for 적용할 규칙 고르기

**시그니처**
```python
def rules_for(facts: ClaimFacts) -> list[CoverageRule]
```

**처리**
1. 규칙마다 범위를 본다. `if 영역·세대·보험사 목록이 있고 사실이 그 밖이다 → 제외` · `if 사고일이 있고 적용 기간 밖이다 → 제외`
2. 남은 규칙을 우선순위 순으로 돌려준다

**테스트 관점** 세대를 모르면 4세대로 보고 범위를 따진다

근거: [[INS-SEQ-002#SEQ-C1]]

#### CoverageEngine.compute_deductible 자기부담 계산

**시그니처**
```python
def compute_deductible(facts: ClaimFacts) -> DeductibleBreakdown | None
```

**입력** 청구 사실(급여·비급여 금액, 청구 금액, 세대, 치료 유형)

**처리**
1. `if 급여·비급여 금액이 둘 다 없다 → None`
2. 없는 쪽은 0으로 둔다. 총액은 `if 청구 금액이 있다 → 그 값 · else → 급여 + 비급여`
3. 세대별 비율(없으면 4세대)로 `급여 × 급여율 + 비급여 × 비급여율`을 구한다
4. `if 통원 → 통원 최소 공제액과 비교해 큰 값`
5. 공제액을 총액으로 자르고 `지급 추정 = 총액 − 공제액`

**출력** `DeductibleBreakdown`(청구액·공제액·지급 추정·계산식) 또는 None

**예외** 없음

**테스트 관점**
- 4세대 급여 100,000·비급여 100,000이면 공제 50,000, 지급 추정 150,000이다
- 통원이고 비율 공제가 최소 공제보다 작으면 최소 공제가 쓰인다
- 공제가 총액을 넘지 않는다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-UC-002#UC-S3]]

#### ProrationCalculator.compute 추천과 비례 안분

**시그니처**
```python
def compute(items: list[tuple[str, int | None, CoverageAssessment]]) -> ProrationResult
```

**입력** 보험마다 (증권 번호, 세대, 판정)

**처리**
1. 보험마다 세대별 자기부담률과 지급 추정액(판정의 자기부담 계산 결과)을 적는다
2. 안분 대상: 판정이 `covered`이고 지급 추정액이 0보다 큰 보험
3. `if 대상이 2개 이상 → 실제 지급 총액 = 가장 큰 추정액, 보험마다 총액 × 자기 추정액 / 추정액 합`
4. 추천: `if 보장되는 보험이 있다 → 그 가운데 비급여 자기부담률이 가장 낮은 보험 · else → 없음`
5. 요약 문장을 만든다

**출력** `ProrationResult`(보험별 값, 요약, 추천 증권 번호)

**예외** 없음

**테스트 관점**
- 지급 추정액이 없으면 안분액이 모두 비어 있다(지금 대화 흐름이 그렇다)
- 추천은 금액이 없어도 나온다
- 세대를 모르는 보험은 4세대 비율로 본다

근거: [[INS-SEQ-002#SEQ-5]] · [[INS-UC-002#UC-H5]]

### 2.4 조항 검색

#### RagService.retrieve 대화 정보로 조항 찾기

**시그니처**
```python
def retrieve(slots: SlotState, top_k: int = 8) -> list[dict[str, Any]]
```

**입력** 대화 정보, 가져올 수

**처리**
1. `if RAG_RERANK → top_k의 두 배를 가져온다 · else → top_k`
2. 서킷 브레이커(5번 연속 실패 시 60초 열림) 안에서 `NeuroSymbolicRetriever.retrieve`를 부른다 · `if 브레이커가 열렸거나 예외 → 빈 목록`
3. `if RAG_RERANK이고 2개 이상 → Solar로 순위를 다시 매긴다`
4. 앞의 top_k개를 돌려준다

**출력** 청크 목록(id·원문·점수·메타·출처 뉴럴/심볼릭)

**예외** 없음 — 실패는 빈 목록이다

**호출하는 것** [[#NeuroSymbolicRetriever.retrieve]]

**테스트 관점** 저장소가 멈추면 빈 목록이고, 한 턴 처리가 되묻기로 간다. 5번 실패 뒤에는 60초 동안 부르지도 않는다

근거: [[INS-SEQ-002#SEQ-C2]] · [[INS-UC-002#UC-S2]]

#### RagService.retrieve_freeform 자유 질의로 조항 찾기

**시그니처**
```python
def retrieve_freeform(text: str, top_k: int = 8, insurer_id: str | None = None) -> list[dict[str, Any]]
```

**처리**
1. 서킷 브레이커 안에서 `NeuroSymbolicRetriever.retrieve_freeform(text, top_k, insurer_id)` · `if 실패 → 빈 목록`

**호출하는 것** [[#NeuroSymbolicRetriever.retrieve_freeform]]

**테스트 관점** 보험사를 주면 그 보험사 약관만 나온다

근거: [[INS-SEQ-002#SEQ-3]] · [[INS-SEQ-002#SEQ-8]]

#### NeuroSymbolicRetriever.retrieve 대화 정보를 질의와 필터로

**시그니처**
```python
def retrieve(self, slots: SlotState, top_k: int = 8) -> list[RetrievalResult]
```

**처리**
1. 대화 정보로 질의 문장과 필터(영역·보험사 코드)를 만든다 · `if 보험사 이름이 알려진 다섯 곳이 아니다 → 보험사 필터 없음`
2. `NeuroSymbolicRetriever.retrieve_fused(질의, 보험사 코드, 필터, top_k)`

**호출하는 것** [[#NeuroSymbolicRetriever.retrieve_fused]]

**테스트 관점** 보험사 코드가 있으면 이름보다 코드를 쓴다

근거: [[INS-SEQ-002#SEQ-C2]]

#### NeuroSymbolicRetriever.retrieve_freeform 자유 질의를 필터로

**시그니처**
```python
def retrieve_freeform(self, text: str, top_k: int = 8, insurer_id: str | None = None) -> list[RetrievalResult]
```

**처리**
1. `if 보험사 코드가 있다 → 보험사 필터 · else → 필터 없음`
2. `NeuroSymbolicRetriever.retrieve_fused(text, 보험사 코드, 필터, top_k)`

**호출하는 것** [[#NeuroSymbolicRetriever.retrieve_fused]]

**테스트 관점** 필터가 없으면 다섯 보험사를 교차 검색한다

근거: [[INS-SEQ-002#SEQ-C2]]

#### NeuroSymbolicRetriever.retrieve_fused 뉴럴·심볼릭 융합

**시그니처**
```python
def retrieve_fused(self, query: str, insurer_id: str | None, filters: dict[str, Any] | None, top_k: int) -> list[RetrievalResult]
```

**입력** 질의, 보험사 코드, 필터, 가져올 수

**처리**
1. 뉴럴: `VectorStoreAdapter.query(질의, top_k × 2, 필터)` · `if RAG_SCORE_RATIO > 0 → 1등 점수 × 비율보다 낮은 것을 버린다(하나는 남긴다)`
2. `if RAG_GRAPH_ENABLED → 심볼릭` : `SymbolicGraphChannel.clause_candidates(질의, 보험사, top_k)`와 `SymbolicGraphChannel.expand(뉴럴 상위 top_k개, top_k × 2)` · `if 예외 → 뉴럴만`
3. 뉴럴에 없던 심볼릭 후보의 본문과 메타를 PostgreSQL에서 읽는다 · `if 보험사를 정했는데 다른 보험사 청크다 → 버린다`
4. 가중 RRF(k 60)로 합친다. 뉴럴 가중 1.0, 심볼릭 순위마다 `RAG_SYMBOLIC_WEIGHT`(기본 0.02)
5. 점수 순으로 top_k개를 돌려준다. 출처(`neural`·`symbolic`)를 붙인다

**출력** 청크 목록

**예외** 뉴럴 쪽 예외는 그대로 올라간다(부르는 쪽 브레이커가 받는다)

**호출하는 것** [[#VectorStoreAdapter.query]] · [[#SymbolicGraphChannel.clause_candidates]] · [[#SymbolicGraphChannel.expand]]

**테스트 관점**
- 그래프 저장소가 멈춰도 뉴럴 결과가 나온다
- 두 채널에 모두 잡힌 청크가 위로 올라간다
- 보험사를 정하면 다른 보험사의 심볼릭 후보가 섞이지 않는다

근거: [[INS-SEQ-002#SEQ-C2]] · [[INS-UC-002#UC-S2]]

#### VectorStoreAdapter.query pgvector 검색

**시그니처**
```python
def query(self, query_text: str, top_k: int = 8, filters: dict[str, Any] | None = None) -> list[dict[str, Any]]
```

**입력** 질의, 가져올 수, 필터(영역·보험사·상품·버전·문서 유형·문서)

**처리**
1. `if 질의가 비었다 → 빈 목록`
2. `EmbeddingService.embed_texts([질의], role=query)`
3. 조건을 만든다: 임베딩이 있는 청크만, 필터 키는 청크의 복사 컬럼에 건다 · `if 모르는 키 → 버린다`
4. 문서·버전·상품·보험사를 조인해 코사인 거리 순으로 top_k개를 읽는다. 점수는 `1 − 거리`
5. 메타(문서·보험사·상품·버전·조 번호·페이지 등)를 붙여 돌려준다

**출력** 청크 목록(점수 포함)

**예외**

| 조건 | 에러 |
|---|---|
| 질의 임베딩 실패 | `EmbeddingError` |
| DB 오류 | SQLAlchemy 예외 |

**호출하는 것** [[#EmbeddingService.embed_texts]]

**테스트 관점** 임베딩이 없는 청크는 결과에 나오지 않는다. 필터가 없으면 다섯 보험사를 모두 본다. 인덱스가 없어 거른 행 전부와 거리를 잰다

근거: [[INS-SEQ-002#SEQ-C2]] · [[INS-DOM-006#clause_chunks]]

#### SymbolicGraphChannel.clause_candidates 조항 제목 후보

**시그니처**
```python
def clause_candidates(self, query: str, insurer_id: str | None, limit: int = 12) -> list[dict[str, Any]]
```

**처리**
1. Memgraph에서 `Clause` 노드를 읽는다 · `if 보험사 코드가 있다 → 그 보험사만`
2. 조항 제목의 낱말 가운데 질의에 들어 있는 것의 수를 점수로 삼는다 · `if 점수 0 → 버린다`
3. 점수 순으로 `limit`개

**테스트 관점** 모델을 부르지 않는다. 같은 질의면 같은 후보다

근거: [[INS-SEQ-002#SEQ-C2]]

#### SymbolicGraphChannel.expand 구조 이웃 확장

**시그니처**
```python
def expand(self, chunk_ids: list[str], limit: int = 16) -> list[dict[str, Any]]
```

**처리**
1. `if 입력이 비었다 → 빈 목록`
2. 1차: 입력 청크가 `REFERS_TO`로 가리키는 별표·붙임 청크
3. 2차: 같은 문서·같은 조 번호의 다른 청크(관계가 아니라 속성으로 찾는다)
4. 입력 순위 순으로 겹치지 않게 모으고 `limit`개에서 멈춘다

**테스트 관점** 입력 청크 자신은 나오지 않는다. 별표 참조가 형제보다 먼저다

근거: [[INS-SEQ-002#SEQ-C2]]

#### EmbeddingService.embed_texts 임베딩

**시그니처**
```python
def embed_texts(texts: list[str], *, role: EmbeddingRole = 'passage') -> list[list[float]]
```

**처리**
1. `if 입력이 비었다 → 빈 목록`
2. `role`로 모델을 고른다(`passage`·`query`). 글마다 3,800토큰에서 자른다
3. 100개씩 묶어 부른다 · `if 호출 실패 또는 개수가 다르다 → EmbeddingError`

**테스트 관점** 4096차원 벡터가 입력 순서대로 나온다. 긴 글은 잘려도 실패하지 않는다

근거: [[INS-SEQ-002#SEQ-10]] · [[INS-SEQ-002#SEQ-C2]]

### 2.5 원본 캡처

#### PdfImageService.find_highlights 하이라이트 상자

**시그니처**
```python
def find_highlights(pdf_path: str | Path, page_no: int, text: str, clause_no: str | None = None) -> list[dict[str, float]]
```

**입력** 원본 PDF, 쪽 번호, 인용 원문, 조 번호

**처리**
1. `if 파일이 없거나 원문이 비었거나 쪽이 범위 밖 → 빈 목록`
2. 원문을 공백 없이 이어 붙인다 · `if 6자보다 짧다 → 빈 목록`
3. `if 조 번호가 있다 → 그 글자의 첫 위치를 기준점으로 삼는다` · 세로 페이지는 기준점 위 40 ~ 아래 520 안의 글자만, 가로 2단 페이지는 기준점과 같은 단의 아래쪽(왼쪽 단이면 오른쪽 단도)만 본다
4. 표 셀 안 글자는 셀별로, 나머지는 줄(세로 4pt 단위)·단별로 모은다. 두 글자 이상 낱말이 원문에 들어 있으면 맞은 글자로 센다
5. 줄: 맞은 비율 0.5 이상을 씨앗으로, 씨앗 사이의 2줄 이하 틈은 이어 붙인다 · `if 3줄 이상 이어진 덩어리가 있다 → 1줄짜리 덩어리는 버린다`
6. 셀: 맞은 글자 8자 이상, 비율 0.75 이상만
7. 상자를 페이지 크기 비율(0~1)로 바꿔 60개까지 돌려준다

**출력** 상자 목록(x·y·w·h)

**예외** 없음 — PDF 오류는 빈 목록이다

**테스트 관점**
- 인용 원문이 있는 쪽에서는 이어진 블록이 나오고, 다른 쪽에서는 빈 목록이다
- 가로 2단 페이지에서 반대 단의 같은 글자를 칠하지 않는다
- 표 안 인용은 셀 경계를 넘지 않는다

근거: [[INS-SEQ-002#SEQ-C3]] · [[INS-UC-002#UC-S6]]

#### PdfImageService.render_page 페이지 이미지

**시그니처**
```python
def render_page(document_id: int, page_no: int, file_path: str | Path, *, scale: float = 1.5) -> Path
```

**처리**
1. 저장 경로는 `page_images/<문서 id>/<쪽 4자리>.png`다 · `if 이미 있다 → 그대로 돌려준다`
2. `if 원본이 없다 → FileNotFoundError` · `if 쪽이 범위 밖 → ValueError`
3. 1.5배로 그려 저장한다

**테스트 관점** 두 번째 부르면 다시 그리지 않는다

근거: [[INS-SEQ-002#SEQ-C3]] · [[INS-SEQ-002#SEQ-2]]

### 2.6 청구 준비

#### ClaimsService.build_checklist 필요 서류

**시그니처**
```python
def build_checklist(slots: SlotState) -> ClaimChecklist
```

**처리**
1. 공통 3개(보험금 청구서·신분증 사본·통장 사본)를 넣는다
2. `if 영역이 실손 → 진단서·진료비 영수증과 세부내역서`
3. `if 입원 일수 > 0 → 입·퇴원 확인서`

**테스트 관점** 지금은 모든 서류가 필수다. 선택 서류가 하나도 없다(3장)

근거: [[INS-SEQ-002#SEQ-7]] · [[INS-API-002#GET/api/v1/sessions/{session_id}/checklist]]

#### ClaimsService.build_summary 청구 준비 요약

**시그니처**
```python
def build_summary(session: Session) -> ClaimSummary
```

**처리**
1. 대화 정보의 보험사·상품·영역을 옮긴다
2. `if 마지막 판정 답이 있다 → 가능성·요약·충족·미충족·다음 행동을 옮긴다 · else → 비워 둔다`
3. `ClaimsService.build_checklist`로 필요 서류를 붙인다

**호출하는 것** [[#ClaimsService.build_checklist]]

**테스트 관점** 판정 전에 부르면 가능성과 요약이 비어 있다

근거: [[INS-SEQ-002#SEQ-7]] · [[INS-API-002#GET/api/v1/sessions/{session_id}/summary]]

#### ClaimsService.submit_claim 가정 접수

**시그니처**
```python
def submit_claim(session: Session) -> ClaimReceipt
```

**처리**
1. 접수 번호 `CLM-<UTC 날짜>-<세션 id 앞 6자 대문자>`를 만든다
2. 상태 `접수완료`, 예상 5일, "데모 — 실제 전송 아님" 안내를 담아 돌려준다

**테스트 관점** 아무 데도 보내지 않는다. 같은 날 같은 세션이면 같은 번호다

근거: [[INS-SEQ-002#SEQ-7]] · [[INS-API-002#POST/api/v1/sessions/{session_id}/submit]]

### 2.7 서류

#### attachments_router.upload_document 서류 업로드 절차

**시그니처**
```python
def upload_document(session_id: str, file: UploadFile = File(...), current_user: User | None = Depends(get_current_user_optional)) -> dict[str, Any]
```

**입력** 세션 id, 올린 파일, 로그인 사용자(선택)

**처리**
1. `SessionStore.get` · `if 없음 → 404 SESSION_NOT_FOUND`
2. 파일을 읽는다 · `if 실패 → 400 FILE_READ_ERROR`
3. `AttachmentsService.save_bytes` · `if 형식·크기 위반 → 400 INVALID_FILE · if 저장 실패 → 500 STORAGE_ERROR`
4. OCR로 글자를 뽑는다(`OcrAdapter.extract_text`) · `if 설정 없음 → 503 OCR_NOT_CONFIGURED · if 호출 실패 → 502 OCR_FAILED`
5. 글자를 가린다(`PiiMasker.mask_pii`)
6. `SessionLlm.classify_document(가린 글자)`
7. `if IE 스키마가 있는 종류 → Upstage IE로 원본 이미지에서 항목을 뽑는다 · if 실패 → 다음으로`
8. `if 뽑은 항목이 없다 → SessionLlm.extract_slots_from_document(가린 글자, 종류)`
9. `if 6~8에서 LLMError → 빈 분류·빈 항목으로 둔다`
10. `if 뽑은 항목이 있고 OCR 신뢰도가 0.6 이상 → SessionService.seed_slots(세션 id, 항목)로 대화 정보에 병합(모델 없음)` · `if 값이 SlotState 검증을 못 넘는다 → 병합하지 않고 경고만` · 저신뢰면 병합하지 않는다
11. 첨부 메타·종류·신뢰도·뽑은 항목·OCR 신뢰도·저신뢰(0.6 미만)·병합 여부(`applied`)·병합 뒤 대화 정보와 빠진 칸을 돌려준다

**출력** JSON(`attachment`, `doc_type`, `doc_type_confidence`, `extracted_slots`, `ocr_confidence`, `low_confidence`, `applied`, `slots`, `missing`)

**예외**

| 조건 | 에러 |
|---|---|
| 세션 없음 | `SESSION_NOT_FOUND`(404) |
| 파일을 읽지 못했다 | `FILE_READ_ERROR`(400) |
| 형식·크기 위반 | `INVALID_FILE`(400) |
| 저장 실패 | `STORAGE_ERROR`(500) |
| OCR 설정 없음·호출 실패 | `OCR_NOT_CONFIGURED`(503)·`OCR_FAILED`(502) |

**호출하는 것** [[#SessionStore.get]] · [[#AttachmentsService.save_bytes]] · [[#PiiMasker.mask_pii]] · [[#SessionLlm.classify_document]] · [[#SessionLlm.extract_slots_from_document]] · [[#SessionService.seed_slots]]

**테스트 관점**
- 뽑은 항목이 세션의 대화 정보에 들어가고, 세션 조회에 보인다
- 저신뢰 OCR(0.6 미만)과 형식이 틀린 값(예: 입원 일수가 글자)은 대화 정보에 들어가지 않고 응답에만 남는다
- IE가 실패해도 모델 추출로 항목이 나온다
- 분류 중 연결 오류가 3번 이어지면 500이 된다. 파일은 이미 저장돼 있다(3장)

근거: [[INS-SEQ-002#SEQ-6]] · [[INS-API-002#POST/api/v1/sessions/{session_id}/documents]]

#### AttachmentsService.save_bytes 첨부 저장

**시그니처**
```python
def save_bytes(session_id: str, filename: str, mime_type: str, content: bytes) -> AttachmentMeta
```

**처리**
1. `if 형식이 JPEG·PNG·WebP가 아니다 → DomainError` · `if 크기가 10MB를 넘는다 → DomainError`
2. `<첨부 폴더>/<세션 id>/<uuid><확장자>`로 쓴다 · `if 쓰기 실패 → StorageError`
3. SHA-256, 크기, 만든 시각, 만료 시각(지금 + 24시간)을 담은 메타를 돌려준다

**테스트 관점** PDF는 거절한다. 메타는 DB에 남지 않는다

근거: [[INS-SEQ-002#SEQ-6]]

#### AttachmentsService.cleanup_expired 만료 첨부 지우기

**시그니처**
```python
def cleanup_expired() -> int
```

**처리**
1. `if TTL ≤ 0 이거나 첨부 폴더가 없다 → 0`
2. 세션 폴더마다 수정 시각이 TTL보다 오래된 파일을 지운다 · `if 지우기 실패 → 경고만`
3. 빈 세션 폴더를 지우고 지운 수를 돌려준다

**테스트 관점** 앱이 시작할 때 등록한 스케줄러가 1시간마다 부른다

근거: [[INS-UC-002#UC-S9]]

### 2.8 약관 적재

#### IngestionService.run_ingest 약관 적재

**시그니처**
```python
def run_ingest(items: Iterable[PathInfo], *, dry_run: bool = False, force: bool = False) -> IngestStats
```

**입력** 경로에서 읽은 약관 목록, 시험 실행 여부, 강제 여부

**처리**
1. `if dry_run → 처리 수만 센다`
2. 문서마다: 파일 해시를 구한다 · `if 같은 해시 문서가 있고 force가 아니다 → 건너뜀`
3. `ChunksService.process_pdf`로 청크를 만든다
4. 한 트랜잭션에서 `DocumentsService.register_document` → `if 바뀐 게 없고 force가 아니다 → 건너뜀` → 청크마다 문서 id와 복사 메타(보험사·상품·영역·문서 유형)를 채워 문서 단위로 교체한다
5. 커밋 뒤 벡터 저장소에서 그 문서를 지우고(실패는 무시), `EmbeddingService.embed_texts`로 임베딩해 넣는다
6. `GraphIndexer.sync_document` · `if 실패 → 경고만`
7. 문서별 예외는 `failed`로 세고 다음 문서로 간다

**출력** `IngestStats`(처리·건너뜀·실패 수)

**예외** 없음 — 문서별 실패는 수로 센다

**호출하는 것** [[#ChunksService.process_pdf]] · [[#DocumentsService.register_document]] · [[#EmbeddingService.embed_texts]] · [[#GraphIndexer.sync_document]]

**테스트 관점**
- 같은 파일을 두 번 적재하면 두 번째는 건너뛴다
- 임베딩이 실패하면 청크는 남고 임베딩은 없다. 다음 적재도 해시 때문에 건너뛴다(3장)
- 그래프가 멈춰도 적재는 성공으로 센다

근거: [[INS-SEQ-002#SEQ-10]] · [[INS-UC-002#UC-A2]]

#### IngestionService.parse_pdf_path 약관 경로를 메타로

**시그니처**
```python
def parse_pdf_path(pdf_path: Path, raw_root: Path) -> PathInfo
```

**처리**
1. `if 경로가 <보험사>/<영역>/<상품>/<버전>/<파일> 다섯 단계가 아니다 → IngestionError`
2. `if 영역이 허용 목록 밖 → IngestionError`
3. 버전 이름에서 적용 기간을 읽는다(`YYYY-MM-DD_present`·`YYYY-MM-DD_YYYY-MM-DD`·`YYYYMMDD`) · `if 읽지 못했다 → 1900-01-01 시작`
4. 파일 이름의 낱말로 문서 유형을 정한다 · `if 못 정했다 → IngestionError`
5. 보험사·상품의 이름 자리에는 코드를 넣는다

**테스트 관점** 버전 이름이 틀려도 적재는 되고 시작일이 1900-01-01이다

근거: [[INS-SEQ-002#SEQ-10]]

#### ChunksService.process_pdf PDF를 청크로

**시그니처**
```python
def process_pdf(pdf_path: Path | str, *, max_tokens: int = DEFAULT_MAX_TOKENS) -> ProcessedPdf
```

**처리**
1. 파싱: `if TERMS_PARSER = upstage → Document Parse를 페이지 묶음으로 부르고 header·footer·index 요소를 버린다 · else(pymupdf) → 로컬에서 읽고 쪽마다 되풀이되는 줄을 머리말·꼬리말로 보고 뺀다`
2. 구조 인식: 조·항·호·별표 경계를 찾는다
3. 자르기: 1,000토큰을 넘으면 항 단위로 나누고, 3,500토큰에서 강제로 자른다
4. 청크·쪽수·파서 버전을 돌려준다

**테스트 관점** 어떤 청크도 3,500토큰을 넘지 않는다

근거: [[INS-SEQ-002#SEQ-10]] · [[INS-UC-002#UC-A2]]

#### DocumentsService.register_document 약관 메타 등록

**시그니처**
```python
def register_document(session: Session, *, insurer_id: str, insurer_name: str, area: str, product_id: str, product_name: str, valid_from: date, valid_to: date | None, version_label: str, doc_type: str, file_path: str, file_sha256: str, page_count: int, parser_version: str) -> tuple[int, int, bool]
```

**처리**
1. 보험사·상품을 없으면 만든다. 있으면 이름을 고치지 않는다
2. 상품·시작일로 버전을 찾거나 만든다
3. 버전·문서 유형으로 문서를 넣거나 고친다

**테스트 관점** (문서 id, 버전 id, 바뀌었는지)를 돌려준다. 트랜잭션은 부르는 쪽이 연다

근거: [[INS-SEQ-002#SEQ-10]] · [[INS-DOM-006#documents]]

#### GraphIndexer.sync_document 문서 하나를 그래프에 맞추기

**시그니처**
```python
def sync_document(document_id: int) -> dict[str, int]
```

**처리**
1. 제약·인덱스를 적용하고 그 문서의 `Clause`·`SubClause` 노드를 지운다
2. 그 문서의 보험사·상품·버전·문서 노드와 관계를 MERGE한다
3. 청크를 `Clause`(조)·`SubClause`(나머지)로 넣고 `CONTAINS`·`HAS_SUBCLAUSE`·`REFERS_TO`를 만든다

**테스트 관점** 두 번 돌려도 노드 수가 같다(멱등)

근거: [[INS-SEQ-002#SEQ-10]] · [[INS-DOM-006]] 4장

### 2.9 로그인과 외부 데이터

#### DemoPersonaRegistry.find_demo_user 시연 계정 찾기

**시그니처**
```python
def find_demo_user(session: Session, name: str, phone: str) -> User | None
```

**처리**
1. 이름(앞뒤 공백을 뗀 것)과 휴대폰 번호(숫자만)로 합성 페르소나를 찾는다 · `if 없음 → None`
2. `demo-<외부 키>@example.com` 사용자를 찾는다 · `if 없다 → 공용 비밀번호 해시로 만든다 · if 외부 키가 다르다 → 고친다`

**테스트 관점** 시드가 없어도 로그인할 때 사용자가 생긴다

근거: [[INS-SEQ-002#SEQ-4]] · [[INS-API-002#POST/api/v1/auth/demo-login]]

#### TokenService.get_current_user_optional 로그인 사용자 확인

**시그니처**
```python
def get_current_user_optional(access_token_cookie: str | None = Cookie(default=None, alias=COOKIE_NAME), authorization: str | None = Header(default=None), session: Session = Depends(get_db_session)) -> User | None
```

**처리**
1. 쿠키 `access_token`을 먼저, 없으면 `Authorization: Bearer`를 본다 · `if 없다 → None`
2. 토큰을 풀어 사용자 id를 얻는다 · `if 틀렸거나 만료 → None`
3. 사용자를 읽어 돌려준다

**테스트 관점** 토큰이 틀려도 예외가 아니라 None이다

근거: [[INS-SEQ-002#SEQ-4]]

#### MydataAdapter.fetch_insurances 가입 보험 가져오기

**시그니처**
```python
def fetch_insurances(self, user_external_id: str) -> list[InsuranceDict]
```

**처리**
1. `if MYDATA_BACKEND = real → 실연동` · `else → 더미`
2. 더미: `data/demo/mydata.json`에서 외부 키의 레코드를 그대로 돌려준다(세대가 이미 들어 있다)
3. 실연동: `if 주소나 토큰이 없다 → MydataNotConfiguredError` · 표준 API로 목록과 기본 정보를 읽고, 가입일로 세대를 정하며, 실손이 아니거나 정상 계약이 아니면 뺀다

**테스트 관점** 더미에는 실손만 들어 있다. 외부 키가 없으면 빈 목록이다

근거: [[INS-SEQ-002#SEQ-4]] · [[INS-UC-002#UC-S7]]

### 2.10 감사 기록

#### AuditService.begin 감사 기록 열기

**시그니처**
```python
def begin(*, session_id: str | None = None, turn: int | None = None, raw_user_input: str | None = None, user_id: int | None = None) -> AuditContext
```

**처리**
1. 응답 id(uuid4 16진수)를 만들고, 입력을 `PiiMasker.mask_pii`로 가려 컨텍스트에 담는다

**호출하는 것** [[#PiiMasker.mask_pii]]

**테스트 관점** DB에 쓰지 않는다. 원문은 컨텍스트에 남지 않는다

근거: [[INS-SEQ-002#SEQ-C1]] · [[INS-UC-002#UC-S9]]

#### AuditService.complete 감사 기록 성공

**시그니처**
```python
def complete(ctx: AuditContext, *, assistant_response_type: str, assistant_message: str | None = None, confidence: str | None = None) -> None
```

**처리**
1. `if 감사 기록이 꺼져 있다 → 아무것도 안 한다`
2. 응답 본문의 SHA-256을 구하고, 도구·외부 호출 기록의 문자열을 가린 뒤(`PiiMasker.mask_pii`) 한 행을 넣는다 · `if DB 오류 → 경고만`

**호출하는 것** [[#PiiMasker.mask_pii]]

**테스트 관점** DB가 멈춰도 응답은 나간다. 본문은 남지 않는다

근거: [[INS-SEQ-002#SEQ-C1]]

#### AuditService.fail 감사 기록 실패

**시그니처**
```python
def fail(ctx: AuditContext, *, error: str) -> None
```

**처리**
1. `if 감사 기록이 꺼져 있다 → 아무것도 안 한다`
2. 호출 기록을 가리고(`PiiMasker.mask_pii`) 에러를 담아 한 행을 넣는다 · `if DB 오류 → 경고만`

**호출하는 것** [[#PiiMasker.mask_pii]]

**테스트 관점** 에러 문장은 부르는 쪽이 이미 가려서 넘긴다

근거: [[INS-SEQ-002#SEQ-C1]]

#### PiiMasker.mask_pii 개인정보 가림

**시그니처**
```python
def mask_pii(text: str) -> str
```

**처리**
1. `if 비었다 → 그대로`
2. 주민등록번호 → 이메일 → 휴대폰 → 계좌 → 일반 전화 순으로 `[RRN]`·`[EMAIL]`·`[PHONE]`·`[ACCOUNT]`·`[TEL]`로 바꾼다

**테스트 관점** 휴대폰 번호가 일반 전화로 가려지지 않는다(순서). 진단명은 가리지 않는다

근거: [[INS-UC-002#UC-S9]]

### 2.11 관리자

#### AdminGraphService.fetch_graph 그래프 범위 읽기

**시그니처**
```python
def fetch_graph(insurer_id: str | None = None, scope: str | None = None) -> dict[str, Any]
```

**처리**
1. 그래프 전체를 읽는다(60초 캐시) · `if 연결 실패 → RuntimeError`
2. 기준점 = `scope` 또는 `insurer:<코드>` · `if 기준점이 root이거나 없다 → 전체`
3. `if 구간 기준점 → 구간 구성원마다 BFS로 닿는 노드의 합 · else → 기준점에서 BFS로 닿는 노드`
4. 남은 노드 사이의 관계만 남기고 수를 붙여 돌려준다

**테스트 관점** 없는 기준점이면 빈 그래프다

근거: [[INS-SEQ-002#SEQ-9]] · [[INS-API-002#GET/api/v1/admin/graph]]

## 3. 미결사항

- [ ] **구조화 LLM 호출에 재시도가 없다** — 재시도 데코레이터(2번)가 LLM 호출 함수가 아니라 JSON 조각 파서(`_partial_json_string`)에 붙어 있다. 그래서 판정·설명·도움 답의 구조화 호출은 SDK 자체 재시도(2번)만 받는다. 도구 호출(사실 추출·되묻기·의도·서류)은 3번까지 다시 시도한다. 데코레이터를 옮길지
- [ ] **서류 분류의 연결 오류가 500이 된다** — [[#SessionLlm.classify_document]]·[[#SessionLlm.extract_slots_from_document]]는 SDK 예외를 `LLMError`로 감싸지 않는다. 업로드 라우터는 `LLMError`만 받으므로 연결 오류가 3번 이어지면 500이 되고, 파일은 이미 저장돼 있다. 감쌀지
- [ ] **선택 서류가 없다** — [[#ClaimsService.build_checklist]]의 서류가 모두 필수다. 화면의 "선택" 표시가 나올 일이 없다
- [ ] **임베딩 실패 뒤 건너뛰기** — [[#IngestionService.run_ingest]]는 청크를 커밋한 뒤 임베딩한다. 임베딩이 실패하면 다음 적재가 해시 때문에 그 문서를 건너뛰고, [[#VectorStoreAdapter.query]]는 임베딩 없는 청크를 보지 않는다([[INS-SEQ-002]] 3장과 같은 항목)

---
doc_id: INS-MS-001
type: MS
title: MINISPEC — 보험청구심사 어시스턴트
status: draft
upstream: [INS-SEQ-001, INS-DOM-002]
---

# MINISPEC: 보험청구심사 어시스턴트

## 1. 함수 목록

#### SessionService.post_message 한 턴 처리

**시그니처** `async def post_message(session_id: str, text: str, user: User | None = None) -> SessionResponse`

근거: [[INS-SEQ-001#SEQ-1]] · [[INS-UC-001#UC-H4]]

**처리**
1. `session = store.get(session_id)` · 없으면 `! session-not-found`
2. 감사 시작 · 이력에 사용자 발화 추가
3. 잡담 가드 (정규식) → 해당하면 LLM 없이 환영 응답
4. `intent = llm.classify_intent(text)` → 일반 질의면 `answer_general(...)`, 도메인 밖이면 안내
5. `slots = merge(session.slots, llm.extract_slots(history, text, session.slots))`
6. `missing = compute_missing(slots)` — **코드가 판정한다. LLM이 아니다**
7. `cov = coverage.evaluate(facts_from(slots))`
8. 분기
   - `cov`가 면책 → 되묻기 건너뛰고 알림
   - `missing` 있고 되묻기 여유 있음 → `llm.next_question(...)` → `ask`
   - 보험 2건 이상 → `build_comparison(...)`
   - 그 외 → `chunks = retriever.retrieve(slots)` · 0건이면 `ask`
9. `assessment = llm.generate_assessment(slots, chunks, cov, notes)`
10. 없는 청크 인용 제거 · 남은 게 없으면 한 번 재생성
11. `hydrate_citations(assessment)` · `readiness(...)`
12. 감사 마감 · `→ SessionResponse`

**테스트 관점** 세션 없음 → 404 · 잡담이면 LLM 0회 · 되묻기 2회차에는 범용 답변 · 검색 0건이면 판정 안 함 · 면책이면 되묻기 생략

#### SessionService.compute_missing 부족한 정보 판정

**시그니처** `def compute_missing(slots: SlotState) -> list[str]`

근거: [[INS-SEQ-001#SEQ-1]] 6단계

**처리** 공통 필수와 영역별 필수를 본다. **모름으로 확정된 항목은 부족으로 세지 않는다** — 물어봐야 답이 없다.

**테스트 관점** 모름 확정 항목은 다시 안 묻는다 · 영역을 모르면 그것부터

#### LlmGateway.extract_slots 슬롯 추출

**시그니처** `def extract_slots(history: list[Turn], text: str, current: SlotState) -> SlotState`

근거: [[INS-UC-001#UC-S1]]

**처리** 구조화 출력으로 값을 받는다. **"몰라"는 모름으로, "안 했어"는 0으로** 구분해 넣는다. 기존 값은 새 값이 있을 때만 덮는다.

**테스트 관점** "입원 안 했어" → 0 (비어 있음 아님) · "몰라" → 모름 확정 · 앞 턴 값이 안 지워진다

#### LlmGateway.generate_assessment 판정 생성

**시그니처** `def generate_assessment(slots, chunks, coverage, notes) -> AssistantAssessment`

근거: [[INS-SEQ-001#SEQ-1]] 9단계 · [[INS-PRD-001#R6]]

**처리** 구조화 출력(`citations` 최소 1건 강제). 규칙 판정 결과를 함께 넣는다. 결과에서 **존재하지 않는 청크 id를 걸러내고**, 남은 인용이 0이면 한 번 다시 만든다. 내부 식별자가 문장에 섞였으면 후처리로 지운다.

**테스트 관점** 인용 0 → 재시도 · 없는 청크 id → 걸러짐 · 응답에 UUID가 안 남는다 · 금지 표현 0

#### NeuroSymbolicRetriever.retrieve 융합 검색

**시그니처** `def retrieve(slots: SlotState, top_k: int = 8) -> list[Chunk]`

근거: [[INS-UC-001#UC-S2]] · [[INS-PRD-001#R2]]

**처리**
1. 뉴럴: 임베딩 검색 `2 × top_k` → 최고점 대비 상대 점수컷
2. 심볼릭: 조항 후보 + 참조·형제 확장 — **LLM 0회**
3. 가중 RRF로 합친다
4. 보험사 범위 검증 → `top_k`

**예외** 그래프가 응답하지 않으면 **경고만 남기고 뉴럴 단독으로 계속한다.** 검색을 실패시키지 않는다

**테스트 관점** 그래프 다운에도 결과가 나온다 · 범위 밖 청크가 안 섞인다 · 가중치 0이면 뉴럴 단독과 같다

#### SymbolicGraphChannel.clause_candidates 조항 후보

**시그니처** `def clause_candidates(query: str, insurer_id: int | None) -> list[Candidate]`

근거: [[INS-PRD-001#R2]]

**처리** 조항 제목 토큰과 질의 키워드의 교집합 크기를 점수로. **Cypher가 고정 문자열이다.**

**테스트 관점** 같은 질의에 같은 결과 (결정론) · LLM 호출 0

#### CoverageEngine.evaluate 규칙 판정

**시그니처** `def evaluate(facts: ClaimFacts) -> CoverageAssessment`

근거: [[INS-UC-001#UC-S4]] · [[INS-PRD-001#R10]]

**처리** 선언된 규칙을 순서대로 적용한다. 면책이 하나라도 걸리면 그것으로 확정. 사실이 모자라 판정할 수 없는 규칙은 `missing`에 남긴다. **순수 함수다.**

**테스트 관점** 같은 사실이면 같은 결과 · LLM 호출 0 · 치과면 면책 · **보장기간 밖이면 배제** · **급여/비급여가 있으면 자기부담 산출**

#### CoverageFacts.from_slots 사실 만들기

**시그니처** `def from_slots(slots: SlotState) -> ClaimFacts`

근거: [[INS-SEQ-001#SEQ-1]] 7단계

**처리** 슬롯에서 규칙 입력을 만든다. **보장기간(가입일·만기일)과 급여/비급여 분리 금액도 채워야 한다** — 지금은 슬롯에 그 항목이 없어 못 채운다(미결).

**테스트 관점** 슬롯이 비면 사실도 빈다 · 가입일이 있으면 보장기간 규칙이 발화한다

#### CitationHydrator.hydrate 인용 채우기

**시그니처** `def hydrate(citations: list[Citation]) -> list[Citation]`

근거: [[INS-UC-001#UC-S6]] · [[INS-PRD-001#R15]]

**처리** 청크 id로 문서·페이지를 찾고, 하이라이트 상자를 계산해 **면적이 가장 큰 페이지**를 고른다. 그 페이지를 이미지로 만든다. 가로 약관이면 반으로 자른다. **모델이 준 URL은 쓰지 않는다.**

**테스트 관점** 하이라이트 0박스가 없다 · 모델이 URL을 넣어도 무시된다 · 가로 약관이 잘린다

#### IngestionPipeline.ingest 약관 적재

**시그니처** `async def ingest(root: Path) -> IngestResult`

근거: [[INS-UC-001#UC-A12]] · [[INS-PRD-001#R1]]

**처리** 폴더 스캔 → 파싱 → 청킹 → 관계형 저장 → 임베딩 → 그래프. 같은 해시 문서는 건너뛴다.

**테스트 관점** 조항 인식률 95% 이상 · 임베딩 한도 초과 청크 0 · 세 저장소 개수 일치 · 같은 PDF 재적재 시 중복 안 생김

#### OcrAdapter.extract_information 서류 정보 추출

**시그니처** `async def extract_information(data: bytes, mime: str, schema: dict) -> dict`

근거: [[INS-UC-001#UC-H7]] · [[INS-PRD-001#R11]]

**처리** 스키마를 주고 한 번에 구조화 값을 받는다. 실패하면 텍스트 추출 후 모델로 뽑는 2단계로 떨어진다.

**테스트 관점** 진단서 필드가 채워진다 · 신뢰도 낮으면 표시된다 · 허용 필드 밖 값은 버려진다

#### AuditContext.complete 감사 마감

**시그니처** `def complete(self, response, retrieved_chunk_ids: list[str]) -> None`

근거: [[INS-UC-001#UC-S5]] · [[INS-PRD-001#R19]]

**처리** 응답 유형·해시·검색 청크·LLM 호출·신뢰도를 기록한다. **본문은 남기지 않는다.**

**테스트 관점** 본문이 DB에 없다 · 마스킹된 입력만 · 도중 실패해도 행이 남는다

## 2. 미결사항

- [ ] **`CoverageFacts.from_slots`가 보장기간·급여 분리 금액을 못 채운다** — 슬롯에 그 항목이 없다. 대화가 묻지 않아서다. 규칙 둘이 발화하지 못한다. 근거: [[INS-PRD-001#R10]]
- [ ] **`post_message`가 너무 크다** — 잡담·의도·게이팅·비교·검색·판정이 한 함수에 있다. 근거: [[INS-DOM-002]] 5장
- [ ] **업로드 뒤 슬롯 병합을 부르는 함수가 없다.** 근거: [[INS-SEQ-001]] 4장
- [ ] `hydrate`가 요청 경로에서 PDF를 연다 — 미리 만드는 편이 낫다
- [ ] 도구 경로가 별도 검색기를 쓴다 — 본 경로의 서킷 브레이커·점수컷을 우회한다

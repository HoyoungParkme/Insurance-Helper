---
doc_id: INS-SEQ-001
type: SEQ
title: SEQ — 보험청구심사 어시스턴트 시퀀스
status: draft
upstream: [INS-API-001, INS-DOM-002]
---

# 시퀀스: 보험청구심사 어시스턴트

## 1. 생명선

| 약칭 | 무엇 |
|---|---|
| B | 브라우저 |
| W | HTTP 라우터 |
| SS | SessionService |
| ST | SessionStore (메모리) |
| L | LlmGateway (Solar) |
| RG | NeuroSymbolicRetriever |
| V | 벡터 저장소 |
| GR | 그래프 저장소 |
| CE | CoverageEngine (규칙) |
| CH | CitationHydrator |
| AU | 감사 |

## 2. 시퀀스

#### SEQ-1 한 턴을 처리한다

근거: [[INS-UC-001#UC-H4]] · [[INS-API-001#POST/api/v1/sessions/{sessionId}/messages/stream]]

```mermaid
sequenceDiagram
  B->>W: POST /messages/stream {text}
  W->>SS: post_message(session_id, text)
  SS->>ST: get(session_id)
  alt 없음
    SS-->>B: 404 SESSION_NOT_FOUND
  end
  SS->>AU: 턴 시작
  SS->>SS: 잡담 가드 (정규식, LLM 0회)
  SS->>L: 의도 분류
  alt 일반 질의
    SS->>RG: 자유 질의 검색
    SS->>L: 설명 생성
    SS-->>B: answer
  end
  SS->>L: 슬롯 추출 (이력 + 이번 발화 + 현재 슬롯)
  SS->>SS: 부족한 것 계산 (코드, LLM 무관)
  SS->>CE: 규칙 판정 (사실 = 슬롯에서)
  CE-->>SS: 면책 / 조건부 / 해당 없음
  alt 면책에 걸림
    Note over SS: 되묻지 않고 바로 알린다
  else 부족하고 되묻기 여유 있음
    SS->>L: 다음 질문
    SS-->>B: ask (선택지 + "모르겠습니다")
  else 충분하거나 되묻기 소진
    SS->>RG: 검색
    RG->>V: 뉴럴
    RG->>GR: 심볼릭
    alt 그래프 장애
      Note over RG: 경고만 남기고 뉴럴 단독
    end
    RG-->>SS: 청크 top-k
    alt 0건
      Note over SS: 판정하지 않는다 — 인용 없는 답은 만들지 않는다
      SS-->>B: ask
    end
    SS->>L: 판정 생성 (구조화 출력, 스트리밍)
    L-->>B: delta*
    SS->>SS: 없는 청크 인용 걸러내기
    SS->>CH: 인용 URL·하이라이트 채우기
    CH-->>SS: Citation[]
    SS->>SS: 준비도 계산
    SS->>AU: 검색 청크 id 기록 · 턴 마감
    SS-->>B: final(assessment)
  end
```

**읽을 때 볼 것**
- 규칙 판정(`CE`)이 **되묻기보다 먼저**다. 면책이면 물어볼 이유가 없다
- **인용 URL은 서버가 채운다.** 모델이 URL을 만들면 환각이 그대로 화면에 나간다
- 검색 0건이면 판정을 만들지 않는다. 근거: [[INS-PRD-001#R6]]
- 감사는 턴 **시작**에 열고 끝에 닫는다 — 도중에 죽어도 기록이 남는다

#### SEQ-2 여러 보험을 비교한다

근거: [[INS-UC-001#UC-H6]]

```mermaid
sequenceDiagram
  SS->>SS: policies 2건 이상 확인
  loop 보험마다
    SS->>RG: 그 보험사 범위로 검색
    SS->>L: 판정 생성
  end
  SS->>CE: 비례 분담 계산
  alt 자기부담 금액을 못 구함
    Note over CE: 분담 금액을 비우고 등급 비교만
  end
  SS-->>B: comparison
```

#### SEQ-3 서류를 올린다

근거: [[INS-UC-001#UC-H7]]

```mermaid
sequenceDiagram
  B->>W: POST /documents (multipart)
  W->>ST: 세션 존재 확인
  W->>W: 저장 (해시·크기·형식 검증, TTL 24h)
  W->>L: OCR 텍스트 추출
  W->>W: 개인정보 마스킹
  W->>L: 문서 유형 분류
  alt 정밀 추출 대상
    W->>L: 구조화 정보 추출
  else
    W->>L: 마스킹된 텍스트에서 슬롯 추출
  end
  W-->>B: {doc_type, extracted_slots, low_confidence}
  Note over B,ST: 여기서 끊긴다 — 세션에 반영되지 않는다
```

**되먹일 것**: 추출한 슬롯을 `POST /slots`로 넣는 경로가 이미 있는데 부르지 않는다. 근거: [[INS-PRD-001#R11]]

#### SEQ-4 약관을 적재한다

근거: [[INS-UC-001#UC-A12]]

```mermaid
sequenceDiagram
  participant CLI
  CLI->>CLI: data/raw 폴더 스캔
  loop PDF마다
    CLI->>L: 문서 파싱 (구조 인식)
    CLI->>CLI: 조항·항·표 단위 청킹
    CLI->>CLI: 토큰 한도 검사
    CLI->>V: 임베딩 생성·저장
    CLI->>GR: 조항 관계 그래프 (LLM 0회)
  end
  CLI->>CLI: 세 저장소 개수 대조
  alt 어긋남
    Note over CLI: 재색인 절차
  end
```

#### SEQ-5 감사 기록을 되짚는다

근거: [[INS-UC-001#UC-A11]]

```mermaid
sequenceDiagram
  participant OP as 운영자
  OP->>AU: response_id로 조회
  AU-->>OP: 마스킹된 입력 · 검색 청크 id · LLM 호출 · 응답 유형 · 해시
  OP->>V: 그 청크들을 직접 확인
```

## 3. 대응표

| 시퀀스 | 유스케이스 | 엔드포인트 |
|---|---|---|
| [[#SEQ-1]] | [[INS-UC-001#UC-H4]] | [[INS-API-001#POST/api/v1/sessions/{sessionId}/messages/stream]] |
| [[#SEQ-2]] | [[INS-UC-001#UC-H6]] | 동일 |
| [[#SEQ-3]] | [[INS-UC-001#UC-H7]] | [[INS-API-001#POST/api/v1/sessions/{sessionId}/documents]] |
| [[#SEQ-4]] | [[INS-UC-001#UC-A12]] | CLI |
| [[#SEQ-5]] | [[INS-UC-001#UC-A11]] | — |

## 4. 되먹일 것

- **[[#SEQ-3]]이 끊겨 있다.** OCR 슬롯이 세션에 안 들어간다. 화면이 "인식된 정보"를 보여주지만 그건 화면 안에만 있고, 다음 턴의 슬롯 추출이 보는 이력에는 없다. 근거: [[INS-PRD-001#R11]]
- **[[#SEQ-1]]의 규칙 판정이 절반만 돈다.** 면책은 발화하는데 보장기간·자기부담은 사실이 안 채워져 못 돈다. 연쇄로 [[#SEQ-2]]의 분담 금액이 늘 빈다. 근거: [[INS-PRD-001#R10]]
- **인용 채우기가 요청 경로에서 PDF를 연다.** 인용 건수만큼 페이지를 스캔해 응답 지연에 직결된다. 미리 만들어 두는 편이 낫다

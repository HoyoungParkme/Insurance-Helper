---
doc_id: INS-UI-002
type: UI
title: UI — 보험청구심사 어시스턴트 와이어프레임
status: draft
upstream: [INS-UI-001]
---

# 와이어프레임: 보험청구심사 어시스턴트

## 1. 형식

요소 번호는 `data-el`이다. 화면 설계의 화면 하나가 여기 배치 하나에 대응한다.

#### UI-1 시작 배치

근거: [[INS-UI-001#UI-1]]

```html
<div class="page" data-el="1">
  <h1 data-el="1.1">보험금, 받을 수 있을까요?</h1>
  <p data-el="1.2">약관을 근거로 가능성을 알려드립니다. 최종 판단은 보험사에 확인하세요.</p>
  <button data-el="2" class="primary">내 보험으로 정확히 확인하기</button>
  <button data-el="3">로그인 없이 그냥 물어볼게요</button>
  <a data-el="4" href="/admin/graph">약관 지식그래프 탐색기 (관리자)</a>
  <footer data-el="5">면책 · 개인정보 · 출처 · 접근성</footer>
</div>
```

#### UI-3 내 보험 확인 배치

근거: [[INS-UI-001#UI-3]]

```html
<div class="page" data-el="1">
  <h1 data-el="1.1">가입하신 실손보험</h1>
  <div class="banner" data-el="2">두 개 이상 가입 시 중복 보상은 비례 분담됩니다</div>
  <ul data-el="3">
    <li data-el="3.1"><input type="checkbox" data-el="3.2"> 메리츠화재 실손 · 4세대 · 2023-05 가입</li>
  </ul>
  <button data-el="4" class="primary">2개 보험 비교하기</button>
</div>
```

#### UI-6 대화 배치

근거: [[INS-UI-001#UI-6]]

```html
<div class="page split" data-el="1">
  <aside class="doc" data-el="2">
    <img data-el="2.1" alt="약관 26쪽">
    <div class="highlight" data-el="2.2"></div>
  </aside>
  <main class="chat" data-el="3">
    <div class="msg user" data-el="3.1">발목 골절로 5일 입원했어요</div>
    <div class="msg bot assessment" data-el="4">
      <span class="badge high" data-el="4.1">가능성 높음</span>
      <p data-el="4.2">입원 의료비는 보장 대상입니다. 다만…</p>
      <ul class="ok" data-el="4.3">충족: 상해 · 입원</ul>
      <ul class="no" data-el="4.4">미충족: 자기부담금 정보 없음</ul>
      <ol class="cite" data-el="4.5"><li data-el="4.6">제3조(보상하는 손해) …</li></ol>
      <div class="readiness" data-el="4.7">청구 준비 72%</div>
      <p class="disclaimer" data-el="4.8">참고용이며 최종 판단을 대체하지 않습니다</p>
    </div>
    <div class="options" data-el="5">
      <button data-el="5.1">네</button><button data-el="5.2">아니요</button><button data-el="5.3">모르겠습니다</button>
    </div>
    <form data-el="6"><input data-el="6.1"><button data-el="6.2">첨부</button><button data-el="6.3">보내기</button></form>
  </main>
</div>
```

#### UI-8 약관 그래프 탐색 배치

근거: [[INS-UI-001#UI-8]]

```html
<div class="page" data-el="1">
  <aside data-el="2">조항 트리</aside>
  <main data-el="3"><canvas data-el="3.1"></canvas></main>
  <aside class="inspector" data-el="4">
    <div data-el="4.1">약관 원문</div>
    <div data-el="4.2">경로 찾기</div>
  </aside>
</div>
```

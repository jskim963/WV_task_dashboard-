# 우측 유틸리티 패널 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 메모 / 계산기(사칙연산+괄호+평↔㎡ 환산) / AI챗봇을 하나의 우측 슬라이드 패널로 통합하고, 손잡이 탭 클릭으로 모든 페이지에서 열고 닫을 수 있게 한다.

**Architecture:** `index.html` 단일 파일 안에서 기존 `#chatbot-panel`(플로팅 챗봇)을 `#utility-tab`(손잡이) + `#utility-drawer`(슬라이드 패널, `#notif-panel`과 동일한 트랜지션 패턴)로 교체한다. 기존 챗봇 로직(`sendChat` 등)과 메모 로직(`scheduleMemoSave` 등)은 그대로 재사용하고 DOM 위치만 이전한다. 계산기는 신규로 작성하며, `eval()` 대신 자체 recursive-descent 파서로 계산한다.

**Tech Stack:** Vanilla JS, Tailwind 유틸리티 클래스(기존 커스텀 테마: `bg-primary`, `text-on-surface`, `outline-variant` 등), localStorage(패널 상태/계산 이력), Firebase Firestore(기존 `memos` 컬렉션 재사용, 변경 없음).

---

## 사전 확인 사항 (참고용, 코드 작성 불필요)

- 이 프로젝트는 테스트 프레임워크가 없다. 검증은 아래 명령으로 `<script>` 블록 문법 오류만 확인하는 것이 사실상의 "테스트"다:
  ```bash
  node -e "
  const fs = require('fs');
  const html = fs.readFileSync('index.html', 'utf8');
  const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
  let ok = true;
  scripts.forEach((s,i) => { try { new Function(s); } catch(e) { ok = false; console.log('Block', i, 'ERROR:', e.message); } });
  console.log(ok ? 'SYNTAX OK' : 'SYNTAX ERRORS FOUND');
  "
  ```
  모든 태스크의 마지막 검증 단계에서 이 명령을 실행한다.
- 이 프로젝트 컨벤션상 브라우저 프리뷰 검증은 기본적으로 생략한다(사용자가 요청할 때만 수행). 이 계획의 Task 10에 수동 검증 체크리스트를 남겨두되, 스킵해도 무방하다.
- 커밋 전 `git status`로 의도치 않은 파일이 스테이징되지 않는지 확인한다.

---

### Task 1: CSS — 유틸리티 패널 스타일 추가

**Files:**
- Modify: `index.html:159-163` (기존 `/* AI Chatbot */` 블록), `index.html:209` (반응형 `.chat-window` 규칙)

- [ ] **Step 1: 기존 챗봇 고정 위치 CSS를 유틸리티 패널 CSS로 교체**

`index.html:159-163`의 다음 블록을:

```css
  /* AI Chatbot */
  #chatbot-panel { position:fixed; bottom:24px; right:24px; z-index:300; }
  .chat-window { width:440px; height:600px; background:#fff; border-radius:16px; box-shadow:0 16px 48px rgba(0,0,0,0.18); display:flex; flex-direction:column; overflow:hidden; border:1px solid #e2e8f0; }
```

다음으로 교체한다(`.chat-window` 규칙은 제거하고 `#utility-view-chat`이 그 역할을 대신함):

```css
  /* 우측 유틸리티 패널 (메모/계산기/AI챗봇) */
  #utility-tab { position:fixed; top:50%; right:0; transform:translateY(-50%); z-index:301; background:#1d4ed8; color:#fff; padding:16px 7px; border-radius:12px 0 0 12px; box-shadow:-2px 0 12px rgba(0,0,0,0.18); cursor:pointer; writing-mode:vertical-rl; font-size:12px; font-weight:700; letter-spacing:2px; transition:right 0.25s ease; user-select:none; }
  #utility-drawer { position:fixed; top:0; right:0; height:100%; width:360px; min-width:280px; max-width:560px; background:#fff; box-shadow:-4px 0 24px rgba(0,0,0,0.12); z-index:300; transform:translateX(100%); transition:transform 0.25s ease; display:flex; flex-direction:column; }
  #utility-drawer.open { transform:translateX(0); }
  #utility-resize-handle { position:absolute; top:0; left:-4px; width:8px; height:100%; cursor:ew-resize; z-index:302; }
  .utility-header { display:flex; border-bottom:1px solid #e2e8f0; flex-shrink:0; }
  .utility-tab-btn { flex:1; padding:12px 0; text-align:center; font-size:13px; font-weight:600; color:#64748b; cursor:pointer; border-bottom:2px solid transparent; background:none; border-left:none; border-right:none; border-top:none; }
  .utility-tab-btn.active { color:#1d4ed8; border-bottom-color:#1d4ed8; }
  .utility-view { display:none; flex:1; overflow-y:auto; flex-direction:column; min-height:0; }
  .utility-view.active { display:flex; }
  .chat-msg { padding:10px 14px; border-radius:12px; max-width:90%; font-size:13px; line-height:1.6; }
  .chat-msg.ai { background:#eff6ff; color:#1e40af; border-bottom-left-radius:4px; }
  .chat-msg.user { background:#2563eb; color:#fff; border-bottom-right-radius:4px; margin-left:auto; }
  .chat-msg table { width:100%; border-collapse:collapse; margin:6px 0; font-size:12px; }
  .chat-msg th { background:#dbeafe; padding:4px 8px; text-align:left; border:1px solid #bfdbfe; font-weight:600; }
  .chat-msg td { padding:4px 8px; border:1px solid #bfdbfe; vertical-align:top; }
  .chat-msg tr:nth-child(even) td { background:#f0f7ff; }
  .chat-msg ul { margin:4px 0 4px 16px; padding:0; list-style:disc; }
  .chat-msg li { margin:2px 0; }
  .typing-dot { width:6px; height:6px; background:#94a3b8; border-radius:50%; animation:bounce 1.4s infinite ease-in-out; }
  .typing-dot:nth-child(2) { animation-delay:0.2s; }
  .typing-dot:nth-child(3) { animation-delay:0.4s; }
  @keyframes bounce { 0%,80%,100%{transform:scale(0.6)} 40%{transform:scale(1)} }
  .calc-btn { padding:14px 0; text-align:center; font-size:16px; font-weight:600; border-radius:10px; background:#f1f5f9; color:#1e293b; cursor:pointer; user-select:none; }
  .calc-btn:hover { background:#e2e8f0; }
  .calc-btn.op { background:#dbeafe; color:#1d4ed8; }
  .calc-btn.eq { background:#1d4ed8; color:#fff; }
  .calc-history-item { padding:6px 10px; font-size:12px; color:#64748b; cursor:pointer; border-bottom:1px solid #f1f5f9; }
  .calc-history-item:hover { background:#f8fafc; }
```

- [ ] **Step 2: 반응형 규칙에서 `.chat-window` 참조 제거**

`index.html:209`의 다음 줄을:

```css
    .chat-window { width:calc(100vw - 32px); right:16px; }
```

다음으로 교체한다(모바일에서 유틸리티 드로어가 화면 대부분을 차지하도록):

```css
    #utility-drawer { width:100vw !important; max-width:100vw !important; }
```

- [ ] **Step 3: 문법 검증**

Run: 위 "사전 확인 사항"의 `node -e` 명령
Expected: `SYNTAX OK` (CSS는 이 스크립트가 검증하지 않지만, HTML 구조가 깨지지 않았는지 브라우저에서 육안 확인 권장 — 이 단계에서는 필수 아님)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: 유틸리티 패널 CSS 추가, 챗봇 고정 CSS 제거"
```

---

### Task 2: HTML — 챗봇 마크업을 유틸리티 패널 셸로 교체

**Files:**
- Modify: `index.html:1118-1151` (`<!-- ===== AI CHATBOT ===== -->` 블록)

- [ ] **Step 1: 기존 챗봇 마크업 전체를 유틸리티 패널 셸로 교체**

`index.html:1118-1151`의 다음 블록을:

```html
<!-- ===== AI CHATBOT ===== -->
<div id="chatbot-panel">
  <div id="chat-window" class="chat-window hidden flex-col">
    <div class="px-4 py-3 bg-primary flex items-center justify-between flex-shrink-0">
      <div class="flex items-center gap-2">
        <span class="material-symbols-outlined text-white text-base fill-icon">smart_toy</span>
        <div>
          <div class="text-white font-semibold text-sm">AI 어시스턴트</div>
          <div class="text-blue-200 text-xs" id="chat-status-label">온라인 · 무엇이든 물어보세요</div>
        </div>
      </div>
      <div class="flex gap-1">
        <button onclick="setChatApiKey()" class="text-white hover:bg-blue-700 p-1 rounded" title="API 키 설정"><span class="material-symbols-outlined text-base">key</span></button>
        <button onclick="clearChat()" class="text-white hover:bg-blue-700 p-1 rounded" title="대화 초기화"><span class="material-symbols-outlined text-base">restart_alt</span></button>
        <button onclick="toggleChat()" class="text-white hover:bg-blue-700 p-1 rounded"><span class="material-symbols-outlined text-base">close</span></button>
      </div>
    </div>
    <div id="chat-messages" class="flex-1 overflow-y-auto p-4 flex flex-col gap-3"></div>
    <div class="p-3 border-t border-outline-variant flex-shrink-0">
      <div class="flex flex-wrap gap-1.5 mb-2" id="quick-chips"></div>
      <div class="flex gap-2">
        <input type="text" id="chat-input" placeholder="메시지를 입력하세요..." class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" onkeydown="if(event.key==='Enter')sendChat()">
        <button onclick="sendChat()" class="bg-primary text-white px-3 py-2 rounded-lg hover:bg-blue-700 transition-all">
          <span class="material-symbols-outlined text-base">send</span>
        </button>
      </div>
      <p class="text-xs text-secondary mt-1.5 text-center">AI는 실수할 수 있으므로 중요한 정보는 확인해 주세요</p>
    </div>
  </div>
  <button onclick="toggleChat()" id="chat-toggle-btn" class="bg-primary text-white p-3.5 rounded-full shadow-lg hover:bg-blue-700 transition-all relative">
    <span class="material-symbols-outlined">chat</span>
    <span class="notif-badge"></span>
  </button>
</div>
```

다음으로 교체한다:

```html
<!-- ===== UTILITY PANEL (메모/계산기/AI챗봇) ===== -->
<div id="utility-tab" onclick="toggleUtilityPanel()">메모·계산기·AI</div>
<div id="utility-drawer">
  <div id="utility-resize-handle"></div>
  <div class="utility-header">
    <button class="utility-tab-btn" id="utility-tabbtn-memo" onclick="switchUtilityTab('memo')">메모</button>
    <button class="utility-tab-btn" id="utility-tabbtn-calc" onclick="switchUtilityTab('calc')">계산기</button>
    <button class="utility-tab-btn" id="utility-tabbtn-chat" onclick="switchUtilityTab('chat')">AI챗봇</button>
    <button onclick="toggleUtilityPanel()" class="px-3 text-secondary hover:bg-surface-container-low"><span class="material-symbols-outlined text-base">close</span></button>
  </div>

  <!-- 메모 뷰 -->
  <div id="utility-view-memo" class="utility-view p-4">
    <div class="flex items-center justify-between mb-2 flex-shrink-0">
      <h3 class="font-bold text-sm text-on-surface flex items-center gap-1.5"><span class="material-symbols-outlined text-yellow-500 text-base">sticky_note_2</span>개인 메모장</h3>
      <span class="text-xs text-secondary" id="memo-saved-indicator"></span>
    </div>
    <textarea id="personal-memo" class="flex-1 w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none text-on-surface placeholder:text-secondary" placeholder="개인 메모를 입력하세요... (자동 저장)" oninput="scheduleMemoSave()"></textarea>
  </div>

  <!-- 계산기 뷰 -->
  <div id="utility-view-calc" class="utility-view p-4">
    <div class="flex gap-1.5 mb-3 flex-shrink-0">
      <button class="utility-tab-btn" id="calc-subtab-basic" onclick="switchCalcSubtab('basic')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">일반</button>
      <button class="utility-tab-btn" id="calc-subtab-unit" onclick="switchCalcSubtab('unit')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">단위환산</button>
    </div>

    <div id="calc-basic-view">
      <div id="calc-display" class="text-right text-2xl font-semibold text-on-surface border border-outline-variant rounded-lg px-3 py-4 mb-3 min-h-[64px] break-all">0</div>
      <div class="grid grid-cols-4 gap-2 mb-3">
        <div class="calc-btn op" onclick="calcInput('(')">(</div>
        <div class="calc-btn op" onclick="calcInput(')')">)</div>
        <div class="calc-btn op" onclick="calcClear()">C</div>
        <div class="calc-btn op" onclick="calcBackspace()">⌫</div>
        <div class="calc-btn" onclick="calcInput('7')">7</div>
        <div class="calc-btn" onclick="calcInput('8')">8</div>
        <div class="calc-btn" onclick="calcInput('9')">9</div>
        <div class="calc-btn op" onclick="calcInput('÷')">÷</div>
        <div class="calc-btn" onclick="calcInput('4')">4</div>
        <div class="calc-btn" onclick="calcInput('5')">5</div>
        <div class="calc-btn" onclick="calcInput('6')">6</div>
        <div class="calc-btn op" onclick="calcInput('×')">×</div>
        <div class="calc-btn" onclick="calcInput('1')">1</div>
        <div class="calc-btn" onclick="calcInput('2')">2</div>
        <div class="calc-btn" onclick="calcInput('3')">3</div>
        <div class="calc-btn op" onclick="calcInput('-')">-</div>
        <div class="calc-btn" onclick="calcInput('0')">0</div>
        <div class="calc-btn" onclick="calcInput('.')">.</div>
        <div class="calc-btn eq" onclick="calcEquals()">=</div>
        <div class="calc-btn op" onclick="calcInput('+')">+</div>
      </div>
      <div class="flex items-center justify-between mb-1">
        <span class="text-xs font-semibold text-secondary">계산 이력</span>
        <button onclick="calcClearHistory()" class="text-xs text-secondary hover:text-red-500">지우기</button>
      </div>
      <div id="calc-history-list" class="border border-outline-variant rounded-lg max-h-32 overflow-y-auto"></div>
    </div>

    <div id="calc-unit-view" class="hidden">
      <div class="flex gap-2 mb-3">
        <select id="conv-direction" onchange="convUnit(document.getElementById('conv-input').value)" class="flex-1 border border-outline-variant rounded-lg px-2 py-2 text-sm">
          <option value="py2m2">평 → ㎡</option>
          <option value="m22py">㎡ → 평</option>
        </select>
      </div>
      <input type="number" id="conv-input" oninput="convUnit(this.value)" placeholder="숫자 입력" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm mb-3">
      <div id="conv-result" class="text-right text-xl font-semibold text-primary"></div>
    </div>
  </div>

  <!-- AI챗봇 뷰 -->
  <div id="utility-view-chat" class="utility-view">
    <div class="px-4 py-3 bg-primary flex items-center justify-between flex-shrink-0">
      <div class="flex items-center gap-2">
        <span class="material-symbols-outlined text-white text-base fill-icon">smart_toy</span>
        <div>
          <div class="text-white font-semibold text-sm">AI 어시스턴트</div>
          <div class="text-blue-200 text-xs" id="chat-status-label">온라인 · 무엇이든 물어보세요</div>
        </div>
      </div>
      <div class="flex gap-1">
        <button onclick="setChatApiKey()" class="text-white hover:bg-blue-700 p-1 rounded" title="API 키 설정"><span class="material-symbols-outlined text-base">key</span></button>
        <button onclick="clearChat()" class="text-white hover:bg-blue-700 p-1 rounded" title="대화 초기화"><span class="material-symbols-outlined text-base">restart_alt</span></button>
      </div>
    </div>
    <div id="chat-messages" class="flex-1 overflow-y-auto p-4 flex flex-col gap-3"></div>
    <div class="p-3 border-t border-outline-variant flex-shrink-0">
      <div class="flex flex-wrap gap-1.5 mb-2" id="quick-chips"></div>
      <div class="flex gap-2">
        <input type="text" id="chat-input" placeholder="메시지를 입력하세요..." class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" onkeydown="if(event.key==='Enter')sendChat()">
        <button onclick="sendChat()" class="bg-primary text-white px-3 py-2 rounded-lg hover:bg-blue-700 transition-all">
          <span class="material-symbols-outlined text-base">send</span>
        </button>
      </div>
      <p class="text-xs text-secondary mt-1.5 text-center">AI는 실수할 수 있으므로 중요한 정보는 확인해 주세요</p>
    </div>
  </div>
</div>
```

- [ ] **Step 2: 사이드바 하단 "AI 챗봇" 버튼을 유틸리티 패널 오픈으로 변경**

`index.html:397`의 다음 줄을:

```html
    <button onclick="toggleChat()" class="w-full flex items-center justify-center gap-2 bg-primary text-white py-2.5 rounded-lg hover:bg-blue-700 transition-all text-sm font-semibold relative">
```

다음으로 교체한다:

```html
    <button onclick="openUtilityTab('chat')" class="w-full flex items-center justify-center gap-2 bg-primary text-white py-2.5 rounded-lg hover:bg-blue-700 transition-all text-sm font-semibold relative">
```

- [ ] **Step 3: 문법 검증**

Run: `node -e` 문법 체크 명령 (Task 1 참고)
Expected: `SYNTAX OK`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: 유틸리티 패널 마크업으로 챗봇 UI 교체, 메모/계산기 뷰 셸 추가"
```

---

### Task 3: 대시보드 개인 메모장 카드 제거

**Files:**
- Modify: `index.html:495-502` (Panel 5), `index.html:5872-5876` (`Pages.overview.render()` 내 메모 초기화 코드)

메모 textarea(`#personal-memo`)는 Task 2에서 이미 유틸리티 드로어로 옮겨졌으므로, 대시보드 카드는 완전히 제거하고 실시간 동기화는 기존 `onSnapshot` 리스너(`index.html:5748-5757`, 변경 없음)에만 맡긴다.

- [ ] **Step 1: 대시보드 Panel 5 HTML 제거**

`index.html:495-502`의 다음 블록을 삭제한다:

```html
        <!-- Panel 5: 개인 메모장 -->
        <div class="bg-white rounded-xl border border-outline-variant shadow-sm p-5 flex flex-col">
          <div class="flex items-center justify-between mb-3">
            <h2 class="font-bold text-sm text-on-surface flex items-center gap-1.5"><span class="material-symbols-outlined text-yellow-500 text-base">sticky_note_2</span>개인 메모장</h2>
            <span class="text-xs text-secondary" id="memo-saved-indicator"></span>
          </div>
          <textarea id="personal-memo" rows="6" class="flex-1 w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none text-on-surface placeholder:text-secondary" placeholder="개인 메모를 입력하세요... (자동 저장)" oninput="scheduleMemoSave()"></textarea>
        </div>
```

> 참고: 이 자리는 `2026-07-15-mentions-inbox.md` 플랜의 Task에서 "멘션함" 카드로 채워진다. 두 플랜 실행 순서는 무관하지만, 이 플랜을 먼저 실행하면 대시보드에 빈 자리가 생기는 게 정상이다(멘션함 플랜에서 채워짐).

- [ ] **Step 2: `Pages.overview.render()`의 메모 초기화 코드 제거**

`index.html:5872-5876`의 다음 블록을:

```js
    // Panel 5: 메모
    const memo = DB.memos ? DB.memos.find(m => m.userId === myId) : null;
    const memoEl = document.getElementById('personal-memo');
    if (memoEl) memoEl.value = memo ? memo.content : '';
    document.getElementById('memo-saved-indicator').textContent = '';
```

삭제한다(빈 줄만 남긴다). `#personal-memo`는 이제 대시보드 렌더링과 무관하게 항상 DOM에 존재하며, `index.html:5748-5757`의 `onSnapshot` 리스너가 최초 구독 시점에 즉시 현재 값을 채워주므로 초기화 코드가 별도로 필요 없다.

- [ ] **Step 3: 문법 검증 및 육안 확인**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "refactor: 대시보드 개인 메모장 카드 제거 (유틸리티 패널로 이전)"
```

---

### Task 4: 패널 열기/닫기 + 탭 전환 + 상태 기억

**Files:**
- Modify: `index.html` — Task 2에서 삭제한 `toggleChat()` 자리(원래 `index.html:4760-4770` 부근)에 새 함수들 추가

- [ ] **Step 1: 기존 `toggleChat()`을 제거하고 유틸리티 패널 제어 함수로 교체**

`index.html:4760-4770`의 다음 블록을:

```js
function toggleChat() {
  chatOpen = !chatOpen;
  const win = document.getElementById('chat-window');
  const btn = document.getElementById('chat-toggle-btn');
  win.classList.toggle('hidden', !chatOpen);
  btn.classList.toggle('hidden', chatOpen);
  if (chatOpen) {
    updateChatStatusLabel();
    if (chatHistory.length === 0) initChat();
  }
}
```

다음으로 교체한다:

```js
let utilityPanelOpen = false;
let utilityActiveTab = 'memo';

function toggleUtilityPanel() {
  setUtilityPanelOpen(!utilityPanelOpen);
}

function setUtilityPanelOpen(open) {
  utilityPanelOpen = open;
  const drawer = document.getElementById('utility-drawer');
  const tab = document.getElementById('utility-tab');
  drawer.classList.toggle('open', utilityPanelOpen);
  tab.style.right = utilityPanelOpen ? drawer.offsetWidth + 'px' : '0px';
  try { localStorage.setItem('utilityPanelOpen', utilityPanelOpen ? '1' : '0'); } catch(e) {}
  if (utilityPanelOpen && utilityActiveTab === 'chat') {
    updateChatStatusLabel();
    if (chatHistory.length === 0) initChat();
  }
}

function switchUtilityTab(tabName) {
  utilityActiveTab = tabName;
  ['memo','calc','chat'].forEach(t => {
    document.getElementById('utility-tabbtn-' + t).classList.toggle('active', t === tabName);
    document.getElementById('utility-view-' + t).classList.toggle('active', t === tabName);
  });
  try { localStorage.setItem('utilityPanelTab', tabName); } catch(e) {}
  if (tabName === 'chat') {
    updateChatStatusLabel();
    if (chatHistory.length === 0) initChat();
  }
}

function openUtilityTab(tabName) {
  switchUtilityTab(tabName);
  if (!utilityPanelOpen) setUtilityPanelOpen(true);
}

function initUtilityPanelState() {
  let savedTab = 'memo', savedOpen = false, savedWidth = 360;
  try {
    savedTab = localStorage.getItem('utilityPanelTab') || 'memo';
    savedOpen = localStorage.getItem('utilityPanelOpen') === '1';
    savedWidth = parseInt(localStorage.getItem('utilityPanelWidth'), 10) || 360;
  } catch(e) {}
  document.getElementById('utility-drawer').style.width = savedWidth + 'px';
  switchUtilityTab(savedTab);
  if (savedOpen) setUtilityPanelOpen(true);
}
```

- [ ] **Step 2: `chatOpen` 변수 제거, 앱 초기화 시 `initUtilityPanelState()` 호출 추가**

`index.html:4733`의 다음 줄을:

```js
let chatOpen = false;
```

삭제한다(더 이상 사용하지 않음 — `utilityPanelOpen`으로 대체됨).

`applyCurrentUser()` 함수(`index.html:5737`에서 시작) 안, 메모 실시간 동기화 `onSnapshot` 등록 블록 바로 다음, 함수를 닫는 마지막 `}`(`index.html:5759`) 바로 앞에 다음 줄을 추가한다:

```js
  initUtilityPanelState();
```

`applyCurrentUser()`는 신규 로그인(`doLogin`/`doSetPassword`) 시점과 세션 복원(`checkAuth()`, `index.html:5611`) 시점 양쪽에서 모두 호출되는 유일한 함수이므로, 여기에 넣으면 새로고침 후에도 유틸리티 패널 상태가 항상 초기화된다.

- [ ] **Step 3: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`. `chatOpen` 미정의 참조가 남아있지 않은지 확인하려면 다음도 실행:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const leftover = html.match(/\bchatOpen\b/g);
console.log(leftover ? 'STILL REFERENCED: ' + leftover.length : 'NO LEFTOVER REFERENCES');
"
```

Expected: `NO LEFTOVER REFERENCES`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: 유틸리티 패널 열기/닫기, 탭 전환, localStorage 상태 기억 구현"
```

---

### Task 4.5: 탭 방식 → 상중하단 스택 레이아웃 전환 (사용자 요청으로 설계 변경)

> **왜 이 태스크가 필요한가:** Task 2/4까지는 메모/계산기/AI챗봇을 클릭형 탭으로 전환하는 방식으로 구현했다. 사용자가 구현 도중 "탭을 골라서 하는 것 말고 상중하단으로 한번에 보이면 좋겠다"고 요청해, 세 섹션을 항상 동시에 세로로 쌓아 보여주고 섹션 경계를 드래그해 높이를 조절하는 방식으로 변경한다. 이 태스크는 Task 2(마크업)와 Task 4(탭 전환 JS)가 만든 결과물을 대체한다. 계산기 내부의 "일반/단위환산" 서브탭은 이번 변경과 무관하며 그대로 유지한다.

**Files:**
- Modify: `index.html:159-168` (유틸리티 패널 CSS), `index.html:1123-1219` (유틸리티 패널 HTML), `index.html:410` 부근(사이드바 AI챗봇 버튼), `index.html:4826-4886` 부근(패널 제어 JS)

- [ ] **Step 1: CSS — 탭 바 스타일 제거, 스택 섹션 + 세로 리사이즈 핸들 스타일 추가**

`index.html:164-168`의 다음 블록을:

```css
  .utility-header { display:flex; border-bottom:1px solid #e2e8f0; flex-shrink:0; }
  .utility-tab-btn { flex:1; padding:12px 0; text-align:center; font-size:13px; font-weight:600; color:#64748b; cursor:pointer; border-bottom:2px solid transparent; background:none; border-left:none; border-right:none; border-top:none; }
  .utility-tab-btn.active { color:#1d4ed8; border-bottom-color:#1d4ed8; }
  .utility-view { display:none; flex:1; overflow-y:auto; flex-direction:column; min-height:0; }
  .utility-view.active { display:flex; }
```

다음으로 교체한다. `.utility-tab-btn`/`.active`는 계산기 내부 "일반/단위환산" 서브탭이 계속 사용하므로 그대로 남긴다:

```css
  .utility-header { display:flex; align-items:center; justify-content:space-between; padding:0 6px 0 16px; border-bottom:1px solid #e2e8f0; flex-shrink:0; }
  .utility-tab-btn { flex:1; padding:12px 0; text-align:center; font-size:13px; font-weight:600; color:#64748b; cursor:pointer; border-bottom:2px solid transparent; background:none; border-left:none; border-right:none; border-top:none; }
  .utility-tab-btn.active { color:#1d4ed8; border-bottom-color:#1d4ed8; }
  .utility-view { display:flex; flex-direction:column; overflow-y:auto; }
  #utility-view-memo { flex:0 0 auto; height:160px; min-height:80px; border-bottom:1px solid #e2e8f0; }
  #utility-view-calc { flex:0 0 auto; height:320px; min-height:200px; border-bottom:1px solid #e2e8f0; }
  #utility-view-chat { flex:1 1 auto; min-height:160px; }
  .utility-vresize-handle { height:6px; cursor:row-resize; background:#f8fafc; flex-shrink:0; position:relative; }
  .utility-vresize-handle::after { content:''; position:absolute; left:50%; top:50%; transform:translate(-50%,-50%); width:32px; height:3px; border-radius:2px; background:#cbd5e1; }
  .utility-vresize-handle:hover::after { background:#94a3b8; }
```

> **후속 수정 (코드 리뷰 반영):** 위 `#utility-view-calc`의 `height:320px`는 이 태스크가 실제 구현된 시점의 값이며, 곧이어 코드 품질 리뷰에서 "계산기 버튼 그리드+이력이 기본 높이에서 스크롤 없이 다 보이지 않는다"는 지적을 받아 별도 후속 커밋(`2dd06d5`, "계산기 기본 높이 확대 및 리사이즈 상한선 추가")에서 `460px`로 상향 조정되었다. **현재 실제 코드의 값은 320px가 아니라 460px**이며, `initUtilityPanelState()`의 `savedCalcH` 기본값도 동일하게 460으로 맞춰져 있다. 이 계획 문서는 각 태스크가 실행된 시점의 코드를 기록하는 역사적 문서이므로 위 스니펫 자체는 수정하지 않았다.

- [ ] **Step 2: HTML — 탭 헤더를 정적 타이틀로, 세 뷰를 항상 표시되는 스택 섹션으로 교체**

`index.html:1123-1219`의 다음 블록을(현재 실제 파일 내용 기준):

```html
<div id="utility-tab" onclick="toggleUtilityPanel()">메모·계산기·AI</div>
<div id="utility-drawer">
  <div id="utility-resize-handle"></div>
  <div class="utility-header">
    <button class="utility-tab-btn" id="utility-tabbtn-memo" onclick="switchUtilityTab('memo')">메모</button>
    <button class="utility-tab-btn" id="utility-tabbtn-calc" onclick="switchUtilityTab('calc')">계산기</button>
    <button class="utility-tab-btn" id="utility-tabbtn-chat" onclick="switchUtilityTab('chat')">AI챗봇</button>
    <button onclick="toggleUtilityPanel()" class="px-3 text-secondary hover:bg-surface-container-low"><span class="material-symbols-outlined text-base">close</span></button>
  </div>

  <!-- 메모 뷰 -->
  <div id="utility-view-memo" class="utility-view p-4">
    <div class="flex items-center justify-between mb-2 flex-shrink-0">
      <h3 class="font-bold text-sm text-on-surface flex items-center gap-1.5"><span class="material-symbols-outlined text-yellow-500 text-base">sticky_note_2</span>개인 메모장</h3>
      <span class="text-xs text-secondary" id="memo-saved-indicator"></span>
    </div>
    <textarea id="personal-memo" class="flex-1 w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none text-on-surface placeholder:text-secondary" placeholder="개인 메모를 입력하세요... (자동 저장)" oninput="scheduleMemoSave()"></textarea>
  </div>

  <!-- 계산기 뷰 -->
  <div id="utility-view-calc" class="utility-view p-4">
    <div class="flex gap-1.5 mb-3 flex-shrink-0">
      <button class="utility-tab-btn" id="calc-subtab-basic" onclick="switchCalcSubtab('basic')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">일반</button>
      <button class="utility-tab-btn" id="calc-subtab-unit" onclick="switchCalcSubtab('unit')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">단위환산</button>
    </div>

    <div id="calc-basic-view">
      <div id="calc-display" class="text-right text-2xl font-semibold text-on-surface border border-outline-variant rounded-lg px-3 py-4 mb-3 min-h-[64px] break-all">0</div>
      <div class="grid grid-cols-4 gap-2 mb-3">
        <div class="calc-btn op" onclick="calcInput('(')">(</div>
        <div class="calc-btn op" onclick="calcInput(')')">)</div>
        <div class="calc-btn op" onclick="calcClear()">C</div>
        <div class="calc-btn op" onclick="calcBackspace()">⌫</div>
        <div class="calc-btn" onclick="calcInput('7')">7</div>
        <div class="calc-btn" onclick="calcInput('8')">8</div>
        <div class="calc-btn" onclick="calcInput('9')">9</div>
        <div class="calc-btn op" onclick="calcInput('÷')">÷</div>
        <div class="calc-btn" onclick="calcInput('4')">4</div>
        <div class="calc-btn" onclick="calcInput('5')">5</div>
        <div class="calc-btn" onclick="calcInput('6')">6</div>
        <div class="calc-btn op" onclick="calcInput('×')">×</div>
        <div class="calc-btn" onclick="calcInput('1')">1</div>
        <div class="calc-btn" onclick="calcInput('2')">2</div>
        <div class="calc-btn" onclick="calcInput('3')">3</div>
        <div class="calc-btn op" onclick="calcInput('-')">-</div>
        <div class="calc-btn" onclick="calcInput('0')">0</div>
        <div class="calc-btn" onclick="calcInput('.')">.</div>
        <div class="calc-btn eq" onclick="calcEquals()">=</div>
        <div class="calc-btn op" onclick="calcInput('+')">+</div>
      </div>
      <div class="flex items-center justify-between mb-1">
        <span class="text-xs font-semibold text-secondary">계산 이력</span>
        <button onclick="calcClearHistory()" class="text-xs text-secondary hover:text-red-500">지우기</button>
      </div>
      <div id="calc-history-list" class="border border-outline-variant rounded-lg max-h-32 overflow-y-auto"></div>
    </div>

    <div id="calc-unit-view" class="hidden">
      <div class="flex gap-2 mb-3">
        <select id="conv-direction" onchange="convUnit(document.getElementById('conv-input').value)" class="flex-1 border border-outline-variant rounded-lg px-2 py-2 text-sm">
          <option value="py2m2">평 → ㎡</option>
          <option value="m22py">㎡ → 평</option>
        </select>
      </div>
      <input type="number" id="conv-input" oninput="convUnit(this.value)" placeholder="숫자 입력" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm mb-3">
      <div id="conv-result" class="text-right text-xl font-semibold text-primary"></div>
    </div>
  </div>

  <!-- AI챗봇 뷰 -->
  <div id="utility-view-chat" class="utility-view">
    <div class="px-4 py-3 bg-primary flex items-center justify-between flex-shrink-0">
      <div class="flex items-center gap-2">
        <span class="material-symbols-outlined text-white text-base fill-icon">smart_toy</span>
        <div>
          <div class="text-white font-semibold text-sm">AI 어시스턴트</div>
          <div class="text-blue-200 text-xs" id="chat-status-label">온라인 · 무엇이든 물어보세요</div>
        </div>
      </div>
      <div class="flex gap-1">
        <button onclick="setChatApiKey()" class="text-white hover:bg-blue-700 p-1 rounded" title="API 키 설정"><span class="material-symbols-outlined text-base">key</span></button>
        <button onclick="clearChat()" class="text-white hover:bg-blue-700 p-1 rounded" title="대화 초기화"><span class="material-symbols-outlined text-base">restart_alt</span></button>
      </div>
    </div>
    <div id="chat-messages" class="flex-1 overflow-y-auto p-4 flex flex-col gap-3"></div>
    <div class="p-3 border-t border-outline-variant flex-shrink-0">
      <div class="flex flex-wrap gap-1.5 mb-2" id="quick-chips"></div>
      <div class="flex gap-2">
        <input type="text" id="chat-input" placeholder="메시지를 입력하세요..." class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" onkeydown="if(event.key==='Enter')sendChat()">
        <button onclick="sendChat()" class="bg-primary text-white px-3 py-2 rounded-lg hover:bg-blue-700 transition-all">
          <span class="material-symbols-outlined text-base">send</span>
        </button>
      </div>
      <p class="text-xs text-secondary mt-1.5 text-center">AI는 실수할 수 있으므로 중요한 정보는 확인해 주세요</p>
    </div>
  </div>
</div>
```

다음으로 교체한다:

```html
<div id="utility-tab" onclick="toggleUtilityPanel()">메모·계산기·AI</div>
<div id="utility-drawer">
  <div id="utility-resize-handle"></div>
  <div class="utility-header">
    <span class="font-bold text-sm text-on-surface">메모 · 계산기 · AI챗봇</span>
    <button onclick="toggleUtilityPanel()" class="p-2 text-secondary hover:bg-surface-container-low rounded-lg"><span class="material-symbols-outlined text-base">close</span></button>
  </div>

  <!-- 메모 뷰 -->
  <div id="utility-view-memo" class="utility-view p-4">
    <div class="flex items-center justify-between mb-2 flex-shrink-0">
      <h3 class="font-bold text-sm text-on-surface flex items-center gap-1.5"><span class="material-symbols-outlined text-yellow-500 text-base">sticky_note_2</span>개인 메모장</h3>
      <span class="text-xs text-secondary" id="memo-saved-indicator"></span>
    </div>
    <textarea id="personal-memo" class="flex-1 w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none text-on-surface placeholder:text-secondary" placeholder="개인 메모를 입력하세요... (자동 저장)" oninput="scheduleMemoSave()"></textarea>
  </div>

  <div class="utility-vresize-handle" id="utility-vresize-1"></div>

  <!-- 계산기 뷰 -->
  <div id="utility-view-calc" class="utility-view p-4">
    <h3 class="font-bold text-sm text-on-surface flex items-center gap-1.5 mb-2 flex-shrink-0"><span class="material-symbols-outlined text-blue-500 text-base">calculate</span>계산기</h3>
    <div class="flex gap-1.5 mb-3 flex-shrink-0">
      <button class="utility-tab-btn" id="calc-subtab-basic" onclick="switchCalcSubtab('basic')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">일반</button>
      <button class="utility-tab-btn" id="calc-subtab-unit" onclick="switchCalcSubtab('unit')" style="flex:1;border-radius:8px;border:1px solid #e2e8f0;">단위환산</button>
    </div>

    <div id="calc-basic-view">
      <div id="calc-display" class="text-right text-2xl font-semibold text-on-surface border border-outline-variant rounded-lg px-3 py-4 mb-3 min-h-[64px] break-all">0</div>
      <div class="grid grid-cols-4 gap-2 mb-3">
        <div class="calc-btn op" onclick="calcInput('(')">(</div>
        <div class="calc-btn op" onclick="calcInput(')')">)</div>
        <div class="calc-btn op" onclick="calcClear()">C</div>
        <div class="calc-btn op" onclick="calcBackspace()">⌫</div>
        <div class="calc-btn" onclick="calcInput('7')">7</div>
        <div class="calc-btn" onclick="calcInput('8')">8</div>
        <div class="calc-btn" onclick="calcInput('9')">9</div>
        <div class="calc-btn op" onclick="calcInput('÷')">÷</div>
        <div class="calc-btn" onclick="calcInput('4')">4</div>
        <div class="calc-btn" onclick="calcInput('5')">5</div>
        <div class="calc-btn" onclick="calcInput('6')">6</div>
        <div class="calc-btn op" onclick="calcInput('×')">×</div>
        <div class="calc-btn" onclick="calcInput('1')">1</div>
        <div class="calc-btn" onclick="calcInput('2')">2</div>
        <div class="calc-btn" onclick="calcInput('3')">3</div>
        <div class="calc-btn op" onclick="calcInput('-')">-</div>
        <div class="calc-btn" onclick="calcInput('0')">0</div>
        <div class="calc-btn" onclick="calcInput('.')">.</div>
        <div class="calc-btn eq" onclick="calcEquals()">=</div>
        <div class="calc-btn op" onclick="calcInput('+')">+</div>
      </div>
      <div class="flex items-center justify-between mb-1">
        <span class="text-xs font-semibold text-secondary">계산 이력</span>
        <button onclick="calcClearHistory()" class="text-xs text-secondary hover:text-red-500">지우기</button>
      </div>
      <div id="calc-history-list" class="border border-outline-variant rounded-lg max-h-32 overflow-y-auto"></div>
    </div>

    <div id="calc-unit-view" class="hidden">
      <div class="flex gap-2 mb-3">
        <select id="conv-direction" onchange="convUnit(document.getElementById('conv-input').value)" class="flex-1 border border-outline-variant rounded-lg px-2 py-2 text-sm">
          <option value="py2m2">평 → ㎡</option>
          <option value="m22py">㎡ → 평</option>
        </select>
      </div>
      <input type="number" id="conv-input" oninput="convUnit(this.value)" placeholder="숫자 입력" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm mb-3">
      <div id="conv-result" class="text-right text-xl font-semibold text-primary"></div>
    </div>
  </div>

  <div class="utility-vresize-handle" id="utility-vresize-2"></div>

  <!-- AI챗봇 뷰 -->
  <div id="utility-view-chat" class="utility-view">
    <div class="px-4 py-3 bg-primary flex items-center justify-between flex-shrink-0">
      <div class="flex items-center gap-2">
        <span class="material-symbols-outlined text-white text-base fill-icon">smart_toy</span>
        <div>
          <div class="text-white font-semibold text-sm">AI 어시스턴트</div>
          <div class="text-blue-200 text-xs" id="chat-status-label">온라인 · 무엇이든 물어보세요</div>
        </div>
      </div>
      <div class="flex gap-1">
        <button onclick="setChatApiKey()" class="text-white hover:bg-blue-700 p-1 rounded" title="API 키 설정"><span class="material-symbols-outlined text-base">key</span></button>
        <button onclick="clearChat()" class="text-white hover:bg-blue-700 p-1 rounded" title="대화 초기화"><span class="material-symbols-outlined text-base">restart_alt</span></button>
      </div>
    </div>
    <div id="chat-messages" class="flex-1 overflow-y-auto p-4 flex flex-col gap-3"></div>
    <div class="p-3 border-t border-outline-variant flex-shrink-0">
      <div class="flex flex-wrap gap-1.5 mb-2" id="quick-chips"></div>
      <div class="flex gap-2">
        <input type="text" id="chat-input" placeholder="메시지를 입력하세요..." class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" onkeydown="if(event.key==='Enter')sendChat()">
        <button onclick="sendChat()" class="bg-primary text-white px-3 py-2 rounded-lg hover:bg-blue-700 transition-all">
          <span class="material-symbols-outlined text-base">send</span>
        </button>
      </div>
      <p class="text-xs text-secondary mt-1.5 text-center">AI는 실수할 수 있으므로 중요한 정보는 확인해 주세요</p>
    </div>
  </div>
</div>
```

변경 요약: 헤더의 3개 탭 버튼(`utility-tabbtn-*`)을 정적 타이틀 텍스트로 교체, 두 섹션 경계에 `.utility-vresize-handle` div 2개(`utility-vresize-1`, `utility-vresize-2`) 추가, 계산기 뷰 상단에 제목(`계산기`) 추가. 계산기 내부 일반/단위환산 서브탭과 각 섹션의 내부 내용(메모 textarea, 계산기 버튼 그리드, 챗봇 메시지창)은 완전히 동일하게 유지.

- [ ] **Step 3: 사이드바 "AI 챗봇" 버튼 갱신**

사이드바 하단 버튼(`index.html:410` 부근)의 다음 줄을:

```html
    <button onclick="openUtilityTab('chat')" class="w-full flex items-center justify-center gap-2 bg-primary text-white py-2.5 rounded-lg hover:bg-blue-700 transition-all text-sm font-semibold relative">
```

다음으로 교체한다:

```html
    <button onclick="openUtilityPanel('chat')" class="w-full flex items-center justify-center gap-2 bg-primary text-white py-2.5 rounded-lg hover:bg-blue-700 transition-all text-sm font-semibold relative">
```

- [ ] **Step 4: JS — 탭 전환 로직 제거, 패널 열기/상태기억을 스택 레이아웃에 맞게 재작성**

`function toggleChat()`이 있던 자리에 Task 4에서 추가했던 다음 블록 전체를(`function toggleUtilityPanel`부터 `function initUtilityPanelState`까지, `let utilityPanelOpen`/`let utilityActiveTab`/`let chatInitialized` 선언 포함):

```js
let utilityPanelOpen = false;
let utilityActiveTab = 'memo';
let chatInitialized = false;

function toggleUtilityPanel() {
  setUtilityPanelOpen(!utilityPanelOpen);
}

function ensureChatReady() {
  updateChatStatusLabel();
  if (!chatInitialized) { chatInitialized = true; initChat(); }
}

function setUtilityPanelOpen(open) {
  utilityPanelOpen = open;
  const drawer = document.getElementById('utility-drawer');
  const tab = document.getElementById('utility-tab');
  drawer.classList.toggle('open', utilityPanelOpen);
  tab.style.right = utilityPanelOpen ? drawer.offsetWidth + 'px' : '0px';
  try { localStorage.setItem('utilityPanelOpen', utilityPanelOpen ? '1' : '0'); } catch(e) {}
  if (utilityPanelOpen && utilityActiveTab === 'chat') {
    ensureChatReady();
  }
}

function switchUtilityTab(tabName) {
  utilityActiveTab = tabName;
  ['memo','calc','chat'].forEach(t => {
    document.getElementById('utility-tabbtn-' + t).classList.toggle('active', t === tabName);
    document.getElementById('utility-view-' + t).classList.toggle('active', t === tabName);
  });
  try { localStorage.setItem('utilityPanelTab', tabName); } catch(e) {}
  if (tabName === 'chat') {
    ensureChatReady();
  }
}

function openUtilityTab(tabName) {
  switchUtilityTab(tabName);
  if (!utilityPanelOpen) setUtilityPanelOpen(true);
}

function initUtilityPanelState() {
  let savedTab = 'memo', savedOpen = false, savedWidth = 360;
  try {
    savedTab = localStorage.getItem('utilityPanelTab') || 'memo';
    savedOpen = localStorage.getItem('utilityPanelOpen') === '1';
    savedWidth = parseInt(localStorage.getItem('utilityPanelWidth'), 10) || 360;
  } catch(e) {}
  document.getElementById('utility-drawer').style.width = savedWidth + 'px';
  switchUtilityTab(savedTab);
  if (savedOpen) setUtilityPanelOpen(true);
}
```

다음으로 교체한다(`ensureChatReady`/`initChat`은 그대로 재사용, `utilityActiveTab`/`switchUtilityTab`/`openUtilityTab`은 완전히 제거):

```js
let utilityPanelOpen = false;
let chatInitialized = false;

function toggleUtilityPanel() {
  setUtilityPanelOpen(!utilityPanelOpen);
}

function ensureChatReady() {
  updateChatStatusLabel();
  if (!chatInitialized) { chatInitialized = true; initChat(); }
}

function setUtilityPanelOpen(open) {
  utilityPanelOpen = open;
  const drawer = document.getElementById('utility-drawer');
  const tab = document.getElementById('utility-tab');
  drawer.classList.toggle('open', utilityPanelOpen);
  tab.style.right = utilityPanelOpen ? drawer.offsetWidth + 'px' : '0px';
  try { localStorage.setItem('utilityPanelOpen', utilityPanelOpen ? '1' : '0'); } catch(e) {}
  if (utilityPanelOpen) ensureChatReady();
}

function openUtilityPanel(section) {
  if (!utilityPanelOpen) setUtilityPanelOpen(true);
  const el = document.getElementById('utility-view-' + section);
  if (el) el.scrollIntoView({ behavior: 'smooth', block: 'start' });
}

function initUtilityPanelState() {
  let savedOpen = false, savedWidth = 360, savedMemoH = 160, savedCalcH = 320;
  try {
    savedOpen = localStorage.getItem('utilityPanelOpen') === '1';
    savedWidth = parseInt(localStorage.getItem('utilityPanelWidth'), 10) || 360;
    savedMemoH = parseInt(localStorage.getItem('utilityMemoHeight'), 10) || 160;
    savedCalcH = parseInt(localStorage.getItem('utilityCalcHeight'), 10) || 320;
  } catch(e) {}
  document.getElementById('utility-drawer').style.width = savedWidth + 'px';
  document.getElementById('utility-view-memo').style.height = savedMemoH + 'px';
  document.getElementById('utility-view-calc').style.height = savedCalcH + 'px';
  if (savedOpen) setUtilityPanelOpen(true);
}

(function initUtilityVResizeHandles() {
  function bindVHandle(handleId, panelId, minH, storageKey) {
    const handle = document.getElementById(handleId);
    const panel = document.getElementById(panelId);
    if (!handle || !panel) return;
    let dragging = false;
    handle.addEventListener('mousedown', (e) => { dragging = true; e.preventDefault(); });
    document.addEventListener('mousemove', (e) => {
      if (!dragging) return;
      const rect = panel.getBoundingClientRect();
      let h = e.clientY - rect.top;
      h = Math.max(minH, h);
      panel.style.height = h + 'px';
    });
    document.addEventListener('mouseup', () => {
      if (!dragging) return;
      dragging = false;
      try { localStorage.setItem(storageKey, String(panel.offsetHeight)); } catch(e) {}
    });
  }
  bindVHandle('utility-vresize-1', 'utility-view-memo', 80, 'utilityMemoHeight');
  bindVHandle('utility-vresize-2', 'utility-view-calc', 200, 'utilityCalcHeight');
})();
```

`initUtilityVResizeHandles`는 Task 5(가로 폭 리사이즈 핸들)와 마찬가지로, 이 파일의 유일한 `<script>` 블록(`index.html:1921`부터 시작)이 모든 HTML 마크업 뒤에 위치하므로 `#utility-vresize-1`/`#utility-vresize-2`가 이미 DOM에 존재하는 시점에 실행되어 `DOMContentLoaded` 대기가 필요 없다.

- [ ] **Step 5: 문법 검증 + 남은 참조 확인**

Run:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
let ok = true;
scripts.forEach((s,i) => { try { new Function(s); } catch(e) { ok = false; console.log('Block', i, 'ERROR:', e.message); } });
console.log(ok ? 'SYNTAX OK' : 'SYNTAX ERRORS FOUND');
"
```
Expected: `SYNTAX OK`

`switchUtilityTab`/`utilityActiveTab`/`openUtilityTab(`/`utility-tabbtn-` 참조가 남아있지 않은지 확인:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
['switchUtilityTab', 'utilityActiveTab', 'openUtilityTab(', 'utility-tabbtn-'].forEach(term => {
  const count = (html.match(new RegExp(term.replace(/[.*+?^\${}()|[\]\\\\]/g,'\\\\\$&'), 'g')) || []).length;
  console.log(term, ':', count);
});
"
```
Expected: 전부 `0`

- [ ] **Step 6: Commit**

```bash
git add index.html docs/superpowers/plans/2026-07-15-utility-panel.md
git commit -m "refactor: 유틸리티 패널을 탭 방식에서 상중하단 스택 레이아웃으로 변경"
```

> 참고: Task 8(계산기 키보드 단축키)은 아직 구현 전이므로, 그 태스크를 시작할 때 원래 스펙의 `if (!utilityPanelOpen || utilityActiveTab !== 'calc') return;` 가드를 `if (!utilityPanelOpen) return;`로 바꿔야 한다(더 이상 "활성 탭" 개념이 없고 계산기 섹션이 항상 보이기 때문). 아래 Task 8 본문은 이미 이 변경을 반영해 수정해 두었다.

---

### Task 5: 리사이즈 핸들 (폭 조절)

**Files:**
- Modify: `index.html` — Task 4에서 추가한 함수들 바로 다음에 추가

- [ ] **Step 1: 드래그 리사이즈 로직 추가**

`initUtilityPanelState()` 함수 정의 바로 다음에 추가한다:

이 프로젝트의 유일한 `<script>` 블록은 `index.html:1921`에서 시작해 모든 HTML 마크업(`#utility-resize-handle` 포함, Task 2에서 `index.html:1118` 부근에 추가됨) 뒤에 위치하므로, 스크립트 실행 시점에 이미 해당 엘리먼트가 DOM에 존재한다. 별도의 `DOMContentLoaded` 대기 없이 바로 `getElementById`로 참조할 수 있다.

```js
(function initUtilityResizeHandle() {
  let dragging = false;
  const handle = document.getElementById('utility-resize-handle');
  if (handle) handle.addEventListener('mousedown', (e) => { dragging = true; e.preventDefault(); });
  document.addEventListener('mousemove', (e) => {
    if (!dragging) return;
    const drawer = document.getElementById('utility-drawer');
    let w = window.innerWidth - e.clientX;
    w = Math.max(280, Math.min(560, w));
    drawer.style.width = w + 'px';
    if (utilityPanelOpen) document.getElementById('utility-tab').style.right = w + 'px';
  });
  document.addEventListener('mouseup', () => {
    if (!dragging) return;
    dragging = false;
    const w = document.getElementById('utility-drawer').offsetWidth;
    try { localStorage.setItem('utilityPanelWidth', String(w)); } catch(e) {}
  });
})();
```

- [ ] **Step 2: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: 유틸리티 패널 폭 드래그 리사이즈 추가"
```

---

### Task 6: 계산기 — 수식 파서 (사칙연산 + 괄호)

**Files:**
- Modify: `index.html` — Task 5 코드 바로 다음에 추가

- [ ] **Step 1: `evalExpr` 파서 함수 추가**

```js
function evalExpr(rawStr) {
  const str = rawStr.replace(/\s+/g, '');
  if (!str) throw new Error('empty expression');
  let i = 0;
  function peek() { return str[i]; }
  function consumeNumber() {
    const start = i;
    while (i < str.length && /[0-9.]/.test(str[i])) i++;
    if (start === i) throw new Error('invalid number at ' + i);
    return parseFloat(str.slice(start, i));
  }
  function parseFactor() {
    if (peek() === '(') {
      i++;
      const v = parseExprInner();
      if (peek() !== ')') throw new Error('missing )');
      i++;
      return v;
    }
    if (peek() === '-') { i++; return -parseFactor(); }
    if (peek() === '+') { i++; return parseFactor(); }
    return consumeNumber();
  }
  function parseTerm() {
    let v = parseFactor();
    while (peek() === '×' || peek() === '÷' || peek() === '*' || peek() === '/') {
      const op = str[i]; i++;
      const rhs = parseFactor();
      if (op === '×' || op === '*') v *= rhs;
      else {
        if (rhs === 0) throw new Error('divide by zero');
        v /= rhs;
      }
    }
    return v;
  }
  function parseExprInner() {
    let v = parseTerm();
    while (peek() === '+' || peek() === '-') {
      const op = str[i]; i++;
      const rhs = parseTerm();
      v = op === '+' ? v + rhs : v - rhs;
    }
    return v;
  }
  const result = parseExprInner();
  if (i !== str.length) throw new Error('unexpected trailing input at ' + i);
  if (!isFinite(result)) throw new Error('not finite');
  return result;
}
```

- [ ] **Step 2: 파서 동작을 node로 수동 검증**

이 프로젝트에는 테스트 프레임워크가 없으므로, `<script>` 블록에서 `evalExpr` 함수 본문만 추출해 node로 직접 검증한다:

```bash
node -e "
$(node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const m = html.match(/function evalExpr[\s\S]*?\n}\n/);
console.log(m[0]);
")
const cases = [
  ['1+2×3', 7],
  ['(1+2)×3', 9],
  ['10÷2-3', 2],
  ['-5+10', 5],
  ['2×(3+(4-1))', 12],
];
let ok = true;
cases.forEach(([expr, expected]) => {
  const r = evalExpr(expr);
  if (r !== expected) { ok = false; console.log('FAIL', expr, 'got', r, 'expected', expected); }
});
try { evalExpr('1÷0'); ok = false; console.log('FAIL: divide by zero did not throw'); } catch(e) {}
console.log(ok ? 'CALC PARSER OK' : 'CALC PARSER FAILED');
"
```

Expected: `CALC PARSER OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: 계산기 수식 파서(evalExpr) 추가 - 사칙연산+괄호"
```

---

### Task 7: 계산기 — 버튼 연결 + 이력

**Files:**
- Modify: `index.html` — Task 6 코드 바로 다음에 추가

- [ ] **Step 1: 계산기 상태/버튼 핸들러 추가**

```js
let calcExpr = '';
let calcHistory = [];

function calcInput(ch) {
  calcExpr += ch;
  document.getElementById('calc-display').textContent = calcExpr || '0';
}

function calcBackspace() {
  calcExpr = calcExpr.slice(0, -1);
  document.getElementById('calc-display').textContent = calcExpr || '0';
}

function calcClear() {
  calcExpr = '';
  document.getElementById('calc-display').textContent = '0';
}

function calcEquals() {
  if (!calcExpr) return;
  const display = document.getElementById('calc-display');
  try {
    const result = evalExpr(calcExpr);
    display.textContent = String(result);
    calcHistory.unshift({ expr: calcExpr, result, ts: Date.now() });
    calcHistory = calcHistory.slice(0, 20);
    try { localStorage.setItem('calcHistory', JSON.stringify(calcHistory)); } catch(e) {}
    calcExpr = String(result);
    renderCalcHistory();
  } catch (e) {
    display.textContent = '오류';
    calcExpr = '';
  }
}

function calcClearHistory() {
  calcHistory = [];
  try { localStorage.removeItem('calcHistory'); } catch(e) {}
  renderCalcHistory();
}

function calcLoadHistory(idx) {
  const item = calcHistory[idx];
  if (!item) return;
  calcExpr = String(item.result);
  document.getElementById('calc-display').textContent = calcExpr;
}

function renderCalcHistory() {
  const list = document.getElementById('calc-history-list');
  if (!list) return;
  list.innerHTML = calcHistory.map((h, idx) =>
    `<div class="calc-history-item" onclick="calcLoadHistory(${idx})">${h.expr} = ${h.result}</div>`
  ).join('') || '<div class="calc-history-item" style="cursor:default">이력 없음</div>';
}

function initCalcHistory() {
  try {
    const saved = localStorage.getItem('calcHistory');
    calcHistory = saved ? JSON.parse(saved) : [];
  } catch(e) { calcHistory = []; }
  renderCalcHistory();
}
```

- [ ] **Step 2: 초기화 호출 추가**

Task 4에서 추가한 `initUtilityPanelState()` 함수 본문 맨 끝(닫는 `}` 직전)에 다음 줄을 추가한다:

```js
  initCalcHistory();
```

- [ ] **Step 3: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: 계산기 버튼 동작 및 이력(localStorage) 구현"
```

---

### Task 8: 계산기 — 키보드 단축키

**Files:**
- Modify: `index.html` — Task 7 코드 바로 다음에 추가

- [ ] **Step 1: 키보드 리스너 추가**

```js
document.addEventListener('keydown', function(e) {
  if (!utilityPanelOpen) return;
  const calcBasicView = document.getElementById('calc-basic-view');
  if (!calcBasicView || calcBasicView.classList.contains('hidden')) return;
  const active = document.activeElement;
  const isForeignInput = active && (active.tagName === 'INPUT' || active.tagName === 'TEXTAREA') && !calcBasicView.contains(active);
  if (isForeignInput) return;

  if (e.key === 'Enter' || e.code === 'NumpadEnter') { e.preventDefault(); calcEquals(); return; }
  if (e.key === 'Backspace') { e.preventDefault(); calcBackspace(); return; }
  if (e.key === 'Escape') { e.preventDefault(); calcClear(); return; }

  const keyMap = {
    '0':'0','1':'1','2':'2','3':'3','4':'4','5':'5','6':'6','7':'7','8':'8','9':'9',
    'Numpad0':'0','Numpad1':'1','Numpad2':'2','Numpad3':'3','Numpad4':'4','Numpad5':'5','Numpad6':'6','Numpad7':'7','Numpad8':'8','Numpad9':'9',
    'NumpadDecimal':'.',
    'NumpadAdd':'+', 'NumpadSubtract':'-', 'NumpadMultiply':'×', 'NumpadDivide':'÷'
  };
  const directKeys = { '.':'.', '(':'(', ')':')', '+':'+', '-':'-', '*':'×', '/':'÷' };

  const mapped = keyMap[e.code] || keyMap[e.key] || directKeys[e.key];
  if (mapped) { e.preventDefault(); calcInput(mapped); }
});
```

> **개정:** 원래 `const mapped = keyMap[e.code] || directKeys[e.key];`로 작성했으나, 상단 숫자키(예: `5`)는 `e.code`가 `"Digit5"`라 `keyMap`에 매칭되지 않고 `directKeys`에도 숫자가 없어 전혀 입력되지 않는 버그가 있었다. `keyMap[e.key]`를 중간에 추가해(이미 `keyMap`에 있는 `'0'`~`'9'` 항목이 상단 숫자키의 `e.key`와 매칭됨) 수정. 위 코드에 이미 반영함.

- [ ] **Step 2: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: 계산기 키보드(넘버패드+상단 숫자키) 연동"
```

---

### Task 9: 단위환산 서브탭 + 서브탭 전환

**Files:**
- Modify: `index.html` — Task 8 코드 바로 다음에 추가

- [ ] **Step 1: 서브탭 전환 함수 + 환산 함수 추가**

```js
function switchCalcSubtab(tab) {
  document.getElementById('calc-subtab-basic').classList.toggle('active', tab === 'basic');
  document.getElementById('calc-subtab-unit').classList.toggle('active', tab === 'unit');
  document.getElementById('calc-basic-view').classList.toggle('hidden', tab !== 'basic');
  document.getElementById('calc-unit-view').classList.toggle('hidden', tab !== 'unit');
}

function convUnit(rawVal) {
  const dir = document.getElementById('conv-direction').value;
  const n = parseFloat(rawVal);
  const out = document.getElementById('conv-result');
  if (isNaN(n)) { out.textContent = ''; return; }
  const PYEONG_TO_M2 = 3.305785;
  const result = dir === 'py2m2' ? n * PYEONG_TO_M2 : n / PYEONG_TO_M2;
  out.textContent = result.toFixed(2) + (dir === 'py2m2' ? ' ㎡' : ' 평');
}
```

- [ ] **Step 2: 기본 서브탭 활성화 상태 초기화**

Task 4의 `initUtilityPanelState()` 함수 본문에서 `initCalcHistory();` 다음 줄에 추가:

```js
  switchCalcSubtab('basic');
```

- [ ] **Step 3: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

수동 계산 검증(선택):
```bash
node -e "console.log((10 * 3.305785).toFixed(2), (10 / 3.305785).toFixed(2))"
```
Expected: `33.06 3.03`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: 계산기 평↔㎡ 단위환산 서브탭 추가"
```

---

### Task 10: 정리 + 수동 검증

**Files:**
- Modify: `index.html` (필요시 정리만)

- [ ] **Step 1: 남은 구식 참조 확인**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
['chat-toggle-btn','chatbot-panel','toggleChat(', 'chatOpen'].forEach(term => {
  const count = (html.match(new RegExp(term.replace(/[.*+?^\${}()|[\]\\\\]/g,'\\\\\$&'), 'g')) || []).length;
  console.log(term, ':', count);
});
"
```

Expected: 전부 `0` (단, `toggleChat(` 은 이 플랜에서 완전히 제거했으므로 0이어야 함)

- [ ] **Step 2: 최종 문법 검증**

Run: `node -e` 문법 체크 명령 (Task 1 참고)
Expected: `SYNTAX OK`

- [ ] **Step 3: (선택) 수동 브라우저 검증**

이 프로젝트 컨벤션상 기본 생략 가능하지만, 이번 변경은 전 페이지에 영향을 주는 전역 UI라 한 번은 확인 권장:

1. 아무 페이지에서나 우측 가장자리의 세로 손잡이("메모·계산기·AI") 클릭 → 드로어가 슬라이드로 열리는지 확인
2. 메모 탭에서 텍스트 입력 → "입력 중..." → "저장됨 ✓" 표시 확인, 새로고침 후 값 유지 확인
3. 계산기 탭 "일반" 서브탭: `(1+2)×3=` 버튼 클릭 시 9 표시, 이력에 추가되는지 확인. `1÷0=` 시 "오류" 표시 확인
4. 계산기 탭에서 키보드로 `12+8` 입력 후 Enter → 20 표시 확인 (단, 다른 입력창에 포커스가 없을 때만 동작해야 함 — 메모 textarea에 포커스 두고 숫자 입력 시 계산기에 반영되지 않아야 함)
5. "단위환산" 서브탭에서 10 입력 → 33.06 ㎡ 표시 확인, 방향 토글 후 재확인
6. AI챗봇 탭에서 기존처럼 메시지 전송 가능한지 확인 (API 키 설정 상태에 따라 정상 동작 또는 안내 메시지)
7. 드로어 좌측 테두리를 드래그해 폭이 바뀌는지, 새로고침 후 폭이 유지되는지 확인
8. 사이드바 하단 "AI 챗봇" 버튼 클릭 시 드로어가 열리고 AI챗봇 탭으로 바로 전환되는지 확인
9. 대시보드에서 기존 "개인 메모장" 카드가 사라졌는지 확인

- [ ] **Step 4: Commit (정리 사항이 있는 경우에만)**

```bash
git add index.html
git commit -m "chore: 유틸리티 패널 구현 마무리 정리"
```

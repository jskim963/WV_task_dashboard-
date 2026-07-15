# @멘션함 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 업무현황 코멘트와 센터별 특이사항에 `@이름`을 입력하면(자동완성 지원), 그 사람이 대시보드에서 자신이 언급된 내역을 시간순으로 확인하고 클릭해서 원본으로 이동할 수 있게 한다.

**Architecture:** 별도 Firestore 컬렉션을 새로 만들지 않고, 이미 로드되어 있는 `DB.tasks`/`DB.centers`의 코멘트 텍스트에서 `@이름` 패턴을 매 렌더링 시점에 계산(derive)한다. `@` 자동완성은 document 레벨 이벤트 위임으로 구현해 동적으로 렌더링되는 여러 입력창(`#comment-input`, `#ops-note-input-*`)에 개별 바인딩 없이 동작하게 한다.

**Tech Stack:** Vanilla JS (기존 `Pages.dashboard`/`Pages.overview` IIFE 모듈 패턴), 정규식 기반 텍스트 매칭, 기존 Firestore `tasks`/`centers` 컬렉션(스키마에 `ts` 필드만 추가).

---

## 실행 순서 / 의존성

이 플랜은 **`2026-07-15-utility-panel.md` 플랜의 Task 3까지 완료된 상태**를 전제로 한다. 그 플랜에서 대시보드의 "개인 메모장" 카드(Panel 5, `index.html:495-502`)가 이미 제거되었고, 이 플랜의 Task 3에서 그 빈자리에 "멘션함" 카드를 채운다.

먼저 `index.html`에서 `495` 근처를 열어 Panel 5가 이미 제거되어 있는지(= Panel 4 "내 업무 목록" 바로 다음이 곧바로 "Panel 6: 오늘 일정" 주석인지) 확인한 뒤 시작한다. 아직 제거되지 않았다면 유틸리티 패널 플랜의 Task 3을 먼저 실행한다.

## 사전 확인 사항 (참고용)

- 테스트 프레임워크 없음. `<script>` 블록 문법 오류 확인용 명령(모든 태스크 마지막에 실행):
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
- 브라우저 프리뷰 검증은 이 프로젝트 컨벤션상 기본 생략(요청 시에만 수행).

---

### Task 1: 코멘트 저장 시 정렬용 타임스탬프 추가

**Files:**
- Modify: `index.html:2660-2672` (`addComment()`, `Pages.dashboard` 내부)

- [ ] **Step 1: `addComment()`에 숫자 타임스탬프 `ts` 필드 추가**

`index.html:2660-2672`의 다음 함수를:

```js
  function addComment() {
    const t = DB.tasks.find(t => t.id === detailTaskId);
    const input = document.getElementById('comment-input');
    const text = input.value.trim();
    if (!t || !text) return;
    const now = new Date();
    const timeStr = now.toLocaleDateString('ko-KR') + ' ' + now.toLocaleTimeString('ko-KR', {hour:'2-digit', minute:'2-digit'});
    t.comments.push({author: DB.myProfile.name || '나', text, time: timeStr});
    saveDB();
    input.value = '';
    renderComments(detailTaskId);
    updateNotifBadge();
  }
```

다음으로 교체한다(추가된 줄은 `ts: now.getTime()` 하나뿐, 나머지는 동일):

```js
  function addComment() {
    const t = DB.tasks.find(t => t.id === detailTaskId);
    const input = document.getElementById('comment-input');
    const text = input.value.trim();
    if (!t || !text) return;
    const now = new Date();
    const timeStr = now.toLocaleDateString('ko-KR') + ' ' + now.toLocaleTimeString('ko-KR', {hour:'2-digit', minute:'2-digit'});
    t.comments.push({author: DB.myProfile.name || '나', text, time: timeStr, ts: now.getTime()});
    saveDB();
    input.value = '';
    renderComments(detailTaskId);
    updateNotifBadge();
  }
```

> `addOperationNote()`(`index.html:4075-4092`)는 이미 `date: Date.now()` 숫자 필드를 갖고 있으므로 변경하지 않는다.

- [ ] **Step 2: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: 업무 코멘트에 정렬용 타임스탬프(ts) 필드 추가"
```

---

### Task 2: @이름 자동완성

**Files:**
- Modify: `index.html` — CSS 블록(`index.html:196-200` 부근, `#assignee-dropdown-panel` 스타일 다음), HTML(`index.html:1108` 부근, `<!-- ===== NOTIFICATION PANEL ===== -->` 앞), JS(전역 함수 영역)

- [ ] **Step 1: 멘션 드롭다운 CSS 추가**

`index.html:200`(`.assignee-cb-row input[type=checkbox] { ... }` 줄) 바로 다음에 추가한다:

```css
  #mention-dropdown { position:fixed; background:#fff; border:1px solid #e2e8f0; border-radius:12px; box-shadow:0 8px 32px rgba(0,0,0,0.14); z-index:600; max-height:200px; overflow-y:auto; padding:6px 0; }
  #mention-dropdown.hidden { display:none; }
```

- [ ] **Step 2: 멘션 드롭다운 컨테이너 HTML 추가**

`index.html:1108`(`<!-- ===== NOTIFICATION PANEL ===== -->` 주석) 바로 앞에 추가한다:

```html
<!-- ===== MENTION AUTOCOMPLETE ===== -->
<div id="mention-dropdown" class="hidden"></div>

```

- [ ] **Step 3: 자동완성 로직 전역 함수 추가**

전역 함수 영역(예: `index.html`의 `updateNotifBadge` 함수 정의 부근, 또는 아무 전역 함수 다음)에 다음을 추가한다:

```js
let mentionDropdownTarget = null;

function initMentionAutocomplete() {
  document.addEventListener('input', function(e) {
    const el = e.target;
    const isMentionable = el.id === 'comment-input' || (el.id && el.id.startsWith('ops-note-input-'));
    if (!isMentionable) return;
    const cursorPos = el.selectionStart;
    const textBefore = el.value.slice(0, cursorPos);
    const match = textBefore.match(/@([가-힣a-zA-Z0-9]*)$/);
    if (!match) { hideMentionDropdown(); return; }
    const query = match[1];
    const candidates = (DB.members || []).filter(m => m.name && m.name.startsWith(query));
    if (candidates.length === 0) { hideMentionDropdown(); return; }
    showMentionDropdown(el, candidates, textBefore.length - match[0].length, cursorPos);
  });

  document.addEventListener('click', function(e) {
    const dd = document.getElementById('mention-dropdown');
    if (dd && !dd.classList.contains('hidden') && !dd.contains(e.target) && e.target.id !== 'comment-input' && !(e.target.id||'').startsWith('ops-note-input-')) {
      hideMentionDropdown();
    }
  });
}

function escapeMentionName(s) {
  return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/'/g,'&#39;');
}

function showMentionDropdown(el, members, matchStart, cursorPos) {
  const dd = document.getElementById('mention-dropdown');
  mentionDropdownTarget = { el, matchStart, cursorPos };
  dd.innerHTML = members.map(m =>
    `<div class="assignee-cb-row" onmousedown="event.preventDefault();selectMention('${escapeMentionName(m.name)}')">${escapeMentionName(m.name)}</div>`
  ).join('');
  const rect = el.getBoundingClientRect();
  dd.style.left = rect.left + 'px';
  dd.style.top = (rect.bottom + 4) + 'px';
  dd.style.minWidth = Math.max(rect.width, 140) + 'px';
  dd.classList.remove('hidden');
}

function hideMentionDropdown() {
  const dd = document.getElementById('mention-dropdown');
  if (dd) dd.classList.add('hidden');
  mentionDropdownTarget = null;
}

function selectMention(name) {
  if (!mentionDropdownTarget) return;
  const { el, matchStart, cursorPos } = mentionDropdownTarget;
  const before = el.value.slice(0, matchStart);
  const after = el.value.slice(cursorPos);
  el.value = before + '@' + name + ' ' + after;
  const newPos = (before + '@' + name + ' ').length;
  el.focus();
  el.setSelectionRange(newPos, newPos);
  hideMentionDropdown();
}
```

`input`/`click` 이벤트를 `document`에 위임으로 등록하기 때문에, `#comment-input`(모달 안에서 고정 1개)과 `#ops-note-input-${centerId}`(센터마다 동적으로 렌더링되는 여러 개)에 각각 별도로 바인딩할 필요가 없다.

- [ ] **Step 4: 앱 시작 시 `initMentionAutocomplete()` 호출**

`index.html:8373-8383`의 `initApp()` 함수(앱 부트스트랩 진입점)에서 `setupListeners();`(`index.html:8382`) 바로 다음 줄에 추가한다:

```js
  setupListeners();
  initMentionAutocomplete();
  checkAuth();
```

이 함수는 `document`에 위임 리스너만 등록하므로 로그인 여부와 무관하게 앱 시작 시 한 번만 호출하면 된다.

- [ ] **Step 5: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: 업무 코멘트/센터 특이사항 입력창에 @이름 자동완성 추가"
```

---

### Task 3: 멘션 계산 헬퍼 + 대시보드 멘션함 카드

**Files:**
- Modify: `index.html:495` 부근(HTML, Panel 5 자리 — 유틸리티 패널 플랜에서 이미 비워짐), `index.html:5770-5933` 부근(`Pages.overview` IIFE)

- [ ] **Step 1: 대시보드에 멘션함 카드 마크업 추가**

`index.html`에서 "Panel 4: 내 업무 목록" 블록(`<!-- Panel 4: 내 업무 목록 -->`으로 시작, `</div>`로 끝남) 바로 다음, "Panel 6: 오늘 일정" 주석 바로 앞에 다음을 추가한다(유틸리티 패널 플랜 Task 3에서 원래 있던 Panel 5 메모 카드를 제거한 바로 그 자리):

```html
        <!-- Panel 5: 멘션함 -->
        <div class="bg-white rounded-xl border border-outline-variant shadow-sm p-5 flex flex-col">
          <div class="flex items-center justify-between mb-3">
            <h2 class="font-bold text-sm text-on-surface flex items-center gap-1.5"><span class="material-symbols-outlined text-purple-500 text-base">alternate_email</span>멘션함</h2>
          </div>
          <div id="ov-mentions" class="space-y-1.5 max-h-48 overflow-y-auto"></div>
        </div>
```

- [ ] **Step 2: `Pages.overview` IIFE에 멘션 계산/렌더 함수 추가**

`index.html:5931`(`render()` 함수의 닫는 `}`) 바로 앞에 다음 호출을 추가한다:

```js
    renderMentions();
```

`index.html:5932`(`return { render };`) 바로 앞에 다음 함수들을 추가한다:

```js
  function escapeHtml(s) {
    return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  }

  function buildMentionRegex() {
    const names = (DB.members || []).map(m => m.name).filter(Boolean);
    if (names.length === 0) return null;
    const sorted = [...new Set(names)].sort((a, b) => b.length - a.length);
    const escaped = sorted.map(n => n.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'));
    return new RegExp('@(' + escaped.join('|') + ')', 'g');
  }

  function getMyMentions() {
    if (!currentUser) return [];
    const regex = buildMentionRegex();
    if (!regex) return [];
    const results = [];

    (DB.tasks || []).forEach(t => {
      (t.comments || []).forEach(c => {
        regex.lastIndex = 0;
        let m;
        while ((m = regex.exec(c.text)) !== null) {
          if (m[1] === currentUser.name) {
            results.push({ sourceType: 'task', sourceId: t.id, snippet: c.text, authorName: c.author || '', ts: c.ts || 0 });
            break;
          }
        }
      });
    });

    (DB.centers || []).forEach(center => {
      (center.operationNotes || []).forEach(n => {
        regex.lastIndex = 0;
        let m;
        while ((m = regex.exec(n.text)) !== null) {
          if (m[1] === currentUser.name) {
            results.push({ sourceType: 'center', sourceId: center.id, snippet: n.text, authorName: '', ts: n.date || 0 });
            break;
          }
        }
      });
    });

    return results.sort((a, b) => b.ts - a.ts).slice(0, 20);
  }

  function formatMentionTime(ts) {
    if (!ts) return '';
    const diffMs = Date.now() - ts;
    const min = Math.floor(diffMs / 60000);
    if (min < 1) return '방금 전';
    if (min < 60) return min + '분 전';
    const hr = Math.floor(min / 60);
    if (hr < 24) return hr + '시간 전';
    const day = Math.floor(hr / 24);
    return day + '일 전';
  }

  function renderMentions() {
    const el = document.getElementById('ov-mentions');
    if (!el) return;
    const mentions = getMyMentions();
    if (mentions.length === 0) {
      el.innerHTML = '<p class="text-xs text-secondary text-center py-4">아직 멘션된 내역이 없습니다</p>';
      return;
    }
    el.innerHTML = mentions.map(m => `
      <div class="flex items-start gap-2 cursor-pointer hover:bg-surface-container-low rounded-lg px-2 py-1.5 -mx-2" onclick="goToMention('${m.sourceType}', ${m.sourceId})">
        <span class="material-symbols-outlined text-sm ${m.sourceType === 'task' ? 'text-green-600' : 'text-orange-500'} flex-shrink-0 mt-0.5">${m.sourceType === 'task' ? 'work' : 'warning'}</span>
        <div class="min-w-0 flex-1">
          <p class="text-xs text-on-surface truncate">${escapeHtml(m.snippet)}</p>
          <p class="text-[11px] text-secondary">${escapeHtml(m.authorName)} · ${formatMentionTime(m.ts)}</p>
        </div>
      </div>
    `).join('');
  }
```

> `escapeHtml`는 프로젝트 컨벤션대로 `Pages.overview` 안에 로컬로 새로 정의한다(다른 모듈의 동일 함수와 중복되지만 기존 컨벤션을 따름).

- [ ] **Step 3: 원본으로 이동하는 전역 함수 추가**

`Pages.overview = (() => { ... })();` 블록 바로 다음(예: `index.html:5935` `// --- Overview shim ---` 주석 다음)에 추가한다:

```js
function goToMention(sourceType, sourceId) {
  if (sourceType === 'task') {
    showPage('dashboard');
    setTimeout(() => showTaskDetail(sourceId), 100);
  } else if (sourceType === 'center') {
    showPage('centers');
    setTimeout(() => openCenterDetail(sourceId), 100);
  }
}
```

- [ ] **Step 4: 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: 대시보드에 @멘션함 카드 추가 (업무/센터 코멘트에서 계산)"
```

---

### Task 4: 정리 + 수동 검증

**Files:**
- Modify: `index.html` (필요시 정리만)

- [ ] **Step 1: 정규식 매칭 로직 수동 검증**

```bash
node -e "
function buildMentionRegex(names) {
  const sorted = [...new Set(names)].sort((a, b) => b.length - a.length);
  const escaped = sorted.map(n => n.replace(/[.*+?^\${}()|[\]\\\\]/g, '\\\\\$&'));
  return new RegExp('@(' + escaped.join('|') + ')', 'g');
}
const names = ['김철수', '김철', '이영희'];
const regex = buildMentionRegex(names);
const text = '@김철수 님, @이영희 님 확인 부탁드려요. @김철 님도요.';
let m, found = [];
while ((m = regex.exec(text)) !== null) found.push(m[1]);
console.log(found.join(','));
console.log(found.join(',') === '김철수,이영희,김철' ? 'MENTION REGEX OK' : 'MENTION REGEX FAILED');
"
```

Expected: `MENTION REGEX OK` — 특히 `김철수`가 `김철`의 부분 문자열이지만 긴 이름이 먼저 매칭되어 `@김철수`가 `@김철`로 잘못 잘리지 않는지 확인.

- [ ] **Step 2: 최종 문법 검증**

Run: `node -e` 문법 체크 명령
Expected: `SYNTAX OK`

- [ ] **Step 3: (선택) 수동 브라우저 검증**

1. 업무현황 코멘트 입력창에서 `@` 입력 → 팀원 자동완성 드롭다운이 뜨는지, 선택 시 이름이 삽입되는지 확인
2. 센터별 특이사항 입력창에서도 동일하게 동작하는지 확인 (동적으로 렌더링된 센터를 열어서 확인)
3. 자신을 멘션한 코멘트를 작성한 뒤(다른 계정으로 로그인해 본인을 멘션하거나, 테스트 데이터로 확인), 대시보드 "멘션함" 카드에 나타나는지 확인
4. 멘션함 항목 클릭 시 해당 업무/센터 상세로 이동하는지 확인
5. 멘션이 없는 상태에서 "아직 멘션된 내역이 없습니다" 문구가 뜨는지 확인

- [ ] **Step 4: Commit (정리 사항이 있는 경우에만)**

```bash
git add index.html
git commit -m "chore: 멘션함 구현 마무리 정리"
```

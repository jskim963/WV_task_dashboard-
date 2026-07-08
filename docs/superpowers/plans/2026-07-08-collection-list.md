# 취합리스트 기능 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 회신 취합 현황을 대시보드 안에서 등록·관리하고, 팀원별 제출 여부를 체크하고, 미제출자에게 Webex로 알림을 보낼 수 있는 "취합리스트" 페이지를 추가한다.

**Architecture:** 기존 `index.html` 단일 파일(정적 HTML + Firestore + Firebase Storage) 구조를 그대로 따른다. `drafts`/`projects` 페이지와 동일하게 `Pages.collections` 모듈 + 전용 Firestore 컬렉션(`collections`) + 문서 단위 실시간 리스너(onSnapshot) 패턴을 사용한다.

**Tech Stack:** Vanilla JS, Firebase Firestore, Firebase Storage, Tailwind(CDN), Webex REST API(`POST /v1/messages`, 브라우저에서 직접 호출)

**참고 스펙:** [docs/superpowers/specs/2026-07-08-collection-list-design.md](../specs/2026-07-08-collection-list-design.md)

---

## 테스트/검증 방식에 대한 안내 (중요)

이 프로젝트는 `index.html` 단일 파일로 구성된 정적 웹앱이며 **테스트 프레임워크가 없다** (package.json, 테스트 러너 없음, 전부 인라인 `<script>` + DOM 조작). 따라서 이 계획은 표준 "실패하는 테스트 → 구현 → 통과" TDD 사이클 대신, 이 코드베이스에서 실제로 써온 방식을 따른다:

1. 코드 작성 (HTML + JS)
2. **문법 검증** — 아래 명령으로 인라인 스크립트에 문법 오류가 없는지 확인 (매 태스크 공통, 브라우저 없이 실행 가능):
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
   `SYNTAX OK`가 출력되어야 통과.
3. **구조 검증** — `grep`으로 새로 추가한 함수/HTML id가 실제로 파일에 존재하는지 확인 (각 태스크에 구체적 명령 명시)
4. 커밋

이 프로젝트의 기존 컨벤션(사용자 메모리 기준)은 브라우저 프리뷰 검증을 기본적으로 생략하는 것이다. 이 계획도 동일하게 따르되, **Task 7**에 전체 기능을 한 번에 확인하는 선택적 브라우저 스모크 테스트를 마지막에 배치해, 사용자가 원할 때만 수행할 수 있게 한다.

---

## 파일 구조

이 기능은 새 파일을 만들지 않고 기존 `index.html` 한 파일만 수정한다 (기존 `drafts`/`projects`/`library` 기능도 전부 같은 파일 안에 있음 — 코드베이스 컨벤션).

수정 위치 (전부 `index.html` 내부):
- DB 초기 스키마 + `FS_COLLS` + 실시간 리스너 (약 1829~1924행)
- 사이드바 네비게이션 (약 359~390행)
- 페이지 shell(`#page-collections`) (약 890행 부근, `#page-drafts` 뒤)
- 등록/수정 모달 + 상세 모달 HTML (약 1400행 부근, `#modal-add-task` 뒤)
- `Pages.collections` 모듈 + 전역 shim 함수 (약 7860행 부근, drafts shim 뒤)
- 설정 페이지 Webex 입력 활성화 + `loadSettings()`/`saveWebexSettings()` (약 976행, 4510행)

---

### Task 1: 데이터 계층 — DB 스키마, FS_COLLS, 실시간 리스너

**Files:**
- Modify: `index.html:1829` (`FS_COLLS` 배열)
- Modify: `index.html:1834-1848` (`DB` 초기값)
- Modify: `index.html:1894-1924` (`setupListeners()`)

- [ ] **Step 1: `FS_COLLS`에 `'collections'` 추가**

`index.html`에서 다음을 찾는다:
```js
const FS_COLLS = ['tasks', 'members', 'centers', 'events', 'notices', 'users', 'memos', 'library', 'projects', 'drafts', 'draft_templates'];
```
다음으로 교체:
```js
const FS_COLLS = ['tasks', 'members', 'centers', 'events', 'notices', 'users', 'memos', 'library', 'projects', 'drafts', 'draft_templates', 'collections'];
```

- [ ] **Step 2: `DB` 초기 스키마에 `collections: []` 추가**

다음을 찾는다:
```js
let DB = {
  tasks: [],
  members: [],
  centers: [],
  events: [],
  users: [],
  notices: [],
  todos: [],
  memos: [],
  library: [],
  projects: [],
  drafts: [],
  draft_templates: [],
  myProfile: { name: '나', email: '', rank: '' },
};
```
다음으로 교체:
```js
let DB = {
  tasks: [],
  members: [],
  centers: [],
  events: [],
  users: [],
  notices: [],
  todos: [],
  memos: [],
  library: [],
  projects: [],
  drafts: [],
  draft_templates: [],
  collections: [],
  myProfile: { name: '나', email: '', rank: '' },
};
```

- [ ] **Step 3: `setupListeners()`에 `collections` 실시간 리스너 추가**

`drafts`/`draft_templates` 리스너 바로 뒤(함수 닫는 `}` 직전)에 다음을 찾는다:
```js
  db.collection('draft_templates').onSnapshot(snapshot => {
    DB.draft_templates = snapshot.docs.map(d => d.data());
    refreshCurrentPage();
  }, err => console.error('onSnapshot 오류 (draft_templates):', err));
}
```
다음으로 교체:
```js
  db.collection('draft_templates').onSnapshot(snapshot => {
    DB.draft_templates = snapshot.docs.map(d => d.data());
    refreshCurrentPage();
  }, err => console.error('onSnapshot 오류 (draft_templates):', err));

  db.collection('collections').onSnapshot(snapshot => {
    DB.collections = snapshot.docs.map(d => d.data());
    refreshCurrentPage();
  }, err => console.error('onSnapshot 오류 (collections):', err));
}
```

- [ ] **Step 4: 문법 검증**

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

- [ ] **Step 5: 구조 검증**

Run: `grep -n "collections" index.html | grep -E "FS_COLLS|DB = \{|onSnapshot 오류 \(collections\)"`
Expected: 3개 라인 출력 (FS_COLLS 배열, DB 초기값 근처는 `collections: []`로 직접 안 잡힐 수 있으니 아래 명령도 함께 실행)

Run: `grep -n "collections: \[\]" index.html`
Expected: 1줄 출력

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 데이터 계층 추가 (collections 컬렉션)

FS_COLLS/DB 초기 스키마/실시간 리스너에 collections 컬렉션을 등록.
drafts 컬렉션과 동일한 패턴(초기 로드 + onSnapshot)을 따른다.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: 사이드바 네비게이션 + 페이지 shell + Pages.collections 모듈 스켈레톤

**Files:**
- Modify: `index.html` (사이드바 nav, `#page-drafts` 뒤, drafts shim 함수 뒤)

- [ ] **Step 1: 사이드바에 "취합리스트" 메뉴 추가**

다음을 찾는다:
```html
      <div class="nav-item" onclick="showPage('drafts')" id="nav-drafts">
        <span class="material-symbols-outlined">rate_review</span> 검토방
      </div>
      <div class="nav-item" onclick="showPage('library')" id="nav-library">
```
다음으로 교체:
```html
      <div class="nav-item" onclick="showPage('drafts')" id="nav-drafts">
        <span class="material-symbols-outlined">rate_review</span> 검토방
      </div>
      <div class="nav-item" onclick="showPage('collections')" id="nav-collections">
        <span class="material-symbols-outlined">checklist</span> 취합리스트
      </div>
      <div class="nav-item" onclick="showPage('library')" id="nav-library">
```

- [ ] **Step 2: 페이지 shell 추가**

`#page-drafts`가 끝나는 지점(`#page-library` 시작 직전)을 찾는다:
```html
  <div id="drafts-empty" class="hidden flex flex-col items-center justify-center py-20 text-secondary">
    <span class="material-symbols-outlined text-5xl mb-3 opacity-40">rate_review</span>
    <p class="text-sm">아직 기안서가 없습니다</p>
    <button onclick="openAddDraftModal()" class="mt-4 text-primary text-sm hover:underline">+ 첫 기안서 작성하기</button>
  </div>
</div>
    <div id="page-library" class="page">
```
다음으로 교체:
```html
  <div id="drafts-empty" class="hidden flex flex-col items-center justify-center py-20 text-secondary">
    <span class="material-symbols-outlined text-5xl mb-3 opacity-40">rate_review</span>
    <p class="text-sm">아직 기안서가 없습니다</p>
    <button onclick="openAddDraftModal()" class="mt-4 text-primary text-sm hover:underline">+ 첫 기안서 작성하기</button>
  </div>
</div>
<div id="page-collections" class="page">
  <div class="flex items-center justify-between mb-6">
    <div>
      <h1 class="text-xl lg:text-2xl font-bold text-on-surface" style="font-family:'Hanken Grotesk'">취합리스트</h1>
      <p class="text-sm text-secondary mt-0.5">회신 요청 취합 및 제출 현황 관리</p>
    </div>
    <button onclick="openAddCollectionModal()" class="flex items-center gap-2 bg-primary text-white px-4 py-2.5 rounded-lg text-sm font-semibold hover:bg-blue-700">
      <span class="material-symbols-outlined text-base">add</span> 취합건 등록
    </button>
  </div>
  <div class="flex gap-2 mb-4">
    <button class="coll-tab-btn" data-tab="진행중" onclick="switchCollectionTab('진행중')">진행중</button>
    <button class="coll-tab-btn" data-tab="완료" onclick="switchCollectionTab('완료')">완료</button>
  </div>
  <div class="bg-white rounded-xl border border-outline-variant shadow-sm overflow-x-auto">
    <table class="w-full text-left" style="min-width:800px">
      <thead class="bg-surface-container text-xs text-secondary">
        <tr>
          <th class="px-4 py-2.5 font-semibold">요청부서</th>
          <th class="px-4 py-2.5 font-semibold">구분</th>
          <th class="px-4 py-2.5 font-semibold">제목</th>
          <th class="px-4 py-2.5 font-semibold">회신기한</th>
          <th class="px-4 py-2.5 font-semibold">취합 담당자</th>
          <th class="px-4 py-2.5 font-semibold">제출현황</th>
          <th class="px-4 py-2.5 font-semibold">비고</th>
        </tr>
      </thead>
      <tbody id="coll-tbody"></tbody>
    </table>
  </div>
</div>
    <div id="page-library" class="page">
```

- [ ] **Step 3: `Pages.collections` 모듈 스켈레톤 추가 (render만 우선 구현)**

drafts shim 함수들이 끝나는 지점을 찾는다:
```js
function addDraftComment()                  { Pages.drafts.addComment(); }
function switchDraftMobileTab(tab) {
```
다음으로 교체 (drafts shim과 switchDraftMobileTab 사이에 새 모듈을 끼워 넣는다):
```js
function addDraftComment()                  { Pages.drafts.addComment(); }

// ============================================================
//  PAGE: COLLECTIONS (취합리스트)
// ============================================================
Pages.collections = (() => {
  let _tab = '진행중';
  let _collAttachments = [];

  function isFullyChecked(item) {
    return item.targetIds.length > 0 && item.targetIds.every(mid => item.submitted[String(mid)]?.checked);
  }

  function switchTab(tab) {
    _tab = tab;
    render();
  }

  function updateTabButtons() {
    document.querySelectorAll('.coll-tab-btn').forEach(b => {
      const isActive = b.dataset.tab === _tab;
      b.className = 'coll-tab-btn px-4 py-1.5 rounded-full text-sm font-medium transition-all ' +
        (isActive ? 'bg-primary text-white' : 'border border-outline-variant text-secondary hover:border-primary hover:text-primary');
    });
  }

  function render() {
    updateTabButtons();
    const tbody = document.getElementById('coll-tbody');
    if (!tbody) return;
    tbody.innerHTML = `<tr><td colspan="7" class="text-center text-sm text-secondary py-10">준비 중입니다</td></tr>`;
  }

  return { switchTab, render, isFullyChecked };
})();

function switchDraftMobileTab(tab) {
```

- [ ] **Step 4: 전역 shim 함수 추가 (라우팅용)**

drafts shim 블록의 맨 끝, `switchDraftMobileTab` 함수가 끝나는 지점(다음 섹션 주석 `// ---- Library global shims ----` 직전)을 찾는다:
```js
// ---- Library global shims ----
```
다음으로 교체:
```js
function switchCollectionTab(tab) { Pages.collections.switchTab(tab); }

// ---- Library global shims ----
```

- [ ] **Step 5: 문법 검증**

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

- [ ] **Step 6: 구조 검증**

Run: `grep -n 'id="nav-collections"\|id="page-collections"\|Pages.collections = \|function switchCollectionTab' index.html`
Expected: 4줄 출력, 전부 존재

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 네비게이션 및 페이지 shell 추가

사이드바 메뉴, 빈 페이지 레이아웃(탭+테이블), Pages.collections 모듈
스켈레톤을 추가한다. 목록 렌더링 로직은 이후 태스크에서 구현.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: 목록 화면 렌더링 (탭 필터 + 테이블)

**Files:**
- Modify: `index.html` (`Pages.collections` 모듈 내 `render` 함수)

- [ ] **Step 1: `render()` 함수를 실제 렌더링 로직으로 교체**

Task 2에서 넣은 다음 블록을 찾는다:
```js
  function render() {
    updateTabButtons();
    const tbody = document.getElementById('coll-tbody');
    if (!tbody) return;
    tbody.innerHTML = `<tr><td colspan="7" class="text-center text-sm text-secondary py-10">준비 중입니다</td></tr>`;
  }

  return { switchTab, render, isFullyChecked };
```
다음으로 교체:
```js
  function render() {
    updateTabButtons();
    const tbody = document.getElementById('coll-tbody');
    if (!tbody) return;
    const items = (DB.collections || []).filter(c => (c.status || '진행중') === _tab);
    items.sort((a, b) => _tab === '진행중'
      ? new Date(a.deadline) - new Date(b.deadline)
      : (b.createdAt || 0) - (a.createdAt || 0));
    if (!items.length) {
      tbody.innerHTML = `<tr><td colspan="7" class="text-center text-sm text-secondary py-10">${_tab === '진행중' ? '진행중인 취합건이 없습니다' : '완료된 취합건이 없습니다'}</td></tr>`;
      return;
    }
    const todayStr = formatDate(new Date());
    tbody.innerHTML = items.map(item => {
      const manager = DB.members.find(m => m.id === item.managerId);
      const checkedCount = item.targetIds.filter(mid => item.submitted[String(mid)]?.checked).length;
      const overdue = _tab === '진행중' && item.deadline < todayStr;
      const tagBadge = item.tag === '신규' ? '<span class="text-xs font-semibold text-blue-600">신규</span>'
        : item.tag === '중요' ? '<span class="text-xs font-semibold text-red-600">중요</span>' : '';
      return `<tr class="border-b border-outline-variant hover:bg-surface-container-low cursor-pointer" onclick="Pages.collections.openDetail(${item.id})">
        <td class="px-4 py-3 text-sm">${item.dept}</td>
        <td class="px-4 py-3">${tagBadge}</td>
        <td class="px-4 py-3 text-sm font-medium">${item.title}</td>
        <td class="px-4 py-3 text-sm ${overdue ? 'text-red-600 font-semibold' : ''}">${item.deadline}</td>
        <td class="px-4 py-3 text-sm">${manager ? manager.name : '-'}</td>
        <td class="px-4 py-3 text-sm">${checkedCount}/${item.targetIds.length}</td>
        <td class="px-4 py-3 text-xs text-secondary">${item.memo || ''}</td>
      </tr>`;
    }).join('');
  }

  return { switchTab, render, isFullyChecked, openDetail: (id) => {} };
```

> `openDetail`은 Task 5에서 구현한다. 지금은 클릭 시 에러가 나지 않도록 빈 함수로 노출해 둔다 (Task 4/5에서 실제 구현으로 교체).

- [ ] **Step 2: 문법 검증**

Run: (Task 1의 검증 명령과 동일)
Expected: `SYNTAX OK`

- [ ] **Step 3: 로직 검증 (Node로 순수 함수 부분만 시뮬레이션)**

DOM 없이 정렬/필터 로직만 별도로 검증한다:
```bash
node -e "
const DB = { collections: [
  { id: 1, status: '진행중', deadline: '2026-07-10', targetIds: [1,2], submitted: {'1':{checked:true},'2':{checked:false}} },
  { id: 2, status: '진행중', deadline: '2026-07-08', targetIds: [1], submitted: {'1':{checked:false}} },
  { id: 3, status: '완료', deadline: '2026-07-01', targetIds: [1], submitted: {'1':{checked:true}} },
] };
const items = DB.collections.filter(c => c.status === '진행중');
items.sort((a,b) => new Date(a.deadline) - new Date(b.deadline));
console.log(items.map(i=>i.id).join(',')); // 기한 빠른 순: 2,1 이어야 함
"
```
Expected: `2,1`

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 목록 테이블 렌더링 구현

진행중/완료 탭 필터링, 기한순 정렬, 제출현황(n/m) 표시, 기한 초과 강조 추가.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: 등록/수정 모달 (폼 + 첨부파일 업로드 + 저장)

**Files:**
- Modify: `index.html` (`#modal-add-task` 뒤에 모달 HTML 추가, `Pages.collections` 모듈에 함수 추가, drafts shim 뒤에 전역 shim 추가)

- [ ] **Step 1: 등록/수정 모달 HTML 추가**

`#modal-add-task`가 끝나는 지점을 찾는다:
```html
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-task').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container transition-all">취소</button>
      <button onclick="saveTask()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700 transition-all">저장</button>
    </div>
  </div>
</div>

<!-- Task Detail Modal -->
```
다음으로 교체:
```html
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-task').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container transition-all">취소</button>
      <button onclick="saveTask()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700 transition-all">저장</button>
    </div>
  </div>
</div>

<!-- Collection Add/Edit Modal -->
<div id="modal-add-collection" class="modal-overlay hidden" onclick="if(event.target===this)this.classList.add('hidden')" ontouchend="if(event.target===this){event.preventDefault();this.classList.add('hidden')}">
  <div class="modal-box" style="width:520px;max-height:90vh;overflow-y:auto;">
    <div class="flex items-center justify-between mb-5">
      <h3 class="text-lg font-bold text-on-surface" style="font-family:'Hanken Grotesk'" id="collection-modal-title">취합건 등록</h3>
      <button onclick="document.getElementById('modal-add-collection').classList.add('hidden')" class="text-secondary p-1 rounded-lg hover:bg-surface-container"><span class="material-symbols-outlined">close</span></button>
    </div>
    <input type="hidden" id="editing-collection-id">
    <div class="space-y-4">
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">요청부서 *</label>
        <input type="text" id="coll-dept" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="예: TLS기획팀">
      </div>
      <div class="grid grid-cols-2 gap-3">
        <div>
          <label class="block text-xs font-semibold text-secondary mb-1.5">구분</label>
          <select id="coll-tag" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm bg-white">
            <option value="">일반</option>
            <option value="신규">신규</option>
            <option value="중요">중요</option>
          </select>
        </div>
        <div>
          <label class="block text-xs font-semibold text-secondary mb-1.5">회신기한 *</label>
          <input type="date" id="coll-deadline" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm">
        </div>
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">제목 *</label>
        <input type="text" id="coll-title" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="취합 항목 제목">
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">취합 담당자 *</label>
        <select id="coll-manager" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm bg-white"></select>
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">대상 팀원 *</label>
        <div id="coll-target-members" class="person-check-list w-full"></div>
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">비고</label>
        <textarea id="coll-memo" rows="2" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none"></textarea>
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">첨부파일</label>
        <input type="file" id="coll-file-input" multiple class="w-full text-xs border border-outline-variant rounded-lg px-2 py-1.5">
        <div id="coll-attachment-list" class="mt-2 space-y-1"></div>
      </div>
    </div>
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-collection').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container transition-all">취소</button>
      <button onclick="saveCollection()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700 transition-all">저장</button>
    </div>
  </div>
</div>

<!-- Task Detail Modal -->
```

- [ ] **Step 2: `Pages.collections` 모듈에 폼/업로드/저장 함수 추가**

Task 3에서 만든 `return` 문을 찾는다:
```js
  return { switchTab, render, isFullyChecked, openDetail: (id) => {} };
})();
```
다음으로 교체 (기존 `render`/`isFullyChecked`/`switchTab` 함수는 그대로 두고, 그 사이에 아래 함수들을 추가한 뒤 `return`을 갱신):
```js
  function populateManagerSelect(selectedId) {
    const sel = document.getElementById('coll-manager');
    sel.innerHTML = '<option value="">담당자 선택</option>' +
      DB.members.map(m => `<option value="${m.id}" ${selectedId === m.id ? 'selected' : ''}>${m.name}</option>`).join('');
  }

  function renderAttachmentList() {
    const wrap = document.getElementById('coll-attachment-list');
    wrap.innerHTML = _collAttachments.map((a, i) => `
      <div class="flex items-center justify-between text-xs bg-surface-container rounded-lg px-2 py-1.5">
        <a href="${a.url}" target="_blank" class="text-primary hover:underline truncate">${a.name}</a>
        <button type="button" onclick="Pages.collections.removeAttachment(${i})" class="text-secondary hover:text-red-500"><span class="material-symbols-outlined text-sm">close</span></button>
      </div>`).join('');
  }

  function removeAttachment(i) {
    _collAttachments.splice(i, 1);
    renderAttachmentList();
  }

  async function uploadCollectionFiles(inputId, existingAttachments, collectionId) {
    const input = document.getElementById(inputId);
    const files = input ? Array.from(input.files) : [];
    const uploaded = [...existingAttachments];
    for (const file of files) {
      const path = `collections/${collectionId}/${Date.now()}_${file.name}`;
      const ref = firebase.storage().ref(path);
      try {
        await ref.put(file);
        const url = await ref.getDownloadURL();
        uploaded.push({ name: file.name, path, url });
      } catch (e) { console.error('파일 업로드 오류:', e); }
    }
    return uploaded;
  }

  function openAddModal() {
    document.getElementById('editing-collection-id').value = '';
    document.getElementById('coll-dept').value = '';
    document.getElementById('coll-title').value = '';
    document.getElementById('coll-deadline').value = '';
    document.getElementById('coll-tag').value = '';
    document.getElementById('coll-memo').value = '';
    document.getElementById('coll-file-input').value = '';
    _collAttachments = [];
    renderAttachmentList();
    populateManagerSelect(null);
    renderPersonCheckboxes('coll-target-members', [], false);
    document.getElementById('collection-modal-title').textContent = '취합건 등록';
    document.getElementById('modal-add-collection').classList.remove('hidden');
  }

  function editModal(id) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    document.getElementById('editing-collection-id').value = item.id;
    document.getElementById('coll-dept').value = item.dept;
    document.getElementById('coll-title').value = item.title;
    document.getElementById('coll-deadline').value = item.deadline;
    document.getElementById('coll-tag').value = item.tag || '';
    document.getElementById('coll-memo').value = item.memo || '';
    document.getElementById('coll-file-input').value = '';
    _collAttachments = [...(item.attachments || [])];
    renderAttachmentList();
    populateManagerSelect(item.managerId);
    renderPersonCheckboxes('coll-target-members', item.targetIds, false);
    document.getElementById('collection-modal-title').textContent = '취합건 수정';
    document.getElementById('modal-add-collection').classList.remove('hidden');
  }

  async function save() {
    const dept = document.getElementById('coll-dept').value.trim();
    const title = document.getElementById('coll-title').value.trim();
    const deadline = document.getElementById('coll-deadline').value;
    const managerId = parseInt(document.getElementById('coll-manager').value) || null;
    const memo = document.getElementById('coll-memo').value.trim();
    const tag = document.getElementById('coll-tag').value || null;
    const targetIds = [...document.querySelectorAll('#coll-target-members input:checked')].map(cb => parseInt(cb.value));
    if (!dept || !title || !deadline || !managerId) { alert('요청부서, 제목, 회신기한, 취합 담당자는 필수입니다'); return; }
    if (!targetIds.length) { alert('대상 팀원을 1명 이상 선택하세요'); return; }

    const editId = document.getElementById('editing-collection-id').value;
    const id = editId ? parseInt(editId) : Date.now();
    const existing = editId ? (DB.collections || []).find(c => c.id === id) : null;
    const attachments = await uploadCollectionFiles('coll-file-input', _collAttachments, id);

    const submitted = {};
    targetIds.forEach(mid => {
      const key = String(mid);
      submitted[key] = existing?.submitted?.[key] || { checked: false, at: null };
    });
    const status = isFullyChecked({ targetIds, submitted }) ? '완료' : '진행중';

    const data = {
      id, dept, tag, title, deadline, managerId, memo, targetIds, submitted, attachments, status,
      lastNotifiedAt: existing?.lastNotifiedAt || null,
      createdAt: existing?.createdAt || id,
      createdBy: existing?.createdBy || currentUser?.username || '',
    };

    if (existing) {
      Object.assign(existing, data);
      db.collection('collections').doc(String(id)).update(data).catch(e => console.error(e));
      showToast('취합건이 수정되었습니다');
    } else {
      if (!DB.collections) DB.collections = [];
      DB.collections.push(data);
      db.collection('collections').doc(String(id)).set(data).catch(e => console.error(e));
      showToast('취합건이 등록되었습니다');
    }
    document.getElementById('modal-add-collection').classList.add('hidden');
    render();
  }

  return { switchTab, render, isFullyChecked, openDetail: (id) => {}, openAddModal, editModal, save, removeAttachment };
})();
```

- [ ] **Step 3: 전역 shim 함수 추가**

Task 2에서 추가한 다음 라인을 찾는다:
```js
function switchCollectionTab(tab) { Pages.collections.switchTab(tab); }
```
다음으로 교체:
```js
function switchCollectionTab(tab) { Pages.collections.switchTab(tab); }
function openAddCollectionModal() { Pages.collections.openAddModal(); }
function saveCollection() { Pages.collections.save(); }
```

- [ ] **Step 4: 문법 검증**

Run: (Task 1과 동일한 node 검증 명령)
Expected: `SYNTAX OK`

- [ ] **Step 5: 구조 검증**

Run: `grep -n 'id="modal-add-collection"\|id="coll-target-members"\|function openAddCollectionModal\|function saveCollection' index.html`
Expected: 4줄 이상 출력, 전부 존재

- [ ] **Step 6: 로직 검증 — `isFullyChecked` 순수 로직 시뮬레이션**

```bash
node -e "
function isFullyChecked(item) {
  return item.targetIds.length > 0 && item.targetIds.every(mid => item.submitted[String(mid)]?.checked);
}
console.log(isFullyChecked({ targetIds: [1,2], submitted: {'1':{checked:true},'2':{checked:true}} })); // true
console.log(isFullyChecked({ targetIds: [1,2], submitted: {'1':{checked:true},'2':{checked:false}} })); // false
console.log(isFullyChecked({ targetIds: [], submitted: {} })); // false
"
```
Expected: `true`, `false`, `false`

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 등록/수정 모달 및 저장 로직 구현

요청부서/구분/제목/회신기한/담당자/비고 입력, 대상 팀원 다중선택
(renderPersonCheckboxes 재사용), 첨부파일 업로드(Firebase Storage),
신규 등록 시 status 자동 계산 로직 추가.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: 상세 화면 — 체크리스트, 완료 자동전환, 첨부파일, 삭제

**Files:**
- Modify: `index.html` (모달 HTML, `Pages.collections` 모듈, 전역 shim)

- [ ] **Step 1: 상세 모달 HTML 추가**

Task 4에서 추가한 `<!-- Collection Add/Edit Modal -->` 블록이 끝나는 지점을 찾는다:
```html
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-collection').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container transition-all">취소</button>
      <button onclick="saveCollection()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700 transition-all">저장</button>
    </div>
  </div>
</div>

<!-- Task Detail Modal -->
```
다음으로 교체:
```html
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-collection').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container transition-all">취소</button>
      <button onclick="saveCollection()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700 transition-all">저장</button>
    </div>
  </div>
</div>

<!-- Collection Detail Modal -->
<div id="modal-collection-detail" class="modal-overlay hidden" onclick="if(event.target===this)this.classList.add('hidden')" ontouchend="if(event.target===this){event.preventDefault();this.classList.add('hidden')}">
  <div class="modal-box" style="width:520px;max-height:90vh;overflow-y:auto;">
    <div class="flex items-start justify-between mb-1">
      <h3 class="text-lg font-bold text-on-surface" style="font-family:'Hanken Grotesk'" id="cd-title"></h3>
      <div class="flex gap-1">
        <button onclick="editCollection(window._activeCollectionId)" class="p-1.5 rounded-lg text-secondary hover:bg-surface-container"><span class="material-symbols-outlined text-sm">edit</span></button>
        <button onclick="deleteCollection(window._activeCollectionId)" class="p-1.5 rounded-lg text-red-400 hover:bg-red-50"><span class="material-symbols-outlined text-sm">delete</span></button>
        <button onclick="closeCollectionDetail()" class="p-1.5 rounded-lg text-secondary hover:bg-surface-container"><span class="material-symbols-outlined text-sm">close</span></button>
      </div>
    </div>
    <p id="cd-meta" class="text-xs text-secondary mb-3"></p>
    <p id="cd-memo" class="text-sm text-on-surface mb-4"></p>
    <div>
      <label class="block text-xs font-semibold text-secondary mb-1.5 uppercase tracking-wide">제출 현황</label>
      <div id="cd-checklist" class="border border-outline-variant rounded-lg overflow-hidden mb-2"></div>
      <button id="cd-notify-btn" onclick="notifyUnsubmitted(window._activeCollectionId)" class="w-full border border-primary text-primary rounded-lg py-2 text-sm font-semibold hover:bg-blue-50 transition-all disabled:opacity-40 disabled:cursor-not-allowed">미제출자에게 Webex 알림 보내기</button>
    </div>
    <div class="mt-4">
      <label class="block text-xs font-semibold text-secondary mb-1.5 uppercase tracking-wide">첨부파일</label>
      <div id="cd-attachments" class="space-y-1"></div>
    </div>
  </div>
</div>

<!-- Task Detail Modal -->
```

- [ ] **Step 2: `Pages.collections`에 상세/체크/삭제 함수 추가**

Task 4에서 만든 `return` 문을 찾는다:
```js
  return { switchTab, render, isFullyChecked, openDetail: (id) => {}, openAddModal, editModal, save, removeAttachment };
})();
```
다음으로 교체 (플레이스홀더였던 `openDetail: (id) => {}`를 실제 구현으로 교체하고 관련 함수 추가):
```js
  function renderDetail(id) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    const manager = DB.members.find(m => m.id === item.managerId);
    document.getElementById('cd-title').textContent = item.title;
    document.getElementById('cd-meta').textContent = `${item.dept} · 회신기한 ${item.deadline} · 취합담당 ${manager ? manager.name : '-'}`;
    document.getElementById('cd-memo').textContent = item.memo || '';
    document.getElementById('cd-checklist').innerHTML = item.targetIds.map(mid => {
      const m = DB.members.find(mm => mm.id === mid);
      const s = item.submitted[String(mid)] || { checked: false, at: null };
      return `<label class="flex items-center justify-between px-3 py-2 border-b border-outline-variant last:border-b-0 cursor-pointer">
        <span class="flex items-center gap-2 text-sm"><input type="checkbox" ${s.checked ? 'checked' : ''} onchange="Pages.collections.toggleCheck(${item.id},${mid})" class="w-4 h-4 accent-primary">${m ? m.name : mid}</span>
        <span class="text-xs text-secondary">${s.checked && s.at ? new Date(s.at).toLocaleString('ko-KR') : '미제출'}</span>
      </label>`;
    }).join('');
    const uncheckedCount = item.targetIds.filter(mid => !item.submitted[String(mid)]?.checked).length;
    const notifyBtn = document.getElementById('cd-notify-btn');
    notifyBtn.textContent = `미제출자에게 Webex 알림 보내기 (${uncheckedCount}명)`;
    notifyBtn.disabled = uncheckedCount === 0;
    document.getElementById('cd-attachments').innerHTML = (item.attachments || []).map(a =>
      `<a href="${a.url}" target="_blank" class="flex items-center gap-1.5 text-xs text-primary hover:underline"><span class="material-symbols-outlined text-sm">attach_file</span>${a.name}</a>`
    ).join('') || '<p class="text-xs text-secondary">첨부파일 없음</p>';
  }

  function openDetail(id) {
    window._activeCollectionId = id;
    renderDetail(id);
    document.getElementById('modal-collection-detail').classList.remove('hidden');
  }

  function closeDetail() {
    document.getElementById('modal-collection-detail').classList.add('hidden');
    window._activeCollectionId = null;
  }

  function toggleCheck(id, memberId) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    if (!item.submitted) item.submitted = {};
    const key = String(memberId);
    const wasChecked = !!item.submitted[key]?.checked;
    item.submitted[key] = { checked: !wasChecked, at: !wasChecked ? Date.now() : null };
    item.status = isFullyChecked(item) ? '완료' : '진행중';
    db.collection('collections').doc(String(id)).update({ submitted: item.submitted, status: item.status }).catch(e => console.error(e));
    renderDetail(id);
    render();
  }

  function deleteItem(id) {
    if (!confirm('취합건을 삭제하시겠습니까?')) return;
    DB.collections = (DB.collections || []).filter(c => c.id !== id);
    db.collection('collections').doc(String(id)).delete().catch(e => console.error(e));
    closeDetail();
    showToast('삭제되었습니다');
    render();
  }

  return { switchTab, render, isFullyChecked, openDetail, closeDetail, toggleCheck, deleteItem, openAddModal, editModal, save, removeAttachment };
})();
```

- [ ] **Step 3: 전역 shim 함수 추가**

Task 4에서 추가한 다음 라인을 찾는다:
```js
function saveCollection() { Pages.collections.save(); }
```
다음으로 교체:
```js
function saveCollection() { Pages.collections.save(); }
function editCollection(id) { Pages.collections.editModal(id); }
function deleteCollection(id) { Pages.collections.deleteItem(id); }
function closeCollectionDetail() { Pages.collections.closeDetail(); }
```

> `editCollection`은 상세 모달을 닫지 않고 그대로 등록/수정 모달을 여는데, 두 모달이 동시에 열려 있으면 배경이 겹쳐 보일 수 있다. `Pages.collections.editModal(id)` 내부에서 상세 모달을 닫도록 Step 4에서 보정한다.

- [ ] **Step 4: `editModal`이 상세 모달을 먼저 닫도록 보정**

다음을 찾는다:
```js
  function editModal(id) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    document.getElementById('editing-collection-id').value = item.id;
```
다음으로 교체:
```js
  function editModal(id) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    closeDetail();
    document.getElementById('editing-collection-id').value = item.id;
```

- [ ] **Step 5: 문법 검증**

Run: (Task 1과 동일한 node 검증 명령)
Expected: `SYNTAX OK`

- [ ] **Step 6: 구조 검증**

Run: `grep -n 'id="modal-collection-detail"\|function toggleCheck\|function deleteItem\|window._activeCollectionId' index.html`
Expected: 여러 줄 출력, 전부 존재

- [ ] **Step 7: 로직 검증 — 체크 토글 + 완료 자동전환 시뮬레이션**

```bash
node -e "
function isFullyChecked(item) {
  return item.targetIds.length > 0 && item.targetIds.every(mid => item.submitted[String(mid)]?.checked);
}
const item = { targetIds: [1,2], submitted: { '1': {checked:false}, '2': {checked:false} }, status: '진행중' };
// 1번 체크
item.submitted['1'] = { checked: true, at: Date.now() };
item.status = isFullyChecked(item) ? '완료' : '진행중';
console.log(item.status); // 진행중
// 2번 체크 -> 전원 완료
item.submitted['2'] = { checked: true, at: Date.now() };
item.status = isFullyChecked(item) ? '완료' : '진행중';
console.log(item.status); // 완료
"
```
Expected: `진행중` 그다음 `완료`

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 상세 화면(체크리스트/완료 자동전환/첨부파일/삭제) 구현

팀원별 제출 체크 토글, 전원 체크 시 자동으로 완료 탭으로 이동,
첨부파일 다운로드 링크 표시, 삭제 기능 추가.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Webex 알림 발송 + 설정 페이지 봇 토큰 연동

**Files:**
- Modify: `index.html` (설정 페이지 HTML, `loadSettings()`, `Pages.collections` 모듈, 전역 shim)

- [ ] **Step 1: 설정 페이지 Webex 입력 활성화**

다음을 찾는다:
```html
            <div class="pt-2 border-t border-outline-variant">
              <p class="text-xs text-secondary mb-2">Webex 연동</p>
              <input type="text" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm mb-2" placeholder="Webex Webhook URL (추후 설정)" disabled>
              <p class="text-xs text-secondary">Outlook/Webex 연동은 IT 관리자에게 문의하세요</p>
            </div>
```
다음으로 교체:
```html
            <div class="pt-2 border-t border-outline-variant hidden" id="settings-webex-section">
              <p class="text-xs text-secondary mb-2">Webex 연동 (취합리스트 알림 전용 봇 토큰)</p>
              <input type="password" id="webex-bot-token" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm mb-2" placeholder="Webex Bot Access Token">
              <button onclick="saveWebexSettings()" class="w-full border border-primary text-primary rounded-lg py-1.5 text-xs font-semibold hover:bg-blue-50">저장</button>
              <p class="text-xs text-secondary mt-2">취합리스트의 "미제출자에게 알림 보내기"에 사용됩니다. 관리자만 설정할 수 있습니다.</p>
            </div>
```

- [ ] **Step 2: `loadSettings()`에 Webex 섹션 로드 로직 추가**

다음을 찾는다:
```js
function loadSettings() {
  document.getElementById('my-name').value = DB.myProfile.name || '';
  document.getElementById('my-email').value = DB.myProfile.email || '';
  document.getElementById('my-rank').value = DB.myProfile.rank || '';
  // 관리자 전용 양식 관리 섹션
  const tplSection = document.getElementById('settings-templates-section');
  if (tplSection) {
    const isAdmin = currentUser?.username === 'jungsoo.kim';
    tplSection.classList.toggle('hidden', !isAdmin);
    if (isAdmin) renderTemplateList();
  }
}
```
다음으로 교체:
```js
function loadSettings() {
  document.getElementById('my-name').value = DB.myProfile.name || '';
  document.getElementById('my-email').value = DB.myProfile.email || '';
  document.getElementById('my-rank').value = DB.myProfile.rank || '';
  // 관리자 전용 양식 관리 섹션
  const tplSection = document.getElementById('settings-templates-section');
  if (tplSection) {
    const isAdmin = currentUser?.username === 'jungsoo.kim';
    tplSection.classList.toggle('hidden', !isAdmin);
    if (isAdmin) renderTemplateList();
  }
  // 관리자 전용 Webex 봇 토큰 설정
  const webexSection = document.getElementById('settings-webex-section');
  if (webexSection) {
    const isAdmin = currentUser?.username === 'jungsoo.kim';
    webexSection.classList.toggle('hidden', !isAdmin);
    if (isAdmin) {
      db.collection('settings').doc('webex').get().then(snap => {
        const tokenInput = document.getElementById('webex-bot-token');
        if (tokenInput) tokenInput.value = snap.exists ? (snap.data().botToken || '') : '';
      }).catch(e => console.error('Webex 설정 로드 실패:', e));
    }
  }
}

function saveWebexSettings() {
  const token = document.getElementById('webex-bot-token').value.trim();
  if (!token) { showToast('토큰을 입력하세요'); return; }
  db.collection('settings').doc('webex').set({ botToken: token }).then(() => {
    showToast('Webex 설정이 저장되었습니다');
  }).catch(e => { console.error(e); showToast('저장 실패'); });
}
```

> `settings` 컬렉션은 의도적으로 `FS_COLLS`/`setupListeners()`에 추가하지 않는다 (스펙 4절: 앱 부팅 시 전체 동기화 대상에서 제외해 노출을 최소화).

- [ ] **Step 3: `Pages.collections`에 `notifyUnsubmitted` 추가**

Task 5에서 만든 `return` 문을 찾는다:
```js
  return { switchTab, render, isFullyChecked, openDetail, closeDetail, toggleCheck, deleteItem, openAddModal, editModal, save, removeAttachment };
})();
```
다음으로 교체:
```js
  async function notifyUnsubmitted(id) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    const uncheckedIds = item.targetIds.filter(mid => !item.submitted[String(mid)]?.checked);
    if (!uncheckedIds.length) { showToast('미제출자가 없습니다'); return; }

    let botToken;
    try {
      const snap = await db.collection('settings').doc('webex').get();
      botToken = snap.exists ? snap.data().botToken : null;
    } catch (e) {
      showToast('Webex 설정을 불러오지 못했습니다');
      return;
    }
    if (!botToken) { showToast('Webex 봇 토큰이 설정되지 않았습니다. 설정 페이지에서 등록해주세요'); return; }

    const text = `[취합리스트] ${item.dept} · ${item.title} — 회신기한 ${item.deadline}까지 제출 부탁드립니다.`;
    let successCount = 0;
    const failNames = [];
    for (const mid of uncheckedIds) {
      const member = DB.members.find(m => m.id === mid);
      if (!member || !member.email) { failNames.push(member ? member.name : String(mid)); continue; }
      try {
        const res = await fetch('https://webexapis.com/v1/messages', {
          method: 'POST',
          headers: { 'Authorization': `Bearer ${botToken}`, 'Content-Type': 'application/json' },
          body: JSON.stringify({ toPersonEmail: member.email, text }),
        });
        if (res.ok) successCount++; else failNames.push(member.name);
      } catch (e) { failNames.push(member.name); }
    }
    item.lastNotifiedAt = Date.now();
    db.collection('collections').doc(String(id)).update({ lastNotifiedAt: item.lastNotifiedAt }).catch(e => console.error(e));
    showToast(failNames.length
      ? `${successCount}명에게 발송완료, 실패: ${failNames.join(', ')}`
      : `${successCount}명에게 알림 발송완료`);
    render();
  }

  return { switchTab, render, isFullyChecked, openDetail, closeDetail, toggleCheck, deleteItem, openAddModal, editModal, save, removeAttachment, notifyUnsubmitted };
})();
```

- [ ] **Step 4: 전역 shim 함수 추가**

Task 5에서 추가한 다음 라인을 찾는다:
```js
function closeCollectionDetail() { Pages.collections.closeDetail(); }
```
다음으로 교체:
```js
function closeCollectionDetail() { Pages.collections.closeDetail(); }
function notifyUnsubmitted(id) { Pages.collections.notifyUnsubmitted(id); }
```

- [ ] **Step 5: 문법 검증**

Run: (Task 1과 동일한 node 검증 명령)
Expected: `SYNTAX OK`

- [ ] **Step 6: 구조 검증**

Run: `grep -n 'id="settings-webex-section"\|function saveWebexSettings\|function notifyUnsubmitted\|webexapis.com' index.html`
Expected: 4줄 이상 출력, 전부 존재

Run: `grep -n "'settings'" index.html`
Expected: `settings` 컬렉션 참조가 `db.collection('settings')` 호출 2곳(로드/저장)과 `notifyUnsubmitted`의 1곳, 총 3곳에서만 나와야 하며 `FS_COLLS`/`setupListeners()` 안에는 나오지 않아야 한다 (수동으로 확인).

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 Webex 알림 발송 + 설정 페이지 봇 토큰 연동

관리자 전용으로 Webex Bot Token을 settings/webex 문서에 저장(부팅 시
동기화 대상에서 제외해 노출 최소화). 상세 화면에서 미제출자에게
Webex 메시지를 직접 발송하는 기능 추가.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Firestore 보안 규칙 설정 안내 + 최종 통합 확인

이 태스크는 리포지토리에 `firestore.rules` 파일이 없어(콘솔에서 직접 관리) 코드 변경이 아니라 **사람이 Firebase 콘솔에서 수동으로 해야 하는 설정**과 **최종 확인**으로 구성된다.

- [ ] **Step 1: Firestore 보안 규칙에 다음 내용이 반영되어 있는지 Firebase 콘솔에서 확인**

`collections`와 `settings` 컬렉션 모두 인증된 사용자만 read/write 가능해야 한다. 기존 다른 컬렉션(`drafts`, `projects` 등)에 이미 적용된 것과 동일한 수준의 규칙을 그대로 적용하면 된다. 예시:

```
match /collections/{docId} {
  allow read, write: if request.auth != null;
}
match /settings/{docId} {
  allow read, write: if request.auth != null;
}
```

> 이 프로젝트가 Firebase Authentication이 아니라 자체 `users` 컬렉션 기반 로그인을 쓰고 있다면(로그인 방식은 기존 `requireAuth()`/`currentUser` 로직 참고), `request.auth != null` 대신 기존 다른 컬렉션에 실제로 걸려 있는 규칙과 동일하게 맞춰야 한다. 코드 변경이 아니므로 실제 규칙 문구는 Firebase 콘솔에서 기존 규칙을 열어 확인 후 그대로 따라 작성한다.

- [ ] **Step 2: Webex 봇 생성 및 토큰 발급 (관리자 1회 작업)**

1. https://developer.webex.com/my-apps 에서 "Create a Bot" 진행, 메시지 발송 권한만 있는 전용 봇 생성
2. 발급된 Access Token을 대시보드 로그인 후 설정 페이지 → Webex 연동 입력창에 붙여넣고 저장

- [ ] **Step 3: (선택) 브라우저 스모크 테스트**

이 프로젝트 컨벤션상 기본적으로 생략 가능하지만, 이번 기능은 Firestore 쓰기 + Storage 업로드 + 외부 API 호출이 얽혀 있어 최초 1회는 실제 동작 확인을 권장한다. 사용자가 원할 경우 다음을 확인한다 (원치 않으면 이 스텝은 건너뛰고 Task 7을 완료로 처리):

1. `preview_start`로 로컬 서버 기동
2. 로그인 우회 스크립트로 `showPage('collections')` 진입
3. 취합건 등록 → 목록에 표시되는지
4. 상세에서 체크 토글 → 전원 체크 시 완료 탭으로 이동하는지
5. Webex 알림 버튼(토큰 미설정 상태에서는 "토큰이 설정되지 않았습니다" 토스트가 뜨는지)

- [ ] **Step 4: 최종 문법/구조 검증 (전체 태스크 통합 확인)**

Run: (Task 1과 동일한 node 검증 명령)
Expected: `SYNTAX OK`

Run:
```bash
grep -c "Pages.collections" index.html
```
Expected: 15 이상 (모듈 정의 + 여러 호출 지점)

- [ ] **Step 5: 커밋 (Step 1~2 관련 문서만 있을 경우, 없으면 생략)**

Step 1/2는 Firebase 콘솔 작업이라 커밋할 코드 변경이 없다. 만약 이 계획을 실행하며 별도 메모(예: CHANGELOG.md)를 남겼다면 그 파일만 커밋한다:

```bash
git add CHANGELOG.md
git commit -m "$(cat <<'EOF'
docs: 취합리스트 기능 릴리즈 노트 추가

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## 스펙 커버리지 체크 (자체 검토 결과)

- 데이터 모델(`collections` 컬렉션, `settings/webex` 문서) → Task 1, 6
- 네비게이션/페이지 shell → Task 2
- 목록 화면(탭, 정렬, 배지, 기한 강조) → Task 3
- 등록/수정 폼 + 대상 팀원 다중선택 + 첨부파일 업로드 → Task 4
- 상세 화면 체크리스트 + 완료 자동전환 + 첨부파일 표시 + 삭제 → Task 5
- Webex 알림 발송 + 보안(토큰 미동기화, 관리자 전용 설정 UI) → Task 6
- Firestore 보안 규칙 + Webex 봇 생성(수동) + 최종 확인 → Task 7
- 권한(로그인 팀원 전원 편집 가능) → 별도 제한 로직을 추가하지 않음으로써 구현됨 (스펙 5절)

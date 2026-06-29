# 프로젝트 관리 구현 플랜

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 업무현황 태스크를 프로젝트 단위로 묶어 진척률·마일스톤·댓글을 관리하는 프로젝트 탭을 추가한다.

**Architecture:** 단일 `index.html` 파일에 모든 코드가 있는 구조. `Pages.projects` IIFE를 추가하고, 기존 `DB`, `FS_COLLS`, 대시보드 태스크 모달, 사이드바 nav를 수정한다. Firestore `projects` 컬렉션을 실시간 동기화한다.

**Tech Stack:** Firebase Firestore compat 10.12.2, Tailwind CSS CDN, Material Symbols, vanilla JS IIFE pattern

---

## 파일 수정 목록

- Modify: `index.html` (전체 — 아래 각 태스크에서 정확한 위치 지정)

---

## Task 1: DB + FS_COLLS + 사이드바 Nav 추가

**Files:**
- Modify: `index.html` (line ~1290 FS_COLLS, ~1295 DB, ~301 nav sidebar)

- [ ] **Step 1: FS_COLLS에 `projects` 추가**

`index.html` line ~1290에서:
```js
// 변경 전
const FS_COLLS = ['tasks', 'members', 'centers', 'events', 'notices', 'users', 'memos', 'library'];

// 변경 후
const FS_COLLS = ['tasks', 'members', 'centers', 'events', 'notices', 'users', 'memos', 'library', 'projects'];
```

- [ ] **Step 2: DB에 `projects: []` 추가**

`index.html` line ~1295에서 `let DB = {` 블록에 추가:
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
  projects: [],   // ← 추가
  myProfile: { name: '나', email: '', rank: '' },
};
```

- [ ] **Step 3: 사이드바 nav에 프로젝트 항목 추가**

`index.html` line ~301 `nav-library` 바로 위에 삽입:
```html
<div class="nav-item" onclick="showPage('projects')" id="nav-projects">
  <span class="material-symbols-outlined">folder_special</span> 프로젝트
</div>
```

- [ ] **Step 4: showPage에 `projects` 케이스 추가**

`showPage()` 함수를 찾아 `'library'` 케이스 근처에 추가:
```js
case 'projects': Pages.projects.render(); break;
```

그리고 `refreshCurrentPage()` 함수도 동일하게 추가:
```js
case 'projects': Pages.projects.render(); break;
```

- [ ] **Step 5: 커밋**
```bash
git add index.html
git commit -m "feat: DB projects[] + FS_COLLS + nav-projects 추가"
```

---

## Task 2: page-projects HTML 구조 추가

**Files:**
- Modify: `index.html` (line ~659 `page-library` 바로 앞에 삽입)

- [ ] **Step 1: page-projects 페이지 div 삽입**

`id="page-library"` div 바로 앞에 아래 HTML을 삽입:

```html
<!-- Projects Page -->
<div id="page-projects" class="page">
  <div class="p-6">
    <!-- 헤더 -->
    <div class="flex items-center justify-between mb-6">
      <div>
        <h1 class="text-2xl font-bold text-on-surface" style="font-family:'Hanken Grotesk'">프로젝트</h1>
        <p class="text-sm text-secondary mt-0.5">프로젝트별 진척률 및 업무 현황 관리</p>
      </div>
      <button onclick="openAddProjectModal()" class="flex items-center gap-2 bg-primary text-white px-4 py-2.5 rounded-xl text-sm font-semibold hover:bg-blue-700">
        <span class="material-symbols-outlined text-base">add</span> 프로젝트 추가
      </button>
    </div>
    <!-- 프로젝트 레이아웃: 카드 그리드(기본) / 마스터-디테일(선택 시) -->
    <div id="projects-layout" class="flex gap-6">
      <!-- 카드 목록 (기본: 3열 그리드 / 선택 시: 좌측 1/3 목록) -->
      <div id="projects-card-area" class="w-full">
        <div id="projects-grid" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"></div>
        <div id="projects-empty" class="hidden flex flex-col items-center justify-center py-20 text-secondary">
          <span class="material-symbols-outlined text-5xl mb-3 opacity-40">folder_special</span>
          <p class="text-sm">아직 프로젝트가 없습니다</p>
          <button onclick="openAddProjectModal()" class="mt-4 text-primary text-sm hover:underline">+ 첫 프로젝트 만들기</button>
        </div>
      </div>
      <!-- 상세 패널 (기본: hidden) -->
      <div id="projects-detail-panel" class="hidden w-2/3 flex-shrink-0 bg-surface rounded-2xl border border-outline-variant overflow-y-auto" style="max-height:calc(100vh - 180px)">
        <div id="projects-detail-content" class="p-6"></div>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 2: 브라우저에서 확인**

앱을 열고 사이드바 "프로젝트" 클릭 → 빈 페이지에 헤더와 "프로젝트 추가" 버튼 보이면 정상.

- [ ] **Step 3: 커밋**
```bash
git add index.html
git commit -m "feat: page-projects HTML 레이아웃 추가"
```

---

## Task 3: modal-add-project HTML 추가

**Files:**
- Modify: `index.html` (modal-add-task 바로 위에 삽입)

- [ ] **Step 1: 프로젝트 추가/수정 모달 HTML 삽입**

`<!-- Task Add/Edit Modal -->` 또는 `id="modal-add-task"` 바로 위에 삽입:

```html
<!-- Project Add/Edit Modal -->
<div id="modal-add-project" class="modal-overlay hidden" onclick="if(event.target===this)this.classList.add('hidden')">
  <div class="modal-box" style="width:600px; max-height:90vh; overflow-y:auto;">
    <div class="flex items-center justify-between mb-5">
      <h3 class="text-lg font-bold text-on-surface" style="font-family:'Hanken Grotesk'" id="project-modal-title">새 프로젝트</h3>
      <button onclick="document.getElementById('modal-add-project').classList.add('hidden')" class="text-secondary p-1 rounded-lg hover:bg-surface-container"><span class="material-symbols-outlined">close</span></button>
    </div>
    <input type="hidden" id="editing-project-id">
    <div class="space-y-4">
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">프로젝트명 *</label>
        <input type="text" id="project-title" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="프로젝트명을 입력하세요">
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">개요</label>
        <input type="text" id="project-summary" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="한 줄 요약">
      </div>
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">내용</label>
        <textarea id="project-description" rows="3" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm resize-none" placeholder="프로젝트 상세 내용..."></textarea>
      </div>
      <div class="grid grid-cols-2 gap-3">
        <div>
          <label class="block text-xs font-semibold text-secondary mb-1.5">시작일</label>
          <input type="date" id="project-start-date" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm">
        </div>
        <div>
          <label class="block text-xs font-semibold text-secondary mb-1.5">목표일</label>
          <input type="date" id="project-end-date" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm">
        </div>
      </div>
      <!-- 담당자 멀티선택 드롭박스 -->
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">담당자</label>
        <div class="relative" id="project-assignee-wrap">
          <button type="button" onclick="toggleProjectAssigneeDropdown()" id="project-assignee-btn"
            class="w-full flex items-center justify-between border border-outline-variant rounded-lg px-3 py-2 text-sm bg-white text-left">
            <span id="project-assignee-label" class="text-secondary">담당자 선택</span>
            <span class="material-symbols-outlined text-secondary text-base">expand_more</span>
          </button>
          <div id="project-assignee-panel" class="hidden absolute left-0 right-0 top-full mt-1 bg-white border border-outline-variant rounded-xl shadow-lg z-20 max-h-48 overflow-y-auto">
            <div id="project-assignee-list"></div>
          </div>
        </div>
      </div>
      <!-- 마일스톤 -->
      <div>
        <label class="block text-xs font-semibold text-secondary mb-1.5">마일스톤 (선택)</label>
        <div id="project-milestone-list" class="space-y-2 mb-2"></div>
        <div class="flex gap-2">
          <input type="text" id="new-milestone-title" class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="마일스톤명">
          <input type="date" id="new-milestone-date" class="border border-outline-variant rounded-lg px-3 py-2 text-sm">
          <button type="button" onclick="addModalMilestone()" class="bg-surface-container border border-outline-variant rounded-lg px-3 py-2 text-sm text-secondary hover:bg-surface-container-high">추가</button>
        </div>
      </div>
    </div>
    <div class="flex gap-3 mt-6">
      <button onclick="document.getElementById('modal-add-project').classList.add('hidden')" class="flex-1 border border-outline-variant rounded-lg py-2.5 text-sm text-secondary hover:bg-surface-container">취소</button>
      <button onclick="saveProject()" class="flex-1 bg-primary text-white rounded-lg py-2.5 text-sm font-semibold hover:bg-blue-700">저장</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: 커밋**
```bash
git add index.html
git commit -m "feat: modal-add-project HTML 추가"
```

---

## Task 4: Pages.projects IIFE — 렌더링 + 상세 패널

**Files:**
- Modify: `index.html` (Pages.library IIFE 끝 `})();` 이후에 추가)

- [ ] **Step 1: Pages.projects IIFE 기본 구조 + render() 추가**

Library shims 블록 (`// ---- Library global shims ----`) 바로 앞에 삽입:

```js
// ============================================================
//  PAGES.PROJECTS
// ============================================================
Pages.projects = (() => {
  let selectedProjectId = null;
  // 모달 내 임시 마일스톤 목록
  let _modalMilestones = [];
  // 담당자 드롭박스 선택 목록
  let _selectedAssigneeIds = [];

  // ---- 진척률 계산 ----
  function calcProgress(projectId) {
    const tasks = DB.tasks.filter(t => t.projectId === projectId);
    if (!tasks.length) return 0;
    const weights = { '완료': 100, '진행중': 50, '지연': 25 };
    const sum = tasks.reduce((acc, t) => acc + (weights[t.status] || 0), 0);
    return Math.round(sum / tasks.length);
  }

  // ---- 원형 진척률 SVG ----
  function progressRing(pct, size = 48) {
    const r = (size - 6) / 2;
    const circ = 2 * Math.PI * r;
    const dash = (pct / 100) * circ;
    const color = pct >= 80 ? '#16a34a' : pct >= 40 ? '#2563eb' : '#f59e0b';
    return `<svg width="${size}" height="${size}" viewBox="0 0 ${size} ${size}" class="flex-shrink-0">
      <circle cx="${size/2}" cy="${size/2}" r="${r}" fill="none" stroke="#e2e8f0" stroke-width="5"/>
      <circle cx="${size/2}" cy="${size/2}" r="${r}" fill="none" stroke="${color}" stroke-width="5"
        stroke-dasharray="${dash} ${circ}" stroke-dashoffset="${circ/4}" stroke-linecap="round"/>
      <text x="${size/2}" y="${size/2+1}" text-anchor="middle" dominant-baseline="middle"
        font-size="10" font-weight="700" fill="${color}">${pct}%</text>
    </svg>`;
  }

  // ---- 프로젝트 카드 HTML ----
  function buildCard(p, mini = false) {
    const pct = calcProgress(p.id);
    const tasks = DB.tasks.filter(t => t.projectId === p.id);
    const assignees = (p.assigneeIds || []).map(id => DB.members.find(m => m.id === id)).filter(Boolean);
    const isSelected = p.id === selectedProjectId;
    if (mini) {
      return `<div class="p-3 rounded-xl border cursor-pointer transition-all ${isSelected ? 'border-primary bg-primary-container' : 'border-outline-variant hover:bg-surface-container'}"
        onclick="selectProject(${p.id})">
        <div class="flex items-center gap-2">
          ${progressRing(pct, 36)}
          <div class="min-w-0">
            <div class="text-xs font-bold truncate">${p.title}</div>
            <div class="text-xs text-secondary">${tasks.length}개 업무</div>
          </div>
        </div>
      </div>`;
    }
    return `<div class="bg-surface rounded-2xl border border-outline-variant p-5 cursor-pointer hover:shadow-md transition-all hover:border-primary/40"
      onclick="selectProject(${p.id})">
      <div class="flex items-start justify-between mb-3">
        <h3 class="font-bold text-on-surface text-sm leading-snug flex-1 mr-2">${p.title}</h3>
        ${progressRing(pct)}
      </div>
      ${p.summary ? `<p class="text-xs text-secondary mb-3 line-clamp-2">${p.summary}</p>` : ''}
      <div class="flex items-center justify-between text-xs text-secondary mt-3">
        <div class="flex items-center gap-1">
          <span class="material-symbols-outlined text-sm">calendar_today</span>
          ${p.endDate || '기한 없음'}
        </div>
        <div class="flex items-center gap-1">
          <span class="material-symbols-outlined text-sm">task_alt</span>
          ${tasks.length}개 업무
        </div>
      </div>
      ${assignees.length ? `<div class="flex gap-1 mt-3">${assignees.slice(0,4).map(m=>`<div class="w-6 h-6 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold" title="${m.name}">${m.name[0]}</div>`).join('')}${assignees.length>4?`<div class="w-6 h-6 rounded-full bg-surface-container-high text-secondary flex items-center justify-center text-xs">+${assignees.length-4}</div>`:''}</div>` : ''}
    </div>`;
  }

  // ---- render() ----
  function render() {
    if (!requireAuth()) return;
    const grid = document.getElementById('projects-grid');
    const empty = document.getElementById('projects-empty');
    const detail = document.getElementById('projects-detail-panel');
    if (!grid) return;

    const projects = DB.projects || [];

    if (!projects.length) {
      grid.innerHTML = '';
      grid.classList.add('hidden');
      empty?.classList.remove('hidden');
      detail?.classList.add('hidden');
      return;
    }
    empty?.classList.add('hidden');
    grid.classList.remove('hidden');

    if (selectedProjectId && projects.find(p => p.id === selectedProjectId)) {
      // 마스터-디테일 모드
      const cardArea = document.getElementById('projects-card-area');
      cardArea.classList.remove('w-full');
      cardArea.classList.add('w-1/3', 'flex-shrink-0');
      grid.className = 'flex flex-col gap-2';
      grid.innerHTML = projects.map(p => buildCard(p, true)).join('');
      detail?.classList.remove('hidden');
      const detailContent = document.getElementById('projects-detail-content');
      if (detailContent) detailContent.innerHTML = buildDetailPanel(projects.find(p => p.id === selectedProjectId));
    } else {
      // 그리드 모드
      selectedProjectId = null;
      const cardArea = document.getElementById('projects-card-area');
      cardArea.classList.add('w-full');
      cardArea.classList.remove('w-1/3', 'flex-shrink-0');
      grid.className = 'grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4';
      grid.innerHTML = projects.map(p => buildCard(p, false)).join('');
      detail?.classList.add('hidden');
    }
  }

  // ---- selectProject / closeDetail ----
  function selectProject(id) {
    selectedProjectId = (selectedProjectId === id) ? null : id;
    render();
  }

  function closeDetail() {
    selectedProjectId = null;
    render();
  }

  return {
    render,
    selectProject,
    closeDetail,
    get selectedId() { return selectedProjectId; },
  };
})();

function selectProject(id) { Pages.projects.selectProject(id); }
function closeProjectDetail() { Pages.projects.closeDetail(); }
```

- [ ] **Step 2: 브라우저에서 확인**

앱 → 프로젝트 탭 → Firestore에 테스트 프로젝트 문서 추가 후 새로고침. 카드가 보이면 정상.

- [ ] **Step 3: 커밋**
```bash
git add index.html
git commit -m "feat: Pages.projects render + 카드 그리드 + 마스터-디테일 레이아웃"
```

---

## Task 5: 상세 패널 buildDetailPanel + 간트 바

**Files:**
- Modify: `index.html` (Pages.projects IIFE 내부, `return {` 앞에 추가)

- [ ] **Step 1: buildGanttBar + buildDetailPanel 함수 추가**

Pages.projects IIFE 내 `return {` 바로 위에 삽입:

```js
  // ---- 간트 바 ----
  function buildGanttBar(p) {
    if (!p.startDate || !p.endDate) return '<p class="text-xs text-secondary">기간 정보 없음</p>';
    const start = new Date(p.startDate);
    const end = new Date(p.endDate);
    const total = end - start;
    if (total <= 0) return '<p class="text-xs text-secondary">기간 오류</p>';
    const today = new Date();
    const todayPct = Math.min(100, Math.max(0, Math.round(((today - start) / total) * 100)));
    const milestones = p.milestones || [];

    return `<div class="relative mt-2 mb-1" style="height:32px;">
      <!-- 배경 바 -->
      <div class="absolute inset-y-2 left-0 right-0 rounded-full bg-surface-container-high"></div>
      <!-- 진행 바 -->
      <div class="absolute inset-y-2 left-0 rounded-full bg-primary/30" style="width:${todayPct}%"></div>
      <!-- 오늘 선 -->
      <div class="absolute inset-y-0 w-0.5 bg-primary/60" style="left:${todayPct}%" title="오늘"></div>
      <!-- 마일스톤 마커 -->
      ${milestones.map(m => {
        const mDate = new Date(m.date);
        const pct = Math.min(100, Math.max(0, Math.round(((mDate - start) / total) * 100)));
        return `<div class="absolute top-0.5 w-4 h-4 flex items-center justify-center -translate-x-2 cursor-pointer" style="left:${pct}%" title="${m.title} (${m.date})">
          <div class="w-3 h-3 rotate-45 ${m.done ? 'bg-green-500' : 'bg-amber-400'} border-2 ${m.done ? 'border-green-700' : 'border-amber-600'}"></div>
        </div>`;
      }).join('')}
    </div>
    <div class="flex justify-between text-xs text-secondary mt-1">
      <span>${p.startDate}</span><span>${p.endDate}</span>
    </div>`;
  }

  // ---- 상세 패널 HTML ----
  function buildDetailPanel(p) {
    if (!p) return '';
    const pct = calcProgress(p.id);
    const tasks = DB.tasks.filter(t => t.projectId === p.id);
    const assignees = (p.assigneeIds || []).map(id => DB.members.find(m => m.id === id)).filter(Boolean);
    const milestones = p.milestones || [];
    const comments = p.comments || [];
    const isAdmin = currentUser?.username === 'jungsoo.kim';
    const isOwner = currentUser?.username === p.createdBy;
    const canEdit = isAdmin || isOwner;

    const statusCounts = {
      완료: tasks.filter(t => t.status === '완료').length,
      진행중: tasks.filter(t => t.status === '진행중').length,
      지연: tasks.filter(t => t.status === '지연').length,
      기타: tasks.filter(t => !['완료','진행중','지연'].includes(t.status)).length,
    };

    return `
    <!-- 헤더 -->
    <div class="flex items-start justify-between mb-5">
      <div class="flex-1 mr-3">
        <h2 class="text-lg font-bold text-on-surface" style="font-family:'Hanken Grotesk'">${p.title}</h2>
        ${p.summary ? `<p class="text-sm text-secondary mt-0.5">${p.summary}</p>` : ''}
      </div>
      <div class="flex items-center gap-2 flex-shrink-0">
        ${canEdit ? `
          <button onclick="editProject(${p.id})" class="p-1.5 rounded-lg text-secondary hover:bg-surface-container" title="수정">
            <span class="material-symbols-outlined text-sm">edit</span>
          </button>
          <button onclick="deleteProject(${p.id})" class="p-1.5 rounded-lg text-red-400 hover:bg-red-50" title="삭제">
            <span class="material-symbols-outlined text-sm">delete</span>
          </button>
        ` : ''}
        <button onclick="closeProjectDetail()" class="p-1.5 rounded-lg text-secondary hover:bg-surface-container">
          <span class="material-symbols-outlined text-sm">close</span>
        </button>
      </div>
    </div>

    <!-- 메타 정보 -->
    <div class="bg-surface-container-low rounded-xl p-4 mb-4 space-y-2 text-sm">
      ${p.description ? `<p class="text-on-surface text-sm leading-relaxed mb-2">${p.description}</p>` : ''}
      <div class="flex items-center gap-2 text-secondary text-xs">
        <span class="material-symbols-outlined text-sm">calendar_today</span>
        ${p.startDate||'?'} ~ ${p.endDate||'?'}
      </div>
      ${assignees.length ? `<div class="flex items-center gap-2">
        <span class="material-symbols-outlined text-sm text-secondary">group</span>
        <div class="flex gap-1.5 flex-wrap">${assignees.map(m=>`<div class="flex items-center gap-1 bg-white border border-outline-variant rounded-full px-2 py-0.5 text-xs"><div class="w-4 h-4 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold">${m.name[0]}</div>${m.name}</div>`).join('')}</div>
      </div>` : ''}
    </div>

    <!-- 진척률 -->
    <div class="mb-4">
      <div class="flex items-center justify-between mb-2">
        <span class="text-xs font-semibold text-secondary">전체 진척률</span>
        <span class="text-sm font-bold text-primary">${pct}%</span>
      </div>
      <div class="w-full bg-surface-container-high rounded-full h-2.5 mb-2">
        <div class="h-2.5 rounded-full bg-primary transition-all" style="width:${pct}%"></div>
      </div>
      <div class="flex gap-3 text-xs text-secondary">
        <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-green-500 inline-block"></span>완료 ${statusCounts.완료}</span>
        <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-blue-500 inline-block"></span>진행중 ${statusCounts.진행중}</span>
        <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-orange-400 inline-block"></span>지연 ${statusCounts.지연}</span>
        <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-gray-300 inline-block"></span>기타 ${statusCounts.기타}</span>
      </div>
    </div>

    <!-- 간트 바 -->
    <div class="mb-4">
      <div class="text-xs font-semibold text-secondary mb-2 flex items-center gap-1">
        <span class="material-symbols-outlined text-sm">timeline</span> 일정 타임라인
      </div>
      ${buildGanttBar(p)}
    </div>

    <!-- 마일스톤 목록 -->
    <div class="mb-4">
      <div class="text-xs font-semibold text-secondary mb-2 flex items-center gap-1">
        <span class="material-symbols-outlined text-sm">flag</span> 마일스톤
      </div>
      ${milestones.length ? `<div class="space-y-1.5 mb-2">
        ${milestones.map((m,i) => `<div class="flex items-center gap-2 text-sm">
          <input type="checkbox" ${m.done?'checked':''} onchange="toggleProjectMilestone(${p.id},${m.id})"
            class="w-4 h-4 accent-primary flex-shrink-0">
          <span class="${m.done?'line-through text-secondary':''} flex-1">${m.title}</span>
          <span class="text-xs text-secondary">${m.date}</span>
          ${canEdit?`<button onclick="deleteProjectMilestone(${p.id},${m.id})" class="text-red-300 hover:text-red-500 p-0.5"><span class="material-symbols-outlined text-sm">close</span></button>`:''}
        </div>`).join('')}
      </div>` : '<p class="text-xs text-secondary mb-2">마일스톤이 없습니다</p>'}
      ${canEdit ? `<div class="flex gap-2">
        <input type="text" id="detail-ms-title-${p.id}" class="flex-1 border border-outline-variant rounded-lg px-2 py-1.5 text-xs" placeholder="마일스톤명">
        <input type="date" id="detail-ms-date-${p.id}" class="border border-outline-variant rounded-lg px-2 py-1.5 text-xs">
        <button onclick="addProjectMilestone(${p.id})" class="bg-surface-container border border-outline-variant rounded-lg px-2 py-1.5 text-xs text-secondary hover:bg-surface-container-high">추가</button>
      </div>` : ''}
    </div>

    <!-- 소속 태스크 -->
    <div class="mb-4">
      <div class="text-xs font-semibold text-secondary mb-2 flex items-center gap-1">
        <span class="material-symbols-outlined text-sm">task_alt</span> 소속 업무 (${tasks.length}건)
      </div>
      ${tasks.length ? `<div class="space-y-1.5">
        ${tasks.map(t => {
          const assigneeNames = (t.assigneeIds||[]).map(id=>DB.members.find(m=>m.id===id)?.name).filter(Boolean).join(', ');
          return `<div class="flex items-center gap-2 text-sm p-2 rounded-lg hover:bg-surface-container cursor-pointer" onclick="showTaskDetail(${t.id})">
            <span class="text-xs px-2 py-0.5 rounded-full font-medium flex-shrink-0 ${statusBadge(t.status)}">${t.status}</span>
            <span class="flex-1 text-xs font-medium truncate">${t.name}</span>
            <span class="text-xs text-secondary flex-shrink-0">${assigneeNames}</span>
            <span class="text-xs text-secondary flex-shrink-0">${t.deadline||'-'}</span>
          </div>`;
        }).join('')}
      </div>` : '<p class="text-xs text-secondary">연결된 업무가 없습니다</p>'}
    </div>

    <!-- 댓글 -->
    <div>
      <div class="text-xs font-semibold text-secondary mb-2 flex items-center gap-1">
        <span class="material-symbols-outlined text-sm">comment</span> 댓글 (${comments.length})
      </div>
      ${comments.map(c => {
        const canDelC = currentUser?.username === 'jungsoo.kim' || currentUser?.memberId === c.authorId;
        return `<div class="flex gap-2 mb-2">
          <div class="w-6 h-6 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold flex-shrink-0">${(c.authorName||'?')[0]}</div>
          <div class="flex-1 bg-surface-container-low rounded-xl px-3 py-2">
            <div class="flex justify-between items-center mb-0.5">
              <span class="text-xs font-semibold">${c.authorName}</span>
              <div class="flex items-center gap-1">
                <span class="text-xs text-secondary">${new Date(c.createdAt).toLocaleDateString('ko-KR')}</span>
                ${canDelC?`<button onclick="deleteProjectComment(${p.id},${c.id})" class="text-red-300 hover:text-red-500 p-0.5"><span class="material-symbols-outlined text-sm">close</span></button>`:''}
              </div>
            </div>
            <p class="text-xs text-on-surface">${c.text}</p>
          </div>
        </div>`;
      }).join('')}
      <div class="flex gap-2 mt-3">
        <input type="text" id="project-comment-input-${p.id}" class="flex-1 border border-outline-variant rounded-lg px-3 py-2 text-sm" placeholder="댓글을 입력하세요..." onkeydown="if(event.key==='Enter')addProjectComment(${p.id})">
        <button onclick="addProjectComment(${p.id})" class="bg-primary text-white rounded-lg px-4 py-2 text-sm font-semibold hover:bg-blue-700">등록</button>
      </div>
    </div>`;
  }
```

- [ ] **Step 2: render() return 블록에 buildDetailPanel 노출 확인**

이미 `render()` 내에서 `buildDetailPanel(...)` 호출을 사용하므로 별도 return 노출 불필요.

- [ ] **Step 3: 커밋**
```bash
git add index.html
git commit -m "feat: 프로젝트 상세 패널 + 간트 바 + 마일스톤/태스크/댓글 UI"
```

---

## Task 6: 프로젝트 CRUD + 마일스톤/댓글 액션

**Files:**
- Modify: `index.html` (Pages.projects IIFE return 블록 확장 + 전역 shim 추가)

- [ ] **Step 1: IIFE return 블록에 모든 함수 추가 및 내부 함수 구현**

Pages.projects IIFE의 `return {` 블록을 아래로 교체:

```js
  // ---- 담당자 드롭박스 ----
  function renderProjectAssigneeList() {
    const list = document.getElementById('project-assignee-list');
    if (!list) return;
    list.innerHTML = DB.members.map(m => `
      <label class="assignee-cb-row">
        <input type="checkbox" value="${m.id}" ${_selectedAssigneeIds.includes(m.id)?'checked':''}
          onchange="toggleProjectAssignee(${m.id})">
        <div class="w-5 h-5 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold flex-shrink-0">${m.name[0]}</div>
        <span>${m.name}</span>
      </label>`).join('');
    const btn = document.getElementById('project-assignee-label');
    if (btn) btn.textContent = _selectedAssigneeIds.length
      ? DB.members.filter(m => _selectedAssigneeIds.includes(m.id)).map(m=>m.name).join(', ')
      : '담당자 선택';
  }

  function toggleProjectAssigneeDropdown() {
    const panel = document.getElementById('project-assignee-panel');
    if (!panel) return;
    const isHidden = panel.classList.contains('hidden');
    panel.classList.toggle('hidden', !isHidden);
    if (isHidden) renderProjectAssigneeList();
  }

  function toggleProjectAssignee(id) {
    const idx = _selectedAssigneeIds.indexOf(id);
    if (idx === -1) _selectedAssigneeIds.push(id);
    else _selectedAssigneeIds.splice(idx, 1);
    renderProjectAssigneeList();
  }

  // ---- 모달 마일스톤 목록 렌더 ----
  function renderModalMilestones() {
    const list = document.getElementById('project-milestone-list');
    if (!list) return;
    list.innerHTML = _modalMilestones.map((m, i) => `
      <div class="flex items-center gap-2 text-xs text-secondary">
        <span class="material-symbols-outlined text-sm">flag</span>
        <span class="flex-1">${m.title}</span>
        <span>${m.date}</span>
        <button type="button" onclick="removeModalMilestone(${i})" class="text-red-300 hover:text-red-500">
          <span class="material-symbols-outlined text-sm">close</span>
        </button>
      </div>`).join('');
  }

  function addModalMilestone_fn() {
    const title = document.getElementById('new-milestone-title')?.value.trim();
    const date = document.getElementById('new-milestone-date')?.value;
    if (!title) { alert('마일스톤명을 입력하세요'); return; }
    _modalMilestones.push({ id: Date.now(), title, date: date || '', done: false });
    document.getElementById('new-milestone-title').value = '';
    document.getElementById('new-milestone-date').value = '';
    renderModalMilestones();
  }

  function removeModalMilestone_fn(idx) {
    _modalMilestones.splice(idx, 1);
    renderModalMilestones();
  }

  // ---- 프로젝트 추가 모달 열기 ----
  function openAddModal() {
    _modalMilestones = [];
    _selectedAssigneeIds = [];
    document.getElementById('editing-project-id').value = '';
    document.getElementById('project-title').value = '';
    document.getElementById('project-summary').value = '';
    document.getElementById('project-description').value = '';
    document.getElementById('project-start-date').value = '';
    document.getElementById('project-end-date').value = '';
    document.getElementById('project-modal-title').textContent = '새 프로젝트';
    document.getElementById('project-assignee-label').textContent = '담당자 선택';
    document.getElementById('project-assignee-panel').classList.add('hidden');
    renderModalMilestones();
    document.getElementById('modal-add-project').classList.remove('hidden');
  }

  // ---- 프로젝트 수정 모달 열기 ----
  function editProject_fn(id) {
    const p = (DB.projects||[]).find(p => p.id === id);
    if (!p) return;
    _modalMilestones = (p.milestones||[]).map(m => ({...m}));
    _selectedAssigneeIds = [...(p.assigneeIds||[])];
    document.getElementById('editing-project-id').value = id;
    document.getElementById('project-title').value = p.title;
    document.getElementById('project-summary').value = p.summary || '';
    document.getElementById('project-description').value = p.description || '';
    document.getElementById('project-start-date').value = p.startDate || '';
    document.getElementById('project-end-date').value = p.endDate || '';
    document.getElementById('project-modal-title').textContent = '프로젝트 수정';
    renderModalMilestones();
    renderProjectAssigneeList();
    document.getElementById('project-assignee-panel').classList.add('hidden');
    document.getElementById('modal-add-project').classList.remove('hidden');
  }

  // ---- 저장 ----
  function saveProject_fn() {
    const title = document.getElementById('project-title').value.trim();
    if (!title) { alert('프로젝트명을 입력하세요'); return; }
    const editId = document.getElementById('editing-project-id').value;
    const projectData = {
      title,
      summary: document.getElementById('project-summary').value.trim(),
      description: document.getElementById('project-description').value.trim(),
      startDate: document.getElementById('project-start-date').value,
      endDate: document.getElementById('project-end-date').value,
      assigneeIds: [..._selectedAssigneeIds],
      milestones: _modalMilestones.map(m => ({...m})),
    };
    if (editId) {
      const p = (DB.projects||[]).find(p => p.id === parseInt(editId));
      if (!p) return;
      Object.assign(p, projectData, { updatedAt: Date.now() });
      db.collection('projects').doc(String(p.id)).update({...projectData, updatedAt: Date.now()})
        .catch(e => console.error(e));
      showToast('프로젝트가 수정되었습니다');
    } else {
      const id = Date.now();
      const newProject = {
        id,
        ...projectData,
        comments: [],
        createdBy: currentUser?.username || '',
        createdAt: id,
      };
      if (!DB.projects) DB.projects = [];
      DB.projects.push(newProject);
      db.collection('projects').doc(String(id)).set(newProject)
        .catch(e => console.error(e));
      showToast('프로젝트가 추가되었습니다');
    }
    document.getElementById('modal-add-project').classList.add('hidden');
    render();
  }

  // ---- 삭제 ----
  function deleteProject_fn(id) {
    if (!confirm('프로젝트를 삭제하겠습니까?\n연결된 업무의 프로젝트 연결은 해제됩니다.')) return;
    DB.projects = (DB.projects||[]).filter(p => p.id !== id);
    DB.tasks.forEach(t => { if (t.projectId === id) t.projectId = null; });
    db.collection('projects').doc(String(id)).delete().catch(e => console.error(e));
    if (selectedProjectId === id) selectedProjectId = null;
    showToast('프로젝트가 삭제되었습니다');
    render();
  }

  // ---- 마일스톤 토글 (상세 패널) ----
  function toggleMilestone_fn(projectId, milestoneId) {
    const p = (DB.projects||[]).find(p => p.id === projectId);
    if (!p) return;
    const m = (p.milestones||[]).find(m => m.id === milestoneId);
    if (!m) return;
    m.done = !m.done;
    db.collection('projects').doc(String(projectId)).update({ milestones: p.milestones })
      .catch(e => console.error(e));
    render();
  }

  // ---- 마일스톤 추가 (상세 패널) ----
  function addMilestone_fn(projectId) {
    const p = (DB.projects||[]).find(p => p.id === projectId);
    if (!p) return;
    const title = document.getElementById(`detail-ms-title-${projectId}`)?.value.trim();
    const date = document.getElementById(`detail-ms-date-${projectId}`)?.value;
    if (!title) { alert('마일스톤명을 입력하세요'); return; }
    if (!p.milestones) p.milestones = [];
    p.milestones.push({ id: Date.now(), title, date: date || '', done: false });
    db.collection('projects').doc(String(projectId)).update({ milestones: p.milestones })
      .catch(e => console.error(e));
    render();
  }

  // ---- 마일스톤 삭제 (상세 패널) ----
  function deleteMilestone_fn(projectId, milestoneId) {
    const p = (DB.projects||[]).find(p => p.id === projectId);
    if (!p) return;
    p.milestones = (p.milestones||[]).filter(m => m.id !== milestoneId);
    db.collection('projects').doc(String(projectId)).update({ milestones: p.milestones })
      .catch(e => console.error(e));
    render();
  }

  // ---- 댓글 추가 ----
  function addComment_fn(projectId) {
    const p = (DB.projects||[]).find(p => p.id === projectId);
    const input = document.getElementById(`project-comment-input-${projectId}`);
    if (!p || !input || !input.value.trim()) return;
    const uploader = DB.members.find(m => m.id === currentUser?.memberId);
    if (!p.comments) p.comments = [];
    const comment = {
      id: Date.now(),
      text: input.value.trim(),
      authorId: currentUser?.memberId || null,
      authorName: uploader?.name || currentUser?.username || '?',
      createdAt: Date.now(),
    };
    p.comments.push(comment);
    db.collection('projects').doc(String(projectId)).update({ comments: p.comments })
      .catch(e => console.error(e));
    input.value = '';
    render();
  }

  // ---- 댓글 삭제 ----
  function deleteComment_fn(projectId, commentId) {
    const p = (DB.projects||[]).find(p => p.id === projectId);
    if (!p) return;
    p.comments = (p.comments||[]).filter(c => c.id !== commentId);
    db.collection('projects').doc(String(projectId)).update({ comments: p.comments })
      .catch(e => console.error(e));
    render();
  }

  return {
    render,
    selectProject,
    closeDetail,
    openAddModal,
    editProject: editProject_fn,
    saveProject: saveProject_fn,
    deleteProject: deleteProject_fn,
    toggleMilestone: toggleMilestone_fn,
    addMilestone: addMilestone_fn,
    deleteMilestone: deleteMilestone_fn,
    addComment: addComment_fn,
    deleteComment: deleteComment_fn,
    toggleProjectAssigneeDropdown,
    toggleProjectAssignee,
    addModalMilestone: addModalMilestone_fn,
    removeModalMilestone: removeModalMilestone_fn,
    get selectedId() { return selectedProjectId; },
  };
```

- [ ] **Step 2: 전역 shims 추가 (기존 `function selectProject` 블록 아래에)**

```js
function openAddProjectModal()              { Pages.projects.openAddModal(); }
function saveProject()                      { Pages.projects.saveProject(); }
function editProject(id)                    { Pages.projects.editProject(id); }
function deleteProject(id)                  { Pages.projects.deleteProject(id); }
function toggleProjectMilestone(pid, mid)   { Pages.projects.toggleMilestone(pid, mid); }
function addProjectMilestone(pid)           { Pages.projects.addMilestone(pid); }
function deleteProjectMilestone(pid, mid)   { Pages.projects.deleteMilestone(pid, mid); }
function addProjectComment(pid)             { Pages.projects.addComment(pid); }
function deleteProjectComment(pid, cid)     { Pages.projects.deleteComment(pid, cid); }
function toggleProjectAssigneeDropdown()    { Pages.projects.toggleProjectAssigneeDropdown(); }
function toggleProjectAssignee(id)          { Pages.projects.toggleProjectAssignee(id); }
function addModalMilestone()                { Pages.projects.addModalMilestone(); }
function removeModalMilestone(i)            { Pages.projects.removeModalMilestone(i); }
```

- [ ] **Step 3: 외부 클릭 시 담당자 패널 닫기 이벤트 추가**

기존 assignee/priority 외부 클릭 핸들러 블록 근처에 추가:
```js
document.addEventListener('click', function(e) {
  const wrap = document.getElementById('project-assignee-wrap');
  if (wrap && !wrap.contains(e.target)) {
    document.getElementById('project-assignee-panel')?.classList.add('hidden');
  }
});
```

- [ ] **Step 4: 브라우저 테스트**

1. 프로젝트 탭 → "+ 프로젝트 추가" 클릭 → 모달 열림 확인
2. 제목 입력 + 담당자 선택 + 저장 → 카드 생성 확인
3. 카드 클릭 → 좌측 목록 + 우측 상세 패널 열림 확인
4. 마일스톤 추가 → 간트 바에 마커 표시 확인
5. 댓글 입력/등록 확인

- [ ] **Step 5: 커밋**
```bash
git add index.html
git commit -m "feat: 프로젝트 CRUD + 마일스톤/댓글 액션 + 전역 shims"
```

---

## Task 7: 대시보드 연동 (태스크 모달 + 업무 목록 뱃지 + 요약 카드)

**Files:**
- Modify: `index.html` (dashboard 영역)

- [ ] **Step 1: 업무 추가/수정 모달에 프로젝트 선택 드롭박스 추가**

`id="task-center"` select 블록 바로 위에 삽입 (line ~885 근처):

```html
<div>
  <label class="block text-xs font-semibold text-secondary mb-1.5">프로젝트 <span class="font-normal text-secondary">(선택)</span></label>
  <select id="task-project" class="w-full border border-outline-variant rounded-lg px-3 py-2 text-sm bg-white">
    <option value="">없음</option>
  </select>
</div>
```

- [ ] **Step 2: openAddTaskModal()에서 프로젝트 드롭박스 채우기**

`openAddTaskModal()` 함수 내 `cSel.innerHTML = ...` 이후에 추가:
```js
// 프로젝트 드롭박스 채우기
const pSel = document.getElementById('task-project');
pSel.innerHTML = '<option value="">없음</option>' + (DB.projects||[]).map(p => `<option value="${p.id}">${p.title}</option>`).join('');
pSel.value = '';
```

- [ ] **Step 3: editTask()에서 프로젝트 드롭박스 채우기 및 현재값 복원**

`editTask()` 함수 내 `cSel.value = t.centerId || '';` 이후에 추가:
```js
const pSel2 = document.getElementById('task-project');
pSel2.innerHTML = '<option value="">없음</option>' + (DB.projects||[]).map(p => `<option value="${p.id}">${p.title}</option>`).join('');
pSel2.value = t.projectId || '';
```

- [ ] **Step 4: saveTask()에서 projectId 저장**

`saveTask()` 함수에서 `centerId` 읽는 줄 근처에 추가:
```js
const projectId = parseInt(document.getElementById('task-project').value) || null;
```

그리고 `Object.assign(t, {...})` 과 `DB.tasks.push({...})` 양쪽에 `projectId` 추가:
```js
// 수정 시
Object.assign(t, {name, assigneeIds, centerId, projectId, status, priority, deadline, desc, updatedAt: Date.now()});

// 신규 시
DB.tasks.push({id: Date.now(), name, assigneeIds, centerId, projectId, status, priority, deadline, desc, comments:[], createdAt: Date.now(), updatedAt: Date.now()});
```

- [ ] **Step 5: renderTaskTable()에서 업무명 위에 프로젝트 뱃지 표시**

`renderTaskTable()` 내 업무명 td 부분을 찾아서 (line ~1651) 업무명 `<span>` 위에 프로젝트 뱃지 추가:

```js
// 기존 코드에서 아래 부분을 찾아 교체:
// <span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span>

// 교체 후:
const proj = t.projectId ? (DB.projects||[]).find(p=>p.id===t.projectId) : null;
// 그 다음 해당 td의 flex div 안에서:
`${proj ? `<div class="flex flex-col">
  <span class="text-xs text-primary/70 font-medium leading-none mb-0.5">${proj.title}</span>
  <span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span>
</div>` : `<span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span>`}
${isNew(t)?'<span class="new-badge">NEW</span>':''}`
```

구체적으로 `renderTaskTable()` 내 tbody.innerHTML 템플릿 리터럴에서 아래 줄:
```js
<span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span>${isNew(t)?'<span class="new-badge">NEW</span>':''}
```
을 아래로 교체:
```js
${(() => { const proj = t.projectId ? (DB.projects||[]).find(p=>p.id===t.projectId) : null; return proj ? `<div class="flex flex-col"><span class="text-xs text-primary/70 font-semibold leading-none mb-0.5">${proj.title}</span><span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span></div>` : `<span class="font-medium text-on-surface text-sm whitespace-nowrap">${t.name}</span>`; })()}${isNew(t)?'<span class="new-badge">NEW</span>':''}
```

- [ ] **Step 6: 대시보드 상단 요약 카드에 "진행중 프로젝트" 추가**

대시보드 `render()` 함수 내 상단 요약 카드 4개를 렌더하는 부분을 찾아, 마지막 카드 이후에 5번째 카드 추가:

```js
const activeProjects = (DB.projects||[]).filter(p => {
  const tasks = DB.tasks.filter(t => t.projectId === p.id);
  return tasks.some(t => t.status !== '완료');
}).length;
```

그리고 기존 4카드 html 뒤에:
```html
<div class="stat-card cursor-pointer" onclick="showPage('projects')">
  <div class="flex items-center justify-between mb-2">
    <span class="text-xs font-semibold text-secondary">진행중 프로젝트</span>
    <span class="material-symbols-outlined text-secondary text-lg">folder_special</span>
  </div>
  <div class="text-2xl font-bold text-on-surface">${activeProjects}</div>
  <div class="text-xs text-secondary mt-1">프로젝트 탭 →</div>
</div>
```

- [ ] **Step 7: 브라우저 테스트**

1. 업무 추가 모달에서 "프로젝트" 드롭박스 표시 확인
2. 프로젝트 선택 후 저장 → 업무 목록 행에 프로젝트명 뱃지 표시 확인
3. 대시보드 상단에 "진행중 프로젝트" 카드 확인, 클릭 시 프로젝트 탭 이동 확인

- [ ] **Step 8: 최종 커밋 + 푸시**
```bash
git add index.html
git commit -m "feat: 대시보드 프로젝트 연동 — 태스크 모달/목록 뱃지/요약 카드"
git push origin master:main
```

---

## 셀프 리뷰

**스펙 커버리지:**
- ✅ projects Firestore 컬렉션 + DB 추가 (Task 1)
- ✅ tasks.projectId 필드 (Task 7)
- ✅ 3열 카드 그리드 → 마스터-디테일 (Task 4)
- ✅ 원형 진척률 게이지 SVG (Task 4)
- ✅ 상세 패널: 메타/진척률/간트/마일스톤/태스크/댓글 (Task 5)
- ✅ 프로젝트 추가/수정 모달 (Task 3/6)
- ✅ 담당자 체크박스 멀티선택 드롭박스 (Task 6)
- ✅ 마일스톤 CRUD + 간트 바 마커 (Task 5/6)
- ✅ 댓글 추가/삭제 (Task 6)
- ✅ 권한: 생성자+관리자만 수정/삭제 (Task 6)
- ✅ 사이드바 nav 추가 (Task 1)
- ✅ 업무 추가 모달 프로젝트 드롭박스 (Task 7)
- ✅ 업무 목록 프로젝트명 뱃지 (Task 7)
- ✅ 대시보드 진행중 프로젝트 요약 카드 (Task 7)

**타입 일관성:**
- `p.id`: number (Date.now())
- `m.id` (milestone): number (Date.now())
- `c.id` (comment): number (Date.now())
- `_selectedAssigneeIds`: number[]
- 모든 shim 함수명이 IIFE return과 일치 확인됨

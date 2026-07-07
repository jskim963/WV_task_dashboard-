# 주간일정 그리드 리뉴얼 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 데스크탑 "주간일정" 화면을 여러 주가 이어서 스크롤되는 그리드로 바꾸고, 셀 클릭(단일/Ctrl 다중/Ctrl+Shift 범위)으로 팝오버를 열어 기존 "일정 추가" 모달 없이 빠르게 일정을 입력·삭제할 수 있게 한다.

**Architecture:** 모든 변경은 `index.html`(단일 파일) 안의 `Pages.weekly` IIFE, 관련 CSS(`<style>` 블록), `#page-weekly` HTML 셸에 국한된다. 모바일 주간일정(요일 아코디언 스택)은 범위 밖이며 코드/동작을 그대로 유지한다. 새 상호작용은 module-level 상태(`selection` Set, `anchorCell`, `extraPastWeeks`/`extraFutureWeeks`)와 새 함수들을 `Pages.weekly`에 추가하는 방식으로 구현하고, 기존 `DB.events` 스키마·`saveDB()`/`deleteEvent()` 등 기존 함수를 그대로 재사용한다.

**Tech Stack:** 기존 그대로 (Vanilla JS, Tailwind CDN, Firebase Firestore). 이 프로젝트에는 자동화 테스트 프레임워크가 없으므로 각 태스크의 검증은 `grep`(코드 존재 확인)과 브라우저(`mcp__Claude_Preview__*`) 기반 수동/스크립트 검증으로 한다.

**⚠️ 중요 (모든 검증 태스크 공통):** 이 앱은 beta 브랜치에서도 운영과 **동일한 실시간 Firebase Firestore**를 사용한다([[beta-environment]] 설계 참고). 로컬 프리뷰로 클릭/저장을 테스트할 때 실제 팀 데이터에 테스트 데이터가 저장되지 않도록, 저장 동작을 검증할 때는 반드시 브라우저 콘솔에서 `saveDB = function(){}`로 **무력화한 뒤** 테스트한다 (아래 Task 9에 상세 절차 있음). 절대 실제 `saveDB()`가 Firestore에 커밋되게 둔 채로 테스트 데이터를 저장하지 말 것.

---

### Task 1: CSS 추가

**Files:**
- Modify: `index.html:249` (기존 `.wk-today` 규칙 바로 다음)

- [ ] **Step 1: 기존 위치 확인**

Run: `grep -n "wk-today" index.html`
Expected: `249:.wk-today { background:#eff6ff !important; }` 한 줄 출력 (다른 위치에서도 매칭될 수 있으나 CSS 정의 줄은 249).

- [ ] **Step 2: 새 CSS 규칙 삽입**

`index.html`의 다음 블록:

```css
.wk-today { background:#eff6ff !important; }
```

을 아래로 교체 (기존 줄 유지 + 새 규칙 추가):

```css
.wk-today { background:#eff6ff !important; }
.wk-scroll-area { max-height:65vh; overflow-y:auto; position:relative; }
.wk-table thead th { position:sticky; top:0; z-index:5; }
.wk-week-sep td { background:#1e293b; color:#fff; font-size:11px; font-weight:700; padding:6px 10px; text-align:left; }
.wk-selected { background:#dbeafe !important; box-shadow:inset 0 0 0 2px #2563eb; }
.wk-cell { cursor:pointer; }
.wk-more-btn { display:block; width:100%; text-align:center; padding:8px; font-size:12px; color:#64748b; background:#f8fafc; border:none; border-top:1px solid #e2e8f0; cursor:pointer; }
.wk-more-btn:hover { background:#f1f5f9; }
.wk-selection-bar { position:fixed; left:50%; bottom:24px; transform:translateX(-50%); background:#1e293b; color:#fff; padding:10px 16px; border-radius:999px; display:flex; align-items:center; gap:12px; font-size:13px; box-shadow:0 8px 20px rgba(0,0,0,.25); z-index:60; }
.wk-selection-bar button { background:#2563eb; color:#fff; border:none; padding:6px 14px; border-radius:999px; font-size:12px; font-weight:600; cursor:pointer; }
.wk-popover { position:fixed; z-index:70; border:1px solid #e2e8f0; border-radius:16px; padding:12px; width:230px; background:#fff; box-shadow:0 10px 24px rgba(0,0,0,.16); }
.wk-popover .wk-pop-existing { font-size:12px; background:#f1f5f9; border-radius:10px; padding:6px 10px; margin-bottom:6px; display:flex; justify-content:space-between; align-items:center; gap:6px; }
.wk-popover .wk-pop-existing button { border:none; background:none; color:#94a3b8; cursor:pointer; font-size:13px; padding:0; }
.wk-popover input.wk-pop-input { width:100%; border:1px solid #d1d5db; border-radius:10px; padding:8px 12px; font-size:13px; margin-bottom:10px; box-sizing:border-box; }
.wk-pop-chip { display:inline-block; padding:5px 12px; border-radius:999px; font-size:11px; margin-right:5px; margin-bottom:8px; font-weight:600; cursor:pointer; border:2px solid transparent; }
.wk-pop-chip.active { border-color:currentColor; }
.wk-pop-save { width:100%; border:none; border-radius:10px; padding:9px; font-size:12px; font-weight:600; background:#2563eb; color:#fff; cursor:pointer; }
.wk-pop-context { font-size:11px; color:#64748b; margin-bottom:8px; font-weight:600; }
```

- [ ] **Step 3: 확인**

Run: `grep -c "wk-pop-chip\|wk-selection-bar\|wk-scroll-area" index.html`
Expected: `6` 이상 (선언부 + 사용부 합산이라 이후 태스크 진행하면서 늘어남; 지금 시점엔 CSS 선언만 있으므로 최소 3 이상이면 됨).

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 리뉴얼용 CSS 추가 (선택 하이라이트/팝오버/액션바)"
```

---

### Task 2: HTML 셸 재구성 (데스크탑/모바일 툴바 분리 + 팝오버·액션바 컨테이너)

**Files:**
- Modify: `index.html:664-692` (`#page-weekly` 전체 블록)

- [ ] **Step 1: 현재 블록 재확인**

Run: `grep -n "PAGE: WEEKLY" index.html`
Expected: `664:<!-- ====== PAGE: WEEKLY ====== -->` (라인 번호가 앞선 태스크의 CSS 삽입으로 몇 줄 밀렸을 수 있음 — grep 결과 줄 번호를 기준으로 삼을 것).

- [ ] **Step 2: 블록 전체 교체**

다음 기존 블록을:

```html
<!-- ====== PAGE: WEEKLY ====== -->
<div id="page-weekly" class="page">
  <div class="flex items-center justify-between mb-6">
    <div>
      <h1 class="text-xl lg:text-2xl font-bold text-on-surface" style="font-family:'Hanken Grotesk'">주간 출장/휴가 현황</h1>
      <p class="text-sm text-secondary mt-0.5" id="weekly-range-label"></p>
    </div>
    <button onclick="openAddEventModal()" class="flex items-center gap-2 bg-primary text-white px-4 py-2.5 rounded-lg text-sm font-semibold hover:bg-blue-700 transition-all shadow-sm">
      <span class="material-symbols-outlined text-base">add</span> 일정 추가
    </button>
  </div>
  <div class="bg-white rounded-xl border border-outline-variant shadow-sm p-4">
    <div class="flex items-center gap-3 mb-4 flex-wrap">
      <button onclick="changeWeek(-1)" class="p-1.5 rounded-lg hover:bg-surface-container text-secondary"><span class="material-symbols-outlined">chevron_left</span></button>
      <button onclick="changeWeek(1)" class="p-1.5 rounded-lg hover:bg-surface-container text-secondary"><span class="material-symbols-outlined">chevron_right</span></button>
      <button onclick="goToThisWeek()" class="px-3 py-1 border border-outline-variant rounded-lg text-sm text-secondary hover:bg-surface-container">이번 주</button>
      <div class="flex gap-3 text-xs text-secondary ml-2">
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#fee2e2"></span>휴무</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#dbeafe"></span>출장/외근</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#fef9c3"></span>미팅/회의</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#ede9fe"></span>기타</span>
      </div>
    </div>
    <!-- 데스크탑 테이블 -->
    <div class="overflow-x-auto hidden lg:block" id="weekly-content"></div>
    <!-- 모바일 요일 스택 -->
    <div class="lg:hidden flex flex-col gap-2" id="weekly-mobile-stack"></div>
  </div>
</div>
```

아래로 교체한다 (모바일 전용 툴바와 데스크탑 전용 툴바를 분리하고, `#weekly-content`에 `wk-scroll-area` 클래스를 추가하고, 팝오버·선택 액션바 컨테이너를 추가):

```html
<!-- ====== PAGE: WEEKLY ====== -->
<div id="page-weekly" class="page">
  <div class="flex items-center justify-between mb-6">
    <div>
      <h1 class="text-xl lg:text-2xl font-bold text-on-surface" style="font-family:'Hanken Grotesk'">주간 출장/휴가 현황</h1>
      <p class="text-sm text-secondary mt-0.5" id="weekly-range-label"></p>
    </div>
    <button onclick="openAddEventModal()" class="flex items-center gap-2 bg-primary text-white px-4 py-2.5 rounded-lg text-sm font-semibold hover:bg-blue-700 transition-all shadow-sm">
      <span class="material-symbols-outlined text-base">add</span> 일정 추가
    </button>
  </div>
  <div class="bg-white rounded-xl border border-outline-variant shadow-sm p-4">
    <div class="flex items-center gap-3 mb-4 flex-wrap">
      <!-- 모바일 전용: 기존 주 단위 네비게이션 유지 -->
      <div class="lg:hidden flex items-center gap-3">
        <button onclick="changeWeek(-1)" class="p-1.5 rounded-lg hover:bg-surface-container text-secondary"><span class="material-symbols-outlined">chevron_left</span></button>
        <button onclick="changeWeek(1)" class="p-1.5 rounded-lg hover:bg-surface-container text-secondary"><span class="material-symbols-outlined">chevron_right</span></button>
        <button onclick="goToThisWeek()" class="px-3 py-1 border border-outline-variant rounded-lg text-sm text-secondary hover:bg-surface-container">이번 주</button>
      </div>
      <!-- 데스크탑 전용: 연속 스크롤이라 이번 주로 이동만 제공 -->
      <div class="hidden lg:flex items-center gap-3">
        <button onclick="Pages.weekly.scrollToThisWeek()" class="px-3 py-1 border border-outline-variant rounded-lg text-sm text-secondary hover:bg-surface-container">이번 주로 이동</button>
      </div>
      <div class="flex gap-3 text-xs text-secondary ml-2">
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#fee2e2"></span>휴무</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#dbeafe"></span>출장/외근</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#fef9c3"></span>미팅/회의</span>
        <span class="flex items-center gap-1.5"><span class="w-3 h-3 rounded-sm inline-block" style="background:#ede9fe"></span>기타</span>
      </div>
    </div>
    <!-- 데스크탑 테이블 (연속 스크롤 영역) -->
    <div class="overflow-x-auto hidden lg:block wk-scroll-area" id="weekly-content"></div>
    <!-- 모바일 요일 스택 (기존과 동일, 변경 없음) -->
    <div class="lg:hidden flex flex-col gap-2" id="weekly-mobile-stack"></div>
  </div>
</div>
<!-- 셀 클릭 팝오버 / 다중선택 액션바 (주간일정 전용, 최초엔 비어있음) -->
<div id="wk-popover" class="wk-popover hidden"></div>
<div id="wk-selection-bar" class="wk-selection-bar hidden"></div>
```

- [ ] **Step 3: 확인**

Run: `grep -n 'id="wk-popover"\|id="wk-selection-bar"\|wk-scroll-area\|scrollToThisWeek' index.html`
Expected: 4개 항목 모두 매칭되어 출력됨 (`wk-popover` div, `wk-selection-bar` div, `weekly-content`에 붙은 `wk-scroll-area` 클래스, `scrollToThisWeek` 버튼 onclick).

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 데스크탑/모바일 툴바 분리 + 팝오버/액션바 컨테이너 추가"
```

---

### Task 3: `Pages.weekly` 상태값 및 헬퍼 함수 추가

**Files:**
- Modify: `index.html` (`Pages.weekly` IIFE 시작 부분, `render()` 함수 정의 직전)

- [ ] **Step 1: 삽입 위치 확인**

Run: `grep -n "Pages.weekly = (() => {" index.html`
Expected: 한 줄 출력. 그 다음 줄이 `let weekStart = getWeekMonday(new Date());` 인지 확인:

Run: `grep -n "let weekStart = getWeekMonday" index.html`

- [ ] **Step 2: 상태값 + 헬퍼 함수 삽입**

다음 코드를:

```js
Pages.weekly = (() => {
  let weekStart = getWeekMonday(new Date());

  function render() {
```

아래로 교체 (기존 두 줄 사이에 상태값·상수·헬퍼 함수 삽입):

```js
Pages.weekly = (() => {
  let weekStart = getWeekMonday(new Date());
  const BASE_PAST_WEEKS = 2, BASE_FUTURE_WEEKS = 6, EXTEND_CHUNK = 4;
  let extraPastWeeks = 0, extraFutureWeeks = 0;
  let selection = new Set();
  let anchorCell = null;
  let popoverMode = null, popoverMember = null, popoverDate = null, popoverType = null;
  const EVENT_COLORS = {휴무:'background:#fee2e2;color:#dc2626', 출장:'background:#dbeafe;color:#1d4ed8', 미팅:'background:#fef9c3;color:#ca8a04', 업무보고:'background:#dcfce7;color:#16a34a', 기타:'background:#ede9fe;color:#7c3aed'};
  const TYPE_LIST = ['출장','휴무','미팅','기타'];
  const TYPE_ICON = {출장:'🔵', 휴무:'🔴', 미팅:'🟡', 기타:'🟣'};

  function getCellEvents(memberId, dateStr) {
    return DB.events.filter(e => e.type !== '업무보고' && eventFallsOnDate(e, dateStr) && (isTeamEvent(e) || getEventPersonIds(e).includes(memberId)));
  }

  function getVisibleDateList() {
    const start = new Date(weekStart);
    start.setDate(start.getDate() - 7 * (BASE_PAST_WEEKS + extraPastWeeks));
    const totalWeeks = BASE_PAST_WEEKS + extraPastWeeks + 1 + BASE_FUTURE_WEEKS + extraFutureWeeks;
    const list = [];
    for (let i = 0; i < totalWeeks * 7; i++) {
      const d = new Date(start);
      d.setDate(start.getDate() + i);
      list.push(formatDate(d));
    }
    return list;
  }

  function addRectToSelection(a, b) {
    const memberIds = DB.members.map(m => m.id);
    const dateList = getVisibleDateList();
    const mi1 = memberIds.indexOf(a.memberId), mi2 = memberIds.indexOf(b.memberId);
    const di1 = dateList.indexOf(a.dateStr), di2 = dateList.indexOf(b.dateStr);
    if (mi1 === -1 || mi2 === -1 || di1 === -1 || di2 === -1) return;
    const mLo = Math.min(mi1, mi2), mHi = Math.max(mi1, mi2);
    const dLo = Math.min(di1, di2), dHi = Math.max(di1, di2);
    for (let mi = mLo; mi <= mHi; mi++) {
      for (let di = dLo; di <= dHi; di++) {
        selection.add(memberIds[mi] + '|' + dateList[di]);
      }
    }
  }

  function formatDateLabel(dateStr) {
    const d = new Date(dateStr);
    const days = ['일','월','화','수','목','금','토'];
    return `${d.getMonth()+1}/${d.getDate()}(${days[d.getDay()]})`;
  }

  function render() {
```

- [ ] **Step 3: 확인**

Run: `grep -n "BASE_PAST_WEEKS\|function addRectToSelection\|function getVisibleDateList" index.html`
Expected: 3개 이상 매칭.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 다중선택용 상태값/헬퍼 함수 추가"
```

---

### Task 4: `render()` 데스크탑 부분을 다중 주 렌더링으로 교체

**Files:**
- Modify: `index.html` (`Pages.weekly.render()` 함수 본문)

- [ ] **Step 1: 현재 `render()` 전체 재확인**

Run: `grep -n "function render() {" index.html`

주간일정 파일의 `render()`는 데스크탑 테이블 렌더링 부분(`weekly-content`까지)과 모바일 스택 렌더링 부분(`weekly-mobile-stack` 이후)으로 나뉜다. 이번 태스크는 **데스크탑 부분만** 교체한다.

- [ ] **Step 2: 데스크탑 렌더링 블록 교체**

다음 코드를 (Task 3 이후 기준, `render()` 함수 시작부터 `document.getElementById('weekly-content').innerHTML = ...` 줄까지):

```js
  function render() {
    if (!requireAuth()) return;
    const days = Array.from({length:7}, (_,i) => { const d = new Date(weekStart); d.setDate(weekStart.getDate()+i); return d; });
    const todayStr = formatDate(new Date());
    const dayNames = ['월','화','수','목','금','토','일'];
    const endD = days[6];
    document.getElementById('weekly-range-label').textContent =
      `${weekStart.getFullYear()}년 ${weekStart.getMonth()+1}월 ${weekStart.getDate()}일 - ${endD.getMonth()+1}월 ${endD.getDate()}일`;

    const evColors = {휴무:'background:#fee2e2;color:#dc2626', 출장:'background:#dbeafe;color:#1d4ed8', 미팅:'background:#fef9c3;color:#ca8a04', 업무보고:'background:#dcfce7;color:#16a34a', 기타:'background:#ede9fe;color:#7c3aed'};

    const thead = `<tr>
      <th class="px-3 py-2 text-left" style="min-width:90px">팀원</th>
      ${days.map((d,i) => {
        const ds = formatDate(d); const isToday = ds === todayStr;
        const isSat = i===5; const isSun = i===6;
        return `<th class="px-2 py-2 ${isToday?'wk-today':''}"
          <div class="text-xs font-semibold ${isSun?'text-red-500':isSat?'text-blue-500':isToday?'text-primary':''}">${dayNames[i]}</div>
          <div class="text-xs font-normal text-secondary">${d.getMonth()+1}/${d.getDate()}</div>
        </th>`;
      }).join('')}
    </tr>`;

    const tbody = DB.members.map(m => {
      const cells = days.map((d,i) => {
        const ds = formatDate(d); const isToday = ds === todayStr; const isSat=i===5; const isSun=i===6;
        const evs = DB.events.filter(e => e.type !== '업무보고' && eventFallsOnDate(e, ds) && (isTeamEvent(e) || getEventPersonIds(e).includes(m.id)));
        const evHtml = evs.map(e => {
          const s = evColors[e.type] || evColors['기타'];
          const label = (e.personId===0?'(팀) ':'')+e.title;
          return `<div class="wk-event" style="${s}" onclick="openEventDetail(${e.id})" title="${label}">${label}</div>`;
        }).join('');
        return `<td class="px-2 py-1 wk-cell ${isToday?'wk-today':isSat||isSun?'bg-slate-50':''}">${evHtml}</td>`;
      }).join('');
      return `<tr>
        <td class="px-2 py-2 bg-white" style="vertical-align:middle">
          <div class="flex items-center gap-1.5">
            <div class="w-6 h-6 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold flex-shrink-0">${m.name[0]}</div>
            <div class="flex items-baseline gap-1 min-w-0">
              <span class="text-xs font-semibold whitespace-nowrap">${m.name}</span>
              <span class="text-xs text-secondary whitespace-nowrap opacity-70">${m.rank||''}</span>
            </div>
          </div>
        </td>${cells}
      </tr>`;
    }).join('');

    document.getElementById('weekly-content').innerHTML =
      `<table class="wk-table"><thead>${thead}</thead><tbody>${tbody}</tbody></table>`;
```

아래로 교체 (요일 이름만 있는 고정 헤더 + 여러 주 블록을 순회하며 구분줄과 팀원 행을 반복 생성, 셀에 `data-member`/`data-date`와 `cellClick` 연결):

```js
  function render() {
    if (!requireAuth()) return;
    const days = Array.from({length:7}, (_,i) => { const d = new Date(weekStart); d.setDate(weekStart.getDate()+i); return d; });
    const todayStr = formatDate(new Date());
    const dayNames = ['월','화','수','목','금','토','일'];
    const endD = days[6];
    document.getElementById('weekly-range-label').textContent =
      `${weekStart.getFullYear()}년 ${weekStart.getMonth()+1}월 ${weekStart.getDate()}일 - ${endD.getMonth()+1}월 ${endD.getDate()}일`;

    const totalWeeks = BASE_PAST_WEEKS + extraPastWeeks + 1 + BASE_FUTURE_WEEKS + extraFutureWeeks;
    const rangeStart = new Date(weekStart);
    rangeStart.setDate(rangeStart.getDate() - 7 * (BASE_PAST_WEEKS + extraPastWeeks));

    const thead = `<tr>
      <th class="px-3 py-2 text-left" style="min-width:90px">팀원</th>
      ${dayNames.map(n => `<th class="px-2 py-2">${n}</th>`).join('')}
    </tr>`;

    let tbody = '';
    for (let w = 0; w < totalWeeks; w++) {
      const wkMonday = new Date(rangeStart);
      wkMonday.setDate(rangeStart.getDate() + w * 7);
      const wkDays = Array.from({length:7}, (_,i) => { const d = new Date(wkMonday); d.setDate(wkMonday.getDate()+i); return d; });
      const wkEnd = wkDays[6];
      const mondayKey = formatDate(wkMonday);
      tbody += `<tr class="wk-week-sep" id="wk-week-${mondayKey}"><td colspan="8">${wkMonday.getMonth()+1}/${wkMonday.getDate()} ~ ${wkEnd.getMonth()+1}/${wkEnd.getDate()}</td></tr>`;
      tbody += DB.members.map(m => {
        const cells = wkDays.map((d,i) => {
          const ds = formatDate(d);
          const isToday = ds === todayStr;
          const isWeekend = i===5 || i===6;
          const evs = getCellEvents(m.id, ds);
          const evHtml = evs.map(e => {
            const s = EVENT_COLORS[e.type] || EVENT_COLORS['기타'];
            const label = (isTeamEvent(e)?'(팀) ':'') + e.title;
            return `<div class="wk-event" style="${s}" onclick="event.stopPropagation();openEventDetail(${e.id})" title="${label}">${label}</div>`;
          }).join('');
          return `<td class="px-2 py-1 wk-cell ${isToday?'wk-today':isWeekend?'bg-slate-50':''}" data-member="${m.id}" data-date="${ds}" onclick="Pages.weekly.cellClick(event, ${m.id}, '${ds}')">${evHtml}</td>`;
        }).join('');
        return `<tr>
          <td class="px-2 py-2 bg-white" style="vertical-align:middle">
            <div class="flex items-center gap-1.5">
              <div class="w-6 h-6 rounded-full bg-primary text-white flex items-center justify-center text-xs font-bold flex-shrink-0">${m.name[0]}</div>
              <div class="flex items-baseline gap-1 min-w-0">
                <span class="text-xs font-semibold whitespace-nowrap">${m.name}</span>
                <span class="text-xs text-secondary whitespace-nowrap opacity-70">${m.rank||''}</span>
              </div>
            </div>
          </td>${cells}
        </tr>`;
      }).join('');
    }

    document.getElementById('weekly-content').innerHTML =
      `<button class="wk-more-btn" onclick="Pages.weekly.extendPast()">↑ 이전 주 더보기</button>
       <table class="wk-table"><thead>${thead}</thead><tbody>${tbody}</tbody></table>
       <button class="wk-more-btn" onclick="Pages.weekly.extendFuture()">↓ 다음 주 더보기</button>`;
    renderSelectionHighlight();
```

- [ ] **Step 3: 모바일 스택 렌더링의 `mobileEvColors`를 공용 `EVENT_COLORS`로 통일**

바로 이어지는 모바일 렌더링 블록에서 다음 줄을:

```js
    const mobileEvColors = {휴무:'background:#fee2e2;color:#dc2626', 출장:'background:#dbeafe;color:#1d4ed8', 미팅:'background:#fef9c3;color:#ca8a04', 업무보고:'background:#dcfce7;color:#16a34a', 기타:'background:#ede9fe;color:#7c3aed'};
```

삭제하고, 같은 블록 안에서 사용되는:

```js
        const s = mobileEvColors[e.type] || mobileEvColors['기타'];
```

를 다음으로 교체한다 (모바일 렌더링 로직 자체는 동일, 색상표만 Task 3에서 추가한 공용 상수 재사용):

```js
        const s = EVENT_COLORS[e.type] || EVENT_COLORS['기타'];
```

나머지 모바일 스택 코드(카드 HTML, `toggleWkCard` 호출 등)는 전혀 건드리지 않는다.

- [ ] **Step 4: 확인**

Run: `grep -n "data-member=\|wk-week-sep\|mobileEvColors" index.html`
Expected: `data-member=` 매칭 1개(템플릿 리터럴 안), `wk-week-sep` 매칭 2개 이상(CSS 선언 + JS 템플릿), `mobileEvColors`는 **0건**(완전히 제거됐어야 함).

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 데스크탑 렌더링을 다중 주 이어보기로 교체"
```

---

### Task 5: 셀 클릭 핸들러 + 선택 하이라이트 + 액션바

**Files:**
- Modify: `index.html` (`render()` 함수 종료 직후, `function changeWeek(dir) {` 직전)

- [ ] **Step 1: 삽입 위치 확인**

Run: `grep -n "function changeWeek(dir) {" index.html`
Expected: `Pages.weekly` IIFE 안의 `changeWeek` 정의 줄 (파일 하단 전역 shim `function changeWeek(dir) { Pages.weekly.changeWeek(dir); }`과는 다른, IIFE 내부의 것). 앞뒤 문맥으로 구분: 바로 위에 `render()` 함수의 마지막 줄(모바일 카드 자동 펼침 로직)이 있어야 한다.

- [ ] **Step 2: 함수 추가**

`render()` 함수의 닫는 `}` 바로 다음, `function changeWeek(dir) {` 바로 앞에 아래 함수들을 삽입:

```js
  function renderSelectionHighlight() {
    document.querySelectorAll('#weekly-content td[data-member]').forEach(td => {
      const key = td.dataset.member + '|' + td.dataset.date;
      td.classList.toggle('wk-selected', selection.has(key));
    });
    updateSelectionBar();
  }

  function updateSelectionBar() {
    const bar = document.getElementById('wk-selection-bar');
    if (selection.size === 0) { bar.classList.add('hidden'); bar.innerHTML = ''; return; }
    bar.innerHTML = `<span>${selection.size}칸 선택됨</span><button onclick="Pages.weekly.openBulkPopover()">일정 입력</button>`;
    bar.classList.remove('hidden');
  }

  function cellClick(evt, memberId, dateStr) {
    if (evt.ctrlKey && evt.shiftKey) {
      if (!anchorCell) anchorCell = { memberId, dateStr };
      addRectToSelection(anchorCell, { memberId, dateStr });
      renderSelectionHighlight();
      return;
    }
    if (evt.ctrlKey) {
      const key = memberId + '|' + dateStr;
      if (selection.has(key)) selection.delete(key); else selection.add(key);
      anchorCell = { memberId, dateStr };
      renderSelectionHighlight();
      return;
    }
    selection.clear();
    anchorCell = { memberId, dateStr };
    renderSelectionHighlight();
    openCellPopover(evt.currentTarget, memberId, dateStr);
  }

```

(다음 태스크에서 `openCellPopover`를 바로 이어서 추가하므로, 지금은 참조만 하고 아직 정의되지 않은 상태 — Task 6을 이어서 진행할 것.)

- [ ] **Step 3: 확인**

Run: `grep -n "function cellClick\|function renderSelectionHighlight\|function updateSelectionBar" index.html`
Expected: 3줄 모두 매칭.

- [ ] **Step 4: 커밋 (Task 6과 함께 커밋 — 이 시점에서는 `openCellPopover` 미정의라 아직 커밋하지 않고 다음 태스크로 진행)**

이 태스크는 Task 6과 이어서 커밋한다 (미완성 상태로 커밋하면 `openCellPopover is not defined` 런타임 에러가 나므로).

---

### Task 6: 팝오버 열기/닫기 + 기존 항목 삭제 + 단일 저장

**Files:**
- Modify: `index.html` (Task 5에서 추가한 `cellClick` 함수 바로 다음)

- [ ] **Step 1: 함수 추가**

Task 5에서 추가한 `cellClick` 함수의 닫는 `}` 바로 다음에 이어서 삽입:

```js
  function positionPopover(pop, anchorEl) {
    const rect = anchorEl.getBoundingClientRect();
    const popWidth = 230;
    let left = rect.left;
    if (left + popWidth > window.innerWidth - 12) left = window.innerWidth - popWidth - 12;
    pop.style.left = Math.max(12, left) + 'px';
    pop.style.top = (rect.bottom + 6) + 'px';
  }

  function chipsHtml() {
    return TYPE_LIST.map(t => `<span class="wk-pop-chip" data-type="${t}" style="${EVENT_COLORS[t]}" onclick="Pages.weekly.chipClick('${t}')">${TYPE_ICON[t]}${t}</span>`).join('');
  }

  function chipClick(type) {
    popoverType = type;
    document.querySelectorAll('#wk-popover .wk-pop-chip').forEach(el => el.classList.toggle('active', el.dataset.type === type));
    const input = document.getElementById('wk-pop-input');
    if (input && !input.value.trim()) input.value = type;
  }

  function openCellPopover(cellEl, memberId, dateStr) {
    popoverMode = 'single'; popoverMember = memberId; popoverDate = dateStr; popoverType = null;
    const member = DB.members.find(m => m.id === memberId);
    const existing = getCellEvents(memberId, dateStr);
    const existingHtml = existing.map(e => {
      const del = isTeamEvent(e) ? '' : `<button onclick="Pages.weekly.deleteExisting(${e.id})">✕</button>`;
      const label = (isTeamEvent(e) ? '(팀) ' : '') + e.title;
      return `<div class="wk-pop-existing"><span>${label}</span>${del}</div>`;
    }).join('');
    const pop = document.getElementById('wk-popover');
    pop.innerHTML = `
      <div class="wk-pop-context">${member ? member.name : ''} · ${formatDateLabel(dateStr)}</div>
      ${existingHtml}
      <input type="text" class="wk-pop-input" id="wk-pop-input" placeholder="새 일정 추가..." onkeydown="if(event.key==='Enter')Pages.weekly.saveSingle()">
      <div>${chipsHtml()}</div>
      <button class="wk-pop-save" onclick="Pages.weekly.saveSingle()">추가</button>
    `;
    positionPopover(pop, cellEl);
    pop.classList.remove('hidden');
    document.getElementById('wk-pop-input').focus();
  }

  function closePopover() {
    const pop = document.getElementById('wk-popover');
    pop.classList.add('hidden');
    pop.innerHTML = '';
    popoverMode = null;
  }

  function saveSingle() {
    const input = document.getElementById('wk-pop-input');
    const title = input.value.trim();
    if (!title) return;
    const type = popoverType || '기타';
    DB.events.push({ id: Date.now(), title, date: popoverDate, personId: popoverMember, personIds: [popoverMember], type, memo: '', createdAt: Date.now() });
    saveDB();
    showToast('일정이 추가되었습니다');
    closePopover();
    render();
  }

  function deleteExisting(id) {
    closePopover();
    deleteEvent(id);
  }

```

- [ ] **Step 2: 확인**

Run: `grep -n "function openCellPopover\|function saveSingle\|function deleteExisting\|function closePopover" index.html`
Expected: 4줄 모두 매칭.

- [ ] **Step 3: 커밋 (Task 5 + Task 6 함께)**

```bash
git add index.html
git commit -m "feat: 주간일정 셀 클릭 팝오버 — 단일 추가/삭제, 선택 하이라이트"
```

---

### Task 7: 다중선택 일괄 저장 (Bulk save)

**Files:**
- Modify: `index.html` (Task 6에서 추가한 `deleteExisting` 함수 바로 다음)

- [ ] **Step 1: 함수 추가**

`deleteExisting` 함수의 닫는 `}` 바로 다음에 삽입:

```js
  function openBulkPopover() {
    if (selection.size === 0) return;
    popoverMode = 'bulk'; popoverType = null;
    const pop = document.getElementById('wk-popover');
    pop.innerHTML = `
      <div class="wk-pop-context">${selection.size}칸에 동일하게 입력됩니다</div>
      <input type="text" class="wk-pop-input" id="wk-pop-input" placeholder="예: 출장(이천2)" onkeydown="if(event.key==='Enter')Pages.weekly.saveBulk()">
      <div>${chipsHtml()}</div>
      <button class="wk-pop-save" onclick="Pages.weekly.saveBulk()">저장 (${selection.size}건 생성)</button>
    `;
    const bar = document.getElementById('wk-selection-bar');
    positionPopover(pop, bar);
    pop.classList.remove('hidden');
    document.getElementById('wk-pop-input').focus();
  }

  function saveBulk() {
    const input = document.getElementById('wk-pop-input');
    const title = input.value.trim();
    if (!title) return;
    const type = popoverType || '기타';
    let i = 0;
    selection.forEach(key => {
      const [memberIdStr, dateStr] = key.split('|');
      DB.events.push({ id: Date.now() + i, title, date: dateStr, personId: parseInt(memberIdStr), personIds: [parseInt(memberIdStr)], type, memo: '', createdAt: Date.now() });
      i++;
    });
    const count = selection.size;
    saveDB();
    showToast(count + '건의 일정이 추가되었습니다');
    selection.clear();
    closePopover();
    render();
  }

```

- [ ] **Step 2: 확인**

Run: `grep -n "function openBulkPopover\|function saveBulk" index.html`
Expected: 2줄 모두 매칭.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 다중선택 일괄 입력(Bulk save) 추가"
```

---

### Task 8: "더보기"/"이번 주로 이동" + 바깥클릭·Esc 처리 + public API 노출

**Files:**
- Modify: `index.html` (기존 `changeWeek`/`goThisWeek`/`toggleCard` 함수 및 `return {...}` 부분)

- [ ] **Step 1: `extendPast`/`extendFuture`/`scrollToThisWeek` 추가**

Task 7에서 추가한 `saveBulk` 함수의 닫는 `}` 바로 다음, 기존 `function changeWeek(dir) {` 바로 앞에 삽입:

```js
  function extendPast() {
    const container = document.getElementById('weekly-content');
    const oldHeight = container.scrollHeight;
    const oldTop = container.scrollTop;
    extraPastWeeks += EXTEND_CHUNK;
    render();
    container.scrollTop = oldTop + (container.scrollHeight - oldHeight);
  }

  function extendFuture() {
    extraFutureWeeks += EXTEND_CHUNK;
    render();
  }

  function scrollToThisWeek() {
    const el = document.getElementById('wk-week-' + formatDate(weekStart));
    if (el) el.scrollIntoView({ block: 'start' });
  }

```

- [ ] **Step 2: `return {...}` 객체 갱신**

기존:

```js
  return { render, changeWeek, goThisWeek, toggleCard,
           get weekStart() { return weekStart; } };
})();
```

를 아래로 교체:

```js
  return { render, changeWeek, goThisWeek, toggleCard,
           cellClick, openBulkPopover, saveSingle, saveBulk, chipClick,
           deleteExisting, closePopover, extendPast, extendFuture, scrollToThisWeek,
           get weekStart() { return weekStart; } };
})();

document.addEventListener('click', (e) => {
  if (e.target.closest('.wk-table td')) return;
  if (e.target.closest('#wk-popover')) return;
  if (e.target.closest('#wk-selection-bar')) return;
  Pages.weekly.closePopover();
});
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    Pages.weekly.closePopover();
  }
});
```

- [ ] **Step 3: 확인**

Run: `grep -n "function extendPast\|function scrollToThisWeek\|closePopover();$" index.html`
Expected: 관련 줄들이 매칭됨 (정확한 패턴은 파일 내용에 맞게 확인).

Run: `node -e "new Function(require('fs').readFileSync('index.html','utf8').match(/<script id=\"tailwind-config\"[\s\S]*?<\/script>/)?'':'')" 2>/dev/null; echo "skip-syntax-shortcut"`

대신 다음으로 JS 문법 오류 여부를 확인한다 (전체 `<script>` 태그 내용을 추출해 Node로 파싱만 시도):

Run:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const scripts = [...html.matchAll(/<script(?![^>]*src)[^>]*>([\s\S]*?)<\/script>/g)].map(m => m[1]);
const code = scripts.join('\n;\n');
new Function(code);
console.log('OK: no syntax errors');
"
```
Expected: `OK: no syntax errors` (구문 오류가 있으면 `SyntaxError`와 함께 줄 정보가 출력되므로 그걸 보고 수정한다).

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 주간일정 더보기/이번주이동 + 바깥클릭·Esc로 팝오버 닫기"
```

---

### Task 9: 브라우저 수동 검증 (실 데이터 저장 금지) + 최종 커밋

**Files:** 없음 (검증만)

**⚠️ 안전 수칙:** 이 앱은 로컬 프리뷰에서도 운영과 동일한 실시간 Firestore(`wv-team-tool`)에 연결된다. 아래 절차대로 `saveDB`를 임시로 무력화한 뒤 테스트해서, 테스트용 가짜 일정이 실제 팀 DB에 저장되지 않게 한다. 이 무력화는 **브라우저 런타임에서만** 하는 것이고 `index.html` 소스 코드는 건드리지 않는다.

- [ ] **Step 1: 로컬 서버 기동**

`preview_start` 툴로 `"static"` 설정(이미 `.claude/launch.json`에 있음)을 실행한다.

- [ ] **Step 2: 로그인 우회 + 저장 무력화 (콘솔 스크립트)**

`preview_eval`로 아래를 실행한다 (실제 비밀번호 없이 화면에 접근하고, 이후 어떤 저장도 Firestore에 반영되지 않도록 `saveDB`를 무력화):

```js
(() => {
  if (!DB.members.length) return 'NO_MEMBERS';
  const m = DB.members[0];
  currentUser = { memberId: m.id, username: 'preview-test', name: m.name, rank: m.rank || '' };
  hideLoginOverlay();
  window.__origSaveDB = saveDB;
  saveDB = function() { console.log('[TEST] saveDB() 호출 무시됨 (실제 저장 안 함)'); };
  showPage('weekly');
  return 'READY: ' + m.name;
})()
```

Expected: `"READY: <멤버이름>"` 반환.

- [ ] **Step 3: 그리드/스크롤 구조 확인**

`preview_snapshot`으로 페이지 구조를 확인한다. `주간 출장/휴가 현황` 제목과 표, "↑ 이전 주 더보기" / "↓ 다음 주 더보기" 버튼, "이번 주로 이동" 버튼이 보이는지 확인한다.

- [ ] **Step 4: 일반 클릭 → 팝오버 열림 확인**

`preview_eval`로 첫 번째 데이터 셀을 클릭한다 (실제 마우스 클릭 시뮬레이션):

```js
(() => {
  const td = document.querySelector('#weekly-content td[data-member]');
  td.click();
  return { member: td.dataset.member, date: td.dataset.date, popoverVisible: !document.getElementById('wk-popover').classList.contains('hidden') };
})()
```

Expected: `popoverVisible: true`.

`preview_screenshot`으로 팝오버가 해당 셀 근처에 둥근 모서리로 떠 있는지 육안 확인.

- [ ] **Step 5: 팝오버로 추가 (테스트 데이터, saveDB 무력화된 상태이므로 안전)**

```js
(() => {
  document.getElementById('wk-pop-input').value = '테스트출장(검증용)';
  Pages.weekly.chipClick('출장');
  Pages.weekly.saveSingle();
  const td = document.querySelector('#weekly-content td[data-member]');
  return td.innerHTML.includes('테스트출장');
})()
```

Expected: `true` (셀 안에 방금 추가한 항목이 즉시 보임 — `saveDB`는 무력화되어 있어 Firestore에는 반영 안 됨).

- [ ] **Step 6: 같은 셀 재클릭 → 기존 항목 + 삭제(✕) 확인**

```js
(() => {
  const td = document.querySelector('#weekly-content td[data-member]');
  td.click();
  const pop = document.getElementById('wk-popover');
  return pop.innerHTML.includes('테스트출장') && pop.innerHTML.includes('✕');
})()
```

Expected: `true`.

- [ ] **Step 7: Ctrl+클릭 다중선택 시뮬레이션 (실제 클릭은 modifier 키를 못 주므로 `cellClick` 직접 호출)**

```js
(() => {
  Pages.weekly.closePopover();
  const cells = document.querySelectorAll('#weekly-content td[data-member]');
  const c1 = cells[0], c2 = cells[1];
  Pages.weekly.cellClick({ ctrlKey: true, shiftKey: false, currentTarget: c1 }, parseInt(c1.dataset.member), c1.dataset.date);
  Pages.weekly.cellClick({ ctrlKey: true, shiftKey: false, currentTarget: c2 }, parseInt(c2.dataset.member), c2.dataset.date);
  const bar = document.getElementById('wk-selection-bar');
  return { barVisible: !bar.classList.contains('hidden'), barText: bar.textContent, selectedCount: document.querySelectorAll('.wk-selected').length };
})()
```

Expected: `barVisible: true`, `barText`에 "2칸 선택됨" 포함, `selectedCount: 2`.

- [ ] **Step 8: Ctrl+Shift 범위선택 시뮬레이션 (같은 행 내 3칸)**

```js
(() => {
  const rows = document.querySelectorAll('#weekly-content tbody tr');
  // 첫 번째 팀원 데이터 행 찾기 (구분 행 다음)
  const dataRow = [...rows].find(r => r.querySelector('td[data-member]'));
  const cellsInRow = dataRow.querySelectorAll('td[data-member]');
  const first = cellsInRow[0], third = cellsInRow[2];
  Pages.weekly.cellClick({ ctrlKey: false, shiftKey: false, currentTarget: first }, parseInt(first.dataset.member), first.dataset.date);
  Pages.weekly.cellClick({ ctrlKey: true, shiftKey: true, currentTarget: third }, parseInt(third.dataset.member), third.dataset.date);
  return document.querySelectorAll('.wk-selected').length;
})()
```

Expected: `3` (같은 행의 연속 3칸이 선택됨). 참고: 일반 클릭(첫 줄)은 팝오버를 열지만 이 검증에서는 선택 개수만 확인하면 되므로 무시.

- [ ] **Step 9: Bulk 저장 확인**

```js
(() => {
  Pages.weekly.openBulkPopover();
  document.getElementById('wk-pop-input').value = '테스트일괄';
  Pages.weekly.saveBulk();
  return document.querySelectorAll('.wk-event').length > 0;
})()
```

Expected: `true`.

- [ ] **Step 10: 정리 — 페이지 새로고침으로 모든 테스트 변경 폐기**

```js
location.reload()
```

이 새로고침으로 `DB.events`에 추가됐던 "테스트출장(검증용)", "테스트일괄" 등은 (Firestore에 저장된 적이 없으므로) 완전히 사라진다. 새로고침 후 `preview_eval`로 아래를 실행해 실제로 사라졌는지 확인:

```js
DB.events.some(e => e.title && e.title.includes('테스트'))
```

Expected: `false`.

- [ ] **Step 11: 서버 종료**

`preview_stop` 툴로 서버를 종료한다.

- [ ] **Step 12: 최종 확인 커밋 없음 안내**

이 태스크는 코드 변경이 없으므로 커밋할 것이 없다. Task 1~8의 커밋들이 이미 `beta` 브랜치에 쌓여 있는지 최종 확인한다:

Run: `git log --oneline master..beta`
Expected: Task 1~8에서 만든 커밋들이 순서대로 모두 보임.

---

## Self-Review 결과

- **스펙 커버리지:** `docs/superpowers/specs/2026-07-07-weekly-grid-redesign-design.md`의 4개 섹션(①연속 스크롤, ②셀 클릭 인터랙션 2-1~2-6, ③데이터/삭제 설계 결정, ④범위 밖) 모두 Task 1~9에 반영됨. 팀 이벤트 삭제 버튼 미표시(③)는 Task 6의 `openCellPopover`에서 `isTeamEvent(e) ? '' : ...`로 구현. 모바일/캘린더/기존 모달 불변(④)은 각 태스크에서 모바일 코드·기존 모달 코드를 건드리지 않도록 명시.
- **플레이스홀더 스캔:** 없음 — 모든 스텝에 완전한 코드와 실행 가능한 명령이 포함됨.
- **타입/이름 일관성:** `cellClick`, `openCellPopover`, `openBulkPopover`, `saveSingle`, `saveBulk`, `chipClick`, `deleteExisting`, `closePopover`, `extendPast`, `extendFuture`, `scrollToThisWeek` — Task 5~8에서 정의한 이름과 Task 8의 `return {...}` 노출 목록, HTML의 `onclick="Pages.weekly.xxx(...)"` 호출부가 모두 동일한 이름을 사용하는지 재확인 완료.
- **실 데이터 보호:** Task 9에서 `saveDB` 런타임 무력화 절차를 필수 단계로 명시 — beta가 운영과 DB를 공유하는 설계([[beta-environment]])를 고려한 안전장치.

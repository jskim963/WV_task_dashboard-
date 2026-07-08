# 취합리스트 회신자별 첨부파일 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 취합리스트 상세 화면의 체크리스트에서, 각 회신자(대상 팀원) 줄에 직접 파일을 업로드할 수 있게 하고, 업로드 시 자동으로 제출 체크가 되도록 한다.

**Architecture:** 기존 `Pages.collections` 모듈(index.html)에 함수를 추가/수정한다. 새 파일은 없다. `submitted[memberId]`에 `files` 배열을 추가하고, Firebase Storage에 `collections/{id}/{memberId}/...` 경로로 업로드한다.

**Tech Stack:** Vanilla JS, Firebase Firestore, Firebase Storage — 기존 취합리스트 기능(index.html, `Pages.collections`)의 연장선.

**참고 스펙:** [docs/superpowers/specs/2026-07-08-collection-list-respondent-attachments-design.md](../specs/2026-07-08-collection-list-respondent-attachments-design.md)

---

## 테스트/검증 방식

이 프로젝트는 테스트 프레임워크가 없다. 원본 취합리스트 계획과 동일하게, 각 태스크는 다음으로 검증한다:
1. `node -e "...new Function(s)..."` 로 인라인 스크립트 문법 오류 확인
2. `grep`으로 새 함수/HTML 요소가 실제로 존재하는지 확인
3. Node로 실행 가능한 순수 로직(예: `toggleCheck`가 `files`를 보존하는지)은 별도 시뮬레이션으로 검증

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

---

## 파일 구조

기존 `index.html` 한 파일만 수정한다.

- `Pages.collections` 모듈 내 `toggleCheck` 함수 수정 (`files` 보존)
- `Pages.collections` 모듈에 `uploadReplyFile`, `removeReplyFile` 함수 추가
- `Pages.collections` 모듈의 `return` 문 갱신
- `Pages.collections.renderDetail()`의 체크리스트 렌더링 부분 수정 (업로드 버튼 + 파일 목록 추가)
- 전역 shim 함수 2개 추가

---

### Task 1: 데이터 로직 — toggleCheck 보존 수정 + 업로드/삭제 함수

**Files:**
- Modify: `index.html` (`Pages.collections` 모듈 내 `toggleCheck`, `return` 문, drafts/library shim 근처 전역 함수 영역)

- [ ] **Step 1: `toggleCheck`가 기존 `files`를 보존하도록 수정**

다음을 찾는다:
```js
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
```
다음으로 교체:
```js
  function toggleCheck(id, memberId) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    if (!item.submitted) item.submitted = {};
    const key = String(memberId);
    const wasChecked = !!item.submitted[key]?.checked;
    item.submitted[key] = { ...item.submitted[key], checked: !wasChecked, at: !wasChecked ? Date.now() : null };
    item.status = isFullyChecked(item) ? '완료' : '진행중';
    db.collection('collections').doc(String(id)).update({ submitted: item.submitted, status: item.status }).catch(e => console.error(e));
    renderDetail(id);
    render();
  }

  async function uploadReplyFile(id, memberId, inputEl) {
    const files = inputEl && inputEl.files ? Array.from(inputEl.files) : [];
    if (!files.length) return;
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    if (!item.submitted) item.submitted = {};
    const key = String(memberId);
    const existing = item.submitted[key] || { checked: false, at: null, files: [] };
    const uploaded = [...(existing.files || [])];
    for (const file of files) {
      const path = `collections/${id}/${memberId}/${Date.now()}_${file.name}`;
      const ref = firebase.storage().ref(path);
      try {
        await ref.put(file);
        const url = await ref.getDownloadURL();
        uploaded.push({ name: file.name, path, url });
      } catch (e) { console.error('파일 업로드 오류:', e); }
    }
    item.submitted[key] = { ...existing, checked: true, at: Date.now(), files: uploaded };
    item.status = isFullyChecked(item) ? '완료' : '진행중';
    db.collection('collections').doc(String(id)).update({ submitted: item.submitted, status: item.status }).catch(e => console.error(e));
    inputEl.value = '';
    renderDetail(id);
    render();
  }

  function removeReplyFile(id, memberId, fileIndex) {
    const item = (DB.collections || []).find(c => c.id === id);
    if (!item) return;
    const key = String(memberId);
    const entry = item.submitted[key];
    if (!entry || !entry.files) return;
    entry.files = entry.files.filter((_, i) => i !== fileIndex);
    db.collection('collections').doc(String(id)).update({ submitted: item.submitted }).catch(e => console.error(e));
    renderDetail(id);
  }
```

**Step 2: `return` 문에 새 함수 추가**

다음을 찾는다:
```js
  return { switchTab, render, isFullyChecked, openDetail, closeDetail, toggleCheck, deleteItem, openAddModal, editModal, save, removeAttachment, notifyUnsubmitted };
```
다음으로 교체:
```js
  return { switchTab, render, isFullyChecked, openDetail, closeDetail, toggleCheck, deleteItem, openAddModal, editModal, save, removeAttachment, notifyUnsubmitted, uploadReplyFile, removeReplyFile };
```

**Step 3: 전역 shim 함수 추가**

다음을 찾는다:
```js
function notifyUnsubmitted(id) { Pages.collections.notifyUnsubmitted(id); }
```
다음으로 교체:
```js
function notifyUnsubmitted(id) { Pages.collections.notifyUnsubmitted(id); }
function uploadReplyFile(id, memberId, inputEl) { Pages.collections.uploadReplyFile(id, memberId, inputEl); }
function removeReplyFile(id, memberId, fileIndex) { Pages.collections.removeReplyFile(id, memberId, fileIndex); }
```

> 실제로는 체크리스트 HTML(Task 2)에서 `Pages.collections.uploadReplyFile(...)`/`Pages.collections.removeReplyFile(...)`를 직접 호출할 예정이라 이 전역 shim은 당장 쓰이지 않을 수 있다. 그래도 이 파일의 다른 모든 `Pages.collections.*` 함수가 동일한 이름의 전역 shim을 갖고 있는 기존 컨벤션([index.html:8280-8283](../../../index.html) 근처 `editCollection`/`deleteCollection`/`closeCollectionDetail`/`notifyUnsubmitted` 참고)을 따르기 위해 추가한다.

**Step 4: 문법 검증**

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

**Step 5: 구조 검증**

Run: `grep -n 'function uploadReplyFile\|function removeReplyFile\|function toggleCheck' index.html`
Expected: 4줄 이상 출력 (모듈 내부 정의 2개 + 전역 shim 2개 + toggleCheck 1개, 총 5개 매치)

**Step 6: 로직 검증 — `toggleCheck`가 기존 `files`를 보존하는지 시뮬레이션**

```bash
node -e "
function isFullyChecked(item) {
  return item.targetIds.length > 0 && item.targetIds.every(mid => item.submitted[String(mid)]?.checked);
}
const item = { targetIds: [1], submitted: { '1': { checked: false, at: null, files: [{name:'a.pdf'}] } } };
const key = '1';
const wasChecked = !!item.submitted[key]?.checked;
item.submitted[key] = { ...item.submitted[key], checked: !wasChecked, at: !wasChecked ? Date.now() : null };
item.status = isFullyChecked(item) ? '완료' : '진행중';
console.log(item.submitted['1'].checked);            // true
console.log(item.submitted['1'].files.length);       // 1 (보존됨)
console.log(item.status);                              // 완료
"
```
Expected: `true`, `1`, `완료` (세 줄)

**Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 회신자별 첨부파일 업로드/삭제 로직 추가

submitted[memberId]에 files 배열을 추가하고, toggleCheck가 기존
files를 보존하도록 수정. uploadReplyFile(업로드 시 자동 체크),
removeReplyFile(체크 상태는 유지) 함수 추가.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: 상세 화면 체크리스트 UI — 업로드 버튼 + 파일 목록 렌더링

**Files:**
- Modify: `index.html` (`Pages.collections.renderDetail` 함수)

- [ ] **Step 1: 체크리스트 렌더링 부분을 업로드 UI 포함하도록 교체**

다음을 찾는다:
```js
    document.getElementById('cd-checklist').innerHTML = item.targetIds.map(mid => {
      const m = DB.members.find(mm => mm.id === mid);
      const s = item.submitted[String(mid)] || { checked: false, at: null };
      return `<label class="flex items-center justify-between px-3 py-2 border-b border-outline-variant last:border-b-0 cursor-pointer">
        <span class="flex items-center gap-2 text-sm"><input type="checkbox" ${s.checked ? 'checked' : ''} onchange="Pages.collections.toggleCheck(${item.id},${mid})" class="w-4 h-4 accent-primary">${m ? m.name : mid}</span>
        <span class="text-xs text-secondary">${s.checked && s.at ? new Date(s.at).toLocaleString('ko-KR') : '미제출'}</span>
      </label>`;
    }).join('');
```
다음으로 교체:
```js
    document.getElementById('cd-checklist').innerHTML = item.targetIds.map(mid => {
      const m = DB.members.find(mm => mm.id === mid);
      const s = item.submitted[String(mid)] || { checked: false, at: null, files: [] };
      const files = s.files || [];
      const fileListHtml = files.map((f, i) => `
        <div class="flex items-center justify-between text-xs bg-surface-container rounded px-2 py-1 mt-1 ml-6">
          <a href="${f.url}" target="_blank" class="text-primary hover:underline truncate">${escapeHtml(f.name)}</a>
          <button type="button" onclick="Pages.collections.removeReplyFile(${item.id},${mid},${i})" class="text-secondary hover:text-red-500"><span class="material-symbols-outlined text-sm">close</span></button>
        </div>`).join('');
      return `<div class="px-3 py-2 border-b border-outline-variant last:border-b-0">
        <div class="flex items-center justify-between">
          <label class="flex items-center gap-2 text-sm cursor-pointer">
            <input type="checkbox" ${s.checked ? 'checked' : ''} onchange="Pages.collections.toggleCheck(${item.id},${mid})" class="w-4 h-4 accent-primary">${m ? m.name : mid}
          </label>
          <div class="flex items-center gap-2">
            <span class="text-xs text-secondary">${s.checked && s.at ? new Date(s.at).toLocaleString('ko-KR') : '미제출'}</span>
            <label class="material-symbols-outlined text-sm text-secondary hover:text-primary cursor-pointer" title="파일 첨부">
              attach_file
              <input type="file" multiple class="hidden" onchange="Pages.collections.uploadReplyFile(${item.id},${mid},this)">
            </label>
          </div>
        </div>
        ${fileListHtml}
      </div>`;
    }).join('');
```

> `escapeHtml`는 이미 `Pages.collections` 모듈 상단에 정의되어 있다 (최종 통합 리뷰에서 첨부파일명 XSS 방지를 위해 추가됨, [index.html:7814](../../../index.html) 근처). 새로 정의할 필요 없다.

**Step 2: 문법 검증**

Run: (Task 1과 동일한 node 검증 명령)
Expected: `SYNTAX OK`

**Step 3: 구조 검증**

Run: `grep -n "attach_file" index.html`
Expected: 최소 2줄 (기존 `cd-attachments`의 등록자 첨부 아이콘 1개 + 이번에 추가한 회신자별 업로드 아이콘 1개, 총 2개 이상)

Run: `grep -n "Pages.collections.uploadReplyFile\|Pages.collections.removeReplyFile" index.html`
Expected: 각각 2줄 이상 (Task 1의 함수 정의 + Task 2의 HTML 호출)

**Step 4: 로직 검증 — 체크리스트 HTML 생성 자체를 Node로 시뮬레이션 (문자열 생성 로직만)**

```bash
node -e "
function escapeHtml(s) { return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
const item = { id: 1, submitted: { '5': { checked: true, at: Date.now(), files: [{name:'<script>.pdf', url:'https://x/y'}] } } };
const mid = 5;
const s = item.submitted[String(mid)];
const files = s.files || [];
const fileListHtml = files.map((f,i) => \`<a>\${escapeHtml(f.name)}</a>\`).join('');
console.log(fileListHtml.includes('&lt;script&gt;'));  // true — 이스케이프됨
console.log(fileListHtml.includes('<script>'));         // false — 원문 그대로 안 들어감
"
```
Expected: `true` 그다음 `false`

**Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
feat: 취합리스트 상세 체크리스트에 회신자별 파일 업로드 UI 추가

각 회신자 줄에 파일 첨부 아이콘 추가. 업로드된 파일은 줄 아래 작게
나열되고 파일명은 escapeHtml로 이스케이프해 XSS를 방지한다.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## 스펙 커버리지 체크 (자체 검토 결과)

- `submitted[memberId].files` 데이터 모델 → Task 1
- 업로드 시 자동 체크, 기존 `files` 보존(toggleCheck 수정) → Task 1
- 파일 목록 표시/다운로드/개별 삭제(체크 상태 유지) → Task 1(로직) + Task 2(UI)
- 로그인한 모든 팀원이 업로드 가능 → 별도 권한 체크를 추가하지 않음으로써 구현됨 (기존 "누구나" 정책과 동일)
- 기존 등록자용 첨부파일(`item.attachments`, `cd-attachments`) 유지 → 이번 계획에서 건드리지 않음 (변경 없음 자체가 요구사항)
- XSS 방지(escapeHtml) → Task 2에서 기존 모듈 상단의 `escapeHtml` 재사용

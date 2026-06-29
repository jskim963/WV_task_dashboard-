# 프로젝트 관리 기능 설계

**날짜:** 2026-06-29  
**상태:** 승인됨

---

## 1. 개요

업무현황의 태스크들을 프로젝트 단위로 묶어 진척률·마일스톤·댓글을 관리하는 기능.  
자료실 파일 업로드 버그(0% 정체)도 동시에 수정.

---

## 2. 데이터 모델

### 신규 Firestore 컬렉션 `projects`

```js
{
  id: number,            // Date.now() timestamp
  title: string,         // 프로젝트명
  summary: string,       // 개요 (한 줄)
  description: string,   // 내용 (상세)
  startDate: string,     // 'YYYY-MM-DD'
  endDate: string,       // 목표일정 종료일
  assigneeIds: number[], // 담당자 멤버 ID 배열 (체크박스 멀티선택)
  milestones: [
    { id: number, title: string, date: string, done: boolean }
  ],
  comments: [
    { id: number, text: string, authorId: number, authorName: string, createdAt: number }
  ],
  createdBy: string,     // currentUser.username
  createdAt: number,
}
```

### 기존 `tasks` 필드 추가

```js
projectId: number | null   // 연결된 프로젝트 ID (없으면 null)
```

`DB` 객체에 `projects: []` 추가, `FS_COLLS`에 `'projects'` 추가.

---

## 3. 진척률 계산

프로젝트에 속한 태스크 상태별 가중치 평균:

| 상태 | 가중치 |
|------|--------|
| 완료 | 100 |
| 진행중 | 50 |
| 지연 | 25 |
| 그 외 | 0 |

```js
function calcProgress(projectId) {
  const tasks = DB.tasks.filter(t => t.projectId === projectId);
  if (!tasks.length) return 0;
  const weights = { '완료': 100, '진행중': 50, '지연': 25 };
  const sum = tasks.reduce((acc, t) => acc + (weights[t.status] || 0), 0);
  return Math.round(sum / tasks.length);
}
```

---

## 4. UI 구조

### 4-1. 프로젝트 탭 (page-projects)

**상세 닫힘 상태:**
- 3열 카드 그리드
- 각 카드: 프로젝트명, 원형 진척률 게이지(SVG), 기간, 담당자 아바타, 소속 태스크 수
- 우상단 "+ 프로젝트 추가" 버튼 (전체 멤버 생성 가능)

**카드 클릭 → 상세 열림:**
- 카드 그리드 → 좌측 1/3 세로 목록으로 전환 (미니 카드형)
- 우측 2/3 상세 패널 슬라이드인
- X 버튼으로 패널 닫기

### 4-2. 상세 패널 구성 (위→아래)

1. **헤더** — 프로젝트명, 수정/삭제 버튼 (생성자 또는 `jungsoo.kim`만 표시)
2. **메타** — 개요, 설명, 담당자 아바타 목록, 기간
3. **진척률** — 가로 진척률 바 + % 수치 + 태스크 상태 분포 (완료/진행중/지연/기타 건수)
4. **간트 바** — 전체 기간을 수평 바로 표시, 마일스톤을 다이아몬드(◆) 마커로 오버레이, 완료된 마일스톤은 채움 색상
5. **마일스톤 목록** — 추가 입력창, 완료 체크박스, 삭제 버튼
6. **소속 태스크 목록** — 상태 뱃지, 태스크명, 담당자, 마감일, 클릭 시 `showTaskDetail()` 호출
7. **댓글** — 입력창 + 댓글 목록 (삭제는 작성자·관리자만)

### 4-3. 프로젝트 추가/수정 모달 (modal-add-project)

필드:
- 프로젝트명 (필수)
- 개요 (한 줄 텍스트)
- 내용 (textarea)
- 시작일 / 목표일 (date input)
- 담당자: 체크박스 목록 드롭박스 (기존 assignee dropdown 패턴 재사용)
- 마일스톤: 모달 내 추가 가능 (제목 + 날짜)

---

## 5. 기존 대시보드 연동

### 5-1. 업무 추가/수정 모달

- "프로젝트" 드롭박스 추가 (없음 + 프로젝트 목록)
- 저장 시 `task.projectId` 기록

### 5-2. 업무 목록 행 표시

프로젝트에 속한 태스크는 업무명 위에 작은 뱃지로 프로젝트명 표시:

```
No.  [프로젝트명 뱃지]
     업무명
```

### 5-3. 대시보드 상단 요약

기존 4개 카드 옆에 "진행중 프로젝트" 카드 추가:
- 진행중 프로젝트 수 (완료되지 않은 것)
- 클릭 시 프로젝트 탭으로 이동

---

## 6. 네비게이션

사이드바에 `프로젝트` 항목 추가:
```html
<div class="nav-item" onclick="showPage('projects')" id="nav-projects">
  <span class="material-symbols-outlined">folder_special</span> 프로젝트
</div>
```

---

## 7. 자료실 업로드 버그 수정 (동시 진행)

**원인:** Firebase Storage 보안 규칙이 미인증 쓰기를 차단.  
앱이 커스텀 인증(Firebase Auth 미사용)을 쓰므로 Storage 규칙을 공개 쓰기 허용으로 설정해야 함.

**Firebase Console에서 설정할 규칙:**
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if true;
    }
  }
}
```

**코드 측 보완:** `ref.put(file)` 에러 핸들러에서 `err.code`와 `err.message`를 콘솔에 출력하도록 강화, 에러 토스트 표시 유지.

---

## 8. 권한 정리

| 작업 | 허용 대상 |
|------|-----------|
| 프로젝트 생성 | 전체 멤버 |
| 프로젝트 수정/삭제 | 생성자 또는 관리자(jungsoo.kim) |
| 마일스톤 추가/완료/삭제 | 생성자 또는 관리자 |
| 댓글 작성 | 전체 멤버 |
| 댓글 삭제 | 작성자 또는 관리자 |
| 태스크-프로젝트 연결 | 태스크 담당자 또는 관리자 |

---

## 9. Pages 네임스페이스

`Pages.projects = (() => { ... })()` IIFE로 격리.  
전역 shims: `openAddProjectModal()`, `saveProject()`, `deleteProject()`, `toggleProjectDetail()`, `addMilestone()`, `toggleMilestone()`, `deleteMilestone()`, `addProjectComment()`, `deleteProjectComment()`.

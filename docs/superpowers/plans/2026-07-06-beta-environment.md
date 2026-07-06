# Beta 환경 구축 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 운영(`master`)과 Firestore DB를 실시간 공유하면서 UI/레이아웃/기능을 독립적으로 실험할 수 있는 `beta` 브랜치 + Vercel 프리뷰 배포 환경을 구축한다.

**Architecture:** `master`에서 `beta` 브랜치를 분기하고, `beta` 전용 `index.html`에 눈에 띄는 "BETA" 배지를 추가한 뒤 `origin/beta`로 push한다. Vercel의 기본 Git 연동이 비-프로덕션 브랜치를 자동으로 Preview Deployment로 배포하므로 별도 Vercel 설정은 필요 없다. Firebase config는 그대로 두어 운영과 동일한 Firestore(`wv-team-tool`)를 실시간 공유한다.

**Tech Stack:** 기존 스택 그대로 (Firebase Firestore, Tailwind CDN, Vanilla JS, Vercel). 이 프로젝트에는 자동화 테스트 프레임워크가 없으므로, 각 태스크의 "검증" 단계는 `grep`/브라우저 스크린샷/`curl`을 이용한 수동 검증으로 대체한다.

---

### Task 1: `beta` 브랜치 생성

**Files:** 없음 (git 작업만)

- [ ] **Step 1: 현재 브랜치와 작업 트리 상태 확인**

Run: `git status --short && git branch --show-current`
Expected: 출력에 `master`가 보이고, 커밋 대기 중인 변경사항이 이번 작업과 무관함을 확인 (untracked 파일들은 무시해도 됨).

- [ ] **Step 2: `master`에서 `beta` 브랜치 생성 후 전환**

Run: `git checkout -b beta`
Expected: `Switched to a new branch 'beta'`

- [ ] **Step 3: 분기 확인**

Run: `git branch -a`
Expected: `* beta`와 `master`가 함께 표시됨 (아직 `origin/beta`는 없음 — 정상).

---

### Task 2: `beta` 전용 BETA 배지 추가

**Files:**
- Modify: `index.html:283`

- [ ] **Step 1: `<body>` 태그 직후에 배지 마크업 삽입**

`index.html`의 283번째 줄은 다음과 같다:

```html
<body class="flex h-screen overflow-hidden">
```

이 줄 바로 다음에 배지 `<div>`를 추가한다 (로그인 오버레이의 `z-[999]`보다 높은 `z-index`를 줘서 로그인 화면 위에도 항상 보이게 함):

```html
<body class="flex h-screen overflow-hidden">
<div id="beta-badge" style="position:fixed;top:0;right:0;z-index:9999;background:#f97316;color:#fff;font-weight:700;font-size:12px;letter-spacing:0.05em;padding:4px 14px;border-bottom-left-radius:8px;pointer-events:none;font-family:sans-serif;">BETA</div>
```

- [ ] **Step 2: 삽입 확인**

Run: `grep -n "beta-badge" index.html`
Expected: `284:<div id="beta-badge" ...` 한 줄이 출력됨.

---

### Task 3: 로컬 프리뷰로 배지 시각 검증

**Files:**
- Create: `.claude/launch.json`

- [ ] **Step 1: 정적 서버 launch 설정 생성**

`.claude/launch.json` 파일이 아직 없으므로 새로 만든다:

```json
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "static",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["serve", "-l", "3000", "."],
      "port": 3000
    }
  ]
}
```

- [ ] **Step 2: 서버 시작**

`preview_start` 툴로 `"static"` 설정을 실행한다.
Expected: 서버가 포트 3000에서 기동됨.

- [ ] **Step 3: 스크린샷으로 배지 확인**

`preview_screenshot` 툴로 화면을 캡처한다.
Expected: 화면 우측 상단에 주황색 "BETA" 배지가 보임 (로그인 화면이 뜨든 메인 화면이 뜨든 항상 보여야 함).

- [ ] **Step 4: 프리뷰 서버 종료**

`preview_stop` 툴로 방금 시작한 서버를 종료한다.

---

### Task 4: 배지 변경사항 커밋

**Files:**
- `index.html` (Task 2에서 수정)
- `.claude/launch.json` (Task 3에서 생성)

- [ ] **Step 1: 변경사항 스테이징 및 커밋**

```bash
git add index.html .claude/launch.json
git commit -m "feat: BETA 배지 추가 및 로컬 프리뷰 설정 (beta 전용)"
```

Expected: `beta` 브랜치에 새 커밋 생성. (`master`에는 영향 없음 — `git log master..beta`로 확인 가능)

- [ ] **Step 2: 커밋이 beta 전용임을 확인**

Run: `git log master..beta --oneline`
Expected: 방금 만든 커밋 1개만 출력됨.

---

### Task 5: `beta` 브랜치를 원격에 push

**Files:** 없음 (git 작업만)

- [ ] **Step 1: origin에 `beta` 브랜치 push (upstream 설정 포함)**

```bash
git push -u origin beta
```

Expected: `remote: Create a pull request for 'beta' on GitHub by visiting: ...` 메시지와 함께 push 성공. (`master`는 여전히 로컬에만 최신 상태로 남고, 운영 브랜치인 `main`에는 아무 영향 없음.)

- [ ] **Step 2: 원격 추적 확인**

Run: `git branch -vv`
Expected: `beta` 줄에 `[origin/beta]`가 표시됨.

---

### Task 6: Vercel 프리뷰 배포 확인

**Files:** 없음 (배포 확인만)

- [ ] **Step 1: 예상 프리뷰 URL로 확인 시도**

Vercel Git 연동은 브랜치 push 후 보통 1~2분 내에 프리뷰를 빌드한다. 프로젝트 슬러그(`wv-task-dashboard`)와 팀 슬러그(`jskim963s-projects`) 기준으로 예상되는 URL을 확인한다:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://wv-task-dashboard-git-beta-jskim963s-projects.vercel.app
```

Expected: `200`. (Vercel의 실제 프리뷰 URL 네이밍 규칙은 프로젝트 설정에 따라 다를 수 있음.)

- [ ] **Step 2: 예상 URL이 실패할 경우 폴백**

`200`이 아니거나 아직 빌드 중이면, 사용자에게 Vercel 대시보드(`https://vercel.com/jskim963s-projects/wv-task-dashboard`)의 Deployments 탭에서 `beta` 브랜치 배포 항목을 열어 실제 프리뷰 URL을 확인해달라고 안내한다. 이 단계는 실패해도 인프라 구축 자체의 실패가 아니다 (Vercel 대시보드 UI는 이 세션에서 직접 확인 불가).

- [ ] **Step 3: 접속 후 실데이터 공유 확인**

프리뷰 URL 접속 후 로그인하여 기존 운영 계정으로 로그인이 되는지, 업무/센터 데이터가 운영과 동일하게 보이는지 확인한다 (동일 Firestore 프로젝트를 보고 있다는 증거). 화면 우측 상단에 BETA 배지가 보이는지 함께 확인한다.

---

## Self-Review 결과

- **스펙 커버리지:** 스펙의 4개 섹션(브랜치/배포 구조, BETA 배지, 향후 워크플로, 리스크) 중 "향후 워크플로"(cherry-pick 절차)는 이번 인프라 구축 태스크가 아니라 *사용될* 절차이므로 별도 태스크 불필요 — 스펙에 이미 문서화됨. 나머지는 Task 1~6에서 모두 커버.
- **플레이스홀더 스캔:** 없음. 모든 스텝에 실행 가능한 명령/코드 포함.
- **범위 밖 항목 재확인:** ONBOARDING.md/CHANGELOG.md 갱신은 이번 스펙의 "범위 밖"에 해당하므로 포함하지 않음.

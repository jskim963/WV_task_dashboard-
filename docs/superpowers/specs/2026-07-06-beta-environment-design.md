# Beta 환경 구축 설계

> 작성일: 2026-07-06

## 목적

현재 운영 중인 W&V팀 내부관리 Tool(`index.html`, `master` → GitHub `main` → Vercel 운영 배포)과 별도로,
UI·레이아웃·신규 기능을 자유롭게 실험할 수 있는 **beta 환경**을 만든다.

- 핵심 요구사항: **DB(Firestore)는 운영과 동일하게 실시간 공유**한다. beta에서 별도 데이터 격리는 하지 않는다.
- 이번 작업 범위는 **beta 인프라 구축까지**. 구체적인 신규 UI/기능은 이후 별도로 브레인스토밍한다.

## 1. 브랜치 & 배포 구조

```
master ──●──●──●──●──●────────▶ (운영, GitHub main ← push, wv-task-dashboard.vercel.app)
           \
            ●beta 분기 (2026-07-06 시점)
             \●──●──●───────────▶ (beta, GitHub beta ← push, Vercel Preview URL)
```

- `master`에서 `beta` 브랜치를 분기한다 (분기 시점: 현재 HEAD).
- 로컬 `beta` 브랜치를 원격 `beta` 브랜치로 push한다 (`git push -u origin beta`). `master→main`처럼 이름을 다르게 가져갈 이유가 없으므로 로컬/원격 모두 `beta`로 통일.
- Vercel은 GitHub 연동 시 기본적으로 모든 브랜치를 Preview Deployment로 자동 배포한다. 이 저장소에는 `vercel.json`에 브랜치 제한 설정이 없으므로, 추가 설정 없이 push만 하면 프리뷰 URL이 자동 생성된다.
  - 정확한 프리뷰 URL은 첫 push 후 Vercel 대시보드(또는 GitHub PR 체크)에서 확인한다.
- `index.html`은 `master`와 동일한 내용으로 시작한다. Firebase config(`firebaseConfig`, 프로젝트 `wv-team-tool`)는 그대로 두므로, 브랜치가 달라도 자동으로 **같은 Firestore를 실시간 공유**한다. 이 부분은 코드 변경이 필요 없다.

## 2. BETA 배지

- `beta` 브랜치의 `index.html`에만 고정 배지를 추가한다. `master`에는 반영하지 않는다.
- 로그인 화면을 포함해 항상 보이도록 최상단 레이어에 배치한다 (`position:fixed`, 클릭을 막지 않도록 `pointer-events:none`).
- 눈에 띄는 색상(주황/빨강 계열), 텍스트 "BETA".
- 목적: 실제 운영 데이터를 그대로 보여주는 화면이므로, 접속한 사람이 운영 화면으로 착각하지 않도록 시각적으로 구분.

## 3. 향후 작업 흐름

- 새 UI/레이아웃/기능 실험은 전부 `beta` 브랜치에서 진행 → push → Vercel 프리뷰 자동 갱신.
- **beta 커밋은 기능 단위로 쪼개서 작성한다** (한 커밋 = 한 기능). 나중에 특정 기능만 골라 운영에 반영하기 쉽게 하기 위함.
- 운영에만 있는 버그 수정/기능을 beta에도 반영하고 싶을 때: 그때그때 사용자가 요청 → 필요한 커밋만 `master`에서 `beta`로 반영 (자동 동기화 없음, 상시 병합 없음).
- beta에서 검증된 기능을 운영에 반영하고 싶을 때 (핵심 플로우):
  1. 사용자가 "beta의 OO 기능 master에 반영해줘"라고 요청.
  2. `beta` 브랜치 git 로그에서 해당 기능의 커밋(들)을 식별.
  3. `git cherry-pick <커밋SHA>`로 `master`에 적용 시도.
     - 충돌 없이 적용되면 그대로 커밋 후 `push origin master:main`.
     - 충돌 발생 시(단일 대형 HTML 파일 특성상 가능성 높음): 자동 병합 대신 beta의 diff를 참고해 `master`의 해당 부분을 수동으로 동일하게 재현하여 커밋.
  4. 반영 후 BETA 배지 등 beta 전용 코드가 실수로 함께 딸려오지 않았는지 확인.

## 4. 리스크 (의도된 설계, 참고용)

- beta는 운영과 동일한 실제 Firestore 데이터를 읽고 쓴다. beta에서 테스트 중 데이터를 수정/삭제하면 운영 데이터도 함께 바뀐다.
- 이는 "DB는 같이 연동되어야 한다"는 요구사항에 따른 의도된 설계이며, BETA 배지 표시 외 별도의 데이터 격리 장치(별도 컬렉션, 별도 Firestore 프로젝트 등)는 두지 않는다.

## 범위 밖 (Out of scope)

- 구체적인 신규 UI/레이아웃/기능 설계 — 이번 spec은 인프라(브랜치, 배포, 배지, 워크플로)까지만 다룬다.
- beta ↔ master 자동 동기화, CI/CD 파이프라인 구축.
- 데이터 격리, 별도 Firestore 프로젝트/컬렉션 분리.

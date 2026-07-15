# 우측 유틸리티 패널 설계

> 작성일: 2026-07-15
> 개정: 2026-07-15 — 구현 도중 사용자 요청으로 "클릭형 탭 전환" 대신 "메모/계산기/AI챗봇을 상중하단에 항상 동시 표시 + 섹션 경계 드래그로 높이 조절" 방식으로 변경. 아래 본문의 "탭"/"클릭형 손잡이 탭" 서술 중 열기/닫기 손잡이(`#utility-tab`) 관련 부분은 그대로 유효하며, 메모/계산기/AI챗봇 사이의 전환 방식만 탭 선택에서 항상-표시-스택으로 바뀌었다. 구체적인 마크업/CSS/JS 변경은 `docs/superpowers/plans/2026-07-15-utility-panel.md`의 "Task 4.5"를 참고.

## 배경

현재 AI 챗봇은 우측 하단 플로팅 버튼(`#chatbot-panel`)으로 모든 페이지에서 접근 가능하다. 개인 메모장은 대시보드 카드(`#personal-memo`)로만 존재해 다른 페이지에서는 쓸 수 없다. 계산기는 없다.

이 셋을 하나의 상시 접근 가능한 우측 슬라이드 패널로 통합한다. 대시보드의 개인 메모장 카드는 제거하고(그 자리는 별도 스펙 `2026-07-15-mentions-inbox-design.md`의 "멘션함" 카드로 대체), 메모 기능은 새 패널의 탭으로 이전한다.

## 목표 / 비목표

**목표**
- 모든 페이지에서 접근 가능한 메모 / 계산기 / AI챗봇 통합 패널
- 클릭형 손잡이 탭으로 열고 닫기 (실제 드래그 제스처는 구현하지 않음)
- 사용자가 패널 폭을 직접 조절 가능
- 열림 상태 / 폭 / 마지막 탭을 기기별로 기억

**비목표**
- 실제 마우스 드래그로 패널을 "당겨서 여는" 제스처 (클릭형으로 충분하다고 판단)
- 계산 이력의 기기 간 동기화 (localStorage, 브라우저 로컬에만 저장)
- 메모 내용의 리치 텍스트/포맷팅 (기존과 동일하게 plain textarea)

## 구조

### 마크업 배치

기존 `#chatbot-panel` 및 `#chat-toggle-btn`(플로팅 버튼)을 제거하고 `<body>` 하단(기존 위치)에 다음으로 교체한다:

```html
<div id="utility-tab" onclick="toggleUtilityPanel()">...</div>
<div id="utility-drawer">
  <div id="utility-resize-handle"></div>
  <div class="utility-header">
    <button data-tab="memo">메모</button>
    <button data-tab="calc">계산기</button>
    <button data-tab="chat">AI챗봇</button>
    <button onclick="toggleUtilityPanel()">닫기</button>
  </div>
  <div id="utility-body">
    <div id="utility-view-memo">...</div>   <!-- 기존 #personal-memo textarea 이전 -->
    <div id="utility-view-calc">...</div>   <!-- 신규 -->
    <div id="utility-view-chat">...</div>   <!-- 기존 #chat-window 내용 이전 -->
  </div>
</div>
```

`#notif-panel`과 동일한 `translateX` 슬라이드 트랜지션 패턴을 재사용한다. 세 뷰는 `.hidden` 토글로 전환하며 동시에 하나만 표시한다(기존 `draft-content-panel` 방식과 동일).

기존 챗봇 관련 함수(`sendChat()`, `clearChat()`, `setChatApiKey()`, `toggleChat()`은 제거하고 `toggleUtilityPanel()`로 통합)는 로직 변경 없이 DOM 위치만 `#utility-view-chat` 내부로 옮긴다.

### 탭 전환 / 상태 저장

- `switchUtilityTab(tabName)`: 활성 탭 버튼 스타일 갱신 + 해당 뷰만 표시 + `localStorage.utilityPanelTab` 저장
- `toggleUtilityPanel()`: 열림/닫힘 토글 + `localStorage.utilityPanelOpen` 저장
- 리사이즈: `#utility-resize-handle`에서 `mousedown` → `mousemove`로 폭 계산(280px~560px clamp) → `mouseup`에서 `localStorage.utilityPanelWidth` 저장
- 페이지 로드 시 세 값을 localStorage에서 읽어 초기 상태 복원

### 메모 탭

기존 `#personal-memo` textarea + `scheduleMemoSave()` / `saveMemo()` 로직을 그대로 `#utility-view-memo`로 이전한다. Firestore `memos` 컬렉션 데이터는 변경 없음. 대시보드 "개인 메모장" 카드(Panel 5, `index.html:495-502`)는 제거한다.

### 계산기 탭

서브탭 2개(`일반` / `단위환산`)를 상단에 둔다.

**일반 서브탭**
- 디스플레이(현재 수식 + 계산 결과) + 버튼 그리드: `7 8 9 ( ) ⌫` / `4 5 6 × ÷ C` / `1 2 3 - +` / `0 . =`
- 괄호를 지원해야 하므로 `eval()`을 쓰지 않고, 사칙연산 우선순위 + 괄호를 처리하는 자체 recursive-descent 파서(`evalExpr(str)`)로 계산한다. 파서는 숫자/연산자/괄호 토큰만 허용하고 그 외 문자는 무시하거나 무효 처리한다.
- `=` 입력 시 결과가 유효하면 `{expr, result, ts: Date.now()}`를 이력 배열 앞에 추가하고 `localStorage.calcHistory`에 저장(최대 20개, 초과분은 잘라냄)
- 이력 목록은 디스플레이 아래 접이식 리스트로 표시하고, 항목 클릭 시 해당 `expr`을 디스플레이에 다시 로드
- 0으로 나누기 등 계산 불가 상황은 디스플레이에 "오류"를 표시하고 이력에는 추가하지 않는다

**키보드 연동**
- `document`에 `keydown` 리스너를 등록하되, 다음 조건을 모두 만족할 때만 동작: 유틸리티 패널이 열려 있음 + 활성 탭이 `계산기` + 활성 서브탭이 `일반` + `document.activeElement`가 계산기 외부의 텍스트 입력(메모 textarea, 챗봇 input, 단위환산 input, 다른 모달의 input 등)이 아님
- 매핑: 숫자키(상단 0-9 + Numpad0-9) → 숫자 입력 / `+ - * /`(상단 + Numpad) → 연산자 / `(` `)` → 괄호 / `.` `NumpadDecimal` → 소수점 / `Enter` `NumpadEnter` → `=` / `Backspace` → 마지막 문자 삭제 / `Escape` → 전체 초기화
- 매핑되지 않은 키는 무시(브라우저 기본 동작 방해 안 함, `preventDefault()`는 매핑된 키에만 적용)

**단위환산 서브탭**
- 숫자 입력 1개 + 방향 토글 버튼(평→㎡ / ㎡→평)
- 변환식: `㎡ = 평 × 3.305785`, `평 = ㎡ ÷ 3.305785`
- 입력값 변경 시 실시간 변환(별도 `=` 없음), 결과는 소수 둘째 자리 반올림

### AI챗봇 탭

기존 `#chat-window` 내부 구조(메시지 목록, 퀵칩, 입력창, API 키 설정/초기화 버튼)를 그대로 `#utility-view-chat`으로 이전한다. `sendChat()` 등 기존 함수는 변경 없음.

## 에러 처리

- 계산기 파싱 실패(잘못된 수식) / 0으로 나누기: 디스플레이에 "오류" 표시, throw하지 않고 조용히 처리
- localStorage 접근 실패(프라이빗 브라우징 등): try/catch로 감싸고 실패 시 상태 저장을 건너뜀(기능은 계속 동작, 단 새로고침 시 기억 안 됨)
- 메모 저장 실패: 기존과 동일하게 `console.error`만 남김(사용자 대면 에러 처리 없음, 기존 컨벤션 유지)

## 테스트/검증

- `node -e` 문법 체크 스크립트로 구문 오류 확인 (프로젝트 표준 검증 방법)
- 계산기 파서는 대표 케이스(사칙연산 우선순위, 중첩 괄호, 0으로 나누기, 잘못된 입력)를 콘솔에서 수동 확인
- 이 프로젝트 컨벤션상 브라우저 프리뷰 검증은 기본 생략, 요청 시에만 수행

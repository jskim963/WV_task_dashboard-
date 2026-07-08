# 취합리스트 — 회신자별 첨부파일 추가 설계

**날짜:** 2026-07-08
**상태:** 승인됨
**관련 스펙:** [2026-07-08-collection-list-design.md](2026-07-08-collection-list-design.md) (원본 설계, 이미 구현 완료)

---

## 1. 배경

원본 스펙 작성 시 "취합건 별로 첨부파일 업로드 기능 활성화(담당자가 일일이 메일로 받지 않게끔)"라는 요구사항을, 취합건 등록자가 참고자료(양식 등)를 올리는 기능으로만 해석해 구현했다. 실제로는 **각 회신자(대상 팀원)가 제출할 때 자신의 회신 파일을 올려서, 담당자가 이메일로 하나씩 받을 필요가 없게** 하는 것이 목적이었다.

논의 끝에 다음으로 확정:
- 기존 등록/수정 모달의 첨부파일(작성자용 양식/참고자료)은 **그대로 유지** — 제거하지 않는다.
- 상세 화면 체크리스트에 **회신자별 첨부**를 **신규 추가**한다. 즉 "제거 후 교체"가 아니라 "추가"다.

---

## 2. 데이터 모델 변경

`collections` 문서의 `submitted[memberId]`에 `files` 배열을 추가한다:

```js
submitted: {
  "3": { checked: true, at: 1720000000000, files: [
    { name: "회신서.pdf", path: "collections/172.../3/172..._회신서.pdf", url: "https://..." }
  ] }
}
```

- 기존 `item.attachments` (등록자용 참고자료)는 필드명·형식 변경 없이 그대로 유지.
- Storage 경로: `collections/{collectionId}/{memberId}/{timestamp}_{fileName}` — 회신자별로 폴더가 분리되어 나중에 누가 뭘 올렸는지 스토리지 상에서도 구분 가능.

---

## 3. 동작

### 체크리스트 각 줄 (상세 화면)

- 기존: 체크박스 + 이름 + 제출시각/미제출 텍스트
- 추가: 파일 업로드 버튼(작은 아이콘) + 업로드된 파일 목록(줄 아래, 작게)
  - 파일 업로드 시 → 해당 팀원 `submitted[memberId]`에 `files` 추가 + **자동으로 `checked: true`, `at: Date.now()`** 로 설정 (파일 제출 = 응답 완료로 간주)
  - 업로드된 각 파일은 클릭 시 새 탭에서 열기/다운로드, 옆의 X로 개별 삭제 가능
  - 파일을 전부 삭제해도 체크 상태는 **자동으로 해제되지 않는다** (수동 체크와 마찬가지로, 체크 해제는 체크박스를 통해서만)
  - 파일 없이 체크박스만 수동으로 체크/해제하는 기존 동작은 그대로 유지 (예: 메일로 직접 받았거나 구두로 확인한 경우)
- **권한**: 로그인한 모든 팀원이 어떤 회신자 줄에든 업로드 가능 (기존 체크 권한과 동일한 "누구나" 정책) — 담당자가 이메일 등으로 대신 받은 파일을 대리 업로드하는 것도 허용

### 기존 동작 유지 (변경 없음)

- 등록/수정 모달의 첨부파일 섹션 (`coll-file-input`, `coll-attachment-list`) — 그대로
- 상세 화면 하단의 `cd-attachments` (등록자 참고자료 표시) — 그대로
- `toggleCheck`의 수동 체크/해제 — 그대로, 단 `submitted[key]` 객체를 갱신할 때 기존 `files` 필드를 보존해야 함 (현재 구현은 `item.submitted[key] = {checked, at}`로 완전히 덮어써서 `files`가 날아간다 — 이번 작업에서 `{...item.submitted[key], checked, at}` 형태로 고쳐야 함)

---

## 4. 구현 범위

- `Pages.collections.renderDetail()`: 체크리스트 각 줄에 파일 업로드 버튼 + 파일 목록 렌더링 추가
- `Pages.collections.toggleCheck()`: `submitted[key]` 갱신 시 기존 `files` 보존하도록 스프레드 적용
- 신규 함수 `uploadReplyFile(id, memberId)`: 파일 input에서 파일을 읽어 Storage 업로드 → `submitted[memberId].files`에 추가 → `checked: true, at: Date.now()` 설정 → Firestore `.update({submitted})` → 재렌더링
- 신규 함수 `removeReplyFile(id, memberId, fileIndex)`: 해당 파일을 배열에서 제거 → Firestore 갱신 → 재렌더링 (체크 상태는 건드리지 않음)
- 전역 shim 2개 추가

## 5. 범위 외

- 파일 업로드 시 자동 체크가 아닌, 체크 시 자동으로 파일 업로드창을 띄우는 등의 역방향 연동은 하지 않는다.
- 취합건 삭제 시 회신자 첨부파일의 Storage 정리(orphaned file cleanup)는 이번에도 범위 밖 (기존 등록자 첨부파일과 동일하게 미정리 상태로 둔다 — 최종 리뷰에서 이미 Minor로 기록된 기존 이슈와 동일 선상).

# 기안서/품의서 검토방 기능 설계

**날짜:** 2026-06-29  
**상태:** 승인됨

---

## 1. 개요

공식 결재 전 사전 검토 공간. 누구나 기안서를 작성·업로드하고, 검토자들이 인라인 코멘트와 자유 코멘트로 피드백을 남기거나 직접 첨삭본을 작성할 수 있다. 결재라인은 순차 승인 없이 참조 표시 전용이다.

---

## 2. 데이터 모델

### `drafts` 컬렉션

```js
{
  id: number,                // Date.now()
  title: string,
  type: 'free' | 'template', // 자유형식 or 양식
  templateId: number | null,
  status: '검토중' | '검토완료' | '반려',
  reviewLine: [
    {
      id: number,
      name: string,           // 표시 이름 (멤버명 or 수기)
      memberId: number | null, // DB 멤버면 ID, 수기면 null
      role: string,           // 직책 자유입력 (팀장, 부문장, 본부장, 대표이사 등)
    }
  ],
  versions: [
    {
      id: number,             // Date.now()
      authorName: string,     // 작성자 이름
      authorUsername: string, // currentUser.username
      label: string,          // 예: "기안자 원본", "이파트장 첨삭"
      content: string,        // 본문 전체 텍스트 (자유형식) or JSON.stringify({field:value}) (양식)
      attachments: [{ name: string, url: string, mimeType: string }],
      inlineComments: [
        {
          id: number,
          authorName: string,
          authorUsername: string,
          lineIndex: number,  // 0-based 줄 번호
          text: string,
          resolved: boolean,
          createdAt: number,
        }
      ],
      createdAt: number,
    }
  ],
  comments: [
    {
      id: number,
      authorName: string,
      authorUsername: string,
      text: string,
      createdAt: number,
    }
  ],
  createdBy: string,          // currentUser.username
  createdAt: number,
  updatedAt: number,
}
```

- `versions[0]`이 기안자 원본. 이후 검토자가 "내 첨삭본 작성" 시 새 버전 push.
- 각 버전은 독립적인 `content` + `inlineComments`를 가진다.

### `draft_templates` 컬렉션 (관리자 전용)

```js
{
  id: number,
  name: string,
  fields: [{ key: string, label: string, type: 'text' | 'textarea' | 'date' }],
  createdBy: string,
  createdAt: number,
}
```

---

## 3. UI 구조

### 3-1. page-drafts (목록)

- 상태 필터 칩: 전체 / 검토중 / 검토완료 / 반려
- 카드 2열 그리드: 제목, 양식명or자유형식, 기안자, 날짜, 결재라인 아바타 표시
- 우상단: "+ 새 기안서" 버튼

### 3-2. 상세 뷰 (전체화면 오버레이)

**상단 헤더**
- 제목, 기안자, 날짜
- 결재라인: 아바타+직책 순서 표시 (보기 전용, 상태 없음)
- 수정/닫기 버튼

**버전 탭 바** (헤더 아래)
- 버전별 탭: `[기안자 아바타] 기안자 원본`, `[첨삭자 아바타] OOO 첨삭`, ...
- "버전 비교" 버튼, "내 첨삭본 작성" 버튼

**본문 영역** (좌 2/3)
- 현재 선택된 버전의 content를 줄 단위로 렌더
- 첨삭본 뷰 시 diff 표시: 삭제 줄(빨강 취소선), 추가 줄(초록 배경)
- 각 줄 우측 말풍선 아이콘: 해당 버전의 인라인 코멘트 표시/입력

**검토 코멘트 패널** (우 1/3)
- 기안서 전체에 대한 자유 코멘트 목록 (전 버전 공통)
- 누구나 등록 가능, 작성자/관리자만 삭제

### 3-3. 기안서 작성/수정 모달

1. 양식 선택: 자유형식 / 관리자 등록 양식
2. 제목 입력
3. 본문 작성 (자유형식: textarea / 양식: 필드별 입력창)
4. 결재라인 구성
   - 팀원 선택 (체크박스 드롭박스) + 직책 입력
   - 수기 추가: 이름 + 직책
   - 순서 위/아래 버튼
5. 파일 첨부 (Firebase Storage, 다중)

### 3-4. 첨삭본 작성 모달

- 원본 content를 초기값으로 채운 textarea (또는 양식 필드)
- 레이블 자동 입력: `{작성자명} 첨삭`
- 파일 첨부 가능
- 저장 시 `draft.versions`에 새 항목 push

### 3-5. 양식 관리 (설정 탭 내 관리자 전용)

- 양식 목록, 추가/수정/삭제
- 필드 구성: 라벨 + 타입(text/textarea/date) 동적 추가

---

## 4. 권한

| 작업 | 허용 대상 |
|------|-----------|
| 기안서 작성 | 전체 멤버 |
| 기안서 수정/삭제 | 작성자 또는 관리자 |
| 첨삭본 작성 | 전체 멤버 |
| 본인 첨삭본 수정/삭제 | 해당 버전 작성자 또는 관리자 |
| 인라인 코멘트 작성 | 전체 멤버 |
| 인라인 코멘트 resolved | 해당 버전 기안자 또는 관리자 |
| 전체 코멘트 작성 | 전체 멤버 |
| 전체 코멘트 삭제 | 작성자 또는 관리자 |
| 양식 관리 | 관리자(jungsoo.kim)만 |

---

## 5. 네비게이션

사이드바에 `검토방` 항목 추가 (projects 아래):
```html
<div class="nav-item" onclick="showPage('drafts')" id="nav-drafts">
  <span class="material-symbols-outlined">rate_review</span> 검토방
</div>
```

---

## 6. Pages 네임스페이스

`Pages.drafts = (() => { ... })()` IIFE 패턴.

전역 shims:
- `openAddDraftModal()`, `saveDraft()`, `openDraftDetail(id)`, `closeDraftDetail()`
- `deleteDraft(id)`, `editDraft(id)`
- `selectDraftVersion(draftId, versionId)`
- `openAddRevisionModal(draftId)`, `saveRevision(draftId)`
- `addInlineComment(draftId, versionId, lineIndex)`, `resolveInlineComment(draftId, versionId, commentId)`, `deleteInlineComment(draftId, versionId, commentId)`
- `addDraftComment(draftId)`, `deleteDraftComment(draftId, commentId)`
- `toggleDraftReviewLine()`, `toggleDraftReviewMember(id)`, `addManualReviewer()`, `removeReviewer(idx)`, `moveReviewer(idx, dir)`

# 기안서/품의서 검토방 기능 설계

**날짜:** 2026-06-29  
**상태:** 승인됨

---

## 1. 개요

공식 결재 전 사전 검토 공간. 누구나 기안서를 작성·업로드하고, 결재라인 검토자들이 인라인 코멘트와 상태 표시로 피드백을 남긴다.

---

## 2. 데이터 모델

### `drafts` 컬렉션

```js
{
  id: number,                // Date.now()
  title: string,
  type: 'free' | 'template', // 자유형식 or 양식
  templateId: number | null, // 양식 사용 시 ID
  content: string,           // 본문 (자유형식: 텍스트 / 양식: JSON.stringify({field:value}))
  attachments: [{ name: string, url: string, mimeType: string }],
  status: '검토중' | '검토완료' | '반려',
  reviewLine: [
    {
      id: number,
      name: string,           // 표시 이름 (수기 or 멤버명)
      memberId: number | null, // DB 멤버면 ID, 수기 입력이면 null
      role: string,           // 직책 자유입력 (팀장, 부문장, 본부장, 대표이사 등)
      status: '대기' | '검토완료' | '수정요청',
      comment: string,        // 전체 코멘트
      reviewedAt: number | null,
    }
  ],
  inlineComments: [
    {
      id: number,
      reviewerId: number,     // reviewLine[].id
      reviewerName: string,
      lineIndex: number,      // 본문 줄 번호 (0-based)
      text: string,
      resolved: boolean,
      createdAt: number,
    }
  ],
  createdBy: string,          // currentUser.username
  createdAt: number,
  updatedAt: number,
}
```

### `draft_templates` 컬렉션 (관리자 전용)

```js
{
  id: number,
  name: string,               // 양식명 (예: "품의서", "업무기안서")
  fields: [
    { key: string, label: string, type: 'text' | 'textarea' | 'date' }
  ],
  createdBy: string,
  createdAt: number,
}
```

---

## 3. UI 구조

### 3-1. page-drafts (목록)

- 상태 필터 칩: 전체 / 검토중 / 검토완료 / 반려
- 카드 그리드 (2열): 제목, 양식명or자유형식, 기안자, 날짜, 결재라인 아바타+상태 아이콘
- 우상단: "+ 새 기안서" 버튼

### 3-2. 상세 뷰 (전체화면 오버레이)

좌측 2/3 — **본문 영역**
- 제목, 작성자, 날짜, 첨부파일 목록
- 본문을 줄 단위로 렌더 (`<div data-line-index="N">`)
- 각 줄 우측에 말풍선 아이콘 (인라인 코멘트 있으면 색상 표시)
- 말풍선 클릭 → 해당 줄 코멘트 입력/표시 패널 인라인 표시

우측 1/3 — **결재라인 패널**
- 결재자 순서 목록 (이름, 직책, 상태 뱃지)
- 각 결재자: 상태 변경 버튼 (검토완료 / 수정요청) + 코멘트 입력 (본인 or 관리자만)
- 작성자·관리자에게 수정/삭제 버튼 표시

### 3-3. 기안서 작성/수정 모달

1. 양식 선택: 자유형식 / 관리자 등록 양식 목록
2. 제목 입력
3. 본문 작성
   - 자유형식: `<textarea>`
   - 양식: 필드별 입력창 자동 생성
4. 결재라인 구성
   - 팀원 선택 (체크박스 드롭박스) + 직책 입력
   - 수기 추가: 이름 + 직책 텍스트 입력
   - 순서 변경: 위/아래 버튼
5. 파일 첨부 (Firebase Storage, 다중 가능)

### 3-4. 양식 관리 (설정 탭 내 관리자 전용 섹션)

- 양식 목록 테이블
- 양식 추가: 이름 + 필드 동적 추가 (라벨, 타입)
- 양식 수정/삭제

---

## 4. 권한

| 작업 | 허용 대상 |
|------|-----------|
| 기안서 작성 | 전체 멤버 |
| 기안서 수정/삭제 | 작성자 또는 관리자 |
| 검토 상태 변경 + 코멘트 | 결재라인에 포함된 멤버(memberId 일치) 또는 관리자 |
| 인라인 코멘트 작성 | 결재라인 포함 멤버 또는 관리자 |
| 인라인 코멘트 resolved | 작성자(기안자) 또는 관리자 |
| 양식 관리 | 관리자(jungsoo.kim)만 |

---

## 5. 네비게이션

사이드바에 `검토방` 항목 추가 (library 위):
```html
<div class="nav-item" onclick="showPage('drafts')" id="nav-drafts">
  <span class="material-symbols-outlined">rate_review</span> 검토방
</div>
```

---

## 6. Pages 네임스페이스

`Pages.drafts = (() => { ... })()` IIFE 패턴.

전역 shims: `openAddDraftModal()`, `saveDraft()`, `openDraftDetail(id)`, `closeDraftDetail()`, `setReviewStatus(draftId, reviewerId, status)`, `submitReviewComment(draftId, reviewerId)`, `addInlineComment(draftId, lineIndex)`, `resolveInlineComment(draftId, commentId)`, `deleteInlineComment(draftId, commentId)`, `deleteDraft(id)`, `editDraft(id)`.

# 센터 현황 대시보드 개편 설계

**날짜:** 2026-06-29  
**상태:** 승인됨

---

## 1. 개요

기존 목록형 아코디언 레이아웃을 W센터/V센터 분리 카드 그리드로 교체하고, 카드 클릭 시 우측 슬라이드 패널(A3 방식)로 상세 현황을 표시한다. 운영 지표(가동률·물동량)는 담당자가 수기 입력한다.

---

## 2. 데이터 모델

### `centers` 컬렉션 (기존 확장)

```js
{
  id: number,
  name: string,
  code: string,
  type: 'W' | 'V' | null,        // 필수 선택, null = 미분류
  managerId: number,              // 본사 담당자 멤버 ID (관리자만 지정)
  centerChief: string,            // 센터장 이름 (수기)
  centerMembers: string,          // 센터원 (수기)
  address: string,
  area: number,                   // 면적 (평)
  landlord: string | null,        // 임대인, null = 자가센터
  leaseStart: string,             // 'YYYY-MM-DD'
  leaseEnd: string,
  monthlyRent: number,
  managementPlan: string,         // 경영계획
  favor: {
    rf: number,                   // 단위: 개
    fitout: number,               // 단위: 개
    ti: number,                   // 단위: 원
    note: string                  // Favor 특이사항
  },
  operationNote: string,          // 운영 특이사항
  photos: [{ url: string, name: string }],  // Firebase Storage

  // W센터 전용
  occupancy: {
    leased: number,               // 임차 평수
    total: number                 // 전체 평수
  },
  tenants: [{
    id: number,
    name: string,
    contractStart: string,
    contractEnd: string,
    area: number,
    monthlyRent: number,
    managementFee: number,
    noticePeriod: number          // 계약만료 전 통보기간 (개월)
  }],
  salesList: [{
    id: number,
    company: string,              // 업체명
    area: number,                 // 평수
    period: string,               // 기간
    product: string,              // 상품
    note: string                  // 특이사항
  }],

  // V센터 전용
  client: string,                 // 고객사
  clientContractStart: string,
  clientContractEnd: string,
  workScope: string,              // 업무범위
  contractor: string,             // 도급사
  contractorStart: string,
  contractorEnd: string,
  contractedWork: string,         // 위탁업무
  annualContractFee: number,      // 연 도급비
  contractorHeadcount: number,    // 도급인원
  volumes: [{
    id: number,
    label: string,                // 자유 입력 (보관, 입고, 출고, 냉동입고 등)
    value: number,
    unit: string                  // 자유 입력 + 제안값: PLT / Unit / CNTR / Each
  }],

  order: number,
  status: string,                 // 정상 / 이슈 / 점검
  issue: string,
  createdAt: number,
  updatedAt: number,
}
```

---

## 3. UI 구조

### 3-1. page-centers (카드 그리드)

```
[ 검색 ]                                         [ 센터 추가 (관리자만) ]

┌────────── W센터 ──────────┐   ┌────────── V센터 ──────────┐
│  [카드]  [카드]  [카드]   │   │  [카드]  [카드]           │
│  [카드]                   │   │                           │
└───────────────────────────┘   └───────────────────────────┘

[ 미분류 센터 (type 미설정, 흐리게 표시) ]
```

**카드 내용:**
- 센터명 + W/V 타입 뱃지 + 상태 뱃지 (정상/이슈/점검)
- 센터코드 · 본사 담당자 아바타
- W센터: 가동률 게이지 바 (임차 평수/전체 평수, %)
- V센터: volumes 항목들 수평 나열 (label: value unit)
- type 미설정 센터: 별도 "미분류" 섹션에 흐리게 표시

### 3-2. 상세 패널 (우측 70% 슬라이드 오버레이)

카드 클릭 → 배경을 약간 어둡게 하고 우측에서 슬라이드인.  
배경의 W/V 카드 그리드는 유지됨.

**패널 헤더:**
- 센터명 + 타입 뱃지
- 수정 버튼 (담당자·관리자 노출) / 삭제 버튼 (관리자만 노출)
- X 닫기 버튼

**탭 구조:**

#### 탭 1: 기본정보·사진
좌우 2열:
- 좌: 센터코드, 주소, 면적(평), 임대인(없으면 "자가센터" 표시), 임대기간, 월 임대료, 경영계획, 운영 특이사항
- 우: 본사 담당자, 센터장, 센터원, Favor (RF개·Fitout개·TI원 + 특이사항), 센터 사진 갤러리

#### 탭 2: 계약/임차
**W센터:**
- 임차사 목록 테이블: 임차사명 / 계약기간 / 평수 / 월 임차료 / 관리비 / 통보기간
- 영업리스트 테이블: 업체명 / 평수 / 기간 / 상품 / 특이사항
- 행 추가/삭제 (담당자·관리자만)

**V센터:**
- 고객사, 고객사 계약기간 (시작~종료)
- 업무범위
- 도급사, 도급계약기간, 위탁업무, 연 도급비, 도급인원

#### 탭 3: 운영현황
**W센터:**
- 가동률 큰 원형 게이지 + 임차 평수 / 전체 평수 수기 입력 (담당자·관리자만 편집 버튼 노출)
- 가동률 = 임차 평수 / 전체 평수 × 100%

**V센터:**
- 물동량 지표 카드들 (label · value · unit)
- 항목 추가/삭제 버튼 (담당자·관리자만)
- 항목 편집: label(자유 입력), value(숫자), unit(자유 입력 + 제안: PLT/Unit/CNTR/Each)

**공통 하단:**
- 관련 진행 업무 목록 (해당 centerId를 가진 tasks, 완료 제외)

### 3-3. 센터 추가/수정 모달

기존 모달 확장:
- **type 선택 (W/V) 필수** — 미선택 시 저장 불가
- managerId 드롭박스 (관리자만 편집 가능, 담당자는 비활성)
- 기본 필드: 이름, 코드, 주소, 면적, 상태, 이슈

---

## 4. 권한

| 작업 | 허용 대상 |
|------|-----------|
| 센터 추가 | 관리자(jungsoo.kim)만 |
| 센터 삭제 | 관리자만 |
| 담당자(managerId) 지정·변경 | 관리자만 |
| 센터 기본정보 수정 | 담당자 + 관리자 |
| 운영 지표 수정 (가동률·물동량 값) | 담당자 + 관리자 |
| 임차사·영업리스트·물동량 항목 추가/삭제/편집 | 담당자 + 관리자 |
| 사진 업로드/삭제 | 담당자 + 관리자 |

담당자 판별: `currentUser.memberId === center.managerId || currentUser.username === 'jungsoo.kim'`

---

## 5. 네비게이션

기존 사이드바 항목 유지 (`showPage('centers')`).

---

## 6. Pages 네임스페이스

`Pages.centers = (() => { ... })()` 기존 IIFE 전면 재작성.

전역 shims:
- `openCenterDetail(id)`, `closeCenterDetail()`
- `selectCenterTab(tab)`
- `editCenter(id)`, `deleteCenter(id)`, `saveCenter()`
- `saveCenterMetrics(id)` — 가동률/물동량 수기 저장
- `addTenant(id)`, `deleteTenant(id, tenantId)`
- `addSalesItem(id)`, `deleteSalesItem(id, itemId)`
- `addVolume(id)`, `deleteVolume(id, volumeId)`, `saveVolumes(id)`
- `uploadCenterPhoto(id)`, `deleteCenterPhoto(id, photoUrl)`
- `searchCenters(q)`, `moveCenter(id, dir)`

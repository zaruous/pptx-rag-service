# 대시보드 인포그래픽 설계

## 목표

벡터 적재가 끝난 문서/슬라이드 현황을 메인 페이지에서 즉시 파악할 수 있도록  
카테고리 분포, 인덱싱 상태, 업로드 추이, 검수 큐, 최근 활동을 인포그래픽으로 제공한다.

## 1. 메인 페이지 레이아웃

```text
+---------------------------------------------------------------+
|  헤더 (워크스페이스 선택 / 검색 바 / 사용자)                  |
+---------------------------------------------------------------+
| KPI 카드                                                     |
|  [전체 문서] [전체 슬라이드] [인덱싱 성공률] [최근 24h 업로드]|
+---------------------------------------------------------------+
| 카테고리 분포 (도넛 + 스택드 바)                              |
|  - 기능별                                                     |
|  - 업종별                                                     |
|  - 유형별                                                     |
+---------------------------------------------------------------+
| 업로드/적재 타임라인 (라인 / 영역 차트)                       |
|  - 일자별 문서 수, 슬라이드 수, 청크 수                       |
+---------------------------------------------------------------+
| 인덱싱 상태 (스택드 바 / 깔때기)                              |
|  PENDING -> PARSING -> EMBEDDING -> INDEXING -> SUCCESS/FAIL  |
+---------------------------------------------------------------+
| 검수 큐 + 최근 업로드 (리스트 카드)                           |
+---------------------------------------------------------------+
```

## 2. 위젯 정의

### 2-1. KPI 카드

| 카드 | 값 |
|---|---|
| 전체 문서 | active 상태 문서 수 |
| 전체 슬라이드 | 적재 완료 슬라이드 수 |
| 인덱싱 성공률 | 최근 30일 `SUCCESS / 전체` 비율 |
| 최근 24h 업로드 | 24시간 내 업로드 건수 |
| 검수 대기 | review_status = PENDING_REVIEW 합계 |

### 2-2. 카테고리 분포

- 형태: 도넛 차트(축별) + 클릭 시 드릴다운
- 데이터: `dashboard_snapshots.snapshot_type = CATEGORY_DISTRIBUTION`
- 정렬: 문서 수 desc, 상위 N개 외엔 `기타`로 묶음
- 인터랙션: 분포 슬라이스 클릭 → 해당 카테고리 필터로 문서 목록 이동

### 2-3. 업로드 타임라인

- 형태: 라인 또는 영역 차트
- 기본 기간: 최근 30일, 토글로 7일/90일/1년
- 측정값: 문서 수, 슬라이드 수, chunk 수 (toggle)
- 데이터: `dashboard_snapshots.snapshot_type = UPLOAD_TIMELINE`

### 2-4. 인덱싱 상태

- 형태: 깔때기 또는 스택드 바
- 상태값: `PENDING`, `PARSING`, `EMBEDDING`, `INDEXING`, `SUCCESS`, `FAILED`, `PARTIAL_FAILED`
- 데이터: `dashboard_snapshots.snapshot_type = INGESTION_STATUS`
- 클릭 시 실패 잡 목록으로 이동

### 2-5. 검수 큐

- 형태: 리스트 카드 (상위 5건)
- 정보: 문서명, 사유(`LOW_CONFIDENCE`, `CATEGORY_MISMATCH`, `WORKFLOW_REVIEW` 등), 등록 시각
- 이동: 검수 페이지로 이동

### 2-6. 최근 업로드

- 형태: 리스트 카드 (상위 5건)
- 정보: 문서명, 카테고리 칩, 상태, 업로더, 업로드 시각

## 3. 데이터 모델

### DashboardSnapshot

| 필드 | 타입 | 설명 |
|---|---|---|
| `snapshotId` | bigint | PK |
| `workspaceId` | bigint | 워크스페이스 |
| `snapshotType` | enum | `KPI`, `CATEGORY_DISTRIBUTION`, `UPLOAD_TIMELINE`, `INGESTION_STATUS`, `REVIEW_QUEUE`, `RECENT_DOCUMENTS` |
| `payloadJson` | text | 위젯별 직렬화된 응답 |
| `capturedAt` | datetime | 캡처 시각 |
| `expiresAt` | datetime | TTL |

### 캐시 정책

- 위젯별 TTL 차등 설정
  - KPI / 카테고리 분포 / 인덱싱 상태: 30초
  - 업로드 타임라인: 5분
  - 검수 큐 / 최근 업로드: 즉시 (이벤트 기반 무효화)
- 적재 이벤트(`SlideChunkIndexed`, `IngestionJobCompleted`) 발생 시 관련 스냅샷 무효화
- 캐시 미스 시 RDB 집계 + Chroma `count_by_metadata` 호출

## 4. API 응답 예시

### `GET /api/dashboard/overview`

```json
{
  "kpi": {
    "documentCount": 124,
    "slideCount": 3120,
    "indexingSuccessRate": 0.965,
    "uploadsLast24h": 7,
    "pendingReviews": 12
  },
  "categories": {
    "function": [
      {"code": "QUALITY_MANAGEMENT", "labelKo": "품질관리", "documentCount": 32},
      {"code": "SALES", "labelKo": "영업", "documentCount": 21}
    ],
    "industry": [
      {"code": "MANUFACTURING", "labelKo": "제조", "documentCount": 67},
      {"code": "DISTRIBUTION", "labelKo": "유통/소매", "documentCount": 24}
    ],
    "docType": [
      {"code": "OPERATION_MANUAL", "labelKo": "업무매뉴얼", "documentCount": 45},
      {"code": "PROPOSAL", "labelKo": "제안서", "documentCount": 30}
    ]
  },
  "timeline": [
    {"date": "2026-05-07", "documents": 3, "slides": 84, "chunks": 412},
    {"date": "2026-05-08", "documents": 5, "slides": 132, "chunks": 678}
  ],
  "status": {
    "PENDING": 2,
    "PARSING": 1,
    "EMBEDDING": 0,
    "INDEXING": 1,
    "SUCCESS": 120,
    "FAILED": 0,
    "PARTIAL_FAILED": 0
  },
  "recentDocuments": [
    {
      "documentId": "doc-1234",
      "documentName": "2026_품질매뉴얼_v2.pptx",
      "categories": {
        "function": ["품질관리"],
        "industry": ["제조"],
        "docType": "업무매뉴얼"
      },
      "status": "SUCCESS",
      "uploadedAt": "2026-05-14T09:21:00Z"
    }
  ]
}
```

## 5. 클라이언트 구현 가이드 (React + Vite)

### 컴포넌트 구조

```text
features/dashboard/
├─ DashboardPage.tsx
├─ widgets/
│   ├─ KpiCards.tsx
│   ├─ CategoryDonut.tsx
│   ├─ UploadTimeline.tsx
│   ├─ IngestionFunnel.tsx
│   ├─ ReviewQueueList.tsx
│   └─ RecentDocumentsList.tsx
├─ hooks/
│   ├─ useDashboardOverview.ts   # react-query
│   └─ useDashboardLive.ts       # SSE / polling
└─ adapters/
    └─ chartLib.ts               # Recharts / ECharts 어댑터
```

### 데이터 로딩

- `react-query` `useQuery(['dashboard-overview', workspaceId], fetchOverview, { refetchInterval: 30_000 })`
- 잡 완료 SSE 수신 시 `queryClient.invalidateQueries(['dashboard-overview'])`
- 차트는 데이터 변환 후 메모이즈 (`useMemo`)

### 시각화 어댑터

- 기본은 `recharts` (간단, 번들 작음)
- 깔때기/복잡 인포그래픽이 필요한 위젯만 `echarts-for-react`
- 어댑터를 두어 차트 라이브러리를 위젯 단위로 교체 가능하게 설계

### 인터랙션

- 도넛 슬라이스 클릭 → 문서 목록 페이지로 카테고리 필터 적용 (`?categoryFunction=...`)
- 인덱싱 상태의 `FAILED` 클릭 → 실패 잡 목록 페이지
- 검수 큐 항목 클릭 → 검수 상세 페이지

## 6. 실시간 갱신

- 서버에서 `IngestionJobCompleted`, `CategoryUpdated` 이벤트 발생 시
  - in-memory 캐시 무효화
  - SSE 채널 (`/api/dashboard/events`)로 `{type, workspaceId}` 통지
- 클라이언트는 SSE 수신 시 해당 위젯만 `invalidateQueries`

## 7. 성능 / 안전

- 대시보드 API는 워크스페이스 권한 필터 필수
- 스냅샷 payload는 응답 직전에 권한 마스킹
- 5초 이상 걸리는 집계는 워커가 미리 만들어 둔 스냅샷만 반환
- 차트 데이터는 최대 N(예: 100) 포인트로 다운샘플링

## 8. 향후 확장 후보

- 검색 품질 위젯: no-answer 비율, 평균 응답 시간, 자주 묻는 질의 키워드
- 카테고리 추세: 월별/분기별 카테고리 비중 변화
- 문서 품질 히트맵: 슬라이드별 confidence 분포
- 인기 문서: evidence 노출 빈도 상위 N

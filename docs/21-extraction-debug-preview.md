# 추출 결과 디버그 미리보기 설계

## 목표

파싱·임베딩 파이프라인을 디버깅하고 추출 품질을 육안으로 확인할 수 있도록,  
각 문서의 페이지(슬라이드)별 추출 결과를 **마크다운 형식**으로 제공한다.

운영자·개발자가 "이 문서에서 무엇이 추출되었는가"를 빠르게 검증할 수 있어야 한다.

---

## 1. 마크다운 미리보기 구조

### 문서 단위 미리보기

````markdown
# 📄 문서 파싱 결과 미리보기

**파일명:** 2024_출하프로세스_v3.pptx
**포맷:** PPTX
**총 페이지 수:** 12
**파서 버전:** PptxDocumentParser v1.2.0
**파싱 시각:** 2026-05-14T09:21:00Z
**파싱 품질:** overall=0.87 | 텍스트 문자 수=4,210 | 표=3 | 이미지=5 | OCR 적용 페이지=1

---

## ⚠️ 파싱 경고
- PAGE_8: 이미지만 존재, OCR 적용됨
- PAGE_11: 표 구조 근사 처리 (TABLE_APPROXIMATE)

---

## 페이지별 추출 결과

### 🔖 페이지 1

**제목:** 출하 승인 프로세스 개요
**슬라이드 유형 후보:** `workflow`, `horizontal-flow`

**본문 텍스트:**
- 주문 접수
- 생산 계획
- 품질 검사
- 출하 승인

**도형/SmartArt 텍스트:**
- 영업, 생산관리, 품질

**발표자 노트:**
> 검사 실패 시 재작업으로 회송

**레이아웃 힌트 (요약):**
| 요소 유형 | 텍스트 | x | y |
|---|---|---|---|
| shape | 주문 접수 | 120 | 140 |
| shape | 생산 계획 | 340 | 140 |
| arrow | (없음) | 260 | 140 |

**이미지:** 없음
**OCR 결과:** 없음

**플래그:** 없음

---

### 🔖 페이지 8

**제목:** 검수 기준표
**슬라이드 유형 후보:** `table`

**본문 텍스트:** (없음)

**이미지:** 1개 (이미지 영역 감지됨)
**OCR 결과:**
> 검사 항목 | 기준값 | 허용 범위
> 외관 검사 | 0건 결함 | -
> 치수 측정 | ±0.1mm | ±0.2mm

**플래그:** `SCAN_DETECTED=false`, `OCR_APPLIED=true`
````

---

## 2. API 설계

### 2-1. 문서 전체 미리보기

```
GET /api/documents/{documentVersionId}/debug/preview
```

**응답:**
```json
{
  "format": "PPTX",
  "filename": "sample.pptx",
  "pageCount": 12,
  "parserMeta": {
    "name": "PptxDocumentParser",
    "version": "1.2.0"
  },
  "quality": {
    "overallScore": 0.87,
    "textCharCount": 4210,
    "tableCount": 3,
    "imageCount": 5,
    "ocrPageCount": 1,
    "qualityWarnings": ["PAGE_8: OCR 적용", "PAGE_11: TABLE_APPROXIMATE"]
  },
  "pages": [ { /* PageDebugView */ } ]
}
```

### 2-2. 페이지 단위 마크다운 미리보기

```
GET /api/documents/{documentVersionId}/debug/preview/{pageNo}/markdown
```

- `Content-Type: text/markdown`
- 해당 페이지의 추출 결과를 가공된 마크다운 텍스트로 반환
- 브라우저 렌더링 또는 IDE/터미널에서 바로 확인 가능

### 2-3. 전체 마크다운 다운로드

```
GET /api/documents/{documentVersionId}/debug/preview/markdown
```

- 전 페이지를 단일 `.md` 파일로 반환
- `Content-Disposition: attachment; filename="{filename}_debug.md"`

### 2-4. Chunk 뷰 (임베딩 전 텍스트 확인)

```
GET /api/documents/{documentVersionId}/debug/chunks
```

```json
{
  "documentVersionId": "ver-1",
  "chunks": [
    {
      "chunkId": "chunk-001",
      "chunkType": "DOC_META",
      "pageNo": 0,
      "text": "파일명: sample.pptx\n총 12장\n카테고리: 품질관리...",
      "tokenCount": 88,
      "confidence": null,
      "flags": []
    },
    {
      "chunkId": "chunk-002",
      "chunkType": "SUMMARY",
      "pageNo": 1,
      "text": "주문 접수 후 생산 계획과 품질 검사를 거쳐 출하 승인으로 진행된다.",
      "tokenCount": 42,
      "confidence": 0.87,
      "flags": []
    }
  ]
}
```

---

## 3. 마크다운 렌더링 규칙

### 섹션 구조

| 섹션 | 표시 조건 |
|---|---|
| 페이지 제목 (H3) | 항상 |
| 슬라이드 유형 후보 | 항상 |
| 본문 텍스트 | 1개 이상 존재할 때 |
| 도형/SmartArt 텍스트 | 1개 이상 존재할 때 |
| 표 | 존재할 때, 마크다운 테이블 형식 |
| 발표자 노트 | PPTX에서 존재할 때 |
| 레이아웃 힌트 | 요소가 3개 이상일 때 (요약 테이블) |
| 이미지 | 개수만 표시 (바이너리 미포함) |
| OCR 결과 | OCR 적용 시 |
| 플래그 | 비어 있지 않을 때 |

### 텍스트 가공 원칙

- 원문 텍스트는 수정하지 않는다 (디버그 목적)
- 빈 문자열, 공백만 있는 항목은 `(없음)` 으로 표시
- 개인정보 마스킹 정책이 활성화된 환경에서는 마스킹 후 렌더링

---

## 4. 프론트엔드 디버그 화면 (React)

### 위치

`/documents/{documentId}/versions/{versionId}/debug`

### 레이아웃

```text
+----------------------------------------------------+
| 문서명: sample.pptx   포맷: PPTX   파서: v1.2.0   |
+------+---------+----------------------------------+
|      | 페이지1 |   📄 페이지 1 미리보기            |
| 페   | 페이지2 |   제목: 출하 승인 프로세스         |
| 이   | 페이지3 |                                    |
| 지   | ⚠️ 페이지8 |  **본문:**                     |
| 목   | ...     |   - 주문 접수                      |
| 록   |         |   - 생산 계획                      |
|      |         |                                    |
|      |         |  **레이아웃:**                     |
|      |         |  | 요소 | 텍스트 | x | y |        |
|      |         |  | shape | 주문접수 | 120 | 140 |  |
+------+---------+----------------------------------+
| [전체 MD 다운로드] [Chunk 보기] [재처리 요청]      |
+----------------------------------------------------+
```

### 컴포넌트

```text
features/debug/
├─ DocumentDebugPage.tsx
├─ PageNavSidebar.tsx       # 페이지 목록 (경고 아이콘 포함)
├─ PagePreviewPanel.tsx     # 선택된 페이지 마크다운 렌더링
├─ ChunkListDrawer.tsx      # Chunk 텍스트/토큰/신뢰도 목록
├─ QualitySummaryCard.tsx   # 파싱 품질 점수
└─ hooks/
   ├─ usePagePreview.ts
   └─ useChunkList.ts
```

### 마크다운 렌더링

- `react-markdown` + `remark-gfm` (테이블, 코드 블록 지원)
- 플래그 배지는 커스텀 컴포넌트로 색상 구분 (경고=노랑, 오류=빨강)

---

## 5. 디버그 데이터 저장 정책

| 항목 | 저장 여부 | 위치 | 보존 기간 |
|---|---|---|---|
| `PageParseResult` 전체 JSON | 저장 | 파일 저장소 (`/debug/{versionId}.json`) | 7일 (설정 가능) |
| 마크다운 렌더링 결과 | 미저장 (동적 생성) | - | - |
| `ParseQuality` | 저장 | RDB `document_versions` | 영구 |
| `ParseFlags` | 저장 | RDB `slides` 또는 `parse_warnings` 컬럼 | 영구 |
| Chunk 텍스트 목록 | 미저장 (Chroma에서 조회) | - | - |

- 7일 후 자동 삭제 (스토리지 절약)
- `enableDebugPersist=false` 환경(prod)에서는 디버그 JSON 저장 비활성화
- 관리자만 `/debug` 엔드포인트 접근 가능 (권한 분리)

---

## 6. 추출 품질 시각적 지표

운영자가 빠르게 품질을 파악할 수 있도록 `ParseQuality.overallScore` 기반 색상 코드 사용:

| 점수 | 의미 | 색상 |
|---|---|---|
| `>= 0.85` | 양호 | 🟢 |
| `0.70 ~ 0.85` | 보통 (검수 권장) | 🟡 |
| `< 0.70` | 낮음 (검수 필요) | 🔴 |

- 페이지 목록 사이드바에서 낮은 품질 페이지를 `⚠️` 아이콘으로 표시
- 문서 목록 화면에서도 품질 배지 노출 가능

---

## 7. 활용 시나리오

| 시나리오 | 방법 |
|---|---|
| 파이프라인 개발 중 파서 출력 확인 | `GET /debug/preview/{pageNo}/markdown` → 터미널/IDE에서 확인 |
| 운영자가 추출 품질 검토 | 웹 UI 디버그 화면 진입 → 페이지별 탐색 |
| 표/OCR 결과가 이상할 때 | 해당 페이지의 `flags`, `ocrTexts`, `tables` 항목 확인 |
| 전체 파싱 결과를 로컬 검토 | 전체 MD 다운로드 → 에디터에서 열기 |
| 재처리 후 변화 비교 | 이전 버전과 현재 버전 `/debug/preview` 나란히 열기 |
| CI 환경에서 골드셋 비교 | JSON 응답을 기준값과 자동 비교 (`assertThat(parsed).isEqualTo(expected)`) |

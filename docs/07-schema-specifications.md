# 스키마 명세

## 목적

샘플 PPTX 없이도 확정 가능한 공통 계약을 고정한다.  
이 문서는 구현 시 DTO, JSON schema, Chroma metadata, 검수 결과 모델의 기준이 된다.

## 공통 규칙

### 네이밍

- JSON 필드는 `camelCase`
- DB 컬럼은 `snake_case`
- enum 값은 `UPPER_SNAKE_CASE`

### 공통 식별자

| 필드 | 타입 | 설명 |
|---|---|---|
| `workspaceId` | `string` | 검색/권한 범위 ID |
| `documentId` | `string` | 논리 문서 ID |
| `documentVersionId` | `string` | 업로드 버전 ID |
| `slideId` | `string` | 슬라이드 ID |
| `chunkId` | `string` | 청크 ID |
| `jobId` | `string` | 인덱싱 작업 ID |

### 공통 Enum

#### DocumentFormat

- `PPTX`
- `PDF`
- `DOCX`
- `UNKNOWN`

#### SlideType

- `TEXT_HEAVY`
- `TABLE`
- `CHART`
- `WORKFLOW`
- `ORG_CHART`
- `TIMELINE`
- `COMPARISON`
- `MIXED`
- `UNKNOWN`

#### ChunkType

- `DOC_META`
- `RAW`
- `SUMMARY`
- `FACT`
- `WORKFLOW_STEP`
- `WORKFLOW_EDGE`
- `WORKFLOW_EXCEPTION`
- `OCR`
- `NOTE`

#### CategoryAxis

- `FUNCTION`
- `INDUSTRY`
- `DOC_TYPE`

#### CategorySource

- `USER_INPUT`
- `LLM_SUGGEST`
- `OPERATOR_REVIEW`

#### QueryType

- `OVERVIEW`
- `FACT_LOOKUP`
- `COMPARISON`
- `NUMERIC`
- `WORKFLOW_SEQUENCE`
- `WORKFLOW_ACTOR`
- `WORKFLOW_CONDITION`
- `WORKFLOW_EXCEPTION`
- `DOCUMENT_DISCOVERY`
- `UNKNOWN`

#### ReviewStatus

- `NOT_REQUIRED`
- `PENDING_REVIEW`
- `APPROVED`
- `REJECTED`
- `REPROCESS_REQUESTED`

## 1. Slide IR Schema

### 목적

결정론적 추출 결과를 LLM에 전달하기 위한 중간 표현

### 필드 명세

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `workspaceId` | `string` | Y | 범위 ID |
| `documentId` | `string` | Y | 문서 ID |
| `documentVersionId` | `string` | Y | 버전 ID |
| `slideId` | `string` | Y | 슬라이드 ID |
| `slideNo` | `integer` | Y | 사용자 표시 슬라이드 번호 |
| `title` | `string` | N | 슬라이드 제목 |
| `bodyTexts` | `string[]` | Y | 본문 텍스트 목록 |
| `shapeTexts` | `string[]` | Y | 도형/SmartArt 텍스트 |
| `tableTexts` | `string[]` | Y | 표 셀 텍스트 평탄화 결과 |
| `chartTexts` | `string[]` | Y | 축, 범례, 라벨 텍스트 |
| `notes` | `string` | N | 발표자 노트 |
| `ocrTexts` | `string[]` | Y | OCR 결과 |
| `layoutHints` | `LayoutHint[]` | Y | 좌표/순서 정보 |
| `visualTypeCandidates` | `SlideType[]` | Y | 규칙 기반 후보 유형 |
| `rawJoinedText` | `string` | Y | 전체 검색용 합성 텍스트 |
| `language` | `string` | N | `ko`, `en` 등 |
| `sourceHash` | `string` | Y | 파일 해시 |

### LayoutHint

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `elementId` | `string` | Y | 요소 ID |
| `elementType` | `string` | Y | `TEXT_BOX`, `TABLE`, `IMAGE`, `ARROW` 등 |
| `text` | `string` | N | 요소 텍스트 |
| `x` | `number` | N | 좌표 |
| `y` | `number` | N | 좌표 |
| `width` | `number` | N | 크기 |
| `height` | `number` | N | 크기 |
| `order` | `integer` | N | 추출 순서 |

## 2. Semantic Extraction Result Schema

### 목적

LLM이 slide IR를 해석해 만드는 구조화 결과

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `slideType` | `SlideType` | Y | 최종 판정 유형 |
| `summary` | `string` | Y | 슬라이드 요약 |
| `facts` | `FactItem[]` | Y | 명시 사실 목록 |
| `searchableParaphrases` | `string[]` | Y | 질의 친화 문장 |
| `workflow` | `WorkflowGraph` | N | 프로세스형일 때만 |
| `confidence` | `ConfidenceBundle` | Y | 신뢰도 |
| `uncertainties` | `string[]` | Y | 불확실 항목 |
| `promptVersion` | `string` | Y | 추출 프롬프트 버전 |
| `schemaVersion` | `string` | Y | 출력 스키마 버전 |
| `modelName` | `string` | Y | 모델명 |

### FactItem

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `type` | `string` | Y | `NUMERIC`, `COMPARISON`, `PROCESS_RULE`, `DEFINITION` 등 |
| `text` | `string` | Y | 검색 가능한 짧은 문장 |
| `evidenceTexts` | `string[]` | Y | 원천 텍스트 참조 |
| `importance` | `number` | N | 0.0~1.0 |

### ConfidenceBundle

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `overall` | `number` | Y | 0.0~1.0 |
| `summary` | `number` | Y | 0.0~1.0 |
| `facts` | `number` | Y | 0.0~1.0 |
| `workflow` | `number` | N | 0.0~1.0 |

## 3. Workflow Graph Schema

### 목적

프로세스형 슬라이드의 단계와 연결 구조 표현

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `processType` | `string` | Y | `APPROVAL`, `PRODUCTION_FLOW`, `ISSUE_HANDOFF` 등 |
| `actors` | `string[]` | Y | 참여 주체 목록 |
| `steps` | `WorkflowStep[]` | Y | 단계 목록 |
| `edges` | `WorkflowEdge[]` | Y | 정상 흐름 |
| `exceptions` | `WorkflowEdge[]` | Y | 예외 흐름 |
| `entrySteps` | `integer[]` | N | 시작 단계 번호 |
| `terminalSteps` | `integer[]` | N | 종료 단계 번호 |

### WorkflowStep

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `stepNo` | `integer` | Y | 1부터 시작하는 논리 순서 |
| `name` | `string` | Y | 단계 이름 |
| `actor` | `string` | N | 담당 주체 |
| `description` | `string` | N | 단계 설명 |
| `evidenceTexts` | `string[]` | Y | 근거 텍스트 |

### WorkflowEdge

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `fromStepNo` | `integer` | Y | 시작 단계 |
| `toStepNo` | `integer` | Y | 다음 단계 |
| `condition` | `string` | N | 분기 조건 |
| `edgeType` | `string` | Y | `NORMAL`, `APPROVAL`, `REJECT`, `LOOPBACK` 등 |
| `evidenceTexts` | `string[]` | Y | 근거 텍스트 |

## 4. Chunk Assembly Schema

### 목적

Chroma 적재 전 최종 청크 공통 포맷

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `chunkId` | `string` | Y | 청크 ID |
| `chunkType` | `ChunkType` | Y | 청크 유형 |
| `text` | `string` | Y | 임베딩 대상 텍스트 |
| `documentId` | `string` | Y | 문서 ID |
| `documentVersionId` | `string` | Y | 버전 ID |
| `slideId` | `string` | Y | 슬라이드 ID |
| `slideNo` | `integer` | Y | 슬라이드 번호 |
| `sectionTitle` | `string` | N | 섹션명 |
| `sourceEvidenceTexts` | `string[]` | Y | 원천 텍스트 참조 |
| `confidence` | `number` | N | chunk 신뢰도 |
| `reviewStatus` | `ReviewStatus` | Y | 검수 상태 |

## 5. Chroma Metadata Schema

### 목적

검색과 필터링에 필요한 최소 메타데이터 고정

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `workspaceId` | `string` | Y | 범위 ID |
| `documentId` | `string` | Y | 문서 ID |
| `documentVersionId` | `string` | Y | 버전 ID |
| `documentName` | `string` | Y | 문서명 |
| `totalSlideCount` | `integer` | Y | 문서 전체 페이지 수 |
| `slideId` | `string` | N | 슬라이드 ID (`DOC_META`는 null) |
| `slideNo` | `integer` | Y | 슬라이드 번호 (`DOC_META`는 0) |
| `chunkId` | `string` | Y | 청크 ID |
| `chunkType` | `ChunkType` | Y | 청크 유형 |
| `slideType` | `SlideType` | N | 슬라이드 유형 (`DOC_META`는 null) |
| `categoryFunction` | `string` | N | 기능 코드 join 문자열 (`A|B`) |
| `categoryIndustry` | `string` | N | 업종 코드 join 문자열 |
| `categoryDocType` | `string` | N | 문서 유형 코드 |
| `categoryLabels` | `string` | N | 한글/영문 라벨 join 문자열 |
| `language` | `string` | N | 언어 |
| `sourceHash` | `string` | Y | 파일 해시 |
| `confidence` | `number` | N | 청크 신뢰도 |
| `reviewStatus` | `ReviewStatus` | Y | 검수 상태 |
| `embeddingModel` | `string` | Y | `bge-m3` |
| `embeddingDim` | `integer` | Y | 1024 |
| `promptVersion` | `string` | N | 프롬프트 버전 |
| `schemaVersion` | `string` | N | 스키마 버전 |
| `modelName` | `string` | N | LLM 모델명 |

> Chroma 메타 필드는 스칼라(`str`, `int`, `float`, `bool`)만 안전하므로 다중 카테고리는 구분자(`|`) join 또는 별도 boolean flag로 평탄화한다.

## 6. Query Classification Result Schema

### 목적

질의 타입 판정과 retrieval plan 생성을 위한 포맷

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `queryType` | `QueryType` | Y | 질의 유형 |
| `keywords` | `string[]` | Y | 핵심 키워드 |
| `actors` | `string[]` | N | 사용자 질문의 주체 |
| `requestedStep` | `string` | N | 특정 단계 표현 |
| `expectedAnswerMode` | `string` | Y | `SUMMARY`, `FACT`, `WORKFLOW`, `MIXED` |
| `classificationConfidence` | `number` | Y | 0.0~1.0 |

## 7. Retrieval Evidence Schema

### 목적

최종 답변에 첨부되는 근거 포맷

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `documentId` | `string` | Y | 문서 ID |
| `documentName` | `string` | Y | 문서명 |
| `slideId` | `string` | Y | 슬라이드 ID |
| `slideNo` | `integer` | Y | 슬라이드 번호 |
| `chunkId` | `string` | Y | 청크 ID |
| `chunkType` | `ChunkType` | Y | 근거 유형 |
| `snippet` | `string` | Y | 사용자 노출 요약 |
| `score` | `number` | Y | 최종 점수 |
| `confidence` | `number` | N | 근거 신뢰도 |
| `reviewStatus` | `ReviewStatus` | Y | 검수 상태 |

## 8. Review Decision Schema

### 목적

운영자 검수 결과와 재처리 요구를 기록

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `reviewId` | `string` | Y | 검수 ID |
| `targetType` | `string` | Y | `SLIDE`, `CHUNK`, `WORKFLOW` |
| `targetId` | `string` | Y | 대상 ID |
| `reviewStatus` | `ReviewStatus` | Y | 결과 상태 |
| `reasonCode` | `string` | N | `LOW_CONFIDENCE`, `WRONG_ACTOR`, `WRONG_EDGE` 등 |
| `reviewComment` | `string` | N | 검수 의견 |
| `requestedAction` | `string` | N | `RETRY_OCR`, `RETRY_PROMPT`, `MANUAL_FIX` |

## 9. 신뢰도 기준값

샘플 없이도 기본 기준은 고정할 수 있다.

| 대상 | 기준 | 기본 정책 |
|---|---|---|
| `overall >= 0.85` | 높음 | 자동 적재 |
| `0.70 <= overall < 0.85` | 보통 | 적재하되 검수 후보 |
| `overall < 0.70` | 낮음 | low-confidence 표시 |
| `workflow < 0.75` | workflow 위험 | workflow chunk 가중치 낮춤 |
| `workflow < 0.60` | workflow 불가 | workflow chunk 비적재 |

## 10. 버전 관리 필수 필드

- `promptVersion`
- `schemaVersion`
- `modelName`
- `modelRevision`
- `embeddingModel` (`bge-m3`)
- `embeddingRevision`
- `ingestionPolicyVersion`
- `categoryTaxonomyVersion`

이 값들은 같은 PPTX라도 재처리 기준을 바꾸는 핵심이므로 누락되면 안 된다.

## 11. Document Upload Request Schema

### 목적

업로드 API 입력 표준화. 카테고리는 사용자가 선택한 값을 그대로 전달.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `workspaceId` | `string` | Y | 검색 범위 |
| `documentName` | `string` | Y | 사용자가 부여한 문서명. 기본은 파일명 |
| `file` | `multipart` | Y | PPTX 바이너리 |
| `categoryFunctionCodes` | `string[]` | N | 기능 코드 다중 선택 |
| `categoryIndustryCodes` | `string[]` | N | 업종 코드 다중 선택 |
| `categoryDocTypeCode` | `string` | N | 문서 유형 단일 선택 |
| `freeTags` | `string[]` | N | taxonomy 미존재 자유 태그. LLM이 정규화 |
| `enableOcr` | `boolean` | N | OCR 보강 여부 |

### 응답

```json
{
  "documentId": "doc-1234",
  "documentVersionId": "ver-1",
  "jobId": "job-987",
  "categories": {
    "function": [{"code": "QUALITY_MANAGEMENT", "label": "품질관리", "source": "USER_INPUT"}],
    "industry": [{"code": "MANUFACTURING", "label": "제조", "source": "USER_INPUT"}],
    "docType": {"code": "OPERATION_MANUAL", "label": "업무매뉴얼", "source": "USER_INPUT"}
  }
}
```

## 12. Document Meta Chunk Schema

### 목적

문서 단위 임베딩 (`DOC_META`) 생성에 사용하는 표준 입력

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `documentVersionId` | `string` | Y | 버전 ID |
| `documentName` | `string` | Y | 파일명 |
| `totalSlideCount` | `integer` | Y | 페이지 수 |
| `documentSummary` | `string` | Y | LLM 생성 문서 요약 |
| `documentKeywords` | `string[]` | Y | 키워드 |
| `categoryFunctionLabels` | `string[]` | Y | 기능 라벨 |
| `categoryIndustryLabels` | `string[]` | Y | 업종 라벨 |
| `categoryDocTypeLabel` | `string` | N | 문서 유형 라벨 |
| `composedText` | `string` | Y | 임베딩 입력으로 합성된 텍스트 |
| `embeddingModel` | `string` | Y | `bge-m3` |

## 13. Document Parse Result Schema

### 목적

`DocumentParser` 출력의 공통 포맷. 포맷별 파서가 동일한 구조로 반환한다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `documentVersionId` | `string` | Y | 버전 ID |
| `format` | `DocumentFormat` | Y | 문서 포맷 |
| `pageCount` | `integer` | Y | 총 페이지 수 |
| `pages` | `PageParseResult[]` | Y | 페이지별 파싱 결과 |
| `parserMeta` | `ParserMeta` | Y | 파서 이름/버전 |
| `quality` | `ParseQuality` | Y | 파싱 품질 |

### PageParseResult

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `pageNo` | `integer` | Y | 1-based 페이지 번호 |
| `pageTitle` | `string` | N | 페이지 제목 또는 헤딩 |
| `bodyTexts` | `string[]` | Y | 본문 텍스트 목록 |
| `shapeTexts` | `string[]` | Y | 도형/SmartArt 텍스트 (PPTX 전용, 타 포맷 빈 배열) |
| `tables` | `TableParseResult[]` | Y | 표 목록 |
| `chartTexts` | `string[]` | Y | 차트 라벨 텍스트 |
| `notes` | `string` | N | 발표자 노트 (PPTX 전용) |
| `imageRegions` | `ImageRegion[]` | Y | 이미지 영역 메타데이터 |
| `layoutHints` | `LayoutHint[]` | Y | 좌표/순서 정보 |
| `ocrTexts` | `string[]` | Y | OCR 결과 |
| `hyperlinks` | `string[]` | Y | 하이퍼링크 URL |
| `flags` | `ParseFlags` | Y | 파싱 경고 플래그 |

### ParseFlags

| 필드 | 타입 | 설명 |
|---|---|---|
| `scanDetected` | `boolean` | 텍스트가 거의 없어 스캔본으로 판단 |
| `tableApproximate` | `boolean` | 표 구조 근사 처리 (PDF 한계) |
| `headingHeuristic` | `boolean` | 제목이 스타일 아닌 폰트 크기 휴리스틱으로 추정 |
| `ocrApplied` | `boolean` | OCR 처리됨 |
| `multimodalApplied` | `boolean` | 멀티모달 LLM 보강 처리됨 |
| `partialParseFailed` | `boolean` | 일부 요소 파싱 실패 |
| `failedPageNos` | `integer[]` | 파싱 실패한 페이지 번호 목록 |

### ParseQuality

| 필드 | 타입 | 설명 |
|---|---|---|
| `overallScore` | `number` | 0.0~1.0 종합 품질 점수 |
| `textCharCount` | `integer` | 추출된 텍스트 총 문자 수 |
| `tableCount` | `integer` | 추출된 표 수 |
| `imageCount` | `integer` | 이미지 영역 수 |
| `ocrPageCount` | `integer` | OCR 처리된 페이지 수 |
| `dominantLanguage` | `string` | 지배적 언어 코드 |
| `qualityWarnings` | `string[]` | 경고 메시지 목록 (예: `PAGE_3: 텍스트 없음`) |

## 14. Debug Preview Response Schema

### 목적

`/api/documents/{versionId}/debug/preview` API 응답 포맷

| 필드 | 타입 | 설명 |
|---|---|---|
| `format` | `DocumentFormat` | 포맷 |
| `filename` | `string` | 파일명 |
| `pageCount` | `integer` | 총 페이지 수 |
| `parserMeta` | `ParserMeta` | 파서 정보 |
| `quality` | `ParseQuality` | 파싱 품질 |
| `pages` | `PageDebugView[]` | 페이지별 디버그 뷰 |

### PageDebugView

| 필드 | 타입 | 설명 |
|---|---|---|
| `pageNo` | `integer` | 페이지 번호 |
| `pageTitle` | `string` | 제목 |
| `bodyTexts` | `string[]` | 본문 |
| `shapeTexts` | `string[]` | 도형 텍스트 |
| `tables` | `string[][]` | 표 (행 × 열) |
| `notes` | `string` | 발표자 노트 |
| `ocrTexts` | `string[]` | OCR 결과 |
| `imageCount` | `integer` | 이미지 수 |
| `layoutHintCount` | `integer` | 레이아웃 힌트 수 |
| `flags` | `ParseFlags` | 플래그 |
| `markdownUrl` | `string` | 해당 페이지 마크다운 URL |

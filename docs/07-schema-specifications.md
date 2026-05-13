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

- `RAW`
- `SUMMARY`
- `FACT`
- `WORKFLOW_STEP`
- `WORKFLOW_EDGE`
- `WORKFLOW_EXCEPTION`
- `OCR`
- `NOTE`

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
| `slideId` | `string` | Y | 슬라이드 ID |
| `slideNo` | `integer` | Y | 슬라이드 번호 |
| `chunkId` | `string` | Y | 청크 ID |
| `chunkType` | `ChunkType` | Y | 청크 유형 |
| `slideType` | `SlideType` | Y | 슬라이드 유형 |
| `language` | `string` | N | 언어 |
| `sourceHash` | `string` | Y | 파일 해시 |
| `confidence` | `number` | N | 청크 신뢰도 |
| `reviewStatus` | `ReviewStatus` | Y | 검수 상태 |
| `promptVersion` | `string` | N | 프롬프트 버전 |
| `schemaVersion` | `string` | N | 스키마 버전 |
| `modelName` | `string` | N | 모델명 |

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
- `ingestionPolicyVersion`

이 값들은 같은 PPTX라도 재처리 기준을 바꾸는 핵심이므로 누락되면 안 된다.

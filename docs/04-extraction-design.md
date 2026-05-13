# 추출 설계

## 목표

PPTX로부터 RAG의 기준 데이터가 될 정보를 최대한 안정적으로 추출한다. 특히 다음 두 종류를 구분한다.

- 일반 정보형 슬라이드: 개념, 설명, 표, 비교, 수치
- 프로세스형 슬라이드: 워킹 프로세스, 승인 흐름, 부서 handoff, 예외 분기

## 핵심 원칙

1. PPTX에서 먼저 구조적으로 추출하고, 그 다음 LLM이 의미를 정리한다.
2. LLM 출력은 자유 문장보다 `JSON schema` 기반 구조화 출력을 우선한다.
3. 원문 계층과 의미 계층을 같이 저장한다.
4. 프로세스 슬라이드는 `요약`만으로 끝내지 않고 `graph`를 뽑아야 한다.

## 단계별 파이프라인

### 1. Deterministic Extraction

`Apache POI`와 보조 추출기로 다음 데이터를 모은다.

- slide title
- body text
- shape text
- table text
- chart labels
- notes text
- image region metadata
- slide element order
- approximate layout position

### 2. Visual Enrichment

텍스트가 적거나 이미지 중심인 슬라이드에만 선택적으로 적용한다.

- OCR
- 썸네일 생성
- 필요 시 멀티모달 LLM

### 3. Intermediate Representation 생성

슬라이드별 원천 데이터를 통합한 `slide IR`을 만든다.

### 4. LLM Semantic Extraction

LLM은 `slide IR`를 입력받아 다음을 생성한다.

- summary
- facts
- searchable paraphrases
- workflow graph
- confidence

### 5. Retrieval Chunk Assembly

최종적으로 `raw`, `summary`, `fact`, `workflow` 계층 chunk를 만든다.

## Slide Intermediate Representation 예시

```json
{
  "slideNo": 8,
  "title": "출하 승인 프로세스",
  "bodyTexts": ["주문 접수", "생산 계획", "품질 검사", "출하 승인"],
  "shapeTexts": ["영업", "생산관리", "품질"],
  "tableTexts": [],
  "chartTexts": [],
  "notes": "검사 실패 시 재작업으로 회송",
  "ocrTexts": [],
  "layoutHints": [
    {"elementType": "shape", "text": "주문 접수", "x": 120, "y": 140},
    {"elementType": "arrow", "text": "", "x": 260, "y": 140}
  ],
  "visualTypeCandidates": ["workflow", "horizontal-flow"]
}
```

## 일반 정보형 슬라이드 추출 항목

| 필드 | 설명 |
|---|---|
| `summary` | 슬라이드 핵심 메시지 |
| `facts[]` | 명시적 사실, 수치, 비교 결과 |
| `keywords[]` | 검색 키워드 |
| `searchableParaphrases[]` | 질의 친화 표현 |
| `confidence` | 추출 신뢰도 |

## 프로세스형 슬라이드 추출 항목

워크플로우를 포함한 슬라이드는 아래 구조를 우선 추출한다.

```json
{
  "processType": "approval-workflow",
  "actors": ["영업", "생산관리", "품질"],
  "steps": [
    {"stepNo": 1, "name": "주문 접수", "actor": "영업"},
    {"stepNo": 2, "name": "생산 계획 수립", "actor": "생산관리"},
    {"stepNo": 3, "name": "품질 검사", "actor": "품질"},
    {"stepNo": 4, "name": "출하 승인", "actor": "품질"}
  ],
  "edges": [
    {"fromStepNo": 1, "toStepNo": 2, "condition": null},
    {"fromStepNo": 2, "toStepNo": 3, "condition": "생산 완료"},
    {"fromStepNo": 3, "toStepNo": 4, "condition": "검사 합격"}
  ],
  "exceptions": [
    {"fromStepNo": 3, "toStepNo": 2, "condition": "검사 불합격"}
  ],
  "summary": "주문 접수 후 생산 계획과 품질 검사를 거쳐 출하 승인으로 진행되며, 검사 불합격 시 재작업 단계로 되돌아간다."
}
```

## 인포그래픽 유형별 전략

| 유형 | 추출 난이도 | 전략 |
|---|---|---|
| 표/비교표 | 낮음 | 표 셀 텍스트 중심 추출 |
| 타임라인 | 중간 | 순서와 날짜 텍스트 결합 |
| 단계형 프로세스 | 중간 | step/edge 추출 |
| Swimlane 프로세스 | 높음 | actor와 lane title 보존 |
| 조직도 | 중간 | parent-child 관계 추정 |
| 복합 차트 | 높음 | 축/범례/라벨 우선 추출 |
| 디자인 중심 인포그래픽 | 높음 | OCR 및 멀티모달 보강 |

## Chroma 적재 전략

### 1. Raw Chunk

- 원문 근거 보존용
- 표, 도형, 노트, OCR 원문 기반

### 2. Summary Chunk

- 슬라이드 맥락 검색용
- 인포그래픽 개요 질의 대응

### 3. Fact Chunk

- 수치, 비교, 규칙, 명시 정보 검색용

### 4. Workflow Chunk

- 단계, 담당자, 분기, 예외 흐름 질의 대응
- 예:
  - `품질 검사는 생산 완료 후 수행된다`
  - `검사 불합격 시 생산 계획 단계로 되돌아간다`
  - `출하 승인은 품질 부서가 담당한다`

## 질의 유형과 대응 데이터

| 질의 유형 | 예시 | 우선 사용할 chunk |
|---|---|---|
| 개요 질의 | "이 PPT에서 출하 프로세스가 어떻게 설명되나" | `SUMMARY`, `WORKFLOW` |
| 순서 질의 | "다음 단계는 무엇인가" | `WORKFLOW` |
| 역할 질의 | "품질팀은 어디서 개입하나" | `WORKFLOW`, `FACT` |
| 조건 질의 | "출하 승인은 어떤 조건에서 진행되나" | `WORKFLOW`, `FACT` |
| 예외 질의 | "불합격이면 어디로 돌아가나" | `WORKFLOW` |
| 수치 질의 | "성장률이 가장 높은 부문은" | `FACT`, `RAW` |

## 신뢰도 관리

- `confidence`가 낮은 workflow 추출 결과는 `workflow_chunk` 적재를 보류하거나 낮은 가중치를 준다.
- 근거가 부족한 슬라이드는 `summary only`로 남기고 workflow 판정을 강제하지 않는다.
- 답변 생성 시에는 workflow 구조와 원문 chunk를 함께 넣어 환각을 줄인다.

## 구현 권장 사항

- `SlideIrBuilder`
- `VisualTypeClassifier`
- `SlideSemanticExtractor`
- `WorkflowExtractor`
- `ChunkAssembler`
- `RetrievalReranker`

위 모듈을 분리하면 나중에 OCR 엔진이나 LLM 모델을 바꿔도 파이프라인 수정 범위가 작다.

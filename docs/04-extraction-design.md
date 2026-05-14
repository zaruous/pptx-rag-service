# 추출 설계

## 목표

PPTX / PDF / DOCX 등 다양한 포맷의 문서로부터 RAG의 기준 데이터가 될 정보를 안정적으로 추출한다.  
포맷에 상관없이 동일한 파이프라인이 작동하도록 `DocumentParser` 인터페이스로 추상화한다.  
포맷별 추출 능력 차이와 한계는 `ParseFlags`와 `ParseQuality`로 명시하고 LLM 보강·검수 큐로 대응한다.  
포맷별 상세 추출 능력 매트릭스와 파서 클래스 설계는 `20-document-format-support.md` 참고.  
추출 결과 디버깅은 `21-extraction-debug-preview.md` 참고.  
다국어 지원 및 워크플로 맥락 추론 세부 전략은 `23-multilingual-workflow-context.md` 참고.

특히 다음 두 종류를 구분한다.

- 일반 정보형 페이지: 개념, 설명, 표, 비교, 수치
- 프로세스형 페이지: 워킹 프로세스, 승인 흐름, 부서 handoff, 예외 분기 **← PPTX뿐 아니라 PDF·DOCX에서도 추론**

## 핵심 원칙

1. 포맷별 파서(`DocumentParser` 구현체)가 먼저 구조적으로 `PageParseResult`를 만든다.
2. 파이프라인 나머지 단계(IR 생성, LLM 추출, Chunk 조립)는 포맷을 모른다.
3. LLM 출력은 자유 문장보다 `JSON schema` 기반 구조화 출력을 우선한다.
4. 원문 계층과 의미 계층을 같이 저장한다.
5. 프로세스 페이지는 `요약`만으로 끝내지 않고 `graph`를 뽑아야 한다. 이미지 기반 플로차트는 멀티모달 LLM으로 추론한다.
6. 추출 결과는 마크다운 디버그 뷰로 언제든 확인할 수 있어야 한다 (`/debug/preview`).
7. 문서 언어를 자동 감지하고, 언어별 OCR 모델과 LLM 프롬프트를 선택한다.
8. 임베딩은 `bge-m3`로 단일화해 교차 언어 검색(한국어 쿼리 → 영어 문서)을 지원한다.

## 단계별 파이프라인

### 0. Upload-time Metadata Capture

업로드 시점에 사용자/시스템이 다음을 수집한다.

- `documentName` (원본 파일명)
- `categoryFunction` (기능, 다중 선택 가능)
- `categoryIndustry` (업종, 다중 선택 가능)
- `categoryDocType` (문서 유형, 단일 선택 권장)
- `uploaderId`, `workspaceId`
- 파일 해시, 파일 크기

사용자가 카테고리를 일부만 지정했거나 자유 입력했을 경우, LLM 보조 정규화 단계에서 taxonomy 코드로 매핑한다.

### 1. Deterministic Extraction (포맷 독립)

`DocumentParserRegistry`에서 포맷에 맞는 `DocumentParser` 구현체를 찾아 실행한다.  
파서는 `ParseContext`를 입력받아 `DocumentParseResult`(페이지 목록)를 반환한다.

**포맷별 파서:**
- PPTX → `PptxDocumentParser` (Apache POI XSLF)
- PDF → `PdfDocumentParser` (Apache PDFBox 3.x + OCR)
- DOCX → `DocxDocumentParser` (Apache POI XWPF)

**공통 추출 항목 (모든 포맷):**
- page title / heading
- body text
- table text (셀 단위 평탄화)
- image region metadata → OCR 연계
- layout hints (좌표 / 순서)
- hyperlinks

**PPTX 추가 추출:**
- shape text, SmartArt text
- chart labels
- presenter notes
- 정확한 EMU 좌표

**PDF 제약:**
- 도형/SmartArt 텍스트 불가 (OCR/멀티모달 보강 필요)
- 표 구조 근사 (PDFBox 한계, `TABLE_APPROXIMATE` 플래그)
- 스캔 PDF → 자동 OCR 전환

**DOCX 제약:**
- 페이지 번호는 섹션 기반 (실제 페이지 분할은 LibreOffice 옵션)
- 제목은 Heading 스타일 기반, 없으면 폰트 크기 휴리스틱

문서 전체 `page_count`와 `ParseQuality`는 파서 반환 즉시 RDB에 저장한다.

### 1-A. Language Detection

파싱과 동시에 언어를 감지한다.

- `LanguageDetector` (fastText LID.176) → 문서 주 언어 + confidence
- confidence < 0.85 → 페이지별 재감지 (혼합 문서)
- 감지 결과를 `PageParseResult.detectedLanguage`에 저장
- `MultilingualOcrRouter`가 언어별 최적 OCR 모델 선택

### 2. Visual Enrichment

텍스트가 적거나 이미지 중심인 슬라이드에만 선택적으로 적용한다.

- 언어별 OCR 모델 선택 (PaddleOCR v4 CJK / Surya 영문 / Tesseract 보조)
- 썸네일 생성
- **워크플로 다이어그램 감지:** DocLayout-YOLO로 `figure` 블록 식별 후 멀티모달 LLM 호출
- 필요 시 멀티모달 LLM 페이지 전체 해석

### 2-A. Workflow Signal Detection

Visual Enrichment와 병렬로, 텍스트 신호 기반 워크플로 감지를 수행한다.

`WorkflowSignalDetector`가 다음 신호를 탐지:
- 번호 패턴 (`1단계`, `Step 1`, `Phase 1`)
- 화살표 문자 (`→`, `▶`, `=>`)
- 순서 접속사 (`이후`, `다음으로`, `then`, `after`, `upon completion`)
- 담당자 패턴 (`담당: OOO`, `Responsible: `)
- Swimlane 표 (열 = actor, 행 = 단계)

감지 결과는 `PageParseResult.workflowSignals`에 저장되어 LLM 추출 단계에서 활용된다.

### 3. Intermediate Representation 생성

`PageParseResult`를 `SlideIr`(포맷 무관한 공통 중간 표현)으로 변환한다.  
이 단계부터는 포맷 구분 없이 동일한 파이프라인이 동작한다.

**포맷별 변환 주의:**
- PDF의 `TABLE_APPROXIMATE` 플래그가 있으면 IR의 `tableTexts`에 `[근사 추출]` 마킹
- `SCAN_DETECTED`이면 IR의 `bodyTexts` 대부분이 `ocrTexts`에서 온다고 표시
- DOCX의 섹션 기반 페이지는 `pageNo`를 섹션 순번으로 채움
- `workflowSignals`가 있으면 IR에 `visualTypeCandidates`에 `WORKFLOW` 추가
- `detectedLanguage`를 IR에 전달해 LLM 프롬프트 언어 선택에 사용

### 4. LLM Semantic Extraction

LLM은 `slide IR`를 입력받아 다음을 생성한다.

- summary
- facts
- searchable paraphrases
- workflow graph
- confidence

### 4-1. LLM Document-Level Extraction

슬라이드별 추출이 끝나면, 문서 전체 단위로 LLM이 다음을 한 번 더 생성한다.

- `documentSummary` (문서 전체를 1~3문장으로 요약)
- `documentKeywords` (전 슬라이드 키워드 정리)
- `suggestedCategories` (taxonomy 코드 후보)
- `documentLanguageMix` (한국어/영어 비중 등)

이 출력은 사용자가 업로드 시 지정한 카테고리와 비교해 누락/충돌을 검수 큐로 보낸다.

### 5. Retrieval Chunk Assembly

최종적으로 다음 계층 chunk를 만든다.

- `DOC_META`: 문서 단위 메타 chunk (파일명 + 페이지수 + 문서 요약 + 카테고리 라벨 + 키워드)
- `SUMMARY`: 슬라이드 요약 chunk
- `FACT`: 슬라이드 fact chunk
- `RAW`: 원문 chunk
- `WORKFLOW_*`: workflow step/edge/exception chunk
- `OCR`, `NOTE`: 부가 chunk

모든 chunk는 `bge-m3`로 임베딩하고, `embedding_model = bge-m3`, `embedding_dim = 1024` 메타를 함께 기록한다.

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

### 0. Document Meta Chunk (`DOC_META`)

- 문서 버전당 1개
- 임베딩 텍스트 예시:
  - `파일명: 2024_출하프로세스_v3.pptx`
  - `총 24장 / 카테고리: 기능=품질관리,출하관리 / 업종=제조 / 유형=업무매뉴얼`
  - `문서 요약: 주문 접수부터 출하 승인까지의 단계와 품질 검사 분기 조건을 정리한다.`
- 용도: `이 회사에 출하 프로세스 관련 문서 있나?` 같은 문서 발견형 질의, 카테고리 검색

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

- `DocumentParser` (인터페이스)
- `DocumentParserRegistry` (포맷 → 파서 매핑)
- `PptxDocumentParser`, `PdfDocumentParser`, `DocxDocumentParser`
- `DocumentFormatDetector` (확장자 + MIME + 매직 바이트)
- `SlideIrBuilder` (PageParseResult → SlideIr 변환)
- `VisualTypeClassifier`
- `SlideSemanticExtractor`
- `DocumentSemanticExtractor` (문서 단위 요약/카테고리 후보)
- `CategoryNormalizer`
- `WorkflowExtractor`
- `ChunkAssembler` (DOC_META 포함)
- `EmbeddingClient` (`bge-m3` 어댑터)
- `RetrievalReranker`
- `DebugPreviewRenderer` (PageParseResult → 마크다운 변환)

포맷 추가 시 `DocumentParser` 구현체만 추가하면 나머지 파이프라인은 수정 불필요.  
OCR 엔진·LLM 모델 교체도 각 어댑터만 변경하면 된다.

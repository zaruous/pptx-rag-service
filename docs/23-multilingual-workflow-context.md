# 다국어 지원 및 워크플로 맥락 추론 설계

## 목표

1. **다국어 지원**: 문서 언어에 무관하게 파싱·임베딩·검색·응답이 동작한다.  
   사용자가 한국어로 질문해도 영어 문서에서 근거를 찾을 수 있어야 한다.

2. **워크플로 맥락 추론**: PPTX뿐 아니라 PDF·DOCX의 플로차트·순서도·표 기반 프로세스에서도  
   단계·담당자·조건·예외 흐름을 추론해 RAG가 "다음 단계는?", "누가 담당?" 질의에 답할 수 있어야 한다.

---

## 1. 다국어 지원 전략

### 1-1. 지원 언어 우선순위

| 티어 | 언어 | 비고 |
|---|---|---|
| Primary | 한국어 (ko), 영어 (en) | 전 기능 완전 지원 |
| Secondary | 일본어 (ja), 중국어 간체 (zh-Hans), 중국어 번체 (zh-Hant) | OCR·임베딩 지원 |
| Extended | 독일어, 프랑스어, 스페인어, 아랍어 등 | bge-m3 범위 내 |

### 1-2. 언어 감지

문서 단위 + 페이지 단위 이중 감지를 수행한다.

```text
[문서 전체 텍스트 샘플]
       │
       ▼
┌─────────────────────────┐
│  langdetect / fastText  │  언어 코드 + confidence 반환
└─────────────────────────┘
       │
       ├── confidence >= 0.85 → 단일 언어 확정
       ├── confidence < 0.85  → 페이지별 재감지 (혼합 문서)
       └── CJK 감지 시        → 한/중/일 세분화 (cld3 또는 regex 보조)
```

**사용 라이브러리:**
- `langdetect` (Python, 55개 언어) — 빠른 기본 감지
- `fasttext` LID.176 모델 (176개 언어, 정확도 높음)
- `cld3` (Google Compact Language Detector 3)

**저장:**
- `document_versions.detected_language` — 문서 주 언어 코드
- `slides.detected_language` — 페이지별 언어 (혼합 문서용)
- Chroma 메타: `language` 필드

### 1-3. 언어별 OCR 모델 선택

```text
언어 감지 결과
    │
    ├── ko / ja / zh  → PaddleOCR v4 (CJK 특화)
    ├── ar / fa / ur  → PaddleOCR (아랍어 모델) + 텍스트 방향 보정
    ├── en / de / fr  → Surya OCR 또는 Tesseract 5
    └── 혼합 (ko+en) → PaddleOCR + Surya 병렬 → confidence 기반 병합
```

### 1-4. 교차 언어 임베딩 (Cross-lingual Retrieval)

**핵심 원칙:** 사용자가 어떤 언어로 질의하든, 다른 언어 문서에서도 의미적으로 근접한 chunk를 찾아야 한다.

```text
[사용자 질의: "출하 승인 조건이 뭔가요?" (한국어)]
        │ bge-m3 임베딩
        ▼
  [쿼리 벡터 (1024차원)]
        │ cosine similarity
        ▼
  [Chroma 검색]
        │
        ├── 한국어 chunk: "출하 승인은 품질 검사 합격 후 진행" → 높은 유사도
        └── 영어 chunk: "Shipping approval requires QC pass" → 같은 공간 내 유사도
```

**bge-m3 선택 이유:**
- 100개 언어 지원 (한국어 포함)
- 한-영, 영-한 교차 검색 품질 우수
- 동일 임베딩 공간 → 언어별 컬렉션 분리 불필요
- 1024차원 Dense + Sparse + ColBERT 다중 표현 지원

**추가 전략: 쿼리 번역 보강 (선택)**
```text
사용자 질의 → langdetect → 문서 주 언어 != 질의 언어 시
    → LLM/번역 API로 문서 언어로 번역
    → 원문 쿼리 + 번역 쿼리 두 번 검색
    → 결과 합산 재랭킹 (Reciprocal Rank Fusion)
```

### 1-5. LLM 프롬프트 다국어 처리

```text
LLM 호출 시:
  1. slide IR의 detectLanguage 확인
  2. 해당 언어로 system prompt 작성
  3. 출력 언어는 항상 문서 언어와 일치하게 강제
     (영어 문서 → facts/summary 영어 출력)
  4. 최종 사용자 응답은 질의 언어 기준으로 LLM 재번역
```

**시스템 프롬프트 템플릿:**
```text
ko: "입력 텍스트는 한국어입니다. 모든 출력을 한국어로 작성하세요..."
en: "The input is in English. All output must be in English..."
ja: "入力テキストは日本語です。すべての出力を日本語で..."
zh: "输入文本为中文。所有输出请使用中文..."
mixed: "문서는 한국어와 영어가 혼재합니다. 원문 언어를 유지하세요..."
```

### 1-6. 다국어 청크 분할 (토크나이저)

언어별로 적절한 분할 전략이 다르다.

| 언어군 | 특이사항 | 전략 |
|---|---|---|
| 한국어 | 어절 기반, 조사 결합 | bge-m3 토크나이저 (BPE) 기준 |
| 영어 | 단어 기반 | 문장 경계 + bpe |
| 중국어 | 띄어쓰기 없음 | jieba 분절 후 bpe |
| 일본어 | 히라가나/한자 혼합 | sudachi 분절 후 bpe |
| 아랍어 | 오른쪽에서 왼쪽 | 방향 정규화 후 bpe |

**최대 토큰 수:** bge-m3 기준 8,192 token — 초과 시 512 token 단위로 슬라이딩 윈도우 분할 (128 overlap)

### 1-7. 다국어 Taxonomy 라벨

카테고리 마스터는 `label_ko`, `label_en`을 기본으로, 필요 시 `label_ja`, `label_zh` 추가.  
검색 시 `category_labels` 필드에 전 언어 라벨을 `|` join으로 저장해 키워드 매칭을 다국어로 지원한다.

### 1-8. 프론트엔드 다국어 (i18n)

```text
web/src/
  └── i18n/
      ├── ko.json   # 한국어 (기본)
      ├── en.json
      └── ja.json   # 선택
```

- `react-i18next` + `i18next-browser-languagedetector`
- 브라우저 언어 자동 감지 → 설정에서 수동 변경 가능
- 날짜/숫자 포맷은 `Intl.DateTimeFormat` / `Intl.NumberFormat` 표준 사용

---

## 2. 워크플로 맥락 추론 전략

### 2-1. 워크플로 감지 신호

포맷별로 워크플로가 있음을 나타내는 신호:

| 포맷 | 시각적 신호 | 텍스트 신호 |
|---|---|---|
| PPTX | SmartArt, 화살표 도형, Swimlane 박스 | "단계", "→", "다음", "이후" |
| PDF | 플로차트 이미지, 화살표 벡터, 번호 리스트 | "Step N:", "Phase", "→", numbered headings |
| DOCX | 도형 DrawingML, 번호 목록, 표 기반 프로세스 | Heading 계층 + 번호, "프로세스", "절차" |

**텍스트 패턴 기반 1차 감지:**
```text
규칙 트리거:
- "1단계", "2단계", "Step 1", "Phase 1" 패턴
- "→", "=>", "▶", "►" 화살표 문자
- "이후", "다음으로", "완료 시", "합격 시", "then", "after", "upon"
- 순서 보장 명사: "절차", "프로세스", "workflow", "process", "flow"
- 담당자 패턴: "담당: OOO", "Responsible: ", "Owner:"
```

### 2-2. 포맷별 워크플로 추출 전략

#### PDF 워크플로 추출

```text
PDF 페이지
    │
    ├── [이미지 기반 플로차트 감지]
    │       │
    │       ├── DocLayout-YOLO로 "figure" 블록 감지
    │       ├── 블록 crop → 멀티모달 LLM (GPT-4o / Claude Vision)
    │       │   Prompt: "이 다이어그램에서 프로세스 단계, 화살표, 담당자, 분기 조건을 JSON으로 추출"
    │       └── WorkflowGraph 구조로 변환
    │
    ├── [텍스트 기반 순서 리스트 감지]
    │       │
    │       ├── 번호 패턴 인식 → 순서 있는 단계로 처리
    │       ├── 화살표 텍스트 ("→", "▶") → edge 관계 추출
    │       └── 헤딩 계층 (H1→H2→H3) → 부모-자식 흐름 추론
    │
    └── [표 기반 워크플로 감지]
            │
            ├── 행이 단계, 열이 담당자/조건인 표 패턴
            └── Swimlane 표 → actor별 단계 분리
```

#### DOCX 워크플로 추출

```text
DOCX
    │
    ├── Heading 계층 분석 → 섹션 흐름
    │     H1: 프로세스 명
    │     H2: 단계명
    │     H3: 세부 조건
    │
    ├── 번호 목록 (1., 2., 3.) → 순서 있는 WorkflowStep
    │
    ├── DrawingML 도형 → PPTX와 동일한 방식으로 텍스트 추출
    │     + 도형 간 연결선 텍스트 → edge condition
    │
    └── 표 기반 Swimlane → 열 = actor, 행 = 단계
```

#### PPTX 워크플로 추출 (기존 방식 강화)

기존 SmartArt/도형 추출에 다음 추가:
- 멀티 슬라이드 워크플로: 슬라이드 연속성 감지 (제목 패턴, 단계 번호 이어짐)
- 슬라이드 노트의 워크플로 보조 설명 통합

### 2-3. 멀티모달 워크플로 추출 (비전 LLM)

텍스트만으로 추론이 어려운 이미지 기반 다이어그램 처리:

**입력:**
- 플로차트 이미지 crop (DocLayout-YOLO bbox 기반)
- 주변 텍스트 컨텍스트 (캡션, 제목)

**프롬프트 전략:**
```text
System: "당신은 비즈니스 프로세스 다이어그램 분석 전문가입니다.
         다이어그램에서 다음을 JSON으로 추출하세요:
         - processType: 프로세스 유형
         - actors: 참여 주체 목록
         - steps: [{stepNo, name, actor, description}]
         - edges: [{fromStepNo, toStepNo, condition}]
         - exceptions: [{fromStepNo, toStepNo, condition}]
         - uncertainties: 불확실한 부분
         원문에 없는 내용을 생성하지 마세요."

User: [이미지] + "이 다이어그램의 언어: {언어}"
```

**신뢰도 규칙:**
- 비전 LLM confidence >= 0.75 → `workflow_chunk` 적재
- 0.60 ~ 0.75 → 적재 + 검수 큐 (`VISION_EXTRACTED` 플래그)
- < 0.60 → summary 텍스트만 적재, workflow 판정 보류

### 2-4. 멀티페이지 / 멀티슬라이드 워크플로

하나의 워크플로가 여러 페이지에 걸쳐 있는 경우:

```text
연속성 감지 신호:
- 페이지 제목에 "1/3", "2/3", "continued", "계속" 포함
- 마지막 단계 번호가 다음 페이지 첫 번호와 이어짐
- 동일 processType + 동일 actors 집합이 연속 페이지에 등장
```

**처리 방식:**
```java
// 연속 워크플로 병합
public WorkflowGraph mergeMultiPageWorkflows(
    List<WorkflowGraph> candidates,
    MergePolicy policy
) { ... }
```

- 같은 문서 내 연속성 감지 → `WorkflowGraph.continuedFromPage` 필드로 연결
- 병합된 전체 워크플로도 별도 chunk로 적재 (`WORKFLOW_MERGED` 타입)

### 2-5. 워크플로 맥락 청크 생성

추출된 워크플로를 RAG가 맥락 추론에 활용할 수 있는 텍스트 chunk로 변환한다.

**변환 규칙:**

| 청크 유형 | 생성 텍스트 예시 |
|---|---|
| `WORKFLOW_STEP` | `"[3단계] 품질 검사: 생산팀이 수행하며, 합격 기준은 ±0.1mm이다."` |
| `WORKFLOW_EDGE` | `"품질 검사(3단계)가 합격하면 출하 승인(4단계)으로 진행한다."` |
| `WORKFLOW_EXCEPTION` | `"품질 검사(3단계)에서 불합격 시 생산 계획(2단계)으로 되돌아간다."` |
| `WORKFLOW_CONTEXT` | `"출하 프로세스는 총 4단계로 구성된다: 주문접수 → 생산계획 → 품질검사 → 출하승인. 담당부서: 영업, 생산, 품질."` |
| `WORKFLOW_ACTOR` | `"품질팀은 품질검사(3단계)와 출하승인(4단계)에 관여한다."` |

**다국어 워크플로 텍스트 생성:**
- LLM이 추출한 구조를 문서 언어로 텍스트화
- 검색 보강을 위해 주요 언어(ko/en) 번역 병렬 청크 생성 (선택)

### 2-6. 워크플로 맥락 추론 쿼리 처리

검색 시 워크플로 맥락을 활용하는 방식:

```text
질의: "출하 승인 거부 시 어떻게 되나?" (WORKFLOW_EXCEPTION 유형)
          │
          ▼
   QueryClassifier → WORKFLOW_EXCEPTION
          │
          ▼
   Chroma 검색:
   - chunkType = WORKFLOW_EXCEPTION 우선
   - chunkType = WORKFLOW_EDGE 보완
   - "거부", "불합격", "반려" 키워드 매칭
          │
          ▼
   WorkflowContextEnricher:
   - 해당 exception edge의 fromStep, toStep 정보 로드
   - 관련 actor 정보 추가
   - 전체 프로세스 흐름 요약 컨텍스트 삽입
          │
          ▼
   LLM 답변:
   "품질 검사(3단계)에서 불합격 시, 생산 계획(2단계)으로 되돌아갑니다.
    [근거: 출하_프로세스.pptx 8페이지]"
```

### 2-7. 워크플로 신뢰도 및 검수

| 추출 방법 | 기본 신뢰도 | 검수 정책 |
|---|---|---|
| PPTX SmartArt 직접 추출 | 0.85 | 자동 적재 |
| PDF 텍스트 기반 파싱 | 0.70 | 검수 후보 |
| 멀티모달 LLM 추출 | 0.65 | 검수 필수 |
| DOCX 번호 목록 파싱 | 0.75 | 검수 후보 |
| 멀티페이지 병합 | 0.60 | 검수 필수 |

`VISION_EXTRACTED`, `MULTIPAGE_MERGED`, `TEXT_INFERRED` 플래그로 추출 방법을 기록한다.

---

## 3. 다국어 × 워크플로 교차 처리

### 3-1. 다국어 워크플로 문서에서의 과제

| 상황 | 문제 | 해결 |
|---|---|---|
| 한-영 혼합 PPT의 워크플로 | actor 이름이 영어, 단계명이 한국어 | 언어 혼합 허용, 통합 entity로 처리 |
| 일본어 Swimlane PDF | OCR 정확도 낮음 + 수직 텍스트 | PaddleOCR 일본어 + 90도 회전 처리 |
| 아랍어 오른쪽→왼쪽 플로차트 | 읽기 순서 역전 | RTL 방향 감지 + 단계 번호 역순 정렬 |
| 다국어 actor 이름 정규화 | "영업팀" = "Sales Dept" | 문서 내 언어 통일 후 정규화 |

### 3-2. 다국어 쿼리 → 다국어 워크플로 검색

```text
쿼리: "What happens if QC fails?" (영어)
   │
   ▼
langdetect → en
   │
   ▼
bge-m3 임베딩 (영어 쿼리)
   │ cosine similarity
   ▼
한국어 워크플로 chunk:
  "품질검사(3단계)에서 불합격 시 생산계획(2단계)으로 되돌아간다"
  → 교차 언어 유사도: 0.87 (bge-m3 덕분)
   │
   ▼
LLM 답변 생성: 쿼리 언어(en)로 응답
  "If QC fails at Step 3, the process returns to Step 2 (Production Planning).
   [Source: 출하_프로세스.pptx, Page 8]"
```

---

## 4. 구현 컴포넌트

### 4-1. 추가/변경 서비스

```text
extraction/
├── language/
│   ├── LanguageDetector.java          -- 문서/페이지 언어 감지
│   ├── MultilingualOcrRouter.java     -- 언어별 OCR 모델 라우팅
│   └── LanguageNormalizer.java        -- 혼합 언어 정규화
│
├── workflow/
│   ├── WorkflowSignalDetector.java    -- 텍스트/시각 신호 감지
│   ├── PptxWorkflowExtractor.java     -- PPTX 전용 (기존)
│   ├── PdfWorkflowExtractor.java      -- PDF 텍스트/이미지 기반
│   ├── DocxWorkflowExtractor.java     -- DOCX 구조 기반
│   ├── VisionWorkflowExtractor.java   -- 멀티모달 LLM 기반
│   ├── MultiPageWorkflowMerger.java   -- 멀티페이지 병합
│   └── WorkflowContextChunkBuilder.java -- 맥락 청크 생성
│
└── chunking/
    └── MultilingualChunkSplitter.java -- 언어별 청크 분할
```

### 4-2. 쿼리 처리 추가

```text
retrieval/
└── context/
    ├── WorkflowContextEnricher.java   -- 워크플로 컨텍스트 삽입
    └── CrossLingualQueryExpander.java -- 쿼리 번역 보강 (선택)
```

### 4-3. Python 파싱 서비스 추가

```python
# parsing_service/
├── language/
│   ├── detect.py          # langdetect + fasttext
│   └── ocr_router.py      # 언어별 OCR 선택
│
└── workflow/
    ├── vision_extractor.py    # 멀티모달 LLM 워크플로 추출
    ├── pdf_flow_parser.py     # PDF 텍스트 흐름 파싱
    └── docx_flow_parser.py    # DOCX 구조 기반 흐름 파싱
```

---

## 5. ERD 추가 필드

```sql
-- document_versions
ALTER TABLE document_versions ADD COLUMN detected_language VARCHAR(10);
ALTER TABLE document_versions ADD COLUMN language_confidence FLOAT;
ALTER TABLE document_versions ADD COLUMN is_multilingual BOOLEAN DEFAULT false;
ALTER TABLE document_versions ADD COLUMN secondary_languages VARCHAR(100); -- "en|ja" join

-- slides
ALTER TABLE slides ADD COLUMN detected_language VARCHAR(10);
ALTER TABLE slides ADD COLUMN workflow_extracted_method VARCHAR(30);
  -- 'SHAPE_DIRECT', 'TEXT_INFERRED', 'VISION_LLM', 'DOCX_LIST', 'PDF_TEXT', 'MULTIPAGE_MERGED'

-- workflow_nodes (기존 테이블 확장)
ALTER TABLE workflow_nodes ADD COLUMN source_language VARCHAR(10);
ALTER TABLE workflow_nodes ADD COLUMN extraction_confidence FLOAT;

-- workflow_edges (기존 테이블 확장)
ALTER TABLE workflow_edges ADD COLUMN condition_language VARCHAR(10);
ALTER TABLE workflow_edges ADD COLUMN extraction_method VARCHAR(30);
```

---

## 6. Chroma 메타데이터 추가 필드

| 필드 | 설명 |
|---|---|
| `language` | chunk 텍스트 언어 코드 |
| `isMultilingual` | 혼합 언어 여부 |
| `workflowExtractionMethod` | `SHAPE_DIRECT`, `TEXT_INFERRED`, `VISION_LLM` 등 |
| `workflowActors` | 관련 actor 이름 join 문자열 (검색 필터용) |
| `workflowStepNo` | 해당 단계 번호 (edge의 경우 fromStepNo) |

---

## 7. 다국어 지원 품질 측정 지표

| 지표 | 측정 방법 | 목표 |
|---|---|---|
| 언어 감지 정확도 | 골드셋 언어 라벨 vs 감지 결과 | > 98% |
| 교차 언어 검색 Recall@5 | KO 쿼리로 EN 문서 검색 | > 85% |
| 워크플로 단계 추출 F1 | 골드셋 step 수 vs 추출 step 수 | > 80% |
| 워크플로 edge 정확도 | 정방향/역방향 edge 매칭률 | > 75% |
| 멀티모달 워크플로 정확도 | 비전 추출 vs 수동 라벨 | > 70% |

---

## 8. 운영 설정

```yaml
app:
  language:
    default: ko
    supported: [ko, en, ja, zh-Hans, zh-Hant]
    detection-threshold: 0.85
    cross-lingual-query: true          # 쿼리 번역 보강 여부
    query-translation-provider: llm    # llm | deepl | none

  workflow:
    enable-vision-extraction: true     # 멀티모달 LLM 플로차트 추출
    vision-provider: claude            # claude | openai | local
    vision-confidence-threshold: 0.65
    enable-multipage-merge: true
    multipage-lookahead-pages: 3       # 연속 페이지 탐색 범위
    text-inference-enabled: true       # 텍스트 기반 워크플로 추론
    text-inference-min-steps: 3        # 최소 N개 단계 감지 시 워크플로 판정

  embedding:
    model: bge-m3
    dim: 1024
    distance: cosine
    multilingual-query-expansion: false # 쿼리 번역 병렬 검색
```

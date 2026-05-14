# LLM 추출 설계

## 목표

LLM을 `원문 추출기`가 아니라 `의미 구조화기`로 사용한다.  
정확도를 높이기 위해 LLM의 역할, 입력, 출력, 검증, fallback을 명확히 고정한다.

## 역할 분리

### LLM이 하면 안 되는 일

- PPTX 바이너리에서 직접 원문 읽기
- 슬라이드 번호 추정
- 표/도형 위치를 임의로 상상해서 복원
- 원문에 없는 수치나 단계 생성

### LLM이 해야 하는 일

- 추출된 원천 데이터를 바탕으로 슬라이드 의미 요약
- 명시 정보와 암시 정보를 구분한 facts 정리
- 검색 친화 표현 생성
- 프로세스형 슬라이드의 step/actor/edge 구조화
- 추출 신뢰도와 불확실성 표시

## 입력 계약

LLM 입력은 반드시 `slide IR` 형태로 고정한다.

```json
{
  "documentName": "sample.pptx",
  "slideNo": 8,
  "title": "출하 승인 프로세스",
  "bodyTexts": ["주문 접수", "생산 계획", "품질 검사", "출하 승인"],
  "shapeTexts": ["영업", "생산관리", "품질"],
  "tableTexts": [],
  "chartTexts": [],
  "notes": "검사 실패 시 재작업으로 회송",
  "ocrTexts": [],
  "layoutHints": [
    {"elementType": "shape", "text": "주문 접수", "x": 120, "y": 140, "order": 1},
    {"elementType": "shape", "text": "생산 계획", "x": 340, "y": 140, "order": 2}
  ],
  "visualTypeCandidates": ["workflow", "horizontal-flow"]
}
```

## 출력 계약

자유 형식 텍스트가 아니라 구조화 출력으로 제한한다.

### 공통 출력 스키마

```json
{
  "slideType": "workflow",
  "summary": "주문 접수 후 생산 계획과 품질 검사를 거쳐 출하 승인으로 진행된다.",
  "facts": [
    {
      "type": "process-rule",
      "text": "품질 검사는 생산 계획 이후 수행된다",
      "evidenceTexts": ["생산 계획", "품질 검사"]
    }
  ],
  "searchableParaphrases": [
    "출하 승인 절차",
    "품질 검사가 포함된 출하 업무 흐름"
  ],
  "workflow": {
    "processType": "approval-workflow",
    "actors": ["영업", "생산관리", "품질"],
    "steps": [
      {"stepNo": 1, "name": "주문 접수", "actor": "영업"},
      {"stepNo": 2, "name": "생산 계획", "actor": "생산관리"},
      {"stepNo": 3, "name": "품질 검사", "actor": "품질"},
      {"stepNo": 4, "name": "출하 승인", "actor": "품질"}
    ],
    "edges": [
      {"fromStepNo": 1, "toStepNo": 2, "condition": null},
      {"fromStepNo": 2, "toStepNo": 3, "condition": null},
      {"fromStepNo": 3, "toStepNo": 4, "condition": "검사 합격"}
    ],
    "exceptions": [
      {"fromStepNo": 3, "toStepNo": 2, "condition": "검사 불합격"}
    ]
  },
  "confidence": {
    "overall": 0.83,
    "summary": 0.90,
    "facts": 0.84,
    "workflow": 0.72
  },
  "uncertainties": [
    "배치만으로 actor를 추정한 단계가 있을 수 있음"
  ]
}
```

## 슬라이드 유형별 출력 정책

| 슬라이드 유형 | 필수 출력 | 선택 출력 |
|---|---|---|
| 일반 텍스트 | `summary`, `facts`, `searchableParaphrases` | `uncertainties` |
| 표/비교 | `summary`, `facts` | `keywords` |
| 수치/차트 | `summary`, `facts`, `uncertainties` | `searchableParaphrases` |
| 프로세스 | `summary`, `facts`, `workflow` | `uncertainties` |
| 조직도 | `summary`, `facts` | `relationship graph` |

## Prompt 전략

### System Prompt 원칙

- 원문에 없는 내용을 생성하지 말 것
- 불명확하면 `uncertainties`에 명시할 것
- `stepNo`는 추정 순서가 아니라 근거 기반 순서만 반환할 것
- `facts`는 검색 가능한 짧은 문장으로 반환할 것
- **출력 언어는 입력 문서의 언어와 일치**시켜야 한다. `detectedLanguage`를 따른다.

### 다국어 System Prompt 선택

```
detectedLanguage = 'ko' → 한국어 프롬프트
detectedLanguage = 'en' → 영어 프롬프트
detectedLanguage = 'ja' → 일본어 프롬프트
detectedLanguage = 'mixed' → 원문 언어 유지 프롬프트
```

각 언어의 system prompt 핵심 지시:
- ko: `"모든 출력을 한국어로 작성하세요. 원문에 없는 내용을 생성하지 마세요."`
- en: `"Write all output in English. Do not generate content not present in the source."`
- 혼합: `"원문의 각 텍스트 언어를 그대로 유지하세요. 번역하지 마세요."`

### Extraction Prompt 패턴

1. 슬라이드 유형 추정 (`workflowSignals` 포함 여부도 힌트로 사용)
2. 요약 생성 (문서 언어로)
3. facts 생성
4. **워크플로 신호가 있거나 유형이 WORKFLOW이면:** 맥락 추론 포함 workflow 생성
5. confidence와 uncertainties 출력

### 워크플로 맥락 추론 지시 (추가)

```
WORKFLOW 유형 또는 workflowSignals 존재 시 추가 지시:
"워크플로 단계를 추출할 때 다음을 반드시 포함하세요:
 - 각 단계의 선행 조건과 완료 조건
 - 담당 주체(actor)가 명시되지 않아도 문맥에서 추론 가능하면 표기 (uncertainties에 명시)
 - 예외 흐름(실패, 반려, 루프백)
 - 멀티페이지 연속 가능성이 있으면 continuedFrom/continuedTo 표시"
```

### 금지 규칙

- 숫자 상상 금지
- actor 없는 step에 actor 임의 대입 금지 (단, 문맥 추론은 uncertainties에 명시 후 허용)
- 화살표 방향이 불명확하면 edge 생성 보류
- 제목만 보고 전체 의미 단정 금지
- **원문이 한국어인데 영어로 출력하는 것 금지 (언어 일관성)**

## 2단계 추출 전략

정확도를 위해 한 번에 모든 걸 뽑지 않고 2단계로 나누는 구성이 안전하다.

### 단계 1. Slide Type Classification

- `text-heavy`
- `table`
- `chart`
- `workflow`
- `org-chart`
- `mixed`

### 단계 2. Type-specific Extraction

- 일반형 프롬프트
- 프로세스형 프롬프트
- 표/차트형 프롬프트

이 구조가 좋은 이유는 `workflow`와 `general summary`를 한 프롬프트에서 같이 처리할 때 오류 전이가 커지기 때문이다.

## Fallback 규칙

### 1. 일반 추출 실패

- raw chunk만 적재
- summary는 rule-based 조합 사용
- 운영자에게 low-confidence 표시

### 2. workflow 추출 실패

- `workflow`는 비워 둔다
- facts와 summary만 적재
- workflow 질의 대응 우선순위에서 제외한다

### 3. JSON 파싱 실패

- 재시도 1회
- 그래도 실패하면 low-confidence raw-only 처리

## Provenance 정책

LLM 출력의 각 fact나 step은 가능하면 원천 텍스트 참조를 가진다.

예:

```json
{
  "text": "품질 검사는 생산 계획 이후 수행된다",
  "evidenceTexts": ["생산 계획", "품질 검사"]
}
```

이 구조가 있어야 나중에 `왜 이렇게 답했는가`를 추적할 수 있다.

## 적재 정책

### Document Meta Chunk

- 문서 버전당 1개
- 임베딩 텍스트는 다음 항목을 줄바꿈으로 합쳐 구성
  - `documentName`
  - `totalSlideCount`
  - `documentSummary`
  - `categoryFunctionLabels`, `categoryIndustryLabels`, `categoryDocTypeLabel`
  - `documentKeywords`
- 검색 시 `chunkType = DOC_META` 필터로 우선 매칭하고, 발견형/카테고리 질의에 사용

### Summary Chunk

- `summary` 그대로 적재
- 인포그래픽 개요 검색에 사용

### Fact Chunk

- `facts[].text` 단위로 적재
- 비교, 수치, 규칙 질의에 사용

### Workflow Chunk

- step별 문장과 edge별 문장을 별도 적재
- 예:
  - `주문 접수 다음 단계는 생산 계획이다`
  - `품질 검사가 합격이면 출하 승인으로 진행된다`

## 문서 단위 LLM 추출

슬라이드 단위 추출이 끝난 후, 문서 전체 컨텍스트에 대해 한 번 더 LLM을 호출한다.

### 입력

- 사용자가 업로드 시 지정한 `userCategories` (function/industry/docType)
- 슬라이드별 `slideType`, `summary`, `keywords` 모음
- 문서명, 총 페이지 수

### 출력

```json
{
  "documentSummary": "주문 접수부터 출하 승인까지의 단계와 품질 검사 분기 조건을 정리한 업무 매뉴얼.",
  "documentKeywords": ["출하 승인", "품질 검사", "재작업"],
  "suggestedCategories": {
    "function": ["QUALITY_MANAGEMENT", "SHIPPING_MANAGEMENT"],
    "industry": ["MANUFACTURING"],
    "docType": "OPERATION_MANUAL"
  },
  "categoryConfidence": {
    "function": 0.86,
    "industry": 0.91,
    "docType": 0.78
  },
  "languageMix": {"ko": 0.92, "en": 0.08}
}
```

### 적재 규칙

- `userCategories`와 `suggestedCategories`를 비교
- 불일치/누락 항목은 `category_review` 큐로 적재
- 운영자가 승인 후에 정식 카테고리로 확정
- 카테고리는 `document_categories` 테이블에 매핑되며, 이후 모든 chunk 메타에 복제된다

## 임베딩 모델 고정

| 항목 | 값 |
|---|---|
| 모델 | `BAAI/bge-m3` |
| 차원 | 1024 |
| 거리 | cosine |
| 입력 길이 제한 | 8192 token |
| 정규화 | L2 normalize |
| 운영 위치 | 내부 추론 서버 또는 HuggingFace TEI |

LLM 출력은 영향 받지 않지만, chunk 텍스트는 모두 `bge-m3` 토크나이저 기준으로 잘라서 적재한다.

## 운영 버전 관리

- `prompt_version`
- `schema_version`
- `model_name` (LLM)
- `model_revision`
- `embedding_model = bge-m3`
- `embedding_revision`

위 값을 `document_version` 또는 `ingestion_job`과 함께 기록해야 재처리 기준이 선다.

## 설계 결론

정확도를 높이려면 `LLM이 PPTX를 해석한다`는 사고보다 `구조화된 입력에 대해 제한된 의미 추출을 수행한다`는 사고가 맞다.  
특히 프로세스 슬라이드는 자유 요약보다 `workflow schema` 강제가 훨씬 중요하다.

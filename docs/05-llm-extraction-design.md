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

### Extraction Prompt 패턴

1. 슬라이드 유형 추정
2. 요약 생성
3. facts 생성
4. 프로세스라면 workflow 생성
5. confidence와 uncertainties 출력

### 금지 규칙

- 숫자 상상 금지
- actor 없는 step에 actor 임의 대입 금지
- 화살표 방향이 불명확하면 edge 생성 보류
- 제목만 보고 전체 의미 단정 금지

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

## 운영 버전 관리

- `prompt_version`
- `schema_version`
- `model_name`
- `model_revision`

위 값을 `document_version` 또는 `ingestion_job`과 함께 기록해야 재처리 기준이 선다.

## 설계 결론

정확도를 높이려면 `LLM이 PPTX를 해석한다`는 사고보다 `구조화된 입력에 대해 제한된 의미 추출을 수행한다`는 사고가 맞다.  
특히 프로세스 슬라이드는 자유 요약보다 `workflow schema` 강제가 훨씬 중요하다.

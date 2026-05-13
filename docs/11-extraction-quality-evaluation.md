# 추출 품질 평가 설계

## 목표

PPTX 추출 품질을 감각적으로 판단하지 않고 측정 가능한 지표로 관리한다.

## 평가 계층

품질 평가는 아래 4계층으로 나눈다.

1. 원천 추출 정확도
2. 의미 구조화 정확도
3. 검색 정확도
4. 최종 답변 근거성

## 1. 원천 추출 정확도

`Apache POI`, OCR, 차트 레이블 추출의 품질을 본다.

### 측정 항목

| 항목 | 설명 |
|---|---|
| text recall | 슬라이드에 있는 주요 텍스트를 얼마나 놓치지 않았는가 |
| title accuracy | 제목을 맞게 추출했는가 |
| table cell accuracy | 표 셀 텍스트 정확도 |
| notes extraction accuracy | 발표자 노트 추출 정확도 |
| ocr usable rate | OCR 결과가 검색 가능한 수준인가 |

## 2. 의미 구조화 정확도

LLM이 summary, facts, workflow를 얼마나 맞게 만들었는지 본다.

### 일반 슬라이드 지표

| 지표 | 설명 |
|---|---|
| summary faithfulness | summary가 원문에 충실한가 |
| fact precision | facts 중 허위 정보 비율이 낮은가 |
| fact recall | 중요한 정보가 빠지지 않았는가 |
| paraphrase usefulness | 실제 질의 표현과 맞닿아 있는가 |

### 프로세스 슬라이드 지표

| 지표 | 설명 |
|---|---|
| actor accuracy | 담당자 추출 정확도 |
| step order accuracy | 단계 순서 정확도 |
| edge accuracy | 연결 관계 정확도 |
| condition accuracy | 조건 문구 정확도 |
| exception path accuracy | 예외 흐름 정확도 |

## 3. 검색 정확도

Chroma에 적재된 결과가 실제로 관련 슬라이드를 찾아오는지 본다.

### 지표

| 지표 | 설명 |
|---|---|
| top-1 slide hit | 정답 슬라이드가 1위인가 |
| top-3 slide hit | 정답 슬라이드가 상위 3개 안에 있는가 |
| top-5 evidence recall | 정답 근거가 상위 5개 안에 있는가 |
| workflow query hit rate | 프로세스 질의에서 workflow chunk가 기여하는가 |
| irrelevant evidence rate | 무관한 슬라이드가 근거로 뜨는 비율 |

## 4. 최종 답변 근거성

답변 자체보다 `근거에 묶여 있는가`를 본다.

### 지표

| 지표 | 설명 |
|---|---|
| grounded answer rate | evidences만으로 답변을 설명할 수 있는 비율 |
| hallucination rate | 근거 없는 내용을 답하는 비율 |
| page citation accuracy | 문서명/슬라이드 번호 인용 정확도 |
| abstention quality | 답을 못 찾을 때 제대로 보류하는가 |

## 평가 데이터셋 설계

## A. Gold PPTX Set

최소 20~30개 PPTX를 유형별로 확보한다.

- 텍스트 중심
- 표 중심
- 차트 중심
- 인포그래픽 중심
- 워킹 프로세스 중심
- 혼합형

## B. Slide Annotation Set

슬라이드별로 사람이 정답 데이터를 만든다.

- title
- 핵심 facts
- expected summary
- workflow actors
- workflow steps
- workflow edges
- exceptions
- answerable queries
- expected slideNo

## C. Query Set

질의 세트는 유형별로 분리한다.

- 개요 질의
- 비교 질의
- 수치 질의
- 프로세스 순서 질의
- 역할 질의
- 조건 질의
- 예외 질의

## 판정 기준 예시

### 일반 슬라이드

- `summary faithfulness >= 0.9`
- `fact precision >= 0.95`
- `top-3 slide hit >= 0.9`

### 프로세스 슬라이드

- `step order accuracy >= 0.9`
- `actor accuracy >= 0.9`
- `exception path accuracy >= 0.8`
- `workflow query hit rate >= 0.85`

## 오차 유형 분류

오류는 단순 실패로 보지 말고 원인별로 분류해야 한다.

| 오차 유형 | 설명 |
|---|---|
| source-miss | 원천 텍스트 추출 누락 |
| ocr-noise | OCR 오탐/누락 |
| type-misclassification | 슬라이드 유형 분류 오류 |
| summary-hallucination | 요약 환각 |
| fact-loss | 중요한 사실 누락 |
| workflow-order-error | 단계 순서 오류 |
| workflow-actor-error | 담당자 오류 |
| workflow-edge-error | 연결 관계 오류 |
| retrieval-miss | 벡터 검색 실패 |
| grounding-fail | 근거는 있는데 답변이 벗어남 |

## 실험 축

설계 단계에서 최소 아래 조합을 비교할 수 있어야 한다.

| 축 | 비교안 |
|---|---|
| chunk 전략 | raw only / raw+summary / raw+summary+workflow |
| OCR 사용 | off / selective / all |
| LLM 전략 | one-pass / two-pass |
| workflow 추출 | off / low-confidence on / strict on |
| reranking | none / slide-group / workflow-biased |

## 운영 모니터링 지표

실서비스에서는 아래 지표를 계속 본다.

- 평균 slide당 chunk 수
- low-confidence slide 비율
- workflow 추출 성공률
- query no-answer 비율
- top-3 evidence click-through rate
- 운영자 재처리 비율

## 사람 검수 루프

프로세스형 슬라이드는 초기에는 사람 검수 루프를 두는 편이 안전하다.

### 검수 대상

- low-confidence workflow
- 예외 경로가 있는 슬라이드
- 차트와 프로세스가 섞인 혼합형 슬라이드

### 검수 결과 활용

- prompt 개선
- visual type classifier 개선
- gold set 확장
- 추출 보류 규칙 조정

## 승인 기준

실서비스 확장 전 최소 기준을 권장한다.

1. 일반 슬라이드 top-3 hit 90% 이상
2. 프로세스 슬라이드 step order accuracy 90% 이상
3. grounded answer rate 95% 이상
4. page citation accuracy 99% 이상

## 설계 결론

이 시스템의 핵심 리스크는 벡터 DB가 아니라 `기준 데이터 추출 품질`이다.  
따라서 설계 단계부터 `평가셋`, `오차 분류`, `재처리 기준`, `prompt/model version 관리`가 함께 들어가야 한다.

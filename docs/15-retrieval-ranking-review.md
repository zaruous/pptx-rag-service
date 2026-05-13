# 검색, 재랭킹, 검수 설계

## 목표

샘플 PPTX 없이도 확정 가능한 질의 처리 규칙과 검수/재처리 운영 규칙을 고정한다.

## 1. 질의 분류 설계

### 분류 목적

- 적절한 chunk 유형 선택
- workflow bias 적용 여부 결정
- no-answer 기준 강화 여부 결정

### 질의 유형

| QueryType | 설명 | 예시 |
|---|---|---|
| `OVERVIEW` | 개요/설명 요청 | "이 PPT에서 생산 흐름을 어떻게 설명하나" |
| `FACT_LOOKUP` | 특정 사실 조회 | "출하 승인 주체는 누구인가" |
| `COMPARISON` | 항목 비교 | "A와 B의 차이는 무엇인가" |
| `NUMERIC` | 수치/최댓값/비율 | "가장 높은 성장률은 무엇인가" |
| `WORKFLOW_SEQUENCE` | 순서/다음 단계 | "검사 다음 단계는 무엇인가" |
| `WORKFLOW_ACTOR` | 담당자/부서 | "품질팀은 어디서 개입하나" |
| `WORKFLOW_CONDITION` | 조건/분기 | "어떤 경우 승인 단계로 가나" |
| `WORKFLOW_EXCEPTION` | 예외/반려/루프백 | "실패하면 어디로 돌아가나" |
| `DOCUMENT_DISCOVERY` | 문서 존재 여부 탐색 | "출하 프로세스를 다루는 문서가 있나" |

### 분류 방식

초기 설계는 `규칙 기반 1차 + LLM 보조 2차`가 적절하다.

#### 1차 규칙 기반

- `다음`, `이전`, `순서`, `절차` -> `WORKFLOW_SEQUENCE`
- `누가`, `담당`, `부서`, `팀` -> `WORKFLOW_ACTOR`
- `조건`, `경우`, `승인되면` -> `WORKFLOW_CONDITION`
- `반려`, `실패`, `예외`, `되돌아` -> `WORKFLOW_EXCEPTION`
- `비교`, `차이` -> `COMPARISON`
- `몇`, `가장`, `%`, `증가율` -> `NUMERIC`

#### 2차 LLM 보조

- 1차 confidence가 낮을 때만 호출
- 질의가 혼합형일 때 `primary query type` 하나만 선택

## 2. 검색 계획

### QueryType별 우선 chunk

| QueryType | 1순위 | 2순위 | 3순위 |
|---|---|---|---|
| `OVERVIEW` | `SUMMARY` | `WORKFLOW_STEP` | `FACT` |
| `FACT_LOOKUP` | `FACT` | `RAW` | `SUMMARY` |
| `COMPARISON` | `FACT` | `SUMMARY` | `RAW` |
| `NUMERIC` | `FACT` | `RAW` | `SUMMARY` |
| `WORKFLOW_SEQUENCE` | `WORKFLOW_STEP` | `WORKFLOW_EDGE` | `SUMMARY` |
| `WORKFLOW_ACTOR` | `WORKFLOW_STEP` | `FACT` | `SUMMARY` |
| `WORKFLOW_CONDITION` | `WORKFLOW_EDGE` | `WORKFLOW_EXCEPTION` | `FACT` |
| `WORKFLOW_EXCEPTION` | `WORKFLOW_EXCEPTION` | `WORKFLOW_EDGE` | `SUMMARY` |
| `DOCUMENT_DISCOVERY` | `SUMMARY` | `FACT` | `WORKFLOW_STEP` |

### 기본 retrieval 단계

1. workspace/document scope 필터
2. query type 판정
3. 우선 chunk type별 top-k 검색
4. slide 단위 그룹화
5. slide-level 재랭킹
6. evidence 선택
7. no-answer 판정
8. grounded answer 생성

## 3. 점수 모델

### Chunk-level score

초기 점수식은 단순 가중합으로 시작하는 편이 안전하다.

```text
chunkScore
= 0.55 * vectorSimilarity
+ 0.15 * chunkTypeBoost
+ 0.10 * confidenceBoost
+ 0.10 * queryKeywordMatch
+ 0.10 * provenanceQuality
- 0.15 * lowConfidencePenalty
```

### ChunkType boost 예시

| 상황 | boost |
|---|---|
| workflow 질의 + workflow chunk | `+1.0` |
| numeric 질의 + fact chunk | `+0.8` |
| overview 질의 + summary chunk | `+0.7` |
| workflow 질의 + raw chunk | `+0.2` |

### Slide-level score

```text
slideScore
= 0.50 * topChunkScore
+ 0.20 * supportingChunkDiversity
+ 0.15 * sameSlideEvidenceCount
+ 0.10 * slideConfidence
+ 0.05 * reviewApprovalBoost
```

### Penalty 규칙

- `reviewStatus = REJECTED` 이면 제외
- `workflow < 0.60` 이면 workflow chunk 제외
- `overall < 0.70` 이면 top result라도 no-answer 후보 검토

## 4. Evidence 선택 규칙

### 기본 규칙

- 최종 evidences는 기본 3개, 최대 5개
- 동일 slide에서 과도한 chunk 중복 노출 금지
- `SUMMARY + FACT` 또는 `WORKFLOW_STEP + WORKFLOW_EDGE` 조합 우선

### 우선순위

1. 답변 핵심을 직접 지지하는 chunk
2. 같은 slide의 보조 chunk
3. 다른 slide의 corroborating chunk

## 5. No-Answer 정책

### no-answer를 반환해야 하는 경우

- top slide score가 최소 기준 이하
- evidence끼리 서로 충돌
- 핵심 actor/step/condition이 비어 있음
- workflow 질의인데 workflow chunk가 모두 low-confidence

### 기본 기준값

| 항목 | 기준 |
|---|---|
| top chunk similarity | `< 0.55` 면 위험 |
| top slide score | `< 0.62` 면 no-answer 검토 |
| evidence count | `< 1` 면 no-answer |
| grounded confidence | `< 0.70` 면 보수 응답 |

### 응답 정책

- 추정 답변 대신 `관련 슬라이드를 충분히 특정하지 못했다` 형태로 응답
- 필요 시 가장 가까운 문서명/슬라이드 번호만 보조 후보로 노출

## 6. Workflow 질의 특수 규칙

### 순서 질의

- `WORKFLOW_STEP`와 `WORKFLOW_EDGE`를 함께 확인
- step 이름만 비슷하고 edge가 없으면 답변 보류 가능

### actor 질의

- actor 없는 step은 직접 답 근거로 쓰지 않는다
- actor는 summary 추론보다 workflow step 명시값 우선

### condition/exception 질의

- edge 또는 exception evidence가 없으면 답변하지 않는다
- notes 기반 조건은 `보조 근거`로만 사용

## 7. 검수 큐 규칙

### 자동 검수 대상

- `overall < 0.70`
- `workflow`가 존재하지만 `workflow < 0.75`
- exception edge가 있는데 confidence 낮음
- actor가 3개 이상인데 lane 정보가 빈약함
- OCR 비중이 높은 슬라이드

### 검수 우선순위

| 우선순위 | 조건 |
|---|---|
| 높음 | workflow low-confidence + 실사용 질의 노출 |
| 중간 | numeric/fact conflict 발생 |
| 낮음 | overview summary만 낮은 confidence |

## 8. 재처리 정책

### 재처리 트리거

- prompt version 변경
- schema version 변경
- OCR 엔진 변경
- 모델 교체
- 검수 반려
- retrieval miss 반복 발생

### 재처리 범위

| 범위 | 조건 |
|---|---|
| slide 단위 | 특정 슬라이드만 추출 오류 |
| document version 단위 | 문서 전반 prompt 변경 |
| workspace 단위 | 모델 전면 교체 |

## 9. 운영자 수동 수정 정책

초기에는 완전 수동 수정 대신 `검수 + 재처리 요청` 중심이 낫다.

### 허용할 수동 조치

- slide type 수정
- workflow 여부 플래그 수정
- 특정 chunk 비노출 처리
- 문서 비활성화

### 지양할 수동 조치

- 임의 summary 본문 직접 편집
- vector text 직접 수정
- edge를 운영자가 임의로 대량 입력

직접 편집을 열면 기준 데이터 관리가 빠르게 불안정해진다.

## 10. 질문 로그 활용

질문 로그는 검색 품질 개선의 핵심 입력이다.

- no-answer가 많았던 질의 유형 집계
- workflow query miss 비율 추적
- 자주 클릭된 evidence 패턴 분석
- 재처리 후 hit rate 개선 여부 비교

## 11. 설계 결론

샘플 PPTX가 없어도 retrieval와 검수 정책은 상당 부분 선결정할 수 있다.  
핵심은 `질의 타입별 우선 chunk`, `no-answer 기준`, `low-confidence 검수 큐`, `재처리 트리거`를 먼저 고정하는 것이다.

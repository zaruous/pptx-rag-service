# 애플리케이션 아키텍처

## 목표

구현 직전 단계에서 `Spring Boot + Gradle` 기준의 모듈 책임, 계층 경계, 배치 흐름, 외부 연동 경계를 고정한다.

## 상위 원칙

1. `업로드/처리/검색/검수`를 명확히 분리한다.
2. 외부 의존성은 모두 adapter 뒤로 숨긴다.
3. retrieval는 동기 경로, ingestion은 비동기 경로로 나눈다.
4. domain model과 vendor-specific model을 섞지 않는다.
5. review는 운영 보조 기능이 아니라 품질 제어의 핵심 축으로 본다.

## 권장 패키지 구조

```text
com.example.pptxrag
├─ common
├─ document
├─ ingestion
├─ extraction
├─ retrieval
├─ review
├─ admin
├─ integration
└─ config
```

## 1. `common`

### 책임

- 공통 예외
- 공통 응답 모델
- ID 생성 규칙
- 시간/해시/언어 유틸
- 이벤트 베이스 타입

### 두면 안 되는 것

- 업무 규칙
- vendor 종속 코드

## 2. `document`

### 책임

- workspace/document/documentVersion/slide의 생명주기 관리
- 업로드 메타데이터 저장
- 문서 조회 모델 제공

### 주요 하위 구성

```text
document
├─ api
├─ application
├─ domain
├─ repository
└─ persistence
```

### 핵심 서비스

- `DocumentCommandService`
- `DocumentQueryService`
- `DocumentVersionService`

## 3. `ingestion`

### 책임

- 업로드 후 비동기 인덱싱 오케스트레이션
- job 생성, 상태 전이, 재시도, 부분 실패 관리

### 주요 하위 구성

```text
ingestion
├─ api
├─ application
├─ domain
├─ scheduler
└─ persistence
```

### 핵심 서비스

- `IngestionJobService`
- `IngestionOrchestrator`
- `IngestionRetryService`

### 오케스트레이션 단계

1. 파일 저장 확인
2. slide parsing
3. slide IR 생성
4. semantic extraction
5. chunk assembly
6. embedding
7. Chroma upsert
8. review candidate 생성
9. job 완료/실패 기록

## 4. `extraction`

### 책임

- PPTX에서 기준 데이터를 추출하고 의미 구조화 결과를 만든다.

### 주요 하위 구성

```text
extraction
├─ application
├─ domain
├─ parser
├─ semantic
├─ workflow
└─ chunking
```

### 세부 책임 분리

- `parser`: Apache POI 기반 원천 추출
- `semantic`: LLM 입력/출력 처리
- `workflow`: 프로세스형 슬라이드 전용 규칙
- `chunking`: raw/summary/fact/workflow chunk 조립

### 핵심 서비스

- `SlideIrBuilder`
- `SlideTypeClassifier`
- `SemanticExtractionService`
- `WorkflowNormalizationService`
- `ChunkAssemblyService`

## 5. `retrieval`

### 책임

- 질의 분류
- 벡터 검색
- slide 그룹화
- reranking
- no-answer 판정
- grounded answer 조립

### 주요 하위 구성

```text
retrieval
├─ api
├─ application
├─ domain
├─ ranking
└─ querylog
```

### 핵심 서비스

- `QueryClassificationService`
- `ChunkRetrievalService`
- `SlideRerankingService`
- `EvidenceSelectionService`
- `GroundedAnswerService`
- `NoAnswerPolicyService`

## 6. `review`

### 책임

- low-confidence 슬라이드와 workflow 검수 큐 관리
- 검수 승인/반려/재처리 요청 처리
- 검수 결과를 retrieval 필터에 반영

### 주요 하위 구성

```text
review
├─ api
├─ application
├─ domain
└─ persistence
```

### 핵심 서비스

- `ReviewQueueService`
- `ReviewDecisionService`
- `ReviewPolicyService`

## 7. `admin`

### 책임

- 인덱스 purge
- 문서 비활성화
- 운영 통계 조회
- 설정 버전 관리

## 8. `integration`

### 책임

- 외부 시스템 adapter 보관

### 권장 하위 구성

```text
integration
├─ chroma
├─ embedding
├─ llm
├─ ocr
└─ storage
```

### 핵심 원칙

- domain은 `integration`을 직접 참조하지 않는다.
- application service가 port interface를 의존하고 adapter가 이를 구현한다.

## Port/Adapter 경계

### Outbound Ports 예시

- `SlideParserPort`
- `OcrPort`
- `SemanticExtractorPort`
- `EmbeddingPort`
- `VectorStorePort`
- `FileStoragePort`

### Inbound Ports 예시

- `UploadDocumentUseCase`
- `RunQueryUseCase`
- `ReviewSlideUseCase`
- `ReingestSlideUseCase`

## 동기/비동기 경계

### 동기 처리

- 문서 업로드 접수
- 문서/슬라이드 조회
- 질의 실행
- 검수 결과 저장

### 비동기 처리

- 인덱싱
- OCR
- semantic extraction
- embedding
- purge/rebuild

## 이벤트 설계

### 내부 이벤트 예시

- `DocumentUploaded`
- `IngestionQueued`
- `SlideParsed`
- `SemanticExtractionCompleted`
- `ReviewCandidateCreated`
- `ReviewDecisionRecorded`
- `ReingestionRequested`

### 이벤트 사용 목적

- 모듈 간 결합 완화
- review queue 생성 자동화
- 운영 통계 집계

## 배치 단위 권장

### 1. Document-level job

- 문서 전체 재처리
- prompt/model 변경 시 사용

### 2. Slide-level job

- 특정 슬라이드만 재처리
- 검수 반려 대응에 유리

### 3. Chunk-level 재적재

- 원칙적으로 직접 노출하지 않는다
- 운영상 복잡도만 높고 정합성 리스크가 크다

## 저장 경계

### RDB

- document metadata
- slide metadata
- workflow nodes/edges
- ingestion jobs
- review decisions
- query logs

### Vector Store

- 최종 retrieval chunk

### File Storage

- source pptx
- slide thumbnail
- OCR intermediate images

## 트랜잭션 원칙

- RDB와 Chroma를 하나의 분산 트랜잭션으로 보지 않는다.
- RDB를 system of record로 본다.
- Chroma 실패 시 `PARTIAL_FAILED`로 기록하고 재시도한다.

## 장애 격리 원칙

- OCR 실패가 전체 업로드 실패로 바로 번지지 않게 한다.
- workflow 추출 실패는 summary/fact 적재를 막지 않는다.
- review 모듈 장애가 retrieval 기본 경로를 막지 않게 한다.

## 설계 결론

구현 시 복잡도가 올라가는 지점은 `retrieval`보다 `ingestion + extraction + review`의 경계다.  
따라서 코드 구조도 이 세 축을 먼저 분리하는 것이 맞다.

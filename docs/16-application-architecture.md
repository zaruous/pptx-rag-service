# 애플리케이션 아키텍처

## 목표

구현 직전 단계에서 다음을 고정한다.

- 서버: `Spring Boot (Java) + Gradle` 모듈 책임, 계층 경계, 배치 흐름, 외부 연동 경계
- 클라이언트: `React + Vite (TypeScript)` SPA 모듈, 라우팅, 상태 관리, 시각화 어댑터
- 임베딩: `BAAI/bge-m3` 클라이언트 단일화 (차원 1024, cosine)
- 벡터 저장소: `Chroma` 컬렉션 분리 (`doc_meta`, `slide_chunks`)

## 전체 토폴로지

```text
+---------------------------+         +---------------------------+
|  React + Vite SPA         |  HTTPS  |  Spring Boot REST API     |
|  - 업로드(카테고리 선택)  | ----->  |  - upload/category        |
|  - 검색/질의              |         |  - query/retrieval        |
|  - 대시보드 인포그래픽    | <-----  |  - dashboard aggregate    |
|  - 검수 큐                |         |  - admin/review           |
+---------------------------+         +-----+----------------+----+
                                            |                |
                                            v                v
                                     +------+------+   +-----+------+
                                     |  RDB        |   |  Object    |
                                     | (Postgres)  |   |  Storage   |
                                     +------+------+   +------------+
                                            |
                                            v
                                     +-------------+   +---------------+
                                     | Ingestion   |-->| bge-m3        |
                                     | Worker      |   | Embedding API |
                                     +------+------+   +-------+-------+
                                            |                  |
                                            v                  v
                                     +-------------+   +---------------+
                                     | Chroma      |<--| Vector Upsert |
                                     | doc_meta /  |   +---------------+
                                     | slide_chunks|
                                     +-------------+
```

## 상위 원칙

1. `업로드/처리/검색/검수/대시보드`를 명확히 분리한다.
2. 외부 의존성(LLM, Embedding, Chroma, OCR, Storage)은 모두 adapter 뒤로 숨긴다.
3. retrieval는 동기 경로, ingestion은 비동기 경로로 나눈다.
4. domain model과 vendor-specific model을 섞지 않는다.
5. review는 운영 보조 기능이 아니라 품질 제어의 핵심 축으로 본다.
6. 카테고리(`기능 / 업종 / 유형`)는 도메인 1급 객체로 두고, 모든 chunk 메타에 평탄화 복제한다.
7. 임베딩 모델은 `bge-m3`로 단일화하고, 차원/거리 정책은 설정으로만 노출한다.
8. 클라이언트는 SPA 단일 번들이고, 정적 자원은 서버 또는 별도 CDN으로 배포한다.

## 권장 패키지 구조 (서버)

```text
com.example.pptxrag
├─ common
├─ document
├─ category        # 카테고리 taxonomy / 정규화 / 매핑
├─ ingestion
├─ extraction
├─ retrieval
├─ dashboard       # 대시보드 집계 / 스냅샷
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

1. 파일 저장 확인 + 포맷 감지 (`DocumentFormatDetector`)
2. `DocumentParserRegistry` → 포맷에 맞는 파서 선택
3. `DocumentParser.parse()` → `DocumentParseResult` (페이지 목록)
4. 디버그 JSON 저장 (설정에 따라)
5. `SlideIrBuilder` → `PageParseResult` → `SlideIr` 변환
6. semantic extraction (LLM)
7. 문서 단위 요약 + 카테고리 추천
8. chunk assembly (DOC_META 포함)
9. embedding (bge-m3)
10. Chroma upsert
11. review candidate 생성
12. job 완료/실패 기록

## 4. `extraction`

### 책임

- PPTX / PDF / DOCX 등 다양한 포맷에서 기준 데이터를 추출하고 의미 구조화 결과를 만든다.
- `DocumentParser` 인터페이스를 통해 포맷별 파서를 교체 가능하게 유지한다.
- 추출 결과를 마크다운 디버그 뷰로 제공한다.

### 주요 하위 구성

```text
extraction
├─ application
├─ domain
├─ parser
│   ├─ DocumentParser.java         (interface)
│   ├─ AbstractDocumentParser.java (abstract)
│   ├─ DocumentParserRegistry.java
│   ├─ DocumentFormatDetector.java
│   ├─ pptx/PptxDocumentParser.java
│   ├─ pdf/PdfDocumentParser.java
│   ├─ pdf/PdfScanDetector.java
│   ├─ docx/DocxDocumentParser.java
│   └─ docx/DocxPageSplitter.java
├─ debug
│   └─ DebugPreviewRenderer.java   (PageParseResult → Markdown)
├─ semantic
├─ workflow
└─ chunking
```

### 세부 책임 분리

- `parser`: 포맷별 파서 구현 + 레지스트리. 출력은 항상 `DocumentParseResult`
- `debug`: `PageParseResult`를 마크다운으로 렌더링, `/debug/preview` API 지원
- `semantic`: LLM 입력/출력 처리 (SlideIr 기반, 포맷 무관)
- `workflow`: 프로세스형 페이지 전용 규칙
- `chunking`: raw/summary/fact/workflow/doc_meta chunk 조립

### 핵심 서비스

- `DocumentFormatDetector`
- `DocumentParserRegistry`
- `PptxDocumentParser`, `PdfDocumentParser`, `DocxDocumentParser`
- `SlideIrBuilder`
- `SlideTypeClassifier`
- `SemanticExtractionService`
- `DocumentSemanticExtractor`
- `WorkflowNormalizationService`
- `ChunkAssemblyService`
- `DebugPreviewRenderer`

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

## 7-A. `category`

### 책임

- 카테고리 taxonomy(`FUNCTION / INDUSTRY / DOC_TYPE`) 마스터 관리
- 사용자 입력 / 자유 태그 / LLM 추천을 정규화해 taxonomy 코드로 매핑
- 카테고리 검수 큐 적재 (사용자 입력과 LLM 추천 불일치 시)
- 모든 chunk 메타에 카테고리 라벨 복제 시 단일 소스 보장

### 핵심 서비스

- `CategoryTaxonomyService`
- `CategoryNormalizationService`
- `CategoryReviewService`

## 7-B. `dashboard`

### 책임

- 인덱싱 이벤트 수신 → 집계 캐시 갱신
- 카테고리 분포, 상태 카운트, 업로드 타임라인, 최근 문서 스냅샷 제공
- SSE 또는 polling으로 클라이언트 실시간 갱신 지원

### 핵심 서비스

- `DashboardAggregationService`
- `DashboardSnapshotRepository`
- `DashboardEventListener` (Spring `@EventListener`)

## 8. `integration`

### 책임

- 외부 시스템 adapter 보관

### 권장 하위 구성

```text
integration
├─ chroma
├─ embedding        # bge-m3 어댑터
├─ llm
├─ ocr              # Tesseract / 외부 OCR API 어댑터
├─ libreoffice      # DOCX 실제 페이지 분할 (선택)
└─ storage
```

### bge-m3 클라이언트 표준

- 운영 모드: HuggingFace TEI(Text Embeddings Inference) 또는 사내 GPU 서버
- 입력: 단일/배치 텍스트, 최대 8192 token
- 출력: 1024차원 float32 벡터, L2 normalize 옵션
- 어댑터 설정: `app.embedding.model=bge-m3`, `app.embedding.dim=1024`, `app.embedding.normalize=true`, `app.embedding.distance=cosine`
- 장애 시 재시도 정책: exponential backoff, dead-letter는 `ingestion_jobs.error_code` 기록

### 핵심 원칙

- domain은 `integration`을 직접 참조하지 않는다.
- application service가 port interface를 의존하고 adapter가 이를 구현한다.

## Port/Adapter 경계

### Outbound Ports 예시

- `DocumentParserPort` (포맷별 파서 어댑터)
- `OcrPort`
- `SemanticExtractorPort`
- `EmbeddingPort`
- `VectorStorePort`
- `FileStoragePort`
- `PageRenderPort` (DOCX 페이지 분할용 LibreOffice 어댑터)

### Inbound Ports 예시

- `UploadDocumentUseCase`
- `RunQueryUseCase`
- `GetDebugPreviewUseCase`
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

## 클라이언트 아키텍처 (React + Vite)

### 빌드 구성

- 빌드: `Vite (TypeScript)` + `react-swc` 플러그인
- 패키지 매니저: `pnpm` 또는 `npm`
- 번들 산출물(`dist/`)을 Spring Boot 정적 리소스(`/static`)로 묶거나 별도 Nginx로 서빙
- 개발 모드: Vite dev server, `server.proxy`로 `/api` → Spring 백엔드 프록시

### 디렉터리 구조

```text
web/
├─ index.html
├─ vite.config.ts
├─ tsconfig.json
└─ src/
   ├─ main.tsx
   ├─ app/                # 라우팅, providers, error boundary
   ├─ pages/
   │   ├─ DashboardPage.tsx     # 메인 인포그래픽
   │   ├─ UploadPage.tsx        # 업로드 + 카테고리 선택
   │   ├─ DocumentsPage.tsx     # 문서 목록 (카테고리 필터)
   │   ├─ QueryPage.tsx         # 질의 응답 + 근거 카드
   │   └─ ReviewPage.tsx        # 검수 큐
   ├─ features/
   │   ├─ upload/               # 업로드 훅, 진행률, 카테고리 셀렉터
   │   ├─ categories/           # taxonomy 트리, chip 컴포넌트
   │   ├─ query/                # 질의 폼, evidence 표시
   │   └─ dashboard/            # 도넛, 스택드 바, 타임라인 차트
   ├─ shared/
   │   ├─ api/                  # OpenAPI 생성 클라이언트
   │   ├─ ui/                   # 디자인 시스템
   │   └─ hooks/
   └─ styles/
```

### 권장 라이브러리

| 영역 | 후보 |
|---|---|
| 라우팅 | `react-router-dom` |
| 서버 상태 | `@tanstack/react-query` |
| 폼/검증 | `react-hook-form` + `zod` |
| 차트 | `recharts` (기본), `echarts-for-react` (복잡 차트) |
| 디자인 시스템 | `shadcn/ui` 또는 사내 표준 |
| 파일 업로드 | `react-dropzone` |
| 테스트 | `vitest`, `@testing-library/react`, `playwright` |

### 클라이언트 모듈 책임

- `features/upload`: 파일 드래그앤드롭, 카테고리 다중 선택, 업로드 진행률, 잡 상태 추적
- `features/categories`: taxonomy 트리, 검색, chip 다중 선택, LLM 추천 미리보기
- `features/query`: 질의 입력, 카테고리 필터, evidence 슬라이드 카드, no-answer 응답 표시
- `features/dashboard`: 카테고리 분포, 인덱싱 상태, 업로드 타임라인 인포그래픽 (상세는 `19-dashboard-design.md` 참고)
- `features/review`: 운영자 검수 큐, 카테고리 충돌 검수 UI

### API 표준 (클라이언트 관점)

- 인증: `Authorization: Bearer <token>`
- 공통 응답: `{ "data": ..., "error": null | {...} }`
- 페이지네이션: `?page=&size=&sort=`
- 카테고리 필터: `?categoryFunction=A,B&categoryIndustry=MANUFACTURING&categoryDocType=PROPOSAL`
- 잡 상태 폴링: `react-query` `refetchInterval`, 또는 SSE `/api/jobs/{id}/events`

## 주요 API 엔드포인트 (서버↔클라이언트 계약)

| Method | Path | 설명 |
|---|---|---|
| `POST` | `/api/documents` | 문서 업로드 (PPTX/PDF/DOCX) + 카테고리 |
| `GET` | `/api/documents` | 카테고리/상태/포맷 필터 목록 |
| `GET` | `/api/documents/{id}` | 문서 상세 |
| `POST` | `/api/documents/{id}/reprocess` | 재처리 트리거 |
| `GET` | `/api/documents/{versionId}/debug/preview` | 추출 결과 디버그 뷰 (JSON) |
| `GET` | `/api/documents/{versionId}/debug/preview/{pageNo}/markdown` | 페이지 마크다운 미리보기 |
| `GET` | `/api/documents/{versionId}/debug/preview/markdown` | 전체 MD 다운로드 |
| `GET` | `/api/documents/{versionId}/debug/chunks` | Chunk 텍스트/토큰 목록 |
| `GET` | `/api/taxonomy` | 카테고리 마스터 조회 |
| `POST` | `/api/taxonomy` | 카테고리 추가 (관리자) |
| `POST` | `/api/query` | 자연어 질의 |
| `GET` | `/api/dashboard/overview` | 메인 인포그래픽 통합 데이터 |
| `GET` | `/api/dashboard/categories` | 카테고리 분포 |
| `GET` | `/api/dashboard/timeline` | 일자별 적재 추이 |
| `GET` | `/api/dashboard/status` | 인덱싱 상태 카운트 |
| `GET` | `/api/review/queue` | 검수 큐 |
| `POST` | `/api/review/{id}` | 검수 결정 |

## Spring 주요 설정

- `spring.servlet.multipart.max-file-size` 문서 업로드 한도 (기본 100MB)
- `spring.task.execution.pool.*` ingestion 비동기 실행기
- `app.embedding.model=bge-m3`, `app.embedding.dim=1024`, `app.embedding.distance=cosine`
- `app.chroma.collections.docMeta=doc_meta`, `app.chroma.collections.slideChunks=slide_chunks`
- `app.llm.provider`, `app.llm.model`, `app.llm.promptVersion`
- `app.taxonomy.version` 카테고리 마스터 버전

## 관측

- 로그: JSON 구조 로그, `correlationId`로 업로드↔ingestion↔chroma upsert 추적
- 메트릭: Micrometer + Prometheus
  - `ingestion.duration{stage=PARSE|EMBED|INDEX}`
  - `embedding.tokens.total`
  - `chroma.upsert.errors.total`
  - `dashboard.query.latency`
  - `category.normalize.fallback.total` (LLM 추천이 taxonomy에 매핑되지 못한 횟수)
- 트레이싱: OpenTelemetry로 Spring + Worker + bge-m3 호출 span

## 설계 결론

구현 시 복잡도가 올라가는 지점은 `retrieval`보다 `ingestion + extraction + review`의 경계이고,  
여기에 `category` 정규화와 `dashboard` 집계가 추가되면 모듈 간 이벤트 경로가 더 중요해진다.  
따라서 코드 구조도 이 다섯 축(`ingestion / extraction / review / category / dashboard`)을 먼저 분리하고,  
클라이언트는 SPA 단일 번들로 `대시보드 / 업로드 / 검색 / 검수` 화면을 명확히 갈라 두는 것이 맞다.

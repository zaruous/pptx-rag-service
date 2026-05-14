# 정보 구조 및 아키텍처

## 상위 구조

```text
[React + Vite SPA]
   |-- 업로드 화면 (카테고리 선택)
   |-- 검색/질의 화면
   |-- 대시보드 (카테고리 분포, 적재 추이, 상태 인포그래픽)
   |-- 운영자 검수 화면
   |
   v
[Spring Boot API (Java + Gradle)]
   |-- 업로드/카테고리 API
   |-- 질의 API
   |-- 대시보드 집계 API
   |-- 관리자/검수 API
   |
   |--> [RDB: 문서/버전/잡/카테고리/감사로그]
   |--> [파일 저장소: 원본 PPTX, 썸네일]
   |--> [Ingestion Worker]
   |        |--> Apache POI 파싱
   |        |--> OCR
   |        |--> Slide Summary 생성
   |        |--> 문서 메타 chunk 생성 (파일명/페이지수/요약/카테고리)
   |        |--> Embedding (bge-m3)
   |        |--> Chroma upsert
   |        |--> Dashboard 집계 이벤트 push
   |
   |--> [Chroma 컬렉션]
            |--> `doc_meta` (문서 단위 메타 임베딩)
            |--> `slide_chunks` (슬라이드 단위 chunk)
```

## 논리 엔티티

- Workspace: 문서 묶음 단위. 팀, 프로젝트, 고객사 기준으로 분리 가능
- Document: 사용자가 업로드한 논리 문서
- DocumentVersion: 실제 업로드 파일 단위
- DocumentCategory: 문서에 부여한 `기능 / 업종 / 유형` 다중 카테고리 매핑
- CategoryTaxonomy: 카테고리 정의 (axis = FUNCTION / INDUSTRY / DOC_TYPE)
- Slide: 문서 버전 내 개별 슬라이드
- SlideChunk: 검색 대상 텍스트 단위
- DocumentMetaChunk: 파일명·페이지수·요약·카테고리를 합친 문서 단위 임베딩 chunk
- WorkflowGraph: 프로세스형 슬라이드의 단계/연결 관계
- IngestionJob: 파싱/임베딩/적재 배치 실행 단위
- DashboardSnapshot: 대시보드 집계 결과 캐시 (카테고리 분포, 상태 카운트)
- QueryLog: 질의 및 반환 근거 기록

## 권장 저장 전략

### 1. RDB에 저장할 것

- 문서 메타데이터 (파일명, 페이지수, 업로더, 업로드 시각)
- 카테고리 분류 (`기능 / 업종 / 유형`) 및 분류 이력
- 카테고리 taxonomy 정의 마스터
- 파일 해시
- 업로드 이력
- 처리 상태 / 실패 원인
- 슬라이드 기본 메타정보
- 대시보드 집계 캐시 (snapshot)
- 질의 로그

### 2. Chroma에 저장할 것

- 문서 메타 chunk 임베딩 (`DOC_META`): 파일명 + 페이지수 + LLM 요약 + 카테고리 라벨
- slide summary chunk 임베딩
- slide detail chunk 임베딩
- OCR chunk 임베딩
- workflow chunk 임베딩
- retrieval용 메타데이터 (카테고리 포함)

### 3. 파일 저장소에 저장할 것

- 원본 PPTX
- 슬라이드 썸네일 이미지
- OCR 임시 이미지

## 청크 전략

### A. Document Meta Chunk

- 단위: 문서 버전당 1개
- 내용: `파일명`, `총 페이지 수`, `LLM 문서 요약`, `카테고리 라벨 (기능/업종/유형)`, `핵심 키워드`
- 용도: 문서 발견형 질의 (`출하 프로세스 다루는 문서 있나?`), 카테고리 기반 검색, 대시보드 검색

### B. Slide Summary Chunk

- 단위: 슬라이드당 1개
- 내용: 제목, 핵심 메시지, 도형/표/이미지 설명을 합친 요약 텍스트
- 용도: 맥락 검색, 인포그래픽 개요 검색

### C. Detail Chunk

- 단위: 슬라이드 내 텍스트 블록, 표, 노트, 섹션별 분할
- 용도: 정밀 근거 검색

### D. OCR Chunk

- 단위: 이미지 내 텍스트
- 용도: 텍스트가 이미지로만 들어간 슬라이드 보강

### E. Workflow Chunk

- 단위: 슬라이드 내 단계, 담당자, 연결 관계, 예외 흐름
- 용도: 프로세스 질의 대응

## Chroma 메타데이터 권장 스키마

| 필드 | 설명 |
|---|---|
| `workspace_id` | 검색 범위 분리용 |
| `document_id` | 논리 문서 ID |
| `document_version_id` | 업로드 버전 ID |
| `document_name` | 원본 파일명 |
| `total_slide_count` | 문서 전체 페이지 수 |
| `slide_id` | 슬라이드 내부 ID (`DOC_META`는 null 가능) |
| `slide_no` | 사용자 표시용 슬라이드 번호 (`DOC_META`는 0) |
| `chunk_id` | 청크 ID |
| `chunk_type` | `DOC_META`, `SUMMARY`, `DETAIL`, `OCR`, `NOTE`, `WORKFLOW_STEP`, `WORKFLOW_EDGE`, `WORKFLOW_EXCEPTION` |
| `category_function` | 기능 카테고리 코드 (예: `SALES_FORECAST`) |
| `category_industry` | 업종 카테고리 코드 (예: `MANUFACTURING`) |
| `category_doc_type` | 문서 유형 코드 (예: `PROPOSAL`) |
| `category_labels` | 검색 친화 라벨 배열 (한글/영문) |
| `section_title` | 추정 섹션명 |
| `text` | 원문 또는 정제 텍스트 |
| `keywords` | 추출 키워드 |
| `language` | 언어 |
| `embedding_model` | `bge-m3` |
| `source_hash` | 파일 해시 |
| `ingested_at` | 적재 시각 |

> Chroma는 메타 필드에 list 타입을 직접 지원하지 않으므로 `category_labels`는 구분자(`|`) join 문자열 또는 boolean flag 컬럼 세트로 평탄화한다.

## 질의 처리 구조

1. 사용자 질문 수신
2. workspace/document/category scope 결정 (UI에서 카테고리 필터 선택 가능)
3. Chroma에서 상위 chunk 검색 (`DOC_META` + slide-level chunk 병행)
4. `slide_no` 기준으로 묶어 slide evidence 후보 생성
5. slide summary와 detail chunk를 합쳐 재랭킹
6. workflow 질의로 보이면 workflow chunk를 우선 가중
7. 상위 슬라이드 3~5개 선정
8. LLM 답변 생성 시 근거 슬라이드, 파일명, 카테고리 삽입
9. 응답에 근거 목록과 카테고리 라벨 별도 포함

## 인포그래픽 대응 전략

- 단순 텍스트 추출만으로는 부족하므로, 슬라이드 단위 요약 생성이 필수다.
- 요약 생성 입력에는 다음을 합친다.
  - 제목/부제
  - 본문 텍스트
  - 표 셀 텍스트
  - SmartArt/도형 텍스트
  - 이미지 OCR 결과
  - 발표자 노트
- 차트 수치 자체를 완전 구조화하지 못하더라도, 축/범례/레이블 텍스트를 우선 확보한다.
- 워킹 프로세스 슬라이드는 step/edge/actor 구조 추출을 추가로 시도한다.

## 대시보드 데이터 흐름

```text
[Ingestion 완료 이벤트]
   |
   v
[DashboardSnapshot 갱신]
   |-- 카테고리 분포 (function / industry / doc_type)
   |-- 인덱싱 상태 카운트 (PENDING / RUNNING / SUCCESS / FAILED)
   |-- 일자별 적재 추이 (slide count, chunk count)
   |-- 최근 업로드 N건
   |
   v
[Spring API: /api/dashboard/*]
   |
   v
[React Dashboard: 카드/도넛/스택드 바/타임라인 인포그래픽]
```

- 대시보드는 RDB 집계 + Chroma `count_by_metadata` 결과를 캐시한다.
- 적재 변경 이벤트는 in-process 메시지 또는 Spring `ApplicationEvent`로 전달해 캐시 무효화한다.

## 추천 구현 메모

- 파싱 모듈, 임베딩 모듈, 벡터 저장소 모듈을 인터페이스로 분리한다.
- 임베딩 모듈은 `bge-m3` 어댑터를 기본으로 제공하고, 차원/거리(`cosine`) 설정을 노출한다.
- 프로세스 추출기는 일반 요약기와 분리한다.
- 카테고리 정규화기는 사용자 입력과 LLM 추천을 합쳐 taxonomy 코드로 매핑한다.
- Chroma 연동은 SDK 종속보다 HTTP 어댑터 레이어를 두는 편이 교체가 쉽다.
- 검색 결과는 chunk 중심이 아니라 slide 중심으로 후처리해 응답한다.
- 프론트엔드는 Vite 기반으로 모듈을 분리하고, 대시보드 시각화는 `Recharts` 또는 `ECharts`를 어댑터로 둔다.

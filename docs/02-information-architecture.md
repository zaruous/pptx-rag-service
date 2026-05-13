# 정보 구조 및 아키텍처

## 상위 구조

```text
[Client]
   |
   v
[Spring API]
   |-- 업로드/상태 API
   |-- 질의 API
   |-- 관리자 API
   |
   |--> [RDB: 문서/버전/잡/감사로그]
   |--> [파일 저장소: 원본 PPTX, 썸네일]
   |--> [Ingestion Worker]
            |--> Apache POI 파싱
            |--> OCR
            |--> Slide Summary 생성
            |--> Embedding
            |--> Chroma upsert
```

## 논리 엔티티

- Workspace: 문서 묶음 단위. 팀, 프로젝트, 고객사 기준으로 분리 가능
- Document: 사용자가 업로드한 논리 문서
- DocumentVersion: 실제 업로드 파일 단위
- Slide: 문서 버전 내 개별 슬라이드
- SlideChunk: 검색 대상 텍스트 단위
- WorkflowGraph: 프로세스형 슬라이드의 단계/연결 관계
- IngestionJob: 파싱/임베딩/적재 배치 실행 단위
- QueryLog: 질의 및 반환 근거 기록

## 권장 저장 전략

### 1. RDB에 저장할 것

- 문서 메타데이터
- 파일 해시
- 업로드 이력
- 처리 상태
- 실패 원인
- 슬라이드 기본 메타정보
- 질의 로그

### 2. Chroma에 저장할 것

- slide summary chunk 임베딩
- slide detail chunk 임베딩
- OCR chunk 임베딩
- retrieval용 메타데이터

### 3. 파일 저장소에 저장할 것

- 원본 PPTX
- 슬라이드 썸네일 이미지
- OCR 임시 이미지

## 청크 전략

### A. Slide Summary Chunk

- 단위: 슬라이드당 1개
- 내용: 제목, 핵심 메시지, 도형/표/이미지 설명을 합친 요약 텍스트
- 용도: 맥락 검색, 인포그래픽 개요 검색

### B. Detail Chunk

- 단위: 슬라이드 내 텍스트 블록, 표, 노트, 섹션별 분할
- 용도: 정밀 근거 검색

### C. OCR Chunk

- 단위: 이미지 내 텍스트
- 용도: 텍스트가 이미지로만 들어간 슬라이드 보강

### D. Workflow Chunk

- 단위: 슬라이드 내 단계, 담당자, 연결 관계, 예외 흐름
- 용도: 프로세스 질의 대응

## Chroma 메타데이터 권장 스키마

| 필드 | 설명 |
|---|---|
| `workspace_id` | 검색 범위 분리용 |
| `document_id` | 논리 문서 ID |
| `document_version_id` | 업로드 버전 ID |
| `document_name` | 원본 파일명 |
| `slide_id` | 슬라이드 내부 ID |
| `slide_no` | 사용자 표시용 슬라이드 번호 |
| `chunk_id` | 청크 ID |
| `chunk_type` | `SUMMARY`, `DETAIL`, `OCR`, `NOTE` |
| `section_title` | 추정 섹션명 |
| `text` | 원문 또는 정제 텍스트 |
| `keywords` | 추출 키워드 |
| `language` | 언어 |
| `source_hash` | 파일 해시 |
| `ingested_at` | 적재 시각 |

## 질의 처리 구조

1. 사용자 질문 수신
2. workspace/document scope 결정
3. Chroma에서 상위 chunk 검색
4. `slide_no` 기준으로 묶어 slide evidence 후보 생성
5. slide summary와 detail chunk를 합쳐 재랭킹
6. workflow 질의로 보이면 workflow chunk를 우선 가중
7. 상위 슬라이드 3~5개 선정
8. LLM 답변 생성 시 근거 슬라이드와 파일명 삽입
9. 응답에 근거 목록 별도 포함

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

## 추천 구현 메모

- 파싱 모듈, 임베딩 모듈, 벡터 저장소 모듈을 인터페이스로 분리한다.
- 프로세스 추출기는 일반 요약기와 분리한다.
- Chroma 연동은 SDK 종속보다 HTTP 어댑터 레이어를 두는 편이 교체가 쉽다.
- 검색 결과는 chunk 중심이 아니라 slide 중심으로 후처리해 응답한다.

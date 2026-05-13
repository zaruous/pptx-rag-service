# 핵심 엔티티 설계

## 1. workspaces

- 목적: 검색 및 권한 범위를 분리하는 최상위 단위
- PK: `workspace_id`
- 주요 컬럼:
  - `workspace_key`
  - `workspace_name`
  - `is_active`

## 2. documents

- 목적: 논리 문서 엔티티
- PK: `document_id`
- FK: `workspace_id -> workspaces`
- 주요 컬럼:
  - `document_name`
  - `current_version_id`
  - `document_status`
  - `latest_source_hash`

## 3. document_versions

- 목적: 실제 업로드 파일 단위
- PK: `document_version_id`
- FK: `document_id -> documents`
- 주요 컬럼:
  - `version_no`
  - `source_file_name`
  - `source_hash`
  - `storage_path`
  - `file_size`
  - `slide_count`
  - `parse_status`
  - `embedding_status`
  - `indexed_at`

## 4. slides

- 목적: 슬라이드 메타정보
- PK: `slide_id`
- FK: `document_version_id -> document_versions`
- 주요 컬럼:
  - `slide_no`
  - `slide_title`
  - `thumbnail_path`
  - `has_image`
  - `has_table`
  - `has_notes`
  - `summary_text`
  - `raw_text_length`

## 5. slide_chunks

- 목적: Chroma에 적재된 chunk의 운영 인덱스
- PK: `chunk_id`
- FK: `slide_id -> slides`
- 주요 컬럼:
  - `chunk_type`
  - `chunk_order`
  - `chunk_text_preview`
  - `chroma_collection`
  - `chroma_vector_id`
  - `token_count`
  - `is_indexed`

## 6. workflow_nodes

- 목적: 프로세스형 슬라이드의 단계 정보 저장
- PK: `workflow_node_id`
- FK: `slide_id -> slides`
- 주요 컬럼:
  - `step_no`
  - `step_name`
  - `actor_name`
  - `node_type`
  - `condition_text`

## 7. workflow_edges

- 목적: 프로세스 단계 간 연결 관계 저장
- PK: `workflow_edge_id`
- FK:
  - `slide_id -> slides`
  - `from_workflow_node_id -> workflow_nodes`
  - `to_workflow_node_id -> workflow_nodes`
- 주요 컬럼:
  - `edge_type`
  - `condition_text`
  - `is_exception_path`

## 8. ingestion_jobs

- 목적: 업로드 후 처리 파이프라인 상태 추적
- PK: `job_id`
- FK: `document_version_id -> document_versions`
- 주요 컬럼:
  - `job_type`
  - `job_status`
  - `started_at`
  - `finished_at`
  - `error_code`
  - `error_message`
  - `retry_count`

## 9. query_logs

- 목적: 질의 이력과 검색 품질 분석
- PK: `query_log_id`
- FK: `workspace_id -> workspaces`
- 주요 컬럼:
  - `question_text`
  - `top_document_ids`
  - `top_slide_nos`
  - `answer_model`
  - `latency_ms`
  - `result_count`

## 권장 인덱스

- `documents(workspace_id, document_status)`
- `document_versions(document_id, version_no desc)`
- `slides(document_version_id, slide_no)`
- `slide_chunks(slide_id, chunk_type, is_indexed)`
- `workflow_nodes(slide_id, step_no)`
- `workflow_edges(slide_id, from_workflow_node_id, to_workflow_node_id)`
- `ingestion_jobs(document_version_id, job_status)`
- `query_logs(workspace_id, created_at desc)`

## 상태 필드 예시

| 테이블 | 필드 | 예시 값 |
|---|---|---|
| `documents` | `document_status` | `ACTIVE`, `INACTIVE`, `DELETED` |
| `document_versions` | `parse_status` | `PENDING`, `RUNNING`, `DONE`, `FAILED` |
| `document_versions` | `embedding_status` | `PENDING`, `RUNNING`, `DONE`, `PARTIAL_FAILED`, `FAILED` |
| `ingestion_jobs` | `job_status` | `QUEUED`, `RUNNING`, `DONE`, `FAILED`, `CANCELLED` |

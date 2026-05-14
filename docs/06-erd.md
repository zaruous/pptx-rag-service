# Mermaid ERD

```mermaid
erDiagram
    workspaces ||--o{ documents : owns
    documents ||--o{ document_versions : has
    documents ||--o{ document_categories : tagged
    category_taxonomy ||--o{ document_categories : refs
    document_versions ||--o{ slides : contains
    document_versions ||--o{ document_meta_chunks : summarizes
    slides ||--o{ slide_chunks : contains
    slides ||--o{ workflow_nodes : contains
    slides ||--o{ workflow_edges : contains
    workflow_nodes ||--o{ workflow_edges : from
    workflow_nodes ||--o{ workflow_edges : to
    document_versions ||--o{ ingestion_jobs : triggers
    workspaces ||--o{ query_logs : stores
    workspaces ||--o{ dashboard_snapshots : aggregates

    workspaces {
        bigint workspace_id PK
        string workspace_key
        string workspace_name
        boolean is_active
    }

    documents {
        bigint document_id PK
        bigint workspace_id FK
        string document_name
        bigint current_version_id
        string document_status
        string latest_source_hash
    }

    document_versions {
        bigint document_version_id PK
        bigint document_id FK
        int version_no
        string source_file_name
        string source_hash
        string storage_path
        int slide_count
        string document_summary
        string parse_status
        string embedding_status
        string embedding_model
        int embedding_dim
    }

    category_taxonomy {
        bigint taxonomy_id PK
        string axis
        string code
        string label_ko
        string label_en
        bigint parent_taxonomy_id FK
        boolean is_active
        int sort_order
    }

    document_categories {
        bigint document_category_id PK
        bigint document_id FK
        bigint taxonomy_id FK
        string axis
        string source
        double confidence
        string review_status
    }

    document_meta_chunks {
        bigint doc_meta_chunk_id PK
        bigint document_version_id FK
        string chroma_collection
        string chroma_vector_id
        string composed_text
        boolean is_indexed
    }

    slides {
        bigint slide_id PK
        bigint document_version_id FK
        int slide_no
        string slide_title
        string thumbnail_path
        string summary_text
    }

    slide_chunks {
        bigint chunk_id PK
        bigint slide_id FK
        string chunk_type
        int chunk_order
        string chroma_collection
        string chroma_vector_id
        boolean is_indexed
    }

    workflow_nodes {
        bigint workflow_node_id PK
        bigint slide_id FK
        int step_no
        string step_name
        string actor_name
        string node_type
    }

    workflow_edges {
        bigint workflow_edge_id PK
        bigint slide_id FK
        bigint from_workflow_node_id FK
        bigint to_workflow_node_id FK
        string edge_type
        boolean is_exception_path
    }

    ingestion_jobs {
        bigint job_id PK
        bigint document_version_id FK
        string job_type
        string job_status
        string error_code
    }

    query_logs {
        bigint query_log_id PK
        bigint workspace_id FK
        string question_text
        string answer_model
        int latency_ms
        string applied_category_filter
    }

    dashboard_snapshots {
        bigint snapshot_id PK
        bigint workspace_id FK
        string snapshot_type
        string payload_json
        datetime captured_at
    }
```

## 보조 설명

- `category_taxonomy.axis` 는 `FUNCTION`, `INDUSTRY`, `DOC_TYPE` 중 하나다.
- `document_categories.source` 는 `USER_INPUT`, `LLM_SUGGEST`, `OPERATOR_REVIEW` 등으로 구분해 검수 흐름 추적.
- `document_meta_chunks` 는 문서 단위 임베딩(`DOC_META`)을 별도 추적하여 슬라이드 chunk와 lifecycle을 분리한다.
- `dashboard_snapshots.snapshot_type` 예: `CATEGORY_DISTRIBUTION`, `INGESTION_STATUS`, `UPLOAD_TIMELINE`, `RECENT_DOCUMENTS`.

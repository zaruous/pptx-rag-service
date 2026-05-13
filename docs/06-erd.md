# Mermaid ERD

```mermaid
erDiagram
    workspaces ||--o{ documents : owns
    documents ||--o{ document_versions : has
    document_versions ||--o{ slides : contains
    slides ||--o{ slide_chunks : contains
    slides ||--o{ workflow_nodes : contains
    slides ||--o{ workflow_edges : contains
    workflow_nodes ||--o{ workflow_edges : from
    workflow_nodes ||--o{ workflow_edges : to
    document_versions ||--o{ ingestion_jobs : triggers
    workspaces ||--o{ query_logs : stores

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
        string parse_status
        string embedding_status
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
    }
```

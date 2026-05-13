# 상태 설계

## Document Status

| 상태 | 설명 | 전이 가능 상태 |
|---|---|---|
| `ACTIVE` | 검색 가능 | `INACTIVE`, `DELETED` |
| `INACTIVE` | 저장은 유지, 검색 제외 | `ACTIVE`, `DELETED` |
| `DELETED` | 삭제 처리 | 없음 |

## Parse Status

| 상태 | 설명 | 전이 가능 상태 |
|---|---|---|
| `PENDING` | 대기 | `RUNNING`, `FAILED` |
| `RUNNING` | 처리 중 | `DONE`, `FAILED` |
| `DONE` | 완료 | `PENDING` |
| `FAILED` | 실패 | `PENDING` |

## Embedding Status

| 상태 | 설명 | 전이 가능 상태 |
|---|---|---|
| `PENDING` | 대기 | `RUNNING`, `FAILED` |
| `RUNNING` | 임베딩/적재 중 | `DONE`, `PARTIAL_FAILED`, `FAILED` |
| `DONE` | 완료 | `PENDING` |
| `PARTIAL_FAILED` | 일부 chunk 실패 | `PENDING`, `FAILED` |
| `FAILED` | 전체 실패 | `PENDING` |

## Ingestion Job Status

| 상태 | 트리거 | 실패/롤백 조건 |
|---|---|---|
| `QUEUED` | 업로드 또는 재처리 요청 | 작업 취소 |
| `RUNNING` | 워커가 작업 시작 | 예외 발생 시 `FAILED` |
| `DONE` | 모든 단계 완료 | 없음 |
| `FAILED` | 치명적 예외 | 재시도 가능 |
| `CANCELLED` | 운영자 취소 | 부분 적재 시 정리 필요 |

## Review Status

| 상태 | 설명 | 전이 가능 상태 |
|---|---|---|
| `NOT_REQUIRED` | 자동 검수 대상 아님 | 없음 |
| `PENDING_REVIEW` | 운영자 검수 대기 | `APPROVED`, `REJECTED`, `REPROCESS_REQUESTED` |
| `APPROVED` | 사용 가능 승인 | `REPROCESS_REQUESTED` |
| `REJECTED` | 검색 노출 제외 대상 | `REPROCESS_REQUESTED` |
| `REPROCESS_REQUESTED` | 재처리 요청 상태 | `PENDING_REVIEW`, `APPROVED` |

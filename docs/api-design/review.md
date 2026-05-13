# Review API

## 목적

저신뢰 슬라이드와 workflow의 검수 큐 조회 및 검수 결정 처리

## 엔드포인트

| Method | Path | 목적 | 요청 | 응답 | 인증 |
|---|---|---|---|---|---|
| `GET` | `/review-queue/slides` | 검수 후보 목록 조회 | workspace, priority, status | candidates | 운영자 |
| `GET` | `/review-queue/slides/{reviewId}` | 검수 대상 상세 조회 | 없음 | slide IR, semantic result, chunks | 운영자 |
| `POST` | `/review-queue/slides/{reviewId}/approve` | 검수 승인 | comment | review result | 운영자 |
| `POST` | `/review-queue/slides/{reviewId}/reject` | 검수 반려 | reasonCode, comment | review result | 운영자 |
| `POST` | `/review-queue/slides/{reviewId}/reprocess` | 재처리 요청 | requestedAction, comment | job info | 운영자 |
| `POST` | `/review-queue/slides/{reviewId}/disable-workflow` | workflow chunk 비활성화 | comment | update result | 운영자 |

## 응답 규칙

- 반려된 대상은 retrieval에서 기본 제외 가능해야 한다.
- 승인된 대상은 `reviewStatus=APPROVED`로 검색 boost 대상이 될 수 있다.
- 재처리 요청은 slide-level job 생성과 연결되는 구성이 바람직하다.

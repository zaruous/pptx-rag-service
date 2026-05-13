# Admin API

## 목적

문서 운영, 삭제, 상태 점검, 인덱스 정리

## 엔드포인트

| Method | Path | 목적 | 요청 | 응답 | 인증 |
|---|---|---|---|---|---|
| `POST` | `/documents/{documentId}/deactivate` | 문서 비활성화 | 없음 | 상태 변경 결과 | 관리자 |
| `DELETE` | `/document-versions/{versionId}` | 특정 버전 삭제 | hard/soft 옵션 | 삭제 결과 | 관리자 |
| `POST` | `/documents/{documentId}/purge-index` | Chroma 인덱스 정리 | 없음 | 삭제 건수 | 관리자 |
| `GET` | `/admin/health/indexes` | 인덱스 상태 점검 | 없음 | 컬렉션/문서 수 | 관리자 |

## 운영 규칙

- 삭제 시 RDB와 Chroma의 정합성을 보장해야 한다.
- hard delete는 감사 요구가 없다면 제한적으로만 허용한다.

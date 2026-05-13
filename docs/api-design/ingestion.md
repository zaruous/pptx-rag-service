# Ingestion API

## 목적

PPTX 업로드와 인덱싱 파이프라인 제어

## 엔드포인트

| Method | Path | 목적 | 요청 | 응답 | 인증 |
|---|---|---|---|---|---|
| `POST` | `/workspaces/{workspaceId}/documents` | PPTX 업로드 | multipart file, OCR 옵션 | document/version/job 정보 | 운영자 |
| `GET` | `/workspaces/{workspaceId}/documents` | 문서 목록 조회 | page, status | 문서 목록 | 조회권한 |
| `GET` | `/documents/{documentId}` | 문서 상세 조회 | 없음 | 문서/버전/슬라이드 요약 | 조회권한 |
| `GET` | `/document-versions/{versionId}/jobs/{jobId}` | 인덱싱 상태 조회 | 없음 | job 상태 | 운영자 |
| `POST` | `/document-versions/{versionId}/reingest` | 재처리 요청 | 재처리 옵션 | 신규 job 정보 | 운영자 |

## 요청 규칙

- 업로드 시 `workspaceId`는 필수
- 중복 파일 정책은 `skip`, `new-version`, `force-reingest` 중 하나를 지원하는 구성이 좋다

## 상태 코드

- `201`: 업로드 접수
- `200`: 조회/재처리 성공
- `409`: 동일 파일 중복 충돌
- `422`: 파싱 불가 파일

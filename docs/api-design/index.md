# API 설계 인덱스

## 기본 원칙

- Base path: `/api/v1`
- 응답은 `data`, `meta`, `error` 구조를 권장
- 업로드 후 인덱싱은 비동기 처리
- 질의 응답은 근거 슬라이드 목록을 반드시 포함

## API 그룹

- `ingestion.md`: 업로드, 상태 조회, 재처리
- `retrieval.md`: 질의, 근거 조회
- `admin.md`: 삭제, 비활성화, 운영 점검
- `review.md`: 검수 큐 조회, 승인/반려/재처리

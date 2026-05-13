# Ingestion And Query Scenarios

| 시나리오 ID | 시나리오명 | 선행조건 | 기대 결과 | 승인 기준 |
|---|---|---|---|---|
| `TS-01` | 단일 PPTX 업로드 성공 | 정상 PPTX 준비 | 문서, 버전, job 생성 | 상태 `DONE` |
| `TS-02` | 동일 파일 중복 업로드 | 같은 해시 파일 존재 | 정책에 따라 skip 또는 새 버전 처리 | 중복 정책 일관 |
| `TS-03` | OCR 비활성 업로드 | 이미지 포함 PPTX | OCR chunk 없이 인덱싱 | summary/detail 검색 가능 |
| `TS-04` | OCR 활성 업로드 | 이미지 텍스트 포함 PPTX | OCR chunk 생성 | 이미지 텍스트 질의 성공 |
| `TS-05` | 인포그래픽 질의 | 도형/표 중심 슬라이드 존재 | summary 기반으로 관련 슬라이드 반환 | slideNo 정확 |
| `TS-06` | 워킹 프로세스 순서 질의 | workflow 슬라이드 존재 | 단계 순서가 맞게 답변 | step 순서 정확 |
| `TS-07` | 워킹 프로세스 역할 질의 | actor 포함 workflow 존재 | 담당 부서가 맞게 반환 | actor 정확 |
| `TS-08` | 워킹 프로세스 예외 질의 | 예외 edge 존재 | 되돌림/분기 경로 반환 | exception 정확 |
| `TS-09` | 일반 텍스트 질의 | 본문 텍스트 존재 | detail chunk 근거 반환 | snippet 적절 |
| `TS-10` | 검색 범위 제한 | 여러 workspace 존재 | 다른 workspace 문서 미노출 | 권한 격리 |
| `TS-11` | 재처리 성공 | 기존 버전 존재 | 신규 ingestion job 완료 | 이전 인덱스 정합성 유지 |
| `TS-12` | Chroma 일부 적재 실패 | 장애 유도 | 상태 `PARTIAL_FAILED` 기록 | 실패 chunk 식별 가능 |
| `TS-13` | 질의 결과 없음 | 무관한 질문 입력 | 빈 evidence 또는 no-answer 응답 | 환각 답변 금지 |

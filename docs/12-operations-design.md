# 운영 설계

## 운영 역할

- 운영자: 업로드, 재처리, 문서 상태 점검
- 관리자: 삭제, 컬렉션 정리, 모델 정책 관리
- 개발자/플랫폼: 장애 대응, 성능 튜닝, 배포

## 운영 정책

### 업로드 정책

- 기본은 새 버전 생성
- 동일 해시 재업로드는 정책 옵션화
- 대용량 파일은 비동기 처리 강제

### 재처리 정책

- 임베딩 모델 변경 시 전체 재처리 가능
- OCR 정책 변경 시 OCR 대상 슬라이드만 부분 재처리 가능
- 검수 반려는 slide-level 재처리의 직접 트리거가 된다

### 삭제 정책

- 문서 비활성화 후 일정 기간 뒤 hard delete 권장
- hard delete 시 RDB, 파일 저장소, Chroma를 모두 정리

### 검수 정책

- low-confidence workflow는 기본적으로 검수 후보에 올린다
- 승인 전이라도 retrieval에 노출할 수는 있지만 score penalty를 둔다
- 반려 대상은 기본 검색 결과에서 제외하거나 강한 penalty를 준다

## 감사 로그 권장

- 누가 어떤 문서를 업로드했는지
- 누가 재처리/삭제를 실행했는지
- 어떤 질문이 어떤 슬라이드를 근거로 사용했는지
- 어떤 `prompt_version`, `model_name`, `schema_version`으로 추출했는지
- 누가 어떤 review decision을 내렸는지

## 장애 대응 포인트

- Chroma 연결 실패
- 임베딩 API 속도 저하
- OCR 과부하
- 손상된 PPTX 파싱 실패
- workflow 추출 환각 또는 오분류
- 검수 큐 적체

## 모니터링 지표

- 업로드 건수
- 평균 인덱싱 시간
- 문서당 평균 chunk 수
- 질의 latency
- evidence 반환 성공률
- 재처리 실패율
- workflow 추출 성공률
- low-confidence slide 비율
- review queue backlog
- review decision turnaround time

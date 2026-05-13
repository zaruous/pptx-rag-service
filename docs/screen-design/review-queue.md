# Review Queue

## 화면명

저신뢰 슬라이드 및 워크플로우 검수

## 목표

운영자가 low-confidence 추출 결과를 검토하고 승인, 반려, 재처리 요청을 수행한다.

## 주요 사용자

- 운영자
- 품질 관리자

## 진입 경로

- 관리자 메뉴의 검수 큐
- 문서 상세 화면의 저신뢰 항목 링크

## 주요 컴포넌트

- 검수 대기 목록 테이블
- 필터 바: workspace, document, slide type, review priority
- 슬라이드 썸네일 미리보기
- 원천 텍스트 패널
- summary/facts/workflow 비교 패널
- 검수 액션 버튼

## 입력

- review candidate ID
- 검수 의견
- 조치 유형

## 출력

- review status 변경
- reingest 요청 생성
- retrieval 노출 여부 반영

## 주요 액션

- 승인
- 반려
- 재처리 요청
- workflow 비활성화
- chunk 비노출 처리

## 검증 규칙

- 반려 시 reason code 필수
- 재처리 요청 시 requested action 필수
- 승인 시 근거 패널 확인 권장

## 빈 상태

- 검수 대기 없음

## 오류 상태

- 썸네일 로딩 실패
- 원천 텍스트 조회 실패
- 검수 결과 저장 실패

## 권한

- 운영자 이상

## 추적 이벤트

- `review_candidate_opened`
- `review_approved`
- `review_rejected`
- `reingest_requested_from_review`

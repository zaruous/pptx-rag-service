# Upload Admin

## 화면명

PPTX 업로드 및 인덱싱 관리

## 목표

운영자가 PPTX를 업로드하고, 인덱싱 상태와 실패 여부를 확인한다.

## 주요 사용자

- 운영자
- 시스템 관리자

## 진입 경로

- 관리자 메뉴

## 주요 컴포넌트

- 파일 업로드 영역
- workspace 선택
- OCR 사용 여부 체크
- 업로드 이력 테이블
- 인덱싱 상태 배지
- 재처리 버튼

## 입력

- PPTX 파일
- workspace ID
- 재처리 옵션

## 출력

- document/version 생성 결과
- ingestion job 상태

## 주요 액션

- 업로드
- 재처리
- 실패 건 재시도

## 검증 규칙

- 파일 확장자 `.pptx` 제한
- 최대 용량 제한
- 동일 해시 파일 중복 정책 적용

## 빈 상태

- 업로드된 문서 없음

## 오류 상태

- 파일 저장 실패
- 파싱 실패
- 임베딩 실패

## 권한

- 운영자 이상만 접근 가능

## 추적 이벤트

- `pptx_upload_started`
- `pptx_upload_completed`
- `ingestion_retry_clicked`

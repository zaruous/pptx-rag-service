# 배포 및 준비 체크리스트

## 환경 가정

- Spring Boot 애플리케이션 서버
- Chroma 서버 또는 컨테이너
- RDB
- 파일 저장소 로컬 또는 오브젝트 스토리지

## 필수 준비 항목

- Chroma collection 생성 정책 정의
- 임베딩 모델 연결 확인
- 업로드 저장 경로 용량 점검
- OCR 엔진 설치 또는 API 키 준비
- 최대 파일 크기 및 타임아웃 설정
- prompt/schema/model version 저장 정책 정의
- 골드셋 평가 절차 준비

## 운영 준비 항목

- 실패 job 재시도 배치
- 로그 마스킹 정책
- index purge 관리자 기능
- 헬스체크 엔드포인트

## 배포 전 점검

- 대형 PPTX 업로드 테스트
- 동시 업로드 테스트
- 다중 문서 질의 테스트
- 슬라이드 번호 정확도 검증
- 삭제 후 Chroma 정리 검증
- workflow 질의 정확도 검증
- low-confidence fallback 동작 검증

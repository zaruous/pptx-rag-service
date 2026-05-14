# PPTX RAG 서비스 설계 문서 인덱스

## 전제

- 서버는 `Spring Boot + Gradle (Java)` 기반이다.
- 클라이언트는 `React + Vite` 기반 SPA다.
- 벡터 저장소는 `Chroma`를 사용하고, 임베딩 모델은 `BAAI/bge-m3`를 표준으로 한다.
- 사용자는 여러 개의 PPTX를 반복 업로드하며, 업로드 시 `기능 / 업종 / 유형` 카테고리를 지정한다.
- 카테고리, 문서 메타데이터(파일명, 페이지 수, 요약 등)도 LLM이 정제해 벡터 DB에 함께 임베딩한다.
- 질의 시 관련 문서, 슬라이드 번호, 카테고리를 함께 확인할 수 있어야 한다.
- 메인 페이지는 적재 현황을 인포그래픽으로 보여주는 대시보드 역할을 한다.
- 목표는 단순 텍스트 검색이 아니라, 인포그래픽과 슬라이드 맥락을 반영한 RAG 파이프라인 설계다.

## 권장 읽기 순서

1. `01-requirements-summary.md`
2. `02-information-architecture.md`
3. `03-user-flow.md`
4. `04-extraction-design.md`
5. `05-llm-extraction-design.md`
6. `07-schema-specifications.md`
7. `15-retrieval-ranking-review.md`
8. `16-application-architecture.md`
9. `18-category-taxonomy.md`
10. `19-dashboard-design.md`
11. `db-design/`
12. `06-erd.md`
13. `api-design/`
14. `08-functional-requirements.md`
15. `09-state-design.md`
16. `10-permissions-matrix.md`
17. `11-extraction-quality-evaluation.md`
18. `test-scenarios/`
19. `12-operations-design.md`
20. `13-deployment-readiness.md`
21. `14-security-privacy.md`
22. `17-business-workflow.md`

## 문서 맵

- 요구사항 요약: `01-requirements-summary.md`
- 정보 구조 및 아키텍처: `02-information-architecture.md`
- 업로드/질의 흐름: `03-user-flow.md`
- 추출 설계: `04-extraction-design.md`
- LLM 추출 설계: `05-llm-extraction-design.md`
- 스키마 명세: `07-schema-specifications.md`
- 검색/재랭킹/검수 설계: `15-retrieval-ranking-review.md`
- 애플리케이션 아키텍처: `16-application-architecture.md`
- 카테고리 분류 체계: `18-category-taxonomy.md`
- 대시보드 인포그래픽 설계: `19-dashboard-design.md`
- 화면 설계: `screen-design/`
- 데이터 설계: `db-design/`
- ERD: `06-erd.md`
- API 설계: `api-design/`
- 기능 요구사항: `08-functional-requirements.md`
- 상태 설계: `09-state-design.md`
- 권한 설계: `10-permissions-matrix.md`
- 추출 품질 평가: `11-extraction-quality-evaluation.md`
- 테스트 시나리오: `test-scenarios/`
- 운영 설계: `12-operations-design.md`
- 배포 체크리스트: `13-deployment-readiness.md`
- 보안/개인정보: `14-security-privacy.md`
- 업무 순서 설계: `17-business-workflow.md`

## 검토 포인트

- `Chroma`만으로 운영 메타데이터를 모두 관리하지 말고, 문서/배치/잡 상태는 별도 RDB에 저장하는 구성이 권장된다.
- 슬라이드별 `summary chunk`와 `detail chunk`를 함께 생성해야 인포그래픽 맥락 검색 정확도가 올라간다.
- 워킹 프로세스 슬라이드는 `workflow graph` 형태의 구조 추출을 별도로 가져가야 한다.
- 카테고리 (`function / industry / docType`)는 업로드 단계에서 사용자가 선택하고, 문서 메타 chunk(파일명·페이지수·요약·카테고리)도 함께 임베딩한다.
- 임베딩 모델은 `bge-m3`로 고정하여 한국어/영어 혼합 검색 품질을 우선 확보한다.
- 추출 품질은 주관적 판단이 아니라 `field accuracy`, `retrieval hit`, `grounded answer rate`로 계량해야 한다.
- 구현 직전으로 가려면 `schema`, `query classification`, `no-answer`, `human review` 정책이 먼저 고정되어야 한다.
- 응답 포맷에는 반드시 `document_name`, `slide_no`, `snippet`, `score`, `categories`를 포함해야 한다.
- 메인 대시보드는 카테고리 분포, 적재 추이, 인덱싱 상태, 최근 업로드를 인포그래픽으로 제공한다.

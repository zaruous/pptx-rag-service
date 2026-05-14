# 카테고리 분류 체계 (Taxonomy)

## 목표

업로드 시점에 사용자가 문서를 `기능 / 업종 / 유형` 세 축으로 분류하고,  
이 카테고리가 모든 chunk 메타데이터, 검색 필터, 대시보드 인포그래픽까지 일관되게 흐르도록 한다.

## 1. 분류 축

| 축 (Axis) | 코드 | 설명 | 다중 선택 |
|---|---|---|---|
| 기능 | `FUNCTION` | 문서가 다루는 업무 기능 (영업/생산/품질/재무/HR 등) | O |
| 업종 | `INDUSTRY` | 문서가 속하는 산업/도메인 (제조/유통/금융/공공 등) | O |
| 유형 | `DOC_TYPE` | 문서의 형태 (제안서/매뉴얼/리포트/회의자료 등) | X (단일) |

- 기능과 업종은 한 문서가 여러 값을 가질 수 있다 (예: `품질관리` + `출하관리`).
- 유형은 원칙적으로 단일 선택을 권장한다 (예: `업무매뉴얼`).
- 모든 축 값은 `category_taxonomy` 테이블의 `code` 컬럼으로 식별한다.

## 2. 기본 시드 (초기 권장값)

### FUNCTION (기능)

| code | label_ko | label_en |
|---|---|---|
| `SALES` | 영업 | Sales |
| `MARKETING` | 마케팅 | Marketing |
| `PRODUCTION` | 생산 | Production |
| `QUALITY_MANAGEMENT` | 품질관리 | Quality Management |
| `SHIPPING_MANAGEMENT` | 출하관리 | Shipping Management |
| `LOGISTICS` | 물류 | Logistics |
| `PROCUREMENT` | 구매/조달 | Procurement |
| `FINANCE` | 재무/회계 | Finance |
| `HR` | 인사 | Human Resources |
| `IT_OPERATIONS` | IT 운영 | IT Operations |
| `DATA_ANALYTICS` | 데이터 분석 | Data Analytics |
| `STRATEGY` | 전략/기획 | Strategy |
| `LEGAL_COMPLIANCE` | 법무/컴플라이언스 | Legal & Compliance |
| `CUSTOMER_SERVICE` | 고객지원 | Customer Service |
| `RESEARCH_DEVELOPMENT` | R&D | Research & Development |

### INDUSTRY (업종)

| code | label_ko | label_en |
|---|---|---|
| `MANUFACTURING` | 제조 | Manufacturing |
| `DISTRIBUTION` | 유통/소매 | Distribution & Retail |
| `FINANCE_INDUSTRY` | 금융 | Finance |
| `INSURANCE` | 보험 | Insurance |
| `HEALTHCARE` | 의료/헬스케어 | Healthcare |
| `PHARMA` | 제약 | Pharmaceutical |
| `ENERGY` | 에너지 | Energy |
| `CONSTRUCTION` | 건설 | Construction |
| `LOGISTICS_INDUSTRY` | 물류 | Logistics |
| `IT_SERVICES` | IT 서비스 | IT Services |
| `TELECOM` | 통신 | Telecommunications |
| `EDUCATION` | 교육 | Education |
| `PUBLIC` | 공공 | Public Sector |
| `MEDIA` | 미디어/엔터테인먼트 | Media & Entertainment |
| `AUTOMOTIVE` | 자동차 | Automotive |

### DOC_TYPE (유형)

| code | label_ko | label_en |
|---|---|---|
| `PROPOSAL` | 제안서 | Proposal |
| `OPERATION_MANUAL` | 업무매뉴얼 | Operation Manual |
| `TRAINING_MATERIAL` | 교육자료 | Training Material |
| `REPORT` | 보고서 | Report |
| `MEETING_DECK` | 회의자료 | Meeting Deck |
| `STRATEGY_DECK` | 전략 발표자료 | Strategy Deck |
| `PROCESS_GUIDE` | 프로세스 가이드 | Process Guide |
| `PRODUCT_INTRO` | 제품 소개 | Product Introduction |
| `RFP_RESPONSE` | RFP 응답 | RFP Response |
| `ANNUAL_REVIEW` | 연간 결산/리뷰 | Annual Review |
| `CASE_STUDY` | 사례 연구 | Case Study |
| `WHITE_PAPER` | 백서 | White Paper |
| `ETC` | 기타 | Etc |

> 위 코드는 초기 시드이며, 운영자가 `/api/taxonomy` 를 통해 추가/비활성화할 수 있다.

## 3. Taxonomy 마스터 스키마

| 필드 | 타입 | 설명 |
|---|---|---|
| `taxonomyId` | bigint | PK |
| `axis` | enum | `FUNCTION` / `INDUSTRY` / `DOC_TYPE` |
| `code` | string | 식별 코드 (UPPER_SNAKE_CASE) |
| `labelKo` | string | 한글 라벨 |
| `labelEn` | string | 영문 라벨 |
| `parentTaxonomyId` | bigint? | 계층형 분류 지원 |
| `description` | string? | 운영자 설명 |
| `isActive` | boolean | 비활성화 플래그 |
| `sortOrder` | int | UI 정렬 순서 |
| `version` | int | taxonomy 마스터 버전 |

- 계층형이 필요한 경우 `parentTaxonomyId`로 트리 표현 (예: `MANUFACTURING > AUTOMOTIVE_MANUFACTURING`)
- `version`은 taxonomy 자체가 바뀔 때 증가 → 재처리 정책 트리거

## 4. 사용자 입력 → 정규화 흐름

```text
[Upload Form]
  | userCategories (function[], industry[], docType)
  v
[CategoryValidator]
  | 존재 코드 검증 / 비활성 코드 거부
  v
[CategoryNormalizer]
  | 자유 입력 freeTags → LLM 추천 → taxonomy 코드 매핑
  v
[DocumentCategory persist]
  | source = USER_INPUT
  v
[LLM Document Summary 단계]
  | suggestedCategories 출력
  v
[CategoryReviewService]
  | 사용자 입력과 LLM 추천 비교
  | - 일치: 자동 승인 (source 그대로)
  | - 불일치/누락: source = LLM_SUGGEST, status = PENDING_REVIEW
```

### 정규화 규칙

- 사용자 입력은 항상 우선이며, LLM 추천은 보조 신호로만 사용
- `freeTags` (자유 입력)는 LLM이 taxonomy 코드 후보를 반환하면 `confidence >= 0.75` 인 경우에만 자동 매핑
- 자동 매핑 실패 시 `category_review` 큐로 적재되고 chunk 메타에는 `OTHER` 처리
- 비활성화된 코드는 신규 매핑 불가, 기존 매핑은 검수 표시

## 5. Chunk 메타 평탄화

Chroma 메타 필드는 스칼라 타입만 안전하므로 다음과 같이 평탄화한다.

| 메타 필드 | 변환 규칙 |
|---|---|
| `categoryFunction` | function 코드 배열을 `|` join (예: `QUALITY_MANAGEMENT|SHIPPING_MANAGEMENT`) |
| `categoryIndustry` | industry 코드 `|` join |
| `categoryDocType` | 단일 코드 |
| `categoryLabels` | 한글+영문 라벨 합쳐 `|` join. 검색 키워드 매칭용 |
| `categoryTaxonomyVersion` | 적재 시점 마스터 버전 |

추가로 자주 검색되는 boolean flag를 두면 필터 성능이 좋다. 예:

- `cat_fn_quality_management = true`
- `cat_ind_manufacturing = true`

이 평탄화는 chunk 적재 직전에 한 번만 수행한다 (DOC_META, SUMMARY, FACT, WORKFLOW_*, OCR 모두 동일 카테고리).

## 6. 카테고리 기반 검색

### UI 노출

- 검색 화면 상단에 `기능 / 업종 / 유형` 드롭다운/칩 셀렉터 노출
- 다중 선택 시 OR 조건
- 축 간 결합은 AND 조건 (`기능=A AND 업종=B AND 유형=C`)

### 백엔드 처리

- Chroma `where` 절에 카테고리 필터를 적용
- 카테고리 미지정 시 전체 검색 (필터 미적용)
- `DOC_META` chunk에 더 강한 매칭 가중치를 주어 발견형 질의에서 카테고리 일치를 우선

## 7. 카테고리 검수 큐

자동 적재가 어려운 경우 검수 큐로 들어간다.

| 사유 | 처리 |
|---|---|
| 사용자가 카테고리를 하나도 지정하지 않음 | LLM 추천을 후보로 제시, 운영자 확정 필요 |
| 자유 입력이 taxonomy 매핑 실패 | 신규 코드 추가 제안 또는 `OTHER` 처리 |
| 사용자 입력과 LLM 추천 차이가 큼 | 운영자 검수 후 확정 |
| `categoryConfidence < 0.6` | 검수 + 재추출 |

검수 결정은 `document_categories.review_status`에 반영되며,  
재추출이 필요한 경우 `IngestionJob`이 `REINGEST_REQUESTED`로 다시 큐잉된다.

## 8. 마이그레이션 / 버전 관리

- taxonomy 마스터 변경(`version` 증가) 시:
  - 기존 문서는 그대로 유지 (역호환)
  - 사용자가 명시적으로 `재분류` 요청해야 chunk 메타가 갱신됨
  - 비활성화된 코드는 신규 분류 차단, 기존 분류는 `LEGACY` 표시
- 코드명 변경은 권장하지 않음. 신규 코드 추가 + 기존 비활성화 패턴 사용
- 다국어 라벨 추가는 안전 (검색 키워드만 확장)

## 9. 운영 권장 사항

- 초기에는 시드 코드만으로 시작하고, 운영자가 검수 큐에서 자주 등장하는 자유 태그를 신규 코드로 승격
- 카테고리 분포를 대시보드로 노출해 한쪽으로 치우치지 않도록 관리
- 검수 결과는 LLM 프롬프트 개선 데이터로 누적 (`few-shot` 예시 갱신)

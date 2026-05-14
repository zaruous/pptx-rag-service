# 문서 → 마크다운 논문 수준 변환 전략

## 목표

PDF / PPTX 등 문서에서 텍스트·표·수식·그림·레이아웃을 **논문 수준의 정확도**로 마크다운으로 변환한다.  
단순 텍스트 덤프가 아니라 **구조·의미·시각 요소를 함께 보존**하는 것이 핵심이다.

---

## 1. 정확도 등급 정의

변환 품질을 4단계로 나눠 목적에 맞는 파이프라인을 선택한다.

| 등급 | 목표 | 대상 | 대표 오류 허용 범위 |
|---|---|---|---|
| **L1 기본** | 텍스트 추출 | 선택형 PDF, 간단한 PPTX | 표 구조 손실 허용 |
| **L2 구조** | 제목/표/목록 보존 | 보고서, 슬라이드 | 수식 손실 허용 |
| **L3 시각** | 인포그래픽·차트 포함 | 인포그래픽 PPT, 스캔 문서 | 수식 텍스트 근사 허용 |
| **L4 논문** | 수식·표·그림 완전 보존 | 학술 논문, 기술 문서 | 오탈자 < 0.5%, 수식 완전 복원 |

이 문서는 **L4 논문 수준** 달성을 목표로 한다.

---

## 2. 문서별 핵심 난이도

### 2-1. PDF

| 요소 | 난이도 | 핵심 문제 |
|---|---|---|
| 선택형 텍스트 | 낮음 | 읽기 순서, 컬럼 분리 |
| 스캔/이미지 PDF | 높음 | OCR 품질, 방향, 노이즈 |
| 표 | 높음 | 셀 병합, 테두리 없는 표 |
| 수식 (LaTeX) | 매우 높음 | 폰트 인코딩, 기호 매핑 |
| 다단 레이아웃 | 높음 | 읽기 순서 재구성 |
| 그림/차트 | 중간 | 캡션 연결, alt text |
| 헤더/푸터 분리 | 중간 | 본문과 혼입 방지 |

### 2-2. PPTX

| 요소 | 난이도 | 핵심 문제 |
|---|---|---|
| 텍스트 박스 | 낮음 | z-order, 읽기 순서 |
| SmartArt / 도형 | 높음 | 연결 관계, 화살표 의미 |
| 이미지 전용 슬라이드 | 매우 높음 | 멀티모달 해석 필요 |
| 표 | 낮음 | POI 직접 추출 가능 |
| 차트 | 높음 | 데이터 원본 없으면 레이블만 |
| 애니메이션 순서 | 높음 | 단계 분리 여부 |

---

## 3. 변환 파이프라인 전략

### 3-1. 4단계 파이프라인

```text
[원본 문서]
     │
     ▼
┌─────────────────────────────┐
│ Stage 1: Pre-processing     │  포맷 감지, 암호화 해제, 방향 보정,
│ (결정론적)                   │  해상도 정규화, 메타 추출
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│ Stage 2: Layout Analysis    │  페이지 세그멘테이션, 블록 분류
│ (딥러닝)                    │  (텍스트/표/수식/그림/캡션/헤더)
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│ Stage 3: Element Extraction │  블록 유형별 전문 추출기 적용
│ (하이브리드)                │  텍스트 OCR / 표 재구성 /
│                             │  수식 인식 / 그림 캡션 매핑
└─────────────────────────────┘
     │
     ▼
┌─────────────────────────────┐
│ Stage 4: MD Assembly        │  읽기 순서 복원, 마크다운 조립,
│ (규칙 + LLM 정제)           │  LLM 후처리, 품질 검증
└─────────────────────────────┘
     │
     ▼
[마크다운 출력 + ParseQuality 보고서]
```

### 3-2. PPTX 전용 경로

PPTX는 PDF 변환 없이 직접 파싱이 더 정확하다.

```text
[PPTX]
  ├── Apache POI XSLF → 텍스트/표/노트 직접 추출
  ├── python-pptx (보조) → 썸네일 생성, 도형 관계 추출
  ├── 슬라이드 렌더링 (LibreOffice headless) → 시각 복잡 슬라이드에 멀티모달
  └── 이미지 슬라이드 → Surya OCR + GOT-OCR (수식)
```

---

## 4. 필수 라이브러리 및 도구

### 4-1. PDF 텍스트/레이아웃 추출 (기반 계층)

| 라이브러리 | 언어 | 역할 | 특징 |
|---|---|---|---|
| **PyMuPDF (fitz)** | Python | 텍스트·이미지 고속 추출, 페이지 렌더링 | 선택형 PDF 최고 속도, bbox 정밀 |
| **pdfplumber** | Python | 표 추출, 텍스트 블록 좌표 | camelot보다 유연, 테두리 없는 표 대응 |
| **pdfminer.six** | Python | 문자 수준 폰트/인코딩 정보 | 수식 폰트 분석, 저수준 접근 |
| **Apache PDFBox 3.x** | Java | JVM 환경 PDF 추출 | Spring 서버 내장용 |
| **camelot-py** | Python | 표 전문 추출 (Lattice/Stream 모드) | 복잡 표에 강함, Ghost Script 의존 |

### 4-2. 레이아웃 분석 (딥러닝)

| 라이브러리/모델 | 역할 | 정확도 | GPU 필요 |
|---|---|---|---|
| **Surya** (VikParuchuri) | 레이아웃 분석 + OCR + 읽기 순서 | ⭐⭐⭐⭐⭐ | 권장 (CPU 가능) |
| **LayoutParser** | Detectron2 기반 문서 레이아웃 | ⭐⭐⭐⭐ | 필요 |
| **DocLayout-YOLO** | YOLO 기반 고속 레이아웃 감지 | ⭐⭐⭐⭐ | 필요 |
| **DiT (Document Image Transformer)** | Microsoft, 레이아웃 이해 | ⭐⭐⭐⭐⭐ | 필요 |
| **DOCLAYNET** | IBM 학습 데이터 기반 모델 | ⭐⭐⭐⭐ | 필요 |

### 4-3. OCR 엔진

| 엔진 | 언어 지원 | 정확도 | 속도 | GPU |
|---|---|---|---|---|
| **Surya OCR** | 90개 언어 | ⭐⭐⭐⭐⭐ | 중간 | 권장 |
| **PaddleOCR v4** | 80개+ 언어 | ⭐⭐⭐⭐⭐ | 빠름 | 권장 |
| **TrOCR (Microsoft)** | 영어/중국어 특화 | ⭐⭐⭐⭐⭐ | 느림 | 필요 |
| **Tesseract 5.x** | 100개+ 언어 | ⭐⭐⭐ | 빠름 | 불필요 |
| **EasyOCR** | 80개 언어 | ⭐⭐⭐ | 중간 | 권장 |
| **GOT-OCR 2.0** | 다국어 + 수식 | ⭐⭐⭐⭐⭐ | 중간 | 필요 |

> **한국어 권장:** PaddleOCR v4 또는 Surya OCR (한국어 지원 확인 필수)

### 4-4. 수식 인식 (논문 수준 핵심)

| 도구 | 방식 | 출력 | 정확도 | GPU |
|---|---|---|---|---|
| **UniMERNet** (Shanghai AI Lab) | 이미지 → LaTeX | LaTeX | ⭐⭐⭐⭐⭐ | 필요 |
| **pix2tex (LaTeX-OCR)** | 수식 이미지 → LaTeX | LaTeX | ⭐⭐⭐⭐ | 권장 |
| **Nougat (Meta)** | PDF 페이지 → MMD (Markdown+LaTeX) | MMD | ⭐⭐⭐⭐⭐ | 필요 |
| **GOT-OCR 2.0** | 수식 포함 전체 OCR | LaTeX/MD | ⭐⭐⭐⭐⭐ | 필요 |
| **Mathpix Snip API** | 클라우드 API | LaTeX | ⭐⭐⭐⭐⭐ | 불필요 (유료) |

### 4-5. 통합 PDF→MD 변환 프레임워크 (가장 중요)

아래 세 프레임워크는 파이프라인 전체를 포함한다. **하나를 메인으로 채택하고 나머지를 보완 사용**이 권장된다.

#### A. Docling (IBM, 2024, 오픈소스)

```text
강점: 논문 수준 표/수식/레이아웃, DocLayNet 기반, 구조화 JSON 출력
약점: 느린 처리 속도, 높은 VRAM 요구
라이선스: MIT
언어: Python
```

- DocLayNet으로 학습된 레이아웃 분석
- EasyOCR / Tesseract 통합
- 표: TableFormer 모델 (병합 셀 지원)
- 출력: Markdown, JSON, HTML
- GitHub: `DS4SD/docling`

#### B. Marker (VikParuchuri, 2024, 오픈소스)

```text
강점: 빠름, 실용적 품질, Surya 통합, 수식 지원
약점: 복잡한 표 완전 복원 미흡
라이선스: GPL-3.0 (비상업 무료)
언어: Python
```

- Surya(레이아웃+OCR) + pdftext(텍스트 추출) + 수식 인식 통합
- GPU 없어도 동작 (느리지만 가능)
- 한국어 포함 다국어 지원
- GitHub: `VikParuchuri/marker`

#### C. MinerU (Shanghai AI Lab, 2024, 오픈소스)

```text
강점: 최고 수준 논문 파싱, PDF-Extract-Kit 통합, 수식/표/읽기순서 모두 우수
약점: 셋업 복잡, 대용량 모델 필요
라이선스: AGPL-3.0
언어: Python
```

- PDF-Extract-Kit: DocLayout-YOLO + UniMERNet + TableMaster + PaddleOCR
- 읽기 순서 재구성 우수
- PPTX/DOCX도 지원 (2024년 후반 추가)
- GitHub: `opendatalab/MinerU`

#### D. Nougat (Meta, 논문 특화)

```text
강점: 학술 논문 LaTeX 수식 최고 정확도
약점: 학술 논문 외 일반 문서에는 과적합, 느림
라이선스: CC-BY-NC 4.0
언어: Python
```

- Transformer 기반 end-to-end PDF → MMD
- 수식 LaTeX 출력 품질 최고
- 제한: 스캔 품질 낮으면 성능 저하

#### 선택 가이드

```text
학술 논문/기술 문서 → MinerU (기본) + Nougat (수식 보완)
일반 비즈니스 문서 → Marker
표 정확도 최우선   → Docling
빠른 배치 처리     → Marker (GPU) 또는 Surya 단독
한국어 문서        → MinerU + PaddleOCR (언어 설정)
```

### 4-6. 표 전문 복원

| 도구 | 방식 | 특징 |
|---|---|---|
| **TableTransformer (Microsoft)** | 이미지 → 표 구조 | 병합 셀, 헤더 행 감지 |
| **TableMaster (PaddlePaddle)** | OCR + 구조 동시 | MinerU 내장 |
| **TableFormer (IBM)** | 구조 특화 | Docling 내장 |
| **camelot-py** | 좌표 기반 | 테두리 있는 표에 강함 |
| **pdfplumber** | 텍스트 블록 클러스터링 | 테두리 없는 표 |

### 4-7. LLM 후처리 (품질 보완)

| 도구 | 역할 | 비용 |
|---|---|---|
| **Claude 3.5 Sonnet / Opus** | 오탈자 교정, 구조 재정비 | API 사용량 |
| **LlamaParse (LlamaIndex)** | LLM 기반 PDF 파싱 API | 유료 |
| **GPT-4o / Gemini Flash** | 이미지 포함 페이지 멀티모달 해석 | API 사용량 |
| **로컬 LLM (Llama 3.1 8B)** | 배치 후처리, 비용 절감 | GPU 메모리 |

### 4-8. Java/Spring 서버 통합 라이브러리

Spring Boot 서버에서 직접 사용 가능한 라이브러리:

| 라이브러리 | 역할 |
|---|---|
| **Apache PDFBox 3.x** | PDF 텍스트 추출, 이미지 렌더링, 기본 레이아웃 |
| **Apache POI XSLF** | PPTX 파싱 |
| **Apache Tika** | 포맷 감지, 기본 텍스트 추출 (100+ 포맷) |
| **iText 7 / OpenPDF** | PDF 조작, 텍스트 추출 보완 |
| **Tessj4 (Tesseract Java)** | JVM에서 Tesseract OCR 호출 |

> **아키텍처 권장:** 논문 수준 변환은 Python 마이크로서비스로 분리하고, Spring이 REST/gRPC로 호출.  
> JVM 단독으로는 L4 수준 달성 불가 (딥러닝 모델 불필요).

---

## 5. 권장 아키텍처: Python 파싱 마이크로서비스

```text
[Spring Boot API]
       │  REST /api/documents/parse
       ▼
[Python Parsing Service (FastAPI)]
       │
       ├── 포맷 라우터
       │     ├── PDF → MinerU / Marker 파이프라인
       │     ├── PPTX → python-pptx + LibreOffice 경로
       │     └── DOCX → python-docx + 필요시 LibreOffice
       │
       ├── OCR Worker (PaddleOCR / Surya)
       ├── Formula Worker (UniMERNet / GOT-OCR)
       ├── Table Worker (TableTransformer)
       └── LLM Post-processor (선택)
              │
              ▼
       [마크다운 + ParseQuality JSON 반환]
```

**서비스 분리 이유:**
- GPU 리소스를 파싱 서비스에만 집중
- 모델 교체 시 Spring 코드 변경 없음
- 처리량에 따라 파싱 서비스만 수평 확장

---

## 6. 하드웨어 스펙

### 6-1. 등급별 최소 스펙

#### Tier 1 — 개발/소규모 (월 ~500건)

```
CPU: Intel Core i7-12세대 이상 / AMD Ryzen 7 5800X 이상
RAM: 32GB DDR4
GPU: NVIDIA RTX 3080 10GB 또는 RTX 4070 12GB
SSD: NVMe 500GB (모델 캐시 + 임시 처리)
OS: Ubuntu 22.04 LTS

처리 능력:
- Marker (GPU): ~2-4 페이지/초
- MinerU (GPU): ~0.5-1 페이지/초
- Tesseract (CPU): ~0.3-1 페이지/초
```

#### Tier 2 — 운영/중규모 (월 ~5,000건)

```
CPU: Intel Xeon Gold 6342 / AMD EPYC 7313 이상
RAM: 64GB DDR4 ECC
GPU: NVIDIA RTX 4090 24GB (단일) 또는 A10G 24GB
SSD: NVMe 1TB (RAID 0 권장)
OS: Ubuntu 22.04 LTS

처리 능력:
- MinerU: ~2-3 페이지/초
- Marker: ~5-8 페이지/초
- 배치 병렬 처리 (16 worker): ~30-50 페이지/초
```

#### Tier 3 — 대규모/엔터프라이즈 (월 ~50,000건 이상)

```
CPU: 듀얼 Intel Xeon Platinum 8370C / AMD EPYC 9554
RAM: 256GB DDR5 ECC
GPU: NVIDIA A100 80GB × 4 (또는 H100 80GB × 2)
스토리지: NVMe RAID 2TB + 오브젝트 스토리지 (S3 호환)
네트워크: 25GbE 이상
OS: Ubuntu 22.04 LTS

처리 능력 (A100 × 4):
- MinerU 배치: ~100-200 페이지/초
- 논문 1편(20페이지): ~5-10초
```

### 6-2. GPU VRAM별 가능 모델

| VRAM | 가능한 모델 조합 | 불가 모델 |
|---|---|---|
| **4GB** | Tesseract OCR, pdfplumber, EasyOCR (CPU) | 딥러닝 대부분 |
| **8GB** | Surya, PaddleOCR, pix2tex, Marker (경량) | MinerU, Nougat, TableTransformer 동시 로드 |
| **12GB** | Marker (전체), UniMERNet, TableTransformer, GOT-OCR | A100 수준 배치 |
| **16GB** | MinerU 기본, Docling, Nougat | 대용량 배치 |
| **24GB** | MinerU 전체 + LLM 후처리 동시 | H100급 모델 |
| **40GB+** | MinerU + Llama 3.1 8B 후처리 동시 로드 | - |
| **80GB (A100)** | 전체 파이프라인 동시 로드 + 배치 | - |

### 6-3. 모델별 VRAM 요구량

| 모델 | VRAM | 비고 |
|---|---|---|
| Tesseract 5 | 0 (CPU) | GPU 미사용 |
| PaddleOCR v4 | 2-4GB | 한국어 모델 포함 |
| Surya (OCR+Layout) | 4-6GB | float16 |
| EasyOCR | 2-3GB | |
| pix2tex (LaTeX-OCR) | 2-3GB | |
| UniMERNet | 4-6GB | |
| GOT-OCR 2.0 | 8GB | 580M 파라미터 |
| TableTransformer | 2-4GB | |
| Nougat small | 4GB | |
| Nougat base | 8GB | |
| Marker (전체) | 8-12GB | Surya + 수식 |
| MinerU (전체) | 12-16GB | 모든 서브모델 |
| Docling (전체) | 8-16GB | TableFormer 포함 |
| Llama 3.1 8B (후처리) | 16GB | float16 |
| Llama 3.1 8B (후처리) | 8GB | 4-bit quantize |

### 6-4. 클라우드 GPU 옵션 (온프레미스 대안)

| 서비스 | 인스턴스 | GPU | 가격 |
|---|---|---|---|
| AWS | `g5.2xlarge` | A10G 24GB | ~$1.01/h |
| AWS | `p3.2xlarge` | V100 16GB | ~$3.06/h |
| GCP | `a2-highgpu-1g` | A100 40GB | ~$3.67/h |
| RunPod | RTX 4090 | 24GB | ~$0.44/h |
| Lambda Labs | A100 80GB | 80GB | ~$1.99/h |

> 초기에는 RunPod/Lambda Labs 같은 GPU 클라우드를 사용하고, 일정 처리량 이상이면 온프레미스 전환이 경제적이다.

---

## 7. 처리 시간 벤치마크

A4 10페이지 논문(혼합 텍스트+표+수식, 스캔 아님) 기준:

| 파이프라인 | 하드웨어 | 처리 시간 | 품질 |
|---|---|---|---|
| PDFBox 텍스트 추출만 | CPU | < 1초 | L1 |
| pdfplumber + camelot | CPU | 2-5초 | L2 |
| Marker | RTX 4090 | 5-15초 | L3~L4 |
| MinerU | RTX 4090 | 15-40초 | L4 |
| MinerU + LLM 후처리 | RTX 4090 + API | 20-60초 | L4+ |
| Nougat base | RTX 4090 | 30-90초 | L4 (수식 최강) |
| Docling | RTX 4090 | 20-50초 | L4 |

---

## 8. 마크다운 출력 품질 기준

논문 수준으로 인정하려면 아래 기준을 모두 충족해야 한다:

| 항목 | 기준 | 측정 방법 |
|---|---|---|
| 텍스트 오탈자율 | < 0.5% | 골드셋 CER |
| 표 셀 정확도 | > 95% | 행×열 매칭률 |
| 수식 LaTeX 정확도 | > 90% | UniMERNet 검증 점수 |
| 읽기 순서 정확도 | > 98% | 문단 순서 일치율 |
| 헤더/푸터 분리 | 100% | 본문 혼입 = 0 |
| 그림 캡션 연결 | > 90% | 캡션 매칭률 |
| 제목 계층 보존 | > 95% | H1/H2/H3 깊이 일치 |

---

## 9. 한국어 문서 특이사항

| 항목 | 권장 처리 |
|---|---|
| 한국어 OCR | PaddleOCR v4 `korean` 모델 (Tesseract보다 30%↑) |
| 한영 혼합 | PaddleOCR + Surya 병행, confidence 높은 쪽 선택 |
| 한글 수식 (수식에 한글 텍스트 포함) | GOT-OCR 2.0 (한글 포함 수식 지원) |
| 한글 PDF 인코딩 | pdfminer.six로 CID폰트 매핑 처리 |
| 세로쓰기 | 지원 미흡. 별도 회전 감지 + OCR |

---

## 10. 다국어 문서 변환 전략

PDF/PPTX/DOCX는 단일 언어뿐 아니라 다국어 혼합 문서도 빈번하다.  
변환 파이프라인이 언어를 무시하면 OCR 오인식, 읽기 순서 오류, 잘못된 청킹이 발생한다.

### 10-1. 언어 감지 단계

```text
[원본 문서]
     │
     ▼
┌─────────────────────────────────────┐
│ Pre-process: Language Detection     │
│  - 선택형 텍스트 샘플 추출          │
│  - fastText LID.176 모델 적용       │
│  - 신뢰도 0.85 미만 → 혼합 처리    │
└─────────────────────────────────────┘
     │
     ├── 단일 언어 (ko / en / ja / zh / ...) → 언어 전용 경로
     └── 혼합 언어 (mixed) → 페이지/블록 단위 재감지
```

- **fastText LID.176**: 176개 언어 지원, 단일 모델, 오프라인 동작
- 스캔 PDF는 OCR 이전에 언어 감지가 불가 → 페이지 이미지에서 문자 특성(Unicode 분포)으로 CJK 여부 판정

### 10-2. 언어별 OCR 라우팅

| 감지 언어 | 1순위 OCR | 2순위 OCR | 특이사항 |
|---|---|---|---|
| **ko** (한국어) | PaddleOCR `korean` | Surya | 한영 혼합 빈번, 두 엔진 병행 권장 |
| **en** (영어) | Surya | Tesseract `eng` | 수식 있으면 GOT-OCR 추가 |
| **ja** (일본어) | PaddleOCR `japan` | EasyOCR `ja` | 히라가나/가타카나/한자 혼합 |
| **zh** (중국어 간체/번체) | PaddleOCR `ch` | EasyOCR `ch_sim` | 세로쓰기 감지 필요 |
| **mixed** | 페이지별 재감지 | Surya (다국어) | 블록 단위 confidence 비교 |

```python
class MultilingualOcrRouter:
    def route(self, page_image: Image, detected_lang: str) -> OcrResult:
        if detected_lang == "ko":
            result = self.paddle_ocr.run(page_image, lang="korean")
            if result.confidence < 0.75:
                result = self.surya_ocr.run(page_image)  # fallback
        elif detected_lang in ("ja", "zh"):
            result = self.paddle_ocr.run(page_image, lang=detected_lang)
        else:
            result = self.surya_ocr.run(page_image)
        return result
```

### 10-3. CJK 특수 처리

| 문제 | 한국어 | 일본어 | 중국어 |
|---|---|---|---|
| 인코딩 | EUC-KR / UTF-8 혼합 PDF | Shift-JIS 레거시 | GBK / UTF-8 |
| 세로쓰기 | 드묾 | 빈번 | 빈번 |
| 단어 경계 | 띄어쓰기 있음 | 없음 (형태소 분석 필요) | 없음 |
| 수식 혼합 | GOT-OCR 2.0 | GOT-OCR 2.0 | GOT-OCR 2.0 |
| 한자 처리 | 한국 한자 OCR | 일본 한자 OCR | 중국 한자 OCR |

**CJK PDF 인코딩 처리:**
```python
# pdfminer.six를 통한 CID 폰트 매핑
from pdfminer.high_level import extract_text
from pdfminer.layout import LAParams

laparams = LAParams(
    detect_vertical=True,   # 세로쓰기 감지
    all_texts=True
)
text = extract_text(pdf_path, laparams=laparams, codec='utf-8')
```

### 10-4. 혼합 언어 문서 처리

```text
[혼합 언어 페이지]
     │
     ▼
블록 단위 언어 감지 (각 텍스트 블록에 fastText 적용)
     │
     ├── 블록 A: ko → PaddleOCR korean
     ├── 블록 B: en → Surya
     ├── 블록 C: formula → GOT-OCR 2.0
     └── 블록 D: ko+en 혼합 → PaddleOCR ko + 신뢰도 필터
     │
     ▼
블록별 OCR 결과 병합 → 읽기 순서 재구성 → Markdown 조립
```

**Markdown 출력 언어 정책:**
- 원문 텍스트 언어를 그대로 유지 (번역 없음)
- 메타데이터(제목, 캡션 레이블)도 원문 언어 유지
- `detectedLanguage` 필드에 주 언어 코드 기록
- 혼합 비율 `languageMix: {ko: 0.7, en: 0.3}` 기록

### 10-5. 언어별 Markdown 후처리

| 언어 | 주의사항 |
|---|---|
| 한국어 | 조사 분리로 인한 줄바꿈 오류 보정; 한글 headings 유지 |
| 일본어 | 형태소 경계에서의 줄바꿈 보정; 세로쓰기→가로쓰기 변환 표시 |
| 중국어 | 단어 경계 없는 긴 줄 처리; 간체/번체 혼용 표시 |
| 한영 혼합 | 영문 단어 중간 줄바꿈 방지; 코드 블록은 `code fence` |

### 10-6. 크로스 링궐 검색을 위한 임베딩 준비

변환된 Markdown은 bge-m3 임베딩 전에 언어별 청킹 조정이 필요하다:

```text
언어별 토큰 밀도 차이:
  - 한국어: 평균 1.5 토큰/문자 (형태소 분리)
  - 영어:   평균 0.3 토큰/단어
  - 중국어: 평균 1.0 토큰/문자

권장 청킹 전략:
  - 한국어 문서: 문장 단위 청킹, 최대 512 bge-m3 토큰
  - 영어 문서:   단락 단위 청킹, 최대 512 토큰
  - 혼합 문서:   의미 단위 우선, 언어 경계에서 청크 분리
```

### 10-7. 다국어 파싱 서비스 API 확장

```python
POST /parse
{
  "file": <binary>,
  "targetLevel": "L3",
  "language": "auto",          # "auto" | "ko" | "en" | "ja" | "zh" | "mixed"
  "enableOcr": true,
  "enableMultilingualOcr": true,  # 블록별 언어 재감지
  "preserveOriginalLanguage": true,  # 번역 없이 원문 언어 유지
  "cjkVerticalTextDetection": true   # CJK 세로쓰기 감지
}

Response:
{
  "markdown": "...",
  "detectedLanguage": "ko",
  "languageMix": {"ko": 0.85, "en": 0.15},
  "quality": { ... },
  "perPageLanguage": ["ko", "ko", "en", "ko", "mixed"]
}
```

---

## 11. 권장 구현 조합 (실용 기준)

### 비즈니스 문서 (일반 보고서, PPT)

```
Level: L2~L3
조합: Marker + PaddleOCR (한국어)
서버: RTX 3080 10GB 또는 RTX 4070 12GB
처리량: 10-30페이지/초
```

### 학술 논문 / 기술 문서

```
Level: L4
조합: MinerU (기본) + UniMERNet (수식 보완)
서버: RTX 4090 24GB (개발) / A100 40GB (운영)
처리량: 1-5페이지/초
```

### 스캔 문서 / 인포그래픽 PDF

```
Level: L3~L4
조합: Surya (레이아웃+OCR) + TableTransformer + LLM 후처리
서버: RTX 4090 24GB
처리량: 0.5-2페이지/초
```

### 빠른 배치 (품질 L2, 속도 우선)

```
Level: L2
조합: pdfplumber + PaddleOCR + Marker (경량 모드)
서버: RTX 3080 10GB × 4 병렬
처리량: 50-100페이지/초
```

---

## 12. 시스템 통합 API

파이프라인을 서비스로 노출하는 표준 인터페이스:

```python
POST /parse
Content-Type: multipart/form-data

{
  "file": <binary>,
  "targetLevel": "L4",          # L1~L4
  "enableOcr": true,
  "enableFormula": true,
  "enableTableRestructure": true,
  "language": "ko",
  "postProcessWithLlm": false
}

Response:
{
  "markdown": "# 제목\n...",
  "quality": {
    "overallScore": 0.93,
    "textCer": 0.003,
    "tableAccuracy": 0.97,
    "formulaAccuracy": 0.91,
    "readingOrderAccuracy": 0.99,
    "ocrPagesCount": 3,
    "warnings": ["PAGE_5: 복잡한 표, 근사 처리"]
  },
  "parseTimeMs": 18400,
  "pagesProcessed": 10
}
```

---

## 13. 설치 의존성 요약

```bash
# Python 핵심
pip install marker-pdf       # Marker
pip install mineru            # MinerU (alias: magic-pdf)
pip install docling           # Docling
pip install surya-ocr         # Surya
pip install paddleocr         # PaddleOCR
pip install pdfplumber        # 표 추출 보조
pip install PyMuPDF           # 고속 PDF 렌더링
pip install camelot-py[cv]    # 복잡 표 전용
pip install unimernet         # UniMERNet 수식

# System deps (Ubuntu)
apt install ghostscript poppler-utils libgl1 tesseract-ocr tesseract-ocr-kor

# LibreOffice (PPTX/DOCX → PDF 변환)
apt install libreoffice --no-install-recommends

# CUDA (GPU 가속)
# CUDA 12.1 + cuDNN 8.9 권장
```

```xml
<!-- Java/Spring (기본 텍스트 추출용) -->
<dependency>
  <groupId>org.apache.pdfbox</groupId>
  <artifactId>pdfbox</artifactId>
  <version>3.0.2</version>
</dependency>
<dependency>
  <groupId>org.apache.tika</groupId>
  <artifactId>tika-parsers-standard-package</artifactId>
  <version>2.9.2</version>
</dependency>
```

# 문서 포맷 지원 설계

## 목표

PPTX 외 PDF, DOCX 등 다양한 문서 포맷에서도 텍스트·인포그래픽·표를 일관된 방식으로 추출한다.  
포맷별 파서를 인터페이스로 분리하여 신규 포맷 추가 시 기존 파이프라인 수정 없이 확장 가능하게 한다.

---

## 1. 지원 포맷 및 추출 능력 매트릭스

| 추출 항목 | PPTX | PDF | DOCX | 비고 |
|---|---|---|---|---|
| 본문 텍스트 | ✅ 우수 | ✅ 양호 (선택형 PDF는 직접 추출) | ✅ 우수 | |
| 표 (테이블) | ✅ 셀 단위 | ⚠️ 구조 손실 위험 (PDFBox 한계) | ✅ 셀 단위 | PDF 표는 텍스트 블록 근사 |
| 제목/헤딩 구조 | ✅ 슬라이드 제목 | ⚠️ 폰트 크기 휴리스틱 필요 | ✅ 스타일 기반 |  |
| 이미지 추출 | ✅ EMF/PNG 포함 | ✅ 내장 이미지 추출 가능 | ✅ 내장 이미지 | OCR 후처리 별도 |
| 도형/SmartArt 텍스트 | ✅ POI 직접 | ❌ 불가 (렌더링 결과만 존재) | ⚠️ 일부 DrawingML | |
| 발표자 노트 | ✅ | N/A | N/A | PPTX 전용 |
| 페이지/슬라이드 번호 | ✅ 슬라이드 인덱스 | ✅ 페이지 번호 | ✅ 섹션/페이지 | |
| 하이퍼링크 | ✅ | ✅ | ✅ | |
| 레이아웃 힌트 (좌표) | ✅ EMU 좌표 | ⚠️ 텍스트 블록 bbox 근사 | ⚠️ 제한적 | |
| 차트 구조 | ⚠️ 축/레이블만 | ❌ | ⚠️ 축/레이블만 | 완전 구조화 불가 |
| 워크플로우/SmartArt | ✅ 텍스트 + 연결 추정 | ❌ | ❌ | PPTX 강점 |
| OCR 보강 | ✅ | ✅ (스캔 PDF 포함) | ✅ | 옵션 활성화 시 |
| 멀티모달 LLM 보강 | ✅ 썸네일 기반 | ✅ 페이지 렌더링 기반 | ✅ 페이지 렌더링 기반 | |

**등급 기준:** ✅ 직접 구조 추출 가능 / ⚠️ 부분적·근사 추출 / ❌ 불가 (OCR/멀티모달 필요)

---

## 2. 포맷별 파서 세부 설계

### 2-1. PPTX Parser

**라이브러리:** `Apache POI (XSLF)`

| 추출 소스 | 처리 방식 |
|---|---|
| `XSLFSlide.getShapes()` | 도형/텍스트/SmartArt |
| `XSLFTable` | 셀 단위 평탄화 |
| `XSLFGraphicFrame` | 차트 내 텍스트 |
| `XSLFPictureShape` | 이미지 바이트 추출 → OCR |
| `XSLFNotes` | 발표자 노트 |
| `CTShape.getSpPr()` | 좌표(EMU) → pt 변환 |

**특이사항:**
- SmartArt는 텍스트 노드는 추출되나 연결 관계(edge)는 별도 휴리스틱 필요
- 마스터/레이아웃 슬라이드의 텍스트는 제외 (중복 방지)

### 2-2. PDF Parser

**라이브러리:** `Apache PDFBox 3.x` (1차), 스캔 PDF는 OCR 추가

| 추출 소스 | 처리 방식 |
|---|---|
| `PDFTextStripper` | 페이지별 텍스트 순서 보장 |
| `PDFTextStripperByArea` | 텍스트 블록 bbox 근사 |
| `PDResources.getXObjectNames()` | 내장 이미지 추출 |
| `PDAnnotation` | 링크·주석 |
| 폰트 크기/스타일 | 헤딩 추정 휴리스틱 |

**제약 및 대응:**
- **선택 불가 PDF (스캔본):** 전체 페이지를 이미지로 렌더링 후 OCR → 텍스트 추출
- **표 구조:** PDFBox로 직접 추출 불가 → 텍스트 블록 y좌표 클러스터링으로 행/열 근사. 신뢰도 낮으면 `TABLE_APPROXIMATE` 플래그
- **벡터 도형:** 텍스트가 없는 벡터 그래픽은 OCR/멀티모달만으로 대응 가능
- **DRM/암호화 PDF:** 파싱 불가, 업로드 거부 처리

**스캔 PDF 감지:**
- 텍스트 추출량이 전체 페이지의 10% 미만이면 `SCAN_DETECTED = true`
- 자동 OCR 큐 전환 (사용자 알림 포함)

### 2-3. DOCX Parser

**라이브러리:** `Apache POI (XWPF)`

| 추출 소스 | 처리 방식 |
|---|---|
| `XWPFParagraph` | 스타일(Heading1/2)로 제목 판별 |
| `XWPFTable` | 셀 단위 평탄화 |
| `XWPFPicture` | 내장 이미지 추출 → OCR |
| `XWPFFootnote/Endnote` | 각주·후주 |
| `XWPFHyperlinkRun` | 하이퍼링크 |

**페이지 개념:**
- DOCX는 렌더링 전까지 페이지가 확정되지 않으므로 **섹션 단위**를 논리 페이지로 처리
- 실제 페이지 번호가 필요하면 LibreOffice headless 렌더링 후 페이지 추출 (비용 높음, 선택 옵션)

**특이사항:**
- Heading 스타일이 없는 DOCX는 폰트 크기 + 굵기 휴리스틱으로 제목 추정
- 표 안의 중첩 표(Nested Table) 지원 (재귀 평탄화)

---

## 3. 파서 인터페이스 및 클래스 설계

### 3-1. 핵심 인터페이스

```java
/**
 * 단일 문서 파일을 파싱해 DocumentParseResult를 반환하는 최상위 계약.
 * 포맷별 구현체가 이 인터페이스를 구현한다.
 */
public interface DocumentParser {

    /** 이 파서가 처리할 수 있는 포맷인지 판단 */
    boolean supports(DocumentFormat format);

    /** 실제 파싱 실행 */
    DocumentParseResult parse(ParseContext context) throws ParseException;

    /** 파서 이름/버전 (디버그·감사 로그용) */
    ParserMeta getMeta();
}
```

```java
/**
 * 파싱에 필요한 입력 컨텍스트.
 */
public record ParseContext(
    String documentVersionId,
    DocumentFormat format,
    InputStream fileStream,
    ParseOptions options       // OCR 여부, 멀티모달 여부 등
) {}
```

```java
/**
 * 파서의 출력. 포맷에 관계없이 동일한 구조.
 */
public record DocumentParseResult(
    String documentVersionId,
    DocumentFormat format,
    int pageCount,
    List<PageParseResult> pages,
    ParserMeta parserMeta,
    ParseQuality quality
) {}
```

```java
/**
 * 슬라이드 또는 페이지 단위 파싱 결과.
 * PPTX=슬라이드, PDF/DOCX=페이지(또는 섹션)
 */
public record PageParseResult(
    int pageNo,               // 1-based, 사용자 표시용
    String pageTitle,
    List<String> bodyTexts,
    List<String> shapeTexts,
    List<TableParseResult> tables,
    List<String> chartTexts,
    String notes,
    List<ImageRegion> imageRegions,
    List<LayoutHint> layoutHints,
    List<String> ocrTexts,
    List<String> hyperlinks,
    ParseFlags flags          // SCAN_DETECTED, TABLE_APPROXIMATE 등
) {}
```

### 3-2. 포맷별 구현 클래스 계층

```text
DocumentParser (interface)
│
├── AbstractDocumentParser (abstract)
│   ├── extractImages(page) : List<ImageRegion>
│   ├── buildLayoutHints(elements) : List<LayoutHint>
│   └── detectLanguage(texts) : String
│
├── PptxDocumentParser
│   └── supports: PPTX
│
├── PdfDocumentParser
│   ├── supports: PDF
│   └── PdfScanDetector (내부)
│
├── DocxDocumentParser
│   ├── supports: DOCX
│   └── DocxPageSplitter (내부 - 섹션 기반)
│
└── (확장 예시)
    ├── HwpDocumentParser  (추후)
    └── XlsxDocumentParser (추후)
```

### 3-3. 파서 레지스트리

```java
/**
 * 포맷에 맞는 파서를 찾아주는 레지스트리.
 * 새 포맷 추가 시 구현체를 Spring @Component로 등록하면 자동 반영.
 */
@Component
public class DocumentParserRegistry {

    private final List<DocumentParser> parsers;

    public DocumentParser find(DocumentFormat format) {
        return parsers.stream()
            .filter(p -> p.supports(format))
            .findFirst()
            .orElseThrow(() -> new UnsupportedFormatException(format));
    }
}
```

### 3-4. 포맷 감지

```java
public enum DocumentFormat {
    PPTX("application/vnd.openxmlformats-officedocument.presentationml.presentation"),
    PDF("application/pdf"),
    DOCX("application/vnd.openxmlformats-officedocument.wordprocessingml.document"),
    UNKNOWN("");

    public static DocumentFormat detect(String filename, String mimeType) { ... }
}
```

- 파일 확장자 + MIME type + 매직 바이트(첫 4바이트) 세 가지를 조합해 판별
- 불일치 시 보수 처리 (매직 바이트 우선)

### 3-5. ParseOptions

```java
public record ParseOptions(
    boolean enableOcr,
    boolean enableMultimodal,
    boolean enableDocxPageSplit,   // DOCX 실제 페이지 분할 (LibreOffice 필요)
    int ocrDpi,                    // 기본 150
    String language                // OCR 언어 힌트 (null이면 자동 감지)
) {
    public static ParseOptions defaults() {
        return new ParseOptions(false, false, false, 150, null);
    }
}
```

### 3-6. ParseFlags

```java
public record ParseFlags(
    boolean scanDetected,           // 스캔 PDF/이미지 기반
    boolean tableApproximate,       // 표 구조 근사 처리됨
    boolean headingHeuristic,       // 제목이 스타일 아닌 휴리스틱으로 추정됨
    boolean ocrApplied,
    boolean multimodalApplied,
    boolean partialParseFailed,     // 일부 페이지 파싱 실패
    List<Integer> failedPageNos     // 실패 페이지 번호
) {}
```

### 3-7. ParseQuality

```java
public record ParseQuality(
    double overallScore,         // 0.0~1.0
    int textCharCount,
    int tableCount,
    int imageCount,
    int ocrPageCount,
    String dominantLanguage,
    List<String> qualityWarnings // 예: "PAGE_3: 텍스트 없음, OCR 적용"
) {}
```

---

## 4. 포맷별 알려진 제약 및 권장 대응

| 상황 | 포맷 | 현상 | 권장 대응 |
|---|---|---|---|
| 스캔본 PDF | PDF | 텍스트 0건 | 자동 OCR 전환 + 사용자 알림 |
| 암호화 파일 | PDF/DOCX | 파싱 불가 | 업로드 거부, 오류 메시지 |
| 이미지 전용 슬라이드 | PPTX/DOCX | 텍스트 없음 | OCR 적용 또는 멀티모달 보강 |
| SmartArt 연결 관계 | PPTX | 노드 텍스트만 추출 | 텍스트 기반 workflow 추정 |
| 복잡한 표 (PDF) | PDF | 행/열 구조 손실 | `TABLE_APPROXIMATE` 플래그, 멀티모달 보강 권고 |
| 섹션 없는 DOCX | DOCX | 페이지 분할 불가 | 단일 페이지로 처리 또는 LibreOffice 렌더링 |
| 비정형 폰트 DOCX | DOCX | 제목 추정 실패 | `HEADING_HEURISTIC` 플래그, 사용자 검수 |
| 대용량 PPTX (100장↑) | PPTX | 메모리 압박 | 슬라이드 청크 단위 스트리밍 파싱 |

---

## 5. 포맷 확장 가이드

새 포맷(예: HWP, XLSX)을 추가하는 절차:

1. `DocumentFormat` enum에 새 항목 추가 (MIME type 포함)
2. `AbstractDocumentParser` 를 상속한 `XxxDocumentParser` 구현
3. `supports()` 에 새 포맷 코드 반환
4. `parse()` 에서 포맷 전용 라이브러리로 `PageParseResult` 목록 생성
5. Spring `@Component` 로 등록 → `DocumentParserRegistry` 자동 반영
6. `ParseFlags` 에 포맷 특이 플래그 필요 시 추가
7. `20-document-format-support.md` 매트릭스 업데이트

기존 파이프라인(`IngestionOrchestrator`, `SlideIrBuilder`, `EmbeddingClient` 등)은 **수정 없이** 신규 파서를 수용한다.

---

## 6. 업로드 허용 포맷 설정

```yaml
app:
  upload:
    allowed-formats: PPTX, PDF, DOCX
    max-file-size: 100MB
    scan-pdf-auto-ocr: true        # 스캔 PDF 자동 OCR 전환
    docx-page-split: false         # LibreOffice 기반 실제 페이지 분할 (비용 고려)
```

- 허용 포맷은 설정으로만 제어 → 배포 없이 추가 가능
- 실험적 포맷은 `experimental-formats: HWP` 로 분리 관리

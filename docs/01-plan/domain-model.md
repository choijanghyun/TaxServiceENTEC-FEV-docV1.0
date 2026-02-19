# TaxServiceENTEC-FEV Domain Model

> **Project**: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
> **Version**: 1.0.0
> **Date**: 2026-02-19
> **Purpose**: 도메인 엔티티, 관계, 집합체(Aggregate), 바운디드 컨텍스트 정의

---

## 1. Domain Overview

```
┌───────────────────────────────────────────────────────────────────┐
│                     FEV Domain Model                              │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐    │
│  │  File Input   │───▶│  Validation  │───▶│   Reporting      │    │
│  │  Context      │    │  Context     │    │   Context        │    │
│  │              │    │              │    │                  │    │
│  │ - FileUpload │    │ - Pipeline   │    │ - Result         │    │
│  │ - TaxType    │    │ - Rule       │    │ - Report         │    │
│  │ - Parser     │    │ - TaxModule  │    │ - FixGuide       │    │
│  └──────────────┘    └──────┬───────┘    └──────────────────┘    │
│                             │                                     │
│                    ┌────────▼────────┐                            │
│                    │  Rule Mgmt      │                            │
│                    │  Context        │                            │
│                    │                 │                            │
│                    │ - Layout        │                            │
│                    │ - ValidationRule│                            │
│                    │ - TaxRate       │                            │
│                    │ - Code          │                            │
│                    └─────────────────┘                            │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Cross-Cutting: Auth, Audit, Common Utilities             │    │
│  └──────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

---

## 2. Bounded Contexts (바운디드 컨텍스트)

### 2.1 File Input Context (파일 입력 컨텍스트)

| 항목 | 설명 |
|------|------|
| **책임** | 전자신고 파일 업로드, 인코딩 감지, 세목 식별, 레코드 파싱 |
| **핵심 엔티티** | ValidationRequest, ValidationFile, ParsedRecord, ParsedField |
| **패턴** | Strategy Pattern (세목별 파서 플러그인) |
| **관련 FR** | FR-001 (파일 파서 엔진) |

### 2.2 Validation Context (검증 컨텍스트)

| 항목 | 설명 |
|------|------|
| **책임** | 5단계 검증 파이프라인 실행, 세목별 전문 검증 |
| **핵심 엔티티** | ValidationPipeline, ValidationLevel, ValidationError |
| **패턴** | Chain of Responsibility (L1→L2→L3→L4→L5), Strategy (세목별 모듈) |
| **관련 FR** | FR-002 (검증 파이프라인), FR-003 (세목별 검증 모듈) |

### 2.3 Rule Management Context (규칙 관리 컨텍스트)

| 항목 | 설명 |
|------|------|
| **책임** | 레코드 레이아웃, 검증 규칙, 세율표, 공제한도, 코드 관리 |
| **핵심 엔티티** | RecordLayout, FieldLayout, ValidationRule, TaxRate, DeductionLimit, Code |
| **패턴** | Repository Pattern, 귀속연도별 버전 관리 |
| **관련 FR** | FR-004 (검증 규칙 관리) |

### 2.4 Reporting Context (리포팅 컨텍스트)

| 항목 | 설명 |
|------|------|
| **책임** | 검증 결과 요약/상세, 수정 가이드, 법적 근거, PDF/Excel 출력 |
| **핵심 엔티티** | ValidationResult, ValidationDetail, ValidationReport |
| **패턴** | Builder Pattern (리포트 생성) |
| **관련 FR** | FR-005 (리포팅), FR-007 (파일 뷰어) |

### 2.5 Cross-Cutting Concerns (횡단 관심사)

| 항목 | 설명 |
|------|------|
| **인증/인가** | JWT 기반 인증, Spring Security 권한 제어 |
| **감사 로그** | 검증 처리 로그, 규칙 변경 이력 (5년 보관) |
| **공통 유틸리티** | 체크디지트, 날짜, 인코딩, 단수처리, 마스킹 |
| **관련 FR** | FR-006 (공통 유틸리티) |

---

## 3. Core Entities (핵심 엔티티)

### 3.1 Entity Catalog

```
[Aggregate Root]        [Entity]               [Value Object]
─────────────────       ──────────────         ──────────────────
ValidationRequest       ValidationFile         TaxType
RecordLayout            FieldLayout            FormCode
ValidationRule          ValidationError        Severity
ValidationResult        ValidationDetail       ValidationLevel
                        ValidationReport       DataType
                        TaxRate                RequiredType
                        DeductionLimit         ErrorCode
                        ValidationLog          CheckDigitResult
                        RuleChangeHistory      BytePosition
```

### 3.2 Aggregate 1: Validation Request (검증 요청)

```
ValidationRequest (Aggregate Root)
│
├── ValidationFile (1:N)
│   ├── originalFileName: String
│   ├── storedFileName: String (UUID)
│   ├── fileSize: Long
│   ├── fileExtension: String
│   ├── taxType: TaxType (VO)
│   ├── encoding: String
│   ├── isMerged: Boolean
│   ├── mergedCount: Integer
│   └── status: FileStatus
│
├── requestNo: String (YYYYMMDD-NNNN)
├── userId: String
├── status: RequestStatus (PENDING/PROCESSING/COMPLETED/FAILED)
├── totalFileCount: Integer
├── completedFileCount: Integer
├── requestedAt: LocalDateTime
└── completedAt: LocalDateTime
```

**Invariants (불변 규칙)**:
- requestNo는 시스템 내 유일해야 한다
- completedFileCount <= totalFileCount
- status=COMPLETED 일 때 completedFileCount == totalFileCount
- 파일 확장자는 .101/.102/.201/.301/.401/.501/.601/.701 중 하나

### 3.3 Aggregate 2: Record Layout (레코드 레이아웃)

```
RecordLayout (Aggregate Root)
│
├── FieldLayout (1:N, ordered by fieldSeq)
│   ├── fieldSeq: Integer
│   ├── fieldName: String
│   ├── fieldStart: Integer (바이트 위치)
│   ├── fieldLength: Integer (바이트 길이)
│   ├── dataType: DataType (VO: C/N/CN)
│   ├── requiredType: RequiredType (VO: M/C/O)
│   ├── validValues: List<String>
│   ├── validationType: String (BRN/RRN/DATE 등)
│   └── description: String
│
├── taxType: TaxType (VO)
├── formCode: FormCode (VO)
├── recordType: Character (A~Z)
├── recordName: String
├── recordLength: Integer (바이트)
├── requiredYn: Boolean
├── applyStartDate: LocalDate
└── applyEndDate: LocalDate
```

**Invariants**:
- FieldLayout의 fieldStart + fieldLength <= recordLength
- FieldLayout의 fieldSeq는 연속이며 중복 불가
- 같은 taxType + formCode + recordType + applyPeriod 조합은 유일

### 3.4 Aggregate 3: Validation Rule (검증 규칙)

```
ValidationRule (Aggregate Root)
│
├── ruleId: String (예: VAT-C001)
├── taxType: TaxType (VO)
├── ruleName: String
├── ruleLevel: ValidationLevel (VO: L1~L5)
├── severity: Severity (VO: FATAL/ERROR/WARNING/INFO)
├── ruleFormula: String (검증 표현식)
├── legalBasis: String (법조항)
├── penaltyRef: String (가산세 참조)
├── errorMessage: String (템플릿, {field} 치환)
├── fixGuide: String
├── applyStartDate: LocalDate
├── applyEndDate: LocalDate
└── useYn: Boolean
```

**Invariants**:
- ruleId 형식: `{TAX_TYPE}-{LEVEL_CODE}{3DIGIT_NUMBER}` (예: VAT-C001)
- ruleLevel은 1~5 범위
- severity는 FATAL/ERROR/WARNING/INFO 중 하나
- applyStartDate <= applyEndDate (종료일이 존재할 때)

### 3.5 Aggregate 4: Validation Result (검증 결과)

```
ValidationResult (Aggregate Root)
│
├── ValidationDetail (1:N, ordered by seq)
│   ├── seq: Integer
│   ├── ruleId: String (FK → ValidationRule)
│   ├── severity: Severity (VO)
│   ├── recordType: Character
│   ├── recordSeq: Integer
│   ├── fieldName: String
│   ├── fieldPosition: BytePosition (VO)
│   ├── actualValue: String
│   ├── expectedValue: String
│   ├── errorMessage: String
│   ├── legalBasis: String
│   └── fixGuide: String
│
├── ValidationReport (1:N)
│   ├── reportType: ReportType (SUMMARY/DETAIL/PDF/EXCEL)
│   ├── reportFileName: String
│   ├── reportFilePath: String
│   ├── reportData: String (JSON)
│   └── generatedAt: LocalDateTime
│
├── fileId: Long (FK → ValidationFile)
├── mergedSeq: Integer
├── taxType: TaxType (VO)
├── brn: String (사업자등록번호)
├── taxpayerName: String
├── taxYear: String
├── taxPeriod: String
├── totalRuleCount: Integer
├── passCount: Integer
├── fatalCount: Integer
├── errorCount: Integer
├── warningCount: Integer
├── infoCount: Integer
├── overallResult: OverallResult (PASS/FAIL)
├── processingMs: Long
└── validatedAt: LocalDateTime
```

**Invariants**:
- totalRuleCount == passCount + fatalCount + errorCount + warningCount + infoCount
- overallResult == FAIL if fatalCount > 0 || errorCount > 0
- overallResult == PASS if fatalCount == 0 && errorCount == 0

---

## 4. Value Objects (값 객체)

| Value Object | 속성 | 설명 |
|-------------|------|------|
| **TaxType** | code(String), name(String), fileExt(String) | 세목 (VAT, CIT, INC, WHT, CGT) |
| **FormCode** | code(String), name(String), recordType(Char) | 서식코드 (V101001 등) |
| **Severity** | code(String) | FATAL / ERROR / WARNING / INFO |
| **ValidationLevel** | level(int), code(String), name(String) | L1~L5 검증 레벨 |
| **DataType** | code(String) | C(문자) / N(숫자) / CN(혼합) |
| **RequiredType** | code(String) | M(필수) / C(조건부) / O(선택) |
| **ErrorCode** | code(String), taxType(String), levelCode(String), number(int) | 오류코드 (VAT-C001) |
| **BytePosition** | start(int), length(int) | 필드 바이트 위치 |
| **CheckDigitResult** | valid(boolean), algorithm(String) | 체크디지트 검증 결과 |
| **TruncationRule** | type(String), unit(long) | 단수처리 규칙 (1000원 절사 등) |
| **TaxBracket** | from(long), to(long), rate(BigDecimal), deduction(long) | 세율 구간 |

---

## 5. Entity Relationship Diagram

```
┌─────────────────┐     ┌──────────────────┐
│  MST_TAX_TYPE   │◄────│  MST_FORM_CODE   │
│  (세목 마스터)    │ 1:N │  (서식코드)       │
└────────┬────────┘     └──────────────────┘
         │ 1:N
         ├──────────────────────────────────────┐
         │                                      │
┌────────▼────────┐     ┌──────────────────┐   │
│ MST_RECORD_     │◄────│ MST_FIELD_       │   │
│   LAYOUT        │ 1:N │   LAYOUT         │   │
│ (레코드 레이아웃) │     │ (필드 레이아웃)    │   │
└─────────────────┘     └──────────────────┘   │
                                                │
┌─────────────────┐     ┌──────────────────┐   │
│ MST_VALIDATION_ │◄────│ MST_ERROR_CODE   │   │
│   RULE          │ 1:1 │ (오류코드)        │   │
│ (검증 규칙)      │     └──────────────────┘   │
└────────┬────────┘                             │
         │                                      │
         │ Referenced by                        │
         │                                      │
┌────────▼────────┐     ┌──────────────────┐   │
│ REQ_VALIDATION  │────▶│ REQ_VALIDATION_  │   │
│ (검증 요청)      │ 1:N │   FILE           │   │
└─────────────────┘     │ (검증 파일)       │   │
                        └────────┬─────────┘   │
                                 │ 1:N          │
                        ┌────────▼─────────┐   │
                        │ RES_VALIDATION_  │   │
                        │   RESULT         │◄──┘
                        │ (검증 결과 요약)   │
                        └────┬────┬────────┘
                             │    │
                        1:N  │    │ 1:N
                             │    │
                ┌────────────▼┐  ┌▼─────────────┐
                │ RES_VALID_  │  │ RES_VALID_    │
                │   DETAIL    │  │   REPORT      │
                │ (결과 상세)  │  │ (리포트)       │
                └─────────────┘  └───────────────┘

[이력]
┌─────────────────┐     ┌──────────────────┐
│ HIS_VALIDATION_ │     │ HIS_RULE_CHANGE  │
│   LOG           │     │ (규칙 변경 이력)   │
│ (검증 로그)      │     └──────────────────┘
└─────────────────┘
```

---

## 6. Domain Services (도메인 서비스)

### 6.1 File Parsing Service

| 서비스 | 책임 | 입력 | 출력 |
|--------|------|------|------|
| `FileParserService` | 파일 파싱 총괄 (세목 식별 → 파서 선택 → 파싱) | ValidationFile | List\<ParsedRecord\> |
| `EncodingDetector` | 파일 인코딩 자동 감지 (EUC-KR / UTF-8) | byte[] | Encoding |
| `TaxTypeIdentifier` | 파일 확장자 기반 세목 자동 식별 | String (ext) | TaxType |
| `MergedFileSplitter` | 합본 파일을 개별 신고건으로 분리 | ParsedRecords | List\<FilingUnit\> |

**Strategy Pattern (세목별 파서)**:
```
interface TaxFileParser {
    List<ParsedRecord> parse(InputStream input, RecordLayout layout);
}

- VatFileParser implements TaxFileParser    // .101, .102
- CitFileParser implements TaxFileParser    // .201
- IncFileParser implements TaxFileParser    // .301
- WhtFileParser implements TaxFileParser    // .401
```

### 6.2 Validation Pipeline Service

| 서비스 | 책임 | 패턴 |
|--------|------|------|
| `ValidationPipelineService` | 5단계 파이프라인 실행 총괄 | Chain of Responsibility |
| `FormatValidator` | L1 형식검증 (확장자, 인코딩, 길이, 타입, 코드값) | Chain Link |
| `StructuralValidator` | L2 구조검증 (A/Z 존재, 순서, 건수, 필수항목) | Chain Link |
| `ContentValidator` | L3 내용검증 (체크디지트, 합계, 세액계산, 단수처리) | Chain Link |
| `LogicValidator` | L4 논리검증 (서식간 교차, 기간 정합, 상호참조) | Chain Link |
| `RegulatoryValidator` | L5 법규검증 (세율, 공제한도, 비과세, 최저한세) | Chain Link |

**Chain of Responsibility**:
```
FormatValidator → StructuralValidator → ContentValidator → LogicValidator → RegulatoryValidator
    (L1)              (L2)                 (L3)              (L4)              (L5)

- Fatal 오류 발생 시 후속 레벨 중단 (fail-fast)
- Error/Warning은 축적하고 계속 진행
```

### 6.3 Tax Module Service

| 서비스 | 세목 | 핵심 검증 |
|--------|------|---------|
| `VatValidationModule` | 부가가치세 | 매출세액(10%), 매입세액 공제, 안분계산, 합계표 교차 |
| `CitValidationModule` | 법인세 | 누진세율, 최저한세, 세무조정, 법인세-1/2 교차 |
| `IncValidationModule` | 종합소득세 | 8단계 누진세율, 인적공제, 세액공제 |
| `WhtValidationModule` | 원천세 | 소득종류별 합계, 간이세액표, 지급명세서 교차 |
| `CgtValidationModule` | 양도소득세 | 취득가액, 양도차익, 장기보유특별공제 |

### 6.4 Reporting Service

| 서비스 | 책임 |
|--------|------|
| `ValidationReportService` | 검증 결과 리포트 생성 (요약/상세) |
| `FixGuideService` | 오류별 수정 가이드 + 법적근거 연결 |
| `ReportExportService` | PDF/Excel 내보내기 |
| `FileViewerService` | 레코드/필드 시각화, 오류 위치 하이라이트 |

### 6.5 Common Utility Services

| 서비스 | 책임 |
|--------|------|
| `CheckDigitService` | BRN(사업자등록번호), RRN(주민등록번호), CRN(법인등록번호) 검증 |
| `DateUtilService` | 날짜 형식 검증, 윤년 처리, 업연도/경정청구기한 계산 |
| `TruncationService` | 과세표준 1,000원 미만 절사, 세액 원 미만 절사 |
| `MaskingService` | 주민번호 뒤 6자리, 계좌번호 마스킹 |
| `EncodingService` | EUC-KR ↔ UTF-8 변환, 바이트 단위 한글 처리 |
| `AmountFormatService` | 금액 포맷팅/절사/한글 변환 |

---

## 7. Domain Events (도메인 이벤트)

| Event | 발생 시점 | Payload |
|-------|---------|---------|
| `FileUploadedEvent` | 파일 업로드 완료 | fileId, fileName, taxType |
| `FileParsedEvent` | 파일 파싱 완료 | fileId, recordCount, isMerged |
| `ValidationStartedEvent` | 검증 시작 | validationId, fileId |
| `ValidationLevelCompletedEvent` | 레벨별 검증 완료 | resultId, level, errorCount |
| `ValidationCompletedEvent` | 전체 검증 완료 | resultId, overallResult, processingMs |
| `ReportGeneratedEvent` | 리포트 생성 완료 | reportId, reportType |
| `RuleChangedEvent` | 검증 규칙 변경 | ruleId, changeType, changedBy |
| `FileDeletedEvent` | 원본 파일 삭제 | fileId, deletedAt |

---

## 8. Data Flow (데이터 흐름)

```
[사용자]
   │
   │ ① POST /api/v1/validations (파일 업로드)
   ▼
[File Input Context]
   │
   │ ② 인코딩 감지 → 세목 식별 → 합본 분리
   │ ③ 레코드 레이아웃 로드 (Rule Mgmt Context)
   │ ④ 고정길이 필드 파싱 → ParsedRecord 생성
   ▼
[Validation Context]
   │
   │ ⑤ L1: 형식검증 (확장자, 인코딩, 길이, 타입)
   │ ⑥ L2: 구조검증 (A/Z존재, 순서, 건수, 필수)
   │ ⑦ L3: 내용검증 (체크디지트, 합계, 세액계산)
   │ ⑧ L4: 논리검증 (서식간 교차, 기간 정합성)
   │ ⑨ L5: 법규검증 (세율, 공제한도, 최저한세)
   │
   │ ← ValidationRule, TaxRate, DeductionLimit 참조
   ▼
[Reporting Context]
   │
   │ ⑩ ValidationResult 저장 (요약/상세)
   │ ⑪ 수정 가이드 + 법적 근거 연결
   │ ⑫ 리포트 생성 (PDF/Excel 선택)
   ▼
[사용자]
   │
   │ ⑬ GET /api/v1/validations/{id} (결과 조회)
   │ ⑭ GET /api/v1/validations/{id}/details (상세 오류)
   │ ⑮ GET /api/v1/validations/{id}/report (리포트 다운로드)
```

---

## 9. Architecture Pattern Mapping

### 9.1 Clean Architecture Layer Mapping

| Layer | Package | 포함 요소 |
|-------|---------|---------|
| **Presentation** | `presentation/` | Controller, Request/Response DTO, Mapper |
| **Application** | `application/` | UseCase, ApplicationService, Command/Query |
| **Domain** | `domain/` | Entity, Value Object, Domain Service, Repository Interface |
| **Infrastructure** | `infrastructure/` | JPA Entity, Repository Impl, MyBatis Mapper, File I/O, Cache |
| **Common** | `common/` | Utility, Exception, Config |

### 9.2 Design Patterns Used

| Pattern | 적용 위치 | 목적 |
|---------|---------|------|
| **Strategy** | 세목별 파서 (TaxFileParser) | 세목별 파싱 로직 분리, 신규 세목 추가 용이 |
| **Strategy** | 세목별 검증 모듈 (TaxValidationModule) | 세목별 검증 로직 분리 |
| **Chain of Responsibility** | 검증 파이프라인 (L1→L5) | 5단계 순차 검증, Fatal 시 중단 |
| **Builder** | 리포트 생성 (ReportBuilder) | 다양한 형태의 리포트 구성 |
| **Repository** | 데이터 접근 (JPA/MyBatis) | 도메인-인프라 분리 |
| **Factory** | 파서/검증모듈 생성 | 세목코드 기반 인스턴스 생성 |
| **Template Method** | 검증 레벨 공통 흐름 | 검증 전처리-실행-후처리 흐름 통일 |
| **Observer** | 도메인 이벤트 | 검증 완료 → 리포트 생성, 파일 삭제 등 |

---

## 10. Ubiquitous Language (유비쿼터스 언어)

프로젝트 전반에서 일관되게 사용하는 핵심 용어:

| 용어 | 의미 | 사용 예 |
|------|------|--------|
| **검증 요청** | 사용자가 파일을 업로드하여 검증을 시작하는 행위 | "검증 요청을 생성한다" |
| **파싱** | 전자신고 파일을 레코드/필드 단위로 분해하는 작업 | "파일을 파싱한다" |
| **파이프라인** | L1→L5까지 순차적으로 검증하는 전체 흐름 | "파이프라인을 실행한다" |
| **Fatal 중단** | L1/L2에서 치명적 오류 발생 시 후속 검증 중단 | "Fatal 오류로 파이프라인이 중단되었다" |
| **교차검증** | 서로 다른 서식 간 금액/건수 일치 여부 확인 | "매출합계표와 신고서를 교차검증한다" |
| **세율 구간** | 누진세율 적용을 위한 과세표준 범위 분류 | "법인세 세율 구간을 적용한다" |
| **수정 가이드** | 오류 수정 방법을 안내하는 메시지 | "수정 가이드를 표시한다" |
| **리포트** | 검증 결과를 문서 형태로 출력한 산출물 | "검증 리포트를 생성한다" |
| **레이아웃** | 레코드/필드의 위치와 규격을 정의한 메타데이터 | "레이아웃을 로드한다" |
| **규칙 외부화** | 검증 규칙을 코드가 아닌 DB에서 관리하는 설계 | "규칙을 DB에 외부화한다" |

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2026-02-19 | Initial domain model: 4 Bounded Contexts, 4 Aggregates, 10 Value Objects, 6 Service groups, 8 Domain Events, Architecture Pattern mapping | Claude |

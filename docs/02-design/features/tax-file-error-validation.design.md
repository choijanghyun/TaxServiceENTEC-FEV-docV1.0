# TaxFileErrorValidation Design Document

> **Summary**: 전자신고 파일 오류검증 시스템의 상세 설계서 (아키텍처, 클래스 설계, API 명세, DB 상세, 검증 파이프라인)
>
> **Project**: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
> **Version**: 1.2.0
> **Author**: Claude
> **Date**: 2026-02-19
> **Status**: Reviewed
> **Planning Doc**: [tax-file-error-validation.plan.md](../../01-plan/features/tax-file-error-validation.plan.md)

### Pipeline References

| Phase | Document | Status |
|-------|----------|--------|
| Phase 1 | [Schema Definition](../../01-plan/schema.md) | ✅ |
| Phase 1 | [Glossary](../../01-plan/glossary.md) | ✅ |
| Phase 1 | [Domain Model](../../01-plan/domain-model.md) | ✅ |
| Phase 2 | Coding Conventions | ❌ (본 문서 Section 10에 정의) |
| Phase 3 | Mockup | N/A (Backend-first) |
| Phase 4 | API Spec | ✅ (본 문서 Section 4에 정의) |

---

## 1. Overview

### 1.1 Design Goals

| 목표 | 설명 | 측정 기준 |
|------|------|---------|
| **독립 검증 엔진** | 특정 세무프로그램에 종속되지 않는 범용 파일 검증 | 5종 이상 세무프로그램 출력 파일 호환 |
| **규칙 외부화** | 세법 개정 시 코드 변경 없이 DB 데이터만으로 대응 | 규칙 추가 시 재배포 불필요 |
| **플러그인 아키텍처** | 신규 세목 추가 시 기존 코드 수정 최소화 | Strategy 패턴으로 모듈 분리 |
| **5단계 파이프라인** | 형식→구조→내용→논리→법규 순차 검증 | Fatal 시 fail-fast, 이외 누적 |
| **법적 근거 제시** | 모든 오류에 세법 조항 + 가산세 참조 + 수정 가이드 | 검증 결과 신뢰성 확보 |
| **성능** | 10MB 이하 3초, 100MB 이상 30초, 100파일 5분 | JMeter 부하 테스트 |

### 1.2 Design Principles

- **Clean Architecture**: Presentation → Application → Domain ← Infrastructure 의존 방향
- **Single Responsibility**: 파서/검증/리포팅/관리 각각 독립 모듈
- **Open/Closed**: Strategy 패턴으로 세목별 확장, 기존 코드 수정 불필요
- **Dependency Inversion**: 도메인은 외부 프레임워크에 의존하지 않음
- **Fail-Fast**: Fatal 오류 시 후속 검증 즉시 중단, Error/Warning은 누적 진행
- **Externalized Rules**: 검증 규칙/세율표/레이아웃을 DB에 외부화

---

## 2. Architecture

### 2.1 System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         FEV System Architecture                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [Client Layer]                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Web Browser  │  │  REST Client │  │  Swagger UI  │              │
│  │  (Admin UI)   │  │  (API 연동)   │  │  (API 문서)   │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                  │                  │                       │
│  ═══════╪══════════════════╪══════════════════╪══════════             │
│         │           [API Gateway]             │                       │
│  ═══════╪══════════════════╪══════════════════╪══════════             │
│                                                                      │
│  [Presentation Layer]                                                │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  REST Controllers                                     │           │
│  │  ├─ ValidationController        (파일 검증 API)        │           │
│  │  ├─ ReportController            (리포트 API)           │           │
│  │  ├─ AdminRuleController         (규칙 관리 API)        │           │
│  │  ├─ AdminFormCodeController     (서식코드 관리 API)     │           │
│  │  ├─ AdminErrorCodeController    (오류코드 관리 API)     │           │
│  │  ├─ AdminLayoutController       (레이아웃 관리 API)     │           │
│  │  ├─ AdminCodeController         (공통코드 관리 API)     │           │
│  │  ├─ AdminDeductionController    (공제한도 관리 API)     │           │
│  │  ├─ DashboardController         (모니터링 API)         │           │
│  │  └─ AuthController              (인증 API)             │           │
│  └──────────────────────────────────────────────────────┘           │
│                              │                                       │
│  [Application Layer]                                                 │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  Services / Use Cases                                  │           │
│  │  ├─ ValidationService      (검증 요청 처리 총괄)        │           │
│  │  ├─ FileParserService      (파일 파싱 총괄)             │           │
│  │  ├─ PipelineService        (파이프라인 실행)            │           │
│  │  ├─ ReportService          (리포트 생성)                │           │
│  │  ├─ RuleManagementService  (규칙 CRUD)                 │           │
│  │  └─ AuthService            (인증/인가)                  │           │
│  └──────────────────────────────────────────────────────┘           │
│                              │                                       │
│  [Domain Layer]                                                      │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  Core Business Logic                                   │           │
│  │  ├─ parser/     (파서 엔진 - Strategy)                  │           │
│  │  ├─ validator/  (검증 파이프라인 - Chain of Resp.)       │           │
│  │  ├─ taxmodule/  (세목별 검증 모듈 - Strategy)            │           │
│  │  ├─ model/      (도메인 엔티티/VO)                      │           │
│  │  └─ service/    (도메인 서비스)                          │           │
│  └──────────────────────────────────────────────────────┘           │
│                              │                                       │
│  [Infrastructure Layer]                                              │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  ├─ persistence/  (JPA Entity, Repository, MyBatis)    │           │
│  │  ├─ fileio/       (파일 업로드/스트림 처리)              │           │
│  │  ├─ cache/        (Redis 캐시)                         │           │
│  │  └─ security/     (JWT, Spring Security)               │           │
│  └──────────────────────────────────────────────────────┘           │
│                              │                                       │
│  [Data Layer]                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  MySQL 8.x /  │  │  Redis       │  │  File Storage│              │
│  │  PostgreSQL   │  │  (캐시/세션)   │  │  (임시 파일)  │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
[1] 사용자 파일 업로드
    │
    ▼
[2] ValidationController.validate()
    ├── 파일 저장 (임시 경로)
    ├── REQ_VALIDATION, REQ_VALIDATION_FILE 생성
    │
    ▼
[3] FileParserService.parse()
    ├── EncodingDetector: 인코딩 감지 (EUC-KR/UTF-8)
    ├── TaxTypeIdentifier: 확장자 기반 세목 식별
    ├── MergedFileSplitter: 합본 파일 분리 (해당 시)
    ├── TaxFileParser (Strategy): 세목별 파서 선택
    │   ├── VatFileParser (.101/.102)
    │   ├── CitFileParser (.201)
    │   ├── IncFileParser (.301)
    │   └── WhtFileParser (.401)
    └── 결과: List<ParsedRecord>
    │
    ▼
[4] PipelineService.execute()
    ├── L1 FormatValidator
    │   ├── 확장자/인코딩/레코드길이/필드타입/코드값/날짜/패딩
    │   └── Fatal 시 → STOP (후속 레벨 중단)
    ├── L2 StructuralValidator
    │   ├── A/Z레코드 존재/순서/건수/필수레코드/필수항목
    │   └── Fatal 시 → STOP
    ├── L3 ContentValidator
    │   ├── 체크디지트(BRN/RRN/CRN)/합계정합/세액계산/단수처리
    │   └── Error/Warning 누적
    ├── L4 LogicValidator
    │   ├── 서식간교차/기간정합/상호참조/이중신고
    │   └── Error/Warning 누적
    └── L5 RegulatoryValidator
        ├── 세율구간/공제한도/비과세요건/최저한세/가산세
        └── Error/Warning 누적
    │
    ▼
[5] RES_VALIDATION_RESULT + RES_VALIDATION_DETAIL 저장
    │
    ▼
[6] ReportService.generate()
    ├── 요약 리포트 (통과/실패, 심각도별 건수)
    ├── 상세 오류목록 (오류코드, 위치, 파일값 vs 기대값)
    ├── 수정 가이드 + 법적 근거
    └── RES_VALIDATION_REPORT 저장
    │
    ▼
[7] 사용자에게 결과 응답 (JSON / PDF / Excel)
```

### 2.3 Component Dependencies

| Component | Depends On | Dependency Type |
|-----------|-----------|----------------|
| ValidationController | ValidationService | DI (Interface) |
| ValidationService | FileParserService, PipelineService, ReportService | DI |
| FileParserService | TaxFileParser (Strategy), RecordLayoutRepository | DI |
| PipelineService | FormatValidator~RegulatoryValidator (Chain) | DI |
| ContentValidator | TaxValidationModule (Strategy), CheckDigitService | DI |
| LogicValidator | TaxValidationModule (Strategy) | DI |
| RegulatoryValidator | TaxValidationModule (Strategy), TaxRateRepository | DI |
| ReportService | ValidationResultRepository, ReportExportService | DI |
| All Validators | ValidationRuleRepository (규칙 조회) | DI |

---

## 3. Data Model

> 전체 DDL은 [schema.md](../../01-plan/schema.md) 참조. 본 섹션에서는 JPA 엔티티 설계와 관계를 정의.
>
> **Clean Architecture Note**: 아래 JPA Entity 클래스들은 `infrastructure/persistence/entity/` 패키지에 위치합니다.
> Domain Layer에서는 JPA 어노테이션이 없는 순수 POJO 도메인 모델(`domain/model/`)을 사용하며,
> Infrastructure Layer의 Mapper를 통해 JPA Entity ↔ Domain Model 간 변환합니다.
> 이를 통해 도메인 로직이 JPA/Spring 프레임워크에 의존하지 않는 Clean Architecture 원칙을 준수합니다.

### 3.1 JPA Entity 설계 (infrastructure/persistence/entity/)

> **엔티티 명명 규칙**: Section 3에서는 편의상 `TaxType`, `ValidationRule` 등으로 표기하나,
> 실제 `infrastructure/persistence/entity/` 패키지에서는 `TaxTypeEntity`, `ValidationRuleEntity` 등
> `Entity` 접미사를 붙여 Domain POJO와 구분합니다.
>
> **BaseEntity 상속**: HIS_VALIDATION_LOG, HIS_RULE_CHANGE, RES_VALIDATION_DETAIL은
> 이력/상세 테이블로 `chg_dt`/`chg_id` 감사 컬럼이 불필요하여 BaseEntity를 상속하지 않고
> `reg_dt`/`reg_id`만 직접 관리합니다. 나머지 13개 엔티티는 BaseEntity를 상속합니다.

#### 3.1.1 BaseEntity (공통 감사 컬럼)

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "reg_dt", nullable = false, updatable = false)
    private LocalDateTime regDt;

    @CreatedBy
    @Column(name = "reg_id", nullable = false, updatable = false, length = 50)
    private String regId;

    @LastModifiedDate
    @Column(name = "chg_dt")
    private LocalDateTime chgDt;

    @LastModifiedBy
    @Column(name = "chg_id", length = 50)
    private String chgId;
}
```

#### 3.1.2 Master Entities

```java
// MST_TAX_TYPE
@Entity @Table(name = "MST_TAX_TYPE")
public class TaxType extends BaseEntity {
    @Id @Column(name = "TAX_TYPE_CD", length = 10)
    private String taxTypeCd;

    @Column(name = "TAX_TYPE_NM", nullable = false, length = 50)
    private String taxTypeNm;

    @Column(name = "TAX_TYPE_NM_EN", nullable = false, length = 50)
    private String taxTypeNmEn;

    @Column(name = "FILE_EXT", nullable = false, length = 10)
    private String fileExt;

    @Column(name = "ENCODING", nullable = false, length = 10)
    private String encoding;

    @Column(name = "SORT_ORDER", nullable = false)
    private Integer sortOrder = 0;

    @Column(name = "USE_YN", nullable = false, length = 1)
    private String useYn = "Y";
}

// MST_RECORD_LAYOUT
@Entity @Table(name = "MST_RECORD_LAYOUT")
public class RecordLayout extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "LAYOUT_ID")
    private Long layoutId;

    @Column(name = "TAX_TYPE_CD", nullable = false, length = 10)
    private String taxTypeCd;

    @Column(name = "FORM_CD", nullable = false, length = 10)
    private String formCd;

    @Column(name = "RECORD_TYPE", nullable = false, length = 1)
    private Character recordType;

    @Column(name = "RECORD_NM", nullable = false, length = 100)
    private String recordNm;

    @Column(name = "RECORD_LEN", nullable = false)
    private Integer recordLen;

    @Column(name = "APPLY_START_DT", nullable = false)
    private LocalDate applyStartDt;

    @Column(name = "REQUIRED_YN", length = 1)
    private String requiredYn = "Y";

    @Column(name = "APPLY_END_DT")
    private LocalDate applyEndDt;

    @OneToMany(mappedBy = "layout", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @OrderBy("fieldSeq ASC")
    private List<FieldLayout> fields = new ArrayList<>();
}

// MST_FIELD_LAYOUT
@Entity @Table(name = "MST_FIELD_LAYOUT")
public class FieldLayout extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "FIELD_ID")
    private Long fieldId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "LAYOUT_ID", nullable = false)
    private RecordLayout layout;

    @Column(name = "FIELD_SEQ", nullable = false)
    private Integer fieldSeq;

    @Column(name = "FIELD_NM", nullable = false, length = 100)
    private String fieldNm;

    @Column(name = "FIELD_START", nullable = false)
    private Integer fieldStart;

    @Column(name = "FIELD_LEN", nullable = false)
    private Integer fieldLen;

    @Column(name = "DATA_TYPE", nullable = false, length = 2)
    private String dataType; // C, N, CN

    @Column(name = "REQUIRED_TYPE", length = 1)
    private String requiredType; // M, C, O

    @Column(name = "VALID_VALUES", length = 500)
    private String validValues; // JSON array

    @Column(name = "VALIDATION_TYPE", length = 30)
    private String validationType; // BRN, RRN, DATE 등

    @Column(name = "DESCRIPTION", length = 500)
    private String description;
}

// MST_VALIDATION_RULE
@Entity @Table(name = "MST_VALIDATION_RULE")
public class ValidationRule extends BaseEntity {
    @Id @Column(name = "RULE_ID", length = 20)
    private String ruleId; // VAT-C001

    @Column(name = "TAX_TYPE_CD", nullable = false, length = 10)
    private String taxTypeCd;

    @Column(name = "RULE_NM", nullable = false, length = 200)
    private String ruleNm;

    @Column(name = "RULE_LEVEL", nullable = false)
    private Integer ruleLevel; // 1~5

    @Column(name = "SEVERITY", nullable = false, length = 10)
    private String severity; // FATAL, ERROR, WARNING, INFO

    @Column(name = "RULE_FORMULA", columnDefinition = "TEXT")
    private String ruleFormula;

    @Column(name = "LEGAL_BASIS", length = 200)
    private String legalBasis;

    @Column(name = "PENALTY_REF", length = 200)
    private String penaltyRef;

    @Column(name = "ERROR_MSG", nullable = false, length = 500)
    private String errorMsg;

    @Column(name = "FIX_GUIDE", length = 500)
    private String fixGuide;

    @Column(name = "APPLY_START_DT", nullable = false)
    private LocalDate applyStartDt;

    @Column(name = "APPLY_END_DT")
    private LocalDate applyEndDt;

    @Column(name = "USE_YN", length = 1)
    private String useYn = "Y";
}

// MST_TAX_RATE
@Entity @Table(name = "MST_TAX_RATE")
public class TaxRate extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "RATE_ID")
    private Long rateId;

    @Column(name = "TAX_TYPE_CD", nullable = false, length = 10)
    private String taxTypeCd;

    @Column(name = "TAX_YEAR", nullable = false, length = 4)
    private String taxYear;

    @Column(name = "BRACKET_SEQ", nullable = false)
    private Integer bracketSeq;

    @Column(name = "BASE_FROM", nullable = false)
    private Long baseFrom;

    @Column(name = "BASE_TO")
    private Long baseTo;

    @Column(name = "RATE_PERCENT", nullable = false, precision = 5, scale = 2)
    private BigDecimal ratePercent;

    @Column(name = "DEDUCTION_AMT")
    private Long deductionAmt;

    @Column(name = "DESCRIPTION", length = 200)
    private String description;
}

// MST_FORM_CODE
@Entity @Table(name = "MST_FORM_CODE")
public class FormCode extends BaseEntity {
    @Id @Column(name = "FORM_CD", length = 10)
    private String formCd;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TAX_TYPE_CD", nullable = false)
    private TaxType taxType;

    @Column(name = "FORM_NM", nullable = false, length = 200)
    private String formNm;

    @Column(name = "RECORD_TYPE", nullable = false, length = 1)
    private Character recordType;

    @Column(name = "REQUIRED_YN", nullable = false, length = 1)
    private String requiredYn = "Y";

    @Column(name = "REQUIRED_COND", length = 200)
    private String requiredCond;

    @Column(name = "APPLY_START_DT", nullable = false)
    private LocalDate applyStartDt;

    @Column(name = "APPLY_END_DT")
    private LocalDate applyEndDt;

    @Column(name = "USE_YN", nullable = false, length = 1)
    private String useYn = "Y";
}

// MST_DEDUCTION_LIMIT
@Entity @Table(name = "MST_DEDUCTION_LIMIT")
public class DeductionLimit extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "LIMIT_ID")
    private Long limitId;

    @Column(name = "TAX_TYPE_CD", nullable = false, length = 10)
    private String taxTypeCd;

    @Column(name = "TAX_YEAR", nullable = false, length = 4)
    private String taxYear;

    @Column(name = "DEDUCTION_CD", nullable = false, length = 20)
    private String deductionCd;

    @Column(name = "DEDUCTION_NM", nullable = false, length = 200)
    private String deductionNm;

    @Column(name = "LIMIT_AMT")
    private Long limitAmt;

    @Column(name = "LIMIT_RATE", precision = 5, scale = 2)
    private BigDecimal limitRate;

    @Column(name = "CONDITION_DESC", length = 500)
    private String conditionDesc;

    @Column(name = "LEGAL_BASIS", length = 200)
    private String legalBasis;
}

// MST_CODE
@Entity @Table(name = "MST_CODE")
@IdClass(CodeId.class)
public class Code extends BaseEntity {
    @Id @Column(name = "CODE_GRP", length = 20)
    private String codeGrp;

    @Id @Column(name = "CODE_CD", length = 20)
    private String codeCd;

    @Column(name = "CODE_NM", nullable = false, length = 100)
    private String codeNm;

    @Column(name = "CODE_NM_EN", length = 100)
    private String codeNmEn;

    @Column(name = "CODE_DESC", length = 200)
    private String codeDesc;

    @Column(name = "SORT_ORDER", nullable = false)
    private Integer sortOrder = 0;

    @Column(name = "PARENT_CD", length = 20)
    private String parentCd;

    @Column(name = "USE_YN", nullable = false, length = 1)
    private String useYn = "Y";
}

// MST_CODE 복합키
public class CodeId implements Serializable {
    private String codeGrp;
    private String codeCd;
}

// MST_ERROR_CODE
@Entity @Table(name = "MST_ERROR_CODE")
public class ErrorCode extends BaseEntity {
    @Id @Column(name = "ERROR_CD", length = 20)
    private String errorCd;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "RULE_ID", nullable = false)
    private ValidationRule validationRule;

    @Column(name = "ERROR_MSG_TEMPLATE", nullable = false, length = 500)
    private String errorMsgTemplate;

    @Column(name = "FIX_GUIDE_TEMPLATE", length = 500)
    private String fixGuideTemplate;

    @Column(name = "LEGAL_BASIS", length = 200)
    private String legalBasis;

    @Column(name = "PENALTY_INFO", length = 200)
    private String penaltyInfo;

    @Column(name = "USE_YN", nullable = false, length = 1)
    private String useYn = "Y";
}
```

#### 3.1.3 Request/Result Entities

```java
// REQ_VALIDATION
@Entity @Table(name = "REQ_VALIDATION")
public class ValidationRequest extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "VALIDATION_ID")
    private Long validationId;

    @Column(name = "REQUEST_NO", nullable = false, unique = true, length = 20)
    private String requestNo;

    @Column(name = "USER_ID", nullable = false, length = 50)
    private String userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "STATUS", nullable = false, length = 10)
    private RequestStatus status = RequestStatus.PENDING;

    @Column(name = "TOTAL_FILE_CNT", nullable = false)
    private Integer totalFileCnt;

    @Column(name = "COMPLETED_FILE_CNT", nullable = false)
    private Integer completedFileCnt = 0;

    @Column(name = "REQUEST_DT", nullable = false)
    private LocalDateTime requestDt;

    @Column(name = "COMPLETE_DT")
    private LocalDateTime completeDt;

    @OneToMany(mappedBy = "validationRequest", cascade = CascadeType.ALL)
    private List<ValidationFile> files = new ArrayList<>();
}

// REQ_VALIDATION_FILE
@Entity @Table(name = "REQ_VALIDATION_FILE")
public class ValidationFile extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "FILE_ID")
    private Long fileId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "VALIDATION_ID", nullable = false)
    private ValidationRequest validationRequest;

    @Column(name = "ORIGINAL_FILE_NM", nullable = false, length = 255)
    private String originalFileNm;

    @Column(name = "STORED_FILE_NM", nullable = false, length = 255)
    private String storedFileNm;

    @Column(name = "FILE_PATH", nullable = false, length = 500)
    private String filePath;

    @Column(name = "FILE_SIZE", nullable = false)
    private Long fileSize;

    @Column(name = "FILE_EXT", nullable = false, length = 10)
    private String fileExt;

    @Column(name = "TAX_TYPE_CD", length = 10)
    private String taxTypeCd;

    @Column(name = "ENCODING", length = 10)
    private String encoding;

    @Column(name = "IS_MERGED", nullable = false, length = 1)
    private String isMerged = "N";

    @Column(name = "MERGED_CNT")
    private Integer mergedCnt;

    @Enumerated(EnumType.STRING)
    @Column(name = "STATUS", nullable = false, length = 10)
    private FileStatus status = FileStatus.UPLOADED;

    @Column(name = "DELETED_AT")
    private LocalDateTime deletedAt;
}

// RES_VALIDATION_RESULT
@Entity @Table(name = "RES_VALIDATION_RESULT")
public class ValidationResult extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "RESULT_ID")
    private Long resultId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "FILE_ID", nullable = false)
    private ValidationFile validationFile;

    @Column(name = "MERGED_SEQ")
    private Integer mergedSeq;

    @Column(name = "TAX_TYPE_CD", nullable = false, length = 10)
    private String taxTypeCd;

    @Column(name = "BRN", length = 10)
    private String brn;

    @Column(name = "TAXPAYER_NM", length = 100)
    private String taxpayerNm;

    @Column(name = "TAX_YEAR", length = 4)
    private String taxYear;

    @Column(name = "TAX_PERIOD", length = 10)
    private String taxPeriod;

    @Column(name = "TOTAL_RULE_CNT", nullable = false)
    private Integer totalRuleCnt = 0;

    @Column(name = "PASS_CNT", nullable = false)
    private Integer passCnt = 0;

    @Column(name = "FATAL_CNT", nullable = false)
    private Integer fatalCnt = 0;

    @Column(name = "ERROR_CNT", nullable = false)
    private Integer errorCnt = 0;

    @Column(name = "WARNING_CNT", nullable = false)
    private Integer warningCnt = 0;

    @Column(name = "INFO_CNT", nullable = false)
    private Integer infoCnt = 0;

    @Enumerated(EnumType.STRING)
    @Column(name = "OVERALL_RESULT", nullable = false, length = 10)
    private OverallResult overallResult; // PASS, FAIL

    @Column(name = "PROCESSING_MS")
    private Long processingMs;

    @Column(name = "VALIDATED_AT", nullable = false)
    private LocalDateTime validatedAt;

    @OneToMany(mappedBy = "validationResult", cascade = CascadeType.ALL)
    @OrderBy("seq ASC")
    private List<ValidationDetail> details = new ArrayList<>();

    @OneToMany(mappedBy = "validationResult", cascade = CascadeType.ALL)
    private List<ValidationReport> reports = new ArrayList<>();
}

// RES_VALIDATION_DETAIL
@Entity @Table(name = "RES_VALIDATION_DETAIL")
public class ValidationDetail {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "DETAIL_ID")
    private Long detailId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "RESULT_ID", nullable = false)
    private ValidationResult validationResult;

    @Column(name = "SEQ", nullable = false)
    private Integer seq;

    @Column(name = "RULE_ID", nullable = false, length = 20)
    private String ruleId;

    @Column(name = "SEVERITY", nullable = false, length = 10)
    private String severity;

    @Column(name = "RECORD_TYPE", length = 1)
    private Character recordType;

    @Column(name = "RECORD_SEQ")
    private Integer recordSeq;

    @Column(name = "FIELD_NM", length = 100)
    private String fieldNm;

    @Column(name = "FIELD_POS", length = 20)
    private String fieldPos;

    @Column(name = "ACTUAL_VALUE", length = 500)
    private String actualValue;

    @Column(name = "EXPECTED_VALUE", length = 500)
    private String expectedValue;

    @Column(name = "ERROR_MSG", nullable = false, length = 500)
    private String errorMsg;

    @Column(name = "LEGAL_BASIS", length = 200)
    private String legalBasis;

    @Column(name = "FIX_GUIDE", length = 500)
    private String fixGuide;

    @Column(name = "reg_dt", nullable = false, updatable = false)
    private LocalDateTime regDt;

    @Column(name = "reg_id", nullable = false, updatable = false, length = 50)
    private String regId;
}
```

#### 3.1.4 Report & History Entities

```java
// RES_VALIDATION_REPORT
@Entity @Table(name = "RES_VALIDATION_REPORT")
public class ValidationReport extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "REPORT_ID")
    private Long reportId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "RESULT_ID", nullable = false)
    private ValidationResult validationResult;

    @Column(name = "REPORT_TYPE", nullable = false, length = 10)
    private String reportType; // SUMMARY, DETAIL, PDF, EXCEL

    @Column(name = "REPORT_FILE_NM", length = 255)
    private String reportFileNm;

    @Column(name = "REPORT_FILE_PATH", length = 500)
    private String reportFilePath;

    @Column(name = "REPORT_DATA", columnDefinition = "LONGTEXT")
    private String reportData; // JSON

    @Column(name = "GENERATED_AT", nullable = false)
    private LocalDateTime generatedAt;
}

// HIS_VALIDATION_LOG
@Entity @Table(name = "HIS_VALIDATION_LOG")
public class ValidationLog {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "LOG_ID")
    private Long logId;

    @Column(name = "VALIDATION_ID", nullable = false)
    private Long validationId;

    @Column(name = "FILE_ID")
    private Long fileId;

    @Column(name = "LOG_LEVEL", nullable = false, length = 10)
    private String logLevel; // INFO, WARN, ERROR

    @Column(name = "LOG_PHASE", nullable = false, length = 20)
    private String logPhase; // UPLOAD, PARSE, L1~L5, REPORT

    @Column(name = "LOG_MSG", nullable = false, length = 1000)
    private String logMsg;

    @Column(name = "ELAPSED_MS")
    private Long elapsedMs;

    @Column(name = "LOG_DT", nullable = false)
    private LocalDateTime logDt;

    @Column(name = "reg_dt", nullable = false, updatable = false)
    private LocalDateTime regDt;

    @Column(name = "reg_id", nullable = false, updatable = false, length = 50)
    private String regId;
}

// HIS_RULE_CHANGE
@Entity @Table(name = "HIS_RULE_CHANGE")
public class RuleChange {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "CHANGE_ID")
    private Long changeId;

    @Column(name = "RULE_ID", nullable = false, length = 20)
    private String ruleId;

    @Column(name = "CHANGE_TYPE", nullable = false, length = 10)
    private String changeType; // CREATE, UPDATE, DELETE

    @Column(name = "BEFORE_DATA", columnDefinition = "TEXT")
    private String beforeData; // JSON

    @Column(name = "AFTER_DATA", columnDefinition = "TEXT")
    private String afterData; // JSON

    @Column(name = "CHANGE_REASON", length = 500)
    private String changeReason;

    @Column(name = "CHANGED_BY", nullable = false, length = 50)
    private String changedBy;

    @Column(name = "CHANGED_AT", nullable = false)
    private LocalDateTime changedAt;

    @Column(name = "reg_dt", nullable = false, updatable = false)
    private LocalDateTime regDt;

    @Column(name = "reg_id", nullable = false, updatable = false, length = 50)
    private String regId;
}
```

### 3.2 Entity Relationships

```
MST_TAX_TYPE (1) ──── (N) MST_FORM_CODE
MST_TAX_TYPE (1) ──── (N) MST_RECORD_LAYOUT
MST_TAX_TYPE (1) ──── (N) MST_VALIDATION_RULE
MST_TAX_TYPE (1) ──── (N) MST_TAX_RATE
MST_TAX_TYPE (1) ──── (N) MST_DEDUCTION_LIMIT
MST_RECORD_LAYOUT (1) ──── (N) MST_FIELD_LAYOUT
MST_FORM_CODE (N) ──── (1) MST_TAX_TYPE
MST_VALIDATION_RULE (1) ──── (N) MST_ERROR_CODE
REQ_VALIDATION (1) ──── (N) REQ_VALIDATION_FILE
REQ_VALIDATION_FILE (1) ──── (N) RES_VALIDATION_RESULT
RES_VALIDATION_RESULT (1) ──── (N) RES_VALIDATION_DETAIL
RES_VALIDATION_RESULT (1) ──── (N) RES_VALIDATION_REPORT
REQ_VALIDATION (1) ──── (N) HIS_VALIDATION_LOG
MST_VALIDATION_RULE (1) ──── (N) HIS_RULE_CHANGE
```

---

## 4. API Specification

### 4.1 공통 규격

**Base URL**: `/api/v1`

**공통 응답 형식**:
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "timestamp": "2026-02-19T12:00:00Z"
}
```

**에러 응답 형식**:
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "VALIDATION_001",
    "message": "파일 확장자가 지원되지 않습니다.",
    "details": { "extension": ".txt", "supported": [".101",".201",".301",".401"] }
  },
  "timestamp": "2026-02-19T12:00:00Z"
}
```

**페이지네이션 응답 형식** (목록 API 공통):
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "totalElements": 150,
    "totalPages": 8,
    "page": 0,
    "size": 20,
    "first": true,
    "last": false
  },
  "timestamp": "2026-02-19T12:00:00Z"
}
```

**인증**: `Authorization: Bearer {JWT_ACCESS_TOKEN}`

**페이지네이션 Query Parameters** (목록 API 공통):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | Integer | 0 | 페이지 번호 (0-based) |
| size | Integer | 20 | 페이지 크기 (max: 100) |
| sort | String | createdAt,desc | 정렬 기준 (field,direction) |

### 4.2 Validation API (검증)

#### POST /api/v1/validations

파일 업로드 및 검증 요청 생성.

**Request**: `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| files | MultipartFile[] | Yes | 전자신고 파일 (최대 100개) |
| async | Boolean | No | 비동기 처리 여부 (default: false) |

**Response (201 Created)**:
```json
{
  "success": true,
  "data": {
    "validationId": 1,
    "requestNo": "20260219-0001",
    "status": "COMPLETED",
    "totalFileCnt": 1,
    "completedFileCnt": 1,
    "results": [
      {
        "resultId": 1,
        "fileId": 1,
        "originalFileName": "신고서_일반과세자.101",
        "taxType": "VAT",
        "taxTypeName": "부가가치세",
        "brn": "123-45-67890",
        "taxpayerName": "(주)테스트회사",
        "taxYear": "2025",
        "taxPeriod": "04",
        "overallResult": "FAIL",
        "totalRuleCnt": 45,
        "passCnt": 42,
        "fatalCnt": 0,
        "errorCnt": 2,
        "warningCnt": 1,
        "infoCnt": 0,
        "processingMs": 1250
      }
    ],
    "requestedAt": "2026-02-19T12:00:00Z",
    "completedAt": "2026-02-19T12:00:01Z"
  }
}
```

#### GET /api/v1/validations/{validationId}

검증 요청 상태 및 결과 요약 조회.

**Response (200 OK)**: (위 POST 응답과 동일 구조)

#### GET /api/v1/validations/{validationId}/results/{resultId}/details

검증 결과 상세 오류 목록 조회.

**Query Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| severity | String | 심각도 필터 (FATAL/ERROR/WARNING/INFO) |
| recordType | String | 레코드타입 필터 (A~Z) |
| page | Integer | 페이지 번호 (default: 0) |
| size | Integer | 페이지 크기 (default: 50) |

**Response (200 OK)**:
```json
{
  "success": true,
  "data": {
    "totalElements": 3,
    "content": [
      {
        "detailId": 1,
        "seq": 1,
        "ruleId": "VAT-C001",
        "ruleName": "매출세액 계산 검증",
        "severity": "ERROR",
        "recordType": "B",
        "recordSeq": 1,
        "fieldName": "세금계산서발급분-세액",
        "fieldPosition": "120-132",
        "actualValue": "5000000",
        "expectedValue": "5500000",
        "errorMessage": "매출세액이 과세표준의 10%와 일치하지 않습니다. (파일값: 5,000,000, 계산값: 5,500,000)",
        "legalBasis": "부가가치세법 제30조",
        "penaltyRef": "과소신고가산세 10% (국세기본법 제47조의3)",
        "fixGuide": "B레코드의 매출세액 필드(120~132바이트)를 과세표준 x 10%인 5,500,000으로 수정하세요."
      }
    ]
  }
}
```

#### GET /api/v1/validations/{validationId}/results/{resultId}/report

검증 리포트 다운로드.

**Query Parameters**:

| Param | Type | Description |
|-------|------|-------------|
| format | String | 출력 형식 (JSON/PDF/EXCEL), default: JSON |

**Response**: `application/json` or `application/pdf` or `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

#### POST /api/v1/validations/batch

복수 파일 일괄 검증 (비동기).

**Request**: `multipart/form-data` (files: MultipartFile[], max 100)

**Response (202 Accepted)**:
```json
{
  "success": true,
  "data": {
    "validationId": 5,
    "requestNo": "20260219-0005",
    "status": "PROCESSING",
    "totalFileCnt": 30,
    "completedFileCnt": 0,
    "statusUrl": "/api/v1/validations/5"
  }
}
```

### 4.3 Admin API (관리)

#### CRUD /api/v1/admin/rules

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/rules | 검증 규칙 목록 (필터: taxType, level, severity) |
| GET | /api/v1/admin/rules/{ruleId} | 규칙 상세 |
| POST | /api/v1/admin/rules | 규칙 등록 |
| PUT | /api/v1/admin/rules/{ruleId} | 규칙 수정 |
| DELETE | /api/v1/admin/rules/{ruleId} | 규칙 삭제 (Soft) |

**POST /api/v1/admin/rules Request**:
```json
{
  "ruleId": "VAT-C060",
  "taxTypeCd": "VAT",
  "ruleNm": "신규 매출세액 검증규칙",
  "ruleLevel": 3,
  "severity": "ERROR",
  "ruleFormula": "FIELD('신규세액') == FIELD('신규과세표준') * 0.10",
  "legalBasis": "부가가치세법 제30조",
  "errorMsg": "신규 매출세액이 불일치합니다. (파일값:{actual}, 계산값:{expected})",
  "fixGuide": "과세표준 x 10%로 수정하세요.",
  "applyStartDt": "2026-01-01"
}
```

#### CRUD /api/v1/admin/tax-rates

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/tax-rates | 세율표 목록 (필터: taxType, year) |
| POST | /api/v1/admin/tax-rates | 세율 구간 등록 |
| PUT | /api/v1/admin/tax-rates/{rateId} | 세율 수정 |
| DELETE | /api/v1/admin/tax-rates/{rateId} | 세율 삭제 |

#### CRUD /api/v1/admin/form-codes

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/form-codes | 서식코드 목록 (필터: taxType, recordType, useYn) |
| GET | /api/v1/admin/form-codes/{formCd} | 서식코드 상세 |
| POST | /api/v1/admin/form-codes | 서식코드 등록 |
| PUT | /api/v1/admin/form-codes/{formCd} | 서식코드 수정 |
| DELETE | /api/v1/admin/form-codes/{formCd} | 서식코드 삭제 (Soft) |

**POST /api/v1/admin/form-codes Request**:
```json
{
  "formCd": "V101001",
  "taxTypeCd": "VAT",
  "formNm": "일반과세자 부가가치세 신고서",
  "recordType": "A",
  "requiredYn": "Y",
  "requiredCond": null,
  "applyStartDt": "2025-01-01"
}
```

#### CRUD /api/v1/admin/error-codes

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/error-codes | 오류코드 목록 (필터: ruleId, useYn) |
| GET | /api/v1/admin/error-codes/{errorCd} | 오류코드 상세 |
| POST | /api/v1/admin/error-codes | 오류코드 등록 |
| PUT | /api/v1/admin/error-codes/{errorCd} | 오류코드 수정 |
| DELETE | /api/v1/admin/error-codes/{errorCd} | 오류코드 삭제 (Soft) |

#### CRUD /api/v1/admin/layouts

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/layouts | 레코드 레이아웃 목록 (필터: taxType, formCode) |
| GET | /api/v1/admin/layouts/{layoutId} | 레이아웃 + 필드 상세 |
| POST | /api/v1/admin/layouts | 레이아웃 등록 (필드 포함) |
| PUT | /api/v1/admin/layouts/{layoutId} | 레이아웃 수정 |

#### CRUD /api/v1/admin/deduction-limits

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/deduction-limits | 공제/감면 한도 목록 (필터: taxType, year, deductionCd) |
| GET | /api/v1/admin/deduction-limits/{limitId} | 한도 상세 |
| POST | /api/v1/admin/deduction-limits | 한도 등록 |
| PUT | /api/v1/admin/deduction-limits/{limitId} | 한도 수정 |
| DELETE | /api/v1/admin/deduction-limits/{limitId} | 한도 삭제 |

**POST /api/v1/admin/deduction-limits Request**:
```json
{
  "taxTypeCd": "INC",
  "taxYear": "2025",
  "deductionCd": "PERSONAL_BASIC",
  "deductionNm": "기본공제 (본인/배우자/부양가족)",
  "limitAmt": 1500000,
  "limitRate": null,
  "conditionDesc": "1인당 연 150만원, 경로우대 100만원 추가",
  "legalBasis": "소득세법 제50조~제51조의2"
}
```

#### CRUD /api/v1/admin/codes

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/admin/codes | 공통코드 목록 (필터: codeGrp, useYn) |
| GET | /api/v1/admin/codes/{codeGrp}/{codeCd} | 코드 상세 |
| POST | /api/v1/admin/codes | 코드 등록 |
| PUT | /api/v1/admin/codes/{codeGrp}/{codeCd} | 코드 수정 |
| DELETE | /api/v1/admin/codes/{codeGrp}/{codeCd} | 코드 삭제 (Soft) |

#### GET /api/v1/admin/dashboard

모니터링 대시보드 데이터.

**Response**:
```json
{
  "success": true,
  "data": {
    "todayStats": {
      "totalValidations": 150,
      "passRate": 72.5,
      "avgProcessingMs": 2340,
      "topErrors": [
        {"ruleId": "VAT-C001", "count": 45, "ruleName": "매출세액 계산 검증"},
        {"ruleId": "WHT-C006", "count": 30, "ruleName": "가감계 합산 오류"}
      ]
    },
    "taxTypeStats": {
      "VAT": {"total": 80, "pass": 60, "fail": 20},
      "CIT": {"total": 30, "pass": 20, "fail": 10},
      "INC": {"total": 25, "pass": 18, "fail": 7},
      "WHT": {"total": 15, "pass": 12, "fail": 3}
    }
  }
}
```

### 4.4 Auth API (인증)

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/auth/login | 로그인 (JWT Access + Refresh 발급) |
| POST | /api/v1/auth/refresh | Access Token 갱신 |
| POST | /api/v1/auth/logout | 로그아웃 (Refresh Token 무효화) |

---

## 5. Class Design (패키지 구조 상세)

### 5.1 패키지 구조

```
com.entec.fev/
├── FevApplication.java                          # Spring Boot Main
│
├── presentation/                                 # Presentation Layer
│   ├── api/                                      # REST Controllers
│   │   ├── ValidationController.java
│   │   ├── ReportController.java
│   │   └── AuthController.java
│   ├── admin/                                    # Admin Controllers
│   │   ├── AdminRuleController.java
│   │   ├── AdminFormCodeController.java
│   │   ├── AdminErrorCodeController.java
│   │   ├── AdminLayoutController.java
│   │   ├── AdminCodeController.java
│   │   ├── AdminDeductionController.java
│   │   └── DashboardController.java
│   └── dto/                                      # Request/Response DTOs
│       ├── request/
│       │   ├── ValidationRequestDto.java
│       │   ├── RuleCreateRequest.java
│       │   └── LoginRequest.java
│       └── response/
│           ├── ValidationResultResponse.java
│           ├── ValidationDetailResponse.java
│           ├── ApiResponse.java                  # 공통 응답 래퍼
│           └── ErrorResponse.java
│
├── application/                                  # Application Layer
│   ├── validation/
│   │   ├── ValidationService.java                # 검증 요청 처리 총괄
│   │   └── ValidationServiceImpl.java
│   ├── parser/
│   │   ├── FileParserService.java                # 파일 파싱 총괄
│   │   └── FileParserServiceImpl.java
│   ├── pipeline/
│   │   ├── PipelineService.java                  # 파이프라인 실행
│   │   └── PipelineServiceImpl.java
│   ├── reporting/
│   │   ├── ReportService.java                    # 리포트 생성
│   │   ├── ReportServiceImpl.java
│   │   └── ReportExportService.java              # PDF/Excel 내보내기
│   ├── management/
│   │   ├── RuleManagementService.java            # 규칙 CRUD
│   │   ├── FormCodeManagementService.java        # 서식코드 CRUD
│   │   ├── ErrorCodeManagementService.java       # 오류코드 CRUD
│   │   ├── LayoutManagementService.java          # 레이아웃 CRUD
│   │   ├── CodeManagementService.java            # 공통코드/세율 CRUD
│   │   └── DeductionLimitService.java            # 공제한도 CRUD
│   └── auth/
│       ├── AuthService.java
│       └── AuthServiceImpl.java
│
├── domain/                                       # Domain Layer
│   ├── parser/                                   # 파서 엔진 (Strategy)
│   │   ├── TaxFileParser.java                    # <<interface>>
│   │   ├── AbstractTaxFileParser.java            # Template Method
│   │   ├── VatFileParser.java                    # 부가가치세 파서
│   │   ├── CitFileParser.java                    # 법인세 파서
│   │   ├── IncFileParser.java                    # 종합소득세 파서
│   │   ├── WhtFileParser.java                    # 원천세 파서
│   │   ├── EncodingDetector.java                 # 인코딩 감지
│   │   ├── TaxTypeIdentifier.java                # 세목 식별
│   │   └── MergedFileSplitter.java               # 합본 분리
│   │
│   ├── validator/                                # 검증 파이프라인 (Chain)
│   │   ├── ValidationLevel.java                  # <<interface>> Chain Link
│   │   ├── AbstractValidationLevel.java          # 공통 처리 흐름
│   │   ├── level1/
│   │   │   └── FormatValidator.java              # L1 형식검증
│   │   ├── level2/
│   │   │   └── StructuralValidator.java          # L2 구조검증
│   │   ├── level3/
│   │   │   └── ContentValidator.java             # L3 내용검증
│   │   ├── level4/
│   │   │   └── LogicValidator.java               # L4 논리검증
│   │   └── level5/
│   │       └── RegulatoryValidator.java          # L5 법규검증
│   │
│   ├── taxmodule/                                # 세목별 검증 모듈 (Strategy)
│   │   ├── TaxValidationModule.java              # <<interface>>
│   │   ├── vat/
│   │   │   ├── VatValidationModule.java
│   │   │   ├── VatSalesValidator.java            # 매출세액 검증
│   │   │   ├── VatPurchaseValidator.java         # 매입세액 검증
│   │   │   ├── VatSummaryValidator.java          # 합계표 교차검증
│   │   │   └── VatSimplifiedValidator.java       # 간이과세자 검증
│   │   ├── cit/
│   │   │   ├── CitValidationModule.java
│   │   │   ├── CitTaxAdjustValidator.java        # 세무조정 검증
│   │   │   ├── CitFinancialValidator.java        # 재무제표 검증 (법인세-2)
│   │   │   └── CitCrossValidator.java            # 법인세-1/2 교차검증
│   │   ├── inc/
│   │   │   ├── IncValidationModule.java
│   │   │   ├── IncDeductionValidator.java        # 소득공제 검증
│   │   │   ├── IncTaxCreditValidator.java        # 세액공제 검증
│   │   │   └── IncProgressiveValidator.java      # 누진세율 검증
│   │   └── wht/
│   │       ├── WhtValidationModule.java
│   │       ├── WhtSubtotalValidator.java         # 소득종류별 소계
│   │       ├── WhtPayStatementValidator.java     # 지급명세서 교차
│   │       └── WhtYearEndValidator.java          # 연말정산 검증
│   │
│   ├── model/                                    # 도메인 엔티티/VO
│   │   ├── ParsedRecord.java                     # 파싱된 레코드
│   │   ├── ParsedField.java                      # 파싱된 필드
│   │   ├── FilingUnit.java                       # 합본 분리 후 단일 신고건
│   │   ├── ValidationError.java                  # 검증 오류 (도메인 VO)
│   │   ├── ValidationContext.java                # 검증 실행 컨텍스트
│   │   └── enums/
│   │       ├── TaxTypeCode.java                  # VAT, CIT, INC, WHT, CGT
│   │       ├── Severity.java                     # FATAL, ERROR, WARNING, INFO
│   │       ├── RequestStatus.java                # PENDING, PROCESSING, COMPLETED, FAILED
│   │       ├── FileStatus.java                   # UPLOADED, PARSING, VALIDATING, DONE, ERROR
│   │       └── OverallResult.java                # PASS, FAIL
│   │
│   └── service/                                  # 도메인 서비스
│       ├── CheckDigitService.java                # 체크디지트 검증
│       ├── TruncationService.java                # 단수처리
│       ├── DateValidationService.java            # 날짜 유효성
│       ├── TaxRateCalculator.java                # 세율 계산
│       └── MaskingService.java                   # 개인정보 마스킹
│
├── infrastructure/                               # Infrastructure Layer
│   ├── persistence/                              # DB 접근
│   │   ├── entity/                               # JPA Entities (Section 3.1 참조)
│   │   │   ├── TaxTypeEntity.java
│   │   │   ├── FormCodeEntity.java
│   │   │   ├── RecordLayoutEntity.java
│   │   │   ├── FieldLayoutEntity.java
│   │   │   ├── ValidationRuleEntity.java
│   │   │   ├── TaxRateEntity.java
│   │   │   ├── DeductionLimitEntity.java
│   │   │   ├── CodeEntity.java
│   │   │   ├── ErrorCodeEntity.java
│   │   │   ├── ValidationRequestEntity.java
│   │   │   ├── ValidationFileEntity.java
│   │   │   ├── ValidationResultEntity.java
│   │   │   ├── ValidationDetailEntity.java
│   │   │   ├── ValidationReportEntity.java
│   │   │   ├── ValidationLogEntity.java
│   │   │   └── RuleChangeEntity.java
│   │   ├── repository/                           # JPA Repositories
│   │   │   ├── TaxTypeRepository.java
│   │   │   ├── FormCodeRepository.java
│   │   │   ├── RecordLayoutRepository.java
│   │   │   ├── FieldLayoutRepository.java
│   │   │   ├── ValidationRuleRepository.java
│   │   │   ├── TaxRateRepository.java
│   │   │   ├── DeductionLimitRepository.java
│   │   │   ├── CodeRepository.java
│   │   │   ├── ErrorCodeRepository.java
│   │   │   ├── ValidationRequestRepository.java
│   │   │   ├── ValidationFileRepository.java
│   │   │   ├── ValidationResultRepository.java
│   │   │   ├── ValidationDetailRepository.java
│   │   │   ├── ValidationReportRepository.java
│   │   │   ├── ValidationLogRepository.java
│   │   │   └── RuleChangeRepository.java
│   │   ├── mapper/                               # Entity ↔ Domain 변환 (MapStruct)
│   │   │   ├── TaxTypeMapper.java
│   │   │   ├── FormCodeMapper.java
│   │   │   ├── RecordLayoutMapper.java
│   │   │   ├── FieldLayoutMapper.java
│   │   │   ├── ValidationRuleMapper.java
│   │   │   ├── TaxRateMapper.java
│   │   │   ├── DeductionLimitMapper.java
│   │   │   ├── ValidationRequestMapper.java
│   │   │   ├── ValidationResultMapper.java
│   │   │   └── ValidationReportMapper.java
│   │   └── mybatis/                              # MyBatis Mappers
│   │       ├── DashboardMapper.java              # 통계 집계 쿼리
│   │       └── ValidationSearchMapper.java       # 복합 검색 쿼리
│   ├── fileio/
│   │   ├── FileStorageService.java               # 파일 저장/삭제
│   │   └── StreamFileReader.java                 # 스트림 기반 대용량 파일 읽기
│   ├── cache/
│   │   └── RedisCacheConfig.java                 # Redis 캐시 설정
│   └── security/
│       ├── JwtTokenProvider.java                 # JWT 생성/검증
│       ├── JwtAuthenticationFilter.java          # JWT 필터
│       ├── SecurityConfig.java                   # Spring Security 설정
│       ├── ValidationAccessChecker.java          # 결과 접근제어 (Section 7.3)
│       └── EncryptionService.java                # AES-256 암호화 (Section 7.5)
│
└── common/                                       # 공통 모듈
    ├── util/
    │   ├── ByteUtils.java                        # 바이트 단위 문자열 처리
    │   ├── EucKrUtils.java                       # EUC-KR 인코딩 유틸리티
    │   └── AmountUtils.java                      # 금액 포맷팅/절사
    ├── exception/
    │   ├── FevException.java                     # 기본 예외
    │   ├── FileParseException.java               # 파일 파싱 예외
    │   ├── ValidationException.java              # 검증 예외
    │   ├── AuthenticationException.java          # 인증 예외
    │   └── GlobalExceptionHandler.java           # @RestControllerAdvice
    └── config/
        ├── AppConfig.java                        # 애플리케이션 설정
        ├── SwaggerConfig.java                    # OpenAPI 3.x 설정
        └── AuditConfig.java                      # JPA Auditing 설정
```

### 5.2 핵심 인터페이스 설계

#### 5.2.1 TaxFileParser (Strategy)

```java
public interface TaxFileParser {
    /** 지원하는 세목코드 목록 */
    List<TaxTypeCode> supportedTaxTypes();

    /** 파일 파싱 실행 */
    List<ParsedRecord> parse(InputStream input, List<RecordLayout> layouts);

    /** 합본 파일 분리 */
    List<FilingUnit> splitMergedFile(List<ParsedRecord> records);
}
```

#### 5.2.2 ValidationLevel (Chain of Responsibility)

```java
public interface ValidationLevel {
    /** 검증 레벨 (1~5) */
    int getLevel();

    /** 검증 실행, Fatal 발생 시 false 반환 (후속 중단) */
    boolean validate(ValidationContext context);

    /** 다음 레벨 설정 */
    void setNext(ValidationLevel next);
}
```

#### 5.2.3 TaxValidationModule (Strategy)

```java
public interface TaxValidationModule {
    /** 지원 세목 */
    TaxTypeCode supportedTaxType();

    /** L3 내용검증 실행 */
    List<ValidationError> validateContent(ValidationContext context);

    /** L4 논리검증 실행 */
    List<ValidationError> validateLogic(ValidationContext context);

    /** L5 법규검증 실행 */
    List<ValidationError> validateRegulatory(ValidationContext context);
}
```

### 5.3 ValidationContext (검증 컨텍스트)

```java
public class ValidationContext {
    private final Long fileId;
    private final TaxTypeCode taxType;
    private final String taxYear;
    private final String taxPeriod;
    private final List<ParsedRecord> records;
    private final List<RecordLayout> layouts;
    private final List<ValidationRule> applicableRules;
    private final List<ValidationError> errors = new ArrayList<>();
    private boolean hasFatal = false;

    /** 오류 추가 */
    public void addError(ValidationError error) {
        errors.add(error);
        if (error.getSeverity() == Severity.FATAL) {
            hasFatal = true;
        }
    }

    /** Fatal 발생 여부 */
    public boolean hasFatalError() { return hasFatal; }

    /** 특정 레코드타입의 ParsedRecord 조회 */
    public List<ParsedRecord> getRecordsByType(char type) { ... }

    /** 특정 필드값 조회 */
    public String getFieldValue(char recordType, int recordSeq, String fieldName) { ... }

    /** 특정 필드의 숫자값 조회 */
    public long getFieldNumericValue(char recordType, int recordSeq, String fieldName) { ... }
}
```

---

## 6. Error Handling

### 6.1 예외 계층 구조

```
FevException (RuntimeException)
├── FileParseException          # 파일 파싱 중 오류
│   ├── UnsupportedEncodingException  # 미지원 인코딩
│   ├── InvalidRecordException        # 잘못된 레코드
│   └── FileSizeLimitException        # 파일 크기 초과
├── ValidationException         # 검증 처리 중 오류
│   ├── RuleNotFoundException         # 규칙 미존재
│   └── LayoutNotFoundException       # 레이아웃 미존재
├── AuthenticationException     # 인증/인가 오류
│   ├── InvalidTokenException         # 토큰 유효하지 않음
│   └── AccessDeniedException         # 접근 권한 없음
└── BusinessException           # 비즈니스 로직 오류
    ├── DuplicateRequestException     # 중복 요청
    └── ResourceNotFoundException     # 리소스 미존재
```

### 6.2 HTTP 상태 코드 매핑

| Exception | HTTP Status | Error Code |
|-----------|-------------|------------|
| FileParseException | 400 | FILE_001~003 |
| FileSizeLimitException | 413 | FILE_004 |
| ValidationException | 422 | VALID_001~002 |
| AuthenticationException | 401 | AUTH_001~002 |
| AccessDeniedException | 403 | AUTH_003 |
| ResourceNotFoundException | 404 | COMMON_001 |
| DuplicateRequestException | 409 | COMMON_002 |
| 기타 | 500 | SYSTEM_001 |

### 6.3 GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(FevException.class)
    public ResponseEntity<ApiResponse<Void>> handleFevException(FevException e) {
        log.error("Business error: {}", e.getMessage());
        return ResponseEntity
            .status(e.getHttpStatus())
            .body(ApiResponse.error(e.getErrorCode(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(ApiResponse.error("VALID_003", message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleException(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.internalServerError()
            .body(ApiResponse.error("SYSTEM_001", "내부 서버 오류가 발생했습니다."));
    }
}
```

---

## 7. Security Considerations

### 7.1 인증/인가

| 항목 | 설계 |
|------|------|
| 인증 방식 | JWT (Access Token: 1h, Refresh Token: 7d) |
| 토큰 저장 | Access: 클라이언트 메모리, Refresh: HttpOnly Cookie |
| 권한 체계 | ROLE_ADMIN, ROLE_USER |
| 필터 체인 | JwtAuthenticationFilter → UsernamePasswordAuthenticationFilter |

### 7.2 API 접근 제어

| Endpoint Pattern | Required Role |
|------------------|---------------|
| `/api/v1/auth/**` | permitAll |
| `/api/v1/validations/**` | ROLE_USER, ROLE_ADMIN |
| `/api/v1/admin/**` | ROLE_ADMIN |
| `/swagger-ui/**`, `/v3/api-docs/**` | permitAll (dev/staging only) |

### 7.3 결과 접근제어 (Method-Level Security)

```java
// ValidationController.java
@GetMapping("/validations/{validationId}")
@PreAuthorize("@validationAccessChecker.isOwner(#validationId, authentication)")
public ApiResponse<ValidationResultResponse> getValidation(@PathVariable Long validationId) { ... }

@GetMapping("/validations/{validationId}/results/{resultId}/details")
@PreAuthorize("@validationAccessChecker.isOwner(#validationId, authentication)")
public ApiResponse<Page<ValidationDetailResponse>> getDetails(...) { ... }

// ValidationAccessChecker.java (infrastructure/security/)
@Component("validationAccessChecker")
public class ValidationAccessChecker {
    public boolean isOwner(Long validationId, Authentication auth) {
        String userId = auth.getName();
        return validationRequestRepository.existsByValidationIdAndUserId(validationId, userId);
    }
}
```

- ROLE_ADMIN은 모든 결과 접근 가능 (`hasRole('ADMIN') or @validationAccessChecker.isOwner(...)`)
- ROLE_USER는 본인 요청 결과만 조회 가능

### 7.4 데이터 보안

| 항목 | 조치 |
|------|------|
| 주민번호 마스킹 | 리포트 출력 시 `800101-1******` 형태로 마스킹 |
| 로그 마스킹 | 로그에 주민번호/계좌번호 출력 금지, MaskingService 적용 |
| 파일 보안 | 검증 완료 후 원본 파일 즉시 삭제 (FILE_RETENTION_SECONDS 경과 시) |
| SQL Injection | JPA Parameterized Query + MyBatis #{} 사용 |
| XSS | Spring Security ContentSecurityPolicy 설정 |
| HTTPS | 운영 환경 HTTPS 강제 |

### 7.5 민감정보 암호화 (AES-256)

```java
// infrastructure/security/EncryptionService.java
@Service
public class EncryptionService {
    @Value("${encryption.aes.key}")
    private String aesKey; // 32바이트 (256bit)

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;

    /** 암호화: RRN, 계좌번호 등 민감정보 저장 시 사용 */
    public String encrypt(String plainText) { ... }

    /** 복호화: 검증 처리 시 필요한 경우만 메모리 내에서 복호화 */
    public String decrypt(String cipherText) { ... }
}
```

**암호화 대상 필드**:
- `RES_VALIDATION_RESULT.BRN`: 사업자등록번호 (검증용 → 평문 유지, 리포트 시 마스킹)
- 파일 내 주민번호/계좌번호: 파싱 후 메모리에서만 처리, DB 저장하지 않음
- 검증 결과의 `ACTUAL_VALUE`에 주민번호가 포함된 경우: 마스킹 후 저장

### 7.6 파일 업로드 보안

| 항목 | 조치 |
|------|------|
| 확장자 화이트리스트 | `.101`, `.102`, `.201`, `.301`, `.401`, `.501`만 허용 |
| MIME Type 검증 | `application/octet-stream` 또는 `text/plain`만 허용 |
| 파일 크기 제한 | 단일 파일 200MB, 총 업로드 1GB (`FILE_MAX_SIZE` 환경변수) |
| 파일명 검증 | Path Traversal 방지 (`../`, `..\\` 차단) |
| 저장 파일명 | UUID 기반 랜덤 파일명으로 저장 (원본 파일명은 DB에만 기록) |
| 임시 디렉터리 | 격리된 임시 경로 사용 (`FILE_UPLOAD_PATH`), 정기 cleanup |
| 바이러스 검사 | (Optional) ClamAV 연동 (운영 환경에서 검토) |

---

## 8. Test Plan

### 8.1 테스트 범위

| Type | Target | Tool | Coverage |
|------|--------|------|----------|
| 단위 테스트 | 체크디지트, 단수처리, 날짜검증, 세율계산 | JUnit 5 + Mockito | 90%+ |
| 단위 테스트 | 검증 규칙 로직 (세목별 검증 모듈) | JUnit 5 + Mockito | 80%+ |
| 통합 테스트 | 파이프라인 전체 흐름 (L1→L5) | @SpringBootTest + TestContainers | 주요 시나리오 100% |
| 통합 테스트 | API 엔드포인트 (Controller) | MockMvc + @WebMvcTest | 전체 API |
| 성능 테스트 | 파일 크기별 응답시간, 동시 처리 | JMeter | 3초/30초/5분 |
| 보안 테스트 | SQL Injection, XSS, 인증 우회 | OWASP ZAP, 코드 리뷰 | 취약점 0건 |
| 검증 정확성 | 실제 신고 파일 샘플 검증 결과 정합성 | 수동 + 자동 | 홈택스 일치 |

### 8.2 핵심 테스트 케이스

#### 파서 엔진 테스트

- [ ] `.101` 부가세 일반과세자 파일 파싱 성공
- [ ] `.201` 법인세 파일 파싱 성공
- [ ] `.301` 종합소득세 파일 파싱 성공
- [ ] `.401` 원천세 파일 파싱 성공
- [ ] 합본 파일 (5개 거래처) 분리 파싱 성공
- [ ] EUC-KR 인코딩 파일 정상 파싱
- [ ] UTF-8 인코딩 파일 정상 파싱
- [ ] 한글 2byte (EUC-KR) 바이트 계산 정확성
- [ ] 미지원 확장자 `.txt` → FileParseException
- [ ] 100MB 파일 스트림 기반 파싱 (OOM 없음)

#### L1~L5 검증 테스트

- [ ] L1: 레코드 길이 불일치 → Fatal
- [ ] L1: 숫자 필드에 문자 → Fatal
- [ ] L2: A 레코드 누락 → Fatal
- [ ] L2: Z 레코드 건수 불일치 → Fatal
- [ ] L3: 사업자등록번호 체크디지트 실패 → Fatal
- [ ] L3: 매출세액 ≠ 과세표준 × 10% → Error
- [ ] L3: 과세표준 1,000원 미만 미절사 → Warning
- [ ] L4: 매출합계표 ↔ 신고서 금액 불일치 → Error
- [ ] L5: 간이과세 부가가치율 오적용 → Error
- [ ] Fatal 발생 후 후속 레벨 중단 확인
- [ ] Error/Warning 누적 후 계속 진행 확인

#### 세목별 검증 테스트

- [ ] VAT: 매출세액/매입세액/납부세액/합계표 교차 (VAT-C001~C053, VAT-L001~L008)
- [ ] CIT: 당기순이익 교차/세무조정/최저한세 (CIT-C001~C010, CIT-L001~L006)
- [ ] CIT2: 재무제표 대차균형/손익계산 (CIT2-E001~E040)
- [ ] INC: 종합소득합산/누진세율/인적공제 (INC-C001~C006, INC-L001~L010)
- [ ] WHT: 소득종류별 소계/지급명세서 교차 (WHT-C001~C009, WHT-L001~L006)

#### 보안 테스트

- [ ] JWT 만료 토큰으로 API 호출 → 401
- [ ] 타인의 검증 결과 조회 시도 → 403 (@PreAuthorize 접근제어)
- [ ] ROLE_ADMIN은 타인 검증 결과 조회 가능
- [ ] 리포트에 주민번호 뒤 6자리 마스킹 확인
- [ ] 로그에 개인정보 출력 없음 확인
- [ ] AES-256 암호화/복호화 정상 동작 (EncryptionService)
- [ ] 암호화 키 없이 서비스 기동 시 실패 확인
- [ ] 미허용 확장자(.txt, .exe) 업로드 시 400 에러
- [ ] Path Traversal 시도 (../../../etc) 차단 확인
- [ ] 200MB 초과 파일 업로드 시 413 에러

---

## 9. Clean Architecture

### 9.1 Layer Structure

| Layer | Package | Responsibility |
|-------|---------|----------------|
| **Presentation** | `presentation/` | REST Controller, DTO, 요청/응답 변환 |
| **Application** | `application/` | Use Case 조립, 트랜잭션 관리, Service 인터페이스 |
| **Domain** | `domain/` | 핵심 비즈니스 로직, Entity, VO, Domain Service |
| **Infrastructure** | `infrastructure/` | JPA Entity, Repository 구현, File I/O, Cache, Security |
| **Common** | `common/` | Utility, Exception, Config |

### 9.2 Dependency Rules

```
┌──────────────────────────────────────────────────┐
│                 Dependency Direction               │
├──────────────────────────────────────────────────┤
│                                                    │
│   Presentation ──→ Application ──→ Domain          │
│                         │              ↑           │
│                         └──→ Infrastructure ──→ Domain │
│                                                    │
│   Domain은 외부 프레임워크(Spring, JPA)에 의존하지 않음  │
│   Infrastructure만 JPA/MyBatis 등 외부 기술 사용      │
│                                                    │
└──────────────────────────────────────────────────┘
```

### 9.3 Domain Model ↔ JPA Entity 매핑 전략

```
[Domain Layer]                          [Infrastructure Layer]
domain/model/                           infrastructure/persistence/entity/
├── ParsedRecord.java (POJO)           ├── TaxTypeEntity.java (@Entity)
├── ParsedField.java (POJO)            ├── RecordLayoutEntity.java (@Entity)
├── ValidationError.java (VO)          ├── ValidationRuleEntity.java (@Entity)
├── ValidationContext.java             ├── ValidationResultEntity.java (@Entity)
└── enums/                              └── ...
    ├── TaxTypeCode.java
    └── Severity.java

infrastructure/persistence/mapper/      # Entity ↔ Domain 변환
├── TaxTypeMapper.java                  # TaxTypeEntity → TaxType (도메인)
├── ValidationRuleMapper.java
└── ValidationResultMapper.java
```

**변환 규칙**:
- Domain → Infrastructure: Service에서 Repository 호출 전 Mapper로 변환
- Infrastructure → Domain: Repository 반환값을 Mapper로 도메인 모델 변환
- Domain Layer의 모델은 `@Entity`, `@Column` 등 JPA 어노테이션을 사용하지 않음
- MapStruct 또는 수동 Mapper로 구현 (MapStruct 권장)

### 9.4 Import Rules

| From | Can Import | Cannot Import |
|------|-----------|---------------|
| Presentation | Application, Domain, Common | Infrastructure directly |
| Application | Domain, Infrastructure (via Interface), Common | Presentation |
| Domain | Common only | Application, Infrastructure, Presentation |
| Infrastructure | Domain, Common | Application, Presentation |

---

## 10. Coding Convention

### 10.1 Naming Conventions

| Target | Rule | Example |
|--------|------|---------|
| 패키지 | lowercase | `com.entec.fev.domain.parser` |
| 클래스 | PascalCase | `VatFileParser`, `ValidationService` |
| 인터페이스 | PascalCase (접미사 없음) | `TaxFileParser`, `ValidationLevel` |
| 구현 클래스 | PascalCase + Impl 또는 구체 이름 | `VatFileParser`, `ValidationServiceImpl` |
| 메서드 | camelCase | `parseFile()`, `validateContent()` |
| 상수 | UPPER_SNAKE_CASE | `MAX_FILE_SIZE`, `DEFAULT_ENCODING` |
| DB 테이블 | UPPER_SNAKE_CASE + 접두어 | `MST_TAX_TYPE`, `REQ_VALIDATION` |
| DB 컬럼 | UPPER_SNAKE_CASE (비즈니스) / lower_snake_case (감사) | `TAX_TYPE_CD`, `reg_dt` |
| API URL | kebab-case, 복수형 명사 | `/api/v1/validations`, `/api/v1/admin/tax-rates` |
| 환경변수 | UPPER_SNAKE_CASE | `DB_URL`, `JWT_SECRET` |
| 오류코드 | `{세목}-{레벨}{번호}` | `VAT-C001`, `CIT-L002` |

### 10.2 코딩 규칙

| 규칙 | 설명 |
|------|------|
| 클래스 주석 | 목적, 역할, 작성자, 생성/수정 일자 |
| 메서드 주석 | 역할, Parameter, Return, Exception |
| 단일 책임 | 함수/클래스는 한 가지 역할만, 50라인 초과 시 분리 |
| 하드코딩 금지 | DB URL, API 키, 파일 경로 → `.env` / `application.yml` |
| Null 체크 | `@NonNull`, Optional 활용, 빈값 체크 필수 |
| 예외 처리 | try-catch 예외 로그 기록, 커스텀 예외 사용 |
| 보안 | 로그에 개인정보 마스킹, Prepared Statement 필수 |
| 테스트 | 순수 함수 형태 설계, 단위 테스트 용이한 구조 |

### 10.3 Git Commit Convention

```
[타입] 제목

타입:
  [feat]     : 새 기능 추가
  [fix]      : 버그 수정
  [refactor] : 리팩토링
  [docs]     : 문서 변경
  [test]     : 테스트 추가/수정
  [chore]    : 빌드/설정 변경

예시:
  [feat] 부가가치세 매출세액 검증 규칙 구현
  [fix] 체크디지트 9번째 자리 가중치 계산 오류 수정
  [test] VatSalesValidator 단위 테스트 추가
```

### 10.4 환경변수

| Variable | Purpose | Default |
|----------|---------|---------|
| `DB_URL` | DB 연결 문자열 | `jdbc:mysql://localhost:3306/fev` |
| `DB_USERNAME` | DB 계정 | `fev_user` |
| `DB_PASSWORD` | DB 비밀번호 | (필수) |
| `JWT_SECRET` | JWT 서명 키 (256bit+) | (필수) |
| `JWT_ACCESS_EXPIRATION` | Access Token 만료 (ms) | `3600000` (1h) |
| `JWT_REFRESH_EXPIRATION` | Refresh Token 만료 (ms) | `604800000` (7d) |
| `FILE_UPLOAD_PATH` | 파일 업로드 임시 경로 | `/tmp/fev/upload` |
| `FILE_MAX_SIZE` | 최대 업로드 파일 크기 | `200MB` |
| `FILE_RETENTION_SECONDS` | 검증 후 파일 보관 시간 | `3600` (1h) |
| `REDIS_HOST` | Redis 서버 주소 | `localhost` |
| `REDIS_PORT` | Redis 서버 포트 | `6379` |
| `ENCRYPTION_AES_KEY` | AES-256 암호화 키 (32바이트 Base64) | (필수) |
| `SPRING_PROFILES_ACTIVE` | 활성 프로파일 | `local` |

---

## 11. Implementation Guide

### 11.1 Phase 1 구현 순서 (MVP)

```
Step 1: 프로젝트 초기화 및 공통 모듈
├── 1.1 Spring Boot 프로젝트 생성 (Gradle Kotlin DSL)
├── 1.2 BaseEntity, ApiResponse, GlobalExceptionHandler
├── 1.3 Security 기본 설정 (JWT, Filter)
├── 1.4 공통 유틸리티 (ByteUtils, EucKrUtils, AmountUtils)
└── 1.5 공통 도메인 서비스 (CheckDigitService, TruncationService, DateValidationService)

Step 2: 기준정보 관리 (FR-004)
├── 2.1 MST 테이블 DDL 실행 (schema.md 참조)
├── 2.2 JPA Entity / Repository 구현
├── 2.3 Admin CRUD API (규칙, 레이아웃, 세율표, 코드)
├── 2.4 초기 데이터 (세목코드, 서식코드, 부가세 레이아웃)
└── 2.5 Redis 캐시 설정 (규칙/레이아웃/세율표)

Step 3: 파일 파서 엔진 (FR-001)
├── 3.1 TaxFileParser 인터페이스 + AbstractTaxFileParser
├── 3.2 EncodingDetector, TaxTypeIdentifier
├── 3.3 VatFileParser 구현 (.101/.102)
├── 3.4 MergedFileSplitter 구현
├── 3.5 FileStorageService, StreamFileReader
└── 3.6 파서 단위 테스트 (샘플 파일 기반)

Step 4: L1~L3 검증 파이프라인 (FR-002 L1~L3)
├── 4.1 ValidationLevel 인터페이스 + AbstractValidationLevel
├── 4.2 FormatValidator (L1) 구현
├── 4.3 StructuralValidator (L2) 구현
├── 4.4 ContentValidator (L3) 구현
├── 4.5 PipelineService (Chain 조립 + 실행)
└── 4.6 파이프라인 통합 테스트

Step 5: 부가가치세 검증 모듈 (FR-003 VAT)
├── 5.1 TaxValidationModule 인터페이스
├── 5.2 VatValidationModule 구현
│   ├── VatSalesValidator (VAT-C001~C005)
│   ├── VatPurchaseValidator (VAT-C010~C013)
│   ├── VatSummaryValidator (VAT-L001~L008)
│   └── VatSimplifiedValidator (간이과세자)
├── 5.3 부가세 검증 규칙 데이터 등록 (163건 중 65건)
└── 5.4 세목별 단위 + 통합 테스트

Step 6: 검증 결과 리포팅 (FR-005 기본)
├── 6.1 RES 테이블 DDL + JPA Entity
├── 6.2 ReportService 구현 (요약/상세/수정가이드/심각도)
├── 6.3 ValidationController + ReportController
└── 6.4 API 통합 테스트 (파일 업로드 → 검증 → 결과 조회)

Step 7: 공통 유틸리티 마무리 (FR-006)
├── 7.1 MaskingService (주민번호/계좌번호)
├── 7.2 TaxRateCalculator (세율 조회/계산)
└── 7.3 API 문서 (Swagger OpenAPI 3.x)
```

### 11.2 Phase 2 구현 순서 (세목 확장)

```
Step 8:  L4~L5 검증 (FR-002 L4~L5)
Step 9:  법인세 검증 모듈 (FR-003 CIT)
Step 10: 종합소득세 검증 모듈 (FR-003 INC)
Step 11: 원천세 검증 모듈 (FR-003 WHT)
Step 12: 양도소득세 검증 모듈 (FR-003 CGT, 개략)
Step 13: 대용량 파일 처리 / 다중 파일 (FR-001-04~05)
Step 14: 거래처 분리 리포트 / PDF/Excel (FR-005-06~07)
```

### 11.3 Phase 3 구현 순서 (고도화)

```
Step 15: 파일 뷰어 (FR-007)
Step 16: 일괄처리 (FR-008)
Step 17: 법적 근거 연결 + 가산세 위험 안내 (FR-005-04)
Step 18: 크로스 세목 검증
Step 19: 모니터링 대시보드
```

### 11.4 핵심 기술 의사결정 요약

| 결정 사항 | 선택 | 근거 |
|---------|------|------|
| 파서 패턴 | Strategy | 세목별 독립 파서, 신규 세목 추가 용이 |
| 파이프라인 패턴 | Chain of Responsibility | L1→L5 순차, Fatal 시 중단 |
| 규칙 저장 | DB (MST_VALIDATION_RULE) | 코드 변경 없이 세법 개정 대응 |
| 레이아웃 관리 | DB (MST_RECORD_LAYOUT + MST_FIELD_LAYOUT) | 파일 포맷 변경 시 데이터만 수정 |
| 캐싱 전략 | Redis (규칙/세율표/레이아웃) | DB 반복 조회 방지, TTL 1시간 |
| 파일 처리 | Stream 기반 (BufferedReader) | 대용량 파일 OOM 방지 |
| 인코딩 처리 | ICU4J 또는 juniversalchardet | EUC-KR 자동 감지 정확도 |

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2026-02-19 | Initial design: Architecture, JPA Entity, API Spec 12 endpoints, Class Design 60+ classes, 5-level Pipeline, 4 Tax Modules, Test Plan, Implementation Guide 19 steps | Claude |
| 1.1.0 | 2026-02-19 | Validation fix: +7 JPA Entities, entity-schema column sync (16개 필드), +1 Admin API (deduction-limits), Clean Architecture separation (Entity↔Domain mapper), @PreAuthorize 접근제어, AES-256 암호화 명세, 파일 업로드 보안, +6 보안 테스트 | Claude |
| 1.2.0 | 2026-02-19 | Iteration fix: +2 Admin API (form-codes, error-codes CRUD), codes DELETE 추가, Entity-Domain mapper 10개로 확장, 페이지네이션 응답 형식 표준화, ENCRYPTION_AES_KEY 환경변수, Entity 명명규칙/BaseEntity 상속 문서화, Architecture diagram Controller 동기화 | Claude |

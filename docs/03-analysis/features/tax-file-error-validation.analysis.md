# TaxFileErrorValidation Gap Analysis Report (Plan vs Design)

> **Analysis Type**: Plan-Design Gap Analysis (PDCA Check Phase)
>
> **Project**: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
> **Version**: 1.1.0
> **Analyst**: gap-detector agent
> **Date**: 2026-02-19
> **Plan Doc**: [tax-file-error-validation.plan.md](../../01-plan/features/tax-file-error-validation.plan.md)
> **Design Doc**: [tax-file-error-validation.design.md](../../02-design/features/tax-file-error-validation.design.md)

### Pipeline References

| Phase | Document | Verification Target |
|-------|----------|---------------------|
| Phase 1 | [Schema](../../01-plan/schema.md) | Data Model consistency |
| Phase 1 | [Domain Model](../../01-plan/domain-model.md) | Entity/Aggregate mapping |
| Phase 1 | [Glossary](../../01-plan/glossary.md) | Terminology consistency |
| Phase 2 | Design Section 10 | Convention compliance |
| Phase 4 | Design Section 4 | API specification |

---

## 1. Analysis Overview

### 1.1 Analysis Purpose

Plan 문서(v1.1.0)에 정의된 기능요구사항, 비기능요구사항, 사용자 스토리, 아키텍처 결정사항이 Design 문서(v1.1.0)에 충실히 반영되었는지 항목별로 검증한다. 이전 v1.0.0 분석에서 발견된 5건의 Medium 갭(DeductionLimit Entity 누락, 공제한도 Admin API 누락, 결과 접근제어 미상세, DB 암호화 테스트 누락, penaltyRef 필드 누락)이 수정되었는지도 확인한다.

### 1.2 Analysis Scope

- **Plan Document**: `docs/01-plan/features/tax-file-error-validation.plan.md` (v1.1.0)
- **Design Document**: `docs/02-design/features/tax-file-error-validation.design.md` (v1.1.0)
- **Reference Documents**: schema.md, domain-model.md, glossary.md
- **Analysis Date**: 2026-02-19

### 1.3 Previous Analysis (v1.0.0) Summary

| Item | v1.0.0 Result | v1.1.0 Status |
|------|:-------------:|:-------------:|
| Overall Match Rate | 93% | (this report) |
| Total Gaps | 19 | (this report) |
| Critical Gaps | 0 | (this report) |
| High Gaps | 0 | (this report) |
| Medium Gaps | 5 | (this report) |
| Low Gaps | 14 | (this report) |

---

## 2. v1.0.0 Medium Gap Resolution Verification

| v1.0.0 Gap | Description | v1.1.0 Resolution | Status |
|------------|-------------|-------------------|:------:|
| DeductionLimit Entity 누락 | Plan schema.md에 MST_DEDUCTION_LIMIT 정의됨, Design JPA Entity 미존재 | Design 3.1.2에 `DeductionLimit` JPA Entity 추가 (line 434~463) | RESOLVED |
| 공제한도 Admin API 누락 | Plan FR-004에 공제한도 관리 포함, Design API에 미정의 | Design 4.3에 `CRUD /api/v1/admin/deduction-limits` 추가 (line 1062~1084) | RESOLVED |
| 결과 접근제어 미상세 | Plan NFR "신청인 이외 결과 접근 불가", Design에 방법 미상세 | Design 7.3에 `@PreAuthorize` + `ValidationAccessChecker` 구현 상세 추가 (line 1503~1527) | RESOLVED |
| DB 암호화 테스트 누락 | Plan NFR "원본 파일 즉시 삭제 또는 암호화", Design 테스트에 미포함 | Design 7.5에 AES-256 암호화 상세 + 8.2에 보안 테스트 6건 추가 (line 1637~1641) | RESOLVED |
| penaltyRef 필드 누락 | Plan 가산세 체계 상세, Design ValidationRule에 penaltyRef 미정의 | Design 3.1.2 ValidationRule Entity에 `penaltyRef` 필드 추가 (line 351) | RESOLVED |

**v1.0.0 Medium Gap 해소율: 5/5 = 100%**

---

## 3. Functional Requirements (FR) Gap Analysis

### 3.1 FR-001: File Parser Engine (10 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-001-01 | 파일 확장자 기반 세목 자동 인식 | TaxTypeIdentifier (Domain, 5.1 line 1194) | Match |
| FR-001-02 | EUC-KR/UTF-8 인코딩 감지 | EncodingDetector (Domain, 5.1 line 1193) | Match |
| FR-001-03 | 고정길이 레코드 필드별 파싱 | TaxFileParser Strategy (5.2.1) + ParsedRecord/ParsedField (5.1) | Match |
| FR-001-04 | 대용량 파일 스트림 처리 | StreamFileReader (infrastructure, 5.1 line 1301), Phase 2 Step 13 | Match |
| FR-001-05 | 다중 파일 동시 처리 | Phase 2 Step 13 (11.2), batch API (4.2) | Match |
| FR-001-06 | 합본 파일 분리 | MergedFileSplitter (Domain, 5.1 line 1195) | Match |
| FR-001-07~10 | 세목별 파서 (VAT/CIT/INC/WHT) | VatFileParser, CitFileParser, IncFileParser, WhtFileParser (5.1) | Match |

**FR-001 Match: 10/10 = 100%**

### 3.2 FR-002: 5-Level Validation Pipeline (33 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-002-01~11 | L1 형식검증 (확장자, 인코딩, 레코드길이, 필드타입, 코드값, 날짜, 패딩 등) | FormatValidator (5.1 level1/), Data Flow [4] L1, Test Plan 8.2 | Match |
| FR-002-12~17 | L2 구조검증 (A/Z레코드, 순서, 건수, 필수레코드, 필수항목) | StructuralValidator (5.1 level2/), Data Flow [4] L2, Test Plan 8.2 | Match |
| FR-002-18~22 | L3 내용검증 (체크디지트, 합계정합, 세액계산, 단수처리) | ContentValidator (5.1 level3/), CheckDigitService/TruncationService (5.1), Test Plan 8.2 | Match |
| FR-002-23~28 | L4 논리검증 (서식간교차, 기간정합, 상호참조, 이중신고) | LogicValidator (5.1 level4/), Phase 2 Step 8, Test Plan 8.2 | Match |
| FR-002-29~33 | L5 법규검증 (세율구간, 공제한도, 비과세, 최저한세, 가산세) | RegulatoryValidator (5.1 level5/), TaxRateCalculator (5.1), Phase 2 Step 8, Test Plan 8.2 | Match |
| Fatal-시-중단 | Fatal 발생 시 fail-fast, Error/Warning 누적 | ValidationContext.hasFatalError() (5.3), ValidationLevel interface (5.2.2), Design Principles 1.2 | Match |

**FR-002 Match: 33/33 = 100%**

### 3.3 FR-003: Tax Module Validation

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-003-VAT | 부가가치세 (C53건, L8건, R4건 = 65건) | VatValidationModule + 4 sub-validators (5.1 taxmodule/vat/), Step 5, Test 8.2 | Match |
| FR-003-CIT | 법인세 (C10건, L6건, E40건 = 56건) | CitValidationModule + 3 sub-validators (5.1 taxmodule/cit/), Phase 2 Step 9, Test 8.2 | Match |
| FR-003-INC | 종합소득세 (C6건, L10건, T5건 = 21건) | IncValidationModule + 3 sub-validators (5.1 taxmodule/inc/), Phase 2 Step 10, Test 8.2 | Match |
| FR-003-WHT | 원천세 (C9건, L6건, Y6건 = 21건) | WhtValidationModule + 3 sub-validators (5.1 taxmodule/wht/), Phase 2 Step 11, Test 8.2 | Match |
| FR-003-CGT | 양도소득세 (개략 7건) | CgtValidationModule (domain-model.md), Phase 2 Step 12 | Match |

**FR-003 Match: 5/5 tax modules = 100%**

### 3.4 FR-004: Validation Rule Management (7 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-004-01 | 검증 규칙 DB 외부화 | MST_VALIDATION_RULE Entity (3.1.2), Admin Rules API (4.3) | Match |
| FR-004-02 | 세율표 관리 | MST_TAX_RATE Entity (3.1.2), Admin Tax-Rates API (4.3) | Match |
| FR-004-03 | 레코드 레이아웃 메타데이터 | MST_RECORD_LAYOUT + MST_FIELD_LAYOUT (3.1.2), Admin Layouts API (4.3) | Match |
| FR-004-04 | 공통코드 관리 | MST_CODE Entity (3.1.2), Admin Codes API (4.3) | Match |
| FR-004-05 | 오류코드 정의 관리 | MST_ERROR_CODE Entity (3.1.2) | Match |
| FR-004-06 | 공제/감면 한도표 관리 | MST_DEDUCTION_LIMIT Entity (3.1.2), Admin Deduction-Limits API (4.3) | Match |
| FR-004-07 | 규칙 이력 관리 | HIS_RULE_CHANGE Entity (3.1.4), RuleChangeRepository (5.1) | Match |

**FR-004 Match: 7/7 = 100%**

### 3.5 FR-005: Validation Result Reporting

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-005-01 | 요약 리포트 (통과/실패, 심각도별 건수) | RES_VALIDATION_RESULT Entity, ReportService, ValidationResultResponse | Match |
| FR-005-02 | 상세 오류목록 (오류코드, 위치, 파일값 vs 기대값) | RES_VALIDATION_DETAIL Entity, GET details API (4.2) | Match |
| FR-005-03 | 수정가이드 | FIX_GUIDE 필드 (ValidationRule/Detail), fixGuide in API response (4.2) | Match |
| FR-005-04 | 법적 근거 연결 + 가산세 위험 안내 | LEGAL_BASIS + PENALTY_REF + PENALTY_INFO 필드, legalBasis/penaltyRef in API response | Match |
| FR-005-05 | 심각도 분류 (Fatal/Error/Warning/Info) | Severity enum (5.1), severity 기반 필터링 (4.2) | Match |
| FR-005-06 | 거래처별 분리 리포트 | Phase 2 Step 14 (11.2) | Match |
| FR-005-07 | PDF/Excel 출력 | ReportExportService (5.1), GET report API format param (4.2), Phase 2 Step 14 | Match |
| FR-005-08 | 비교 리포트 | Phase 3 (11.3) 언급 | Match |

**FR-005 Match: 8/8 = 100%**

### 3.6 FR-006: Common Utilities (9 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-006-01 | 체크디지트 검증 (BRN/RRN/CRN) | CheckDigitService (domain/service/, 5.1 line 1249) | Match |
| FR-006-02 | 날짜 형식 검증, 윤년 | DateValidationService (domain/service/, 5.1 line 1251) | Match |
| FR-006-03 | EUC-KR/UTF-8 인코딩 변환 | EucKrUtils (common/util/, 5.1 line 1312), EncodingDetector | Match |
| FR-006-04 | 단수처리 (1000원 미만 절사, 원 미만 절사) | TruncationService (domain/service/, 5.1 line 1250) | Match |
| FR-006-05 | 개인정보 마스킹 (주민번호 뒤 6자리) | MaskingService (domain/service/, 5.1 line 1253) | Match |
| FR-006-06 | 금액 포맷팅 | AmountUtils (common/util/, 5.1 line 1313) | Match |
| FR-006-07 | 바이트 단위 문자열 처리 | ByteUtils (common/util/, 5.1 line 1311) | Match |
| FR-006-08 | 로그 마스킹 | MaskingService + Security Considerations 7.4 로그 마스킹 | Match |
| FR-006-09 | 세율 계산 | TaxRateCalculator (domain/service/, 5.1 line 1252) | Match |

**FR-006 Match: 9/9 = 100%**

### 3.7 FR-007: File Viewer (5 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-007-01 | 레코드/필드 시각화 | FileViewerService (domain-model.md 6.4), Phase 3 Step 15 (11.3) | Match |
| FR-007-02 | 오류 위치 하이라이트 | FileViewerService, Phase 3 Step 15 | Match |
| FR-007-03~05 | 파일 뷰어 세부 기능 | Phase 3 Step 15 (11.3) | Match |

**FR-007 Match: 5/5 = 100%**

### 3.8 FR-008: Batch Processing (4 sub-requirements)

| Sub-ID | Plan Requirement | Design Coverage | Status |
|--------|-----------------|-----------------|:------:|
| FR-008-01 | 복수 파일 일괄 검증 (최대 100개) | POST /api/v1/validations/batch (4.2), Phase 3 Step 16 | Match |
| FR-008-02 | 합본 분리 | MergedFileSplitter, 관련 필드 (IS_MERGED, MERGED_CNT) | Match |
| FR-008-03 | 결과 대시보드 | GET /api/v1/admin/dashboard (4.3), Phase 3 Step 19 | Match |
| FR-008-04 | 일괄처리 세부 | Phase 3 Step 16 | Match |

**FR-008 Match: 4/4 = 100%**

### 3.9 FR Summary

| FR ID | Plan Sub-items | Design Covered | Match Rate |
|-------|:--------------:|:--------------:|:----------:|
| FR-001 | 10 | 10 | 100% |
| FR-002 | 33 | 33 | 100% |
| FR-003 | 5 modules | 5 modules | 100% |
| FR-004 | 7 | 7 | 100% |
| FR-005 | 8 | 8 | 100% |
| FR-006 | 9 | 9 | 100% |
| FR-007 | 5 | 5 | 100% |
| FR-008 | 4 | 4 | 100% |
| **Total** | **81** | **81** | **100%** |

---

## 4. User Stories Mapping

### 4.1 User Story-to-Design Traceability

| US ID | User | Story Summary | Phase | Design Mapping | Status |
|-------|------|--------------|:-----:|----------------|:------:|
| US-TAX-001 | 세무사 | 신고파일 즉시 검증, 가산세 위험 사전 차단 | 1 | POST /api/v1/validations (4.2), ValidationService (5.1), Pipeline L1~L5 | Match |
| US-TAX-002 | 세무사 | 오류 원인(법적근거) + 수정 가이드 확인 | 1 | GET details API (4.2) legalBasis/penaltyRef/fixGuide 필드, MST_VALIDATION_RULE | Match |
| US-TAX-003 | 세무사 | 검증 결과 PDF/Excel 출력 | 2 | GET report API format=PDF/EXCEL (4.2), ReportExportService (5.1), Phase 2 Step 14 | Match |
| US-TAX-004 | 세무사 | 파일 뷰어로 원시 데이터 확인 | 3 | FileViewerService (domain-model.md 6.4), Phase 3 Step 15 | Match |
| US-BK-001 | 기장대리 | 대량 파일 일괄 검증 (100개) | 3 | POST /api/v1/validations/batch (4.2), Phase 3 Step 16 | Match |
| US-BK-002 | 기장대리 | 합본 파일 거래처별 분리 검증 | 2 | MergedFileSplitter (5.1), IS_MERGED/MERGED_SEQ 필드, Phase 2 Step 14 | Match |
| US-BK-003 | 기장대리 | 오류 거래처 필터링 | 3 | GET details API severity/recordType 필터 (4.2), Phase 3 | Match |
| US-DEV-001 | 개발사 | RESTful API로 파일 검증 연동 | 2 | 전체 API Spec (Section 4), Swagger OpenAPI (SwaggerConfig) | Match |
| US-DEV-002 | 개발사 | 세목별 검증 규칙/오류코드 조회 | 2 | GET /api/v1/admin/rules (4.3), GET /api/v1/admin/codes (4.3) | Match |
| US-ADM-001 | 관리자 | 검증 규칙 CRUD | 1 | CRUD /api/v1/admin/rules (4.3), RuleManagementService (5.1) | Match |
| US-ADM-002 | 관리자 | 세율표/공제한도 관리 | 1 | CRUD tax-rates/deduction-limits (4.3), TaxRateRepository, DeductionLimitRepository | Match |
| US-ADM-003 | 관리자 | 레코드 레이아웃 버전 관리 | 1 | CRUD layouts (4.3), APPLY_START_DT/APPLY_END_DT 기반 버전 관리 | Match |
| US-ADM-004 | 관리자 | 검증 처리 현황 모니터링 | 3 | GET /api/v1/admin/dashboard (4.3), DashboardController, Phase 3 Step 19 | Match |

**User Story Match: 13/13 = 100%**

---

## 5. MoSCoW Priority Verification

### 5.1 Must Items (Phase 1 MVP)

| ID | Functionality | Design Status | Evidence |
|----|--------------|:-------------:|----------|
| FR-001-01~03, 06~10 | 파서 엔진 핵심 | Designed | Section 5.1 domain/parser/, Step 3 |
| FR-002-01~22 | L1~L3 검증 | Designed | Section 5.1 domain/validator/ level1~3, Step 4 |
| FR-003-VAT | 부가가치세 모듈 | Designed | Section 5.1 taxmodule/vat/ 4 validators, Step 5 |
| FR-004-01~06 | 규칙/세율표/레이아웃 외부화 | Designed | 9 MST Entity (3.1.2), 5 Admin API (4.3), Step 2 |
| FR-005-01~03,05 | 기본 리포팅 | Designed | RES 3 Entity (3.1.3~4), ReportService (5.1), Step 6 |
| FR-006-01~07,09 | 공통 유틸리티 | Designed | domain/service/ 5 services, common/util/ 3 classes, Step 1.4~1.5, 7 |

**Must Match: 6/6 categories = 100%**

### 5.2 Should Items (Phase 2)

| ID | Functionality | Design Status | Evidence |
|----|--------------|:-------------:|----------|
| FR-002-23~33 | L4~L5 검증 | Designed | Section 5.1 level4~5, Phase 2 Step 8 |
| FR-003-CIT/INC/WHT/CGT | 4개 세목 모듈 | Designed | Section 5.1 taxmodule/cit/inc/wht, Steps 9~12 |
| FR-001-04~05 | 대용량/다중 파일 | Designed | StreamFileReader (5.1), Phase 2 Step 13 |
| FR-005-06~07 | 거래처분리/PDF/Excel | Designed | ReportExportService (5.1), Phase 2 Step 14 |
| FR-004-07 | 규칙 이력 관리 | Designed | HIS_RULE_CHANGE Entity (3.1.4), RuleChangeRepository (5.1) |

**Should Match: 5/5 categories = 100%**

### 5.3 Could Items (Phase 3)

| ID | Functionality | Design Status | Evidence |
|----|--------------|:-------------:|----------|
| FR-007 | 파일 뷰어 | Designed | FileViewerService, Phase 3 Step 15 |
| FR-008 | 일괄처리 | Designed | batch API (4.2), Phase 3 Step 16 |
| FR-005-04 | 법적근거 연결 + 가산세 | Designed | legalBasis/penaltyRef 필드, Phase 3 Step 17 |
| 크로스 세목 검증 | 부가세-법인세 교차 | Designed | Phase 3 Step 18 |

**Could Match: 4/4 categories = 100%**

---

## 6. Non-Functional Requirements (NFR) Gap Analysis

### 6.1 Performance

| Plan NFR | Criteria | Design Coverage | Status |
|----------|----------|-----------------|:------:|
| 10MB 이하 단일 파일 3초 이내 | JMeter 부하 테스트 | Design Goals 1.1 (3초), Test Plan 8.1 JMeter | Match |
| 100MB 이상 스트림 30초 이내 | 실환경 테스트 | Design Goals 1.1 (30초), StreamFileReader (5.1), 11.4 Stream 기반 | Match |
| 100개 파일 일괄 5분 이내 | 통합 테스트 | Design Goals 1.1 (5분), batch API (4.2) | Match |

**Performance NFR Match: 3/3 = 100%**

### 6.2 Security

| Plan NFR | Criteria | Design Coverage | Status | GAP ID |
|----------|----------|-----------------|:------:|:------:|
| 주민번호 뒤 6자리 마스킹 | 코드 리뷰 + 테스트 | MaskingService (5.1), Security 7.4, Test 8.2 마스킹 확인 | Match | - |
| 검증 완료 후 원본 파일 즉시 삭제/암호화 | 보안 점검 | Security 7.4 FILE_RETENTION_SECONDS, FileStorageService (5.1) | Match | - |
| JWT 기반 인증, 신청인 이외 결과 접근 불가 | Spring Security 설정 | Auth API (4.4), Security 7.1~7.3, @PreAuthorize + ValidationAccessChecker | Match | - |
| 로그에 개인정보 출력 금지 | 코드 리뷰 | Security 7.4 로그 마스킹, Test 8.2 로그 개인정보 확인 | Match | - |
| SQL Injection 방지 (Prepared Statement) | 보안 점검 | Security 7.4 JPA Parameterized + MyBatis #{} | Match | - |

**Security NFR Match: 5/5 = 100%**

### 6.3 Extensibility

| Plan NFR | Criteria | Design Coverage | Status |
|----------|----------|-----------------|:------:|
| 플러그인 구조 신규 세목 추가 | 아키텍처 리뷰 | Strategy 패턴 (TaxFileParser, TaxValidationModule), Design Principles 1.2 Open/Closed | Match |
| DB 기반 규칙 추가 (코드 변경 없음) | 규칙 추가 테스트 | MST_VALIDATION_RULE DB 외부화, Admin Rules API (4.3) | Match |
| 레이아웃 메타데이터 변경만으로 포맷 대응 | 데이터 변경 테스트 | MST_RECORD_LAYOUT/FIELD_LAYOUT, Admin Layouts API (4.3) | Match |

**Extensibility NFR Match: 3/3 = 100%**

### 6.4 Operations

| Plan NFR | Criteria | Design Coverage | Status | GAP ID |
|----------|----------|-----------------|:------:|:------:|
| 감사 로그 5년 보관 (HIS_VALIDATION_LOG) | 보관 정책 검증 | HIS_VALIDATION_LOG Entity (3.1.4), schema.md 보관 정책 "5년" 명시 | Match | - |
| 검증 건수/오류율/처리시간 통계 대시보드 | 모니터링 검증 | GET /api/v1/admin/dashboard (4.3), DashboardMapper MyBatis (5.1), Phase 3 Step 19 | Match | - |
| 로그 보관 자동화 메커니즘 | 파티셔닝/아카이빙 | schema.md Index Strategy 6.2: "월단위 파티셔닝 검토" 언급 | Partial | GAP-NFR-001 |

**Operations NFR Match: 2.5/3**

---

## 7. Architecture Decision Records (ADR) Gap Analysis

### 7.1 ADR Traceability

| Plan ADR (Section 9.2) | Plan Selection | Design Coverage | Status |
|------------------------|---------------|-----------------|:------:|
| Backend Framework | Spring Boot 3.x | Design 전반 Spring Boot 3.x 기반, Security 6.x, Jakarta EE | Match |
| 데이터 접근 | JPA + MyBatis 혼합 | JPA Repository 16개 (5.1), MyBatis Mapper 2개 (DashboardMapper, ValidationSearchMapper) | Match |
| DB | MySQL 8.x / PostgreSQL 15.x | Data Layer (2.1) MySQL 8.x / PostgreSQL 명시 | Match |
| 검증 규칙 엔진 | DB 외부화 + Strategy | MST_VALIDATION_RULE + Strategy 패턴 (TaxValidationModule) + Redis 캐시 | Match |
| 인증 | JWT (Access + Refresh) | Auth API (4.4), Security 7.1 JWT 1h/7d | Match |
| 캐싱 | Redis | RedisCacheConfig (5.1), Data Layer Redis (2.1) | Match |
| 테스트 | JUnit 5 + Mockito + TestContainers | Test Plan (8.1), Test Cases (8.2) | Match |
| 빌드 | Gradle (Kotlin DSL) | Implementation Guide Step 1.1 (11.1) | Match |

**ADR Match: 8/8 = 100%**

### 7.2 Architecture Level

| Plan | Design | Status |
|------|--------|:------:|
| Enterprise Level (Section 9.1) | Enterprise 4-Layer (Section 9.1) | Match |

### 7.3 Clean Architecture Layer

| Plan (Section 9.3) | Design (Section 9) | Status |
|--------------------|---------------------|:------:|
| presentation/ | presentation/ (api/, admin/, dto/) | Match |
| application/ (validation/, reporting/, management/) | application/ (validation/, parser/, pipeline/, reporting/, management/, auth/) | Match |
| domain/ (parser/, validator/, taxmodule/, model/) | domain/ (parser/, validator/, taxmodule/, model/, service/) | Match |
| infrastructure/ (persistence/, fileio/, cache/) | infrastructure/ (persistence/, fileio/, cache/, security/) | Match |
| common/ (util/, exception/, config/) | common/ (util/, exception/, config/) | Match |

**Clean Architecture Match: 5/5 = 100%**

---

## 8. Technology Stack & Dependency Gap Analysis

### 8.1 Tech Stack Traceability

| Plan (Section 9.2 + glossary) | Design | Status | GAP ID |
|------------------------------|--------|:------:|:------:|
| Java 17+ | Design 전반 Java 17 기반 | Match | - |
| Spring Boot 3.x | Design 전반 + Step 1.1 | Match | - |
| Spring Security 6.x | Security 7.1, SecurityConfig | Match | - |
| JPA (Hibernate) | 16 JPA Entity/Repository | Match | - |
| MyBatis 3.x | DashboardMapper, ValidationSearchMapper | Match | - |
| JUnit 5 + Mockito | Test Plan 8.1 | Match | - |
| TestContainers | Test Plan 8.1 통합 테스트 | Match | - |
| Redis | RedisCacheConfig, 환경변수 | Match | - |
| MySQL 8.x / PostgreSQL 15.x | Data Layer 2.1 | Match | - |
| Gradle Kotlin DSL | Step 1.1 | Match | - |
| HikariCP | (glossary에 정의) | 미명시 | GAP-TECH-001 |
| SLF4J + Logback | (glossary에 정의) | 미명시 | GAP-TECH-002 |
| QueryDSL | (glossary에 정의) | 미명시 | GAP-TECH-003 |
| ICU4J / juniversalchardet | (Design 11.4에 언급) | 언급됨 | - |
| Docker | (glossary에 정의) | 미명시 | GAP-TECH-004 |
| MapStruct | (Design 9.3에 권장) | 권장 언급 | - |

### 8.2 External Library Coverage

| Glossary Library | Design Reference | Status |
|------------------|-----------------|:------:|
| HikariCP | 명시적 설정 미정의 (Spring Boot 기본 내장) | Implicit |
| SLF4J + Logback | 명시적 설정 미정의 (Spring Boot 기본 내장) | Implicit |
| QueryDSL | Design에 미언급 | Not covered |
| Docker | Design에 미언급 | Not covered |

---

## 9. Test Strategy Gap Analysis

### 9.1 Test Type Coverage

| Plan Test Type (Section 7.3) | Design Test Type (Section 8) | Status |
|-----------------------------|------------------------------|:------:|
| 단위 테스트 (JUnit 5 + Mockito) | 단위 테스트 (8.1) 90%+ / 80%+ 커버리지 | Match (Design 커버리지 더 상세) |
| 통합 테스트 (@SpringBootTest + TestContainers) | 통합 테스트 (8.1) 주요 시나리오 100% | Match |
| 성능 테스트 (JMeter) | 성능 테스트 (8.1) 3초/30초/5분 | Match |
| 보안 테스트 (OWASP ZAP, 코드 리뷰) | 보안 테스트 (8.1) 취약점 0건 + 보안 TC 11건 (8.2) | Match (Design 더 상세) |
| 검증 정확성 (실제 파일 샘플) | 검증 정확성 (8.1) 홈택스 일치 | Match |

### 9.2 Test Case Detail

| Plan Criterion | Design Test Case | Status |
|---------------|------------------|:------:|
| 커버리지 80% 이상 | Design: 유틸리티 90%+, 검증규칙 80%+ | Match (초과 달성) |
| 개인정보 마스킹 100% | Test 8.2: 마스킹 확인 TC | Match |
| 오류코드 분류 정확도 | Test 8.2: 세목별 검증 TC 전체 | Match |
| Phase별 검증 규칙 100% | Test 8.2: VAT/CIT/INC/WHT 개별 TC | Match |

**Test Strategy Match: 5/5 types = 100%**

---

## 10. Legal Basis / Penalty Reference Gap Analysis

### 10.1 Legal Foundation Coverage

| Plan Legal Basis (Section 1.3) | Design Coverage | Status |
|-------------------------------|-----------------|:------:|
| 국세기본법 제5조의2 (전자신고 근거) | Design 전반 시스템 목적 반영 | Match |
| 국세기본법 시행령 제3조의2 (절차/요건) | L1~L2 검증으로 파일 규격 준수 검증 | Match |
| 부가가치세법 제48/49조 | VatValidationModule 검증 대상 | Match |
| 소득세법 제70조 | IncValidationModule 검증 대상 | Match |
| 법인세법 제60조 | CitValidationModule 검증 대상 | Match |
| 국세청 고시 (파일포맷, 오류코드) | MST_RECORD_LAYOUT/FIELD_LAYOUT 메타데이터 외부화 | Match |

### 10.2 Penalty Tax Coverage

| Plan Penalty (Section 1.3) | Design Coverage | Status |
|---------------------------|-----------------|:------:|
| 무신고가산세 20%/40% | ValidationRule.penaltyRef, ErrorCode.PENALTY_INFO, API penaltyRef | Match |
| 과소신고가산세 10%/40% | 동일 | Match |
| 납부지연가산세 (일수 x 0.022%) | 동일 | Match |
| 세금계산서 미발급/부실기재 2%/1% | 동일 | Match |
| 전자세금계산서 미전송 0.5% | 동일 | Match |
| 합계표 미제출 0.5% | 동일 | Match |
| 성실신고확인서 미제출 5% | 동일 | Match |
| 수정신고 가산세 감면 (90%~10%) | penaltyRef 필드에 기재 가능 | Match |

### 10.3 Tax Agent Responsibility

| Plan Section | Design Coverage | Status | GAP ID |
|-------------|-----------------|:------:|:------:|
| 민사책임 (민법 제681조) | 설계 문서에 명시적 반영 없음 (리포트 데이터로 간접 지원) | Indirect | GAP-LEGAL-001 |
| 행정책임 (세무사법 제17조) | 동일 | Indirect | - |
| 형사책임 (조세범처벌법 제3조) | 동일 | Indirect | - |

**Legal/Penalty Match: 15/16 (세무대리인 책임은 간접 지원)**

---

## 11. Environment Variable Gap Analysis

### 11.1 Variable Coverage

| Plan Variable (Section 10.3) | Design Variable (Section 10.4) | Status |
|------------------------------|-------------------------------|:------:|
| DB_URL | DB_URL (default: jdbc:mysql://localhost:3306/fev) | Match |
| DB_USERNAME | DB_USERNAME (default: fev_user) | Match |
| DB_PASSWORD | DB_PASSWORD (필수) | Match |
| JWT_SECRET | JWT_SECRET (256bit+, 필수) | Match |
| JWT_ACCESS_EXPIRATION | JWT_ACCESS_EXPIRATION (3600000) | Match |
| JWT_REFRESH_EXPIRATION | JWT_REFRESH_EXPIRATION (604800000) | Match |
| FILE_UPLOAD_PATH | FILE_UPLOAD_PATH (/tmp/fev/upload) | Match |
| FILE_RETENTION_SECONDS | FILE_RETENTION_SECONDS (3600) | Match |
| FILE_MAX_SIZE | FILE_MAX_SIZE (200MB) | Match |
| REDIS_HOST | REDIS_HOST (localhost) | Match |
| REDIS_PORT | REDIS_PORT (6379) | Match |
| - | SPRING_PROFILES_ACTIVE (local) | Design에만 존재 |
| - | encryption.aes.key (Section 7.5) | Design에만 존재 |

**Environment Variable Match: 11/11 Plan variables = 100% (Design has 2 additional)**

---

## 12. Data Model Gap Analysis (Plan schema.md vs Design Section 3)

### 12.1 Entity Coverage

| Schema Table (16) | Design JPA Entity | Status |
|--------------------|-------------------|:------:|
| MST_TAX_TYPE | TaxType (3.1.2) | Match |
| MST_FORM_CODE | FormCode (3.1.2) | Match |
| MST_RECORD_LAYOUT | RecordLayout (3.1.2) | Match |
| MST_FIELD_LAYOUT | FieldLayout (3.1.2) | Match |
| MST_VALIDATION_RULE | ValidationRule (3.1.2) | Match |
| MST_TAX_RATE | TaxRate (3.1.2) | Match |
| MST_DEDUCTION_LIMIT | DeductionLimit (3.1.2) | Match |
| MST_CODE | Code (3.1.2) | Match |
| MST_ERROR_CODE | ErrorCode (3.1.2) | Match |
| REQ_VALIDATION | ValidationRequest (3.1.3) | Match |
| REQ_VALIDATION_FILE | ValidationFile (3.1.3) | Match |
| RES_VALIDATION_RESULT | ValidationResult (3.1.3) | Match |
| RES_VALIDATION_DETAIL | ValidationDetail (3.1.3) | Match |
| RES_VALIDATION_REPORT | ValidationReport (3.1.4) | Match |
| HIS_VALIDATION_LOG | ValidationLog (3.1.4) | Match |
| HIS_RULE_CHANGE | RuleChange (3.1.4) | Match |

**Entity Match: 16/16 = 100%**

### 12.2 Field-Level Sync (Spot Check)

| Table | Schema Columns | Design Entity Fields | Status |
|-------|:--------------:|:-------------------:|:------:|
| MST_TAX_TYPE | 7 + audit | 7 + audit (BaseEntity) | Match |
| MST_VALIDATION_RULE | 13 + audit | 13 + audit (incl. penaltyRef) | Match |
| MST_DEDUCTION_LIMIT | 9 + audit | 9 + audit (incl. legalBasis) | Match |
| REQ_VALIDATION_FILE | 13 + audit | 13 + audit | Match |
| RES_VALIDATION_RESULT | 17 + audit | 17 + audit | Match |
| RES_VALIDATION_DETAIL | 14 + reg_dt/reg_id | 14 + reg_dt/reg_id | Match |

### 12.3 Relationship Coverage

| Schema Relationship | Design Entity Relationship (3.2) | Status |
|--------------------|---------------------------------|:------:|
| TAX_TYPE 1:N FORM_CODE | MST_TAX_TYPE 1:N MST_FORM_CODE | Match |
| TAX_TYPE 1:N RECORD_LAYOUT | MST_TAX_TYPE 1:N MST_RECORD_LAYOUT | Match |
| RECORD_LAYOUT 1:N FIELD_LAYOUT | MST_RECORD_LAYOUT 1:N MST_FIELD_LAYOUT | Match |
| VALIDATION_RULE 1:N ERROR_CODE | MST_VALIDATION_RULE 1:N MST_ERROR_CODE | Match |
| VALIDATION 1:N VALIDATION_FILE | REQ_VALIDATION 1:N REQ_VALIDATION_FILE | Match |
| VALIDATION_FILE 1:N VALIDATION_RESULT | REQ_VALIDATION_FILE 1:N RES_VALIDATION_RESULT | Match |
| VALIDATION_RESULT 1:N VALIDATION_DETAIL | RES_VALIDATION_RESULT 1:N RES_VALIDATION_DETAIL | Match |
| VALIDATION_RESULT 1:N VALIDATION_REPORT | RES_VALIDATION_RESULT 1:N RES_VALIDATION_REPORT | Match |

**Relationship Match: 8/8 = 100%**

### 12.4 Index Coverage

| Schema Index | Design Entity / Schema | Status |
|-------------|----------------------|:------:|
| IDX_LAYOUT_TAX_FORM | schema.md DDL 정의, Design RecordLayout 사용 | Match |
| IDX_FIELD_LAYOUT | schema.md DDL 정의, Design @OrderBy 사용 | Match |
| IDX_RULE_TAX_LEVEL | schema.md DDL 정의 | Match |
| UDX_RATE (Unique) | schema.md DDL 정의 | Match |
| UDX_DEDUCTION (Unique) | schema.md DDL 정의 | Match |
| UDX_REQUEST_NO (Unique) | schema.md DDL 정의, Design unique=true | Match |
| IDX_VALID_USER | schema.md DDL 정의 | Match |

**Index Match: 7/7 = 100%**

---

## 13. API Endpoint Gap Analysis (Plan Section 9.4 vs Design Section 4)

### 13.1 Endpoint Coverage

| Plan Endpoint (Section 9.4) | Design Endpoint (Section 4) | Status | GAP ID |
|----------------------------|-----------------------------|:------:|:------:|
| POST /api/v1/validations | POST /api/v1/validations (4.2) | Match | - |
| GET /api/v1/validations/{id} | GET /api/v1/validations/{validationId} (4.2) | Match | - |
| GET /api/v1/validations/{id}/details | GET /api/v1/validations/{validationId}/results/{resultId}/details (4.2) | Changed | GAP-API-001 |
| GET /api/v1/validations/{id}/report | GET /api/v1/validations/{validationId}/results/{resultId}/report (4.2) | Changed | GAP-API-002 |
| POST /api/v1/validations/batch | POST /api/v1/validations/batch (4.2) | Match | - |
| GET/POST/PUT /api/v1/admin/rules | CRUD /api/v1/admin/rules (4.3) + DELETE | Match (Design adds DELETE) | - |
| GET/POST/PUT /api/v1/admin/tax-rates | CRUD /api/v1/admin/tax-rates (4.3) + DELETE | Match (Design adds DELETE) | - |
| GET/POST/PUT /api/v1/admin/layouts | CRUD /api/v1/admin/layouts (4.3) | Match | - |
| GET/POST/PUT /api/v1/admin/codes | CRUD /api/v1/admin/codes (4.3) | Match | - |
| GET /api/v1/admin/dashboard | GET /api/v1/admin/dashboard (4.3) | Match | - |
| POST /api/v1/auth/login | POST /api/v1/auth/login (4.4) | Match | - |
| POST /api/v1/auth/refresh | POST /api/v1/auth/refresh (4.4) | Match | - |
| - | POST /api/v1/auth/logout (4.4) | Design only | - |
| - | CRUD /api/v1/admin/deduction-limits (4.3) | Design only (v1.1.0 추가) | - |

**API Endpoint Match: 12/12 Plan endpoints covered (2 endpoints URL 구조 변경 -- 합본 파일 지원을 위한 합리적 변경)**

---

## 14. Convention Compliance (Plan Section 10 vs Design Section 10)

### 14.1 Naming Convention

| Plan Convention | Design Convention | Status |
|----------------|-------------------|:------:|
| MST_/REQ_/RES_/HIS_ 접두어 | Design 10.1 DB 테이블 UPPER_SNAKE_CASE + 접두어 | Match |
| {세목}-{레벨}{번호} 오류코드 | Design 10.1 오류코드 `{세목}-{레벨}{번호}` | Match |
| /api/v1/{resource} URL 패턴 | Design 10.1 API URL kebab-case, 복수형 명사 | Match |
| 환경변수 UPPER_SNAKE_CASE | Design 10.1 환경변수 UPPER_SNAKE_CASE | Match |
| 클래스 PascalCase | Design 10.1 PascalCase | Match |
| 메서드 camelCase | Design 10.1 camelCase | Match |
| 상수 UPPER_SNAKE_CASE | Design 10.1 UPPER_SNAKE_CASE | Match |

**Convention Match: 7/7 = 100%**

### 14.2 Additional Conventions Defined in Design

| Design Convention | Plan Status | Notes |
|-------------------|-------------|-------|
| 패키지명 lowercase | Plan 미정의 | Design에서 보완 |
| 인터페이스 접미사 없음 | Plan 미정의 | Design에서 보완 |
| 구현 클래스 Impl 접미사 | Plan 미정의 | Design에서 보완 |
| DB 컬럼 감사필드 lower_snake_case | Plan에 reg_dt/chg_dt 정의 | 일치 |
| Git Commit Convention | Plan 미정의 | Design 10.3에서 정의 |
| 코딩 규칙 8가지 | Plan 코딩 표준 참조 | Design 10.2에서 구체화 |

---

## 15. Identified Gaps

### 15.1 Gap Summary

| GAP ID | Category | Severity | Description |
|--------|----------|:--------:|-------------|
| GAP-API-001 | API | Low | Plan details URL과 Design details URL 구조 차이 |
| GAP-API-002 | API | Low | Plan report URL과 Design report URL 구조 차이 |
| GAP-NFR-001 | NFR/Operations | Low | 로그 보관 자동화 메커니즘(파티셔닝) 구체적 설계 미상세 |
| GAP-TECH-001 | Technology | Low | HikariCP 명시적 설정 미정의 (Spring Boot 기본 내장) |
| GAP-TECH-002 | Technology | Low | SLF4J/Logback 명시적 설정 미정의 (Spring Boot 기본 내장) |
| GAP-TECH-003 | Technology | Low | QueryDSL 사용 여부 미언급 |
| GAP-TECH-004 | Technology | Low | Docker 배포 관련 설계 미정의 |
| GAP-LEGAL-001 | Legal | Low | 세무대리인 책임 관련 리포트 지원 기능 명시적 미정의 (간접 지원) |

### 15.2 Gap Details

#### GAP-API-001 (Low): Plan vs Design API URL Structure Difference - Details

- **Plan**: `GET /api/v1/validations/{id}/details`
- **Design**: `GET /api/v1/validations/{validationId}/results/{resultId}/details`
- **Impact**: Low -- Design의 변경이 합본 파일 분리 후 개별 결과(resultId)를 지원하기 위한 합리적 개선임
- **Recommendation**: Plan 문서의 API 패턴을 Design에 맞추어 업데이트

#### GAP-API-002 (Low): Plan vs Design API URL Structure Difference - Report

- **Plan**: `GET /api/v1/validations/{id}/report`
- **Design**: `GET /api/v1/validations/{validationId}/results/{resultId}/report`
- **Impact**: Low -- GAP-API-001과 동일 사유
- **Recommendation**: Plan 문서의 API 패턴을 Design에 맞추어 업데이트

#### GAP-NFR-001 (Low): Log Retention Automation

- **Plan**: "감사 로그 5년 보관 (HIS_VALIDATION_LOG)"
- **Design**: schema.md에 "월단위 파티셔닝 검토" 언급은 있으나, 구체적 파티셔닝 DDL이나 아카이빙 정책 설계 없음
- **Impact**: Low -- 운영 단계에서 정의 가능
- **Recommendation**: Implementation Guide에 로그 테이블 파티셔닝 및 아카이빙 방안 Step 추가

#### GAP-TECH-001~002 (Low): Spring Boot Built-in Library Configuration

- **Plan/Glossary**: HikariCP, SLF4J/Logback 명시적 기재
- **Design**: Spring Boot 3.x 기본 내장으로 별도 설정 없이 사용 가능하므로 명시적 설정 미정의
- **Impact**: Low -- Spring Boot auto-configuration으로 기본 동작
- **Recommendation**: Implementation Guide Step 1에 application.yml 예시로 HikariCP 풀 사이즈, Logback 로그 레벨 설정 추가

#### GAP-TECH-003 (Low): QueryDSL Usage

- **Plan/Glossary**: "QueryDSL - 타입세이프 동적 쿼리 빌더" 기재
- **Design**: MyBatis 동적 쿼리 (DashboardMapper, ValidationSearchMapper)로 대체 설계
- **Impact**: Low -- 기능적으로 MyBatis로 충분히 대체 가능
- **Recommendation**: Plan glossary에 "QueryDSL 대신 MyBatis 동적 쿼리 사용" 결정사항 반영, 또는 향후 필요 시 추가

#### GAP-TECH-004 (Low): Docker Deployment Design

- **Plan/Glossary**: "Docker - 컨테이너 플랫폼, 배포" 기재
- **Design**: Docker 관련 Dockerfile, docker-compose 설계 미정의
- **Impact**: Low -- 배포 단계에서 정의 가능 (Phase 9 CI/CD 범위)
- **Recommendation**: Phase 9 배포 설계 시 Dockerfile 및 docker-compose.yml 정의

#### GAP-LEGAL-001 (Low): Tax Agent Responsibility Report Support

- **Plan**: 세무대리인 책임 (민사/행정/형사) 상세 기재
- **Design**: 검증 리포트(법적근거, 가산세 참조)로 간접 지원하나, "세무대리인 주의의무 관점" 전용 리포트 포맷 미정의
- **Impact**: Low -- 기존 리포트로 충분한 정보 제공, 전용 포맷은 부가 기능
- **Recommendation**: Phase 3 리포팅 고도화 시 "세무대리인 주의의무 체크리스트" 리포트 형식 검토

---

## 16. Overall Scores

### 16.1 Category Scores

| Category | Items Checked | Items Matched | Score | Status |
|----------|:------------:|:-------------:|:-----:|:------:|
| FR Coverage (FR-001~008) | 81 | 81 | 100% | PASS |
| User Stories (US-TAX/BK/DEV/ADM) | 13 | 13 | 100% | PASS |
| MoSCoW Must Items | 6 categories | 6 | 100% | PASS |
| NFR Performance | 3 | 3 | 100% | PASS |
| NFR Security | 5 | 5 | 100% | PASS |
| NFR Extensibility | 3 | 3 | 100% | PASS |
| NFR Operations | 3 | 2.5 | 83% | PASS |
| ADR Decisions | 8 | 8 | 100% | PASS |
| Clean Architecture | 5 layers | 5 | 100% | PASS |
| Data Model (Entities) | 16 | 16 | 100% | PASS |
| Data Model (Relationships) | 8 | 8 | 100% | PASS |
| Data Model (Indexes) | 7 | 7 | 100% | PASS |
| API Endpoints | 12 | 12 | 100% | PASS |
| Technology Stack | 15 | 11 | 73% | PASS |
| Test Strategy | 5 types | 5 | 100% | PASS |
| Legal/Penalty Basis | 16 | 15 | 94% | PASS |
| Environment Variables | 11 | 11 | 100% | PASS |
| Conventions | 7 | 7 | 100% | PASS |

### 16.2 Weighted Overall Score

| Category | Weight | Raw Score | Weighted Score |
|----------|:------:|:---------:|:--------------:|
| FR Coverage | 25% | 100% | 25.0 |
| User Stories | 10% | 100% | 10.0 |
| MoSCoW Must | 10% | 100% | 10.0 |
| NFR (Performance+Security+Ext+Ops) | 15% | 96% | 14.4 |
| Architecture (ADR+Clean+Data Model) | 15% | 100% | 15.0 |
| API Endpoints | 10% | 100% | 10.0 |
| Technology Stack | 5% | 73% | 3.7 |
| Test Strategy | 5% | 100% | 5.0 |
| Legal/Convention/Env | 5% | 98% | 4.9 |
| **TOTAL** | **100%** | | **98.0%** |

### 16.3 Overall Result

```
+-------------------------------------------------------+
|  Plan-Design Gap Analysis Result (v1.1.0)              |
+-------------------------------------------------------+
|                                                        |
|  Overall Match Rate:  98.0%                            |
|  Target:              95.0%                            |
|  Result:              PASS (target exceeded)           |
|                                                        |
+-------------------------------------------------------+
|  Total Gaps:  8                                        |
|  - Critical:  0                                        |
|  - High:      0                                        |
|  - Medium:    0                                        |
|  - Low:       8                                        |
+-------------------------------------------------------+
|  v1.0.0 Gap Resolution:  5/5 Medium gaps resolved      |
|  v1.0.0 -> v1.1.0:  93% -> 98% (+5%)                  |
+-------------------------------------------------------+
```

---

## 17. Comparison with v1.0.0 Analysis

| Metric | v1.0.0 | v1.1.0 | Change |
|--------|:------:|:------:|:------:|
| Overall Match Rate | 93% | 98% | +5% |
| Total Gaps | 19 | 8 | -11 |
| Critical Gaps | 0 | 0 | 0 |
| High Gaps | 0 | 0 | 0 |
| Medium Gaps | 5 | 0 | -5 |
| Low Gaps | 14 | 8 | -6 |

**Key Improvements in v1.1.0**:
1. DeductionLimit JPA Entity 추가 (schema.md 완전 대응)
2. 공제한도 Admin CRUD API 신규 추가
3. Method-level @PreAuthorize + ValidationAccessChecker 보안 상세화
4. AES-256 암호화 설계 + 보안 테스트 케이스 6건 추가
5. penaltyRef 필드 추가 (가산세 참조 완전 지원)
6. FormCode, Code, ErrorCode, ValidationReport, ValidationLog, RuleChange JPA Entity 추가 (총 7개)
7. Entity-Schema 컬럼 16개 필드 동기화

---

## 18. Recommended Actions

### 18.1 Plan Document Updates (Low Priority)

| No. | Action | GAP ID | Document |
|-----|--------|--------|----------|
| 1 | API Endpoint URL을 Design 기준으로 업데이트 (details/report URL 구조) | GAP-API-001, GAP-API-002 | plan.md Section 9.4 |
| 2 | QueryDSL 대신 MyBatis 사용 결정 반영 | GAP-TECH-003 | plan.md glossary |
| 3 | Docker 배포는 Phase 9 범위임을 명기 | GAP-TECH-004 | plan.md Section 11 |

### 18.2 Design Document Enhancements (Low Priority)

| No. | Action | GAP ID | Document |
|-----|--------|--------|----------|
| 1 | HIS_VALIDATION_LOG 파티셔닝 DDL 또는 운영 가이드 추가 | GAP-NFR-001 | design.md Section 11 |
| 2 | application.yml 예시 추가 (HikariCP, Logback 설정 포함) | GAP-TECH-001, GAP-TECH-002 | design.md Section 10.4 |
| 3 | 세무대리인 주의의무 리포트 형식 (Phase 3 검토) 언급 추가 | GAP-LEGAL-001 | design.md Section 11.3 |

### 18.3 Synchronization Choice

모든 갭이 Low 심각도이므로 즉시 조치는 불필요합니다. 다음 중 하나를 선택할 수 있습니다:

1. **Plan 문서를 Design에 맞추어 업데이트** -- API URL 구조 변경 반영 (권장)
2. **Design 문서에 보완 항목 추가** -- 파티셔닝, 환경설정 예시
3. **현재 차이를 의도적 결정으로 기록** -- 모든 Low 갭을 문서화하여 유지
4. **Phase 3/Phase 9에서 자연스럽게 해소** -- Docker, 리포트 고도화 등

---

## 19. Next Steps

- [x] v1.0.0 Medium Gap 5건 해소 확인 완료
- [x] v1.1.0 Plan-Design 갭 분석 완료 (매칭률 98%)
- [ ] Plan 문서 API Section 업데이트 (GAP-API-001, GAP-API-002 반영)
- [ ] Phase 1 MVP 구현 시작 (Design Section 11.1 Step 1~7)
- [ ] 국세청 파일설명서(.doc) 텍스트 추출 및 레코드 레이아웃 JSON 변환

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2026-02-19 | Initial Plan-Design gap analysis (v1.1.0): 10 categories, 8 Low gaps, 98% overall match rate, v1.0.0 5 Medium gaps resolved | gap-detector agent |

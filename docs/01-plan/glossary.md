# TaxServiceENTEC-FEV Glossary (용어 사전)

> **Project**: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
> **Version**: 1.0.0
> **Date**: 2026-02-19
> **Purpose**: 프로젝트 전반에서 사용되는 비즈니스 용어, 글로벌 표준, 매핑 관계를 정의

---

## 1. Business Terms (비즈니스 용어)

### 1.1 시스템 핵심 용어

| 한글 용어 | 영문 | 약어 | 정의 | 코드 사용 |
|----------|------|------|------|----------|
| 전자신고 파일 오류검증 | File for Electronic filing Validation | FEV | 세무프로그램에서 생성된 전자신고 파일의 형식/내용 오류를 홈택스 제출 전 독립적으로 검증하는 시스템 | `FEV` |
| 전자신고 파일 | Electronic Filing File | - | 세무프로그램이 국세청 표준 포맷으로 생성한 고정길이 레코드 기반 텍스트 파일 | `FilingFile` |
| 레코드 레이아웃 | Record Layout | - | 국세청이 정한 파일 내 각 레코드의 필드별 위치/길이/타입 명세 | `RecordLayout` |
| 검증 파이프라인 | Validation Pipeline | - | L1(형식)→L2(구조)→L3(내용)→L4(논리)→L5(법규) 5단계 순차 검증 체계 | `ValidationPipeline` |
| 검증 규칙 | Validation Rule | - | 각 검증 단계에서 적용되는 개별 검증 조건/수식 | `ValidationRule` |
| 검증 결과 | Validation Result | - | 검증 파이프라인 수행 후 생성되는 오류 목록 및 요약 정보 | `ValidationResult` |
| 오류코드 | Error Code | - | 검증 결과 식별을 위한 코드 (예: VAT-C001) | `ErrorCode` |
| 심각도 | Severity | - | 오류의 중요도 분류 (Fatal/Error/Warning/Info) | `Severity` |
| 수정 가이드 | Fix Guide | - | 오류 수정을 위한 안내 문구 | `FixGuide` |
| 법적 근거 | Legal Basis | - | 검증 규칙의 세법 조항 참조 | `LegalBasis` |

### 1.2 파일 구조 용어

| 한글 용어 | 영문 | 정의 | 코드 사용 |
|----------|------|------|----------|
| A 레코드 | Header Record | 파일의 헤더 레코드 (1건, 납세자/신고 기본정보) | `RECORD_TYPE_A` |
| B~Y 레코드 | Body Records | 신고서 본문 및 부속명세서 데이터 레코드 | `RECORD_TYPE_B` ~ `RECORD_TYPE_Y` |
| Z 레코드 | Trailer Record | 파일의 트레일러 레코드 (1건, 건수/합계) | `RECORD_TYPE_Z` |
| 서식코드 | Form Code | 서식(부속명세서)을 식별하는 코드 (예: V101001) | `FormCode` |
| 합본 파일 | Merged File | 여러 거래처의 신고서를 하나의 파일로 합친 것 | `MergedFile` |
| 고정길이 레코드 | Fixed-Length Record | 위치(Position) 기반으로 필드를 배치하는 레코드 형식 | `FixedLengthRecord` |
| 필드 | Field | 레코드 내 개별 데이터 항목 (위치+길이+타입으로 정의) | `Field` |
| 패딩 | Padding | 필드값을 규정 길이에 맞추기 위한 채움 문자 (문자: 공백, 숫자: '0') | `Padding` |

### 1.3 세무 도메인 용어

| 한글 용어 | 영문 | 정의 | 코드 사용 |
|----------|------|------|----------|
| 세목 | Tax Type | 세금의 종류 (부가가치세, 법인세, 종합소득세, 원천세, 양도소득세) | `TaxType` |
| 귀속연도 | Tax Year / Fiscal Year | 과세 대상이 되는 회계연도 | `taxYear` |
| 귀속기간 | Tax Period | 세목별 신고 기간 구분 (예: 1기 예정/확정, 2기 예정/확정) | `taxPeriod` |
| 과세표준 | Tax Base | 세금을 산출하기 위한 기초 금액 | `taxBase` |
| 산출세액 | Calculated Tax | 과세표준에 세율을 적용하여 산출된 세액 | `calculatedTax` |
| 납부세액 | Tax Payable | 산출세액에서 공제/감면 후 실제 납부할 세액 | `taxPayable` |
| 가산세 | Penalty Tax / Surcharge | 법정 의무 불이행 시 부과되는 추가 세금 | `penaltyTax` |
| 체크디지트 | Check Digit | 사업자번호/주민번호 등의 유효성 검증용 마지막 자리 숫자 | `checkDigit` |
| 단수처리 | Rounding / Truncation | 과세표준은 1,000원 미만 절사, 세액은 원 미만 절사 | `truncation` |
| 세무조정 | Tax Adjustment | 회계상 당기순이익을 세법상 소득금액으로 전환하는 절차 | `taxAdjustment` |
| 이월결손금 | Carryforward Loss | 과거 발생한 세무상 결손금을 당기 소득에서 공제하는 금액 | `carryforwardLoss` |
| 최저한세 | Minimum Tax | 감면 적용 후에도 최소한 납부해야 하는 세액 기준 | `minimumTax` |
| 간이세액표 | Simplified Tax Table | 근로소득 원천징수 시 월급여/부양가족수별 세액 조회표 | `simplifiedTaxTable` |
| 공제/감면 | Deduction / Exemption | 세액에서 차감되는 항목 (법정 한도 존재) | `deduction` |
| 세율 | Tax Rate | 과세표준에 적용되는 세금 비율 (누진세율, 비례세율) | `taxRate` |

### 1.4 검증 레벨 용어

| 한글 용어 | 영문 | 레벨 | 정의 | 코드 사용 |
|----------|------|------|------|----------|
| 형식검증 | Format Validation | L1 | 파일 포맷, 필드 길이, 데이터 타입 등 물리적 구조 검증 | `LEVEL_1_FORMAT` |
| 구조검증 | Structural Validation | L2 | 레코드 존재/순서/건수/필수항목 등 파일 구조 검증 | `LEVEL_2_STRUCTURE` |
| 내용검증 | Content Validation | L3 | 체크디지트, 합계정합성, 세액 계산 정확성 검증 | `LEVEL_3_CONTENT` |
| 논리검증 | Logic/Cross-reference Validation | L4 | 서식 간 교차참조, 기간 정합성, 상호참조 일관성 검증 | `LEVEL_4_LOGIC` |
| 법규검증 | Tax Law Compliance Validation | L5 | 세율 구간, 공제한도, 비과세 요건 등 세법 적법성 검증 | `LEVEL_5_REGULATORY` |

### 1.5 사용자 유형

| 한글 용어 | 영문 | 정의 | 코드 사용 |
|----------|------|------|----------|
| 세무사 | Tax Accountant | 세무 신고를 대리하는 전문 자격사 | `TAX_ACCOUNTANT` |
| 기장대리 | Bookkeeping Agent | 다수 거래처의 기장 및 신고를 대행하는 사무원 | `BOOKKEEPING_AGENT` |
| 개발사 | Software Developer | 세무프로그램 개발업체 (파일 품질 검증 연동) | `DEVELOPER` |
| 관리자 | Administrator | 시스템 관리자 (규칙/세율표/레이아웃 관리) | `ADMIN` |

---

## 2. Global Standards (글로벌 표준)

### 2.1 기술 표준

| 용어 | 정의 | 참조 |
|------|------|------|
| EUC-KR | 한글 인코딩 표준 (2바이트/문자), 국세청 전자신고 파일 기본 인코딩 | KS X 1001 |
| UTF-8 | 가변길이 유니코드 인코딩 (1~4바이트), 일부 세목 전환 중 | RFC 3629 |
| CRLF / LF | 줄바꿈 문자 (Carriage Return + Line Feed), 레코드 구분자 | - |
| Fixed-Length Record | 구분자 없이 위치(Position)와 길이(Length)로 필드를 식별하는 파일 형식 | - |
| JSON | 레코드 레이아웃 메타데이터 저장 포맷 | RFC 8259 |
| REST API | Representational State Transfer, API 아키텍처 스타일 | Roy Fielding |
| JWT | JSON Web Token, 무상태 인증 토큰 (Access + Refresh) | RFC 7519 |
| OAuth 2.0 | 인증/인가 프로토콜 (향후 확장 시) | RFC 6749 |
| UUID | 범용 고유 식별자 | RFC 4122 |
| OpenAPI 3.x | API 문서화 규격 (Swagger) | OAS 3.0 |

### 2.2 프레임워크 / 라이브러리

| 용어 | 버전 | 정의 | 용도 |
|------|------|------|------|
| Java | 17+ | 프로그래밍 언어 (LTS) | Backend 전체 |
| Spring Boot | 3.x | Java 웹 프레임워크 | Application Framework |
| Spring Security | 6.x | 인증/인가 프레임워크 | JWT 인증, 권한 제어 |
| JPA (Hibernate) | - | ORM (Object-Relational Mapping) | 기본 CRUD |
| MyBatis | 3.x | SQL Mapper | 복잡 집계 쿼리 |
| QueryDSL | - | 타입세이프 동적 쿼리 빌더 | 동적 쿼리 |
| JUnit 5 | - | 단위 테스트 프레임워크 | 테스트 |
| Mockito | - | Mock 객체 프레임워크 | 단위 테스트 |
| TestContainers | - | 통합 테스트용 Docker 컨테이너 | DB 연동 테스트 |
| HikariCP | - | JDBC 커넥션 풀 | DB 커넥션 관리 |
| SLF4J + Logback | - | 로깅 프레임워크 | 로그 기록 |
| Redis | - | 인메모리 데이터 저장소 | 캐싱, 세션 |
| MySQL | 8.x | RDBMS | 데이터 저장 |
| PostgreSQL | 15.x | RDBMS (대안) | 데이터 저장 |
| Gradle | Kotlin DSL | 빌드 도구 | 의존성/빌드 관리 |
| Docker | - | 컨테이너 플랫폼 | 배포 |

---

## 3. Mapping Table (비즈니스 ↔ 글로벌 표준 매핑)

| 비즈니스 용어 (한글) | 코드 사용 (영문) | 글로벌 표준 매핑 | DB 테이블 |
|-------------------|---------------|---------------|----------|
| 세목 | TaxType | Tax Category | MST_TAX_TYPE |
| 서식코드 | FormCode | Form Identifier | MST_FORM_CODE |
| 레코드 레이아웃 | RecordLayout | Record Metadata | MST_RECORD_LAYOUT |
| 필드 레이아웃 | FieldLayout | Field Metadata | MST_FIELD_LAYOUT |
| 검증 규칙 | ValidationRule | Validation Rule | MST_VALIDATION_RULE |
| 세율표 | TaxRate | Tax Rate Schedule | MST_TAX_RATE |
| 공제한도 | DeductionLimit | Deduction Limit | MST_DEDUCTION_LIMIT |
| 공통코드 | Code | Common Code | MST_CODE |
| 오류코드 | ErrorCode | Error Code | MST_ERROR_CODE |
| 검증 요청 | ValidationRequest | Validation Request | REQ_VALIDATION |
| 검증 대상 파일 | ValidationFile | Validation File | REQ_VALIDATION_FILE |
| 검증 결과 | ValidationResult | Validation Result | RES_VALIDATION_RESULT |
| 검증 상세 | ValidationDetail | Validation Detail | RES_VALIDATION_DETAIL |
| 검증 리포트 | ValidationReport | Validation Report | RES_VALIDATION_REPORT |
| 검증 로그 | ValidationLog | Audit Log | HIS_VALIDATION_LOG |
| 규칙 변경 이력 | RuleChangeHistory | Change History | HIS_RULE_CHANGE |

---

## 4. Code Tables (코드표)

### 4.1 세목코드 (TAX_TYPE_CD)

| 코드 | 세목 (한글) | 영문 약어 | 파일 확장자 |
|------|-----------|---------|-----------|
| VAT | 부가가치세 | VAT (Value Added Tax) | .101, .102 |
| CIT | 법인세 | CIT (Corporate Income Tax) | .201 |
| INC | 종합소득세 | INC (Individual Income Tax) | .301 |
| WHT | 원천세 | WHT (Withholding Tax) | .401 |
| CGT | 양도소득세 | CGT (Capital Gains Tax) | .501 |
| GFT | 증여세 | GFT (Gift Tax) | .601 |
| INH | 상속세 | INH (Inheritance Tax) | .701 |

### 4.2 심각도 코드 (SEVERITY)

| 코드 | 의미 | 설명 | 검증 중단 |
|------|------|------|----------|
| FATAL | 치명적 | 파일 구조 자체가 잘못됨, 후속 검증 불가 | Yes |
| ERROR | 오류 | 세액/합계 불일치, 수정 필수 | No |
| WARNING | 경고 | 패딩 오류, 선택적 수정 권고 | No |
| INFO | 정보 | 참고 사항, 수정 불필요 | No |

### 4.3 검증 레벨 코드 (RULE_LEVEL)

| 레벨 | 코드 접미어 | 검증 유형 | 설명 |
|------|-----------|---------|------|
| 1 | F | 형식검증 | Format Validation |
| 2 | S | 구조검증 | Structural Validation |
| 3 | C | 내용검증 | Content Validation |
| 4 | L | 논리검증 | Logic Validation |
| 5 | R/T | 법규검증 | Regulatory/Tax Law Validation |

### 4.4 오류코드 체계 (ERROR_CODE)

```
{세목코드}-{레벨접미어}{3자리 번호}

예시:
  VAT-C001  → 부가가치세, 내용검증(L3), 규칙번호 001
  CIT-L002  → 법인세, 논리검증(L4), 규칙번호 002
  WHT-F003  → 원천세, 형식검증(L1), 규칙번호 003
```

| 접두어 | 세목 | 범위 |
|--------|------|------|
| VAT-F/S/C/L/R | 부가가치세 | 형식/구조/내용/논리/법규 |
| CIT-F/S/C/L/T | 법인세-1 | 형식/구조/내용/논리/법규 |
| CIT2-F/E/W | 법인세-2 | 형식/내용논리/경고 |
| INC-C/L/T | 종합소득세 | 내용/논리/법규 |
| WHT-F/S/C/L/W | 원천세 | 형식/구조/내용/논리/경고 |

### 4.5 신고구분 코드 (RPT_TYPE_CD)

| 코드 | 의미 |
|------|------|
| 01 | 정기신고 |
| 02 | 수정신고 |
| 03 | 기한후신고 |
| 04 | 경정청구 |

### 4.6 귀속기간 코드 - 부가가치세 (TAX_PERIOD_CD)

| 코드 | 의미 | 기간 |
|------|------|------|
| 01 | 1기 예정 | 1~3월 |
| 02 | 1기 확정 | 1~6월 |
| 03 | 2기 예정 | 7~9월 |
| 04 | 2기 확정 | 7~12월 |

### 4.7 필드 데이터 타입 코드 (DATA_TYPE)

| 코드 | 의미 | 정렬 | 패딩 |
|------|------|------|------|
| C | 문자 (Character) | 좌측정렬 | 우측 공백 (0x20) |
| N | 숫자 (Numeric) | 우측정렬 | 좌측 '0' (0x30) |
| CN | 혼합 (Character-Numeric) | 좌측정렬 | 우측 공백 |

### 4.8 필수유형 코드 (REQUIRED_TYPE)

| 코드 | 의미 | 설명 |
|------|------|------|
| M | Mandatory | 반드시 값이 존재해야 함 |
| C | Conditional | 특정 조건 시 필수 |
| O | Optional | 선택 입력 |

### 4.9 법인구분 코드 (CORP_TYPE_CD)

| 코드 | 구분 |
|------|------|
| 01 | 내국영리법인 |
| 02 | 내국비영리법인 |
| 03 | 외국영리법인 |
| 04 | 외국비영리법인 |
| 05 | 조합법인 |

### 4.10 대손사유 코드 (BAD_DEBT_CD)

| 코드 | 사유 |
|------|------|
| 01 | 부도발생 (6개월 경과) |
| 02 | 강제집행/경매 |
| 03 | 채무자 회생/파산 |
| 04 | 소멸시효 완성 |
| 05 | 회수불능 (법원 확인) |

### 4.11 DB 테이블 접두어 체계

| 접두어 | 영문 | 주제영역 | 설명 |
|--------|------|---------|------|
| MST_ | Master | 기준정보 | 시스템 전반 참조 기초 데이터 |
| REQ_ | Request | 요청/입력 | 사용자 행위로 발생하는 동적 데이터 |
| RES_ | Result | 점검/결과 | 프로세스 완료 후 생성 데이터 |
| HIS_ | History | 관리/로그 | 시스템 운영 및 이력 추적 데이터 |

---

## 5. Term Usage Rules (용어 사용 규칙)

1. **코드에서**: 영문 사용 (`TaxType`, `ValidationRule`, `FormCode`)
2. **UI/문서에서**: 한글 사용 (세목, 검증 규칙, 서식코드)
3. **API 응답에서**: camelCase 영문 (`taxType`, `validationRule`, `formCode`)
4. **DB 컬럼에서**: UPPER_SNAKE_CASE (`TAX_TYPE_CD`, `FORM_CD`, `RULE_ID`)
5. **오류코드에서**: `{세목약어}-{레벨접미어}{번호}` (예: `VAT-C001`)
6. **환경변수에서**: UPPER_SNAKE_CASE (`DB_URL`, `JWT_SECRET`, `FILE_UPLOAD_PATH`)

---

## 6. Check Digit Algorithms (체크디지트 알고리즘)

### 6.1 사업자등록번호 (BRN - Business Registration Number)

```
10자리: D1 D2 D3 D4 D5 D6 D7 D8 D9 D10
가중치:   1  3  7  1  3  7  1  3  5

검증:
  합 = Σ(Di × Wi) + (D9 × 5 / 10)의 정수부
  체크디지트 D10 = (10 - (합 mod 10)) mod 10
```

### 6.2 주민등록번호 (RRN - Resident Registration Number)

```
13자리: D1~D13
가중치: 2  3  4  5  6  7  8  9  2  3  4  5

검증:
  합 = Σ(Di × Wi) for i=1~12
  체크디지트 D13 = (11 - (합 mod 11)) mod 10
```

### 6.3 법인등록번호 (CRN - Corporate Registration Number)

```
13자리: 4(등기소코드) + 2(법인종류) + 6(일련번호) + 1(검증번호)
가중치: 1 2 1 2 1 2 1 2 1 2 1 2

검증:
  각 자리 × 가중치, 10 이상이면 십의 자리 + 일의 자리
  합 = Σ(결과값)
  체크디지트 = (10 - (합 mod 10)) mod 10
```

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2026-02-19 | Initial glossary: 비즈니스 용어 40+, 글로벌 표준 20+, 코드표 11종, 체크디지트 3종 | Claude |

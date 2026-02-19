# TaxServiceENTEC-FEV Data Schema

> **Project**: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
> **Version**: 1.0.0
> **Date**: 2026-02-19
> **Purpose**: 프로젝트 전체 데이터 스키마 정의 (16개 테이블, 4개 주제영역)

---

## 1. Schema Overview

```
┌─────────────────────────────────────────────────────────┐
│                  FEV Data Schema (16 Tables)             │
├──────────────┬──────────────┬──────────────┬────────────┤
│ MST_ (9)     │ REQ_ (2)     │ RES_ (3)     │ HIS_ (2)   │
│ 기준정보      │ 요청/입력     │ 점검/결과     │ 관리/로그   │
├──────────────┼──────────────┼──────────────┼────────────┤
│ TAX_TYPE     │ VALIDATION   │ VALIDATION_  │ VALIDATION_│
│ FORM_CODE    │ VALIDATION_  │   RESULT     │   LOG      │
│ RECORD_      │   FILE       │ VALIDATION_  │ RULE_      │
│   LAYOUT     │              │   DETAIL     │   CHANGE   │
│ FIELD_LAYOUT │              │ VALIDATION_  │            │
│ VALIDATION_  │              │   REPORT     │            │
│   RULE       │              │              │            │
│ TAX_RATE     │              │              │            │
│ DEDUCTION_   │              │              │            │
│   LIMIT      │              │              │            │
│ CODE         │              │              │            │
│ ERROR_CODE   │              │              │            │
└──────────────┴──────────────┴──────────────┴────────────┘
```

### 1.1 공통 감사 컬럼 (모든 테이블 필수)

```sql
reg_dt  DATETIME  NOT NULL  DEFAULT CURRENT_TIMESTAMP  COMMENT '등록일시',
reg_id  VARCHAR(50)  NOT NULL                          COMMENT '등록자',
chg_dt  DATETIME  ON UPDATE CURRENT_TIMESTAMP          COMMENT '수정일시',
chg_id  VARCHAR(50)                                    COMMENT '수정자'
```

### 1.2 공통 규칙

| 규칙 | 설명 |
|------|------|
| PK 전략 | `BIGINT AUTO_INCREMENT` (기본), 인조키 최소화 |
| 삭제 전략 | Soft Delete (`del_yn` 또는 `deleted_at`), 법적 파기 시 Hard Delete |
| 정규화 | 3차 정규화 기본, 통계/대시보드 시 반정규화 허용 |
| NOT NULL | 우선 적용, 기본값 설정 |
| 민감정보 | 암호화 저장, 별도 테이블 분리 |
| 코멘트 | 모든 테이블/컬럼에 비즈니스 의미 Comment 필수 |

---

## 2. Master Tables (기준정보 - MST_)

### 2.1 MST_TAX_TYPE (세목 마스터)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | PK | 세목코드 (VAT, CIT, INC, WHT, CGT, GFT, INH) |
| TAX_TYPE_NM | VARCHAR(50) | NOT NULL | | 세목명 (한글) |
| TAX_TYPE_NM_EN | VARCHAR(50) | NOT NULL | | 세목명 (영문) |
| FILE_EXT | VARCHAR(10) | NOT NULL | | 파일 확장자 (.101, .201 등) |
| ENCODING | VARCHAR(10) | NOT NULL | | 기본 인코딩 (EUC-KR / UTF-8) |
| SORT_ORDER | INT | NOT NULL | | 표시 순서 |
| USE_YN | CHAR(1) | NOT NULL | | 사용여부 (Y/N), Default 'Y' |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_TAX_TYPE (
    TAX_TYPE_CD     VARCHAR(10) PRIMARY KEY             COMMENT '세목코드',
    TAX_TYPE_NM     VARCHAR(50) NOT NULL                COMMENT '세목명(한글)',
    TAX_TYPE_NM_EN  VARCHAR(50) NOT NULL                COMMENT '세목명(영문)',
    FILE_EXT        VARCHAR(10) NOT NULL                COMMENT '파일확장자',
    ENCODING        VARCHAR(10) NOT NULL DEFAULT 'EUC-KR' COMMENT '기본인코딩',
    SORT_ORDER      INT NOT NULL DEFAULT 0              COMMENT '표시순서',
    USE_YN          CHAR(1) NOT NULL DEFAULT 'Y'        COMMENT '사용여부',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50)
) COMMENT '세목 마스터';
```

**초기 데이터**:

| TAX_TYPE_CD | TAX_TYPE_NM | FILE_EXT |
|-------------|-------------|----------|
| VAT | 부가가치세 | .101,.102 |
| CIT | 법인세 | .201 |
| INC | 종합소득세 | .301 |
| WHT | 원천세 | .401 |
| CGT | 양도소득세 | .501 |

### 2.2 MST_FORM_CODE (서식코드 마스터)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| FORM_CD | VARCHAR(10) | NOT NULL | PK | 서식코드 (V101001, CIT0101 등) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | FK | 세목코드 |
| FORM_NM | VARCHAR(200) | NOT NULL | | 서식명 |
| RECORD_TYPE | CHAR(1) | NOT NULL | | 레코드타입 (A~Z) |
| REQUIRED_YN | CHAR(1) | NOT NULL | | 필수여부 (Y/N/C-조건부) |
| REQUIRED_COND | VARCHAR(200) | | | 조건부 필수 조건 설명 |
| APPLY_START_DT | DATE | NOT NULL | | 적용시작일 |
| APPLY_END_DT | DATE | | | 적용종료일 |
| USE_YN | CHAR(1) | NOT NULL | | 사용여부, Default 'Y' |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_FORM_CODE (
    FORM_CD         VARCHAR(10) PRIMARY KEY             COMMENT '서식코드',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                COMMENT '세목코드(FK)',
    FORM_NM         VARCHAR(200) NOT NULL               COMMENT '서식명',
    RECORD_TYPE     CHAR(1) NOT NULL                    COMMENT '레코드타입(A~Z)',
    REQUIRED_YN     CHAR(1) NOT NULL DEFAULT 'Y'        COMMENT '필수여부(Y/N/C)',
    REQUIRED_COND   VARCHAR(200)                        COMMENT '조건부필수조건',
    APPLY_START_DT  DATE NOT NULL                       COMMENT '적용시작일',
    APPLY_END_DT    DATE                                COMMENT '적용종료일',
    USE_YN          CHAR(1) NOT NULL DEFAULT 'Y'        COMMENT '사용여부',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    CONSTRAINT FK_FORM_TAX_TYPE FOREIGN KEY (TAX_TYPE_CD) REFERENCES MST_TAX_TYPE(TAX_TYPE_CD)
) COMMENT '서식코드 마스터';
```

### 2.3 MST_RECORD_LAYOUT (레코드 레이아웃 메타데이터)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| LAYOUT_ID | BIGINT | NOT NULL | PK | 레이아웃ID (AUTO_INCREMENT) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | FK | 세목코드 |
| FORM_CD | VARCHAR(10) | NOT NULL | FK | 서식코드 |
| RECORD_TYPE | CHAR(1) | NOT NULL | | 레코드구분 (A~Z) |
| RECORD_NM | VARCHAR(100) | NOT NULL | | 레코드명 |
| RECORD_LEN | INT | NOT NULL | | 레코드길이 (바이트) |
| REQUIRED_YN | CHAR(1) | | | 필수여부, Default 'Y' |
| APPLY_START_DT | DATE | NOT NULL | | 적용시작일 |
| APPLY_END_DT | DATE | | | 적용종료일 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_RECORD_LAYOUT (
    LAYOUT_ID       BIGINT AUTO_INCREMENT PRIMARY KEY   COMMENT '레이아웃ID',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                COMMENT '세목코드',
    FORM_CD         VARCHAR(10) NOT NULL                COMMENT '서식코드',
    RECORD_TYPE     CHAR(1) NOT NULL                    COMMENT '레코드구분(A~Z)',
    RECORD_NM       VARCHAR(100) NOT NULL               COMMENT '레코드명',
    RECORD_LEN      INT NOT NULL                        COMMENT '레코드길이(바이트)',
    REQUIRED_YN     CHAR(1) DEFAULT 'Y'                 COMMENT '필수여부',
    APPLY_START_DT  DATE NOT NULL                       COMMENT '적용시작일',
    APPLY_END_DT    DATE                                COMMENT '적용종료일',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    INDEX IDX_LAYOUT_TAX_FORM (TAX_TYPE_CD, FORM_CD),
    CONSTRAINT FK_LAYOUT_TAX_TYPE FOREIGN KEY (TAX_TYPE_CD) REFERENCES MST_TAX_TYPE(TAX_TYPE_CD),
    CONSTRAINT FK_LAYOUT_FORM_CD FOREIGN KEY (FORM_CD) REFERENCES MST_FORM_CODE(FORM_CD)
) COMMENT '레코드 레이아웃 메타데이터';
```

### 2.4 MST_FIELD_LAYOUT (필드 레이아웃 메타데이터)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| FIELD_ID | BIGINT | NOT NULL | PK | 필드ID (AUTO_INCREMENT) |
| LAYOUT_ID | BIGINT | NOT NULL | FK | 레이아웃ID |
| FIELD_SEQ | INT | NOT NULL | | 필드순서 |
| FIELD_NM | VARCHAR(100) | NOT NULL | | 필드명 |
| FIELD_START | INT | NOT NULL | | 시작위치 (바이트) |
| FIELD_LEN | INT | NOT NULL | | 필드길이 (바이트) |
| DATA_TYPE | CHAR(2) | NOT NULL | | 데이터타입 (C/N/CN) |
| REQUIRED_TYPE | CHAR(1) | | | 필수유형 (M/C/O), Default 'O' |
| VALID_VALUES | VARCHAR(500) | | | 허용값목록 (JSON 배열) |
| VALIDATION_TYPE | VARCHAR(30) | | | 검증타입 (BRN/RRN/DATE 등) |
| DESCRIPTION | VARCHAR(500) | | | 필드설명 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_FIELD_LAYOUT (
    FIELD_ID        BIGINT AUTO_INCREMENT PRIMARY KEY   COMMENT '필드ID',
    LAYOUT_ID       BIGINT NOT NULL                     COMMENT '레이아웃ID(FK)',
    FIELD_SEQ       INT NOT NULL                        COMMENT '필드순서',
    FIELD_NM        VARCHAR(100) NOT NULL               COMMENT '필드명',
    FIELD_START     INT NOT NULL                        COMMENT '시작위치(바이트)',
    FIELD_LEN       INT NOT NULL                        COMMENT '필드길이(바이트)',
    DATA_TYPE       CHAR(2) NOT NULL                    COMMENT '데이터타입(C:문자,N:숫자,CN:혼합)',
    REQUIRED_TYPE   CHAR(1) DEFAULT 'O'                 COMMENT '필수유형(M:필수,C:조건부,O:선택)',
    VALID_VALUES    VARCHAR(500)                        COMMENT '허용값목록(JSON배열)',
    VALIDATION_TYPE VARCHAR(30)                         COMMENT '검증타입(BRN/RRN/DATE 등)',
    DESCRIPTION     VARCHAR(500)                        COMMENT '필드설명',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    INDEX IDX_FIELD_LAYOUT (LAYOUT_ID, FIELD_SEQ),
    CONSTRAINT FK_FIELD_LAYOUT FOREIGN KEY (LAYOUT_ID) REFERENCES MST_RECORD_LAYOUT(LAYOUT_ID)
) COMMENT '필드 레이아웃 메타데이터';
```

### 2.5 MST_VALIDATION_RULE (검증 규칙 마스터)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| RULE_ID | VARCHAR(20) | NOT NULL | PK | 규칙ID (예: VAT-C001) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | FK | 세목코드 |
| RULE_NM | VARCHAR(200) | NOT NULL | | 규칙명 |
| RULE_LEVEL | TINYINT | NOT NULL | | 검증레벨 (1~5) |
| SEVERITY | VARCHAR(10) | NOT NULL | | 심각도 (FATAL/ERROR/WARNING/INFO) |
| RULE_FORMULA | TEXT | | | 검증수식/조건 (표현식) |
| LEGAL_BASIS | VARCHAR(200) | | | 법적근거 (법조항) |
| PENALTY_REF | VARCHAR(200) | | | 가산세참조 |
| ERROR_MSG | VARCHAR(500) | NOT NULL | | 오류메시지 (템플릿, {field} 치환) |
| FIX_GUIDE | VARCHAR(500) | | | 수정가이드 |
| APPLY_START_DT | DATE | NOT NULL | | 적용시작일 |
| APPLY_END_DT | DATE | | | 적용종료일 |
| USE_YN | CHAR(1) | | | 사용여부, Default 'Y' |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_VALIDATION_RULE (
    RULE_ID         VARCHAR(20) PRIMARY KEY             COMMENT '규칙ID(예:VAT-C001)',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                COMMENT '세목코드',
    RULE_NM         VARCHAR(200) NOT NULL               COMMENT '규칙명',
    RULE_LEVEL      TINYINT NOT NULL                    COMMENT '검증레벨(1~5)',
    SEVERITY        VARCHAR(10) NOT NULL                COMMENT '심각도(FATAL/ERROR/WARNING/INFO)',
    RULE_FORMULA    TEXT                                COMMENT '검증수식/조건(표현식)',
    LEGAL_BASIS     VARCHAR(200)                        COMMENT '법적근거(법조항)',
    PENALTY_REF     VARCHAR(200)                        COMMENT '가산세참조',
    ERROR_MSG       VARCHAR(500) NOT NULL               COMMENT '오류메시지(템플릿)',
    FIX_GUIDE       VARCHAR(500)                        COMMENT '수정가이드',
    APPLY_START_DT  DATE NOT NULL                       COMMENT '적용시작일',
    APPLY_END_DT    DATE                                COMMENT '적용종료일',
    USE_YN          CHAR(1) DEFAULT 'Y'                 COMMENT '사용여부',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    INDEX IDX_RULE_TAX_LEVEL (TAX_TYPE_CD, RULE_LEVEL),
    CONSTRAINT FK_RULE_TAX_TYPE FOREIGN KEY (TAX_TYPE_CD) REFERENCES MST_TAX_TYPE(TAX_TYPE_CD)
) COMMENT '검증 규칙 마스터';
```

### 2.6 MST_TAX_RATE (세율표)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| RATE_ID | BIGINT | NOT NULL | PK | 세율ID (AUTO_INCREMENT) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | FK | 세목코드 |
| TAX_YEAR | CHAR(4) | NOT NULL | | 귀속연도 |
| BRACKET_SEQ | INT | NOT NULL | | 구간순서 |
| BASE_FROM | BIGINT | NOT NULL | | 과세표준 시작 (원) |
| BASE_TO | BIGINT | | | 과세표준 종료 (원, NULL=초과) |
| RATE_PERCENT | DECIMAL(5,2) | NOT NULL | | 세율 (%) |
| DEDUCTION_AMT | BIGINT | | | 누진공제액 (원) |
| DESCRIPTION | VARCHAR(200) | | | 비고 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_TAX_RATE (
    RATE_ID         BIGINT AUTO_INCREMENT PRIMARY KEY   COMMENT '세율ID',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                COMMENT '세목코드',
    TAX_YEAR        CHAR(4) NOT NULL                    COMMENT '귀속연도',
    BRACKET_SEQ     INT NOT NULL                        COMMENT '구간순서',
    BASE_FROM       BIGINT NOT NULL                     COMMENT '과세표준시작(원)',
    BASE_TO         BIGINT                              COMMENT '과세표준종료(원,NULL=초과)',
    RATE_PERCENT    DECIMAL(5,2) NOT NULL               COMMENT '세율(%)',
    DEDUCTION_AMT   BIGINT                              COMMENT '누진공제액(원)',
    DESCRIPTION     VARCHAR(200)                        COMMENT '비고',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    UNIQUE INDEX UDX_RATE (TAX_TYPE_CD, TAX_YEAR, BRACKET_SEQ),
    CONSTRAINT FK_RATE_TAX_TYPE FOREIGN KEY (TAX_TYPE_CD) REFERENCES MST_TAX_TYPE(TAX_TYPE_CD)
) COMMENT '세율표 (귀속연도별)';
```

### 2.7 MST_DEDUCTION_LIMIT (공제/감면 한도표)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| LIMIT_ID | BIGINT | NOT NULL | PK | 한도ID (AUTO_INCREMENT) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | FK | 세목코드 |
| TAX_YEAR | CHAR(4) | NOT NULL | | 귀속연도 |
| DEDUCTION_CD | VARCHAR(20) | NOT NULL | | 공제/감면 종류코드 |
| DEDUCTION_NM | VARCHAR(200) | NOT NULL | | 공제/감면명 |
| LIMIT_AMT | BIGINT | | | 한도금액 (원) |
| LIMIT_RATE | DECIMAL(5,2) | | | 한도비율 (%) |
| CONDITION_DESC | VARCHAR(500) | | | 적용조건 설명 |
| LEGAL_BASIS | VARCHAR(200) | | | 법적근거 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_DEDUCTION_LIMIT (
    LIMIT_ID        BIGINT AUTO_INCREMENT PRIMARY KEY   COMMENT '한도ID',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                COMMENT '세목코드',
    TAX_YEAR        CHAR(4) NOT NULL                    COMMENT '귀속연도',
    DEDUCTION_CD    VARCHAR(20) NOT NULL                COMMENT '공제/감면종류코드',
    DEDUCTION_NM    VARCHAR(200) NOT NULL               COMMENT '공제/감면명',
    LIMIT_AMT       BIGINT                              COMMENT '한도금액(원)',
    LIMIT_RATE      DECIMAL(5,2)                        COMMENT '한도비율(%)',
    CONDITION_DESC  VARCHAR(500)                        COMMENT '적용조건설명',
    LEGAL_BASIS     VARCHAR(200)                        COMMENT '법적근거',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    UNIQUE INDEX UDX_DEDUCTION (TAX_TYPE_CD, TAX_YEAR, DEDUCTION_CD),
    CONSTRAINT FK_DED_TAX_TYPE FOREIGN KEY (TAX_TYPE_CD) REFERENCES MST_TAX_TYPE(TAX_TYPE_CD)
) COMMENT '공제/감면 한도표';
```

### 2.8 MST_CODE (공통코드)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| CODE_GRP | VARCHAR(20) | NOT NULL | PK | 코드그룹 |
| CODE_CD | VARCHAR(20) | NOT NULL | PK | 코드값 |
| CODE_NM | VARCHAR(100) | NOT NULL | | 코드명 (한글) |
| CODE_NM_EN | VARCHAR(100) | | | 코드명 (영문) |
| CODE_DESC | VARCHAR(200) | | | 코드설명 |
| SORT_ORDER | INT | NOT NULL | | 표시순서 |
| PARENT_CD | VARCHAR(20) | | | 상위코드 |
| USE_YN | CHAR(1) | NOT NULL | | 사용여부, Default 'Y' |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_CODE (
    CODE_GRP        VARCHAR(20) NOT NULL                COMMENT '코드그룹',
    CODE_CD         VARCHAR(20) NOT NULL                COMMENT '코드값',
    CODE_NM         VARCHAR(100) NOT NULL               COMMENT '코드명(한글)',
    CODE_NM_EN      VARCHAR(100)                        COMMENT '코드명(영문)',
    CODE_DESC       VARCHAR(200)                        COMMENT '코드설명',
    SORT_ORDER      INT NOT NULL DEFAULT 0              COMMENT '표시순서',
    PARENT_CD       VARCHAR(20)                         COMMENT '상위코드',
    USE_YN          CHAR(1) NOT NULL DEFAULT 'Y'        COMMENT '사용여부',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    PRIMARY KEY (CODE_GRP, CODE_CD)
) COMMENT '공통코드';
```

**주요 코드그룹**:

| CODE_GRP | 설명 | 예시 |
|----------|------|------|
| RPT_TYPE | 신고구분 | 01:정기, 02:수정, 03:기한후, 04:경정청구 |
| TAX_PERIOD | 귀속기간 (부가세) | 01:1기예정, 02:1기확정, 03:2기예정, 04:2기확정 |
| CORP_TYPE | 법인구분 | 01:내국영리, 02:내국비영리, 03:외국영리 |
| BAD_DEBT | 대손사유 | 01:부도, 02:강제집행, 03:회생/파산 |
| DATA_TYPE | 필드데이터타입 | C:문자, N:숫자, CN:혼합 |
| REQUIRED_TYPE | 필수유형 | M:필수, C:조건부, O:선택 |
| INC_TYPE | 소득종류 (원천세) | A01:근로, A02:퇴직, A03:사업 |

### 2.9 MST_ERROR_CODE (오류코드 정의)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| ERROR_CD | VARCHAR(20) | NOT NULL | PK | 오류코드 (예: VAT-F001) |
| RULE_ID | VARCHAR(20) | NOT NULL | FK | 검증규칙ID |
| ERROR_MSG_TEMPLATE | VARCHAR(500) | NOT NULL | | 오류메시지 템플릿 |
| FIX_GUIDE_TEMPLATE | VARCHAR(500) | | | 수정가이드 템플릿 |
| LEGAL_BASIS | VARCHAR(200) | | | 법적근거 |
| PENALTY_INFO | VARCHAR(200) | | | 가산세 정보 |
| USE_YN | CHAR(1) | NOT NULL | | 사용여부, Default 'Y' |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE MST_ERROR_CODE (
    ERROR_CD            VARCHAR(20) PRIMARY KEY          COMMENT '오류코드',
    RULE_ID             VARCHAR(20) NOT NULL             COMMENT '검증규칙ID(FK)',
    ERROR_MSG_TEMPLATE  VARCHAR(500) NOT NULL            COMMENT '오류메시지 템플릿',
    FIX_GUIDE_TEMPLATE  VARCHAR(500)                     COMMENT '수정가이드 템플릿',
    LEGAL_BASIS         VARCHAR(200)                     COMMENT '법적근거',
    PENALTY_INFO        VARCHAR(200)                     COMMENT '가산세정보',
    USE_YN              CHAR(1) NOT NULL DEFAULT 'Y'     COMMENT '사용여부',
    reg_dt              DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id              VARCHAR(50) NOT NULL,
    chg_dt              DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id              VARCHAR(50),
    CONSTRAINT FK_ERR_RULE FOREIGN KEY (RULE_ID) REFERENCES MST_VALIDATION_RULE(RULE_ID)
) COMMENT '오류코드 정의';
```

---

## 3. Request Tables (요청/입력 - REQ_)

### 3.1 REQ_VALIDATION (검증 요청)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| VALIDATION_ID | BIGINT | NOT NULL | PK | 검증요청ID (AUTO_INCREMENT) |
| REQUEST_NO | VARCHAR(20) | NOT NULL | UNQ | 요청번호 (YYYYMMDD-NNNN) |
| USER_ID | VARCHAR(50) | NOT NULL | | 요청자ID |
| STATUS | VARCHAR(10) | NOT NULL | | 상태 (PENDING/PROCESSING/COMPLETED/FAILED) |
| TOTAL_FILE_CNT | INT | NOT NULL | | 총 파일 수 |
| COMPLETED_FILE_CNT | INT | NOT NULL | | 완료 파일 수, Default 0 |
| REQUEST_DT | DATETIME | NOT NULL | | 요청일시 |
| COMPLETE_DT | DATETIME | | | 완료일시 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE REQ_VALIDATION (
    VALIDATION_ID       BIGINT AUTO_INCREMENT PRIMARY KEY  COMMENT '검증요청ID',
    REQUEST_NO          VARCHAR(20) NOT NULL              COMMENT '요청번호',
    USER_ID             VARCHAR(50) NOT NULL              COMMENT '요청자ID',
    STATUS              VARCHAR(10) NOT NULL DEFAULT 'PENDING' COMMENT '상태',
    TOTAL_FILE_CNT      INT NOT NULL DEFAULT 1            COMMENT '총파일수',
    COMPLETED_FILE_CNT  INT NOT NULL DEFAULT 0            COMMENT '완료파일수',
    REQUEST_DT          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '요청일시',
    COMPLETE_DT         DATETIME                          COMMENT '완료일시',
    reg_dt              DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id              VARCHAR(50) NOT NULL,
    chg_dt              DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id              VARCHAR(50),
    UNIQUE INDEX UDX_REQUEST_NO (REQUEST_NO),
    INDEX IDX_VALID_USER (USER_ID, STATUS)
) COMMENT '검증 요청';
```

### 3.2 REQ_VALIDATION_FILE (검증 대상 파일)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| FILE_ID | BIGINT | NOT NULL | PK | 파일ID (AUTO_INCREMENT) |
| VALIDATION_ID | BIGINT | NOT NULL | FK | 검증요청ID |
| ORIGINAL_FILE_NM | VARCHAR(255) | NOT NULL | | 원본 파일명 |
| STORED_FILE_NM | VARCHAR(255) | NOT NULL | | 저장 파일명 (UUID) |
| FILE_PATH | VARCHAR(500) | NOT NULL | | 파일 저장 경로 |
| FILE_SIZE | BIGINT | NOT NULL | | 파일 크기 (bytes) |
| FILE_EXT | VARCHAR(10) | NOT NULL | | 파일 확장자 |
| TAX_TYPE_CD | VARCHAR(10) | | | 식별된 세목코드 |
| ENCODING | VARCHAR(10) | | | 감지된 인코딩 |
| IS_MERGED | CHAR(1) | NOT NULL | | 합본파일여부, Default 'N' |
| MERGED_CNT | INT | | | 합본 내 신고건수 |
| STATUS | VARCHAR(10) | NOT NULL | | 상태 (UPLOADED/PARSING/VALIDATING/DONE/ERROR) |
| DELETED_AT | DATETIME | | | 파일 삭제 일시 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE REQ_VALIDATION_FILE (
    FILE_ID             BIGINT AUTO_INCREMENT PRIMARY KEY  COMMENT '파일ID',
    VALIDATION_ID       BIGINT NOT NULL                   COMMENT '검증요청ID(FK)',
    ORIGINAL_FILE_NM    VARCHAR(255) NOT NULL             COMMENT '원본파일명',
    STORED_FILE_NM      VARCHAR(255) NOT NULL             COMMENT '저장파일명(UUID)',
    FILE_PATH           VARCHAR(500) NOT NULL             COMMENT '파일저장경로',
    FILE_SIZE           BIGINT NOT NULL                   COMMENT '파일크기(bytes)',
    FILE_EXT            VARCHAR(10) NOT NULL              COMMENT '파일확장자',
    TAX_TYPE_CD         VARCHAR(10)                       COMMENT '식별된세목코드',
    ENCODING            VARCHAR(10)                       COMMENT '감지된인코딩',
    IS_MERGED           CHAR(1) NOT NULL DEFAULT 'N'      COMMENT '합본파일여부',
    MERGED_CNT          INT                               COMMENT '합본내신고건수',
    STATUS              VARCHAR(10) NOT NULL DEFAULT 'UPLOADED' COMMENT '상태',
    DELETED_AT          DATETIME                          COMMENT '파일삭제일시',
    reg_dt              DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id              VARCHAR(50) NOT NULL,
    chg_dt              DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id              VARCHAR(50),
    INDEX IDX_FILE_VALID (VALIDATION_ID),
    CONSTRAINT FK_FILE_VALIDATION FOREIGN KEY (VALIDATION_ID) REFERENCES REQ_VALIDATION(VALIDATION_ID)
) COMMENT '검증 대상 파일 정보';
```

---

## 4. Result Tables (점검/결과 - RES_)

### 4.1 RES_VALIDATION_RESULT (검증 결과 요약)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| RESULT_ID | BIGINT | NOT NULL | PK | 결과ID (AUTO_INCREMENT) |
| FILE_ID | BIGINT | NOT NULL | FK | 파일ID |
| MERGED_SEQ | INT | | | 합본 내 순번 (합본 파일 시) |
| TAX_TYPE_CD | VARCHAR(10) | NOT NULL | | 세목코드 |
| BRN | VARCHAR(10) | | | 사업자등록번호 |
| TAXPAYER_NM | VARCHAR(100) | | | 납세자명 |
| TAX_YEAR | CHAR(4) | | | 귀속연도 |
| TAX_PERIOD | VARCHAR(10) | | | 귀속기간 |
| TOTAL_RULE_CNT | INT | NOT NULL | | 적용된 전체 규칙 수 |
| PASS_CNT | INT | NOT NULL | | 통과 건수 |
| FATAL_CNT | INT | NOT NULL | | Fatal 건수 |
| ERROR_CNT | INT | NOT NULL | | Error 건수 |
| WARNING_CNT | INT | NOT NULL | | Warning 건수 |
| INFO_CNT | INT | NOT NULL | | Info 건수 |
| OVERALL_RESULT | VARCHAR(10) | NOT NULL | | 종합결과 (PASS/FAIL) |
| PROCESSING_MS | BIGINT | | | 처리시간 (ms) |
| VALIDATED_AT | DATETIME | NOT NULL | | 검증완료일시 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE RES_VALIDATION_RESULT (
    RESULT_ID       BIGINT AUTO_INCREMENT PRIMARY KEY    COMMENT '결과ID',
    FILE_ID         BIGINT NOT NULL                      COMMENT '파일ID(FK)',
    MERGED_SEQ      INT                                  COMMENT '합본내순번',
    TAX_TYPE_CD     VARCHAR(10) NOT NULL                 COMMENT '세목코드',
    BRN             VARCHAR(10)                          COMMENT '사업자등록번호',
    TAXPAYER_NM     VARCHAR(100)                         COMMENT '납세자명',
    TAX_YEAR        CHAR(4)                              COMMENT '귀속연도',
    TAX_PERIOD      VARCHAR(10)                          COMMENT '귀속기간',
    TOTAL_RULE_CNT  INT NOT NULL DEFAULT 0               COMMENT '적용규칙수',
    PASS_CNT        INT NOT NULL DEFAULT 0               COMMENT '통과건수',
    FATAL_CNT       INT NOT NULL DEFAULT 0               COMMENT 'Fatal건수',
    ERROR_CNT       INT NOT NULL DEFAULT 0               COMMENT 'Error건수',
    WARNING_CNT     INT NOT NULL DEFAULT 0               COMMENT 'Warning건수',
    INFO_CNT        INT NOT NULL DEFAULT 0               COMMENT 'Info건수',
    OVERALL_RESULT  VARCHAR(10) NOT NULL                  COMMENT '종합결과(PASS/FAIL)',
    PROCESSING_MS   BIGINT                               COMMENT '처리시간(ms)',
    VALIDATED_AT    DATETIME NOT NULL                     COMMENT '검증완료일시',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    INDEX IDX_RESULT_FILE (FILE_ID),
    INDEX IDX_RESULT_TAX (TAX_TYPE_CD, TAX_YEAR),
    CONSTRAINT FK_RESULT_FILE FOREIGN KEY (FILE_ID) REFERENCES REQ_VALIDATION_FILE(FILE_ID)
) COMMENT '검증 결과 요약';
```

### 4.2 RES_VALIDATION_DETAIL (검증 결과 상세)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| DETAIL_ID | BIGINT | NOT NULL | PK | 상세ID (AUTO_INCREMENT) |
| RESULT_ID | BIGINT | NOT NULL | FK | 결과ID |
| SEQ | INT | NOT NULL | | 순번 |
| RULE_ID | VARCHAR(20) | NOT NULL | FK | 규칙ID |
| SEVERITY | VARCHAR(10) | NOT NULL | | 심각도 |
| RECORD_TYPE | CHAR(1) | | | 오류 레코드타입 |
| RECORD_SEQ | INT | | | 오류 레코드순번 |
| FIELD_NM | VARCHAR(100) | | | 오류 필드명 |
| FIELD_POS | VARCHAR(20) | | | 오류 위치 (바이트) |
| ACTUAL_VALUE | VARCHAR(500) | | | 파일 내 실제값 |
| EXPECTED_VALUE | VARCHAR(500) | | | 기대값 |
| ERROR_MSG | VARCHAR(500) | NOT NULL | | 오류메시지 |
| LEGAL_BASIS | VARCHAR(200) | | | 법적근거 |
| FIX_GUIDE | VARCHAR(500) | | | 수정가이드 |
| reg_dt ~ reg_id | | | | 등록 감사 컬럼 |

```sql
CREATE TABLE RES_VALIDATION_DETAIL (
    DETAIL_ID       BIGINT AUTO_INCREMENT PRIMARY KEY    COMMENT '상세ID',
    RESULT_ID       BIGINT NOT NULL                      COMMENT '결과ID(FK)',
    SEQ             INT NOT NULL                         COMMENT '순번',
    RULE_ID         VARCHAR(20) NOT NULL                 COMMENT '규칙ID(FK)',
    SEVERITY        VARCHAR(10) NOT NULL                 COMMENT '심각도',
    RECORD_TYPE     CHAR(1)                              COMMENT '오류레코드타입',
    RECORD_SEQ      INT                                  COMMENT '오류레코드순번',
    FIELD_NM        VARCHAR(100)                         COMMENT '오류필드명',
    FIELD_POS       VARCHAR(20)                          COMMENT '오류위치(바이트)',
    ACTUAL_VALUE    VARCHAR(500)                         COMMENT '파일값',
    EXPECTED_VALUE  VARCHAR(500)                         COMMENT '기대값',
    ERROR_MSG       VARCHAR(500) NOT NULL                COMMENT '오류메시지',
    LEGAL_BASIS     VARCHAR(200)                         COMMENT '법적근거',
    FIX_GUIDE       VARCHAR(500)                         COMMENT '수정가이드',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    INDEX IDX_DETAIL_RESULT (RESULT_ID),
    INDEX IDX_DETAIL_SEVERITY (SEVERITY),
    CONSTRAINT FK_DETAIL_RESULT FOREIGN KEY (RESULT_ID) REFERENCES RES_VALIDATION_RESULT(RESULT_ID),
    CONSTRAINT FK_DETAIL_RULE FOREIGN KEY (RULE_ID) REFERENCES MST_VALIDATION_RULE(RULE_ID)
) COMMENT '검증 결과 상세 (오류 목록)';
```

### 4.3 RES_VALIDATION_REPORT (검증 리포트)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| REPORT_ID | BIGINT | NOT NULL | PK | 리포트ID (AUTO_INCREMENT) |
| RESULT_ID | BIGINT | NOT NULL | FK | 결과ID |
| REPORT_TYPE | VARCHAR(10) | NOT NULL | | 리포트유형 (SUMMARY/DETAIL/PDF/EXCEL) |
| REPORT_FILE_NM | VARCHAR(255) | | | 리포트 파일명 |
| REPORT_FILE_PATH | VARCHAR(500) | | | 리포트 파일 경로 |
| REPORT_DATA | LONGTEXT | | | 리포트 데이터 (JSON) |
| GENERATED_AT | DATETIME | NOT NULL | | 생성일시 |
| reg_dt ~ chg_id | | | | 공통 감사 컬럼 |

```sql
CREATE TABLE RES_VALIDATION_REPORT (
    REPORT_ID       BIGINT AUTO_INCREMENT PRIMARY KEY    COMMENT '리포트ID',
    RESULT_ID       BIGINT NOT NULL                      COMMENT '결과ID(FK)',
    REPORT_TYPE     VARCHAR(10) NOT NULL                 COMMENT '리포트유형(SUMMARY/DETAIL/PDF/EXCEL)',
    REPORT_FILE_NM  VARCHAR(255)                         COMMENT '리포트파일명',
    REPORT_FILE_PATH VARCHAR(500)                        COMMENT '리포트파일경로',
    REPORT_DATA     LONGTEXT                             COMMENT '리포트데이터(JSON)',
    GENERATED_AT    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성일시',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    chg_dt          DATETIME ON UPDATE CURRENT_TIMESTAMP,
    chg_id          VARCHAR(50),
    INDEX IDX_REPORT_RESULT (RESULT_ID),
    CONSTRAINT FK_REPORT_RESULT FOREIGN KEY (RESULT_ID) REFERENCES RES_VALIDATION_RESULT(RESULT_ID)
) COMMENT '검증 리포트';
```

---

## 5. History Tables (관리/로그 - HIS_)

### 5.1 HIS_VALIDATION_LOG (검증 처리 로그)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| LOG_ID | BIGINT | NOT NULL | PK | 로그ID (AUTO_INCREMENT) |
| VALIDATION_ID | BIGINT | NOT NULL | FK | 검증요청ID |
| FILE_ID | BIGINT | | FK | 파일ID |
| LOG_LEVEL | VARCHAR(10) | NOT NULL | | 로그레벨 (INFO/WARN/ERROR) |
| LOG_PHASE | VARCHAR(20) | NOT NULL | | 처리단계 (UPLOAD/PARSE/L1~L5/REPORT) |
| LOG_MSG | VARCHAR(1000) | NOT NULL | | 로그메시지 |
| ELAPSED_MS | BIGINT | | | 소요시간 (ms) |
| LOG_DT | DATETIME | NOT NULL | | 로그일시 |
| reg_dt | DATETIME | NOT NULL | | 등록일시 |
| reg_id | VARCHAR(50) | NOT NULL | | 등록자 |

```sql
CREATE TABLE HIS_VALIDATION_LOG (
    LOG_ID          BIGINT AUTO_INCREMENT PRIMARY KEY    COMMENT '로그ID',
    VALIDATION_ID   BIGINT NOT NULL                      COMMENT '검증요청ID',
    FILE_ID         BIGINT                               COMMENT '파일ID',
    LOG_LEVEL       VARCHAR(10) NOT NULL                 COMMENT '로그레벨(INFO/WARN/ERROR)',
    LOG_PHASE       VARCHAR(20) NOT NULL                 COMMENT '처리단계(UPLOAD/PARSE/L1~L5/REPORT)',
    LOG_MSG         VARCHAR(1000) NOT NULL               COMMENT '로그메시지',
    ELAPSED_MS      BIGINT                               COMMENT '소요시간(ms)',
    LOG_DT          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '로그일시',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    INDEX IDX_LOG_VALID (VALIDATION_ID),
    INDEX IDX_LOG_DT (LOG_DT),
    INDEX IDX_LOG_PHASE (LOG_PHASE)
) COMMENT '검증 처리 로그';
```

**보관 정책**: 감사 로그 5년 보관 (법적 의무)

### 5.2 HIS_RULE_CHANGE (규칙 변경 이력)

| Column | Type | Nullable | Key | Description |
|--------|------|----------|-----|-------------|
| CHANGE_ID | BIGINT | NOT NULL | PK | 변경ID (AUTO_INCREMENT) |
| RULE_ID | VARCHAR(20) | NOT NULL | FK | 규칙ID |
| CHANGE_TYPE | VARCHAR(10) | NOT NULL | | 변경유형 (CREATE/UPDATE/DELETE) |
| BEFORE_DATA | TEXT | | | 변경 전 데이터 (JSON) |
| AFTER_DATA | TEXT | | | 변경 후 데이터 (JSON) |
| CHANGE_REASON | VARCHAR(500) | | | 변경사유 |
| CHANGED_BY | VARCHAR(50) | NOT NULL | | 변경자 |
| CHANGED_AT | DATETIME | NOT NULL | | 변경일시 |
| reg_dt | DATETIME | NOT NULL | | 등록일시 |
| reg_id | VARCHAR(50) | NOT NULL | | 등록자 |

```sql
CREATE TABLE HIS_RULE_CHANGE (
    CHANGE_ID       BIGINT AUTO_INCREMENT PRIMARY KEY    COMMENT '변경ID',
    RULE_ID         VARCHAR(20) NOT NULL                 COMMENT '규칙ID',
    CHANGE_TYPE     VARCHAR(10) NOT NULL                 COMMENT '변경유형(CREATE/UPDATE/DELETE)',
    BEFORE_DATA     TEXT                                 COMMENT '변경전데이터(JSON)',
    AFTER_DATA      TEXT                                 COMMENT '변경후데이터(JSON)',
    CHANGE_REASON   VARCHAR(500)                         COMMENT '변경사유',
    CHANGED_BY      VARCHAR(50) NOT NULL                 COMMENT '변경자',
    CHANGED_AT      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '변경일시',
    reg_dt          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    reg_id          VARCHAR(50) NOT NULL,
    INDEX IDX_CHANGE_RULE (RULE_ID),
    INDEX IDX_CHANGE_DT (CHANGED_AT)
) COMMENT '규칙 변경 이력';
```

---

## 6. Index Strategy

### 6.1 Primary Indexes

| Table | Index | Columns | Type |
|-------|-------|---------|------|
| MST_RECORD_LAYOUT | IDX_LAYOUT_TAX_FORM | TAX_TYPE_CD, FORM_CD | Composite |
| MST_FIELD_LAYOUT | IDX_FIELD_LAYOUT | LAYOUT_ID, FIELD_SEQ | Composite |
| MST_VALIDATION_RULE | IDX_RULE_TAX_LEVEL | TAX_TYPE_CD, RULE_LEVEL | Composite |
| MST_TAX_RATE | UDX_RATE | TAX_TYPE_CD, TAX_YEAR, BRACKET_SEQ | Unique |
| MST_DEDUCTION_LIMIT | UDX_DEDUCTION | TAX_TYPE_CD, TAX_YEAR, DEDUCTION_CD | Unique |
| REQ_VALIDATION | UDX_REQUEST_NO | REQUEST_NO | Unique |
| REQ_VALIDATION | IDX_VALID_USER | USER_ID, STATUS | Composite |
| RES_VALIDATION_RESULT | IDX_RESULT_TAX | TAX_TYPE_CD, TAX_YEAR | Composite |
| RES_VALIDATION_DETAIL | IDX_DETAIL_SEVERITY | SEVERITY | Single |
| HIS_VALIDATION_LOG | IDX_LOG_DT | LOG_DT | Single |

### 6.2 Index Design Principles

- 선택도 높은 컬럼 (ID, 이메일, 요청번호) 우선 인덱싱
- 복합 인덱스: WHERE 필터 + ORDER BY 순서 배치
- 과도한 인덱스 지양 (CUD 성능 저하 방지)
- 파티셔닝: HIS_VALIDATION_LOG는 월단위 파티셔닝 검토

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2026-02-19 | Initial schema: 16 tables (MST 9 + REQ 2 + RES 3 + HIS 2), full DDL, index strategy | Claude |

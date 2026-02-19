# Design Validator Memory - TaxServiceENTEC-FEV

## Project Overview
- Backend-first Java/Spring Boot 3.x project
- Tax file error validation system (FEV)
- Enterprise level, Clean Architecture
- 16 DB tables (MST 9, REQ 2, RES 3, HIS 2)

## Key Documents
- Plan: `docs/01-plan/features/tax-file-error-validation.plan.md`
- Design: `docs/02-design/features/tax-file-error-validation.design.md`
- Schema: `docs/01-plan/schema.md` (16 tables, full DDL)
- Glossary: `docs/01-plan/glossary.md`
- Domain Model: `docs/01-plan/domain-model.md`

## Validation History
- 2026-02-19 (v1.0.0): Initial validation, score 82/100
  - Critical: 7/16 JPA entities missing, entity-schema column mismatches
  - Critical: Missing deduction-limits admin API
  - Major: Clean Architecture violation (JPA annotations in domain entities)
  - Major: API path structure differs from Plan doc

- 2026-02-19 (v1.1.0): Re-validation, score 95.3/100
  - All 9 previous issues fixed (100% fix rate)
  - 16/16 entities, 215/215 columns matched (100%)
  - 27+ API endpoints, Clean Arch compliant
  - Remaining: 5 Major (API improvements), 7 Minor (polish)
  - Status: IMPLEMENTATION APPROVED

## Known Design Patterns
- Error codes: `{TAX_TYPE}-{LEVEL}{NUMBER}` (e.g., VAT-C001)
- API error codes: `FILE_001`, `VALID_001`, `AUTH_001`, `SYSTEM_001`
- DB naming: UPPER_SNAKE_CASE with MST_/REQ_/RES_/HIS_ prefixes
- Audit columns: lower_snake_case (reg_dt, reg_id, chg_dt, chg_id)

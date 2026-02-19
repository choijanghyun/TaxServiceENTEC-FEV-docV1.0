# Product Manager Agent Memory - TaxServiceENTEC-FEV

## 프로젝트 기본 정보
- 프로젝트명: TaxServiceENTEC-FEV (Tax File Conversion Error Validation)
- 경로: C:\nts_dev\TaxServiceENTEC-FEV-docV1.0
- 기술 스택: Java 17+, Spring Boot 3.x, JPA+MyBatis, MySQL8/PG15, JWT, Docker
- 아키텍처: Enterprise Level (레이어드: presentation/application/domain/infrastructure)

## 완료된 문서
- refer-doc/전자신고_파일오류검증_기능요구사항_사용자스토리.md (2026-02-19 최초 작성)
- docs/01-plan/features/tax-file-error-validation.plan.md (2026-02-19 최초 작성)

## 핵심 기능 ID 체계
- FR-001: 파일 파서 엔진 | FR-002: 5단계 검증 파이프라인
- FR-003: 세목별 검증 (VAT/CIT/INC/WHT/CGT) | FR-004: 규칙 관리
- FR-005: 리포팅 | FR-006: 공통 유틸리티 | FR-007: 파일뷰어 | FR-008: 일괄처리

## Phase 로드맵
- Phase 1 (MVP): FR-001, FR-002 L1~L3, FR-003 VAT, FR-004, FR-005 기본, FR-006
- Phase 2: FR-002 L4~L5, FR-003 CIT/INC/WHT/CGT, FR-001 대용량, FR-005 PDF/Excel
- Phase 3: FR-007 뷰어, FR-008 일괄처리, 크로스 세목 검증
- Phase 4: AI 패턴 분석, 홈택스 연동 (미래 검토)

## 사용자 유형 4가지
1. 세무사/세무법인 (파일 사전 오류 검증)
2. 기장대리 담당자 (대량 일괄 검증)
3. 세무프로그램 개발사 (API 연동 품질 검증)
4. 관리자 (규칙/세율표/레이아웃 관리)

## 참조 파일 위치
- refer-doc/전자신고_파일오류검증_핵심기능_요구사항.md (세목별 검증규칙 상세)
- refer-doc/전자신고_파일오류검증_관리기능_요구사항.md (DB설계/코딩표준)

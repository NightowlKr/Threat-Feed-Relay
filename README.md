# Threat Feed Relay

Collect. Enrich. Profile. Relay.

외부 Threat/Blacklist Feed를 수집하고 IP·CIDR·Domain·URL 지표를 정규화·보강한 뒤,
Profile별 Output Feed로 재배포하는 플랫폼의 개발 요구 문서 저장소입니다.
Docker·소스 설치, DNS/ASN, LDAP/SAML, 복수 DB·HA·출력 Feed 중심 샤딩을 포함합니다.

현재 단계는 **구현 전 문서 초안**입니다. 애플리케이션, 실행 가능한 배포 설정,
마이그레이션과 자동화 테스트는 아직 포함하지 않습니다.

## 문서 상태

2026-09-09에 이전 대화에서 확인 가능한 내용을 기준으로 새로 작성했습니다.
이전 `c9c1843` 커밋의 원본 16개 파일을 복원하거나 동일성을 검증한 결과는 아닙니다.
과거 대화의 기능 범위와 이번에 작성한 상세 설계·수용기준을 구분합니다.
출처와 상태의 의미는 [문서 기준선](DOCUMENTATION_BASELINE.md)을 참고하세요.

## 문서 안내

| 문서 | 용도 |
| --- | --- |
| [문서 기준선](DOCUMENTATION_BASELINE.md) | 근거, 범위와 미확정 사항의 표시 방법 |
| [대화 반영표](CONVERSATION_COVERAGE.md) | 로컬 3개·클라우드 1개 작업과 상세 대화 대조 |
| [Git 운영 규칙](GIT_WORKFLOW.md) | 역할별 브랜치·검토·통합 원칙 |
| [프로젝트 명세](PROJECT_SPEC.md) | 목표, 사용자, 주요 사용 흐름과 범위 |
| [요구사항](REQUIREMENTS.md) | 요구사항 ID와 제안 수용기준 |
| [아키텍처](ARCHITECTURE.md) | 구성 요소와 작업·데이터 경계 |
| [데이터베이스 설계](DATABASE_DESIGN.md) | 개념 데이터 모델과 정합성 |
| [Feed 파이프라인](FEED_PIPELINE.md) | 수집, 정규화, 보강과 실패 처리 |
| [DNS·ASN 보강](ENRICHMENT.md) | Resolver 비교·변화 이력과 LPM·배치 조회 |
| [제공자 연동](PROVIDERS.md) | 공공데이터·C-TAS·외부 Feed 계약 |
| [클러스터·샤딩](CLUSTER_SHARDING.md) | DB HA, 출력 Feed 분할과 일관성 |
| [인증·권한](IDENTITY_ACCESS.md) | 로컬·LDAP/AD·SAML와 내부 RBAC |
| [Profile·Allowlist](PROFILES_ALLOWLIST.md) | 정책과 제외 규칙 |
| [API 계약](API_CONTRACTS.md) | 관리 API와 Output Feed 계약 초안 |
| [배포·보안](DISTRIBUTION_SECURITY.md) | 구성, 접근 제어와 외부 통신 |
| [운영·백업](OPERATIONS_BACKUP.md) | 상태 확인, 복구와 운영 절차 |
| [개발 계획](DEVELOPMENT_PLAN.md) | 단계별 산출물과 완료 조건 |
| [검증 체크리스트](VERIFICATION_CHECKLIST.md) | 구현 후 수용 검증 |
| [결정 로그](DECISION_LOG.md) | 구현 전에 확정할 의사결정 |
| [AI 작업 규칙](AGENTS.md) | 저장소 작업 안내 |
| [AI 작업 템플릿](AI_TASK_TEMPLATE.md) | 구현 작업 요청 양식 |

## 시작 순서

1. 문서 기준선과 요구사항을 읽고 결정 로그의 해당 항목을 확정합니다.
2. 승인된 테스트 Feed와 기대 출력으로 최소 처리 경로를 구현합니다.
3. 요구사항 ID를 구현·검증 결과와 연결하고 개발 계획의 단계별 완료 조건을 확인합니다.

기술 후보·책임은 [아키텍처](ARCHITECTURE.md), 남은 상세 결정은 [결정 로그](DECISION_LOG.md)를 따릅니다.
현재 작업 브랜치는 `feature/development-requirements`이며 main 직접 반영·병합은 하지 않습니다.

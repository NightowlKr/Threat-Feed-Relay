# 아키텍처

상태: 전달된 최종 기술 대화의 기준선과 상세 설계 제안.
현재 버전의 설치·지원·호환성을 검증한 문서가 아니다.

## 책임 경계

| 구성 | 책임 | 쓰기 소유권 |
| --- | --- | --- |
| Laravel Control Plane | 관리 UI/API, 인증·권한, 설정·정책, 예약·작업 상태, 게시 승인 | Control DB, 스키마 버전·라우팅·공개 포인터 |
| Python Pipeline | 수집·파싱·정규화·DNS·ASN·정제·집계·artifact 생성 | 계약으로 위임된 관측·보강·membership·작업 결과 |
| Redis/Valkey 호환 계층 | 캐시·세션·락, 관리 큐, 파이프라인 메시지 | 임시 전달 상태; 영구 작업 원장은 Control DB |
| 데이터 계층 | 공통 DB 기능과 선택적 Timescale 시계열·샤드 | DB별 공통 계약과 확장 계약 분리 |
| 원문·Artifact 저장소 | 제한된 원문 및 검증된 불변 출력 | 파일 완성·검증 후 DB 참조 연결 |
| Reverse proxy | TLS, 요청 제한, 인증 후 파일 전송 | 내부 저장 경로 직접 공개 금지 |

UI는 Laravel 프로젝트의 `resources/js`에 Inertia 기반으로 통합한다.
Python은 초기에는 외부 API 서버 없이 CLI 작업자로 동작한다.
FastAPI·Celery·Alembic·별도 프런트엔드 서버를 필수로 추가하지 않는다.

## 대화의 기술 후보

| 영역 | 최종 대화 후보 | 구현 때 확인할 사항 |
| --- | --- | --- |
| 관리 | Laravel 13, PHP 8.5, Composer 2 | 실제 패키지·확장 호환성 |
| UI | Inertia 3, React 19, TypeScript, Tailwind 4, Vite | 통합 빌드·브라우저 지원 |
| Python | 3.13 기준, 3.14 호환 CI 후보, uv | 실행·라이브러리 호환성 |
| Python 라이브러리 | HTTPX, dnspython, SQLAlchemy Core, Pydantic, redis-py, structlog | 버전 고정·라이선스·부하 시험 |
| 공통 DB | PostgreSQL 18, MariaDB 11.4/11.8 후보 | 동일 기능 계약 시험과 드라이버 |
| 확장 DB | 선택 PostgreSQL 버전과 호환되는 TimescaleDB | 실제 지원 조합·확장 기능 경계 |
| Laravel 보조 도구 | Fortify, Sanctum, Horizon 후보 | LDAP/SAML·큐 계약과 중복 여부 |

버전은 이 표를 단일 참조로 사용한다. 정확한 조합은 D-06에 검증 근거를 기록한 후 고정한다.
Laravel 14 검토와 2029~2030년 유지보수 요구는 지속 업그레이드 전략이지 특정 버전 고정이나 지원 기간 보증이 아니다.

## 작업 전달 계약

- 관리용 Laravel 작업과 Python 파이프라인 전달을 구분한다.
- 언어 중립 JSON 메시지와 Streams consumer group을 기준선으로 삼는다. Laravel 직렬화 객체를 Python에 전달하지 않는다.
- 메시지는 `schema_version`, `job_id`, `attempt_id`, 입력·정책 참조, 추적 ID를 가진다. 비밀값 본문은 넣지 않는다.
- Control DB에 jobs/attempts/events 및 transactional outbox를 유지한다.
- 발행 재시도·중복 배달·consumer 재시작을 가정하며, 결과 커밋 후 ACK하고 중복 결과 반영을 막는다.
- 작업 상태의 최종 변경은 Control Plane의 결과 반영 계약으로 통일한다. Python이 임의로 게시 포인터를 바꾸지 않는다.
- 락에는 만료·갱신·fencing을 적용하고 scheduler는 단일 실행을 보장한다.
- Redis의 논리 DB 번호를 서비스 격리 계약으로 고정하지 않는다. 키 namespace·권한·배포 토폴로지로 구분한다.
- 상태는 `queued`, `running`, `completed`, `completed_with_errors`, `failed`, `cancelled`, `retry_wait`로 통일한다.
- 부분 성공은 자동 게시 승인이 아니다. 필수 단계의 완전성 조건을 별도로 검사한다.

## 저장·스키마 경계

Control DB, canonical observation, 출력 Feed별 membership, immutable artifact를 분리한다.
Standard는 MariaDB/PostgreSQL의 공통 동작을 제공하고 Advanced는 Timescale 기능·샤딩을 추가한다.
Timescale을 모든 설치의 필수 의존성으로 만들지 않는다.

Control DB 마이그레이션은 Laravel이 소유한다.
샤드 스키마는 버전 관리 SQL과 Laravel 실행기로 관리하며 Python은 별도 마이그레이션을 실행하지 않는다.
릴리스에는 DBMS별 SQL·검증·호환 범위를 포함하고 Expand → Backfill → Switch → Contract로 전개한다.
긴 외부 I/O 동안 DB 트랜잭션을 유지하거나 DB와 파일 저장소 간 분산 ACID를 가정하지 않는다.

[데이터 모델](DATABASE_DESIGN.md), [클러스터·샤딩](CLUSTER_SHARDING.md),
[파이프라인](FEED_PIPELINE.md), [배포](DISTRIBUTION_SECURITY.md)가 상세 계약을 정의한다.

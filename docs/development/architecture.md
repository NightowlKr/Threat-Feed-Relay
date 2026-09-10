# 아키텍처·데이터 설계

기술 후보·작업 전달·스키마 소유권·DB 및 샤드 정합성의 기준 문서입니다.
상태와 검증 한계는 [개발 문서 안내](README.md)를 따릅니다.

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
| 공통 DB | PostgreSQL 18, MariaDB 11.8 | 동일 기능 계약 시험과 드라이버 |
| 확장 DB | 선택 PostgreSQL 버전과 호환되는 TimescaleDB | 실제 지원 조합·확장 기능 경계 |
| Laravel 보조 도구 | Fortify, Sanctum, Horizon 후보 | LDAP/SAML·큐 계약과 중복 여부 |

버전은 이 표를 단일 참조로 사용한다. 정확한 조합은 D-06에 검증 근거를 기록한 후 고정한다.
2029\~2030년 유지보수 요구는 지속 업그레이드 전략이지 특정 버전 고정이나 지원 기간 보증이 아니다.

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

## 데이터 모델

### 논리 엔터티

| 엔터티 | 핵심 필드 | 관계·제약 제안 |
| --- | --- | --- |
| `source` | id, name, endpoint_ref, enabled, parser_version | 인증값은 저장하지 않고 비밀 참조만 저장 |
| `collection_run` | id, source_id, started_at, ended_at, status, error_code | 실행마다 하나; 실패도 기록 |
| `source_snapshot` | id, run_id, content_hash, collected_at, completeness, status | 검증을 통과한 완전본만 활성 후보 |
| `indicator` | id, type, canonical_value, normalization_version | 세 필드 조합 고유; 원문과 구분 |
| `observation` | snapshot_id, indicator_id, raw_value, source_timestamp | 동일 지표의 복수 출처를 보존 |
| `source_indicator_state` | source_id, indicator_id, first_seen, last_seen, expires_at, status | 출처별 현재 상태를 빠르게 조회 |
| `enrichment` | id, indicator_id, kind, provider, observed_at, expires_at, status, payload | 공급자·시점별 결과 유지 |
| `profile_version` | profile_id, version, definition, content_hash, created_at | 발행된 버전은 변경하지 않음 |
| `allowlist_version` | allowlist_id, version, entries, content_hash, created_at | 활성 버전과 이력 분리 |
| `generation_run` | id, profile_version_id, input_manifest, status, generated_at | 사용한 스냅샷·정책·보강 결과 ID와 수명 평가 근거 고정 |
| `artifact` | id, generation_run_id, format, path_ref, checksum, item_count, status | 검증된 생성 실행에 종속 |
| `publication` | profile_id, output_contract_version, manifest_id, revision, published_at | 검증된 artifact 묶음의 현재 포인터 |

`input_manifest`는 Source 스냅샷, Profile·Allowlist 그룹·예외 규칙 집합의 정확한 버전,
parser/normalizer·DNS/ASN 정책과 보강 결과, 평가 시각, 출력 계약 및 routing epoch를 고정한다.
해당 버전은 불변이며 생성·재시도 중 최신 설정을 다시 읽지 않는다.
예외 규칙은 비활성·삭제된 항목도 과거 버전에서 해석 가능하게 보존한다.
버전 집합의 해시·참조로 재현하고 정책 변경은 새 generation에서만 적용한다.

### 정체성과 중복 제거

1. 지표 키는 `type + canonical_value + normalization_version`으로 구성한다.
2. 동일 IP가 여러 소스에 존재해도 지표는 하나로 표현하고 관측 관계는 모두 남긴다.
3. 한 소스의 중복 행은 집계할 수 있으나 수집 건수와 중복 건수는 별도로 기록한다.
4. 정규화 버전 변경은 기존 값을 덮어쓰지 않고 재처리·이관 기록으로 연결한다.
5. 전체 원문 보존 범위와 기간은 [D-03](workflow.md), 정규화 규칙은 D-02에서 확정한다.

### 스냅샷과 실패 처리

- 수집 실행의 성공과 스냅샷의 발행 가능 여부는 별도 상태로 관리한다.
- 부분 다운로드·파싱 중단·검증 실패는 활성 스냅샷을 교체하지 않는다.
- 빈 응답은 자동으로 정상적인 전체 삭제로 해석하지 않는다.
- 소스가 전체 목록인지 증분 목록인지는 어댑터 계약에 명시한다.
- 전체 목록에서의 누락만 해당 출처의 비활성 후보로 처리한다.
- 증분 목록에서는 명시적인 삭제 이벤트 또는 만료 정책으로 상태를 갱신한다.
- 한 출처가 제거되어도 다른 유효 출처가 있으면 통합 지표는 유지한다.
- 이전 정상 스냅샷 사용 여부와 최대 허용 경과 시간은 D-03, D-07에서 결정한다.

### 시간과 유효기간

- 시각은 UTC로 저장하고 소스 제공 시각과 시스템 관측 시각을 분리한다.
- 보존 중인 Source-지표 관계의 `first_seen`은 최초 실제 관측 시각으로 유지한다. 다음 정상 수집은 `last_seen`을 갱신하되 최초 시각을 다시 만들지 않으며, 제거 후 재등장은 별도 활성 구간으로 기록한다.
- `expires_at`은 출처별 지표 상태와 보강 결과에 각각 존재한다.
- DNS 캐시 TTL은 threat 지표 자체의 보존 기간을 의미하지 않는다.
- 생성 작업은 고정한 기준 시각에서 만료 여부를 평가한다.
- 만료 지표의 출력 제외와 관측 이력의 물리 삭제는 독립된 정책으로 둔다.
- 보존 기간이 확정되기 전에는 숫자 기본값을 운영 기준으로 취급하지 않는다.

### 수명 평가와 이력 보존

Source 최신성, Domain-IP 관계의 유효성, Output 제공 가능 여부, 이력의 물리 보존을 분리한다.
아래 필드·규칙은 설계 제안이며 Source/보강 정책 버전에 포함한다.

| 개념 | 기준·계약 |
| --- | --- |
| `last_seen` | 지표 또는 Domain-IP 관계를 실제로 확인한 시각; 출처 관측과 DNS 양성 관측의 값을 구분 |
| `freshness_basis`, `freshness_at` | 일반 Source는 검증 완료한 수집 실행의 성공 시각, 기간 자료는 검증된 최신 대상 기간의 기준 시각으로 평가; 사용한 기준·시각을 스냅샷에 고정 |
| `stale_after`, `expire_after` | 기준 시각부터 경고·기여 만료까지의 기간; 유한한 값이며 `0 < stale_after < expire_after`; DNS 관계는 pipeline의 전용 필드와 last_seen 사용 |
| `retention` | 활성 출력에서 빠진 뒤에도 이력을 조회하기 위한 별도 물리 보존 정책; 만료 시 자동 DB 삭제를 뜻하지 않음 |
| `serve_until` | Output manifest에 고정한 입력·정책으로 계산한 제공 상한 시각; artifact 생성·재게시만으로 연장 불가 |

기간 자료의 freshness 기준은 어댑터가 발행/대상 기간과 완전성을 확인해 정한다. 과거 backfill이나 같은 기간 재다운로드로 기간 기준을 최신 시각으로 바꾸지 않는다.
일반 Source의 동일 해시 재수집은 기존 FR-005처럼 새 관측이며 성공 시각을 갱신하되, 실패는 어느 freshness 기준도 갱신하지 않는다.
이는 외부에서 새로 수집·검증한 실행에 적용한다. 로컬 캐시 재사용·복사·재생성만으로 `last_seen`, 최근 성공 시각, `freshness_at` 또는 만료 기한을 갱신하지 않는다. 캐시 전달 계약은 [수집·보강](pipeline.md#완전성과-캐시-재사용)을 따른다.
DNS 관계의 missing/음성 확인 시각·카운터·종료 사유와 상태 전이는 [수집·보강](pipeline.md)을 단일 기준으로 사용한다.
생성 입력 manifest는 수명 정책 버전·기준 시각·선택/제외한 Source 및 기여의 사유·기여별 만료 시각도 고정한다.
재현은 고정 평가 시각으로 수행하고, 실제 다운로드 가능 여부는 [Output 수명 계약](profiles-api.md)에 따라 현재 시각·회수 상태로 별도 검사한다.
retention 정리에서도 현재 제공본·보존 중 revision·진행 중 작업·백업이 참조하는 필수 입력과 정책은 참조 보호한다.

### 일관성과 검증

- 스냅샷 검증 완료 표시와 활성화는 하나의 DB 트랜잭션으로 처리한다.
- 같은 논리 실행의 재배달은 `collection_run_id + stage` 등 실행 단위 키로 탐지한다. 재시도 attempt가 달라도 결과는 한 번만 반영한다.
- `content_hash`는 원문 blob 재사용에만 사용한다. 별도 정기 실행에서 내용이 같아도 정상 관측·최근 성공 시각·last_seen·수명 갱신을 생략하지 않는다.
- 동일 해시 재수집, 동일 작업 재배달, 제거 후 동일 내용 재등장을 각각 검증한다.
- 생성 중 입력이 변경되어도 입력 manifest에 고정한 버전만 조회한다.
- DB와 파일 저장소를 하나의 트랜잭션으로 가정하지 않는다; 먼저 산출물을 완성한 뒤 포인터를 전환한다.
- 포인터 전환은 예상 공개 revision과 현재 게시가 허용된 정책·입력 세대를 조건으로 검사한다.
- 최신 정책의 성공본 뒤에 완료된 구세대 작업은 산출물만 보관하고 현재 공개 포인터를 되돌리지 않는다.
- 고아 임시 산출물은 실행 상태와 대조하여 정리하고 공개 포인터가 가리키는 파일은 보존한다.
- 필수 검증 사례는 복수 출처, 부분 수집 실패, 만료 경계, 생성 중 정책 변경, 게시 실패 복구이다.

### DB별 주소 표현과 계층

MariaDB는 INET4/INET6와 prefix length를 분리하고 주소 계열·정규 network·range start/end를 저장하는 방향이다.
PostgreSQL은 inet/cidr를 활용하되 공통 주소·포함·LPM 계약을 유지한다.
실제 타입 지원·드라이버 직렬화·인덱스는 DB별 실행 시험으로 확인한다.
DB 함수 차이는 repository/query adapter에 모은다.

Control DB는 설정·정책·인증·권한·작업 원장·outbox·routing catalog를 소유한다.
관측 저장소는 canonical 지표와 출처 관측, membership 저장소는 출력 Feed별 정제 상태와 사건을 보유한다.
Timescale 확장과 물리 배치는 이 문서의 HA·출력 Feed 샤딩 절을 따른다.

### 추가 논리 엔터티

| 엔터티 | 목적과 주요 제약 |
| --- | --- |
| `parser_version`, `resolver_policy`, `asn_dataset` | 파서·보강의 불변 설정과 활성 버전 |
| `dns_observation`, `dns_membership_interval` | 조회 회차·질의 타입·Resolver별 응답, 부모/출처/정책별 도메인-IP 상태·누락 사유와 활성 구간 |
| `profile_source`, `profile_allowlist_group` | Profile의 다중 소스·그룹과 정확한 버전 연결 |
| `exception_rule_set_version`, `monitor_rule`, `alert_event` | 불변 예외 집합·적용 범위 및 원본 관측 기반 경보 분리 |
| `membership_interval` | Feed별 정제 지표의 활성 [start, end) 구간 |
| `publication_membership_interval` | 실제 공개 스냅샷에 포함된 지표의 구간 |
| `snapshot_manifest`, `snapshot_segment` | 정책·입력·수명 평가 근거·serve_until·routing epoch·필수 샤드·artifact 해시 목록 |
| `identity_provider`, `external_identity`, `role_assignment` | 불변 provider/subject, 권한 부여 출처와 회수 |
| `job`, `job_attempt`, `outbox_event`, `consumed_message` | 영구 작업 상태·재시도·결과 중복 방지 |
| `shard_catalog`, `bucket_mapping`, `indicator_directory`, `global_rollup` | 라우팅 버전과 재구축 가능한 조회 요약 |

### 등록 기간과 집계

최초·최근 원본 관측, 정제 membership, 실제 게시 membership의 시각을 분리한다.
제거·재등장을 별도 구간으로 남기고 현재 연속 기간과 누적 기간을 구분한다.
중첩 구간을 이중 합산하지 않으며 생성만 성공하고 게시 실패한 결과는 배포 기간에 넣지 않는다.
다운로드 시각은 게시 기간과 별도의 접근 로그이다.
제공 기한 만료·정책 회수로 새 revision 없이 제공을 중단한 경우에도 실제 게시 membership 구간을 종료하고 사유를 남긴다.
distinct_source_count와 /24별 distinct_ip_count를 구분하고 Source 태그로 근거를 조회한다.
`distinct_source_count`는 등록된 Source ID의 수이며 독립된 증거 수나 악성 확률을 의미하지 않는다. 같은 제공자의 하위 목록·재집계 Feed가 겹칠 수 있으므로 독립성을 근거 없이 가정하지 않는다. 이를 신뢰 점수나 선택 조건에 사용할 때의 출처 계보·중복 가중 정책은 D-01/D-04에서 별도로 결정한다.
rollup에는 기준 시각·지연을 표시하고 원본 이력과 재대조할 수 있어야 한다.
전체 테이블의 소유권·고유 키·재처리 경계를 명시하고 Laravel의 단일 스키마 소유권을 따른다.

## HA·출력 Feed 샤딩

### 저장 계층과 지원 수준

MariaDB/Galera는 애플리케이션 쓰기를 단일 writer로 라우팅하는 기준선이다.
PostgreSQL/Timescale의 복제·리더 전환은 검증한 운영 구성을 사용한다.
권한·토큰 회수·작업 소유권·라우팅·공개 포인터는 지연된 read replica 판단을 허용하지 않는다.
웹 노드는 키·세션·락·파일 접근 정책을 공유하고 scheduler는 singleton, worker는 멱등 실행한다.
DB HA는 애플리케이션 샤딩이나 백업의 대체물이 아니다.

### 라우팅 모델

| 경계 | 키와 목적 |
| --- | --- |
| 출력 Feed | `distribution_profile_id`로 소유권·정책 경계 설정 |
| 가상 버킷 | Feed 안에서 IOC group의 안정적인 해시로 결정 |
| 물리 샤드 | catalog의 버전 있는 bucket-to-shard mapping으로 배치 |
| IOC group | IPv4 /24, 설정 가능한 IPv6 prefix, PSL 기반 registrable domain 또는 URL host |

IPv6 /64와 특정 bucket 수는 대화의 예시이지 강제 기본값이 아니다.
정규화·PSL·grouping·hash 버전을 고정한다.
IPv4 /24 승격 대상과 그 IP는 같은 Feed 안에서 같은 그룹으로 계산한다.
큰 CIDR을 모든 개별 IP로 펼치지 않는다. 여러 그룹에 걸친 대역은 별도 range 경로 또는
제한된 중첩 조회로 처리하고, 실제 전략·상한을 D-11에서 확정한다.
URL host가 IP이면 주소 계열 규칙을 사용하고 registrable domain 계산 실패를 임의 값으로 대체하지 않는다.

single/partitioned/dedicated/replicated 배치 모드를 단계적으로 지원한다.
여기서 replicated는 출력 데이터 복제 정책이며 DB 엔진의 복제와 구분한다.
샤드 수와 복제 수준은 Feed별로 선택하고 용량·부하를 근거로 변경한다.

### 스키마·조회

- 샤드별 공통 테이블에 Feed·bucket 키를 둔다. Feed나 bucket마다 테이블/Hypertable을 만들지 않는다.
- 라우팅 catalog는 mapping epoch, schema version, 샤드 상태·용량을 관리한다.
- indicator directory는 지표가 속한 Feed·샤드를 추적하여 이력 조회를 돕는다.
- directory·rollup의 지연과 재구축 상태를 표시하고 정합성 민감 작업의 원장으로 사용하지 않는다.
- 대시보드는 global rollup을 사용하여 매 요청마다 모든 샤드를 전수 조회하지 않는다.
- 스키마 SQL은 Laravel 실행기 한 곳에서 적용하고 모든 샤드의 버전을 점검한다.

### 출력 원자성

생성 manifest에 입력·정책·라우팅 epoch와 필요한 샤드 segment 목록을 고정한다.
각 segment는 동일 snapshot 세대의 건수·해시·상태를 보고한다.
필수 segment 및 설정된 복제 조건을 충족한 경우에만 병합·검증 후 공개 포인터를 전환한다.
누락 샤드를 조용히 제외한 partial Feed를 기본 성공으로 게시하지 않는다.
실패 시 마지막 정상본 유지 또는 게시 중단을 적용하고 stale 기한은 D-05로 제한한다.
구세대 worker는 fencing과 조건부 공개 검사로 최신 결과를 덮어쓸 수 없어야 한다.

### 재균형·복구

온라인 재균형은 후속 단계에서 copy → catch-up → checksum 검증 → fenced cutover → drain으로 수행한다.
동시 쓰기의 전달·중복 제거·epoch 충돌을 정의한 후 활성화하며 분산 ACID를 가정하지 않는다.
전환 실패는 유효한 이전 mapping을 유지하고 검증 전 원본 데이터를 제거하지 않는다.

백업에는 Control DB, 모든 필수 샤드, mapping epoch, directory 재구축 정보,
스키마 버전과 artifact manifest를 포함한다.
서로 다른 시점의 DB 백업만 모아 정합성을 보증하지 않는다.
정합 지점을 확보할 수 없으면 쓰기·게시·재균형을 잠시 중지한 검증 절차를 사용한다.
복구 후 라우팅·membership·segment 건수·해시·권한·공개 revision을 함께 확인한다.

# 데이터 모델 설계안

출처·상태는 [문서 기준선](DOCUMENTATION_BASELINE.md)을 따른다.
논리 모델과 DB별 표현 기준선을 정의한다. 실제 스키마·인덱스·버전은 [D-06](DECISION_LOG.md)에서 검증 후 확정한다.

## 목적과 연결 요구사항

- FR-001: 수집 실행과 완전한 소스 스냅샷을 구분하여 보관한다.
- FR-002, FR-005: 정규화한 지표를 통합하되 각 출처와 관측 이력은 유지한다.
- FR-003: DNS·ASN 등 보강 결과의 유효기간과 실패 상태를 별도로 기록한다.
- FR-004, FR-006, FR-007: Profile·Allowlist 버전과 생성 산출물을 재현 가능하게 연결한다.

상세 요구사항은 [REQUIREMENTS.md](REQUIREMENTS.md), 흐름은 [FEED_PIPELINE.md](FEED_PIPELINE.md)를 따른다.

## 논리 엔터티

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
| `generation_run` | id, profile_version_id, input_manifest, status, generated_at | 사용한 스냅샷·정책·보강 결과 ID 고정 |
| `artifact` | id, generation_run_id, format, path_ref, checksum, item_count, status | 검증된 생성 실행에 종속 |
| `publication` | profile_id, output_contract_version, manifest_id, revision, published_at | 검증된 artifact 묶음의 현재 포인터 |

`input_manifest`는 소스 스냅샷, Allowlist 버전, 정규화 버전, 보강 결과,
실행 기준 시각 및 출력 계약 버전을 포함하는 불변 목록 또는 이에 대응하는 자식 테이블이다.

## 정체성과 중복 제거

1. 지표 키는 `type + canonical_value + normalization_version`으로 구성한다.
2. 동일 IP가 여러 소스에 존재해도 지표는 하나로 표현하고 관측 관계는 모두 남긴다.
3. 한 소스의 중복 행은 집계할 수 있으나 수집 건수와 중복 건수는 별도로 기록한다.
4. 정규화 버전 변경은 기존 값을 덮어쓰지 않고 재처리·이관 기록으로 연결한다.
5. 전체 원문 보존 범위와 기간은 [D-03](DECISION_LOG.md), 정규화 규칙은 D-02에서 확정한다.

## 스냅샷과 실패 처리

- 수집 실행의 성공과 스냅샷의 발행 가능 여부는 별도 상태로 관리한다.
- 부분 다운로드·파싱 중단·검증 실패는 활성 스냅샷을 교체하지 않는다.
- 빈 응답은 자동으로 정상적인 전체 삭제로 해석하지 않는다.
- 소스가 전체 목록인지 증분 목록인지는 어댑터 계약에 명시한다.
- 전체 목록에서의 누락만 해당 출처의 비활성 후보로 처리한다.
- 증분 목록에서는 명시적인 삭제 이벤트 또는 만료 정책으로 상태를 갱신한다.
- 한 출처가 제거되어도 다른 유효 출처가 있으면 통합 지표는 유지한다.
- 이전 정상 스냅샷 사용 여부와 최대 허용 경과 시간은 D-03, D-07에서 결정한다.

## 시간과 유효기간

- 시각은 UTC로 저장하고 소스 제공 시각과 시스템 관측 시각을 분리한다.
- `expires_at`은 출처별 지표 상태와 보강 결과에 각각 존재한다.
- DNS 캐시 TTL은 threat 지표 자체의 보존 기간을 의미하지 않는다.
- 생성 작업은 고정한 기준 시각에서 만료 여부를 평가한다.
- 만료 지표의 출력 제외와 관측 이력의 물리 삭제는 독립된 정책으로 둔다.
- 보존 기간이 확정되기 전에는 숫자 기본값을 운영 기준으로 취급하지 않는다.

## 일관성과 검증

- 스냅샷 검증 완료 표시와 활성화는 하나의 DB 트랜잭션으로 처리한다.
- 중복 실행은 소스·스냅샷 해시 등 멱등성 키로 탐지한다.
- 생성 중 입력이 변경되어도 입력 manifest에 고정한 버전만 조회한다.
- DB와 파일 저장소를 하나의 트랜잭션으로 가정하지 않는다; 먼저 산출물을 완성한 뒤 포인터를 전환한다.
- 포인터 전환은 예상 공개 revision과 현재 게시가 허용된 정책·입력 세대를 조건으로 검사한다.
- 최신 정책의 성공본 뒤에 완료된 구세대 작업은 산출물만 보관하고 현재 공개 포인터를 되돌리지 않는다.
- 고아 임시 산출물은 실행 상태와 대조하여 정리하고 공개 포인터가 가리키는 파일은 보존한다.
- 필수 검증 사례는 복수 출처, 부분 수집 실패, 만료 경계, 생성 중 정책 변경, 게시 실패 복구이다.

관련 문서: [PROFILES_ALLOWLIST.md](PROFILES_ALLOWLIST.md), [VERIFICATION_CHECKLIST.md](VERIFICATION_CHECKLIST.md).

## DB별 주소 표현과 계층

MariaDB는 INET4/INET6와 prefix length를 분리하고 주소 계열·정규 network·range start/end를 저장하는 방향이다.
PostgreSQL은 inet/cidr를 활용하되 공통 주소·포함·LPM 계약을 유지한다.
실제 타입 지원·드라이버 직렬화·인덱스는 DB별 실행 시험으로 확인한다.
사용자 대화의 INT6 표기는 IPv6 주소 타입 INET6 의도로 정리했다.
DB 함수 차이는 repository/query adapter에 모은다.

Control DB는 설정·정책·인증·권한·작업 원장·outbox·routing catalog를 소유한다.
관측 저장소는 canonical 지표와 출처 관측, membership 저장소는 출력 Feed별 정제 상태와 사건을 보유한다.
Timescale 확장과 물리 배치는 [클러스터·샤딩](CLUSTER_SHARDING.md)을 따른다.

## 추가 논리 엔터티

| 엔터티 | 목적과 주요 제약 |
| --- | --- |
| `parser_version`, `resolver_policy`, `asn_dataset` | 파서·보강의 불변 설정과 활성 버전 |
| `dns_observation`, `dns_membership_interval` | Resolver별 응답 및 도메인-IP 활성 구간 |
| `profile_source`, `profile_allowlist_group` | Profile의 다중 소스·그룹과 정확한 버전 연결 |
| `exception_rule`, `monitor_rule`, `alert_event` | 정제 예외와 원본 관측 기반 경보 분리 |
| `membership_interval` | Feed별 정제 지표의 활성 [start, end) 구간 |
| `publication_membership_interval` | 실제 공개 스냅샷에 포함된 지표의 구간 |
| `snapshot_manifest`, `snapshot_segment` | 정책·입력·routing epoch·필수 샤드·artifact 해시 목록 |
| `identity_provider`, `external_identity`, `role_assignment` | 불변 provider/subject, 권한 부여 출처와 회수 |
| `job`, `job_attempt`, `outbox_event`, `consumed_message` | 영구 작업 상태·재시도·결과 중복 방지 |
| `shard_catalog`, `bucket_mapping`, `indicator_directory`, `global_rollup` | 라우팅 버전과 재구축 가능한 조회 요약 |

## 등록 기간과 집계

최초·최근 원본 관측, 정제 membership, 실제 게시 membership의 시각을 분리한다.
제거·재등장을 별도 구간으로 남기고 현재 연속 기간과 누적 기간을 구분한다.
중첩 구간을 이중 합산하지 않으며 생성만 성공하고 게시 실패한 결과는 배포 기간에 넣지 않는다.
다운로드 시각은 게시 기간과 별도의 접근 로그이다.
distinct_source_count와 /24별 distinct_ip_count를 구분하고 Source 태그로 근거를 조회한다.
rollup에는 기준 시각·지연을 표시하고 원본 이력과 재대조할 수 있어야 한다.
전체 테이블의 소유권·고유 키·재처리 경계를 명시하고 Laravel의 단일 스키마 소유권을 따른다.

# DB 고가용성과 출력 Feed 중심 샤딩

연결: FR-015, NFR-004, NFR-005. 사용자 요청은 DB HA와 Timescale 애플리케이션 샤딩,
그리고 외부 출력 Feed를 1차 분할 기준으로 삼는 것이다. 아래 라우팅·전환 계약은 설계 제안이다.

## 저장 계층과 지원 수준

Standard는 MariaDB 또는 PostgreSQL에서 공통 기능을 제공한다.
Advanced는 Timescale의 시계열·집계와 샤드 기능을 추가한다.
Control DB, canonical observation, 출력 Feed membership, 불변 artifact를 구분하며
모든 설치에 Timescale을 강제하지 않는다.

MariaDB/Galera는 애플리케이션 쓰기를 단일 writer로 라우팅하는 기준선이다.
PostgreSQL/Timescale의 복제·리더 전환은 검증한 운영 구성을 사용한다.
권한·토큰 회수·작업 소유권·라우팅·공개 포인터는 지연된 read replica 판단을 허용하지 않는다.
웹 노드는 키·세션·락·파일 접근 정책을 공유하고 scheduler는 singleton, worker는 멱등 실행한다.
DB HA는 애플리케이션 샤딩이나 백업의 대체물이 아니다.

## 라우팅 모델

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

## 스키마·조회

- 샤드별 공통 테이블에 Feed·bucket 키를 둔다. Feed나 bucket마다 테이블/Hypertable을 만들지 않는다.
- 라우팅 catalog는 mapping epoch, schema version, 샤드 상태·용량을 관리한다.
- indicator directory는 지표가 속한 Feed·샤드를 추적하여 이력 조회를 돕는다.
- directory·rollup의 지연과 재구축 상태를 표시하고 정합성 민감 작업의 원장으로 사용하지 않는다.
- 대시보드는 global rollup을 사용하여 매 요청마다 모든 샤드를 전수 조회하지 않는다.
- 스키마 SQL은 Laravel 실행기 한 곳에서 적용하고 모든 샤드의 버전을 점검한다.

## 출력 원자성

생성 manifest에 입력·정책·라우팅 epoch와 필요한 샤드 segment 목록을 고정한다.
각 segment는 동일 snapshot 세대의 건수·해시·상태를 보고한다.
필수 segment 및 설정된 복제 조건을 충족한 경우에만 병합·검증 후 공개 포인터를 전환한다.
누락 샤드를 조용히 제외한 partial Feed를 기본 성공으로 게시하지 않는다.
실패 시 마지막 정상본 유지 또는 게시 중단을 적용하고 stale 기한은 D-05로 제한한다.
구세대 worker는 fencing과 조건부 공개 검사로 최신 결과를 덮어쓸 수 없어야 한다.

## 재균형·복구

온라인 재균형은 후속 단계에서 copy → catch-up → checksum 검증 → fenced cutover → drain으로 수행한다.
동시 쓰기의 전달·중복 제거·epoch 충돌을 정의한 후 활성화하며 분산 ACID를 가정하지 않는다.
전환 실패는 유효한 이전 mapping을 유지하고 검증 전 원본 데이터를 제거하지 않는다.

백업에는 Control DB, 모든 필수 샤드, mapping epoch, directory 재구축 정보,
스키마 버전과 artifact manifest를 포함한다.
서로 다른 시점의 DB 백업만 모아 정합성을 보증하지 않는다.
정합 지점을 확보할 수 없으면 쓰기·게시·재균형을 잠시 중지한 검증 절차를 사용한다.
복구 후 라우팅·membership·segment 건수·해시·권한·공개 revision을 함께 확인한다.

관련 문서: [DB](DATABASE_DESIGN.md), [운영·백업](OPERATIONS_BACKUP.md), [아키텍처](ARCHITECTURE.md).

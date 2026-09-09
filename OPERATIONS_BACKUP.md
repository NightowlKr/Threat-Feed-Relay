# 운영 및 백업·복구 설계 초안

이 문서는 이전 대화 범위에서 새로 작성한 운영 제안이며 실제 운영 실적이 아니다.
연계 요구사항: FR-007(원자적 스냅샷), NFR-001(Docker), NFR-003(운영 복구).
운영 주기, 보존 기간, RPO·RTO와 경보 임계값은 [DECISION_LOG.md](DECISION_LOG.md)의 D-07에서 확정한다.
stale 응답 허용 조건과 최대 제공 기한은 D-05의 출력 계약으로 정한다.
DB와 스토리지 선택은 D-06, Indicator 수명 정책은 D-03에 따른다.

## 운영 상태 모델

- Source별 마지막 시도 시각, 마지막 성공 시각, 처리 결과 및 오류 분류를 기록한다.
- 수집 성공과 Profile 출력 게시 성공을 별도 상태로 기록한다.
- 다운로드 성공만으로 수집 성공 처리하지 않고 파싱·검증·저장 결과까지 확인한다.
- Profile별 현재 revision, 게시 시각, 마지막 성공 게시 및 구성 revision을 기록한다.
- 실패한 시도는 기존 `last_success_at`을 갱신하지 않는다.
- 재시작 중이거나 수집 실패 상태에서도 현재 정상 revision의 식별자를 유지한다.
- Source 사용 중지, 수집 실패, 유효한 빈 Feed, 파싱 실패를 구분한다.
- 빈 결과를 게시할 조건은 D-05에서 확정하며, 전량 파싱 실패를 빈 정상 결과로 처리하지 않는다.

## 스냅샷 게시와 stale 처리

1. 적용할 Source 데이터, Profile 및 Allowlist의 revision을 확정한다.
2. 비공개 임시 위치에 출력 artifact와 메타데이터를 생성한다.
3. 항목 수, 형식, 무결성 해시 및 참조 revision을 검증한다.
4. artifact를 내구성 있게 저장한 뒤 공개 포인터를 해당 revision으로 전환한다.
5. 게시 완료를 기록하고 이전 revision은 확정된 보존 정책에 따라 정리한다.

- DB 트랜잭션과 파일 저장이 분리되면 준비·게시 상태를 두고 중단 후 재조정 절차를 제공한다.
- 현재 공개 포인터가 가리키는 artifact는 정리 대상에서 제외한다.
- 새 스냅샷 실패 시 마지막 정상 출력의 유지 여부는 D-05의 stale 정책에 따른다.
- Source freshness와 Profile 게시 freshness를 분리해 노출하여 오래된 입력의 재게시를 식별한다.
- stale 출력의 최대 허용 기간, 차단 응답 및 알림 기준은 임의로 고정하지 않는다.
- 허용된 stale 응답에는 해당 상태와 원본 revision을 표시하고 정상 갱신으로 기록하지 않는다.

## 백업 단위와 절차

- 백업 단위는 DB, 공개 및 복구에 필요한 스냅샷 artifact, 설정 revision의 일관된 묶음이다.
- 배포 이미지 식별자, DB 스키마 버전, 앱 버전 및 백업 형식 버전을 manifest에 포함한다.
- manifest에는 백업 시각, 각 파일 해시, 현재 공개 revision 및 설정 revision을 기록한다.
- 별도 비밀 관리 시스템은 복구 시 필요한 참조와 접근 복구 절차를 기록한다.
- DB 파일을 실행 중 단순 복사하지 않고 선택한 DB의 일관된 백업 기능을 사용한다.
- 저장소 간 원자적 백업이 불가능하면 게시·설정 변경을 잠시 중지하는 절차를 제안한다.
- 이 구간에서 DB의 참조 revision을 고정하고 해당 불변 artifact를 확보한 뒤 게시를 재개한다.
- 백업 도중 artifact 정리가 진행되지 않도록 참조 보호 또는 정리 작업 중지를 적용한다.
- 백업 완료 표시 전 manifest의 참조 파일과 해시를 검사한다.
- 백업 저장소는 운영 장애의 영향을 함께 받지 않는 위치에 두는 구성을 제안한다.
- 백업 접근 권한, 암호화 및 키 복구 담당을 정하고 비밀값이 포함된 파일을 별도 통제한다.
- 보존 개수·주기·원격 복제 방식은 D-07에서 정하며, 공간 부족을 운영 경보로 처리한다.

## 복구 실행 순서

1. 격리된 환경에 manifest와 호환되는 앱·DB 버전을 준비한다.
2. 백업 manifest, 해시 및 필요한 파일의 완전성을 검증한다.
3. 수집·게시·정리 작업을 중지한 상태로 DB, artifact, 설정을 복원한다.
4. 비밀 참조를 재연결하고 복구 환경에서 실제 비밀값이 로그에 노출되지 않게 확인한다.
5. DB가 참조하는 공개 revision과 artifact, 구성 revision이 일치하는지 확인한다.
6. 대표 Profile의 항목 수·해시·인증·stale 상태 및 다운로드 응답을 검증한다.
7. 복구 검증을 통과한 뒤 API 트래픽과 작업 스케줄을 순서대로 재개한다.
8. 복구 시점 이후 누락 구간을 기록하고 재수집을 수행하며 복구 시간을 측정한다.

- artifact가 빠졌으면 같은 revision으로 다른 내용을 재생성해 게시하지 않는다.
- 재생성이 필요하면 참조 데이터·설정을 확인하고 새 revision으로 검증 및 게시한다.
- 스키마 비호환 백업을 현재 DB에 강제 적용하지 않고 호환 버전 또는 검증된 마이그레이션 경로를 사용한다.
- 복구 훈련의 실행 주기와 성공 기준을 D-07에 기록한다.
- 알림 대상으로 연속 수집 실패, 게시 실패, stale 초과, 백업 실패, 저장 공간 부족을 제안한다.
- 운영 로그에는 작업 ID와 revision을 남겨 수집·게시·복구 사건을 연결한다.

API 상태 노출은 [API_CONTRACTS.md](API_CONTRACTS.md), 접근 통제는
[DISTRIBUTION_SECURITY.md](DISTRIBUTION_SECURITY.md), 복구 검증은
[VERIFICATION_CHECKLIST.md](VERIFICATION_CHECKLIST.md)와 연결한다.

## 로그·대시보드·경보

JSON 로그에 request_id/job_id/collection_run_id/indicator_id/distribution_profile_id/snapshot_id/alert_event_id를 연결한다.
Feed·DNS·ASN·로그인·권한·게시·백업·업데이트 실패와 소비자 접근을 구분한다.
queue depth/worker lag, 파서 오류율, DB 지연, 샤드 용량, 생성 시간, 경보 전달 실패를 측정한다.
health/readiness/version은 최소 정보만 노출하고 상세는 운영 권한으로 제한한다.
Source 태그·distinct source/IP 수, 현재·누적 등록 기간, 보강·게시 freshness를 구분한다.

모니터링은 IP/CIDR·Source, 고유 Source/IP 수, 최초/재발견/지속 조건을 지원한다.
경보 상태는 normal/firing/acknowledged/resolved/suppressed로 구분한다.
dedup key·cooldown·반복 제한·억제·재시도·전달 결과를 기록한다.
email/webhook 등 채널·임계값은 D-07에서 선택하고 webhook에도 외부 통신 제한을 적용한다.
예외 대상이어도 원문 관측 기반 경보를 유지하고 배포 제외 사유를 보여준다.

## 확장 백업 범위

설정·정책·Identity/RBAC, 수집 원문·정제 데이터, DNS/ASN·membership 이력,
배포 artifact, 접근·운영·감사 로그를 정책별로 포함한다.
큰 원문·로그의 선택적 제외는 manifest에 명시하고 전체 백업으로 오표기하지 않는다.
암호화 키는 데이터와 분리하여 복구 가능하게 관리한다.
모든 필수 샤드·라우팅·schema·checkpoint의 정합 지점은 [클러스터·샤딩](CLUSTER_SHARDING.md)을 따른다.
작업 재개는 영구 원장·outbox·멱등성 키를 사용하고 Redis 덤프만을 유일한 원장으로 삼지 않는다.

## 릴리스·업데이트

Git으로 소스를 관리하되 운영 설치는 검증한 release tag/asset을 사용한다.
stable/beta/development 채널과 자동 확인·다운로드·적용 승인을 분리한다.
운영 실행 디렉터리에서 무조건 git pull하거나 프런트엔드를 즉석 빌드하지 않는다.
CI asset·lockfile·DBMS별 SQL·지원 조합·해시·서명을 release manifest에 연결한다.

1. 업데이트 락·migration leader를 확보하고 작업 drain·백업 가능 상태를 확인한다.
2. 앱/PHP/Python/DB/확장·샤드 schema·디스크 preflight를 검사한다.
3. 신뢰 서명·checksum을 검증하고 새 디렉터리 또는 image를 준비한다.
4. DB·샤드·설정·artifact 백업의 완전성을 검증한다.
5. 단일 소유권으로 Expand → Backfill → Switch를 수행하고 이전 코드의 공존 가능성을 확인한다.
6. 릴리스 포인터를 전환하고 장기 실행 worker를 안전하게 재시작한다.
7. health/readiness·권한·대표 Feed·작업 재개를 확인한다.
8. Contract의 파괴적 제거는 별도 호환성 검증과 승인 후 실행한다.

이전 코드가 변경된 schema와 호환될 때만 코드 release를 되돌린다.
비호환 변경은 백업 복원 또는 전방 수정으로 처리하며 DB 자동 rollback 성공을 보장하지 않는다.
DBMS별 실행 SQL·사전/사후 검증·재실행 안전성·부분 샤드 migration 복구를 릴리스에 포함한다.

## 장기 유지보수

지원 조합표·lockfile·DB 계약 CI·의존성/이미지 보안 검사를 유지한다.
Laravel 14·런타임 업그레이드는 별도 feature 브랜치에서 실제 호환을 검증한다.
2029~2030년 운영은 지원 버전으로 계속 이관하는 계획이며 특정 후보 버전의 고정 지원 약속이 아니다.

# 보안·운영 설계

연결: FR-014, FR-016, FR-017, NFR-001~NFR-007.
설치·인증·통신·백업·업데이트의 개발 계약이며 실행 가능한 운영 매뉴얼은 아닙니다.
출력 생성·revision·stale 응답은 [Profile·API](profiles-api.md), DB·샤드 정합성은 [아키텍처](architecture.md)를 따릅니다.

## 배포 경계

- Docker 기반으로 애플리케이션, 수집 작업, 영속 데이터를 구분한다.
- API와 수집기를 별도 프로세스 또는 컨테이너로 분리하는 구성을 제안한다.
- 런타임·DB·작업 큐의 방향은 [아키텍처](architecture.md), 실제 호환 조합은 D-06을 따른다.
- 외부 공개 대상은 확정된 Output Feed API로 제한하고 관리 API 접근 범위는 별도로 지정한다.
- DB, 작업 큐, 내부 상태 조회 포트의 외부 공개를 기본 배포 예제에 포함하지 않는다.
- DB, 출력 스냅샷, 설정의 영속 볼륨을 구분하고 컨테이너 재생성과 데이터 삭제를 분리한다.
- 실행 사용자는 비관리자로 두고 필요한 디렉터리만 쓰기 가능하게 하는 구성을 제안한다.
- 이미지와 설정에 버전을 부여하여 배포 및 복구 대상의 조합을 추적한다.

## 수집 URL 및 네트워크 경계

- 관리자가 등록한 Source URL을 통해서만 외부 Feed를 수집한다.
- 수집에 허용할 프로토콜, 포트 및 대상 범위를 D-01과 D-08에서 확정한다.
- 기본 제안은 HTTPS이며, 예외가 필요하면 Source별 사유와 허용 범위를 기록한다.
- URL의 사용자 정보 영역에 포함된 계정·비밀번호는 거부하고 별도 비밀값 참조를 사용한다.
- 요청 전에 정규화된 호스트를 확인하고 DNS 응답의 모든 IPv4·IPv6 주소를 검사한다.
- 루프백, 사설망, 링크 로컬, 클라우드 메타데이터 등 내부 목적지 접근은 기본 차단을 제안한다.
- 내부 Feed가 필요하면 광범위한 우회 대신 승인된 Source와 네트워크 범위에 한정한다.
- DNS 재조회로 검사 대상과 실제 접속 대상이 달라지지 않도록 연결 단계까지 동일 정책을 적용한다.
- 호스트를 IP로 연결하더라도 원래 호스트의 TLS 인증서 검증을 유지한다.
- 리다이렉트는 기본 비활성화를 제안하며, 허용 시 매 이동마다 URL·DNS·실제 목적지를 재검증한다.
- 다른 호스트로 이동할 때 Source 인증 헤더나 쿠키를 자동 전달하지 않는다.
- 프록시 사용 시에도 목적지 검증을 우회하지 않도록 프록시의 DNS 및 접속 동작을 검토한다.
- 외부 통신은 네트워크 방화벽 또는 egress 정책으로 추가 제한하는 구성을 제안한다.
- Feed 안의 IP·CIDR·Domain·URL은 데이터로 처리하며, 각 Indicator의 주소에 접속하지 않는다.
- 응답 바이트, 압축 해제 크기, 행 길이, 실행 시간 및 동시성의 상한은 D-07에서 정한다.
- TLS 오류나 파싱 실패 시 기존 정상 스냅샷을 덮어쓰지 않는다.

## API 접근과 비밀값

- 관리 인증·RBAC와 아래 4개 Output 접근 모드를 구현하고 남은 매핑·세션 상세는 D-08에서 확정한다.
- Profile을 구분할 수 있는 것과 해당 Feed를 읽을 수 있는 권한은 별개로 검사한다.
- 인증 실패 또는 권한 부족 상태에서 Feed 본문, 비밀 설정, 존재 여부가 불필요하게 드러나지 않도록 한다.
- 비밀값은 저장소, 이미지, 출력 파일 및 일반 로그에 포함하지 않는다.
- 토큰을 사용한다면 URL 쿼리와 경로에 넣지 않는 방식을 우선 제안한다.
- 비밀값은 환경별 비밀 관리 기능 또는 권한을 제한한 파일로 주입하는 방식을 검토한다.
- API 응답은 비밀값의 실제 내용 대신 설정 여부 또는 참조 식별자만 반환한다.
- 접근 로그는 인증 헤더, 쿠키, Source 인증값, 민감한 쿼리를 제거한다.
- 인증값 교체와 폐기는 서비스 운영 절차에 포함하고 변경 주체를 감사 기록에 남긴다.
- 쿠키 인증을 선택하면 CSRF 및 세션 정책을 함께 설계한다. 선택 여부는 D-08에 따른다.
- 요청 제한과 동시 다운로드 제한의 수치는 D-07의 운영 목표로부터 정한다.

## Feed 접근 모드와 trusted proxy

| 모드 | 허용 조건 |
| --- | --- |
| public | 해당 Profile의 명시적 공개 |
| ip_restricted | 실제 소비자 IP가 허용 IP/CIDR에 포함 |
| authenticated | 유효 계정/token과 Profile 읽기 권한; IP 제한 없음 |
| authenticated_ip_restricted | 계정/token·Profile 권한·허용 IP 모두 충족 |

LDAP/SAML·세션·권한 회수는 [인증·권한](security-operations.md)을 따른다.
Feed token을 지원하되 로그인 비밀번호를 다운로드 URL에 넣지 않는다.
신뢰한 proxy CIDR에서 온 Forwarded/X-Forwarded-For만 해석하며 임의 헤더는 무시한다.
다중 proxy 체인·IPv4/IPv6를 검증하고 컨테이너 gateway를 실제 소비자 IP로 오인하지 않는다.
모든 proxy를 신뢰하지 않으며 원본 앱 포트 직접 접근을 차단한다.
TLS 종료 위치와 실제 peer 주소 해석을 배포 설정에 명시한다.

인증 후 Nginx X-Accel-Redirect 등 내부 전송으로 immutable artifact를 제공하는 기준선이다.
내부 파일을 public alias로 공개하지 않는다. 객체 저장소도 동등한 권한·회수 계약을 만족해야 한다.
manifest/checksum/changes/HEAD/304/이전 revision에 동일 권한을 적용한다.
인증 결과를 공유 캐시에 섞지 않으며 정책 변경 후 이전 URL·캐시의 접근 우회를 막는다.

## Docker·소스 설치의 동등성

Docker Compose와 소스 설치는 같은 설정·작업·스키마·출력 계약을 사용한다.
소스 설치는 Nginx/PHP-FPM과 systemd 관리 Laravel queue/scheduler·Python worker를 기준으로 한다.
실제 서비스 수·이름·명령은 구현 때 제공한다. 현재 실행 가능한 설치 명령은 없다.
코드에 Docker 서비스명·고정 /app 경로를 내장하지 않고 데이터·설정·로그·릴리스 경로를 구성 가능하게 한다.
릴리스 전환은 [운영·백업](security-operations.md)을 따른다.

## 인증·RBAC

### 인증 수단과 정체성

Local, LDAP/AD, SAML 2.0 SP, API token을 서로 구분한다.
IdentityProviderInterface 형태의 어댑터 경계를 두되 패키지 호환성은 구현 때 검증한다.
외부 계정의 키는 `provider_id + immutable_subject`이며 이메일만으로 자동 계정 연결하지 않는다.
AD objectGUID, OpenLDAP entryUUID, SAML persistent NameID 또는 불변 claim을 후보로 사용한다.
계정 연결·해제는 권한 있는 명시적 절차와 감사 기록을 요구한다.

로컬 비상 관리자(break-glass)는 외부 IdP 장애에도 사용할 수 있어야 한다.
외부 그룹으로 비상 관리자 권한을 자동 부여하지 않는다.
로컬 암호는 검증된 단방향 해시를 사용하고 MFA·잠금·복구·사용 경보 정책을 둔다.

### LDAP/AD

- LDAPS 또는 STARTTLS와 인증서 검증을 사용하며 평문 bind를 허용하지 않는다.
- 검색용 계정은 최소 권한·비밀 참조로 관리하고 LDAP filter 입력을 안전하게 처리한다.
- bind-and-sync, JIT, scheduled sync, lookup-only를 운영 정책에 따라 구분한다.
- 디렉터리 중첩 그룹·비활성 사용자·이름 변경·페이지네이션과 부분 동기화 실패를 시험한다.
- 전체 동기화 실패나 페이지 누락을 모든 사용자 제거로 해석하지 않는다.

### SAML

SP-initiated를 초기 기준으로 두고 IdP-initiated 및 SLO는 필요 시 별도 검증한다.
복수 IdP, metadata 가져오기, signing certificate 교체, 속성·그룹 매핑을 지원 범위로 둔다.

서명·issuer·audience·destination·recipient·InResponseTo·유효시간을 검사한다.
재사용 assertion을 거부하고 허용 clock skew·신뢰 인증서를 명시한다.
HTTPS ACS, RelayState 목적지 제한, XML 크기·복잡도·외부 엔터티 차단을 적용한다.
인증서 만료·교체와 IdP 장애 시에도 서명 검증을 끄는 fallback은 제공하지 않는다.

### 내부 RBAC와 동기화

외부 그룹 → 내부 역할 → 권한으로 매핑하고 외부 claim을 직접 관리자 권한으로 신뢰하지 않는다.
역할 후보는 super_admin/system_admin/security_admin/feed_admin/feed_operator/feed_reviewer/
monitor_operator/viewer/api_consumer이며 사용자 정의 역할을 지원한다.
실제 기본 역할 수와 권한 목록은 최소 권한 기준으로 확정한다.

- 역할 부여 출처를 local/ldap/saml/manual_override로 기록한다.
- authoritative/additive/login_only/approval_required 동기화 모드를 구분한다.
- authoritative라도 해당 외부 제공자의 매핑 범위만 갱신하고 로컬 부여 역할을 삭제하지 않는다.
- 고권한 자동 부여는 기본 차단하고 별도 승인·감사를 요구한다.
- 사용 중지·그룹 제거·권한 회수는 활성 세션과 token에도 반영한다.
- 회수 지연의 상한과 IdP 장애 중 기존 세션 정책을 명시한다.
- 삭제된 계정의 감사 주체 참조는 보존하고 비활성 주체를 재사용하지 않는다.

게시 승인·복구·업데이트·IdP/역할/샤드 변경은 별도 권한과 필요한 재인증으로 보호한다.
권한·회수 판단은 권위 있는 최신 상태로 수행하며 stale replica나 캐시로 우회하지 않는다.
Feed 소비자의 인증 성공은 모든 Profile에 대한 읽기 권한을 의미하지 않는다.

### 검증

LDAP 비활성화·이름 변경·sync 실패, SAML 서명/시간/재생/잘못된 audience,
그룹 변경 후 세션·token 회수, 비상 계정·IdP 장애, Profile 간 접근 격리를 시험한다.
감사에는 주체·수단·대상·변경 revision·결과·request ID를 남기되 비밀번호·assertion·token은 기록하지 않는다.

## 운영·백업·업데이트

### 운영 상태 모델

- Source별 마지막 시도 시각, 마지막 성공 시각, 처리 결과 및 오류 분류를 기록한다.
- 수집 성공과 Profile 출력 게시 성공을 별도 상태로 기록한다.
- 다운로드 성공만으로 수집 성공 처리하지 않고 파싱·검증·저장 결과까지 확인한다.
- Profile별 현재 revision, 게시 시각, 마지막 성공 게시 및 구성 revision을 기록한다.
- 실패한 시도는 기존 `last_success_at`을 갱신하지 않는다.
- 재시작 중이거나 수집 실패 상태에서도 현재 정상 revision의 식별자를 유지한다.
- Source 사용 중지, 수집 실패, 유효한 빈 Feed, 파싱 실패를 구분한다.
- 빈 결과를 게시할 조건은 D-05에서 확정하며, 전량 파싱 실패를 빈 정상 결과로 처리하지 않는다.

### 백업 단위와 절차

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

### 복구 실행 순서

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

### 로그·대시보드·경보

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

### 확장 백업 범위

설정·정책·Identity/RBAC, 수집 원문·정제 데이터, DNS/ASN·membership 이력,
배포 artifact, 접근·운영·감사 로그를 정책별로 포함한다.
큰 원문·로그의 선택적 제외는 manifest에 명시하고 전체 백업으로 오표기하지 않는다.
암호화 키는 데이터와 분리하여 복구 가능하게 관리한다.
모든 필수 샤드·라우팅·schema·checkpoint의 정합 지점은 [클러스터·샤딩](architecture.md)을 따른다.
작업 재개는 영구 원장·outbox·멱등성 키를 사용하고 Redis 덤프만을 유일한 원장으로 삼지 않는다.

### 릴리스·업데이트

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

### 장기 유지보수

지원 조합표·lockfile·DB 계약 CI·의존성/이미지 보안 검사를 유지한다.
Laravel 14·런타임 업그레이드는 별도 feature 브랜치에서 실제 호환을 검증한다.
2029~2030년 운영은 지원 버전으로 계속 이관하는 계획이며 특정 후보 버전의 고정 지원 약속이 아니다.

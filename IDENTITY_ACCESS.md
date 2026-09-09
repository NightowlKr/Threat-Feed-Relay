# 인증·역할·권한

연결: FR-016, FR-017, NFR-002. 로컬 계정·LDAP/AD·SAML과 내부 권한 관리는 사용자 요청이다.
구체 라이브러리·세션 시간·매핑 기본값은 D-08에서 검증 후 확정한다.

## 인증 수단과 정체성

Local, LDAP/AD, SAML 2.0 SP, API token을 서로 구분한다.
IdentityProviderInterface 형태의 어댑터 경계를 두되 패키지 호환성은 구현 때 검증한다.
외부 계정의 키는 `provider_id + immutable_subject`이며 이메일만으로 자동 계정 연결하지 않는다.
AD objectGUID, OpenLDAP entryUUID, SAML persistent NameID 또는 불변 claim을 후보로 사용한다.
계정 연결·해제는 권한 있는 명시적 절차와 감사 기록을 요구한다.

로컬 비상 관리자(break-glass)는 외부 IdP 장애에도 사용할 수 있어야 한다.
외부 그룹으로 비상 관리자 권한을 자동 부여하지 않는다.
로컬 암호는 검증된 단방향 해시를 사용하고 MFA·잠금·복구·사용 경보 정책을 둔다.

## LDAP/AD

- LDAPS 또는 STARTTLS와 인증서 검증을 사용하며 평문 bind를 허용하지 않는다.
- 검색용 계정은 최소 권한·비밀 참조로 관리하고 LDAP filter 입력을 안전하게 처리한다.
- bind-and-sync, JIT, scheduled sync, lookup-only를 운영 정책에 따라 구분한다.
- 디렉터리 중첩 그룹·비활성 사용자·이름 변경·페이지네이션과 부분 동기화 실패를 시험한다.
- 전체 동기화 실패나 페이지 누락을 모든 사용자 제거로 해석하지 않는다.

## SAML

SP-initiated를 초기 기준으로 두고 IdP-initiated 및 SLO는 필요 시 별도 검증한다.
복수 IdP, metadata 가져오기, signing certificate 교체, 속성·그룹 매핑을 지원 범위로 둔다.

서명·issuer·audience·destination·recipient·InResponseTo·유효시간을 검사한다.
재사용 assertion을 거부하고 허용 clock skew·신뢰 인증서를 명시한다.
HTTPS ACS, RelayState 목적지 제한, XML 크기·복잡도·외부 엔터티 차단을 적용한다.
인증서 만료·교체와 IdP 장애 시에도 서명 검증을 끄는 fallback은 제공하지 않는다.

## 내부 RBAC와 동기화

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

## 검증

LDAP 비활성화·이름 변경·sync 실패, SAML 서명/시간/재생/잘못된 audience,
그룹 변경 후 세션·token 회수, 비상 계정·IdP 장애, Profile 간 접근 격리를 시험한다.
감사에는 주체·수단·대상·변경 revision·결과·request ID를 남기되 비밀번호·assertion·token은 기록하지 않는다.

배포의 4개 접근 모드와 프록시는 [배포·보안](DISTRIBUTION_SECURITY.md),
관리 API는 [API 계약](API_CONTRACTS.md)을 따른다.

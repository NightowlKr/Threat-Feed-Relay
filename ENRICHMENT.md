# DNS·ASN 보강

연결: FR-003, FR-009, FR-010, FR-011.
기능 범위는 사용자 요청이며 상세 상태·필드·정책은 설계 제안이다.

## 지표 기원

`direct_feed`, `derived_domain`, `derived_url_host`, `derived_dns`, `manual`, `derived_asn`을 구분한다.
파생 지표는 부모 지표·출처·정책·보강 버전을 연결한다.
DNS 응답 IP를 직접 수집 악성 IP와 같은 신뢰 점수로 자동 취급하지 않는다.
ASN 대역을 조회했다는 이유만으로 대역 전체를 차단 대상으로 확대하지 않는다.

## Resolver 관리와 조회

시스템 Resolver와 사용자 지정 UDP/TCP Resolver를 초기 범위로 두고 DoT/DoH는 확장 단계로 둔다.
설정에는 이름·주소·프로토콜·우선순위·timeout·재시도·동시성·TCP fallback·EDNS 정책을 둔다.
허용 Resolver의 목적지와 포트를 제한하며 일반 Source SSRF 예외와 분리한다.
상태는 unknown/healthy/degraded/unreachable/disabled로 구분한다.

- A, AAAA, CNAME과 전체 응답 값·TTL·조회 시각·지연·Resolver를 저장한다.
- NOERROR, NXDOMAIN, SERVFAIL, REFUSED, FORMERR, NOTIMP, TIMEOUT, NETWORK_ERROR, INVALID_RESPONSE를 구분한다.
- 응답 비교의 union과 consensus 정책을 구분하고 필요한 Resolver 수를 설정한다.
- 응답 불일치는 관측 사실로 표시한다. 모든 조회 실패를 정상 빈 집합이나 안전한 도메인으로 취급하지 않는다.
- CNAME 루프·깊이·응답 크기를 제한하고 CDN/공유 IP 확대로 인한 오차단을 방지한다.
- URL은 host만 분리하여 DNS 대상으로 사용할 수 있고 URL 자체에 접속하지 않는다.
- Resolver 정책은 Profile 또는 DNS watch 정책에 연결하며 Source마다 강제로 복제하지 않는다.

## DNS 변화 이력

도메인-IP 관계의 first_seen/last_seen, 활성 구간, 관측·Resolver 수, 이전·현재 응답을 저장한다.
added/removed/unchanged/reactivated 및 CNAME 변경·NXDOMAIN·충돌 사건을 구분한다.
한 번의 timeout이나 일시적 NXDOMAIN으로 기존 파생 지표를 즉시 삭제하지 않는다.
제거 유예·연속 관측 기준·stale 사용은 D-03/D-04에 기록하고 TTL과 threat 수명을 분리한다.
직접 지표, DNS 파생 지표, 혼합 출력은 Profile에서 명시적으로 선택한다.

## ASN/CIDR 데이터

저장 범위는 IP 대역, ASN, AS type, owner, country 및 제공자·시각·데이터셋 버전이다.
GIS·지도·정밀 위치 추적은 제외한다.
로컬 데이터셋의 longest-prefix match(LPM)를 우선하고 정책에 따라 외부 조회로 보완한다.
IPv4/IPv6를 분리하며 ASN/소유자/국가 정보는 악성 여부 판단과 구분한다.

- 개별·일괄·예약 조회, 진행률·취소·재개·실패 항목 재시도를 제공한다.
- batch 크기·동시성·호출량·재시도 backoff는 제공자 제한에 맞춰 설정한다.
- 데이터셋은 버전·checksum·건수·수입 상태·이용 조건을 관리하고 검증 후 활성화한다.
- 상태는 pending/matched/not_found/expired/failed/rate_limited/disabled를 구분한다.
- 오래된 결과와 조회 실패를 구분하고 공급자 장애가 원본 관측을 제거하지 않게 한다.
- 개별 결과와 일괄 데이터셋의 충돌·우선순위는 제공자 정책으로 기록한다.

## 수용 검증

동일 도메인의 여러 A/AAAA, CNAME 변화·순환, Resolver 불일치와 전체 실패,
TTL 경계·일시적 제거·재등장을 합성 데이터로 시험한다.
ASN은 중첩 prefix LPM, IPv6, not_found, 데이터셋 교체 실패, rate limit, 배치 재개를 시험한다.
선택한 보강 ID·버전을 출력 입력 manifest에 고정하여 재현성을 확인한다.

관련 문서: [파이프라인](FEED_PIPELINE.md), [DB](DATABASE_DESIGN.md), [Profile](PROFILES_ALLOWLIST.md).

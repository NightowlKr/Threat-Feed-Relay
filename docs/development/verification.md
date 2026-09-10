# 검증 기준

아래 TFR 기능 검증은 모두 미실행입니다. 문서 형식 점검이나 외부 프로젝트의 동작 확인은 TFR 기능·보안·운영 검증을 대신하지 않습니다.
기준 동작은 [요구사항](requirements.md)과 담당 설계를 따르고, 미정 정책은 [결정 로그](workflow.md)에서 먼저 확정합니다.

## 문서 점검

문서 변경마다 다음을 검사하고 실제 결과를 작업 보고에 기록합니다.

- 상대 링크의 대상·대소문자·anchor, 이동 전 파일명 참조의 잔존 여부.
- Markdown 코드블록·표, 충돌 표시·후행 공백·문자 깨짐 및 `git diff --check`.
- FR-001\~FR-020, NFR-001\~NFR-007, D-01\~D-12의 정의와 참조 유지.
- API/작업 상태·버전·보안 정책의 중복 정의와 모순.
- 비밀 패턴·개인 경로·대화 URL·원문 데이터; 패턴 미검출은 완전한 비밀정보 부재 보증이 아님.
- 구현·수용 검증 완료를 근거 없이 표시하지 않았는지 확인.

## 기능 수용 검증

| 상태 | ID | 필수 시험 사례 |
| --- | --- | --- |
| 미실행 | FR-001 | 정상·인증/네트워크/부분 실패·정상 빈 입력·수집 원문 추적 |
| 미실행 | FR-002 | IPv4/IPv6·CIDR host bit·IDN·URL 경로/쿼리·중복·오류 정규화 |
| 미실행 | FR-003 | 수신 차단 Domain의 출처·부모별 IP 파생 관계; 보강 시각·제공자·없음/실패/만료 구분, 실패 시 원본 유지 |
| 미실행 | FR-004 | 기본 Profile·독립 복제·다중 소스/그룹·고정 입력의 동일 결과·Profile 격리 |
| 미실행 | FR-005 | 복수 출처·한 출처 해제·동일 해시의 다음 수집·같은 작업 재배달·재등장 |
| 미실행 | FR-006 | 정확/비일치·도메인 경계·IPv4/IPv6·ASN·CIDR 완전/부분 중첩·만료 경계·제외 사유 |
| 미실행 | FR-007 | 미완성 공개 차단·구세대 완료·병렬 게시·manifest 고정 다운로드·실패/stale/만료 구분·새 revision 생성과 제공 중단 |
| 미실행 | FR-008 | 파서별 미리보기·encoding·오류 건수·크기/CPU/시간 한도·버전 전환 |
| 미실행 | FR-009 | 정상 집합 누락·NXDOMAIN/NODATA·A/AAAA 부분 실패·CNAME·Resolver 불일치/전체 실패·독립 회차·TTL/유예/최대 수명·추가/제거/재등장·파생 기원; 특수 용도 응답 주소 거부·대량 주소 응답 상한·질의 불일치 응답 거부 |
| 미실행 | FR-010 | IPv4/IPv6 LPM·데이터셋 교체 실패·not_found·rate limit·배치 취소/재개 |
| 미실행 | FR-011 | 출처 수/IP 수 구분, 정제/실제 게시의 연속·누적 구간·중첩 방지 |
| 미실행 | FR-012 | /24 임계 직전/경계·해제·중복·기원, Allowlist 및 배포 제외 IP의 집계 재포함 방지 |
| 미실행 | FR-013 | 원본/경보 보존·범위별 예외·force_include 우회 불가·예외 변경 중 재시도·이전 버전 재현 |
| 미실행 | FR-014 | 최초/재발견/지속·경보 상태·dedup·억제·전달 실패/재시도 |
| 미실행 | FR-015 | Feed별 routing epoch·큰 CIDR·필수 segment·복제 조건·샤드 장애·fenced 재균형 |
| 미실행 | FR-016 | LDAP 부분 sync·비활성/이름 변경·SAML 서명/재생/시간/audience·권한 회수·비상 계정 |
| 미실행 | FR-017 | 접근 4개 모드·위조 proxy·직접 앱 접근·HEAD/304/캐시/이전 revision의 권한·제공 기한·회수 검사 |
| 미실행 | FR-018 | 차단/해제·기간 중복/역순·정상 해제 없음·인증 오류·손상/경로 탈출 ZIP |
| 미실행 | FR-019 | 형식별 타입·직접/파생 기원·manifest·changes·checksum·불변 revision |
| 미실행 | FR-020 | 설정 충돌·권한·작업 상태·취소/재시도·preview 비게시 |
| 미실행 | NFR-001 | 신규 Docker 설치·재시작·영속 데이터·내부 포트 비공개 |
| 미실행 | NFR-002 | SSRF·DNS rebinding·redirect·IPv4/IPv6·TLS·비밀/로그·최소 권한 |
| 미실행 | NFR-003 | 설정/지표/로그/artifact 백업·참조 무결성·권한·격리 복원·RPO/RTO |
| 미실행 | NFR-004 | MariaDB/PostgreSQL 공통 계약·Timescale 확장·DB별 SQL·단일 스키마 소유권 |
| 미실행 | NFR-005 | writer/leader 전환·락 만료·중복 배달·샤드 백업·directory 재구축 |
| 미실행 | NFR-006 | Docker/소스 동등성·서명/해시·호환 preflight·migration 부분 실패·코드/DB 복구 |
| 미실행 | NFR-007 | 지원 조합 CI·의존성/이미지 보안·주 버전 이관·목표 부하/장기 시험 |

FR-012 회귀 예시: Profile 전체 배포 제외 IP가 포함된 /24는 그대로 공개할 수 없습니다.
FR-005 회귀 예시: 하루 뒤 동일 원문 수집은 새 관측이며 같은 작업 재배달과 구분합니다.
FR-013 회귀 예시: 예외 집합 v2가 발행돼도 v1을 고정한 재생성의 결과는 바뀌지 않습니다.
시험 자료는 합성 또는 명시적으로 사용 승인된 자료만 사용합니다.

### 회귀 사례 ID 규칙

`ID` 열은 이 문서의 회귀 사례 각 행을 가리키는 고유 식별자이며 `<대표 FR/NFR-ID>-<2자리 순번>` 형식을 사용합니다.
대표 FR/NFR-ID는 `연결 ID` 열에 나열된 항목 중 첫 번째이며, 순번은 모든 회귀 사례 표를 통틀어 같은 대표 ID를 가진 행이 추가된 순서대로 01부터 매깁니다.
한번 부여한 ID는 행이 이후 재배치·삭제되어도 재사용하거나 앞당겨 채우지 않으며, 새 행은 해당 대표 ID의 마지막 순번 다음 번호를 append합니다.
날짜는 넣지 않습니다. 같은 사례를 여러 시점에 반복 실행할 때도 ID가 안정적이어야 하며, 실행 시점은 아래 실행 결과 양식의 `실행일`로 별도 기록합니다.

### DNS·Output 수명 회귀 사례

아래는 모두 **미실행 시험 계획**입니다. 수치가 있는 판정은 [D-03/D-05](workflow.md)의 미검증 설계 제안을 시험하기 위한 기대 결과이며,
사용자 확정값·운영 기본값·구현 테스트 통과를 뜻하지 않습니다. 확정 시 정책 버전과 경계값을 함께 갱신합니다.
DNS 상태 전이는 [수집·보강](pipeline.md), 제공 자격은 [Profile·API](profiles-api.md)를 기준으로 시험합니다.

| ID | 연결 ID | 입력·경계 | 기대 결과 |
| --- | --- | --- | --- |
| FR-003-01 | FR-003, FR-009 | 차단 Domain을 수신한 뒤 A/AAAA와 CNAME 종단 IP 조회 | 원본 Domain·Source 근거와 `derived_dns` 관계를 연결하고 직접 IP와 구분; 조회만으로 직접 악성 IP와 같은 신뢰 점수 부여 금지 |
| FR-009-01 | FR-009 | 정상 주소 집합이 `{ip_a, ip_b}`에서 `{ip_b, ip_c}`로 변경 | ip_c는 `added`, ip_b는 `unchanged`, ip_a는 `missing`; 제거 확정 후에만 `expired`와 `removed` 기록 |
| FR-009-02 | FR-009 | missing 또는 expired 관계의 IP가 정상 응답에 재등장 | 유효한 부모 근거 확인 후 `active` 복귀·누락 시각/횟수 초기화; expired 관계는 새 활성 구간과 `reactivated`를 남기고 이전 구간 보존 |
| FR-009-03 | FR-009 | A 정상/AAAA timeout, A의 NODATA, CNAME만 수신한 뒤 종단 조회 실패 | 실패한 주소 계열·미완성 경로를 정상 빈 집합으로 해석하지 않음; NODATA는 해당 질의 타입에만 적용, CNAME 종단 NXDOMAIN은 확인된 경로만 평가 |
| FR-009-04 | FR-009 | Resolver 불일치·전체 실패, 같은 회차 재배달/재시도·유효 캐시의 즉시 반복 사용 | 실패/불일치로 제거를 확정하거나 last_seen을 갱신하지 않음; 연속 확인 중단, 중복·캐시 반복은 독립 확인 횟수에 추가하지 않음 |
| FR-009-05 | FR-009 | 정상 누락 2회지만 1시간 미만, 1시간 경과했지만 1회, 두 조건 충족 | 앞의 두 경우 제거하지 않음; 독립 조회 2회 연속 및 1시간 경과를 모두 충족하고 최신 완료 회차도 정상 누락이면 제거 |
| FR-009-06 | FR-009 | 동일 범위의 NXDOMAIN 또는 NODATA가 3회/24시간의 한 조건만 충족하거나 사유 변경·실패가 끼어듦 | 횟수와 유예를 모두 충족해야 제거; 사유 변경·실패 후 연속성을 다시 확인하며 정상 누락 카운터와 혼합하지 않음 |
| FR-009-07 | FR-009, FR-007 | 마지막 정상 양성 응답 이후 24시간·7일 직전/정각/직후, 그동안 DNS 실패·출력 재생성 반복 | 제안값 기준 24시간부터 stale, 7일부터 기여 만료; 실패·재생성·재게시로 last_seen이나 최대 수명을 연장하지 않음 |
| FR-003-02 | FR-003, FR-005 | 부모 Domain의 한 Source 차단 근거 해제/만료 후 DNS 조회 성공; 같은 IP에 다른 Domain/Source·직접 근거 존재 | 종료한 근거의 파생 기여만 제외하고 DNS 성공으로 위협 근거를 부활시키지 않음; 다른 유효 기여와 원본 관측 유지 |
| FR-007-01 | FR-007, FR-012 | 선택 Source 하나가 만료되고 다른 Source는 유효; 반대로 필수 Source 사용 불가 또는 모든 선택 Source 만료 | 전자는 제외 사유·남은 기여·/24 집계를 다시 평가한 새 불변 revision 생성; 후자는 unavailable이며 정상 빈 Feed로 게시하지 않음; 필수 샤드 누락은 계속 게시 거부 |
| FR-007-02 | FR-007, FR-009, FR-017 | stale 또는 missing 기여를 허용하지 않는 Profile에서 stale 시작 경계 또는 첫 missing 전이 도달 | stale 시작 시각을 serve_until에 포함하고, 비허용 missing 전이에는 제거 확정을 기다리지 않고 종속 revision 제공 자격 회수·재생성; 최대 수명 이전이어도 기존본·캐시로 우회 제공하지 않음 |
| FR-005-01 | FR-005, FR-007 | 일반 Source 동일 해시의 다음 정상 수집, 과거 기간 자료 재다운로드, DNS만 갱신, 동일 입력 artifact 재생성 | 실제 Source 성공과 DNS 양성 관측은 각자의 기준만 갱신; 과거 기간 기준이나 부모 근거 만료·고정 manifest의 serve_until을 다른 동작으로 연장하지 않음 |
| FR-007-03 | FR-007, FR-017, FR-019 | serve_until 직전/정각/직후 및 조기 회수 시 현재/이전 revision·manifest·changes·checksum·HEAD·ETag 조건부 요청·캐시 적중 | 기한 내에만 제공하고 만료/회수 후 인증·권한 확인을 거쳐 503, Feed 본문·304 제공 금지; changes의 두 revision도 검사하고 이전 revision 보존 7일로 우회하지 않음 |
| FR-007-04 | FR-007, FR-011 | 기여 만료 뒤 재게시 실패, 새 revision 없이 기존본 제공 중단, 이후 정상 재수신·게시 | 실제 게시 구간은 제공 중단 시 종료하고 재게시 시 새 구간 기록; 정제 제외·게시 기간·DNS 관측·물리 보존을 구분하며 생성만 성공한 기간은 배포 기간에 포함하지 않음 |
| FR-007-05 | FR-007, FR-019 | 제공 중단 503 또는 유효한 입력을 완전히 평가한 정상 빈 결과 200을 실제 소비자가 수신 | 두 응답의 소비자 동작과 재수신을 확인; 서버 제공 중단만으로 이미 받은 차단 목록이 자동 해제됐다고 판정하지 않음 |
| FR-003-03 | FR-003, FR-009 | 악성 Domain이 루프백·사설망·링크 로컬·클라우드 메타데이터 주소로 응답 | `derived_dns` 후보로 채택하지 않고 사유를 남기며, 원본 Domain의 다른 유효 근거는 그대로 유지 |
| FR-009-08 | FR-009 | 같은 초기 상태에서 33개 이상의 고유 A 또는 AAAA 주소를 서로 다른 순서로 수신 | 해당 타입을 incomplete/address_limit_exceeded로 표시; 중복 제거·고정 정렬에 따른 신규 채택은 계열별 최대 32개이며 순서에 따라 바뀌지 않음; 부분집합을 누락 비교용 완료 집합으로 사용하지 않음 |
| FR-009-09 | FR-009 | 기존 ip_a를 포함한 33개 이상의 동일 응답 집합에서 수신된 응답 순서상 ip_a가 회차마다 32번째 안팎에 나타남 | 실제 양성 확인된 ip_a의 last_seen은 신규 채택 상한과 무관하게 갱신; 순서 변경만으로 missing 확인 누적·expired/removed 전이 또는 정제/배포 기여 소멸이 발생하지 않음 |
| FR-009-10 | FR-009 | 정상 누락 또는 동일 음성 응답 뒤 상한 초과 회차를 거쳐 다시 정상 누락/음성 확인; 불완전 회차 중 유예 시각 도달 | 불완전 회차에서 확인 횟수 증가나 제거 확정 금지, 연속성 중단; 뒤의 완료 회차는 새 연속 구간의 첫 확인이며 앞뒤 회차를 합산하지 않음 |
| FR-009-11 | FR-009, FR-007 | 상한 초과 회차를 반복하지만 기존 ip_b는 실제 양성 확인되지 않음; A만 상한 초과이고 AAAA는 완료, 또는 필요한 Resolver 하나만 상한 초과 | ip_b의 last_seen을 일괄 갱신하지 않고 최대 수명·부모 해제는 그대로 적용; 불완전한 타입/합성 결과로 누락 판정하지 않으며 독립적으로 완료된 다른 타입은 기존 규칙 적용 |
| FR-009-12 | FR-009 | 질의(트랜잭션 ID·질문 section·Resolver) 불일치 또는 응답 바이트·파싱 제한으로 안전한 검증 불가 | 신규 주소 채택과 기존 관계의 양성 갱신을 하지 않음; 상한 부분 채택 규칙으로 응답 검증·자원 제한을 우회하지 않음 |

## 외부 Feed 구현 참고와 사전 점검

사용자 요청은 외부 구현을 개발 요구조건과 사전 이슈 점검의 참고 근거로 기록하는 것이다.
검토 기준은 2026-09-10에 확인한 [ziyadnz/threat-intel-ip-feeds의 d2131c1 커밋](https://github.com/ziyadnz/threat-intel-ip-feeds/tree/d2131c1716af9b39b57030b50e555b690b561140)이다.
아래 외부 동작은 해당 커밋의 코드 확인 근거이며 현재 이후 버전까지의 보증이 아니다. 코드 도입·Feed 선정·라이선스 결정 또는 TFR 기능 완료를 뜻하지 않는다.

| 외부 구현에서 확인한 사례·근거 | TFR 연결 ID | 요구조건에 반영할 사전 점검·담당 문서 |
| --- | --- | --- |
| 일반 소스의 빈 결과에는 캐시 대체가 없고, OTX는 일부 페이지 실패 후에도 비어 있지 않은 결과로 캐시를 교체한다([수집 처리][ref-collect], [API 어댑터][ref-api]) | FR-001, D-01 | 기존 불완전본 교체 금지 유지. 페이지·cursor·조회 상한·정상 변경 없음의 구분을 [어댑터 계약](pipeline.md#완전성과-캐시-재사용)에 구체화 |
| 지표 생성마다 first_seen과 last_seen에 같은 실행 시각을 넣는다([지표 모델][ref-entities]) | FR-005, FR-011, D-03 | 기존 관측·등록 기간 요구 유지. 최초 시각 보존과 재등장 구간을 [시간 계약](architecture.md#시간과-유효기간)에 구체화 |
| 캐시 timestamp를 읽어도 만료를 검사하지 않고, API 어댑터의 캐시 반환은 일반 수집 결과처럼 전달된다([일반 캐시][ref-cache], [API 어댑터][ref-api]) | FR-001, FR-005, FR-007, D-03, D-05 | 기존 유한 수명 유지. 캐시 재사용과 신규 검증 성공의 전달·표시를 [파이프라인](pipeline.md#완전성과-캐시-재사용)·[운영 상태](security-operations.md#운영-상태-모델)에 구체화 |
| 주소 객체는 입력 문자열을 유지하고, 기본 텍스트 파서는 IPv4 부분 문자열만 추출한다([지표 모델][ref-entities], [기본 파서][ref-parser]) | FR-002, FR-008, D-02 | 기존 정규화·다형식 파서 요구 유지. 동치 IPv6·CIDR prefix·host bit·명시 추출 범위를 [파서 계약](pipeline.md#파서-설정과-미리보기)과 회귀 시험으로 구체화 |
| Allowlist가 후보 CIDR 전체를 덮지 못하면 해당 CIDR은 남는다([매칭 구현][ref-entities]) | FR-006, FR-012, FR-013, D-04 | [최종 제외·부분 겹침 계약](profiles-api.md#결정적-평가-순서-제안)은 이미 명시됨. 정책을 중복 정의하지 않고 개별 IP 예외를 포함한 원본 CIDR의 회귀 시험 추가 |
| writer 예외를 기록해도 출력 use case는 성공을 반환한다([출력 처리][ref-write]) | FR-007, FR-019, NFR-003, D-05 | 기존 불변·다중 파일 일관성 요구 유지. 필수 파일 실패와 최종 작업 실패 전파를 [게시 단계](pipeline.md#생성게시와-장애-복구)·[운영 상태](security-operations.md#운영-상태-모델)에 구체화 |
| STIX Indicator 출력에서 created·modified가 빠져 있다([STIX writer][ref-stix]) | FR-008 | 후속 STIX 입력 파서가 잘못된 외부 객체를 수신할 사례로 반영. [공식 스키마 검증](pipeline.md#파서-설정과-미리보기)을 요구하며 STIX 출력 기능을 새로 추가하지 않음 |
| 신뢰 점수는 Source 수에 비례하며 같은 제공자의 하위 목록도 별도 Source다([점수 모델][ref-entities], [등록 목록][ref-cli]) | FR-004, FR-011, FR-012, D-01, D-04 | Source 개수와 독립 증거 수를 [집계 의미](architecture.md#등록-기간과-집계)에서 구분. 출처 계보·중복 가중 정책의 채택과 산식은 미정이며 외부 산식을 기본값으로 가져오지 않음 |
| README의 소스 수·비동기 설명과 현재 등록 목록·스레드 구현이 다르고, 수집 단계 오류 이후 workflow가 계속 실행된다([README][ref-readme], [등록 목록][ref-cli], [workflow][ref-workflow]) | NFR-003, NFR-007, D-07, D-12 | [운영 상태](security-operations.md#운영-상태-모델)와 [유지보수 점검](security-operations.md#장기-유지보수)에 실패 전파·문서 대조 추가. 자동 갱신·CI 성공을 전체 기능 검증이나 모든 소스의 최신성으로 해석하지 않음 |

### TFR 사전 회귀 사례

아래는 모두 **미실행 시험 계획**이다. 앞선 외부 코드의 합성 동작 확인을 TFR의 통과 결과로 옮기지 않는다.
기존 요구사항·설계 계약을 시험으로 구체화한 것이며 새 수치·지원 범위의 확정은 아니다.
문서용 주소 예시는 매칭 단위의 합성 fixture이며 실제 수집 단계의 특수 용도 주소 거부 규칙과 구분한다.

| ID | 연결 ID | 입력·경계 | 기대 결과 |
| --- | --- | --- | --- |
| FR-001-01 | FR-001 | 다중 페이지 전체 목록에서 첫 페이지 성공 후 나머지 실패, 반복 cursor 또는 조회 상한 도달 | 불완전 사유·범위를 기록하고 활성 스냅샷·최근 정상 캐시 교체 금지; 전체 Source 성공 비율로 승인 우회 불가 |
| FR-001-02 | FR-001, FR-018 | 기존 정상 전체 목록 뒤 HTTP 200 빈 응답·로그인 HTML·전량 파싱 실패; 별도로 검증된 증분 변경 없음·해제 없음 | 전자는 기존 기여의 전량 해제로 해석하지 않고 D-01에 따라 거부; 후자는 어댑터의 증분 계약에 따라 기존 근거 유지 |
| FR-005-02 | FR-005, FR-011 | 같은 Source-지표의 다음 정상 수집, 같은 작업 재배달, 출력 재생성, 제거 후 재등장 | first_seen 보존; last_seen은 실제 새 정상 관측만 반영; 중복 실행으로 관측 수 증가 금지, 재등장은 별도 활성 구간 |
| FR-005-03 | FR-005, FR-007 | 원래 수집 시각을 고정한 캐시를 일반/API 어댑터에서 실패·예약상 건너뜀 때 재사용하며 stale/expire 경계 통과 | 캐시 기원·현재 시도 결과 표시; 최근 성공·last_seen·freshness·만료 기한 연장 금지; 기존 Profile 수명 정책대로 제외·재생성·제공 중단 |
| FR-002-01 | FR-002 | 동일 IPv6의 축약·전체 표기, CIDR host bit 입력, 동일 값의 여러 Source 관측 | 같은 정규화 버전의 동치 주소는 지표 하나와 복수 관측으로 연결; host bit 입력은 D-02 제안에 따라 거부하고 CIDR을 단일 IP로 축소하지 않음 |
| FR-008-01 | FR-008, FR-002 | IPv4·IPv6·CIDR·주석을 섞은 행 입력과 명시 token/regex 설정 | 선언한 타입·prefix 보존, 지원하지 않는 항목과 추출 오류를 미리보기에 표시; IPv4 부분 문자열만 찾아 성공한 것으로 숨기지 않음 |
| FR-006-01 | FR-006, FR-012, FR-013 | 원본 Feed의 198.51.100.0/24 안에 Allowlist 또는 Profile 전체 배포 제외인 198.51.100.10 존재 | /24 승격 없이 받은 원본 CIDR에도 최종 제외 적용; 허용된 분할 한도에서 제외하거나 게시 거부, 해당 IP를 포함한 /24 그대로 게시 금지 |
| FR-007-06 | FR-007, FR-019, NFR-003 | 고정 출력 계약의 필수 TXT 저장 후 CSV 저장 오류·checksum 불일치, 이후 상태 보고는 성공 | generation 게시 실패·기존 공개 포인터 유지; 새 불완전 manifest 제공 금지; CLI/scheduler/CI가 보고 단계 성공으로 실패를 덮어쓰지 않음 |
| FR-008-02 | FR-008 | 후속 STIX 파서에 문법상 유효하지만 created/modified 누락·잘못된 시각·pattern·참조인 객체 입력 | 명시 STIX 버전의 스키마·의미 검증에서 거부 또는 격리하고 검증 버전·사유 기록; JSON 파싱 성공만으로 활성화하지 않음 |
| FR-011-01 | FR-011, FR-004, FR-012 | 같은 제공자의 전체/하위 목록과 재집계 Feed가 같은 지표를 보고 | 등록 Source 수·고유 IP 수를 구분하고 독립 증거 수·악성 확률로 오표기하지 않음; 가중 선택 조건을 채택한다면 D-01/D-04 확정 정책을 별도 검증 |
| NFR-007-01 | NFR-007, NFR-003 | 어댑터 등록·출력 형식·실행 방식 변경 후 이전 문서 유지, 기능 시험 없이 데이터 갱신 작업만 성공 | 문서와 등록·설정·계약 시험의 불일치 검출; 미실행 시험을 통과로 표시하지 않고 수집·게시·시험의 실제 상태를 구분 |

[ref-collect]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/application/use_cases/collect_threat_intel.py
[ref-api]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/infrastructure/sources/api_sources.py
[ref-entities]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/domain/entities.py
[ref-cache]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/infrastructure/cache/source_cache.py
[ref-parser]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/infrastructure/sources/base.py
[ref-write]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/application/use_cases/write_outputs.py
[ref-stix]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/infrastructure/writers/stix_writer.py
[ref-cli]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/threat_intel/presentation/cli.py
[ref-readme]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/README.md
[ref-workflow]: https://github.com/ziyadnz/threat-intel-ip-feeds/blob/d2131c1716af9b39b57030b50e555b690b561140/.github/workflows/update.yml

## 실행 결과 양식

검증 ID·대상 커밋 / 실행일·실행자·환경 / 입력·기대 결과 / 실제 결과·판정 /
실행 명령·증거 위치 / 결함·미실행 이유를 기록합니다.
검증 ID는 회귀 사례 표의 행이면 해당 `ID` 값을, 그 외에는 `기능 수용 검증`의 FR/NFR ID를 사용합니다.
기능 체크는 실제 증거가 있는 항목만 완료로 변경합니다.

# Profile·배포 API 설계

연결: FR-004, FR-006, FR-007, FR-012, FR-013, FR-019, FR-020.
상세 매칭·API 경로·정책은 설계 제안이며 남은 선택은 [D-04/D-05](workflow.md)에 기록합니다.
권한·프록시·인증은 [보안·운영](security-operations.md)의 계약을 따릅니다.

## 역할

Profile은 어떤 출처·지표 타입·선택 조건으로 어떤 출력을 만들지 선언한다.
Allowlist는 Profile의 후보에서 명시적으로 제외할 지표나 범위를 선언한다.
Allowlist에 해당한다는 것은 안전성 판정이 아니라 해당 출력에서 제외하는 운영 정책을 뜻한다.
출처 관측 이력은 제외 여부와 관계없이 [데이터 모델](architecture.md)에 따라 유지한다.

## Profile 필드 제안

| 필드 | 의미 |
| --- | --- |
| `profile_id`, `version` | 안정적인 식별자와 불변 버전 |
| `name`, `description` | 사용 목적과 대상 소비자 |
| `source_ids` | 포함할 소스를 명시한 목록 |
| `indicator_types` | IP/CIDR/Domain/URL 중 허용한 타입 |
| `selection_rules` | 출처 수 등 지원이 확정된 선택 조건 |
| `enrichment_policy` | 선택 조건에 필요한 보강 항목과 만료 처리 |
| `allowlist_refs` | 적용할 Allowlist ID와 정확한 버전 |
| `output_contract_ref` | 형식·정렬·분할·엔드포인트 계약 버전 |
| `enabled` | 예약 생성·게시 대상 여부 |

필드 기본값과 필수 여부는 D-04, 출력 형식·소비자별 제약은 D-05에서 결정한다.
지원하지 않는 필터나 지표 타입은 무시하지 않고 설정 검증에서 실패시킨다.
명시적인 빈 소스 목록을 전체 소스 선택으로 해석하지 않는 방안을 제안한다.

## Allowlist 항목 제안

각 항목은 `entry_id`, `type`, `value`, `match_mode`, `reason`, `created_at`을 갖는다.
그룹에는 이름·설명·소유자·서비스·태그를 두고 항목에는 담당자·작성자·승인자·근거·활성 여부·유효기간을 기록한다.
시작 시각은 포함하고 만료 시각은 제외하는 UTC 구간을 제안하며 D-04에서 확정한다.
평가 기준 시각은 생성 입력 manifest에 고정하여 경계 시각에서도 재현 가능하게 한다.

| 타입 | 매칭 제안 | 자동 적용하지 않을 동작 |
| --- | --- | --- |
| IP | 같은 주소 계열의 정규화 값 일치 | DNS 이름을 조회하여 IP 예외로 변환 |
| CIDR | 명시한 CIDR 안의 IP, 완전히 포함된 후보 CIDR 제외 | 부분 겹침만으로 큰 CIDR 전체 제외 |
| Domain | 기본은 정확 일치; 명시 모드에서 하위 도메인 포함 | 단순 문자열 suffix로 유사 이름 제외 |
| URL | 동일 정규화 버전의 URL 정확 일치 | 임의 wildcard·정규식·URL 방문 결과 매칭 |
| wildcard_domain | 명시된 점 경계·apex 포함 정책 | 공용 suffix 전체를 의도 없이 허용 |
| ASN | 고정 보강 버전의 ASN과 명시 규칙 일치 | 조회만으로 전 대역 자동 제외 |

Domain의 하위 도메인 모드는 점 경계와 apex 포함 여부를 명시해야 한다.
예를 들어 `example.com` 하위 도메인 매칭이 `notexample.com`에 적용되어서는 안 된다.
Domain Allowlist가 URL host에도 적용되는지는 별도 교차 타입 규칙이며 초기에는 암묵적으로 적용하지 않는다.
IP 예외가 큰 CIDR과 일부만 겹치면 해당 CIDR이 IP를 계속 포함할 수 있으므로 경고·게시 거부 정책을 둔다.
부분 겹침을 해결하는 CIDR 분할은 출력 증가 한도와 소비자 요구를 검토한 후 D-04, D-05에서 결정한다.
정규화 규칙이 달라지는 경우 Allowlist도 검증·재발행하며 구버전을 조용히 재해석하지 않는다.

## 결정적 평가 순서 제안

1. Profile, Allowlist, 입력 스냅샷, 기준 시각과 관련 정책 버전을 고정한다.
2. 선택한 출처에서 유효한 지표를 가져와 타입·선택·보강 조건을 평가한다.
3. 허용된 파생 처리와 CIDR 집계 등 출력 변환이 있다면 먼저 수행한다.
4. 최종 출력 의미를 기준으로 Allowlist와 필수 배포 제외 집합을 함께 검사한다.
   배포 제외 집합에는 해당 Profile에 적용되는 `distribution_exclude`, `monitor_only`, 미해제 `quarantine`을 포함한다.
   Source 한정 규칙은 해당 기원의 기여만 제외하며, 다른 유효 출처까지 제외할지는 규칙 범위에 명시한다.
5. 부분 CIDR 겹침 같은 미해결 충돌이 있으면 해당 산출물 게시를 중단한다.
6. 타입별 정렬·중복 제거·형식 검증 후 [파이프라인](pipeline.md)의 게시 단계로 넘긴다.

최종 제외 뒤에 범위를 넓히는 집계·변환을 수행하여 예외 지표가 다시 포함되지 않게 한다.
여러 Allowlist와 Profile 전체 배포 제외는 합집합으로 적용한다. `force_include`는 이 최종 제외 집합을 덮어쓰지 않는다.
알 수 없는 필수 조건·정책 오류는 이전 정상 산출물을 유지하는 생성 실패로 처리한다.

## Profile 수명과 선택

기본 Profile에서 생성하거나 사용자 Profile을 복제하고 미리보기·활성화·비활성화·삭제·버전 복원을 제공한다.
템플릿과 복제본은 별도 ID로 관리하여 원본 수정이 복제본에 자동 전파되지 않는다.
소스·Allowlist 그룹은 여러 개 선택하고 DNS Resolver/ASN 정책·지표 기원·출력 형식을 버전에 포함한다.
기본 Profile의 정책 값은 D-04에서 확정하고 공격적 차단 기본값을 임의로 넣지 않는다.
삭제·비활성화의 예약 중단, 공개본 회수, 이력 보존 효과를 구분한다.

Domain의 URL host 적용, 신뢰 도메인의 DNS 파생 IP 제외는 명시 교차 타입 정책으로만 허용한다.
공유 IP 전체를 근거 없이 제외하지 않고 부모 도메인·파생 경로를 검사한다.
직접 수집 IP와 DNS 파생 IP의 동일 주소가 병존하면 각각의 출처·제외 근거를 유지한다.

## 수집 예외

IP/CIDR, 적용 Source/Profile, 유효기간, 이유·담당자·우선순위를 불변 예외 규칙 집합 버전으로 관리한다.
발행 버전은 수정하지 않고 새 버전을 만든다. 생성·재시도는 입력 manifest의 예외 버전만 사용한다.
원문·관측 저장과 모니터링 평가 이후 정제·배포 효과를 적용한다.

| 동작 후보 | 의미 |
| --- | --- |
| `exclude` | 지정 범위의 정제 후보에서 제외 |
| `distribution_exclude` | 정제 관측 유지, 배포 제외 |
| `monitor_only` | 관측·경보만 유지, 배포 제외 |
| `quarantine` | 검토 대기로 격리 |
| `score_ignore` | 선택된 점수·집계 기여만 제외 |
| `tag_only` | 태그 추가, 배포 여부는 바꾸지 않음 |
| `force_include` | 정제 후보 포함 요청; 최종 배포 제외 집합·접근 통제 우회 불가 |

조합·충돌 우선순위는 D-04에서 확정한다. 미정 충돌을 임의 순서로 처리해 게시하지 않는다.

## IPv4 /24 승격

같은 출력 Feed·같은 /24·관측 기간의 활성 고유 IP 수 또는 비율로 승격을 판단한다.
Source 수와 IP 수를 별도 집계하고 동일 Source 내 조건과 선택 Source 합집합 조건을 구분한다.
승격·해제 임계치·최소 지속 시간·관측 창을 설정하고 hysteresis로 반복 전환을 줄인다.
대화의 예시 숫자는 운영 기본값으로 확정하지 않는다.

집계할 직접/DNS 파생 기원을 명시하며 IPv6에 /24를 적용하거나 ASN prefix로 자동 승격하지 않는다.
확대 범위·원래 IP·근거·정책을 이력으로 보존한다.
집계 후 모든 필수 배포 제외를 검사한다. 부분 중첩은 제한된 CIDR 분할로 제외를 보장하거나 게시를 거부한다.
예: Profile 전체에서 `198.51.100.42`를 배포 제외했다면, 다른 IP가 임계치를 충족해도
`198.51.100.0/24`를 그대로 게시하지 않는다. 제외 IP를 다시 포함하는지 최종 출력 의미로 검증한다.
큰 대역·주소 경계·임계치·IP 중복·해제·재등장·공유 IP를 검증한다.

## API 계약

### 공통 계약 제안

- 예시 경로의 `/api/v1`과 `/feeds/v1`은 제안이며 구현 전에 확정한다.
- 관리 API는 JSON 요청·응답을 제안하고 외부 Feed의 형식과 분리한다.
- 관리 리소스에는 불변 ID, 구성 revision, 생성·수정 시각을 둔다.
- 시각은 시간대가 명시된 UTC 형식을 사용하고 표시 형식을 계약에 명시한다.
- 요청 ID를 응답과 로그에 연결하되 오류 메시지에 비밀값·내부 주소를 포함하지 않는다.
- 인증과 권한 검사는 목록, 단건 조회, 수정 및 출력 요청 모두에 적용한다.
- 충돌 감지를 위해 수정 요청에 기존 구성 revision을 보내는 방식을 제안한다.
- 목록 응답의 페이지 크기와 정렬 기준, 요청 상한은 D-05와 D-07에서 확정한다.

### Output Feed API 제안

| 메서드·경로 예시 | 의미 | 계약 후보 |
| --- | --- | --- |
| `GET /feeds/v1/{profile_id}/{indicator_type}` | 현재 게시된 Feed | 한 응답은 하나의 불변 revision |
| `HEAD /feeds/v1/{profile_id}/{indicator_type}` | Feed 메타데이터 | GET과 같은 상태·revision, 본문 없음 |
| `GET /api/v1/profiles/{profile_id}/status` | 게시·입력 상태 | 관리 권한으로 freshness 및 최근 실패 조회 |
| `GET /feeds/v1/{profile_id}/manifest` | 다중 파일 출력의 일관된 목록 | 한 revision과 타입별 불변 경로·체크섬 |
| `GET /feeds/v1/{profile_id}/revisions/{revision}/{indicator_type}` | 특정 revision의 파일 | 같은 revision의 내용은 변경하지 않음 |

- 출력은 IPv4/IPv6/CIDR/Domain/URL과 직접 IP·DNS 파생 IP·혼합을 구분한다. 정확한 경로 이름은 D-05에서 확정한다.
- TXT/CSV/JSON/hosts를 지원하며 format 선택·표현별 ETag·Content-Type을 명시한다. hosts에 URL/CIDR을 넣지 않는다.
- manifest는 표현별 불변 경로·checksum·건수·정책·입력·라우팅 버전을 제공한다. changes는 비교할 기준/대상 revision을 명시한다.
- 텍스트 출력은 한 줄에 한 항목을 제안하며 UTF-8, 줄바꿈, 정렬 및 끝 개행을 명시한다.
- IP·CIDR·Domain·URL 정규화와 중복 제거 규칙은 D-02 결정에 맞춘다.
- Profile의 Source 선택·Allowlist 결과를 반영한 게시 완료 artifact만 제공한다.
- 요청 시작 시 공개 revision을 고정하여 응답 도중 새 파일 내용이 섞이지 않도록 한다(FR-007).
- 다중 파일 지원 시 소비자는 manifest를 한 번 읽고 불변 경로를 사용한다. 타입별 현재 경로를 여러 번 읽는 것은 동일 revision을 보장하지 않는다.
- 불변 경로에도 인증·권한 검사를 적용한다. 이전 revision의 제공 기한·회수·stale 정책과 파일 보관 기간을 D-03·D-05에서 함께 정한다.
- ETag를 사용할 경우 실제 응답 표현의 revision과 연결하고 조건부 GET을 지원하는 방식을 제안한다.
- 게시 시각과 마지막 입력 성공 시각은 별도 필드 또는 헤더로 구분해 제공한다.
- 헤더 후보는 `X-Feed-Revision`, `X-Feed-Published-At`, `X-Feed-Stale`이며 이름은 D-05에서 정한다.
- 성공 본문에 오류 설명이나 관리 메타데이터를 항목처럼 섞지 않는다.
- Profile 간 또는 권한 간 캐시가 섞이지 않도록 캐시 키와 공유 캐시 허용 여부를 명시한다.
- 인증·권한 확인은 조건부 요청의 `304` 응답에도 적용한다.
- 첫 게시 전, 유효한 빈 결과, stale 초과, 삭제된 Profile을 서로 구분하는 정책을 확정한다.

### 관리 API 제안

| 메서드·경로 예시 | 목적 | 주요 요청 또는 응답 |
| --- | --- | --- |
| `GET, POST /api/v1/sources` | Source 조회·등록 | 이름, URL, 파서 유형, 활성 여부, 비밀 참조 |
| `GET, PATCH /api/v1/sources/{id}` | Source 조회·변경 | 구성 revision과 변경 필드 |
| `POST /api/v1/sources/{id}/runs` | 수동 수집 요청 | 접수한 작업 ID와 조회 경로 |
| `GET /api/v1/runs/{id}` | 작업 결과 조회 | 공통 작업 상태, 진행률·오류 분류 |
| `GET, POST /api/v1/profiles` | Profile 조회·등록 | 이름, Source 선택, 출력 옵션 |
| `GET, PATCH /api/v1/profiles/{id}` | Profile 조회·변경 | 구성 revision, 정책 참조 |
| `GET, PATCH /api/v1/profiles/{id}/allowlist-groups` | 적용 그룹 조회·변경 | 그룹 ID·버전 참조, 기존 구성 revision |
| `POST /api/v1/profiles/{id}/publications` | 출력 재생성 요청 | 접수한 작업 ID와 대상 구성 revision |

- Allowlist·예외 규칙 집합과 Profile 참조는 새 버전으로 변경한다. 공유 집합의 변경을 기존 Profile·생성 작업에 암묵적으로 전파하지 않는다.
- Source URL 변경은 [보안·운영](security-operations.md)의 수집 경계 검증을 거친다.
- 비밀값은 조회 응답에서 반환하지 않으며 설정 여부와 참조만 노출한다.
- 비동기 작업 접수는 완료로 간주하지 않고 별도 조회에서 성공·실패를 확인한다.
- 중복 실행의 거부·병합·대기 방식과 재시도 키 계약을 D-05에서 확정한다.
- 구성 변경 응답과 실제 출력 게시를 분리하고 적용된 구성 revision을 상태 API에 표시한다.
- Source·Profile 삭제와 비활성화의 데이터 보존 효과는 D-03 결정 후 계약에 추가한다.

### 상태 코드 및 오류 후보

- `200`: 조회 성공 또는 확정된 정책상 허용된 정상·stale Feed 응답.
- `201`, `202`: 리소스 생성, 비동기 작업 접수. 작업 접수 응답에는 추적 정보를 포함한다.
- `304`: 인증·권한 확인을 통과한 조건부 요청에서 표현이 변경되지 않음.
- `400`, `401`, `403`, `404`: 입력 오류, 인증 실패, 권한 부족, 리소스 없음의 계약 후보.
- `409`: 구성 revision 충돌 또는 계약상 동시에 수행할 수 없는 작업.
- `429`, `503`: 요청 제한, 제공 가능한 스냅샷 부재 또는 확정된 stale 차단 정책.
- 관리 오류 본문은 `code`, `message`, `request_id`, 선택적 필드 오류를 제안한다.
- 최초 게시·빈 결과·stale 응답 코드는 D-05 확정 전에는 클라이언트 호환 계약으로 취급하지 않는다.

### 추가 관리 계약

다음은 경로 후보이며 구현 전 OpenAPI/JSON Schema로 확정한다.

| 리소스·동작 | 계약 |
| --- | --- |
| `/api/v1/parsers/preview` | 제한된 샘플·파서 revision, 추출·오류·중복; 외부 URL이면 수집 보안 검사 |
| `/api/v1/resolvers`, `/api/v1/dns/jobs` | Resolver 설정·비교·예약/수동 조회·변화 이력 |
| `/api/v1/asn/datasets`, `/api/v1/asn/jobs` | 데이터셋 검증·활성화, 개별/일괄 조회 |
| `/api/v1/allowlist-groups` | 재사용 그룹·항목 버전 관리, Profile은 그룹 버전 참조 |
| `/api/v1/profiles/{id}/clone`, `/api/v1/profiles/{id}/preview` | 복제, 이전 게시 대비 추가/제외/충돌 비교 |
| `/api/v1/exception-rules`, `/api/v1/monitor-rules`, `/api/v1/alerts` | 예외·모니터링 독립 관리, 경보 확인·해제·억제 |
| `/api/v1/indicators/{id}/history` | 출처·DNS·정제·게시 등록 구간, 조회 기간·페이지 크기 제한 |
| `/api/v1/identity-providers`, `/api/v1/roles` | 비밀 마스킹·고권한 검증·매핑 시험·감사 |
| `/api/v1/shards`, `/api/v1/backups`, `/api/v1/updates` | 별도 운영 권한·재인증·비동기 상태·충돌 제어 |

작업 상태는 [아키텍처](architecture.md)의 공통 enum을 사용한다.
취소·재시도·resume 가능 여부는 작업 종류·checkpoint로 응답한다.
파서/Profile 미리보기는 실제 게시를 수행하지 않으며 민감한 변경은 GET으로 실행하지 않는다.

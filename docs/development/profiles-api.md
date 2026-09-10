# Profile·배포 API 설계

연결: FR-004, FR-006, FR-007, FR-012, FR-013, FR-017, FR-019, FR-020.
상세 매칭·API 경로·정책은 설계 제안이며 남은 선택은 [D-04/D-05](workflow.md)에 기록합니다.
권한·프록시·인증은 [보안·운영](security-operations.md)의 계약을 따릅니다.

## 역할

프로파일은 수집과 배포로 나눈다. 수집 프로파일은 어떤 Source를 어떤 정책으로 수집·보강할지 선언하고,
배포 프로파일은 그 결과에 Allowlist와 예외를 계산 반영해 어떤 출력을 만들지 선언한다.
클라이언트는 배포 프로파일의 출력을 가져가는 주체이며 이 문서의 클라이언트 관리 절에서 다룬다.
Allowlist는 배포 후보에서 명시적으로 제외할 지표나 범위를 선언한다.
Allowlist에 해당한다는 것은 안전성 판정이 아니라 해당 출력에서 제외하는 운영 정책을 뜻한다.
출처 관측 이력은 제외 여부와 관계없이 [데이터 모델](architecture.md)에 따라 유지한다.

하나의 Source를 여러 수집 프로파일이 참조할 수 있고, 하나의 배포 프로파일이 여러 수집 프로파일을 입력으로 삼을 수 있다.
같은 수집 결과를 서로 다른 Allowlist·출력 계약으로 배포하려고 수집을 중복 실행하지 않는다.

## 수집 프로파일 필드 제안

| 필드 | 의미 |
| --- | --- |
| `collection_profile_id`, `version` | 안정적인 식별자와 불변 버전 |
| `name`, `description` | 수집 목적 |
| `source_ids` | 포함할 Source를 명시한 목록 |
| `schedule_override` | 소스별 기본 주기를 덮어쓸 때의 수집 주기·재시도 |
| `parser_policy_ref`, `normalizer_policy_ref` | 적용할 파서·정규화 정책 버전 |
| `enrichment_policy` | DNS·ASN 조회 수행 여부와 Resolver 정책 연결, 보강 결과 만료 처리 |
| `enabled` | 예약 수집 대상 여부 |

## 배포 프로파일 필드 제안

| 필드 | 의미 |
| --- | --- |
| `distribution_profile_id`, `version` | 안정적인 식별자와 불변 버전 |
| `name`, `description` | 사용 목적과 대상 소비자 |
| `collection_profile_refs` | 입력으로 사용할 수집 프로파일 ID와 정확한 버전 |
| `indicator_types` | IP/CIDR/Domain/URL 중 허용한 타입 |
| `selection_rules` | 출처 수 등 지원이 확정된 선택 조건 |
| `derived_output_policy` | DNS 파생 IP·혼합 출력의 포함 여부 |
| `freshness_policy` | Source별 필수 여부·stale 허용, DNS missing/stale 기여 포함 여부와 제공 기한 평가 |
| `allowlist_refs` | 적용할 Allowlist ID와 정확한 버전 |
| `exception_rule_set_ref` | 배포 단계에 반영할 예외 규칙 집합 버전 |
| `output_contract_ref` | 형식·정렬·분할·엔드포인트 계약 버전 |
| `access_mode` | [Feed 접근 모드](security-operations.md) 4종 중 하나 |
| `enabled` | 예약 생성·게시 대상 여부 |

DNS·ASN 보강은 두 프로파일에 나뉜다. **조회를 수행할지는 수집 프로파일**이, **파생 결과를 출력에 포함할지는 배포 프로파일**이 정한다.
어느 한쪽만으로 파생 IP가 배포되지 않으며, 조회를 수행하지 않은 수집 프로파일을 입력으로 삼은 배포 프로파일은 파생 출력을 선택할 수 없다.

필드 기본값과 필수 여부는 D-04, 출력 형식·소비자별 제약은 D-05에서 결정한다.
지원하지 않는 필터나 지표 타입은 무시하지 않고 설정 검증에서 실패시킨다.
명시적인 빈 Source 목록이나 빈 수집 프로파일 목록을 전체 선택으로 해석하지 않는 방안을 제안한다.

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

## 프로파일 수명과 선택

수집·배포 프로파일 모두 기본 프로파일에서 생성하거나 기존 프로파일을 복제하고 미리보기·활성화·비활성화·삭제·버전 복원을 제공한다.
템플릿과 복제본은 별도 ID로 관리하여 원본 수정이 복제본에 자동 전파되지 않는다.
수집 프로파일은 소스와 DNS Resolver/ASN 정책을, 배포 프로파일은 Allowlist 그룹·지표 기원·출력 형식을 버전에 포함한다.
기본 프로파일의 정책 값은 D-04에서 확정하고 공격적 차단 기본값을 임의로 넣지 않는다.
삭제·비활성화의 예약 중단, 공개본 회수, 이력 보존 효과를 구분한다.

Domain의 URL host 적용, 신뢰 도메인의 DNS 파생 IP 제외는 명시 교차 타입 정책으로만 허용한다.
공유 IP 전체를 근거 없이 제외하지 않고 부모 도메인·파생 경로를 검사한다.
직접 수집 IP와 DNS 파생 IP의 동일 주소가 병존하면 각각의 출처·제외 근거를 유지한다.

### Source·DNS 기여와 Output 제공 수명

다음은 D-03/D-05에 연결된 설계 제안이다. Source 최신성의 시각·필드는 [데이터 계약](architecture.md), DNS 관계의 제거 조건은 [DNS 변화 이력](pipeline.md)을 따른다.

| Source의 기준 시각 후 경과 | 상태 | 생성 입력 처리 |
| --- | --- | --- |
| `age < stale_after` | `fresh` | 유효한 기여를 포함 |
| `stale_after <= age < expire_after` | `stale` | Profile이 허용한 경우에만 최근 검증본의 기여를 포함하고 경고 |
| `age >= expire_after` | `expired` | 해당 Source 기여를 제외; 원문·이력은 retention에 따라 보존 |

24시간 주기 일반 Source에는 stale 72시간·expire 7일을 초기 제안으로 둔다. 분기·연간 자료에는 이를 일괄 적용하지 않고 제공 대상 기간·발행 주기에 맞춘 유한한 기한을 명시한다.
미설정·역전된 기한은 활성화 검증에서 거부한다. stale/missing 기여 포함도 명시 설정이며 묵시적으로 무기한 허용하지 않는다.
DNS 파생 기여는 Source와 부모 Domain 근거가 유효하고 DNS 자체 수명도 허용될 때만 포함한다. 어느 한 조건의 만료를 다른 조건의 갱신으로 연장하지 않는다.
만료된 Source/파생 경로를 제외해도 같은 IP에 다른 유효 기여가 있으면 그 근거로 유지하고 /24 집계 근거도 다시 계산한다.

Source의 `required` 여부는 Profile 버전에 명시한다. 선택 Source의 만료는 제외 사유를 기록한 새 generation으로 반영하며 다른 유효 Source의 출력은 계속 생성할 수 있다.
필수 Source를 사용할 수 없거나, 전체 만료·stale 불허·초기 조회 실패 등으로 정책상 사용할 수 있는 Source 스냅샷이 하나도 없으면 `unavailable`로 처리한다. 이를 정상 빈 Feed로 게시하지 않는다.
유효한 입력을 완전히 평가한 결과가 비어 있는 경우만 D-05의 정상 빈 결과 정책으로 처리한다.
정책에 따라 제외한 Source와 필수 샤드 segment 누락을 혼동하지 않으며, 남은 필수 segment는 모두 검증해야 게시할 수 있다.

Output 상태는 `healthy`(선택 입력 정상), `degraded`(허용된 stale/missing 또는 선택 Source 제외), `unavailable`(제공 가능한 완전본 없음)로 구분한다.
manifest에는 제외 사유와 사용한 기여의 기한을 기록한다. `serve_until`은 포함한 Source·DNS 기여의 최대 수명과, stale를 허용하지 않을 때의 stale 시작 시각 등 적용되는 제공 중단 기한 중 가장 이른 시각이다.
정상 누락/음성 응답의 제거 확정·부모 근거 해제·허용하지 않은 missing 전이 등으로 유효성이 먼저 끝나면 종속 revision의 제공 자격을 회수하고 재생성을 요청한다.

- 만료 전에 재평가를 예약하고 새 입력으로 검증한 불변 revision을 게시한다. 다운로드 중 기존 파일에서 항목을 동적으로 빼거나 같은 revision을 덮어쓰지 않는다.
- 새 생성이 실패해도 이전 revision은 현재 제공 자격이 있고 `now < serve_until`일 때만 제공한다. 같은 입력의 재생성·재게시로 기한을 초기화하지 않는다.
- `now >= serve_until` 또는 회수된 revision에는 인증·권한 확인 후 `503`을 반환하고 Feed 본문은 보내지 않는다. 현재/이전 revision·manifest·changes·checksum·HEAD·조건부 GET/304 모두 동일 자격을 검사한다.
- 캐시가 상한 이후 본문/304를 제공하지 않도록 재검증하고, 회수 시 무효화한다. `changes`의 기준/대상 revision도 각각 검사한다.
- 기한 내 `healthy`/`degraded` 출력은 `200`, stale 기여가 포함되면 `X-Feed-Stale: true`를 제공한다. 상태·경고는 고정 입력의 기준 시각과 요청 시각으로 판정하며 본문은 고정한다. 상세 상태·기한·사유는 관리 상태 API에서 보여준다.
- 이전 revision의 7일 조회 보존 제안은 serve_until·정책 회수보다 우선하지 않으며 파일의 물리 보존과도 별개다.

서버의 제공 중단이 소비자가 이미 받은 차단 목록의 자동 해제를 보장하지는 않는다. 소비자별 `503`·정상 빈 결과의 처리와 재수신 동작은 수용 시험으로 확인한다.

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
| `GET /feeds/v1/{distribution_profile_id}/{indicator_type}` | 현재 게시된 Feed | 한 응답은 하나의 불변 revision |
| `HEAD /feeds/v1/{distribution_profile_id}/{indicator_type}` | Feed 메타데이터 | GET과 같은 상태·revision, 본문 없음 |
| `GET /api/v1/distribution-profiles/{id}/status` | 게시·입력 상태 | 관리 권한으로 freshness 및 최근 실패 조회 |
| `GET /feeds/v1/{distribution_profile_id}/manifest` | 다중 파일 출력의 일관된 목록 | 한 revision과 타입별 불변 경로·체크섬 |
| `GET /feeds/v1/{distribution_profile_id}/revisions/{revision}/{indicator_type}` | 특정 revision의 파일 | 같은 revision의 내용은 변경하지 않음 |

- 출력은 IPv4/IPv6/CIDR/Domain/URL과 직접 IP·DNS 파생 IP·혼합을 구분한다. 정확한 경로 이름은 D-05에서 확정한다.
- TXT/CSV/JSON/hosts를 지원하며 format 선택·표현별 ETag·Content-Type을 명시한다. hosts에 URL/CIDR을 넣지 않는다.
- manifest는 표현별 불변 경로·checksum·건수·정책·입력·라우팅 버전을 제공한다. changes는 비교할 기준/대상 revision을 명시한다.
- 텍스트 출력은 한 줄에 한 항목을 제안하며 UTF-8, 줄바꿈, 정렬 및 끝 개행을 명시한다.
- IP·CIDR·Domain·URL 정규화와 중복 제거 규칙은 D-02 결정에 맞춘다.
- 배포 프로파일이 참조한 수집 결과와 Allowlist 반영을 마친 게시 완료 artifact만 제공한다.
- 요청 시작 시 공개 revision을 고정하여 응답 도중 새 파일 내용이 섞이지 않도록 한다(FR-007).
- 다중 파일 지원 시 소비자는 manifest를 한 번 읽고 불변 경로를 사용한다. 타입별 현재 경로를 여러 번 읽는 것은 동일 revision을 보장하지 않는다.
- 불변 경로에도 인증·권한 검사를 적용한다. 이전 revision의 제공 기한·회수·stale 정책과 파일 보관 기간을 D-03·D-05에서 함께 정한다.
- ETag를 사용할 경우 실제 응답 표현의 revision과 연결하고 조건부 GET을 지원하는 방식을 제안한다.
- 게시 시각과 마지막 입력 성공 시각은 별도 필드 또는 헤더로 구분해 제공한다.
- 헤더 후보는 `X-Feed-Revision`, `X-Feed-Published-At`, `X-Feed-Stale`이며 이름은 D-05에서 정한다.
- 성공 본문에 오류 설명이나 관리 메타데이터를 항목처럼 섞지 않는다.
- 배포 프로파일 간 또는 권한 간 캐시가 섞이지 않도록 캐시 키와 공유 캐시 허용 여부를 명시한다.
- 인증·권한 확인은 조건부 요청의 `304` 응답에도 적용한다.
- 첫 게시 전, 유효한 빈 결과, stale 초과, 삭제된 배포 프로파일을 서로 구분하는 정책을 확정한다.

### 관리 API 제안

| 메서드·경로 예시 | 목적 | 주요 요청 또는 응답 |
| --- | --- | --- |
| `GET, POST /api/v1/sources` | Source 조회·등록 | 이름, URL, 파서 유형, 활성 여부, 비밀 참조 |
| `GET, PATCH /api/v1/sources/{id}` | Source 조회·변경 | 구성 revision과 변경 필드 |
| `POST /api/v1/sources/{id}/runs` | 수동 수집 요청 | 접수한 작업 ID와 조회 경로 |
| `GET /api/v1/runs/{id}` | 작업 결과 조회 | 공통 작업 상태, 진행률·오류 분류 |
| `GET, POST /api/v1/collection-profiles` | 수집 프로파일 조회·등록 | 이름, Source 선택, 수집·보강 정책 |
| `GET, PATCH /api/v1/collection-profiles/{id}` | 수집 프로파일 조회·변경 | 구성 revision과 변경 필드 |
| `GET, POST /api/v1/distribution-profiles` | 배포 프로파일 조회·등록 | 이름, 수집 프로파일 참조, 출력 옵션, 접근 모드 |
| `GET, PATCH /api/v1/distribution-profiles/{id}` | 배포 프로파일 조회·변경 | 구성 revision, 정책 참조 |
| `GET, PATCH /api/v1/distribution-profiles/{id}/allowlist-groups` | 적용 그룹 조회·변경 | 그룹 ID·버전 참조, 기존 구성 revision |
| `POST /api/v1/distribution-profiles/{id}/publications` | 출력 재생성 요청 | 접수한 작업 ID와 대상 구성 revision |

- Allowlist·예외 규칙 집합과 Profile 참조는 새 버전으로 변경한다. 공유 집합의 변경을 기존 Profile·생성 작업에 암묵적으로 전파하지 않는다.
- Source URL 변경은 [보안·운영](security-operations.md)의 수집 경계 검증을 거친다.
- 비밀값은 조회 응답에서 반환하지 않으며 설정 여부와 참조만 노출한다.
- 비동기 작업 접수는 완료로 간주하지 않고 별도 조회에서 성공·실패를 확인한다.
- 중복 실행의 거부·병합·대기 방식과 재시도 키 계약을 D-05에서 확정한다.
- 구성 변경 응답과 실제 출력 게시를 분리하고 적용된 구성 revision을 상태 API에 표시한다.
- Source·수집/배포 프로파일 삭제와 비활성화의 데이터 보존 효과는 D-03 결정 후 계약에 추가한다.
- 수집 프로파일의 변경은 이를 참조하는 배포 프로파일에 자동 전파하지 않는다. 배포 프로파일이 참조 버전을 올려야 반영된다.

### 클라이언트 관리 제안

클라이언트는 배포 프로파일의 출력을 가져가는 머신 주체이며 사람 계정과 별도 엔터티다.
LDAP/SAML로 관리하는 사람 계정을 Feed 소비용으로 전용하지 않으며, 클라이언트에는 대화형 로그인 경로를 두지 않는다.

| 메서드·경로 예시 | 목적 | 주요 요청 또는 응답 |
| --- | --- | --- |
| `GET, POST /api/v1/clients` | 클라이언트 조회·등록 | 이름, 소유자·담당자, 설명, 허용 IP/CIDR, 활성 여부 |
| `GET, PATCH /api/v1/clients/{id}` | 클라이언트 조회·변경 | 구성 revision과 변경 필드 |
| `POST /api/v1/clients/{id}/tokens` | 토큰 발급 | 유효기간과 평문 토큰 1회 반환; 이후 조회 불가 |
| `DELETE /api/v1/clients/{id}/tokens/{token_id}` | 토큰 회수 | 즉시 무효화, 회수 시각·주체 감사 기록 |
| `GET /api/v1/clients/{id}/access-logs` | 접근 이력 조회 | 시각, 배포 프로파일, revision, 상태 코드, 전송량 |

- 토큰은 해시로만 저장하고 발급 응답에서 한 번만 평문을 노출한다. 목록·상세 응답은 식별자·만료·마지막 사용 시각만 반환한다.
- 토큰에는 유한한 만료를 요구하고 무기한 토큰을 기본값으로 두지 않는다. 만료와 회수는 별도 사유로 기록한다.
- 회수·만료·클라이언트 비활성화는 진행 중인 접근과 캐시 적중에도 적용한다. 반영 상한은 [인증·RBAC](security-operations.md)의 회수 계약을 따른다.
- 클라이언트가 어떤 배포 프로파일을 읽을 수 있는지는 [권한 모델](security-operations.md)로 검사한다. 인증 성공이 모든 배포 프로파일의 읽기 권한을 뜻하지 않는다.
- 클라이언트별 요청 제한·쿼타는 D-07의 전역 제한과 별개로 두고, 둘 중 먼저 도달한 한도를 적용한다. 초과는 `429`로 응답하며 재시도 가능 시점을 알린다.
- 접근 이력은 인증 실패·권한 부족·기한 만료 응답도 구분해 남기고, 토큰 평문이나 인증 헤더는 기록하지 않는다.
- 클라이언트 삭제 후에도 접근 이력의 주체 참조는 보존하고 식별자를 재사용하지 않는다.

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
| `/api/v1/allowlist-groups` | 재사용 그룹·항목 버전 관리, 배포 프로파일은 그룹 버전 참조 |
| `/api/v1/collection-profiles/{id}/clone` | 수집 프로파일 복제 |
| `/api/v1/distribution-profiles/{id}/clone`, `/api/v1/distribution-profiles/{id}/preview` | 복제, 이전 게시 대비 추가/제외/충돌 비교 |
| `/api/v1/exception-rules`, `/api/v1/monitor-rules`, `/api/v1/alerts` | 예외·모니터링 독립 관리, 경보 확인·해제·억제 |
| `/api/v1/indicators/{id}/history` | 출처·DNS·정제·게시 등록 구간, 조회 기간·페이지 크기 제한 |
| `/api/v1/identity-providers`, `/api/v1/roles` | 비밀 마스킹·고권한 검증·매핑 시험·감사 |
| `/api/v1/shards`, `/api/v1/backups`, `/api/v1/updates` | 별도 운영 권한·재인증·비동기 상태·충돌 제어 |

작업 상태는 [아키텍처](architecture.md)의 공통 enum을 사용한다.
취소·재시도·resume 가능 여부는 작업 종류·checkpoint로 응답한다.
파서/Profile 미리보기는 실제 게시를 수행하지 않으며 민감한 변경은 GET으로 실행하지 않는다.

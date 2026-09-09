# API 계약 초안

이 문서는 개발 검토용 제안이며 동작하는 API나 확정된 외부 계약을 의미하지 않는다.
이전 대화의 Profile별 Output Feed와 관리 영역 기술 후보를 바탕으로 새로 작성했다.
관리 API의 기능·경로·리소스 계약은 이번 초안의 설계 제안이다.
경로, 응답 형식, 빈 결과·stale 처리 및 버전 정책은 [DECISION_LOG.md](DECISION_LOG.md)의 D-05에서 확정한다.
인증 방식·관리 접근 범위는 D-08, 출력 항목 정규화는 D-02, 수명은 D-03에 따른다.

## 공통 계약 제안

- 예시 경로의 `/api/v1`과 `/feeds/v1`은 제안이며 구현 전에 확정한다.
- 관리 API는 JSON 요청·응답을 제안하고 외부 Feed의 형식과 분리한다.
- 관리 리소스에는 불변 ID, 구성 revision, 생성·수정 시각을 둔다.
- 시각은 시간대가 명시된 UTC 형식을 사용하고 표시 형식을 계약에 명시한다.
- 요청 ID를 응답과 로그에 연결하되 오류 메시지에 비밀값·내부 주소를 포함하지 않는다.
- 인증과 권한 검사는 목록, 단건 조회, 수정 및 출력 요청 모두에 적용한다.
- 충돌 감지를 위해 수정 요청에 기존 구성 revision을 보내는 방식을 제안한다.
- 목록 응답의 페이지 크기와 정렬 기준, 요청 상한은 D-05와 D-07에서 확정한다.

## Output Feed API 제안

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

## 관리 API 제안

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

- Allowlist 매칭·우선순위·공통 규칙 여부는 D-04에서 정하고 요청 검증에 동일하게 적용한다.
- Allowlist 그룹·항목 변경과 Profile의 그룹 참조 변경은 새 버전을 발행한다. 공유 그룹 변경을 기존 Profile에 암묵적으로 전파하지 않는다.
- Source URL 변경은 [DISTRIBUTION_SECURITY.md](DISTRIBUTION_SECURITY.md)의 수집 경계 검증을 거친다.
- 비밀값은 조회 응답에서 반환하지 않으며 설정 여부와 참조만 노출한다.
- 비동기 작업 접수는 완료로 간주하지 않고 별도 조회에서 성공·실패를 확인한다.
- 중복 실행의 거부·병합·대기 방식과 재시도 키 계약을 D-05에서 확정한다.
- 구성 변경 응답과 실제 출력 게시를 분리하고 적용된 구성 revision을 상태 API에 표시한다.
- Source·Profile 삭제와 비활성화의 데이터 보존 효과는 D-03 결정 후 계약에 추가한다.

## 상태 코드 및 오류 후보

- `200`: 조회 성공 또는 확정된 정책상 허용된 정상·stale Feed 응답.
- `201`, `202`: 리소스 생성, 비동기 작업 접수. 작업 접수 응답에는 추적 정보를 포함한다.
- `304`: 인증·권한 확인을 통과한 조건부 요청에서 표현이 변경되지 않음.
- `400`, `401`, `403`, `404`: 입력 오류, 인증 실패, 권한 부족, 리소스 없음의 계약 후보.
- `409`: 구성 revision 충돌 또는 계약상 동시에 수행할 수 없는 작업.
- `429`, `503`: 요청 제한, 제공 가능한 스냅샷 부재 또는 확정된 stale 차단 정책.
- 관리 오류 본문은 `code`, `message`, `request_id`, 선택적 필드 오류를 제안한다.
- 최초 게시·빈 결과·stale 응답 코드는 D-05 확정 전에는 클라이언트 호환 계약으로 취급하지 않는다.

검증에는 동시 게시 중 다운로드, 권한 경계, 조건부 GET, 수정 충돌 및 실패 작업 재시도를 포함한다.
운영 상태 정의는 [OPERATIONS_BACKUP.md](OPERATIONS_BACKUP.md), 전체 검증은
[VERIFICATION_CHECKLIST.md](VERIFICATION_CHECKLIST.md)를 참조한다.

## 추가 관리 계약

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

작업 상태는 [아키텍처](ARCHITECTURE.md)의 공통 enum을 사용한다.
취소·재시도·resume 가능 여부는 작업 종류·checkpoint로 응답한다.
파서/Profile 미리보기는 실제 게시를 수행하지 않으며 민감한 변경은 GET으로 실행하지 않는다.

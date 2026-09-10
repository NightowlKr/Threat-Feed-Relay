# 개발 절차·결정 로그

현재 사용자에게 승인된 작업 범위가 실행 권한의 기준입니다.
요구사항은 개발 목표이지 커밋·push·배포·병합·외부 API 실행에 대한 포괄 승인이 아닙니다.
기능 지원 방향을 다시 묻지 않고, 작업에 필요한 상세 미정만 결정합니다.

## 결정 로그

아래 ID를 유지하고 선택·근거·영향·확정일·검증 증거를 갱신합니다.
확인된 방향은 세부 정책이나 기술 호환성까지 승인됐다는 의미가 아닙니다.
"설계 제안" 열은 구현 전 제안값이며 실측·검증·외부 사실 확인 전에는 운영 기본값으로 확정된 것이 아닙니다.
API·필드 등 문서 본문에 이미 구체 제안이 있는 항목은 중복 기술하지 않고 담당 문서를 참조합니다.

| ID | 주제 | 확인된 방향 | 남은 결정 | 설계 제안(미검증) | 책임 문서 |
| --- | --- | --- | --- | --- | --- |
| D-01 | 초기 Feed | 공공 WHOIS·분기 IoC·C-TAS 및 일반 Feed 연동 요청 | 최신 API·인증·주기·재배포 권리·샘플 승인 | 수집 주기 24시간(소스별 override), 소스당 동시 실행 1개, 재시도 지수 백오프 최대 3회; 직전 대비 건수 50% 이상 감소 시 자동 승인하지 않고 검토 대기, 전체 목록 소스의 완전 빈 응답은 즉시 거부. 실제 API·인증·재배포 권리는 여전히 외부 사실 확인 필요 | [제공자](pipeline.md) |
| D-02 | 정규화 | IPv4/IPv6/CIDR/Domain/URL, 여러 파서 | IDNA·host bit·URL 세부 규칙과 버전 | IPv4-mapped IPv6는 IPv4로도 정규화해 두 표현을 연결, 특수 용도 주소(RFC 6890 loopback·link-local·문서용 등)는 지표로 채택하지 않고 거부; CIDR host bit 입력은 자동 보정 없이 거부; Domain은 후행 점 제거 + IDNA UTS-46 non-transitional 정규화, wildcard(`*.`)는 별도 타입; URL은 scheme을 http/https로 제한하고 경로·쿼리는 원본 대소문자·순서 보존 | [파이프라인](pipeline.md) |
| D-03 | 수명·이력 | DNS 변화·출처·정제/배포 기간 추적, 수신 차단 Domain의 IP 조회·정상 변경/실패 구분·출력 제외와 이력 보존 분리 요청 | DNS 조회 간격·TTL/캐시 처리, 누락/음성 확인 횟수·유예, Source별 freshness 기준·기한, 원문·이력 retention | 원문 보존은 성공 90일·실패/거부 30일 제안을 유지하되 필수 참조를 보호한다. DNS 정상 누락·NXDOMAIN/NODATA·실패/불일치의 상태·제거 조건·제안 수치는 [DNS 변화 이력](pipeline.md), Source별 stale/expire와 Output 반영은 [출력 수명](profiles-api.md)을 단일 기준으로 사용한다. 수치는 미검증이며 운영 활성화 전 확정 | [DB](architecture.md), [보강](pipeline.md), [출력](profiles-api.md) |
| D-04 | Profile·예외 | 수집·배포 프로파일 분리, 복제·다중 소스/Allowlist, 예외·모니터링, /24 승격 | 승격·해제 임계치, 교차 타입·부분 CIDR·예외 우선순위, 수집 프로파일 변경의 배포 반영 승인 절차 | 기본 Profile은 소스·Allowlist 없이 생성되고 명시적 활성화 전 `enabled=false`; 예외 우선순위(높은 순) `quarantine` > `distribution_exclude` > `monitor_only` > `score_ignore` > `tag_only`(`exclude`는 정제 단계에서 선적용, `force_include`는 이 순서를 덮어쓰지 않음); 부분 겹침 CIDR 분할 상한 8개(초과 시 자동 분할 대신 게시 거부) | [Profile](profiles-api.md) |
| D-05 | 출력 | TXT/CSV/JSON/hosts, 직접·파생·혼합 구분, 불변 스냅샷 기준선; stale 경고와 제공 중단의 분리 방향 반영 | 경로·정렬·크기·Source 필수 여부·stale/missing 허용·소비자별 만료 응답 처리·빈 결과·revision 보존 | 이전 revision 조회 기간 7일 제안은 정책 회수·serve_until을 넘지 못한다. 최초 게시 전 404, 유효한 빈 결과 200; 기한 내 stale 경고와 만료/회수 후 503 및 새 불변 출력 생성은 [Output 제공 수명](profiles-api.md)을 따른다. 목록 기본 50·최대 200·정렬 `created_at desc`; 중복 생성 요청은 409 + 진행 중 작업 ID, `Idempotency-Key` 동일 시 신규 작업 생성 안 함 | [API](profiles-api.md) |
| D-06 | 기술·DB | Laravel/Python 분리, 복수 DB, Timescale 선택 확장, 단일 스키마 소유권 | 실제 버전·패키지·DB 호환 CI·성능 | 버전 후보는 [아키텍처의 기술 후보 표](architecture.md)를 단일 참조로 사용하며 여기서 중복 기술하지 않는다. 공통 DB의 MariaDB는 11.8로 확정 제안: 12.3 LTS의 신규 기능은 이 프로젝트에 불필요하고, `innodb_snapshot_isolation` 기본값 변경으로 REPEATABLE READ 동작이 달라져 트랜잭션 계약을 다시 검증해야 한다. PostgreSQL 18은 TimescaleDB 2.23.0+에서 지원을 확인했다. CI 매트릭스는 MariaDB 11.8·PostgreSQL 18 각 1개 + Timescale 확장 조합 1개를 최소 커버리지로 제안하며, 실제 패키지 조합·성능은 착수 시점에 재검증한다 | [아키텍처·데이터](architecture.md) |
| D-07 | 운영 목표 | 로그·백업·오류 경보·복구·업데이트 요청 | 규모·지연·보관·RPO/RTO·알림 채널·임계치, 클라이언트별 쿼타 기본값 | 요청 제한 Feed 다운로드 IP당 분당 30회·관리 API 계정당 분당 120회·클라이언트별 쿼타는 전역 제한과 별개 적용; 수집 상한 응답 100MB·압축 해제 500MB·행 8KB·실행 10분·소스 간 동시 5개; RPO 24시간·RTO 4시간, 복구 훈련 분기 1회; 백업 보존 일간 14일 + 주간 8주 + 원격 복제본 1곳 이상; 경보 채널 기본 email(+선택 webhook), 동일 dedup key cooldown 15분 | [운영](security-operations.md) |
| D-08 | 인증·접근 | LDAP/AD·SAML·RBAC와 배포 4개 접근 모드 | IdP 연동 라이브러리·매핑·회수 시간·MFA·세션·감사 보존 | 기본 역할셋은 현재 후보 9개(super_admin\~api_consumer) 유지; 관리 UI는 세션 쿠키+CSRF, Feed/관리 API 연동은 API token만 사용(CSRF 대상 제외); 권한·그룹 변경은 동기화 주기 기준 15분 이내 세션/token에 반영; IdP 장애 중 기존 세션은 자연 만료까지 유지하되 신규 로그인은 fail-closed; 감사 로그 보존 1년. 실제 IdP 연동 라이브러리는 구현 시 확인 | [인증](security-operations.md), [보안](security-operations.md) |
| D-09 | 라이선스 | 공개 README, 프로젝트 라이선스는 정책 결정 전 보류 | 프로젝트·의존성·각 Feed 라이선스 | 전부 외부 사실 확인이 필요해 제안값 없음 | [제공자](pipeline.md) |
| D-10 | Git 운영 | 역할별 작업 브랜치와 PR 검토, GitHub Flow 기준선 | Git Flow 전환·보호 규칙·승인 권한은 별도 결정 | main 브랜치 보호 제안: PR 1인 이상 승인 + CI 통과 필수, force-push·직접 push 금지 | [Git 운영](workflow.md) |
| D-11 | HA·샤딩 | 사용자 요청: DB HA, Timescale 애플리케이션 샤딩, 출력 Feed 1차 분할 | shard/bucket 수, 복제 수준, 라우팅 세부·복구 실측 | 초기 가상 버킷 256개/Feed, IPv6 grouping 기본 `/64`(Profile별 override 가능), 그룹 경계를 넘는 큰 CIDR는 최대 4개 range로 분할 조회(초과 시 게시 거부), 배치 모드는 기본 single(복제가 필요한 Feed만 replicated=2로 전환). `collection_profile_id`와 `distribution_profile_id`는 별개 엔터티로 확정했고 라우팅 경계는 `distribution_profile_id`를 사용한다 | [클러스터](architecture.md) |
| D-12 | 장기 유지보수 | Docker·소스 설치·자동 릴리스 업데이트, 2029\~2030 유지와 주요 버전 업그레이드 검토 | 지원 조합·업그레이드 주기·서명·승인·복구 정책 | 분기 1회 의존성·이미지 보안 점검, 주요 버전 업그레이드는 별도 feature 브랜치에서 정식 릴리스로 최소 1회 검증 후 반영. 실제 지원 조합·서명 방식은 외부 사실 확인이 필요해 여전히 미정 | [운영](security-operations.md) |

## 개발 단계

모든 단계는 계획이며 완료 표시가 아닙니다.

| 단계 | 범위와 연결 요구사항 | 산출물 및 종료 조건 |
| --- | --- | --- |
| 0. 기준선·계약 | 전체 FR/NFR, D-01\~D-12 | 문서 검토, 승인된 합성 샘플, 초기 규모·DB 조합·정책 결정, Git 규칙 |
| 1. 관리 기반 | FR-001, FR-004, FR-016, FR-020, NFR-001, NFR-002, NFR-004 | Laravel UI/API, 로컬 인증·권한, 설정 버전, DB 공통 계약, 작업 원장·outbox, Docker/소스 골격 |
| 2. 수집·정규화 | FR-001, FR-002, FR-005, FR-008, FR-018 | 파서 미리보기, 수집 어댑터·제한·재시도, 원문·출처 스냅샷, 제공자별 계약 시험 |
| 3. 보강·이력 | FR-003, FR-009, FR-010, FR-011 | Resolver·DNS 변화, ASN LPM·배치·데이터셋, 지표 이력과 출처 집계 |
| 4. Profile·출력 | FR-004, FR-006, FR-007, FR-012, FR-019 | 복제·다중 선택, 정제 membership, /24 승격, 최종 Allowlist, manifest·불변 출력·동시 게시 검증 |
| 5. 보안·운영 기능 | FR-013, FR-014, FR-016, FR-017, FR-020, NFR-002, NFR-003 | 예외·모니터링·알림, 대시보드, LDAP/SAML·권한 회수, 4개 접근 모드·trusted proxy, 백업 복원 |
| 6. HA·고급 저장 | FR-015, NFR-004, NFR-005 | DB HA·단일 writer, 출력 Feed별 샤드·directory·rollup, 장애·복제·전체 백업 복원 |
| 7. 릴리스·유지보수 | NFR-006, NFR-007 | DBMS별 SQL, expand/contract, 릴리스 서명·호환 검사, 업데이트·복구 훈련, 업그레이드 CI |
| 8. 확장·성능 | FR-008, FR-009, FR-015, NFR-005, NFR-007 | XML/STIX/plugin, DoT/DoH, 온라인 재균형, 목표 부하·장기 운영 시험 |

단계 1부터 보안·로깅·재시도·스키마 호환성을 적용한다.
단계 5는 보안을 처음 추가하는 시점이 아니라 고급 인증·운영 기능의 통합 완료 단계다.
단계 6을 구현하기 전까지 단일 샤드라도 출력 Feed 경계·라우팅 버전을 보존한다.
여러 DB 지원은 나중에 SQL 문법만 치환하는 방식으로 미루지 않고 단계 1부터 공통 계약을 시험한다.

### 각 단계의 공통 완료 조건

- 대상 FR/NFR 및 D-ID에 구현·테스트 증거를 연결한다.
- 정상·실패·경계·중복·재시작·권한 사례를 확인한다.
- 무단 재배포 자료나 실제 인증정보를 fixture로 쓰지 않는다.
- 이전 단계 데이터·정책·API와 호환되거나 이관 경로를 제공한다.
- 성능·보존 기간·RPO/RTO는 실측과 승인된 목표로 판정한다.
- 기능 브랜치의 검토와 검증 후 별도 승인된 PR 흐름으로 통합한다.

## Git 운영

현재 기준은 짧은 역할 브랜치에서 검토한 뒤 PR로 통합하는 GitHub Flow입니다.
특정 과거 문서 브랜치를 후속 작업의 고정 대상으로 사용하지 않습니다.
브랜치 접두사는 저장소 명명 규칙이며 Git의 특별한 기능은 아닙니다.

| 브랜치 | 역할·흐름 |
| --- | --- |
| `main` | 검토된 기준선; 직접 push하지 않고 승인된 PR로 통합 |
| `feature/<topic>` | 기능·문서 추가; main에서 분기하여 main 대상 PR |
| `bugfix/<topic>` | 일반 오류 수정; main에서 분기하여 main 대상 PR |
| `hotfix/<topic>` | 운영 긴급 수정; 실제 배포 기준에서 분기하여 관련 유지보수 계열에도 반영 |
| `release/<version>` | 필요 시 릴리스 안정화; 기준·통합 대상을 작업별 명시 |
| `develop` | Git Flow 전환 시 다음 릴리스 통합; 현재 필수 아님 |
| `master` | 다른 저장소의 기본 브랜치 명칭; 이 저장소에서는 main 사용 |

기존 변경·기준 커밋·원격을 확인하고 작업 파일만 커밋합니다.
허용된 브랜치만 명시적으로 push하며 force push·전체 브랜치 push·자동 병합을 기본 절차로 두지 않습니다.
승인된 PR 병합은 직접 main push와 구분합니다. 보호 규칙 설정·브랜치 삭제도 별도 승인 범위에서 수행합니다.
커밋 메시지는 목적에 따라 `docs:`, `feat:`, `fix:`, `test:`, `chore:`를 사용합니다.

복수 배포 버전의 안정화가 필요해 Git Flow로 전환하면 D-10을 갱신합니다.
feature/bugfix는 develop으로, release는 main과 develop으로 통합합니다.
hotfix는 배포 기준에서 분기해 main 및 진행 중 develop/release에도 반영하며 두 전략의 대상을 혼용하지 않습니다.

참고: [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow),
[Git Flow 원저자의 설명](https://nvie.com/posts/a-successful-git-branching-model/).

## 구현 작업 양식

아래 항목을 한 작업 요청에 채웁니다. 양식 자체는 실행 지시가 아닙니다.

- 목표·사용자에게 보이는 결과:
- 대상 FR/NFR·설계 문서·D-ID와 결정 상태:
- 기준 커밋·작업 브랜치:
- 포함/제외 범위·수정 파일·기존 변경과의 충돌:
- 승인된 합성/실제 샘플·기대 결과:
- 정상·실패·경계·재시도·권한 테스트:
- 커밋·push·PR·배포에 허용된 작업:
- 완료 보고: 변경 요약·충족 ID·실행 환경/명령/결과·미실행 항목·남은 결정·생성한 커밋/PR.

문서 변경은 [검증 기준](verification.md)으로 확인하고 기능 변경은 해당 수용기준의 실제 증거를 남깁니다.

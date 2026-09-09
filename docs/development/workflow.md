# 개발 절차·결정 로그

현재 사용자에게 승인된 작업 범위가 실행 권한의 기준입니다.
요구사항은 개발 목표이지 커밋·push·배포·병합·외부 API 실행에 대한 포괄 승인이 아닙니다.
기능 지원 방향을 다시 묻지 않고, 작업에 필요한 상세 미정만 결정합니다.

## 결정 로그

아래 ID를 유지하고 선택·근거·영향·확정일·검증 증거를 갱신합니다.
확인된 방향은 세부 정책이나 기술 호환성까지 승인됐다는 의미가 아닙니다.

| ID | 주제 | 확인된 방향 | 남은 결정 | 책임 문서 |
| --- | --- | --- | --- | --- |
| D-01 | 초기 Feed | 공공 WHOIS·분기 IoC·C-TAS 및 일반 Feed 연동 요청 | 최신 API·인증·주기·재배포 권리·샘플 승인 | [제공자](pipeline.md) |
| D-02 | 정규화 | IPv4/IPv6/CIDR/Domain/URL, 여러 파서 | IDNA·host bit·URL 세부 규칙과 버전 | [파이프라인](pipeline.md) |
| D-03 | 수명·이력 | DNS 변화·출처·정제 및 배포 등록 기간 추적 | TTL 외 지표 수명, 실패 유예, 원문·이력 보존 | [DB](architecture.md), [보강](pipeline.md) |
| D-04 | Profile·예외 | 복제·다중 소스/Allowlist, 예외·모니터링, /24 승격 | 승격·해제 임계치, 교차 타입·부분 CIDR·예외 우선순위 | [Profile](profiles-api.md) |
| D-05 | 출력 | TXT/CSV/JSON/hosts, 직접·파생·혼합 구분, 불변 스냅샷 기준선 | 경로·정렬·크기·stale·빈 결과·revision 보존 | [API](profiles-api.md) |
| D-06 | 기술·DB | Laravel/Python 분리, 복수 DB, Timescale 선택 확장, 단일 스키마 소유권 | 실제 버전·패키지·DB 호환 CI·성능 | [아키텍처·데이터](architecture.md) |
| D-07 | 운영 목표 | 로그·백업·오류 경보·복구·업데이트 요청 | 규모·지연·보관·RPO/RTO·알림 채널·임계치 | [운영](security-operations.md) |
| D-08 | 인증·접근 | LDAP/AD·SAML·RBAC와 배포 4개 접근 모드 | IdP 연동 라이브러리·매핑·회수 시간·MFA·세션·감사 보존 | [인증](security-operations.md), [보안](security-operations.md) |
| D-09 | 라이선스 | 공개 README, 프로젝트 라이선스는 정책 결정 전 보류 | 프로젝트·의존성·각 Feed 라이선스 | [제공자](pipeline.md) |
| D-10 | Git 운영 | 역할별 작업 브랜치와 PR 검토, GitHub Flow 기준선 | Git Flow 전환·보호 규칙·승인 권한은 별도 결정 | [Git 운영](workflow.md) |
| D-11 | HA·샤딩 | 사용자 요청: DB HA, Timescale 애플리케이션 샤딩, 출력 Feed 1차 분할 | shard/bucket 수, 복제 수준, 라우팅 세부·복구 실측 | [클러스터](architecture.md) |
| D-12 | 장기 유지보수 | Docker·소스 설치·자동 릴리스 업데이트, 2029~2030 유지와 Laravel 14 검토 | 지원 조합·업그레이드 주기·서명·승인·복구 정책 | [운영](security-operations.md) |

## 개발 단계

모든 단계는 계획이며 완료 표시가 아닙니다.

| 단계 | 범위와 연결 요구사항 | 산출물 및 종료 조건 |
| --- | --- | --- |
| 0. 기준선·계약 | 전체 FR/NFR, D-01~D-12 | 문서 검토, 승인된 합성 샘플, 초기 규모·DB 조합·정책 결정, Git 규칙 |
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

# 기술·제품 결정 로그

기능 요청 자체와 상세 미정을 구분한다.
사용자 요청은 다시 승인받지 않으며, 대화 기준선의 상세 구현은 검증 후 확정한다.
변경 시 날짜·근거·영향·관련 요구사항과 검증 증거를 해당 ID에 추가한다.

| ID | 주제 | 확인된 방향 | 남은 결정 | 책임 문서 |
| --- | --- | --- | --- | --- |
| D-01 | 초기 Feed | 공공 WHOIS·분기 IoC·C-TAS 및 일반 Feed 연동 요청 | 최신 API·인증·주기·재배포 권리·샘플 승인 | [제공자](PROVIDERS.md) |
| D-02 | 정규화 | IPv4/IPv6/CIDR/Domain/URL, 여러 파서 | IDNA·host bit·URL 세부 규칙과 버전 | [파이프라인](FEED_PIPELINE.md) |
| D-03 | 수명·이력 | DNS 변화·출처·정제 및 배포 등록 기간 추적 | TTL 외 지표 수명, 실패 유예, 원문·이력 보존 | [DB](DATABASE_DESIGN.md), [보강](ENRICHMENT.md) |
| D-04 | Profile·예외 | 복제·다중 소스/Allowlist, 예외·모니터링, /24 승격 | 승격·해제 임계치, 교차 타입·부분 CIDR·예외 우선순위 | [Profile](PROFILES_ALLOWLIST.md) |
| D-05 | 출력 | TXT/CSV/JSON/hosts, 직접·파생·혼합 구분, 불변 스냅샷 기준선 | 경로·정렬·크기·stale·빈 결과·revision 보존 | [API](API_CONTRACTS.md) |
| D-06 | 기술·DB | Laravel/Python 분리, 복수 DB, Timescale 선택 확장, 단일 스키마 소유권 | 실제 버전·패키지·DB 호환 CI·성능 | [아키텍처](ARCHITECTURE.md), [DB](DATABASE_DESIGN.md) |
| D-07 | 운영 목표 | 로그·백업·오류 경보·복구·업데이트 요청 | 규모·지연·보관·RPO/RTO·알림 채널·임계치 | [운영](OPERATIONS_BACKUP.md) |
| D-08 | 인증·접근 | LDAP/AD·SAML·RBAC와 배포 4개 접근 모드 | IdP 연동 라이브러리·매핑·회수 시간·MFA·세션·감사 보존 | [인증](IDENTITY_ACCESS.md), [보안](DISTRIBUTION_SECURITY.md) |
| D-09 | 라이선스 | 공개 README, 프로젝트 라이선스는 정책 결정 전 보류 | 프로젝트·의존성·각 Feed 라이선스 | [제공자](PROVIDERS.md) |
| D-10 | Git 운영 | 사용자 요청: main 직접 반영 금지, 역할별 브랜치, future 오탈자 정정 | 현재 GitHub Flow 적용; 향후 Git Flow 전환은 별도 결정 | [Git 운영](GIT_WORKFLOW.md) |
| D-11 | HA·샤딩 | 사용자 요청: DB HA, Timescale 애플리케이션 샤딩, 출력 Feed 1차 분할 | shard/bucket 수, 복제 수준, 라우팅 세부·복구 실측 | [클러스터](CLUSTER_SHARDING.md) |
| D-12 | 장기 유지보수 | Docker·소스 설치·자동 릴리스 업데이트, 2029~2030 유지와 Laravel 14 검토 | 지원 조합·업그레이드 주기·서명·승인·복구 정책 | [운영](OPERATIONS_BACKUP.md) |

## 이번 문서의 적용 결정

D-10: `feature/development-requirements`에만 커밋·업로드한다.
`main` 수정·병합, 원격 보호 규칙 변경, 불필요한 역할 브랜치 일괄 생성은 하지 않는다.
GitHub Flow 선택은 현재 문서 단계의 작업 방식이며 사용자가 모든 세부 규칙을 별도 승인했다는 의미는 아니다.

D-06/D-11: 이전 초안의 “DB·인증·샤딩 전체 미정” 표현을 제거했다.
기능 지원 방향은 복구한 상세 대화로 확인했고, 실제 호환성과 규모만 검증 대기다.

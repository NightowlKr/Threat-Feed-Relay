# 개발 요구사항

Threat Feed Relay(TFR)는 위협 지표의 출처·변경 이력을 유지하며 Profile별 Feed를 생성하는 플랫폼입니다.
관리자·Feed 운영자·보안 검토자·조회 사용자·Feed 소비자의 권한을 분리합니다.
GIS, 악성 URL 방문·실행, 능동 스캔, 무단 데이터 재배포는 범위 밖입니다.
XML/STIX·plugin, DoT/DoH, 온라인 재균형은 후속 개발 단계이며 제거된 요구사항이 아닙니다.

상태는 [개발 문서 안내](README.md)를 따릅니다. 아래 수용기준은 제안이며 구현·검증 결과가 아닙니다.

## 기능 요구사항

| ID | 요구사항과 근거 상태 | 제안 수용기준 | 상세 |
| --- | --- | --- | --- |
| FR-001 | 사용자 요청: 외부 Threat/Blacklist Feed 수집 | 원문 참조·수집 실행·성공/실패·건수 연결; 부분 실패로 정상 스냅샷을 교체하지 않음 | [파이프라인](pipeline.md), [제공자](pipeline.md) |
| FR-002 | 사용자 요청: IP/CIDR/Domain/URL 정규화 | IPv4/IPv6·중복·오류와 URL 경로/쿼리 보존 사례 검증; 버전별 재현 | [파이프라인](pipeline.md) |
| FR-003 | 사용자 요청: DNS·ASN 보강 | 수신 차단 Domain의 IP를 출처·부모별 DNS 파생 기여로 연결; 제공자·조회 시각·유효기간·실패 기록, 실패가 원본 삭제·안전 판정으로 바뀌지 않음 | [보강](pipeline.md) |
| FR-004 | 사용자 요청: Profile별 출력 | 기본 Profile, 생성·복제·수정·비활성화, 다중 Source/Allowlist 그룹; 고정 입력·정책의 결정적 결과 | [Profile](profiles-api.md) |
| FR-005 | 사용자 요청: 출처·수집 이력 | 같은 지표의 복수 출처 보존, 한 Source 해제·실패가 다른 유효 출처를 훼손하지 않음 | [DB](architecture.md) |
| FR-006 | 사용자 요청: Allowlist | IP/CIDR/Domain/wildcard/URL/ASN 규칙·설명·근거·만료, 다중 그룹; 최종 출력의 제외 사유 추적 | [Profile](profiles-api.md) |
| FR-007 | 대화 기준선: 불변 출력 스냅샷 | 미완성·일부 샤드 누락·구세대 덮어쓰기 차단; manifest 고정 다중 파일 일관성, 기여 만료 시 새 revision 생성과 제공 상한 적용 | [API](profiles-api.md), [클러스터](architecture.md) |
| FR-008 | 사용자 요청: 다형식 파서와 미리보기 | TXT·토큰·regex·CSV/TSV·hosts·JSON/JSONL 샘플 검증; XML/STIX/plugin 단계 확장; 오류·추출 건수 확인 후 활성화 | [파이프라인](pipeline.md) |
| FR-009 | 사용자 요청: DNS Resolver 관리·변화 추적 | 시스템/사용자 Resolver의 A/AAAA/CNAME 비교; 정상 누락·NXDOMAIN/NODATA·실패/불일치 구분, 유예·최대 수명에 따른 active/missing/expired 및 추가/제거/재등장 이력과 파생 출력 | [보강](pipeline.md) |
| FR-010 | 사용자 요청: ASN/CIDR 조회 | 개별·일괄·예약 실행, 로컬 LPM과 외부 조회, ASN/type/owner/country·데이터셋 버전; GIS 제외 | [보강](pipeline.md) |
| FR-011 | 사용자 요청: 등록 기간과 대시보드 | 최초·최근 관측, 정제·실제 배포의 현재 연속/누적 등록 기간, 출처 수·태그를 구분해 조회 | [DB](architecture.md), [운영](security-operations.md) |
| FR-012 | 사용자 요청: /24 임계치 집계 | 출력 Feed별 활성 고유 IPv4 수와 관측 창으로 승격; 출처 수 별도; 해제 임계·Allowlist/배포 제외 재포함 검증 | [Profile](profiles-api.md) |
| FR-013 | 사용자 요청: 수집 예외 | IP/CIDR별 적용 범위·기간·동작 관리; 원문·관측·모니터링 보존, 불변 예외 버전 고정·집계 후 배포 제외 | [Profile](profiles-api.md) |
| FR-014 | 사용자 요청: 예외 대상 등 모니터링·알림 | 최초/재발견/지속·출처 수/IP 수 조건, 상태 전이·중복 억제·재시도·전달 실패 추적 | [운영](security-operations.md) |
| FR-015 | 사용자 요청: 출력 Feed 중심 애플리케이션 샤딩 | 선택 Feed 다중 샤드, 버전 라우팅·directory·공통 테이블; 필수 segment 누락 시 불완전 게시 차단 | [클러스터](architecture.md) |
| FR-016 | 사용자 요청: LDAP/AD·SAML와 역할·권한 | 불변 외부 subject 매핑, 내부 RBAC, 그룹 변경·계정 비활성 후 세션/토큰 회수, 로컬 비상 관리자 | [인증](security-operations.md) |
| FR-017 | 사용자 요청: Feed 배포 4개 접근 모드 | 공개/IP 제한/계정 전 IP/계정+IP를 각각 검사; 신뢰 프록시·HEAD/304·캐시·이전 revision으로 권한·제공 기한·회수 우회 불가 | [보안](security-operations.md), [API](profiles-api.md) |
| FR-018 | 사용자 요청: 공공데이터 및 C-TAS 연동 | WHOIS·분기 IoC와 차단/해제·기간 자료 구분, 순서 역전·중복·해제 없음·ZIP 오류 시험 | [제공자](pipeline.md) |
| FR-019 | 사용자 요청: 형식·기원별 배포 | TXT/CSV/JSON/hosts와 직접 IP/DNS 파생/혼합, manifest·changes·checksum; 허용된 타입만 형식에 포함 | [API](profiles-api.md) |
| FR-020 | 사용자 요청 및 대화 기준선: 관리 UI·API·작업 제어 | 설정 revision 충돌, 미리보기·예약/수동·재시도·취소·배치 진행률, 권한별 조회·변경 검증 | [API](profiles-api.md), [아키텍처](architecture.md) |

## 비기능 요구사항

| ID | 요구사항과 근거 상태 | 제안 수용기준 |
| --- | --- | --- |
| NFR-001 | 사용자 요청: Docker 배포 | 신규 설치·재시작·업데이트 시 영속 설정·지표·산출물 보존, 내부 포트 비공개 |
| NFR-002 | 사용자 요청 및 설계 제안: 접근·비밀·수집 보안 | 최소 권한, 비밀 참조, trusted proxy, SSRF/redirect/rebinding·파서·압축 한도·TLS 검증 |
| NFR-003 | 사용자 요청: 로그·백업·복구 | 수집·로그인·게시·백업 오류와 접근 로그, 설정/원문·정제/배포/로그의 정책별 백업; 복원 후 일관성·권한 확인 |
| NFR-004 | 사용자 요청: MariaDB/PostgreSQL/Timescale 호환 | Standard 공통 기능을 동일 계약으로 시험, Advanced 확장 구분; DBMS별 마이그레이션 단일 소유권 |
| NFR-005 | 사용자 요청: HA·클러스터 | 단일 writer·리더 전환·락 만료·중복 배달·샤드 장애에서 데이터 유실/중복 게시 방지; 복구 목표 실측 |
| NFR-006 | 사용자 요청: 소스 설치·Git 연계 릴리스 업데이트 | Docker와 동일 기능 계약, 사전 호환·서명/해시·백업·업데이트 락·상태 검사; DB 호환에 따른 안전한 복구 |
| NFR-007 | 사용자 요청: 2029\~2030 유지보수와 업그레이드 | 지원 조합표·잠금 파일·계약 CI·업그레이드 브랜치·보안 점검; 지원 날짜나 미래 호환을 근거 없이 보증하지 않음 |

성능·보존 기간·RPO/RTO·상세 정책은 [결정 로그](workflow.md)에서 확정하고
[검증](verification.md)에 실제 결과를 연결합니다.
외부 구현 사례에서 얻은 사전 이슈·기존 요구사항과의 연결·추가 시험 계획은 [외부 Feed 구현 참고와 사전 점검](verification.md#외부-feed-구현-참고와-사전-점검)에 기록합니다. 참고 프로젝트의 기능이나 동작을 TFR의 승인된 요구사항·검증 완료로 간주하지 않습니다.

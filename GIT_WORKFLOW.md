# Git 브랜치 운영 규칙

## 현재 전략

문서·구현을 짧은 작업 브랜치에서 검토한 뒤 PR로 통합하는 GitHub Flow를 적용한다.
현재 작업은 `feature/development-requirements`이며 `future-draft`는 사용자 정정으로 사용하지 않는다.
`main`에 직접 커밋·push·병합하지 않는다. 이번 요청에서는 별도 브랜치 등록과 검토까지만 수행한다.

GitHub Flow는 별도 브랜치 변경·검토·통합 흐름이며 `develop`을 필수로 요구하지 않는다.
참고: [GitHub 공식 흐름](https://docs.github.com/en/get-started/using-github/github-flow).

## 역할과 이름

아래 접두사는 이 저장소의 명명 규칙이다. Git 자체의 특별한 브랜치 유형은 아니다.

| 이름 | 역할 | 현재 분기·통합 기준 |
| --- | --- | --- |
| `main` | 검토된 기준선, 향후 배포 가능한 코드 | PR로만 통합하는 운영 규칙 |
| `feature/<topic>` | 기능·설계·요구사항 추가 | main에서 분기 → main 대상 PR |
| `bugfix/<topic>` | 일반 결함·문서 오류 수정 | main에서 분기 → main 대상 PR |
| `hotfix/<topic>` | 운영 긴급 수정 | 배포된 기준에서 분기, 운영 및 진행 중 개발 계열에 수정 반영 |
| `release/<version>` | 별도 안정화가 필요한 릴리스 준비 | 필요할 때만 생성; 기준·지원 범위를 PR에 명시 |
| `develop` | Git Flow 전환 시 다음 릴리스 통합 | 현재 만들지 않음 |
| `master` | 일부 저장소의 기본 브랜치 명칭 | 이 저장소는 main 사용; 병렬 생성하지 않음 |

이름은 영문 소문자·숫자·하이픈으로 목적을 표현한다.
예: `feature/dns-enrichment`, `bugfix/cidr-allowlist`, `hotfix/feed-access-control`.
모든 역할 브랜치를 미리 만들지 않는다.

## 변경·검토 규칙

1. 기준 브랜치·원격·작업 트리와 기존 사용자 변경을 확인한다.
2. 범위별 작업 브랜치를 만들고 관련 파일만 커밋한다.
3. 요구사항·결정 ID, 정상·실패 검증 및 미실행 항목을 기록한다.
4. 해당 브랜치만 명시적으로 push한다. 무차별 전체 브랜치 push나 force push는 하지 않는다.
5. PR에 변경 목적·검증·운영 영향·미정을 남기고 검토 후 승인된 통합을 수행한다.
6. 병합 후 브랜치 삭제는 별도 승인·팀 정책에 따른다.

`docs:`, `feat:`, `fix:`, `test:`, `chore:` 접두사를 커밋 목적에 맞게 사용한다.
문서 기준선은 실제 앱 릴리스 태그로 표시하지 않는다.
main 보호·필수 리뷰·CI 설정은 운영 권장 규칙이며 이번에 GitHub 설정을 변경했다는 의미는 아니다.

## Git Flow로 전환하는 경우

여러 배포 버전을 병행 유지하거나 독립된 릴리스 안정화 기간이 필요하면 D-10을 갱신한다.
그 경우 feature/bugfix는 develop에서 분기·통합하고 release는 develop에서 분기하여
main과 develop에 반영한다. hotfix는 운영 기준에서 분기하여 main 및 진행 중 develop/release에도 반영한다.
현재 GitHub Flow와 이 통합 대상을 혼용하지 않는다.

참고: [Git Flow 원저자의 모델 및 적용 맥락](https://nvie.com/posts/a-successful-git-branching-model/).

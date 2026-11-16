# jaeview/workflow-templates

Mac mini Apple Silicon (`[self-hosted, macOS, ARM64]`) 자체 호스팅 러너를 위한 `jaeview/jaeview-web` 및 `jaeview/workspace-tools` GitHub Actions 워크플로 모음입니다.

- **`.github/workflows/`**: 다른 저장소에서 `uses: jaeview/workflow-templates/.github/workflows/<파일명>@main`으로 바로 호출할 수 있는 **Reusable Workflows (`workflow_call`)** 및 수동 실행(`workflow_dispatch`) 정의 (`workflow-templates` 저장소 자체 푸시에서는 불필요하게 실행되지 않도록 `push`/`pull_request` 트리거를 두지 않습니다)
- **`workflow-templates/`**: 각 저장소(`.github/workflows/`)에 복사해 넣거나 GitHub Starter Workflow로 사용할 수 있는 **Standalone / Caller 템플릿 (`.yml` + `.properties.json`)**

---

## 1. 러너 환경 및 핵심 설계 (`[self-hosted, macOS, ARM64]`)

| 항목 | 적용 내용 |
| --- | --- |
| **러너 라벨** | `runs-on: [self-hosted, macOS, ARM64]` |
| **보호된 Docker Context 차단** | 보존 데이터(Supabase 백업 등)가 있는 `colima-jaeview` 컨텍스트는 실행을 거부하며, `colima-jaeview-release`를 기본 2GiB `colima`/`default` 컨텍스트보다 먼저 확인·기동(24GB 이상 호스트 12GB, 그 외 8GB/6GB)하고 `io.containerd.snapshotter`가 켜져 있으면 `overlay2`로 자동 복구합니다. |
| **Colima VM 바인드 마운트 경로** | macOS 기본 `/var/folders`는 Colima VM과 공유되지 않으므로, 컨테이너에 마운트되는 임시 경로(`TMPDIR`, `ORIGIN_CHECK_TMP_ROOT`)를 `$HOME/.cache/jaeview-ci-tmp/<run_id>` 아래로 지정하며 2일 이상 지난 임시 폴더는 자동 정리합니다. |
| **Apple Silicon 네이티브 DB/API 검사** | `fresh-schema.sh` 실행 시 PostGIS(`postgis/postgis:17-3.5`)는 `linux/amd64`(Rosetta)로, API 컨테이너는 `JAEVIEW_SCHEMA_API_PLATFORM=linux/arm64` 네이티브로 실행하여 에뮬레이션 시 `esbuild` 충돌을 방지합니다. |
| **macOS 호환 Postgres 17 서비스** | Linux 전용인 GitHub Actions `services:` 대신 `operations-checks.yml`에서 격리된 임시 `postgres:17` 컨테이너(`127.0.0.1::<ephemeral>:5432`, `trap` 자동 정리)를 띄웁니다. |
| **macOS ARM64 Trivy & SHA-256 검증** | `web-ecr-publish.yml`에서 Darwin ARM64 감지 시 `trivy_0.76.0_macOS-ARM64.tar.gz`를 내려받고 Node `crypto`로 공식 `checksums.txt` SHA-256을 대조합니다. |
| **`checks.mts` 파서 100% 호환** | `selfhost-migration-checks.yml`은 `jaeview-web/scripts/deploy/checks.mts`의 `parseJob` 문법 및 `control.test.mjs`의 `paths-ignore` 검증을 그대로 통과합니다. |

---

## 2. 수록된 워크플로 목록

### A. `@jaeview/workspace-tools`
| Reusable Workflow (`.github/workflows/`) | Starter / Standalone (`workflow-templates/`) | 설명 |
| --- | --- | --- |
| `workspace-tools-ci.yml` | `workspace-tools-ci.yml`, `workspace-tools-ci-caller.yml` | Node `26.12.1` + `pnpm@12.16.1` 기반 `pnpm install --frozen-lockfile`, `pnpm run typecheck`, `pnpm test` 실행 |
| `workspace-tools-slack-verify.yml` | `workspace-tools-slack-verify.yml` | 외부 메시지 발송 없이 설정 형식(`slack:check`) 또는 봇 인증·채널 접근(`slack:verify`)만 읽기 전용으로 확인 |

### B. `jaeview/jaeview-web` (CI 검증 + 이미지 빌드/게시 + 원격 배포/운영)
| Reusable Workflow (`.github/workflows/`) | Drop-in Template (`workflow-templates/`) | 설명 |
| --- | --- | --- |
| `jaeview-web-selfhost-checks.yml` | `selfhost-migration-checks.yml` | 변경 범위 판정(`scope.mjs`), 코드/포맷 검사, CI 발행본 복원, Next 운영 빌드·Playwright 브라우저 테스트, API/서버 단위 테스트, `fresh-schema.sh`, 전체 스택 리허설, 운영 웹·프록시 컨테이너 검증 |
| `jaeview-web-operations-checks.yml` | `operations-checks.yml` | `deploy/operations` 타입·단위 테스트(격리 Postgres 17 컨테이너), `run-host.sh` 문법 검사, `data-analysis` MCP 스크립트 테스트 |
| `jaeview-web-develop-release.yml` | `develop-release.yml` | `develop` 브랜치 전체 검증 후 Apple Silicon 네이티브 `linux/arm64` 이미지 4종 빌드, 격리 smoke 3종 + Python 자동 배포 테스트(macOS venv `cryptography`), GHCR 비공개 푸시 |
| `jaeview-web-develop-apply.yml` | `develop-apply.yml` | GitHub OIDC 단기 토큰으로 `https://dev.jaeview.com/_deploy/v1` 컨트롤러에 배포 요청 및 영수증 아티팩트 보관 |
| `jaeview-web-api-ecr-publish.yml` | `api-ecr-publish.yml` | `main` 커밋 CI 통과 확인 후 네이티브 `linux/arm64` OCI API 이미지 빌드, `--network none` 격리 smoke, AWS OIDC 인증 및 ECR 게시·스캔 기록 |
| `jaeview-web-web-ecr-publish.yml` | `web-ecr-publish.yml` | 검토된 구역 발행본(`publication.tar.gz`) 검증, `linux/amd64` 웹·프록시 이미지 빌드, 격리 compose/origin smoke, macOS ARM64 Trivy OS 취약점 스캔, AWS ECR 게시 및 스캔 게이트 |
| `jaeview-web-web-ecr-auth-check.yml` | `web-ecr-auth-check.yml` | GitHub OIDC 및 AWS ECR 읽기 권한(`jaeview/web`, `jaeview/web-proxy`) 진단 |
| `jaeview-web-remote-deploy.yml` | `remote-deploy.yml` | AWS Web / OCI API 대상 `status`, `deploy`, `rollback` SSM 원격 제어 및 영수증 아티팩트 보관 |
| `jaeview-web-remote-operations.yml` | `remote-operations.yml` | 발신 협업자 쓰기 권한 확인 후 AWS SSM을 통한 암호화된 원격 운영 명령(`read`/`write`) 실행 |

---

## 3. 적용 방법

### 방법 1: Reusable Workflow로 호출 (`workflow_call`)

#### `jaeview/workspace-tools` (`.github/workflows/ci.yml`)
```yaml
name: Workspace tools CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  verify:
    uses: jaeview/workflow-templates/.github/workflows/workspace-tools-ci.yml@main
```

#### `jaeview/jaeview-web` (`.github/workflows/operations-checks.yml` 예시)
```yaml
name: Remote operations checks
on:
  pull_request:
    paths:
      - 'deploy/operations/**'
      - 'scripts/operations/**'
      - '.agents/skills/data-analysis/**'
      - '.agents/skills/jaeview-ops/**'
      - '.github/workflows/*operations*.yml'
      - 'package.json'
      - 'pnpm-lock.yaml'
      - 'pnpm-workspace.yaml'
      - '.node-version'
  push:
    branches: [main]
    paths:
      - 'deploy/operations/**'
      - 'scripts/operations/**'
      - '.agents/skills/data-analysis/**'
      - '.agents/skills/jaeview-ops/**'
      - '.github/workflows/*operations*.yml'
      - 'package.json'
      - 'pnpm-lock.yaml'
      - 'pnpm-workspace.yaml'
      - '.node-version'

permissions:
  contents: read

jobs:
  verify:
    uses: jaeview/workflow-templates/.github/workflows/jaeview-web-operations-checks.yml@main
```

### 방법 2: Drop-in Standalone 템플릿 복사 (`workflow-templates/`)
- `jaeview-web`의 `scripts/deploy/checks.mts`는 로컬에서 `.github/workflows/selfhost-migration-checks.yml`의 `verify` 잡 단계를 직접 파싱합니다.
- 따라서 `jaeview-web`에 직접 덮어쓸 때는 `workflow-templates/*.yml` 파일들을 `jaeview-web/.github/workflows/`에 그대로 복사하여 사용할 수 있습니다.

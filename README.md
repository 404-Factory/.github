# 404-Factory / .github

404-Factory organization의 공용 CI 워크플로우 및 액션 저장소입니다.  
각 서비스 repo는 이 저장소의 reusable workflow를 호출해 CI를 구성합니다.

---

## 디렉토리 구조

```
.github/
└── workflows/
    ├── reusable-build.yml    # Docker 빌드 + 이미지 푸시
    ├── reusable-deploy.yml   # EKS kubectl apply 배포
    └── caller-ci-example.yml # 서비스 repo용 호출 예시 (복사해서 사용)
```

---

## Workflows

### `reusable-build.yml` — Docker 빌드 & 푸시

| 단계 | 내용 |
|---|---|
| Build | Docker 이미지 빌드 (로컬 로드) |
| Push | 지정 브랜치 push 시에만 레지스트리로 푸시 |

**inputs**

| 이름 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `service-name` | | repo 이름 | 서비스 이름 (이미지 태그에 사용). 생략 시 repo 이름 자동 사용. |
| `docker-registry` | | `ghcr.io` | 레지스트리 호스트 |
| `use-ecr` | | `false` | AWS ECR 사용 여부. `true` 시 AWS 자격증명으로 ECR 로그인. |
| `ecr-registry` | | | ECR 레지스트리 URL (`<account-id>.dkr.ecr.<region>.amazonaws.com`). `use-ecr: true` 시 필요. |
| `aws-region` | | `ap-northeast-2` | ECR 사용 시 AWS 리전 |
| `build-context` | | `.` | Dockerfile 빌드 컨텍스트 경로 |
| `dockerfile` | | `Dockerfile` | Dockerfile 경로 |
| `platforms` | | `linux/amd64` | 빌드 플랫폼 (멀티 아키: `linux/amd64,linux/arm64`) |
| `push-on-ref` | | `refs/heads/main` | 이미지를 푸시할 브랜치 ref |
| `java-version` | | | JVM 서비스용 Java 버전 (예: `17`). 설정 시 pre-build 전 Java 셋업. |
| `pre-build-command` | | | Docker 빌드 전 실행할 명령어 (예: `./gradlew build -x test --no-daemon`). |

**secrets**

| 이름 | 설명 |
|---|---|
| `aws-access-key-id` | AWS Access Key ID. `use-ecr: true` 시 필요. |
| `aws-secret-access-key` | AWS Secret Access Key. `use-ecr: true` 시 필요. |
| `registry-username` | 레지스트리 사용자명. GHCR 등 ECR 외 레지스트리 사용 시 필요. |
| `registry-password` | 레지스트리 비밀번호/토큰. GHCR 등 ECR 외 레지스트리 사용 시 필요. |
| `gpr-token` | GitHub Packages 읽기용 PAT (`read:packages`). pre-build-command에서 다른 레포 패키지 접근 시 필요. |

**outputs**

| 이름 | 설명 |
|---|---|
| `image` | 이미지 레퍼런스 (`registry/name`) |
| `tag` | 빌드에 사용된 SHA 태그 |
| `digest` | 푸시된 이미지 digest (`sha256:...`) |

---

## 각 서비스 repo에서 사용하는 방법

`.github/workflows/caller-ci-example.yml`을 서비스 repo의 `.github/workflows/ci.yml`로 복사한 뒤 아래 항목만 수정합니다.

**GHCR 사용 (기본)**

```yaml
jobs:
  build:
    uses: 404-Factory/.github/.github/workflows/reusable-build.yml@main
    permissions:
      contents: read
      packages: write
    with:
      docker-registry: ghcr.io/404-Factory
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

**AWS ECR 사용**

```yaml
jobs:
  build:
    uses: 404-Factory/.github/.github/workflows/reusable-build.yml@main
    permissions:
      contents: read
      packages: write
    with:
      use-ecr: true
      aws-region: ap-northeast-2
      ecr-registry: ${{ vars.ECR_REGISTRY }}
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**JVM 서비스 (Gradle pre-build 포함)**

```yaml
jobs:
  build:
    uses: 404-Factory/.github/.github/workflows/reusable-build.yml@main
    permissions:
      contents: read
      packages: write
    with:
      use-ecr: true
      aws-region: ap-northeast-2
      ecr-registry: ${{ vars.ECR_REGISTRY }}
      java-version: "17"
      pre-build-command: "./gradlew build -x test --no-daemon"
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      gpr-token: ${{ secrets.GPR_TOKEN }}
```

> `service-name`은 생략 시 repo 이름을 자동으로 사용합니다.  
> `push-on-ref` 기본값은 `refs/heads/main`입니다. 다른 브랜치로 푸시하려면 명시하세요.

---

## Secrets / Variables

| 이름 | 종류 | 등록 위치 | 설명 |
|---|---|---|---|
| `AWS_ACCESS_KEY_ID` | Secret | 각 서비스 repo | AWS Access Key ID (`use-ecr: true` 설정 시 필요) |
| `AWS_SECRET_ACCESS_KEY` | Secret | 각 서비스 repo | AWS Secret Access Key (`use-ecr: true` 설정 시 필요) |
| `GPR_TOKEN` | Secret | 각 서비스 repo | GitHub Packages 읽기용 PAT (`read:packages`). Gradle 플러그인 등 다른 레포 패키지 접근 시 필요. |
| `ECR_REGISTRY` | Variable | 각 서비스 repo | ECR 레지스트리 URL (`use-ecr: true` 설정 시 필요) |
| `PLATFORM` | Variable | 조직 또는 각 서비스 repo | 빌드 대상 플랫폼 (예: `linux/arm64`, `linux/amd64`). EKS 노드 아키텍처와 맞춰야 함. 미설정 시 `reusable-build.yml` 기본값 사용. |
| `GITHUB_TOKEN` | 자동 제공 | — | GHCR 이미지 푸시에 사용 |

# 404-Factory / .github

404-Factory organization의 공용 CI 워크플로우 및 액션 저장소입니다.  
각 서비스 repo는 이 저장소의 reusable workflow를 호출해 CI를 구성합니다.

---

## 디렉토리 구조

```
.github/
├── actions/
│   └── setup-env/
│       └── action.yml        # 언어별 런타임 셋업 composite action
└── workflows/
    ├── reusable-build.yml    # Docker 빌드 + 보안 스캔 + 이미지 서명
    ├── reusable-test.yml     # 테스트 + 커버리지 측정
    └── caller-ci-example.yml # 서비스 repo용 호출 예시 (복사해서 사용)
```

---

## Workflows

### `reusable-build.yml` — Docker 빌드 & 스캔

| 단계 | 내용 |
|---|---|
| Build | Docker 이미지 빌드 (로컬 로드) |
| SBOM | CycloneDX 형식 소프트웨어 명세서 생성 |
| Scan | Trivy로 HIGH/CRITICAL 취약점 게이트 |
| Push | main 브랜치 push 시에만 레지스트리로 푸시 |
| Sign | cosign keyless 서명 + SBOM 어테스테이션 |

**inputs**

| 이름 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `service-name` | | repo 이름 | 서비스 이름 (이미지 태그에 사용). 생략 시 repo 이름 자동 사용. |
| `docker-registry` | | `ghcr.io` | 레지스트리 호스트 (ECR 사용 시 `ecr-registry` secret 우선 적용) |
| `use-ecr` | | `false` | AWS ECR 사용 여부. `true` 시 AWS 자격증명으로 ECR 로그인. |
| `aws-region` | | `ap-northeast-2` | ECR 사용 시 AWS 리전 |
| `build-context` | | `.` | Dockerfile 빌드 컨텍스트 경로 |
| `dockerfile` | | `Dockerfile` | Dockerfile 경로 |
| `platforms` | | `linux/amd64` | 빌드 플랫폼 (멀티 아키: `linux/amd64,linux/arm64`) |
| `push-on-ref` | | `refs/heads/main` | 이미지를 푸시할 브랜치 ref |
| `severity` | | `HIGH,CRITICAL` | Trivy 스캔 실패 기준 심각도 |
| `sign-image` | | `true` | cosign 서명 여부 |

**secrets**

| 이름 | 설명 |
|---|---|
| `ecr-registry` | ECR 레지스트리 URL (`<account-id>.dkr.ecr.<region>.amazonaws.com`). `use-ecr: true` 시 필요. |
| `aws-access-key-id` | AWS Access Key ID. `use-ecr: true` 시 필요. |
| `aws-secret-access-key` | AWS Secret Access Key. `use-ecr: true` 시 필요. |
| `registry-username` | 레지스트리 사용자명. GHCR 등 ECR 외 레지스트리 사용 시 필요. |
| `registry-password` | 레지스트리 비밀번호/토큰. GHCR 등 ECR 외 레지스트리 사용 시 필요. |

**outputs**

| 이름 | 설명 |
|---|---|
| `image` | 이미지 레퍼런스 (`registry/name`) |
| `tag` | 빌드에 사용된 SHA 태그 |
| `digest` | 푸시된 이미지 digest (`sha256:...`) |

---

### `reusable-test.yml` — 테스트 & 커버리지

지원 언어: `node` \| `python` \| `go` \| `java`

| 단계 | 내용 |
|---|---|
| Setup | 언어별 런타임 및 패키지 캐시 설정 |
| Lint | 언어별 린터 실행 |
| Test | 테스트 실행 + 커버리지 측정 |
| Gate | 커버리지 임계값 미달 시 실패 |
| SonarCloud | 정적 분석 (토큰 없으면 스킵) |
| PR Comment | PR에 테스트 결과 코멘트 |

**inputs**

| 이름 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `service-name` | | repo 이름 | 서비스 이름. 생략 시 repo 이름 자동 사용. |
| `language` | ✅ | — | `node` \| `python` \| `go` \| `java` |
| `node-version` | | `20.x` | Node.js 버전 |
| `python-version` | | `3.11` | Python 버전 |
| `go-version` | | `1.22` | Go 버전 |
| `java-version` | | `21` | Java 버전 |
| `java-distribution` | | `temurin` | JDK 배포판 |
| `test-command` | | 언어별 기본값 | 테스트 명령어 override |
| `coverage-threshold` | | `80` | 최소 커버리지 % |
| `run-sonar` | | `false` | SonarCloud 스캔 실행 여부. `true` 시 `sonar-token` secret 필요. |
| `working-directory` | | `.` | 소스 루트 경로 |

---

## 각 서비스 repo에서 사용하는 방법

`.github/workflows/caller-ci-example.yml`을 서비스 repo의 `.github/workflows/ci.yml`로 복사한 뒤 아래 항목만 수정합니다.

**GHCR 사용 (기본)**

```yaml
jobs:
  test:
    uses: 404-Factory/.github/.github/workflows/reusable-test.yml@main
    with:
      language: node               # 사용 언어
      node-version: "20.x"         # 해당 언어의 버전 input만 남기기
      run-sonar: true              # SonarCloud 사용 시 true (SONAR_TOKEN secret 필요)
    secrets:
      sonar-token: ${{ secrets.SONAR_TOKEN }}

  build:
    needs: test
    uses: 404-Factory/.github/.github/workflows/reusable-build.yml@main
    with:
      docker-registry: ghcr.io/404-Factory
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

**AWS ECR 사용**

```yaml
jobs:
  test:
    uses: 404-Factory/.github/.github/workflows/reusable-test.yml@main
    with:
      language: node
      node-version: "20.x"
    secrets:
      sonar-token: ${{ secrets.SONAR_TOKEN }}

  build:
    needs: test
    uses: 404-Factory/.github/.github/workflows/reusable-build.yml@main
    with:
      use-ecr: true
      aws-region: ap-northeast-2
    secrets:
      ecr-registry: ${{ secrets.ECR_REGISTRY }}
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**언어별 버전 input 대응표**

| language | 사용할 input |
|---|---|
| `node` | `node-version` |
| `python` | `python-version` |
| `go` | `go-version` |
| `java` | `java-version` + `java-distribution` |

> 기본값과 동일한 input은 생략 가능합니다.  
> `service-name`은 생략 시 repo 이름을 자동으로 사용합니다.

---

## Secrets

| Secret | 등록 위치 | 설명 |
|---|---|---|
| `SONAR_TOKEN` | 각 서비스 repo | SonarCloud 분석 토큰 (`run-sonar: true` 설정 시 필요) |
| `ECR_REGISTRY` | 각 서비스 repo | ECR 레지스트리 URL (`use-ecr: true` 설정 시 필요) |
| `AWS_ACCESS_KEY_ID` | 각 서비스 repo | AWS Access Key ID (`use-ecr: true` 설정 시 필요) |
| `AWS_SECRET_ACCESS_KEY` | 각 서비스 repo | AWS Secret Access Key (`use-ecr: true` 설정 시 필요) |
| `GITHUB_TOKEN` | 자동 제공 | GHCR 이미지 푸시에 사용 |

# 02 — GitHub Actions: workflow 구조와 OIDC

> **공식 근거:** [GitHub Actions docs](https://docs.github.com/en/actions), [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions), [OIDC in GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
>
> **선행:** [01-ci-fundamentals.md](01-ci-fundamentals.md)

---

## 🎯 한 문장

> **Workflow = *이벤트로 trigger되는 job들의 묶음*. 각 job은 *runner 머신 위에서 step들을 순차 실행*. *OIDC*로 *정적 secret 없이* AWS/GCP/Azure에 *단기 token 발급받아 인증* — 현대 CI의 핵심 보안 패턴.**

---

## 1. 구조 — Workflow / Job / Step

```yaml
name: Build & Deploy           # ← Workflow

on:                            # ← Trigger
  push:
    branches: [main]
  pull_request:

permissions:                   # ← OIDC 등 권한
  id-token: write
  contents: read

jobs:
  test:                        # ← Job 1
    runs-on: ubuntu-latest     #    Runner
    steps:
      - uses: actions/checkout@v4   # ← Step (action 사용)
      - name: Run tests        # ← Step (shell)
        run: npm test
  
  build:                       # ← Job 2 (test 후)
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with: { ... }
```

### Workflow
- `.github/workflows/*.yml` 파일 하나가 하나의 워크플로
- 한 repo에 여러 워크플로 가능 (build, deploy, security 등으로 분리)

### Job
- *독립된 runner 머신* (또는 컨테이너)에서 실행
- 기본은 *모든 job 병렬*. `needs:` 로 의존 명시.
- 각 job은 *fresh environment* — 다른 job의 파일 직접 공유 X (`actions/upload-artifact`, `actions/cache` 사용)

### Step
- Job 안의 *순차 실행 단위*
- `uses: org/action@v1` — *재사용 action 호출*
- `run: ...` — shell 명령

### Runner
- **GitHub-hosted** — GitHub이 제공 (Ubuntu/Windows/macOS, *VM per job*). 무료/유료.
- **Self-hosted** — 직접 운영. 보안 주의 ([01](01-ci-fundamentals.md) Section 8 참조).
- **larger runners** — 더 큰 CPU/RAM (GitHub Enterprise 기능)

---

## 2. Trigger 종류

| `on:` 이벤트 | 언제 |
|---|---|
| `push` | branch에 push |
| `pull_request` | PR 생성·갱신·머지 (*fork은 secret 비공개*) |
| `pull_request_target` | PR 트리거 + *base branch 컨텍스트* (*secret 공개 — 매우 위험*) |
| `schedule` | cron (`0 2 * * *`) |
| `workflow_dispatch` | UI/API에서 수동 실행 |
| `workflow_call` | *다른 workflow에서 호출* (reusable workflow) |
| `repository_dispatch` | 외부 webhook |
| `release` | GitHub Release 생성 |

### `pull_request` vs `pull_request_target` — *반드시* 알아야
- `pull_request` — *PR의 코드*가 trigger 컨텍스트. *fork PR이면 secret 비공개* (안전).
- `pull_request_target` — *base branch (main)*의 컨텍스트로 실행. *fork PR이어도 secret 공개*. ← **fork의 악성 코드가 secret 탈취 가능**. 절대 *checkout PR head 코드 + secret 같이* 사용 금지.

---

## 3. Permissions — *최소 권한 원칙*

```yaml
permissions:
  contents: read           # 코드 읽기만
  id-token: write          # OIDC token 발급
  packages: write          # GHCR push
  pull-requests: write     # PR comment 쓰기
```

- *job 단위로도* 설정 가능 (`jobs.deploy.permissions: ...`)
- *기본값*: repo Settings의 default (보통 `read-all`)
- **`permissions: {}` (또는 명시적 최소)가 보안 모범**

---

## 4. Secrets — *어디서 어떻게 노출되는가*

### Secret 종류
| 위치 | 범위 |
|---|---|
| Repo secret | 한 repo |
| Org secret | 여러 repo (org 단위) |
| Environment secret | *특정 environment 사용 job만* (production 등) |

### 사용
```yaml
env:
  DB_URL: ${{ secrets.DB_URL }}
```

### 자동 마스킹
- secret 값이 *로그에 출력*되면 자동 `***`
- *완벽하진 않음* — base64 인코딩하거나 split 후 echo하면 *우회 가능*
- → **로그에 secret 일부도 출력하지 말기**

### *fork PR + pull_request* — secret 자동 비공개
- 보안 default. fork에서 mutation 못 함.
- `pull_request_target`은 *예외* — *위험 인지*

### *Environment 보호*
```yaml
jobs:
  deploy:
    environment: production    # ← Environment 지정
    steps: ...
```
- Environment에 *required reviewer*, *wait timer*, *branch 제한* 설정 가능
- *secret도 environment-scoped로 격리*

---

## 5. OIDC — *정적 키 없이 클라우드 인증* (가장 중요)

### 옛 방식 (위험)
```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```
- *정적 키*. 유출 시 *영구 위험*. 회전 부담.

### 현대 (OIDC)
GitHub Actions가 *workflow마다 단기 OIDC token 발급* → AWS STS가 *AssumeRoleWithWebIdentity*로 *15분~1시간 단기 자격증명* 부여.

### 흐름
```
1. workflow start → GitHub OIDC provider가 JWT 발급
   - subject (sub): "repo:my-org/my-repo:ref:refs/heads/main"
   - issuer (iss): "https://token.actions.githubusercontent.com"
   - 짧은 유효기간 (수 분)

2. workflow가 AWS STS에 AssumeRoleWithWebIdentity 호출
   - JWT 전달

3. AWS:
   - JWT 서명 검증 (GitHub의 공개 키)
   - IAM Role의 trust policy 확인:
     - "이 OIDC provider에서 발급된 token"
     - "subject가 우리가 신뢰하는 패턴"
   - → 통과하면 임시 자격증명 반환 (~1시간)

4. workflow가 임시 자격증명으로 AWS API 호출
```

### AWS IAM Role의 Trust Policy 예시
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
      }
    }
  }]
}
```

→ *`sub` 조건이 핵심*. 어떤 *repo, branch, env, tag*에서만 assume 허용할지 정확히 명시.

### Workflow에서 사용
```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123:role/github-actions
        aws-region: ap-northeast-2
    - run: aws s3 ls
```

→ *정적 키 0개*. GCP·Azure도 *같은 패턴*.

### 흔한 사고
- *Trust policy의 `sub` 조건이 너무 느슨* (`repo:my-org/*` 등 wildcard) → *다른 branch의 PR이 prod role assume 가능*
- → *최소한 main branch + 특정 environment*만 허용

### IRSA와 *같은 원리*
- EKS의 IRSA = Pod의 SA token → AWS STS → 임시 자격증명
- GitHub OIDC = workflow의 OIDC token → AWS STS → 임시 자격증명
- **둘 다 *OIDC IdP를 외부에 두고* AWS가 *신뢰*하는 패턴**

→ [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) 의 IRSA와 비교.

---

## 6. Reusable Workflow — *workflow 재사용*

```yaml
# .github/workflows/_reusable-build.yml
on:
  workflow_call:
    inputs:
      image-name: { required: true, type: string }
    secrets:
      registry-token: { required: true }

jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]
```

호출:
```yaml
jobs:
  build:
    uses: ./.github/workflows/_reusable-build.yml
    with:
      image-name: myapp
    secrets:
      registry-token: ${{ secrets.GHCR_TOKEN }}
```

### 장점
- *공통 빌드·배포 워크플로* 한 곳에 → 모든 서비스가 *같은 표준*
- *변경 한 번으로 전 서비스 반영*

### Composite Action vs Reusable Workflow
| | Composite Action | Reusable Workflow |
|---|---|---|
| 단위 | *step들의 묶음* | *job들의 묶음* |
| Runner | 호출 측 runner | 자기 runner |
| Secret | 호출 측 환경 | 명시적 전달 |
| 용도 | *작은 헬퍼* (예: setup tool chain) | *전체 워크플로 표준화* |

---

## 7. Matrix — *병렬 실행*

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, macos-latest]
        exclude:
          - { node: 18, os: macos-latest }
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm test
```

→ *각 조합마다 별도 job* 병렬 실행. 위 예 = (3 × 2) - 1 = *5개 job 동시*.

---

## 8. Concurrency — *동시 실행 제어*

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

- *같은 group*은 *동시에 한 워크플로만*
- `cancel-in-progress: true` — 새 trigger 시 *진행 중인 것 취소*

→ 흔한 활용: *같은 PR에 push 여러 번* → *이전 빌드 취소하고 최신만 빌드*.

---

## 9. Caching — 속도 최적화

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: ${{ runner.os }}-node-
```

- *key가 같으면 cache hit* → restore
- *restore-keys*로 *prefix match fallback*

### Docker buildx cache
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```
→ *GitHub Actions cache*를 *image layer cache*로 (별도 registry 없이).

---

## 10. 자주 헷갈리는 것 / 보안 함정

### 10-1. *fork PR + pull_request_target = secret 탈취 위험*
- 절대 *fork PR의 코드 checkout + secret 노출* 같이 하지 말 것
- *공식 가이드*: `pull_request_target` 쓰면 *PR head 코드 안 checkout*, *PR base 코드만*

### 10-2. *Third-party action을 commit SHA로 pin*
- `uses: someone/action@v1` — *tag는 mutable*. 누가 v1 tag를 *악성 코드로 바꾸면* 우리도 영향.
- `uses: someone/action@abc123...` (*commit SHA*) — *불변*. 추천.
- Dependabot이 *자동 갱신* 도와줌.

### 10-3. *roll your own secret 마스킹*
- secret을 *base64 인코딩해서 echo* → *마스킹 우회됨*
- *script가 secret을 *변형해서 출력*하면 노출*
- → 절대 *처리된 secret 일부도 print 금지*

### 10-4. *workflow를 너무 PR로 trigger*
- 모든 PR이 build·test → CI 비용·queue 폭증
- *paths filter* 활용 (`paths: ['src/**']`)

### 10-5. *OIDC sub 조건 너무 느슨*
- `repo:my-org/*` → *어느 repo도* assume 가능
- *최소 main branch + production environment* 명시

### 10-6. *workflow가 너무 길어짐* (1000줄)
- *reusable workflow*로 분해
- *composite action*으로 step 묶음 분해

---

## 🎤 면접 빈출 Q&A

### Q1. GitHub Actions에서 AWS 인증을 어떻게? 정적 키 안 쓰는 방법은?
> **OIDC**. workflow 시작 시 GitHub OIDC provider가 *단기 JWT 발급* → workflow가 *AWS STS AssumeRoleWithWebIdentity* 호출 → AWS가 JWT 서명 검증 + IAM Role의 *trust policy의 `sub` 조건 매치* → *임시 자격증명 (~1시간)* 반환. *정적 AccessKey 0개*. `aws-actions/configure-aws-credentials@v4` action 사용. *trust policy의 sub 조건은 *최소한 repo + branch + (선택) environment* 명시*.

### Q2. `pull_request` vs `pull_request_target` 차이? 왜 위험한가?
> `pull_request` — *PR 코드 컨텍스트*. *fork PR이면 secret 비공개* (안전). `pull_request_target` — *base branch 컨텍스트*. *fork PR이어도 secret 공개* + *PR head 코드 실행 가능* → *fork의 악성 코드가 secret 탈취*. → *fork PR의 코드 checkout + secret 사용* 같이 하면 안 됨. 일반적인 PR check는 *항상 `pull_request`* 사용.

### Q3. Reusable workflow와 composite action 차이?
> *Composite action* = *step들의 묶음* (작은 헬퍼). 호출 측 runner에서 실행. *Reusable workflow* = *job들의 묶음* (전체 워크플로 표준화). 자기 runner에서 실행, *secret 명시적 전달*. → 작은 단위 재사용은 composite, *전 조직 표준 build/deploy 워크플로*는 reusable.

### Q4. Third-party action을 안전하게 사용?
> *commit SHA로 pin* — tag는 mutable이라 *누가 tag를 악성 코드로 바꾸면 영향*. SHA는 *content hash, 불변*. Dependabot이 *자동 갱신 PR* 제출. 추가로 *action을 review하고 사용*, *불필요한 권한 안 줌* (`permissions:` 명시적 최소).

### Q5. Workflow가 *느릴 때* 최적화 방법?
> (1) *Cache 적극* — 의존성 (`actions/cache`), Docker layer (`type=gha`), test 결과. (2) *Matrix로 병렬화*. (3) *concurrency로 중복 빌드 취소*. (4) *paths filter*로 *영향 없는 변경엔 skip*. (5) *job 의존성 최소화* — needs 줄이고 *DAG 평탄화*. (6) *larger runner* (GitHub Enterprise).

### Q6. Secret을 *git에 안 박는데* 그래도 노출 위험?
> 여러 시나리오: (1) *workflow log에 secret 출력* — base64 등 변형은 *마스킹 우회*. (2) *fork PR + `pull_request_target`* 으로 secret 탈취. (3) *Third-party action이 secret 받아서 외부 전송*. (4) *self-hosted runner에 secret 남음*. → *마스킹 신뢰하지 말고 출력 자체 금지*, *fork PR 보호*, *action SHA pin*, *runner ephemeral*.

---

## 🔗 Cross-reference

- **CI 일반 원칙** → [01-ci-fundamentals.md](01-ci-fundamentals.md)
- **ArgoCD GitOps (CD 단계)** → [03-argocd.md](03-argocd.md)
- **IRSA — GitHub OIDC와 같은 원리** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), `aws-basics/` (예정)
- **Image build (Buildx, BuildKit)** → [05-build-and-registry.md](05-build-and-registry.md), [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md)
- **Branch protection + CI gate** → [01](01-ci-fundamentals.md) Section 9

---

## 📝 3줄 요약

1. Workflow = trigger → jobs (parallel) → steps (sequential, runner 머신). *minimal permissions* 명시, *concurrency*로 중복 취소, *cache*로 속도.
2. **OIDC**가 현대 보안의 핵심 — *정적 키 0개*, GitHub JWT → AWS STS → 임시 자격. *trust policy `sub` 조건이 보안 강도 결정*.
3. *`pull_request_target` + fork checkout + secret* 조합은 *치명적 위험*. action은 *SHA pin*. *secret 출력 금지* (마스킹 우회 가능).

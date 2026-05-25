# 05 — Workflow & Best Practices: plan/apply lifecycle · drift · CI · Terragrunt

> **공식 근거:** [Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure), [Recommended Practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices)

---

## 🎯 한 문장

> **운영의 핵심: *plan-review-apply 사이클*, *drift detection*, *immutable 변경 패턴*, *PR 단위 변경*, *secret 외부 관리*, *CI 자동화 (Atlantis/Spacelift)*. *Terragrunt*로 멀티 환경 DRY. **Ansible vs Terraform 역할 분담 명확*** — infra vs config.**

---

## 1. Plan / Apply Lifecycle — *정석 흐름*

### 정석
```
1. Code change (PR)
2. terraform fmt + validate + tflint + checkov  (lint)
3. terraform plan -out=tfplan        (서버 측 plan)
4. PR review (사람·자동) — *plan 출력 검토*
5. PR merge
6. terraform apply tfplan            (저장된 plan 적용)
```

### `-out` 의 의미
- `terraform plan -out=tfplan` — plan 결과를 *파일로 저장*
- `terraform apply tfplan` — *그 plan 그대로* 적용 — *plan과 apply 사이 변경 없음 보장*
- 안 쓰면 — `apply`가 *다시 plan 후 적용* → 그 사이 *cloud 상태 변경 시 다른 결과*

→ 운영에선 *항상 `-out` + 그 plan 적용*.

### `-auto-approve` (CI에서)
- *수동 confirm 건너뜀*
- *PR review가 *실질적 approval*이라는 전제

---

## 2. PR 단위 변경 + Plan output review

### CI 흐름 (GitHub Actions 예)
```yaml
on:
  pull_request:
    paths: ['infra/**/*.tf']

jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      pull-requests: write
    steps:
    - uses: actions/checkout@v4
    
    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.5.7
    
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::...:role/terraform-plan
        aws-region: ap-northeast-2
    
    - run: terraform init
      working-directory: infra/
    
    - run: terraform validate
      working-directory: infra/
    
    - id: plan
      run: terraform plan -out=tfplan -no-color > plan.txt
      working-directory: infra/
    
    - uses: actions/github-script@v7
      with:
        script: |
          const output = require('fs').readFileSync('infra/plan.txt', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '```\n' + output.substring(0, 60000) + '\n```'
          });
    
    - uses: actions/upload-artifact@v4
      with:
        name: tfplan
        path: infra/tfplan
```

→ *PR comment에 plan 결과 자동* — *reviewer가 변경 사항 즉시 확인*.

---

## 3. Drift Detection — *수동 변경 감지*

### Drift란
- *Terraform 외부에서 cloud 자원 변경* (console 클릭, AWS CLI 등)
- HCL과 *실제 cloud 상태 불일치*
- 다음 plan에서 *변경 발견 → 의도 X 일 수도*

### 감지
```bash
terraform plan -detailed-exitcode
# exit code:
# 0 = no changes
# 1 = error
# 2 = changes detected (drift!)
```

CI에서 정기 실행 (cron):
```yaml
on:
  schedule:
  - cron: '0 9 * * *'    # 매일 09:00

jobs:
  drift:
    steps:
    - terraform plan -detailed-exitcode
    - if drift detected:
      - notify Slack
      - create Jira ticket
```

### Drift 대응
- *의도된 변경*이면 → HCL에 반영
- *의도 없음*이면 → `terraform apply`로 *원상복귀* (revert)
- *영구 무시*면 → `lifecycle.ignore_changes` ([01 참조](01-providers-and-resources.md))

---

## 4. 안전한 변경 패턴

### 1. *create_before_destroy* — Downtime 회피
```hcl
resource "aws_launch_template" "app" {
  ...
  lifecycle {
    create_before_destroy = true
  }
}
```

→ 옛 자원 죽이기 *전에 새 자원 생성*. ASG·LB target 등에 핵심.

### 2. *Blue/Green replacement*
```hcl
resource "aws_lb_target_group" "blue" { ... }
resource "aws_lb_target_group" "green" { ... }

# weight 변경으로 트래픽 점진 이동
resource "aws_lb_listener_rule" "main" {
  action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.blue.arn
        weight = var.blue_weight     # 100 → 50 → 0
      }
      target_group {
        arn    = aws_lb_target_group.green.arn
        weight = var.green_weight    # 0 → 50 → 100
      }
    }
  }
}
```

### 3. *Immutable infra*
- *변경 = 새 자원 생성 + 옛 자원 destroy*
- AMI 변경, instance_type 변경 등은 *replace*
- → *재현 가능, drift 적음*

### 4. *Plan을 *항상 read*
- "*그냥 apply*" 절대 안 됨
- *plan output 한 줄씩 검토*
- `- destroy` 가 *의도 맞나*?
- `~` update가 *in-place인지 replace인지*?

---

## 5. Secret 관리 (다시 정리)

### 절대 금지
- HCL에 hardcode
- tfvars에 평문 (git commit)
- environment variable 으로 받아 *그대로 hardcode*

### 정석
- **외부 secret manager** (Vault, AWS Secrets Manager, GCP Secret Manager)
- Terraform이 *읽기만* (data source)
- 또는 *Random 생성 → secret manager에 저장 → 그 값으로 자원 생성*

### State 자체 보호
- *S3 bucket encryption (KMS)*
- *S3 RBAC 빡세게*
- *partial state* — 민감한 자원은 *별도 state*

→ 자세히 [04-variables-and-outputs.md](04-variables-and-outputs.md) Section 8.

---

## 6. Terragrunt — *멀티 환경 DRY*

### 문제
```
envs/
├── dev/
│   ├── main.tf      ← module call, 거의 동일
│   └── ...
├── stg/
│   ├── main.tf      ← 거의 동일 (값만 다름)
│   └── ...
└── prod/
    ├── main.tf      ← 거의 동일
    └── ...
```

→ *코드 중복 90%*. 변경 시 *모든 환경 동시 수정*.

### Terragrunt 해결
```hcl
# terragrunt.hcl (전 환경 공통)
remote_state {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "tf-locks"
  }
}

inputs = {
  region = "ap-northeast-2"
  common_tags = {
    ManagedBy = "terragrunt"
  }
}
```

```hcl
# envs/prod/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/my-org/modules.git//eks?ref=v1.0.0"
}

inputs = {
  environment        = "prod"
  cluster_name       = "prod-eks"
  node_instance_type = "m5.large"
  node_count_min     = 3
  node_count_max     = 30
}
```

→ *환경별 디렉터리 = inputs만*. *backend·module source 공통*. 매우 DRY.

### `run-all` — 여러 환경 동시
```bash
terragrunt run-all plan     # 모든 envs에 plan
terragrunt run-all apply
```

→ 단점: *학습 곡선*, *추가 도구 dependency*. 작은 환경엔 *오버킬*.

---

## 7. Atlantis / Spacelift — *PR-driven 자동화*

### Atlantis
- *오픈소스 self-hosted*
- PR 코멘트 `atlantis plan` → 자동 plan + PR comment에 결과
- `atlantis apply` 코멘트 → apply
- *권한·승인·workflow* 통합

### Spacelift / Terraform Cloud / env0
- *SaaS*
- *Drift detection, policy as code, RBAC, audit log, cost estimation* 통합
- *조직·팀 규모 큼*에 적합

### 기본 흐름 (Atlantis)
```
1. PR open
2. Atlantis detects → auto plan (각 디렉터리)
3. Atlantis posts plan to PR comment
4. Reviewer approves PR + comment `atlantis apply`
5. Atlantis applies + comments result
6. Auto merge (옵션)
```

---

## 8. Testing — Terratest / Terraform native test

### Terraform native test (1.6+)
```hcl
# tests/vpc.tftest.hcl
run "vpc_creation" {
  command = plan
  
  variables {
    cidr_block = "10.0.0.0/16"
  }
  
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR mismatch"
  }
}

run "subnet_count" {
  command = apply
  
  variables {
    availability_zones = ["a", "b", "c"]
  }
  
  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Expected 3 subnets"
  }
}
```

```bash
terraform test
```

### Terratest (Go)
- *Go 기반*. 더 강력하고 유연.
- *실제 cloud에 apply → 검증 → destroy* (end-to-end)
- *AWS·GCP·Azure·K8s helper 풍부*

```go
func TestVPCModule(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "cidr_block": "10.0.0.0/16",
        },
    }
    
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)
    
    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.Contains(t, vpcID, "vpc-")
}
```

→ *비용·시간 ↑* (실제 자원 생성). PR마다 X — *nightly* 또는 *주요 변경 시*.

---

## 9. Policy as Code — *변경 정책 자동화*

### tflint
- *Terraform native linter*
- naming, deprecated module, AWS instance type 등 검사
```bash
tflint --init
tflint
```

### checkov / tfsec
- *보안 정책 검사*
- "public S3 bucket 금지", "encryption 없는 RDS 금지" 등
```bash
checkov -d .
tfsec .
```

→ CI에 통합. *위반 시 PR 차단*.

### OPA + conftest
- *Rego 정책으로 plan 검증*
```bash
terraform show -json tfplan > plan.json
conftest test --policy policies/ plan.json
```

### Terraform Cloud Sentinel
- *HashiCorp의 정책 엔진*
- "production에 manual approval 필수", "특정 region만 허용" 등

---

## 10. *Ansible vs Terraform* — 다시 분리

### 역할 매트릭스
| | Terraform | Ansible |
|---|---|---|
| **목적** | Infra provisioning | Configuration management |
| **대상** | Cloud 자원 (EC2, VPC, RDS, ...) | 그 EC2의 OS·앱 |
| **State** | 외부 state 파일 | State 없음 (target이 진실) |
| **통신** | Cloud API | SSH |
| **선언적?** | 강 (HCL desired state) | 강 (playbook) |
| **Drift** | plan으로 감지 | 매 실행마다 검사 |
| **변경 추적** | state diff | 매번 fresh 검사 |

### 같이 쓰는 흐름
```
1. terraform apply
   → EC2·VPC·security group 생성
   → output: instance IPs

2. terraform output → Ansible inventory
   $ terraform output -json | jq ... > inventory

3. ansible-playbook -i inventory site.yml
   → nginx install, config deploy, ...
```

### 현대적 대안
- **Cloud-init / user_data** — EC2 생성 시 *부팅 script*로 간단 설정
- **Packer + AMI** — *AMI 자체*에 모든 설정 → terraform은 *그 AMI 인스턴스화*만
- **Container** — 모든 환경을 container image에 — *Ansible 불필요해짐*

→ *Container 시대에 Ansible 역할 축소*, 단 *VM 기반·legacy infra*에선 여전히 핵심.

---

## 🎤 면접 빈출 Q&A

### Q1. terraform plan 과 apply의 차이? 왜 `-out` 사용?
> **plan**: *현재 cloud 상태 read + HCL 비교 + 차이 출력*. *변경 X*. **apply**: 그 차이를 *실제 적용*. *`-out=tfplan`*: plan 결과를 파일로 저장 → `apply tfplan`이 *그 plan 그대로* 적용. 안 쓰면 apply가 다시 plan 후 적용 → *plan과 apply 사이 cloud 변경 시 다른 결과*. **운영은 항상 -out**.

### Q2. Drift detection은 어떻게?
> *Terraform 외부에서 cloud 자원 변경* 시 발생. `terraform plan -detailed-exitcode` → exit 2면 drift. CI cron으로 *정기 실행 + Slack/Jira 알림*. 대응: (1) 의도된 변경이면 HCL에 반영, (2) 의도 없음이면 apply로 revert, (3) 영구 무시면 `lifecycle.ignore_changes`.

### Q3. Terraform과 K8s reconciliation 차이?
> 둘 다 *desired state ↔ actual state*. **차이**: K8s = *영원히 reconcile* (controller가 자동), Terraform = *명시적 apply 시만* (외부 사용자 trigger). Terraform은 *drift 자동 보정 X* — apply 안 하는 동안 수동 변경 *그대로*. K8s는 controller가 *즉시 복원*.

### Q4. Ansible과 Terraform 같이 쓰는 패턴?
> *Terraform으로 EC2/VPC/RDS 생성* → *output을 Ansible inventory로 변환* → *Ansible로 OS 설정·앱 배포*. Terraform = infra provisioning, Ansible = configuration management. **Container 시대엔 Ansible 역할 축소** — *Packer로 AMI 빌드* 또는 *container image*에 모든 설정.

### Q5. Terragrunt가 *왜 필요*?
> *멀티 환경 (dev/stg/prod) Terraform 코드 중복 90%* 해결. 환경별 디렉터리에 *terragrunt.hcl* — *공통 backend + module source* 정의, *inputs만 환경별*. `terragrunt run-all apply`로 모든 환경 동시. **DRY 도구**. 작은 환경엔 *오버킬*.

### Q6. Atlantis는 무엇? *왜 필요*?
> *PR-driven Terraform 자동화*. PR open → 자동 plan + PR comment. `atlantis apply` 코멘트로 실행. *권한·승인·workflow 통합*. 직접 노트북에서 apply는 *audit·재현 안 됨* — Atlantis가 *중앙 운영*. *SaaS 대안*: Terraform Cloud, Spacelift, env0.

### Q7. *변경 정책 자동화* (policy as code)?
> CI에 *lint·security·policy 검사 통합*. **tflint** (naming·deprecated), **checkov/tfsec** (보안 — public S3, unencrypted RDS 등), **OPA + conftest** (Rego 정책 — plan JSON 검증), **Sentinel** (Terraform Cloud). *위반 시 PR 차단*. *변경 안전 보장 + 인적 실수 방지*.

---

## 🔗 Cross-reference

- **Resource·Provider** → [01-providers-and-resources.md](01-providers-and-resources.md)
- **State 관리·복구·import** → [02-state-and-backends.md](02-state-and-backends.md)
- **Module 구조** → [03-modules.md](03-modules.md)
- **Variable·Output·Secret** → [04-variables-and-outputs.md](04-variables-and-outputs.md)
- **CI / GitHub Actions / OIDC** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **Ansible 통합** → [`ansible-basics/05`](../ansible-basics/05-execution-and-best-practices.md)
- **K8s controller 비교** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **AWS provider 활용** → `aws-basics/` (예정)

---

## 📝 3줄 요약

1. *plan -out → review → apply*. **`-out` 항상 사용**. CI에서 *PR comment에 plan 자동*. *drift detection cron + Slack/Jira 알림*.
2. *Terragrunt*로 멀티 환경 DRY. *Atlantis/Terraform Cloud*로 PR-driven 자동화 + audit. *tflint/checkov/OPA*로 policy as code.
3. *Terraform (infra) + Ansible (config)* 역할 분담. Container 시대엔 *Packer AMI* 또는 *Container image*가 Ansible 대체 추세.

# 04 — Variables, Locals, Outputs

> **공식 근거:** [Input Variables](https://developer.hashicorp.com/terraform/language/values/variables), [Output Values](https://developer.hashicorp.com/terraform/language/values/outputs), [Local Values](https://developer.hashicorp.com/terraform/language/values/locals)

---

## 🎯 한 문장

> **Variable = *입력*, Local = *내부 계산값*, Output = *출력*. Type system·validation·sensitive·default가 *모듈 인터페이스의 핵심*. tfvars 파일·환경변수·CLI로 *override*.**

---

## 1. Variable 선언

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
  
  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Instance type must start with t3."
  }
  
  sensitive = false
  nullable  = true
}
```

| Field | 의미 |
|---|---|
| `type` | 자료형 (생략 시 any) |
| `description` | 사람 읽는 설명 |
| `default` | 기본값 (없으면 사용 시 명시 필수) |
| `validation` | 조건 + 에러 메시지 |
| `sensitive` | true면 plan/apply 출력에서 가림 |
| `nullable` | null 허용 (1.1+) |

---

## 2. Type 시스템

### Primitive
```hcl
variable "name"        { type = string }
variable "count"       { type = number }
variable "enabled"     { type = bool }
```

### Collection
```hcl
variable "tags"        { type = map(string) }
variable "subnet_ids"  { type = list(string) }
variable "regions"     { type = set(string) }

# 복합
variable "user_groups" { type = map(list(string)) }
```

### Object (구조체)
```hcl
variable "database" {
  type = object({
    name          = string
    engine        = string
    instance_type = string
    storage_gb    = number
    multi_az      = bool
    tags          = map(string)
  })
}
```

호출:
```hcl
module "db" {
  source = "./modules/rds"
  
  database = {
    name          = "myapp"
    engine        = "postgres"
    instance_type = "db.t3.medium"
    storage_gb    = 100
    multi_az      = true
    tags = {
      Environment = "prod"
    }
  }
}
```

### Optional attributes (1.3+)
```hcl
variable "database" {
  type = object({
    name           = string
    engine         = string
    instance_type  = optional(string, "db.t3.micro")    # default
    storage_gb     = optional(number, 20)
    multi_az       = optional(bool, false)
    tags           = optional(map(string), {})
  })
}
```

→ *복잡한 object의 일부 필드*만 default. 매우 유용.

### Tuple — *고정 길이 + 각 타입*
```hcl
variable "pair" {
  type = tuple([string, number, bool])
}

# 사용: ["hello", 42, true]
```

→ *드물게 사용*. 보통 object가 더 명확.

### any (피하기)
```hcl
variable "config" {
  type = any            # 타입 검증 안 함
}
```
→ *타입 안전성 손실*. 가급적 피함.

---

## 3. Variable 값 *제공 방법* (우선순위 낮음→높음)

1. **`default`** in variable block
2. **Environment variables** — `TF_VAR_instance_type=t3.large`
3. **`terraform.tfvars`** 파일 (자동 로드)
4. **`*.auto.tfvars`** 파일 (자동 로드, 알파벳 순서)
5. **`-var-file=prod.tfvars`** (명시 로드, CLI 마지막 것 우선)
6. **`-var key=value`** (CLI, 가장 우선)

### tfvars 예시
```hcl
# terraform.tfvars
instance_type = "t3.large"
region        = "ap-northeast-2"

tags = {
  Environment = "production"
  Team        = "platform"
}

database = {
  name          = "myapp"
  engine        = "postgres"
  instance_type = "db.t3.medium"
}
```

### 환경별 tfvars
```bash
terraform apply -var-file=envs/dev.tfvars
terraform apply -var-file=envs/prod.tfvars
```

### tfvars *git에 commit?*
- *환경별 공개 값*은 commit OK
- *secret 포함*은 절대 X — `.gitignore`에 `*.tfvars`, `secrets/`

---

## 4. Validation

```hcl
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "Environment must be dev, stg, or prod."
  }
}

variable "cidr_block" {
  type = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "port" {
  type = number
  
  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be 1-65535."
  }
}
```

→ *잘못된 값 *plan 시점에 차단*. 런타임 사고 예방.

### 여러 validation
```hcl
variable "name" {
  type = string
  
  validation {
    condition     = length(var.name) > 0
    error_message = "Name required."
  }
  
  validation {
    condition     = length(var.name) <= 63
    error_message = "Name max 63 chars."
  }
  
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.name))
    error_message = "Name must be lowercase alphanumeric."
  }
}
```

→ 여러 조건 *각각 별도 validation block*.

---

## 5. Locals — *내부 계산값*

```hcl
locals {
  # 단순
  full_name = "${var.environment}-${var.app_name}"
  
  # 조건
  instance_count = var.environment == "prod" ? 5 : 1
  
  # 합치기
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = var.project
  }
  
  all_tags = merge(local.common_tags, var.additional_tags)
  
  # 복잡한 변환
  service_configs = {
    for name, svc in var.services : name => {
      port         = svc.port
      cpu          = svc.cpu * 1024
      memory_mb    = svc.memory_gb * 1024
      env_vars     = concat(local.common_env, svc.env)
    }
  }
}
```

### Variable vs Local
| | Variable | Local |
|---|---|---|
| 출처 | 외부 (사용자) | 내부 (계산) |
| Override | 가능 | 불가 |
| 용도 | module 입력 | DRY (중복 제거) |

→ *같은 표현식 여러 곳 반복* 시 locals로.

---

## 6. Output

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "private_subnets" {
  value       = aws_subnet.private[*].id
  description = "Private subnet IDs"
}

output "database_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true             # plan·output에서 가림
}

output "kubeconfig" {
  value = templatefile("${path.module}/kubeconfig.tpl", {
    cluster_name = aws_eks_cluster.main.name
    endpoint     = aws_eks_cluster.main.endpoint
    ca_cert      = aws_eks_cluster.main.certificate_authority[0].data
  })
  sensitive = true
}
```

### Output 활용
- *Module의 호출자*가 활용 (`module.vpc.vpc_id`)
- *외부 도구가 활용* — `terraform output -json | jq .`
- *Ansible inventory 생성* — `terraform output -json | ./gen-inventory.py`
- *CI에서 다음 단계로 전달* — `cluster_endpoint=$(terraform output -raw cluster_endpoint)`

### `terraform_remote_state` — *다른 state의 output 활용*
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-state"
    key    = "infra/vpc/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_id     = data.terraform_remote_state.vpc.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnets
}
```

→ *infra 분리* (VPC state vs EKS state) 시 활용. 단 *간접 결합* 발생.

### 대안: Resource discovery (data source)
```hcl
data "aws_vpc" "main" {
  tags = { Name = "main-vpc" }
}
```
→ *output 없이도 *tag로 찾기*. *Looser coupling*.

---

## 7. Sensitive — *secret 노출 방지*

### Variable
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

→ plan/apply 출력에서 `<sensitive>` 로 가림.

### Output
```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

→ `terraform output` 에서 `<sensitive>`. 단 `terraform output -json` 으론 보임.

### `nonsensitive()` — *역방향*
```hcl
output "public_part" {
  value = nonsensitive(substr(var.secret_token, 0, 4))
}
```

→ *드물게* — sensitive에서 *일부만 평문*으로 (debugging 등).

### Sensitive의 전염
- sensitive variable을 *expression에 사용*하면 *결과도 sensitive*
- `local.x = "${var.secret}-suffix"` → `local.x` sensitive

→ *의도된 전염*. 우회하면 *보안 사고 위험*.

### 한계
- *State에는 평문 저장* (`sensitive` 는 *terminal 가림*만)
- 진짜 보안은 *backend encryption + RBAC* + *외부 secret manager*

---

## 8. *민감 정보 관리 패턴*

### Anti-pattern: tfvars 평문 password
```hcl
# ❌ git commit 절대 X
db_password = "supersecret"
```

### Pattern 1: 환경변수
```bash
export TF_VAR_db_password=$(vault kv get -field=password secret/myapp/db)
terraform apply
```

→ git에 없음, CI secret store에서 fetch.

### Pattern 2: Vault provider
```hcl
data "vault_generic_secret" "db" {
  path = "secret/myapp/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}
```

→ *Vault가 source of truth*. terraform이 *읽기만*. State에는 평문 (어쩔 수 없음).

### Pattern 3: AWS Secrets Manager
```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "myapp/db/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}
```

### Pattern 4: Random + Secrets Manager 자동 회전
```hcl
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "db" {
  name = "myapp/db/password"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}

resource "aws_db_instance" "main" {
  password = random_password.db.result
}
```

→ Terraform이 *random 생성 → Secrets Manager 저장 → RDS 적용*. 회전은 *별도 Lambda*.

---

## 9. terraform-docs — *자동 문서화*

```bash
# install
brew install terraform-docs

# README 자동 생성
terraform-docs markdown table . > README.md
```

생성 예:
```markdown
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.5 |
| aws | ~> 5.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| instance_type | EC2 instance type | `string` | `"t3.micro"` | no |
| environment | Environment name | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
| private_subnets | Private subnet IDs |
```

→ *variable·output 명세* 자동. CI에서 *생성 후 변경 있으면 fail* — 코드와 문서 *항상 동기화*.

---

## 10. 자주 헷갈리는 것

### 10-1. *Default 없는 variable*
- 사용 시 *반드시 명시* (tfvars / -var / 환경변수 / interactive prompt)
- *interactive prompt*는 *CI에서 hang* — 항상 명시 권장

### 10-2. *Sensitive 가 *모든 곳에 가림이 아님**
- terminal 출력만 가림
- state 평문
- `terraform output -json` 으론 보임
- log 통합 시스템에 *plan 출력 보존 시* 노출 가능

### 10-3. *Output을 *너무 많이* 노출*
- 모든 attribute output → *변경 추적·module 사용 복잡*
- *외부에서 진짜 필요한 것만* output

### 10-4. *Local에 *순환 참조**
```hcl
locals {
  a = local.b + 1
  b = local.a + 1   # ❌ 순환
}
```
→ 에러. 명확한 dependency 그래프 필요.

### 10-5. *Validation은 *variable 자체 값만* 검증*
- 다른 variable과의 *상관 관계* 검증 불가
- 그건 *resource의 precondition* (1.2+) 또는 *lifecycle*에

### 10-6. *terraform_remote_state vs data source*
- *terraform_remote_state* = 다른 terraform state의 output 직접 활용 — *tight coupling*
- *data source* (예: `data "aws_vpc"`) = tag·이름으로 찾기 — *loose coupling*
- 가능하면 후자 권장

---

## 🎤 면접 빈출 Q&A

### Q1. Variable 값을 *제공하는 방법*과 우선순위?
> 6가지 (낮음→높음): (1) variable block의 `default`, (2) `TF_VAR_*` 환경변수, (3) `terraform.tfvars` 자동 로드, (4) `*.auto.tfvars` 자동 로드 (알파벳 순), (5) `-var-file=prod.tfvars` 명시 로드 (CLI 마지막 우선), (6) `-var key=value` CLI (가장 우선). 운영에선 *환경별 tfvars + CI에서 환경변수로 secret*.

### Q2. Variable type 시스템?
> Primitive (string/number/bool), Collection (list/set/map), Object (구조체 — 필드별 type), Tuple (고정 길이), Optional (1.3+ — object 내 default), any (피하기). **항상 type 명시** — type 검증으로 잘못된 값 plan 시점에 차단.

### Q3. Variable vs Local 차이?
> *Variable* = *외부 입력* (사용자가 override). *Local* = *내부 계산값* (override 불가). **같은 표현식 반복 시 local로 DRY**. Variable은 module 인터페이스, local은 module 내부 logic. 예: `local.full_name = "${var.env}-${var.app}"`.

### Q4. Validation의 활용?
> Variable block 안 `validation { condition error_message }`. 잘못된 값 *plan 시점 차단* — 런타임 사고 예방. 예: environment must be `dev/stg/prod`, CIDR must be valid, port 1-65535, name pattern. 여러 validation은 *각각 별도 block*. **상관 관계 검증은 *resource precondition* (1.2+)에**.

### Q5. Sensitive variable·output 의 한계?
> *Terminal 출력에서 가림*만. **State에는 평문 저장**. `terraform output -json` 으론 보임. CI log·monitoring 통합에서 *plan 출력 보존 시* 노출 가능. 진짜 보안은 **(1) state backend encryption + RBAC, (2) 외부 secret manager (Vault, AWS Secrets Manager) 통합 — terraform이 *읽기만***. State에 평문은 *불가피*.

### Q6. *기존 다른 state의 output*을 어떻게 활용?
> **`terraform_remote_state`** data source. 다른 state 파일을 *읽어 output 추출*. 예: VPC state의 vpc_id를 EKS state에서 활용. **단 tight coupling** — VPC state 변경이 EKS에 직접 영향. **대안**: *resource discovery data source* (예: `data "aws_vpc" { tags = {...} }`) — *tag로 찾기, looser coupling*.

### Q7. Terraform-docs를 *CI에 통합*하면?
> Module의 *variable·output 명세 자동 문서화*. CI step: `terraform-docs markdown table . > README.md && git diff --exit-code README.md`. 변경 있으면 fail → *코드와 문서 항상 동기화*. 개발자는 `terraform-docs` 실행 후 commit. 사내 모든 module에 *적용 표준화* 권장.

---

## 🔗 Cross-reference

- **Resource·Provider** → [01-providers-and-resources.md](01-providers-and-resources.md)
- **State (sensitive 평문 저장)** → [02-state-and-backends.md](02-state-and-backends.md)
- **Module 인터페이스 설계** → [03-modules.md](03-modules.md)
- **CI 통합 (terraform-docs, plan output)** → [05-workflow-and-best-practices.md](05-workflow-and-best-practices.md)
- **외부 secret manager (Vault·Secrets Manager)** → `aws-basics/`, `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *Variable (입력) + Local (계산) + Output (출력)*. Type system·validation·sensitive·default·optional이 *module 인터페이스의 핵심*.
2. *Variable 값 6단계 우선순위* — `default` → env → tfvars → auto.tfvars → -var-file → -var. **운영은 환경별 tfvars + CI에서 환경변수로 secret**.
3. *Sensitive는 terminal 가림만* — state는 평문. 진짜 보안은 *backend encryption + 외부 secret manager*. *terraform-docs로 module 문서 자동화*.

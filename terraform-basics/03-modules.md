# 03 — Modules: 재사용 단위

> **공식 근거:** [Modules](https://developer.hashicorp.com/terraform/language/modules), [Module Composition](https://developer.hashicorp.com/terraform/language/modules/develop/composition)

---

## 🎯 한 문장

> **Module = *resource·variable·output을 묶은 *재사용 단위**. *입력 (variable) + 출력 (output) + 내부 (resource·logic)*. Helm chart·Ansible role과 *같은 발상*. *Terraform Registry*에 publish 가능.**

---

## 1. Module의 *구조*

### Root module + Child module
- **Root module** = `terraform apply` 실행하는 디렉터리
- **Child module** = root 또는 다른 module이 호출하는 module

### 예시
```
infra/
├── main.tf                 # root module — child module 호출
├── variables.tf
├── outputs.tf
└── modules/
    ├── vpc/                # child module
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    └── eks/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Root에서 module 호출
```hcl
# main.tf (root)
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block       = "10.0.0.0/16"
  availability_zones = ["ap-northeast-2a", "ap-northeast-2c"]
  
  tags = {
    Environment = "production"
  }
}

module "eks" {
  source = "./modules/eks"
  
  cluster_name    = "prod-cluster"
  vpc_id          = module.vpc.vpc_id           # ← module의 output 활용
  subnet_ids      = module.vpc.private_subnets
  kubernetes_version = "1.28"
}
```

### Module 안 (vpc 예시)
```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type        = string
  description = "VPC CIDR block"
}

variable "availability_zones" {
  type        = list(string)
  description = "AZs to use"
}

variable "tags" {
  type    = map(string)
  default = {}
}

# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  tags       = merge(var.tags, { Name = "main-vpc" })
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  availability_zone = var.availability_zones[count.index]
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  
  tags = merge(var.tags, { Name = "private-${count.index}" })
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnets" {
  value = aws_subnet.private[*].id
}
```

---

## 2. Module source — *어디서 가져오나*

### Local
```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

### Git
```hcl
module "vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//vpc?ref=v1.0.0"
}

# 또는 SSH
module "vpc" {
  source = "git::git@github.com:my-org/terraform-modules.git//vpc?ref=v1.0.0"
}

# ref = branch / tag / commit hash
```

### Terraform Registry — public
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

→ `terraform-aws-modules/vpc/aws` = *registry namespace*. *대규모 사내 표준 모듈 활용* 시 매우 유용.

### Private Registry — Terraform Cloud/Enterprise
```hcl
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}
```

### HTTP / S3
```hcl
module "vpc" {
  source = "https://github.com/my-org/terraform-modules/archive/v1.0.0.zip//terraform-modules-1.0.0/vpc"
}
```

### `terraform init` 동작
- 모든 module source를 *fetch* → `.terraform/modules/` 에 캐시
- *후속 init*에선 캐시 활용
- *version 변경 시* `init -upgrade` 필요

---

## 3. Module Versioning

### Git tag 활용
```hcl
module "vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//vpc?ref=v1.2.3"
}
```

- *semver 권장* (`v1.2.3`)
- breaking change는 *MAJOR 증가* (v2.0.0)
- *commit hash로 절대 pin*도 가능 (`?ref=abc123`)

### Registry semver constraint
```hcl
version = "~> 5.0"    # 5.x.x
version = ">= 5.0, < 6.0"
version = "5.1.2"     # 정확히
```

### Lock file
- *root module에 `.terraform.lock.hcl`*
- provider + module 의존성의 *정확한 버전 lock*
- **git commit 필수** — 재현성

---

## 4. Module 작성 *best practice*

### 1. *Input·Output 명확히*
- 모든 variable에 `type`·`description` 명시
- 모든 output에 `description` 명시
- `validation` 사용 ([04 참조](04-variables-and-outputs.md))

### 2. *Sensible defaults*
- 최소 명세로 동작 — *기본값 풍부*
- 단 *security 관련*은 *명시 강제* (default 없음)

### 3. *작게 유지*
- 한 module이 *한 가지 일*만
- 큰 module은 *작은 module로 분해*
- 예: `vpc`, `eks-cluster`, `eks-node-group` 분리

### 4. *Provider 명시 안 함 (보통)*
- module은 *provider를 상속*받음 (root에서 정의)
- module 안에 provider 정의 X — *passing problem* 발생

### 5. *README 필수*
- 사용 예 + 모든 variable·output 문서
- `terraform-docs` 도구로 자동 생성

### 6. *예시 디렉터리*
```
modules/vpc/
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
└── examples/
    ├── simple/
    │   ├── main.tf       # 최소 사용 예
    │   └── README.md
    └── complete/
        ├── main.tf       # 모든 옵션 활용
        └── README.md
```

---

## 5. 자주 쓰는 *공식 module*

### AWS (terraform-aws-modules)
- **vpc** — VPC + subnet + IGW + NAT + route
- **eks** — EKS cluster + node group + addons
- **rds** — RDS instance + parameter group + subnet group
- **alb** — ALB + listener + target group
- **iam** — IAM user/role/policy
- **s3-bucket** — S3 bucket + encryption + lifecycle

→ AWS infra 90% 이 module들로 가능.

### Helm provider
```hcl
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.7.1"
  namespace  = "ingress-nginx"
  
  values = [
    file("${path.module}/values.yaml")
  ]
  
  set {
    name  = "controller.replicaCount"
    value = 3
  }
}
```

→ Helm chart도 *Terraform으로 install* 가능. 단 *Helm CLI / ArgoCD가 더 흔함*.

### Kubernetes provider
```hcl
resource "kubernetes_namespace" "app" {
  metadata {
    name = "myapp"
  }
}

resource "kubernetes_deployment" "app" { ... }
```

→ K8s 자원도 Terraform으로 가능 — 단 *Helm/ArgoCD가 더 친화적* (선언적 + Git 통합).

---

## 6. Module Composition Patterns

### 1. *Flat composition* — root가 여러 module 호출
```hcl
module "vpc"      { source = "./modules/vpc"; ... }
module "eks"      { source = "./modules/eks"; vpc_id = module.vpc.vpc_id }
module "rds"      { source = "./modules/rds"; vpc_id = module.vpc.vpc_id }
module "monitoring" { source = "./modules/monitoring"; cluster = module.eks.cluster_name }
```

→ 가장 흔함. *명확*. 단 root file이 *클 수 있음*.

### 2. *Nested module* — module이 다른 module 호출
```hcl
# modules/eks/main.tf
module "iam" {
  source = "../iam-cluster"
  # ...
}

module "node_group" {
  source = "../eks-node-group"
  # ...
}
```

→ *복잡한 자원*을 *내부 분해* 가능. 단 *너무 깊으면 디버깅 어려움* — *2~3단계까지*.

### 3. *Wrapper module* — public module + 사내 default
```hcl
# modules/company-vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  # 사내 표준 default
  enable_nat_gateway     = true
  single_nat_gateway     = false   # HA를 위해
  enable_vpn_gateway     = false
  enable_dns_hostnames   = true
  
  tags = merge(var.tags, {
    ManagedBy = "terraform"
    Company   = "ourorg"
  })
  
  # 호출자가 override 가능한 부분만 노출
  cidr_block         = var.cidr_block
  availability_zones = var.availability_zones
}
```

→ public module을 *사내 컨벤션과 맞춰 wrap*. 각 팀은 wrapper만 사용 → *표준 강제*.

---

## 7. Module 안에서 *count / for_each*

### Module 호출에 count
```hcl
module "ec2" {
  source = "./modules/ec2"
  count  = var.create_instance ? 1 : 0
  
  # ...
}

# 참조
output "instance_id" {
  value = var.create_instance ? module.ec2[0].id : null
}
```

### Module 호출에 for_each
```hcl
module "service" {
  source = "./modules/service"
  for_each = var.services
  
  name = each.key
  port = each.value.port
  cpu  = each.value.cpu
}

# variable
variable "services" {
  type = map(object({
    port = number
    cpu  = number
  }))
  default = {
    api      = { port = 8080, cpu = 256 }
    worker   = { port = 9090, cpu = 512 }
  }
}
```

→ *같은 module로 여러 인스턴스* — *DRY*.

---

## 8. Provider passing — *복잡한 경우*

### 기본
- module은 *root의 provider 상속*

### 여러 provider alias가 필요한 경우
```hcl
# root
provider "aws" {
  region = "ap-northeast-2"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

module "global" {
  source = "./modules/global"
  
  providers = {
    aws.default = aws            # default
    aws.us      = aws.us         # alias 매핑
  }
}
```

```hcl
# modules/global/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.us]
    }
  }
}

# modules/global/main.tf
resource "aws_acm_certificate" "global" {
  provider = aws.us
  # ...
}
```

→ *멀티 region module*에 활용.

---

## 9. Terraform Registry

### Public Registry
- https://registry.terraform.io
- *공식 + community module*
- *download 수*, *star*, *최근 update* 가시화

### 사용 시 *검증*
- *최근 update* — 1년 안
- *download 수* — 검증된 module은 100k+ /월
- *Issue·PR 활성도*
- *공식 namespace* 우선 (`hashicorp/`, `terraform-aws-modules/`)

### Private Registry
- **Terraform Cloud / Enterprise** — 사내 module 호스팅
- **Git-based** — *git tag를 version으로* 사용

---

## 10. 자주 헷갈리는 것

### 10-1. *Module 안에서 *상위 module의 변수* 접근 안 됨*
- module은 *입력 (variable)만 받음*
- 상위에서 *명시적으로 전달* 필요

### 10-2. *Module 호출 시 *변경한 결과*가 *plan 안 보임***
- module source가 *git* — *ref 명시 안 하면* `terraform init -upgrade` 필요
- 또는 `terraform get -update`

### 10-3. *Module 안에 provider 정의*
- *passing problem* — 호출하는 root와 충돌·중복
- module은 *provider 상속*만, 정의는 root에

### 10-4. *너무 거대한 module*
- 한 module이 *VPC + EKS + RDS + monitoring 다*
- 변경 시 *영향 범위 너무 큼*, 재사용 X
- → *작게 분해*

### 10-5. *너무 깊은 nesting*
- module → module → module → module
- *debug 어려움*, *plan 결과 추적 어려움*
- 2~3단계까지

### 10-6. *Module version 안 pin*
- `source = "git::...?ref=main"` — *main 변경 시 *예상 못 한 plan 차이*
- *항상 tag로 pin*

---

## 🎤 면접 빈출 Q&A

### Q1. Module의 *입출력 설계* 원칙?
> *Variable에 `type`·`description` 명시*, *sensible defaults* (최소 명세 동작), *security 관련은 default 없이 명시 강제*. *Output에 description*, 외부에서 *조합 필요한 값만* output. **작게 유지** — 한 module이 한 가지 일. **README 필수** (`terraform-docs`로 자동 생성).

### Q2. Module source 종류?
> **Local** (`./modules/vpc`), **Git** (`git::https://...?ref=v1.0.0`), **Terraform Registry** (`terraform-aws-modules/vpc/aws`), **Private Registry** (Terraform Cloud), **HTTP/S3**. **항상 version pin** — git은 tag, Registry는 semver constraint. lock file commit.

### Q3. Provider passing은 언제·어떻게?
> *기본은 module이 root provider 상속*. 멀티 region이나 멀티 account 시 *provider alias 명시 전달* 필요. module의 `terraform.required_providers`에 `configuration_aliases = [aws.us]` 선언 + 호출 시 `providers = { aws.us = aws.us_region }` 매핑.

### Q4. Wrapper module 패턴이란?
> *Public module을 사내 default와 함께 wrap*. 예: `terraform-aws-modules/vpc/aws` 위에 *우리 회사 표준* (단일 NAT 금지, 특정 tag 강제, ...) 적용한 사내 module. 각 팀은 *wrapper만 사용* → *표준 자동 강제*. Public module 업그레이드 시 *wrapper 한 곳만 수정*.

### Q5. Module 안 `count` vs `for_each` ?
> 같음 — `for_each` 권장. 단순 N개 동일은 `count`, 식별 가능한 map/set은 `for_each`. **list 중간 삭제 시 `count`는 뒤 index 흔들림 → 자원 destroy/create**. `for_each`는 *다른 key 안 흔들림*. **module 호출에도 적용** — `module "service" { for_each = var.services; ... }`.

### Q6. Module 너무 깊게 nesting 하면?
> *Debug 어려움* — plan 결과 추적 복잡. *전체 그래프 크게 보임*. 일반적으로 *2~3단계까지*. 더 깊어지면 *책임 다시 분리* 고려 — 한 module이 *너무 많은 일* 하는 신호.

### Q7. Module *재사용성과 *유연성*의 균형?
> *너무 generic* — variable 수십 개, 사용 복잡. *너무 specific* — 재사용 X. 균형: *공통 default는 hardcode*, *환경 차이만 variable*. *opinion 있는 wrapper module*이 보통 답 (사내 표준 강제 + 최소 variable 노출). *examples 디렉터리*에 *주요 사용 패턴* 명시.

---

## 🔗 Cross-reference

- **Variable·Output 깊이** → [04-variables-and-outputs.md](04-variables-and-outputs.md)
- **CI에서 module 테스트 (Terratest)** → [05-workflow-and-best-practices.md](05-workflow-and-best-practices.md)
- **AWS module 예시 (VPC, EKS, RDS)** → `aws-basics/` (예정)
- **Helm chart의 dependency와 비교** → [`helm-basics/05`](../helm-basics/05-repos-and-distribution.md) — *같은 발상*
- **Ansible role과 비교** → [`ansible-basics/04`](../ansible-basics/04-roles-and-collections.md)

---

## 📝 3줄 요약

1. *Module = variable(입력) + resource(내부) + output(출력)의 재사용 단위*. *Helm chart·Ansible role과 같은 발상*.
2. Source 종류: *local, git (tag pin), Terraform Registry (semver)*. **lock file commit + version pin 필수**. Provider는 *root에서 정의, module은 상속* (필요 시 explicit passing).
3. *Wrapper module 패턴*으로 *사내 표준 강제*. 작은 module + 적절한 composition. 깊은 nesting (3단계+)은 *재분리 신호*.

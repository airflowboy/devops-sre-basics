# 01 — Providers & Resources

> **공식 근거:** [Terraform Language docs](https://developer.hashicorp.com/terraform/language), [Resources](https://developer.hashicorp.com/terraform/language/resources)

---

## 🎯 한 문장

> **Provider = *cloud/SaaS API와 통신하는 plugin*. Resource = *그 API로 만들 자원* (EC2, S3, K8s deployment 등). Data source = *기존 자원 정보 읽기 (read-only)*. Meta-argument로 *동작 제어* (count, for_each, depends_on, lifecycle).**

---

## 1. Provider — *cloud와의 통로*

### 선언
```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
  
  # 인증 (보통 환경변수·IAM role)
  # access_key, secret_key — 명시는 *피함*
}

provider "kubernetes" {
  config_path = "~/.kube/config"
  # 또는 host/token/cluster_ca_certificate
}
```

### 인증 방법
| 방법 | 비고 |
|---|---|
| 환경변수 (`AWS_ACCESS_KEY_ID` 등) | 가장 흔함, CI에서 |
| `~/.aws/credentials` profile | 로컬 개발 |
| **IAM Role (EC2 instance profile / IRSA / OIDC)** | 권장 — 정적 키 0 |
| Provider config 안 hardcode | *절대 안 됨* — 보안 사고 |

### Provider 종류
- **Cloud**: aws, google (GCP), azurerm, oci, alicloud, ...
- **K8s**: kubernetes, helm
- **DNS**: cloudflare, dnsimple, route53 (aws에 포함)
- **VCS**: github, gitlab, bitbucket
- **Secret**: vault, hashicorp/vault
- **Monitoring**: datadog, newrelic, grafana
- **Database**: mysql, postgresql, mongodbatlas
- **수천 종** — Terraform Registry에서 검색

### Provider 별칭 (alias)
같은 provider 여러 인스턴스 — *멀티 region·멀티 account*:
```hcl
provider "aws" {
  region = "ap-northeast-2"
}

provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "global" {
  provider = aws.us-east-1   # ← 명시적 선택
  ...
}
```

---

## 2. Resource — *생성·관리할 자원*

### 기본 syntax
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

- `aws_instance` — *resource type* (provider가 정의)
- `web` — *local name* (Terraform 안에서 참조용)
- block 안: *resource argument*

### 참조 — `<type>.<name>.<attribute>`
```hcl
resource "aws_eip" "web_eip" {
  instance = aws_instance.web.id      # ← 다른 resource의 attribute
  domain   = "vpc"
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

→ 참조가 *implicit dependency*. Terraform이 *생성 순서 자동 결정*.

### Resource 작동 원리
- *Create*: API 호출해 자원 생성, ID·attribute 반환, *state에 기록*
- *Read*: state의 ID로 API에 *현재 상태 조회*
- *Update*: API 호출 (자원에 따라 in-place 또는 destroy+recreate)
- *Delete*: API 호출해 삭제, state에서 제거

→ CRUD = provider plugin의 책임.

---

## 3. Data Source — *기존 자원 정보 읽기 (read-only)*

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  
  owners = ["099720109477"]    # Canonical
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id    # ← data source 활용
  instance_type = "t3.micro"
}
```

### 자주 쓰는 data source
- `aws_ami` — 최신 AMI 자동 검색
- `aws_caller_identity` — 현재 account/user
- `aws_region`, `aws_availability_zones`
- `aws_vpc`, `aws_subnet` — 기존 자원 참조
- `aws_iam_policy_document` — IAM 정책 JSON 생성
- `http` — 외부 URL fetch
- `template_file` (deprecated, use `templatefile()` function)

### Resource vs Data Source
- *Resource* = *terraform이 *만들고 관리*
- *Data Source* = *기존 자원 정보 *읽기만**
- *내가 만든 게 아닌* 자원은 data source 사용

---

## 4. Meta-Arguments — *resource 동작 제어*

### `count` — *동일 자원 N개*
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-..."
  instance_type = "t3.micro"
  
  tags = {
    Name = "web-${count.index}"
  }
}

# 참조: aws_instance.web[0].id, aws_instance.web[1].id, ...
# 전체 list: aws_instance.web[*].id
```

### `for_each` — *map/set 기반 생성*
```hcl
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}

# 또는 dict
resource "aws_s3_bucket" "buckets" {
  for_each = {
    logs = { versioning = true }
    data = { versioning = false }
  }
  bucket = "my-${each.key}-bucket"
}

# 참조: aws_iam_user.users["alice"].arn
```

### `count` vs `for_each` 차이
| | count | for_each |
|---|---|---|
| 식별 | index (`[0]`, `[1]`) | key (`["alice"]`) |
| list 중간 삭제 | 뒤 자원 *모두 인덱스 변경 → destroy+recreate* | 안 흔들림 |
| 권장 | 단순 N개 동일 | *대부분의 경우* 더 안전 |

→ **`for_each` 권장**. `count`는 *순서 변경에 약함*.

### `depends_on` — *명시적 의존성*
```hcl
resource "aws_iam_role_policy" "policy" { ... }

resource "aws_instance" "web" {
  ami = "..."
  
  # IAM policy 적용 후에 EC2 시작 (의존이 implicit 안 잡힐 때)
  depends_on = [aws_iam_role_policy.policy]
}
```

→ 보통 *implicit (attribute 참조)*이면 자동. *implicit 안 잡힐 때*만 명시.

### `lifecycle` — *변경·삭제 제어*
```hcl
resource "aws_instance" "web" {
  ami = "..."
  
  lifecycle {
    # 이 자원은 *절대 삭제 안 됨* (terraform이 거부)
    prevent_destroy = true
    
    # 새 자원 *먼저 생성 후 옛 자원 삭제* (downtime 회피)
    create_before_destroy = true
    
    # tags 변경은 *무시* (drift 허용)
    ignore_changes = [
      tags,
      ami,                          # AMI 변경 무시 (수동 변경 허용)
    ]
    
    # 어떤 변경이 *replace 강제*하는지 명시
    replace_triggered_by = [
      aws_security_group.web.id     # SG 바뀌면 EC2 replace
    ]
  }
}
```

### `provisioner` — *생성 후 명령 실행* (가급적 회피)
```hcl
resource "aws_instance" "web" {
  ...
  provisioner "remote-exec" {
    inline = ["sudo apt update"]
  }
}
```

→ *provisioner는 *마지막 수단**. Ansible/cloud-init/user_data 권장.

---

## 5. Resource ID와 *변경 시 동작*

### In-place update
- *tags 변경* 같이 *자원 재생성 없이* 변경
- API의 *update operation*으로

### Destroy + Create (replace)
- *어떤 attribute는 변경 불가* (예: AMI, instance_type 일부, VPC subnet)
- → Terraform이 *destroy → create*. 새 ID 부여.
- `plan` 결과에 `-/+ destroy and then create replacement` 표시

### Plan 결과 기호
```
+ create
~ update in-place
- destroy
-/+ destroy and create (replace)
<= read (data source)
```

---

## 6. Functions — *값 변형*

```hcl
locals {
  upper_name = upper(var.name)
  
  # 문자열
  joined  = join(",", var.list)
  split   = split(",", "a,b,c")
  
  # 조건
  env_tag = var.env == "prod" ? "Production" : "Dev"
  
  # collection
  merged  = merge(var.tags1, var.tags2)
  flat    = flatten([[1,2], [3,4]])
  
  # JSON / YAML
  config_json = jsonencode({ name = var.name, port = 8080 })
  config_yml  = yamlencode({ ... })
  
  # 파일
  user_data = file("${path.module}/user-data.sh")
  template  = templatefile("${path.module}/config.tpl", { var = value })
  
  # crypto
  hash = sha256("some string")
  
  # 시간
  timestamp = formatdate("YYYY-MM-DD", timestamp())
}
```

→ 함수 *수백 개*. [Terraform docs - Functions](https://developer.hashicorp.com/terraform/language/functions).

---

## 7. Expression — *조건 / 반복*

### 조건
```hcl
resource "aws_instance" "web" {
  count         = var.create_instance ? 1 : 0
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
}
```

### Splat (`*`)
```hcl
# 모든 instance의 id list
output "all_ids" {
  value = aws_instance.web[*].id
}

# 또는
output "all_ids" {
  value = [for inst in aws_instance.web : inst.id]
}
```

### For expression
```hcl
locals {
  # list 변형
  upper_names = [for n in var.names : upper(n)]
  
  # 조건 필터
  prod_only = [for n in var.names : n if can(regex("^prod-", n))]
  
  # map 만들기
  name_to_id = {for inst in aws_instance.web : inst.tags.Name => inst.id}
}
```

### Dynamic block — *동적 nested block*
```hcl
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

---

## 8. 파일 구조 — *관례*

```
infra/
├── main.tf              # 메인 resource·module
├── variables.tf         # variable 선언
├── outputs.tf           # output 선언
├── versions.tf          # terraform·provider 버전
├── providers.tf         # provider 설정
├── locals.tf            # locals
├── data.tf              # data source
├── vpc.tf               # VPC 관련 (분리)
├── eks.tf               # EKS 관련 (분리)
├── rds.tf               # RDS 관련 (분리)
└── terraform.tfvars     # variable 값 (보통 .gitignore)
```

→ Terraform은 *디렉터리 안 모든 *.tf 파일을 자동 로딩* — 분리는 *가독성* 위해.

### Override 패턴 — `*_override.tf`
- 같은 이름 `_override.tf` 파일이 *기존 정의를 override*
- 환경별 차이에 활용 (가끔)

---

## 9. CLI 기본 명령

```bash
terraform init                  # provider/module 다운로드, backend 초기화
terraform init -upgrade          # 의존성 업그레이드
terraform init -reconfigure      # backend 재구성

terraform validate               # syntax 검증
terraform fmt                    # 포맷 자동 정리
terraform fmt -recursive

terraform plan                   # 변경 사항 미리
terraform plan -out=tfplan       # 파일로 저장
terraform plan -var="key=value"
terraform plan -var-file=prod.tfvars

terraform apply tfplan           # 저장된 plan 실행 (정석)
terraform apply                  # plan + apply (개발용)
terraform apply -auto-approve    # confirm 건너뜀 (CI)

terraform destroy                # 모든 자원 삭제 (위험)
terraform destroy -target=aws_instance.web   # 특정 자원만

terraform show                   # 현재 state 보기
terraform state list             # state의 자원 목록
terraform state show aws_instance.web

terraform output                 # output 값 보기
terraform output -json
terraform output instance_ip     # 특정 output
```

---

## 10. 자주 헷갈리는 것

### 10-1. *`resource` vs `data`* 헷갈림
- `resource` = *terraform이 만들고 관리*
- `data` = *기존 자원 *읽기만**
- *이미 존재하는 VPC*를 사용하면 data, *내가 만드는* VPC면 resource

### 10-2. *count의 *중간 삭제* 함정*
- list `[a, b, c]` → count 3
- 중간 b 삭제 → list `[a, c]` → *c가 *index 1로 옮겨감* → c가 *destroy+recreate*
- 해결: `for_each`로

### 10-3. *Provider version 안 pin*
- `version = "~> 5.0"` 또는 *명시 안 함* → *나중에 *예상 못 한 변경*
- *lock file* (`.terraform.lock.hcl`) commit으로 *정확한 버전 lock*

### 10-4. *`depends_on`을 *implicit 가능한데 명시**
- 보통 *attribute 참조*면 자동 추적
- `depends_on`은 *불필요 + 그래프 복잡*

### 10-5. *resource block 안에서 *for/if 직접 안 됨*
- meta-argument (`count`, `for_each`) 외엔 안 됨
- 더 풍부한 표현은 *dynamic block*

### 10-6. *output에 secret 박힘*
- DB password 같은 거 output으로 노출
- `sensitive = true` 옵션 — *terraform output에 가림*
- 그러나 *state에는 그대로 평문*

---

## 🎤 면접 빈출 Q&A

### Q1. Provider는 무엇이고 어떻게 작동?
> Provider = *cloud/SaaS API와 통신하는 plugin*. HashiCorp가 *core만 제공*, 각 cloud는 *별도 provider* (aws, google, azurerm, kubernetes, ...). `required_providers`에 선언 → `terraform init` 시 다운로드 → *resource CRUD를 API 호출로 변환*. 인증은 환경변수·IAM Role·OIDC 권장 (hardcode 금지).

### Q2. Resource와 Data Source 차이?
> **Resource**: terraform이 *만들고 관리* (Create/Update/Delete + state 기록). **Data Source**: *기존 자원 정보 read-only*. 내가 만든 자원이 아닌 *기존 VPC/AMI/...* 참조 시 data source. 두 자원 *같은 자원 가리키면 안 됨* — 충돌.

### Q3. `count`와 `for_each` 차이? 무엇을 권장?
> `count` = *index 기반* (`[0]`, `[1]`). list 중간 삭제 시 *뒤 자원 모두 index 변경 → destroy+recreate* (위험). `for_each` = *map/set의 key 기반* (`["alice"]`). 중간 삭제해도 *다른 key 자원 안 흔들림*. **`for_each` 권장**. count는 *단순 N개 동일 자원*에만.

### Q4. `lifecycle` block의 옵션과 활용?
> `prevent_destroy` — *삭제 차단* (중요 자원 보호). `create_before_destroy` — *새 먼저 생성 후 옛 삭제* (downtime 회피, 예: ASG). `ignore_changes` — *특정 attribute 변경 무시* (수동 변경 허용, 예: tags·AMI). `replace_triggered_by` — *외부 변경에 자원 replace 강제*.

### Q5. `depends_on`은 언제 명시?
> 보통은 *불필요* — *attribute 참조*면 implicit dependency 자동 추적. 명시는 *implicit 안 잡히는 경우*만: (1) IAM policy 적용 후 자원 시작 같이 *권한 의존*, (2) 외부 자원 (data source)이 *terraform 자원 적용 후* 존재, (3) provisioner에서 다른 자원 필요. 남발하면 *그래프 복잡 + 병렬화 손해*.

### Q6. Dynamic block은 언제·어떻게?
> *resource 안 nested block (security group rule 등)을 *반복 생성* 필요할 때*. `dynamic "ingress" { for_each = var.rules; content { ... } }` 형식. 변수 list로 *security group rule N개*, *IAM policy statement N개* 같이 *수가 변동*인 경우. 정적이면 그냥 block 여러 개.

### Q7. Provider version pin이 왜 필요?
> Version 안 pin → *팀원·CI마다 다른 버전 → 예상 못 한 plan 차이*. `required_providers { aws = { version = "~> 5.0" } }` + `.terraform.lock.hcl` (자동 생성)을 **git commit**. `terraform init`이 lock 기반 동일 버전 보장. 의도적 업그레이드는 `init -upgrade`.

---

## 🔗 Cross-reference

- **State 자체** → [02-state-and-backends.md](02-state-and-backends.md)
- **Module로 재사용** → [03-modules.md](03-modules.md)
- **Variable·Output 깊이** → [04-variables-and-outputs.md](04-variables-and-outputs.md)
- **AWS resource 예시 (VPC·EC2·IAM·EKS·RDS)** → `aws-basics/` (예정)
- **K8s provider (terraform으로 K8s 자원 관리)** → [`kubernetes-basics/`](../kubernetes-basics/) — 단 *Helm/ArgoCD가 더 흔함*

---

## 📝 3줄 요약

1. *Provider = cloud API plugin, Resource = 자원, Data Source = 읽기 전용 참조*. Meta-argument (`count`/`for_each`/`depends_on`/`lifecycle`)로 동작 제어.
2. **`for_each` 권장** (count의 index 흔들림 회피). **`lifecycle.create_before_destroy`** 로 *downtime 회피*. **`ignore_changes`** 로 *수동 변경 허용*.
3. Provider version + lock file commit으로 *재현성*. *조건·반복은 expression + dynamic block*. *Functions·for·splat*이 복잡한 표현 지원.

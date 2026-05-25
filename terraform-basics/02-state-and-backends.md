# 02 — State & Backends: Terraform의 *기억* 메커니즘

> **공식 근거:** [State](https://developer.hashicorp.com/terraform/language/state), [Backends](https://developer.hashicorp.com/terraform/language/backend)
>
> **이 파일이 Terraform 운영의 *80%*.** state 이해 못 하면 사고 = state 손실 = *복구 매우 어려움*.

---

## 🎯 한 문장

> **State = *Terraform이 *마지막으로 적용한 결과를 기록한 JSON 파일**. cloud API의 *현재 상태*와 *HCL의 desired state* 비교를 *효율적으로* 가능하게 함. 운영에선 *반드시 remote backend (S3 + DynamoDB locking)*. 손실 시 *대형 사고*.**

---

## 1. State는 *왜* 필요한가

### 시나리오: state 없으면
1. 사용자가 HCL 작성 (`resource "aws_instance" "web"`)
2. Terraform: "이 자원이 이미 있나? *모름*. 매번 AWS API에 *전수 조회*?"
3. AWS에 *수많은 EC2 인스턴스*. 그중 *어느 게 이 HCL의 `web`*인지 *어떻게 알아*?
4. → *생성 후 ID 어딘가에 기억해야* 함

### state의 정체
- *Terraform이 마지막으로 적용한 결과*의 JSON
- 각 resource block에 *AWS의 ID 매핑*
- *attribute 모두 기록* (다음 plan 비교용)

```json
{
  "version": 4,
  "terraform_version": "1.5.7",
  "serial": 17,
  "lineage": "abc-def-...",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-0abc1234",
            "instance_type": "t3.micro",
            "public_ip": "54.123.45.67",
            ...
          }
        }
      ]
    }
  ]
}
```

### State의 역할
1. **Mapping**: HCL resource ↔ cloud의 실제 ID
2. **Caching**: 이전 attribute 저장 — *매 plan마다 cloud API 호출 줄임*
3. **Performance**: 큰 인프라에서 *수십~수백 자원* 전수 조회 비현실
4. **Dependency tracking**: resource 간 의존 관계 보존

---

## 2. State Storage — *Local vs Remote*

### Local (기본)
```hcl
# 명시 안 하면 local
terraform {
  # backend "local" {}    ← 기본
}
```
- `terraform.tfstate` 파일이 *현재 디렉터리에*
- 단점:
  - 노트북 *분실 시 state 손실*
  - *팀에서 공유 불가* — 각자 다른 state
  - *git commit 금지* (secret 포함)
  - *동시 apply 시 충돌·corruption*

→ **운영에선 절대 local 금지**.

### Remote — *팀 공유 + 보안 + locking*
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "infra/prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

→ S3에 state 저장 + DynamoDB로 locking. *현대 표준*.

### Remote backend 종류

| Backend | 비고 |
|---|---|
| **s3** | AWS S3 + DynamoDB lock (가장 흔함) |
| **gcs** | GCP Cloud Storage |
| **azurerm** | Azure Storage |
| **remote** (Terraform Cloud) | HashiCorp SaaS — 무료 tier |
| **consul** | Consul KV |
| **postgres** | PostgreSQL |
| **kubernetes** | K8s Secret (간단한 경우) |
| **http** | 사내 HTTP 백엔드 |

---

## 3. S3 Backend 셋업 — *Chicken-and-egg*

### 문제
- S3 bucket·DynamoDB를 *Terraform으로 만들고 싶음*
- 그런데 *그 Terraform이 state를 *어디에 저장*?

### 해결: Bootstrap
```
1. *수동으로* S3 bucket + DynamoDB 1회 생성
2. backend.hcl에 그 bucket 명시
3. 이후 모든 Terraform은 그 backend 사용
```

또는
```
1. *별도 Terraform* (다른 backend, 또는 local)으로 bucket 생성
2. *그 bucket을 main Terraform의 backend로*
```

### S3 bucket 설정 권장
- **Versioning enabled** — state 변경 history 보존, *복구 가능*
- **Encryption (SSE-S3 또는 KMS)** — 평문 secret 보호
- **MFA delete** — 실수 delete 방지
- **Access logging** — audit
- **Bucket policy** — *최소 권한*

### DynamoDB locking
- `terraform apply` 시작 → DynamoDB에 *lock 행 생성*
- 다른 사용자가 apply 시도 → *lock 확인 → 차단*
- apply 완료 → lock 해제

→ *동시 apply로 state corruption 방지*.

---

## 4. State Locking

### 동작
```
User A: terraform apply
  → DynamoDB: "lock-id-123, user=alice, ts=12:00"
  → (apply 진행 중)

User B: terraform apply
  → DynamoDB: "Already locked by alice"
  → ❌ Wait or fail
```

### Lock 정보
- 누가 lock — `user@host`
- 언제 — timestamp
- 어느 자원 — operation
- expire — 기본 없음

### 강제 unlock — *조심*
```bash
terraform force-unlock <lock-id>
```
- *실제 다른 사람이 apply 중이면 *state 손상 위험**
- *명백한 사고 (CI 죽음, 노트북 죽음)*에서만

---

## 5. Workspaces — *환경 분리의 *간단한 방법**

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform workspace list
terraform workspace show
```

### 동작
- 같은 backend에 *workspace별 *별도 state 파일**
- S3 key가 `env:/<workspace>/<key>` 로 자동 변경

### HCL에서 활용
```hcl
locals {
  env = terraform.workspace
}

resource "aws_instance" "web" {
  instance_type = local.env == "prod" ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = local.env
  }
}
```

### Workspace의 한계
- *같은 HCL 코드 + 다른 state* — 환경별 *큰 차이* 표현 어려움
- prod와 dev가 *완전 다른 자원 구조*면 *workspace 부적합*

→ 운영에선 *workspace보다 directory 분리* 패턴이 더 흔함:
```
infra/
├── modules/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   ├── stg/
│   └── prod/
```

또는 **Terragrunt** ([05 참조](05-workflow-and-best-practices.md))로 DRY.

---

## 6. State 조작 명령

### `terraform state list`
```bash
terraform state list
# aws_instance.web
# aws_instance.db
# aws_vpc.main
# module.eks.aws_eks_cluster.this
```

### `terraform state show`
```bash
terraform state show aws_instance.web
# resource "aws_instance" "web" {
#     ami                = "ami-..."
#     id                 = "i-0abc1234"
#     ...
# }
```

### `terraform state mv` — *resource 이름 변경 (refactoring)*
```bash
# HCL을 aws_instance.web → aws_instance.frontend 로 변경했다면
terraform state mv aws_instance.web aws_instance.frontend
# state의 이름만 변경. 실제 AWS 자원은 그대로 (recreate 안 함)
```

### `terraform state rm` — *state에서 제거 (자원은 두기)*
```bash
terraform state rm aws_instance.web
# Terraform이 더 이상 관리 안 함. AWS의 EC2는 그대로.
```

→ 자원을 *Terraform 관리에서 제외*하고 *직접 관리*로 옮길 때.

### `terraform state pull` / `push`
```bash
terraform state pull > state.json   # download
# 수동 편집...
terraform state push state.json     # upload (위험 — backup 후)
```

→ *수동 편집은 *최후 수단**. 보통 `state mv`/`rm`로 해결.

---

## 7. Import — *기존 자원을 Terraform 관리로*

### 시나리오
- 옛 AWS console에서 *수동 생성한 EC2*
- 이제 *Terraform으로 관리*하고 싶음
- → HCL 작성 + *import*

### 절차
```hcl
# 1. HCL 작성 (자원 정의)
resource "aws_instance" "legacy_web" {
  # ami, instance_type, etc. — 실제 자원과 일치하게
}
```

```bash
# 2. import 실행
terraform import aws_instance.legacy_web i-0abc1234

# 3. plan 실행해서 *차이 확인*
terraform plan
# (HCL과 실제 자원이 안 맞으면 plan에 차이 표시 → HCL 수정)

# 4. 차이 없을 때까지 HCL 다듬기
# 5. apply (이제 Terraform이 관리)
```

### Import block (Terraform 1.5+)
```hcl
import {
  to = aws_instance.legacy_web
  id = "i-0abc1234"
}

resource "aws_instance" "legacy_web" {
  # ...
}
```

→ `terraform plan -generate-config-out=imported.tf` — *HCL 자동 생성*. 매우 편리.

---

## 8. State 손실 — *복구 시나리오*

### Versioning이 활성화된 S3 bucket
1. `aws s3api list-object-versions --bucket my-state --prefix infra/prod/terraform.tfstate`
2. 직전 version 다운로드
3. *상태 검증* (`terraform state list` 직전 결과와 비교)
4. *upload* (조심)

### Versioning 없음
- *백업 없으면 매우 어려움*
- 옵션:
  - **모든 자원 `terraform import`** — 수십~수백 자원이면 *지옥*
  - **AWS Resource Tags 활용** — `terraform-managed=true` 같은 tag로 식별
  - **`terraformer`** (외부 도구) — *cloud에서 자원 자동 import + HCL 생성*

→ **예방이 최선**: S3 versioning 필수.

### Lock 깨짐
- DynamoDB에 *lock 행 영원히 남음*
- *진짜 다른 apply 없음 확인 후* `terraform force-unlock <lock-id>`

---

## 9. State *내부 구조*와 *민감 정보*

### State는 *모든 attribute 평문 저장*
- DB password (RDS)
- API key (Secret resource)
- Private key (TLS)

### 함의
- *S3 bucket encryption 필수* (SSE-S3 또는 KMS)
- *S3 bucket RBAC 빡세게* — *팀원·CI 외엔 read 금지*
- *터미널 출력에 secret* 가능 — `sensitive = true` 옵션
- *git commit 절대 X*

### Sensitive output
```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```
- `terraform output` 출력에서 `<sensitive>` 로 가림
- 단 state에는 *여전히 평문*. `terraform output -json`으론 보임.

### Vault·외부 secret manager 통합
- secret을 *terraform state에 안 두기*
- *Vault에 저장 + provider가 fetch*
- 예: `data "vault_generic_secret"` 로 *읽기만*

---

## 10. 자주 헷갈리는 것

### 10-1. *Local backend로 운영 시작*
- *팀원과 공유 안 됨* + *각자 plan 다름* + *secret 노출 위험*
- **무조건 remote backend로 시작**

### 10-2. *S3 bucket 자체를 Terraform이 만들고 그 bucket을 backend로*
- *chicken-and-egg* — 첫 apply 시 backend 없음
- 별도 bootstrap (수동 또는 별도 Terraform)

### 10-3. *State 파일 git commit*
- secret 노출 + 충돌 + 의미 없음
- `.gitignore`에 `*.tfstate*`, `.terraform/`

### 10-4. *Force unlock 남발*
- 실제 동시 apply 중에 unlock → *state 손상*
- *반드시 다른 apply 없음 확인 후*

### 10-5. *Workspace로 prod/dev 분리*
- 같은 HCL — *환경 큰 차이 표현 어려움*
- *directory 분리 + module 재사용* 또는 *Terragrunt* 권장

### 10-6. *Import 후 plan 차이 무시하고 apply*
- HCL이 실제 자원과 다르면 *Terraform이 자원 변경*
- *plan 차이 0 될 때까지 HCL 수정 필수*

---

## 🎤 면접 빈출 Q&A

### Q1. Terraform의 *state는 왜 필요*한가?
> Terraform은 *desired state (HCL)*만 알지 *현재 cloud 상태는 모름*. 매번 *전수 조회는 비현실*. State에 *마지막 적용 결과 + ID 매핑* 저장 → 다음 plan에서 *효율적으로 비교*. 또한 *resource 간 의존 추적*, *attribute caching*. State 없으면 Terraform이 *어느 cloud 자원이 어느 HCL인지 모름*.

### Q2. Local backend vs Remote backend?
> **Local** (기본): `terraform.tfstate` 파일이 *현재 디렉터리*. 노트북 분실 시 손실, 팀 공유 불가, *동시 apply 시 충돌*. **운영 절대 금지**. **Remote** (S3 + DynamoDB / GCS / Terraform Cloud): *팀 공유 + locking + encryption + versioning*. **운영 표준은 S3 + DynamoDB**.

### Q3. State locking이 왜 필요?
> 동시에 두 사람이 `terraform apply` → *state corruption + cloud 자원 불일치*. **Lock** = apply 시작 시 DynamoDB에 *lock 행 생성* → 다른 apply 시도 *차단*. 완료 시 unlock. *진짜 lock 깨짐*은 `terraform force-unlock` (조심) — *실제 다른 apply 없음 확인 후*.

### Q4. Workspace는 무엇? 한계?
> 같은 backend에 *workspace별 별도 state 파일*. `terraform workspace new prod`. HCL에서 `terraform.workspace` 변수 활용. **한계**: 같은 HCL — *prod와 dev가 자원 구조 크게 다르면 표현 어려움*. **운영에선 *directory 분리 + module 재사용*** 또는 *Terragrunt* 권장.

### Q5. 기존 자원을 Terraform 관리로 가져오는 방법?
> **Import**. (1) HCL 작성 (자원 정의). (2) `terraform import aws_instance.legacy_web i-0abc...`. (3) `terraform plan`으로 *HCL과 실제 자원의 차이 확인*. (4) 차이 0될 때까지 HCL 수정. (5) apply. Terraform 1.5+의 **`import` block** + `plan -generate-config-out=` 로 *HCL 자동 생성* 가능 (매우 편리).

### Q6. State 손실 시 복구는?
> **예방이 최선** — S3 bucket *versioning 필수* + 정기 백업. 손실 시: (1) S3 versioning 활성화면 *직전 version 다운로드 + 검증 + upload*. (2) 없으면 *모든 자원 수동 import* (수십이면 지옥). 외부 도구 *terraformer*가 *cloud에서 자원 자동 import + HCL 생성*. (3) *Resource tag*로 *terraform 관리 자원 식별*.

### Q7. State에 secret이 *평문 저장*된다는데?
> 맞음. DB password·API key·private key 다 *평문으로 state JSON에*. 대응: (1) **S3 encryption (SSE-S3/KMS)**, (2) **S3 RBAC 빡세게** (팀·CI만 read), (3) **`sensitive = true`** 로 *terminal 출력 가림* (state는 그대로). (4) **외부 secret manager (Vault, AWS Secrets Manager) 통합** — secret을 *terraform이 만들지 않고 *읽기만**.

---

## 🔗 Cross-reference

- **Resource·Provider 기본** → [01-providers-and-resources.md](01-providers-and-resources.md)
- **Module 구조** → [03-modules.md](03-modules.md)
- **Workflow + Terragrunt + Atlantis** → [05-workflow-and-best-practices.md](05-workflow-and-best-practices.md)
- **AWS S3 / DynamoDB / KMS / IAM** → `aws-basics/` (예정)
- **Vault·Secrets Manager 통합** → `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *State = Terraform의 *기억 (마지막 적용 결과 + ID 매핑)*. cloud API의 *현재 상태 비교*를 효율적으로 가능하게. *운영은 무조건 remote backend (S3 + DynamoDB)*.
2. **State locking**으로 동시 apply 차단. **S3 versioning + encryption + RBAC** 필수. State에 *secret 평문 저장* — 보안 핵심.
3. *기존 자원은 `terraform import`*. *워크스페이스 한계는 directory 분리·Terragrunt로 해결*. State 손실은 *복구 매우 어려움 → 예방이 최선*.

# Terraform — 동작 원리 중심

> *"Terraform의 *state는 왜 필요*한가?"*
> *"plan과 apply의 차이?"*
> *"K8s의 reconciliation과 Terraform의 차이?"*
> *"state를 잃었을 때 복구는?"*

---

## 🎯 한 문장 큰 그림

> **Terraform = *선언적 IaC*. *desired state (HCL)*을 *cloud API*로 *적용*하고, 적용 결과를 *state 파일에 기록*. *plan*으로 *변경 사항 미리 보기*, *apply*로 실행. K8s reconciliation과 *같은 사상, 다른 환경* (infra provisioning vs workload).**

---

## 🧩 한 페이지 요약

```
   ┌────────────────────────────────┐
   │  HCL files (.tf)                │  ← desired state
   │  - resource "aws_instance" {...}│
   │  - variable, locals, outputs    │
   │  - module 구조                   │
   └─────────────┬──────────────────┘
                 │ terraform plan
                 ▼
   ┌────────────────────────────────┐
   │  Plan (변경 사항 미리)            │
   │  + create / ~ update / - destroy │
   └─────────────┬──────────────────┘
                 │ terraform apply
                 ▼
   ┌────────────────────────────────┐
   │  Provider API call               │
   │  - AWS / GCP / Azure / K8s / etc.│
   └─────────────┬──────────────────┘
                 │
                 ▼
   ┌────────────────────────────────┐
   │  State 파일 (.tfstate)            │  ← 적용 결과 기록
   │  - 자원 ID, 메타데이터            │
   │  - remote backend (S3 + DynamoDB)│
   └────────────────────────────────┘
```

---

## 🌪 자주 헷갈리는 7가지

### 1. *State는 *왜 필요*한가*
- HCL은 *desired state*만 기술
- *cloud API는 *현재 상태를 모름* — 매번 *전수 조회 비현실*
- → *Terraform이 *과거에 적용한 결과를 기억*해 *비교*해야 효율
- State = *Terraform의 *기억*

### 2. *Plan vs Apply*
- **Plan**: *현재 cloud 상태 read + HCL과 비교 + 차이 출력*. *변경 X*.
- **Apply**: plan 결과를 *실제 적용*.
- *plan 결과 보고 *수동 검토 후 apply* 가 정석.

### 3. *Terraform vs K8s reconciliation*
- 둘 다 *desired state ↔ actual state* 사상
- 차이: K8s = *영원히 reconcile*, Terraform = *명시적 apply 시만*
- K8s는 *내부 controller가 reconcile*, Terraform은 *외부 사용자가 trigger*
- → Terraform은 *drift detect 안 함 (자동으론)* — apply 안 하는 동안 *변경 무시*

### 4. *Provider plugin 구조*
- HashiCorp가 Terraform *core만* 제공
- 각 cloud는 *provider plugin* (HashiCorp 또는 community)
- AWS provider, GCP provider, Kubernetes provider, GitHub provider, Vault provider, ...
- → *수천 종 cloud/SaaS 통합 가능*

### 5. *State는 *민감 정보 포함*
- DB password, AWS key, secret 등 *평문으로 state에 저장*
- → state 파일 *RBAC 빡세게 + encryption at rest*
- *git commit 절대 X*. *Remote backend (S3) + encryption*.

### 6. *모듈은 *재사용 단위**
- Helm chart, Ansible role과 *같은 발상*
- *입력 (variable) + 출력 (output) + 내부 (resource)*
- *Terraform Registry*에 publish 가능

### 7. *Terraform과 Ansible 역할 분담*
- **Terraform**: *infra provisioning* — *클라우드 자원 CRUD*
- **Ansible**: *configuration management* — *OS·앱 설정*
- *Terraform output → Ansible inventory* 패턴 흔함

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-providers-and-resources.md](01-providers-and-resources.md) | provider, resource, data source, lifecycle, meta-arguments, count/for_each |
| [02-state-and-backends.md](02-state-and-backends.md) | state 의미, remote backend, locking, workspaces, import, taint/replace |
| [03-modules.md](03-modules.md) | module 구조, source, version, root/child, Terraform Registry |
| [04-variables-and-outputs.md](04-variables-and-outputs.md) | variable types, locals, outputs, sensitive, validation, tfvars |
| [05-workflow-and-best-practices.md](05-workflow-and-best-practices.md) | plan/apply lifecycle, drift detection, secret 관리, CI 통합, Terragrunt, Atlantis |

---

## 🔗 Cross-reference

- **AWS 자원 (VPC, EC2, RDS, IAM)** → `aws-basics/` (예정). Terraform이 *가장 자주 다루는 cloud*.
- **K8s reconciliation 비교** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md). *같은 사상, 다른 trigger*.
- **Ansible과의 분담** → [`ansible-basics/05`](../ansible-basics/05-execution-and-best-practices.md).
- **CI에서 terraform plan/apply** → [`cicd-gitops-basics/01`](../cicd-gitops-basics/01-ci-fundamentals.md), [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md).

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| Terraform의 state는 *왜 필요*? | [02](02-state-and-backends.md) |
| Remote backend·locking이 왜? | [02](02-state-and-backends.md) |
| plan과 apply의 차이? | [05](05-workflow-and-best-practices.md) |
| Module의 input/output 설계? | [03](03-modules.md) |
| Terraform과 Ansible 차이? | README + [05](05-workflow-and-best-practices.md) |
| Terraform과 K8s reconciliation 차이? | [05](05-workflow-and-best-practices.md) |
| 기존 자원을 Terraform으로 가져오는 (import) 방법? | [02](02-state-and-backends.md) |
| Drift detection · 안전 변경 패턴? | [05](05-workflow-and-best-practices.md) |
| Secret 관리 (DB password 등) 어떻게? | [04](04-variables-and-outputs.md) + [05](05-workflow-and-best-practices.md) |

---

## 📐 학습 순서 권장

1. **01 providers-and-resources** — *기본 syntax + provider*
2. **02 state-and-backends** — *Terraform의 *기억* 메커니즘*
3. **03 modules** — *재사용*
4. **04 variables-and-outputs** — *입출력 설계*
5. **05 workflow-and-best-practices** — *운영 안전 + CI 통합*

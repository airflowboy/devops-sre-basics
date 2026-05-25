# AWS — 동작 원리 중심

> *"IAM Role과 User 차이?"*
> *"VPC에서 Pod이 외부로 나가는 흐름?"*
> *"KMS envelope encryption은?"*
> *"EKS의 control plane은 어떻게 운영?"*

---

## 🎯 한 문장 큰 그림

> **AWS = *수백 service의 묶음*, 핵심은 **IAM (모든 권한의 시작)** + **VPC (네트워크 베이스)** + **Compute (EC2/EKS/ECS/Lambda)** + **Storage/DB (S3/EBS/RDS/DynamoDB)** + **KMS (모든 암호화)**. *Well-Architected Framework 6 pillar*가 운영 표준.**

---

## 🧩 한 페이지 요약

```
┌─────────────────────────────────────────────────────────────┐
│  IAM (모든 권한의 시작)                                        │
│  - User / Group / Role / Policy                              │
│  - Identity Center (SSO)                                     │
│  - OIDC (IRSA, GitHub Actions)                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  VPC (네트워크 베이스)                                          │
│  - Subnet (public/private) / Route Table / IGW / NAT GW       │
│  - Security Group (stateful) / NACL (stateless)              │
│  - PrivateLink / Transit Gateway                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Compute                                                      │
│  - EC2 (가상 머신, ASG)                                        │
│  - EKS (managed K8s) + Karpenter                             │
│  - ECS / Fargate (managed container)                          │
│  - Lambda (serverless)                                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Storage / DB                                                 │
│  - S3 (object) + versioning + lifecycle + event              │
│  - EBS (block) / EFS (NFS)                                   │
│  - RDS / Aurora (relational, managed)                        │
│  - DynamoDB (NoSQL)                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Secrets + Encryption                                         │
│  - KMS (envelope encryption, CMK, GenerateDataKey)            │
│  - Secrets Manager (회전 + 통합)                                │
│  - Parameter Store (config + secret)                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Observability / Audit                                        │
│  - CloudWatch (metrics / logs / alarms)                       │
│  - CloudTrail (모든 API 호출 audit)                            │
│  - GuardDuty (threat detection)                              │
│  - Security Hub (통합 보안 발견)                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🌪 자주 헷갈리는 7가지

### 1. *User vs Role 차이*
- **User**: *사람·앱의 영구 신원* — 정적 AccessKey
- **Role**: *임시 자격 (STS)* — *누군가가 *assume*해서 사용*
- → 정적 키 사고 회피 — **모든 곳에 Role + STS** 권장

### 2. *Security Group은 stateful, NACL은 stateless*
- SG: *return traffic 자동 허용* — outbound 룰 안 해도 됨
- NACL: *return traffic도 명시 필요* — *ephemeral port 32768-60999 허용 필수*

### 3. *Public IP는 *자동 부여 아님**
- Subnet 설정 + IGW 라우트 + Public IP allocation 모두 필요
- *Private subnet*에선 인스턴스가 IGW 못 봄 — NAT GW 거쳐 외부

### 4. *S3는 *Region service*, IAM은 *Global**
- S3 bucket = region (단 *bucket 이름은 global 유일*)
- IAM·CloudFront·Route53 = global

### 5. *EBS는 *AZ에 묶임**
- EBS volume은 *특정 AZ* — 다른 AZ EC2에 mount 불가
- *EFS는 multi-AZ* (NFS)

### 6. *KMS key는 *직접 데이터 암호화 X**
- KMS는 *Data Key만 발급* — *큰 데이터는 그 data key로 클라이언트가 암호화*
- → *envelope encryption* 패턴
- Data key는 KMS의 master key (CMK)로 *암호화돼 함께 저장*

### 7. *IAM Policy는 *deny가 allow를 항상 이김**
- explicit deny > explicit allow > default deny
- 정책 *N개 합쳐도 deny 하나면 거부*

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-iam-and-identity.md](01-iam-and-identity.md) | User/Role/Policy/STS, Identity Center, OIDC, IRSA, Resource Policy |
| [02-vpc-and-networking.md](02-vpc-and-networking.md) | VPC/Subnet/Route, IGW/NAT, SG vs NACL, PrivateLink, Transit Gateway |
| [03-compute-and-eks.md](03-compute-and-eks.md) | EC2/ASG/Spot, EKS deep (control plane, addons, IRSA), Fargate, Lambda |
| [04-storage-and-database.md](04-storage-and-database.md) | S3 (versioning/lifecycle/replication/event), EBS/EFS, RDS/Aurora, DynamoDB |
| [05-secrets-and-kms.md](05-secrets-and-kms.md) | KMS envelope encryption, Secrets Manager, Parameter Store, audit |
| [06-cost-and-finops.md](06-cost-and-finops.md) | Cost Explorer, Budget, Savings Plan, Reserved, Spot, FinOps culture |

---

## 🔗 Cross-reference

- **Linux + 네트워크 베이스** → [`linux-basics/`](../linux-basics/), [`network-basics/`](../network-basics/)
- **K8s on EKS** → [`kubernetes-basics/`](../kubernetes-basics/) + [03-compute-and-eks.md](03-compute-and-eks.md)
- **Terraform으로 AWS 자원** → [`terraform-basics/`](../terraform-basics/)
- **IRSA = K8s SA → IAM Role** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), [`k8s-security-basics/03`](../k8s-security-basics/03-identity-and-rbac.md)
- **CI에서 OIDC로 AWS access** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **AWS Secrets Manager + External Secrets Operator** → [`k8s-security-basics/05`](../k8s-security-basics/05-secrets-and-encryption.md)

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| IAM User vs Role 차이? | [01](01-iam-and-identity.md) |
| IRSA 동작 원리? | [01](01-iam-and-identity.md) + [03](03-compute-and-eks.md) |
| VPC 안에서 Pod이 외부로 나가는 흐름? | [02](02-vpc-and-networking.md) |
| SG와 NACL 차이? | [02](02-vpc-and-networking.md) |
| EKS의 control plane 누가 관리? | [03](03-compute-and-eks.md) |
| Karpenter vs Cluster Autoscaler? | [03](03-compute-and-eks.md) |
| S3의 *eventual consistency*는? | [04](04-storage-and-database.md) |
| RDS Multi-AZ vs Read Replica 차이? | [04](04-storage-and-database.md) |
| KMS envelope encryption은? | [05](05-secrets-and-kms.md) |
| Spot Instance를 *안전하게 사용*? | [06](06-cost-and-finops.md) |

---

## 📐 학습 순서 권장

1. **01 iam-and-identity** — *모든 것의 시작*
2. **02 vpc-and-networking** — *모든 자원이 살 곳*
3. **03 compute-and-eks** — *워크로드 실행*
4. **04 storage-and-database** — *데이터*
5. **05 secrets-and-kms** — *보안*
6. **06 cost-and-finops** — *운영 효율*

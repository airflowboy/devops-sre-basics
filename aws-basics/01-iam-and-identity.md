# 01 — IAM & Identity: User · Role · Policy · STS · IRSA

> **공식 근거:** [IAM docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/), [STS](https://docs.aws.amazon.com/STS/latest/APIReference/), [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/)

---

## 🎯 한 문장

> **IAM = *모든 AWS API 권한의 시작*. User (사람·앱 영구 신원) vs Role (임시 자격, STS로 assume). **모든 곳에 Role + STS 권장** — 정적 키 0개. Policy는 *deny가 allow 이김*. OIDC trust로 *GitHub Actions·K8s SA*도 IAM Role assume 가능.**

---

## 1. 4가지 핵심 요소

| 요소 | 의미 |
|---|---|
| **User** | *영구 신원* (사람 또는 앱). 정적 AccessKey 가능 (위험). |
| **Group** | User 묶음. *권한을 group에 부여*하면 group의 모든 user가 받음. |
| **Role** | *임시 자격*. *누군가가 assume*해서 사용. STS가 임시 credential 발급. |
| **Policy** | *권한 정의* (JSON). User/Group/Role에 attach. |

### 권장 모델
- **사람**: Identity Center (SSO) → IAM Role assume (정적 키 0)
- **앱 (EC2)**: Instance Profile → IAM Role
- **K8s Pod**: ServiceAccount → IAM Role (IRSA)
- **CI/CD**: OIDC → IAM Role (GitHub Actions, GitLab CI)
- **타 AWS service**: Service-linked Role

→ **User + 정적 AccessKey는 *최후 수단**. 침해 시 영구 위험.

---

## 2. Policy — *권한의 단위*

### Policy 구조 (JSON)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceVpc": "vpc-12345"
        }
      }
    },
    {
      "Sid": "DenyDeleteEvenIfAllowed",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

### 평가 logic
1. **Default = Deny**
2. *어떤 policy든 Allow* → 통과
3. *어떤 policy든 Explicit Deny* → 거부 (다른 Allow 무시)

→ **Deny가 항상 이김**. 정책 N개 합쳐도 *하나의 Deny가 모두 차단*.

### Action·Resource·Condition
- **Action**: `s3:GetObject`, `ec2:DescribeInstances`, `dynamodb:Query` 등
- **Resource**: ARN (`arn:aws:s3:::my-bucket/*`)
- **Condition**: 추가 제약 — IP, MFA, time, tag, source VPC 등

### Wildcard
- `s3:Get*` — Get으로 시작하는 모든 action
- `*` — 모든 (위험)
- **운영은 *정확한 action 명시*** — wildcard는 *공격 표면 ↑*

### Policy 종류
| 종류 | 위치 |
|---|---|
| **AWS Managed Policy** | AWS 제공 (예: `AdministratorAccess`, `ReadOnlyAccess`) |
| **Customer Managed Policy** | 내가 만든 (account 안 공유) |
| **Inline Policy** | User/Role/Group에 *직접 부착* (재사용 X) |

→ 보통 *Customer Managed*가 권장 (재사용 + 가시성).

---

## 3. Role + STS — *임시 자격*

### STS (Security Token Service)
- *임시 credential 발급* service
- *AccessKey + SecretKey + SessionToken* 묶음
- *기본 1시간*, 최대 12시간 (Role 설정에 따라)

### Role의 *두 가지 정책*
1. **Trust Policy** (assume role policy): *누가 이 role assume 가능*
2. **Permission Policy**: *이 role이 무엇 할 수 있나*

### Trust Policy 예시 (EC2)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

→ EC2 인스턴스가 *이 role assume 가능*. *Instance Profile*로 attach.

### Trust Policy 예시 (Cross-account)
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::222222222222:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-secret-string"
    }
  }
}
```

→ *다른 account (222...)의 user가 *external ID와 함께* assume 가능*. SaaS 통합 흔한 패턴.

### Assume 흐름
```
User/EC2/K8s SA → sts:AssumeRole API 호출
                  (with trust policy 검증)
                ↓
              STS가 임시 credential 발급:
              { AccessKey, SecretKey, SessionToken }
                ↓
              그 credential로 다른 AWS API 호출
```

---

## 4. *OIDC Trust* — GitHub Actions·K8s·외부 IdP

### 발상
- 외부 IdP (GitHub, K8s OIDC, Okta)가 *짧은 JWT token 발급*
- *AWS STS가 그 token 검증* → 임시 credential 발급
- *정적 AccessKey 0개*

### GitHub Actions OIDC
[`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md) 참조.

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::111:oidc-provider/token.actions.githubusercontent.com"
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
}
```

→ *repo:my-org/my-repo의 main branch workflow*만 *이 Role assume 가능*.

### IRSA (K8s)
[`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) + [`k8s-security-basics/03`](../k8s-security-basics/03-identity-and-rbac.md) 참조.

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::111:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLE"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.region.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:default:my-app-sa"
    }
  }
}
```

→ *EKS의 default namespace `my-app-sa` ServiceAccount*만 assume.

### **`sub` claim이 trust boundary**
- 너무 헐겁게 (`*`) → *다른 repo·SA도 가로채기 가능*
- *정확히 명시* — `repo:org/repo:ref:refs/heads/main` 또는 `system:serviceaccount:ns:sa`

---

## 5. IAM Identity Center (SSO) — *사람용*

### 발상
- 옛: 각 AWS account에 IAM User → *비밀번호 + MFA 따로*
- 현대: *Identity Center가 *모든 account의 SSO 통합**

### 흐름
```
Okta / Azure AD / Identity Center 자체 → SAML/OIDC
       ↓
Identity Center
       ↓
사용자 portal → "어느 account의 어느 Permission Set" 선택
       ↓
임시 credentials (Role assume) 발급
       ↓
aws CLI / Console 사용
```

### Permission Set
- *Identity Center의 Role 개념*
- 사용자 그룹에 *Permission Set 부여* → 각 account에 *해당 IAM Role 자동 생성*

```bash
# aws CLI에서
aws sso login --profile prod
aws s3 ls --profile prod
```

→ 정적 AccessKey *없음*. *세션 만료 시 재로그인*.

### IAM User vs Identity Center 비교
| | IAM User | Identity Center |
|---|---|---|
| 자격 | 영구 AccessKey | 임시 (1~12h) |
| MFA | 옵션 | 강제 가능 |
| 멀티 account | 각 account 따로 | 한 곳에서 |
| Audit | account별 CloudTrail | 통합 CloudTrail |

→ **운영은 Identity Center로 전환**. IAM User는 *legacy + 일부 서비스 계정*만.

---

## 6. Service Control Policy (SCP) — *Organization 단위*

### AWS Organizations
- *여러 account 묶음* (조직)
- *master account가 member account 관리*
- *통합 billing, 보안 정책*

### SCP
- *Account 자체에 부여되는 *최대 권한 한도**
- 그 account의 *root user도 SCP 위반 못 함*
- 예: "*특정 region만 사용 가능*", "*specific IAM 변경 금지*"

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"]
    }
  }
}
```

→ *어느 user/role이든 ap-northeast-2/us-east-1 외 region에서 *어떤 action도 불가**.

### OU (Organizational Unit)
- account를 *환경별 묶음* (예: prod OU / dev OU)
- *OU에 SCP 부여* — 그 OU의 모든 account에 적용

### 운영 패턴
- **prod OU**: 엄격한 SCP (region 제한, *S3 public bucket 금지*, *KMS key 삭제 금지* 등)
- **dev OU**: 느슨한 SCP
- **sandbox OU**: 거의 제약 없음, 단 *budget 한도*

---

## 7. Resource-based Policy

### vs Identity-based Policy
- *Identity-based*: User/Role/Group에 *attached* — *그 신원이 무엇 할 수 있나*
- *Resource-based*: *자원 자체에 attached* — *누가 이 자원에 무엇 할 수 있나*

### 자원별 지원
| Service | Resource Policy |
|---|---|
| S3 | Bucket Policy |
| KMS | Key Policy (필수) |
| SQS | Queue Policy |
| SNS | Topic Policy |
| Lambda | Resource-based Policy (function policy) |
| ECR | Repository Policy |
| Secrets Manager | Resource Policy |

### S3 Bucket Policy 예시
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOtherAccount",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:role/some-role"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

→ *222 account의 some-role*이 *내 bucket read 가능*. *Cross-account 접근의 흔한 패턴*.

### Identity + Resource Policy의 *결합*
- *둘 다 Allow면* → 가능
- *어느 하나라도 Deny면* → 차단
- *Cross-account*는 *양쪽 모두 Allow 필요*

---

## 8. *Tag-based access control* (ABAC)

### 발상
- 자원과 user/role 둘 다에 *tag 부여*
- *tag 매칭*하면 권한
- *수많은 자원에 일일이 권한 부여 안 함*

### 예시
```json
{
  "Effect": "Allow",
  "Action": "ec2:StartInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```

→ *user의 Team tag와 인스턴스의 Team tag가 같으면* 시작 가능.

### 장점
- *scale* — 새 자원 추가 시 *권한 정책 변경 X*
- *tag 변경으로 권한 재구성*

### 단점
- *tag misuse* — *잘못된 tag = 잘못된 권한*
- *enforcement* — *tag 누락 시 policy로 강제* 필요 (예: 모든 자원에 Team tag 강제)

---

## 9. CloudTrail — *모든 IAM 사용 audit*

### 무엇을 기록
- 모든 *AWS API 호출* (누가, 언제, 무엇)
- *Management events* (자원 변경) 자동 활성
- *Data events* (S3 object read 등) 명시 활성

### 활용
- *권한 misuse 감지* — 비정상 패턴 (야간, 새 IP, 새 region)
- *사고 조사* — 누가 자원 삭제했나
- *컴플라이언스* — *모든 변경 추적*

### 운영
- *Organization Trail* — *전 account 통합 logging*
- *S3에 저장 + KMS 암호화 + bucket lock* — *변조 방지*
- *CloudTrail Lake* — SQL query (또는 Athena 통합)
- *SIEM 통합* (Splunk, Datadog) — 알람

---

## 10. 자주 헷갈리는 것

### 10-1. *Policy 평가는 *deny가 항상 이김**
- N개 policy 합쳐도 *하나의 deny면 거부*
- *SCP의 deny*도 마찬가지 — *root user도 통과 못 함*

### 10-2. *정적 AccessKey 사용*
- 침해 시 *영구 위험*
- *모든 자동화는 OIDC + IAM Role*로 — 정적 키 0

### 10-3. *AssumeRole loop*
- Role A의 trust가 Role B, Role B의 trust가 Role A → *infinite assume*
- AWS가 *자동 차단*하나 *디자인 자체로 회피*

### 10-4. *Wildcard `*` 남발*
- `s3:*` 또는 `Resource: "*"` — *공격 표면 ↑*
- *정확한 action·resource ARN 명시*

### 10-5. *OIDC trust의 *sub claim 너무 헐겁게**
- `repo:my-org/*` — 같은 org의 *모든 repo가 assume 가능*
- *정확히* `repo:my-org/specific-repo:ref:refs/heads/main`

### 10-6. *Cross-account 시 *Identity OR Resource policy 한쪽만 Allow**
- *Cross-account는 양쪽 모두 Allow 필요*
- 한쪽만 OK면 *거부*

---

## 🎤 면접 빈출 Q&A

### Q1. IAM User와 Role 차이는?
> **User**: *영구 신원* (사람·앱), *정적 AccessKey 가능*. **Role**: *임시 자격*, *누군가가 assume*해서 사용, STS가 *임시 credential 발급* (1~12시간). **모든 곳에 Role + STS 권장** — 정적 키 사고 회피. EC2는 Instance Profile, K8s Pod은 IRSA, CI는 OIDC trust. User는 *legacy + 일부 service account*에만.

### Q2. IAM Policy 평가는?
> 4단계: (1) **Default = Deny**, (2) *어떤 policy든 Allow* → 통과, (3) *어떤 policy든 Explicit Deny* → 거부 (다른 Allow 무시), (4) SCP·Permission Boundary 등 *상위 한도도 적용*. **Deny가 항상 이김** — 정책 N개 합쳐도 *하나의 deny면 거부*. SCP의 deny는 *root user도 통과 못 함*.

### Q3. IRSA의 *trust policy* 보안 핵심?
> `sub` claim 조건이 *경계*. 너무 헐겁게 (`*`)면 *다른 SA가 가로채기 가능*. **`system:serviceaccount:namespace:sa-name` 단위 정확히**. `aud` claim도 정확히 (`sts.amazonaws.com`). *namespace·SA name 오타*가 흔한 함정. **trust 조건 = 보안 boundary**.

### Q4. IAM Identity Center가 *왜 권장*?
> *모든 account의 SSO 통합*. 옛 IAM User는 *각 account 따로 + 비밀번호·MFA 따로*. Identity Center는 *Okta/AAD/자체 SSO* → *Permission Set으로 account별 Role 자동* → *임시 credential* (정적 키 0). *MFA 강제·통합 audit*. **운영은 Identity Center 전환**, IAM User는 legacy.

### Q5. SCP (Service Control Policy)는?
> *AWS Organizations의 account 단위 *최대 권한 한도**. 그 account의 *root user도 SCP 위반 못 함*. 예: "특정 region만 허용", "S3 public bucket 금지", "KMS key 삭제 금지". OU별로 부여 — *prod OU는 엄격, dev OU는 느슨*. **컴플라이언스의 *enforcement point***.

### Q6. Cross-account 자원 접근?
> *Identity-based + Resource-based policy 양쪽 모두 Allow 필요*. (1) 자원 owner account의 *Resource Policy* (S3 Bucket Policy 등)에 *다른 account principal 허용*, (2) 그 다른 account의 *IAM Role*에 *해당 자원 action 허용*. Trust policy에 *external ID*도 흔함 (SaaS 통합 보안).

### Q7. CloudTrail이 *왜 필수*?
> *모든 AWS API 호출 audit*. *누가 언제 무엇 변경*. **컴플라이언스 (모든 변경 추적)**, **사고 조사** (누가 자원 삭제), **권한 misuse 감지** (비정상 패턴). 운영: *Organization Trail* (전 account 통합) + *S3 + KMS + bucket lock* (변조 방지) + *SIEM 통합* (Splunk·Datadog). *Management events 자동*, *data events (S3 object read)는 명시 활성*.

---

## 🔗 Cross-reference

- **K8s IRSA** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), [`k8s-security-basics/03`](../k8s-security-basics/03-identity-and-rbac.md)
- **GitHub Actions OIDC** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **VPC + SG** → [02-vpc-and-networking.md](02-vpc-and-networking.md)
- **KMS Key Policy** → [05-secrets-and-kms.md](05-secrets-and-kms.md)
- **Terraform IAM 자원** → [`terraform-basics/01`](../terraform-basics/01-providers-and-resources.md)

---

## 📝 3줄 요약

1. *User (영구) vs Role (임시 STS)*. **모든 곳에 Role + OIDC trust로 정적 키 0**. Identity Center로 *사람 SSO 통합*.
2. Policy 평가 — **Deny가 항상 이김**. *정확한 action·resource·condition 명시* (wildcard `*` 피함). **OIDC trust의 sub claim이 보안 boundary**.
3. *SCP*로 *account 단위 권한 한도*. *CloudTrail*로 *모든 API audit*. *Cross-account는 양쪽 policy 모두 Allow*.

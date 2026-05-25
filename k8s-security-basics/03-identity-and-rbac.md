# 03 — Identity & RBAC: ServiceAccount · IRSA · Workload Identity · Audit

> **공식 근거:** [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/), [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html), [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
>
> **선행:** [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) (RBAC 기본)

---

## 🎯 한 문장

> **K8s에선 *RBAC*이 *cluster 내 API 권한*, *IRSA / Workload Identity*가 *Pod → cloud API 권한*. **정적 키 0개**가 현대 표준. *Audit log*로 *누가 무엇 했는지 추적*. Multi-tenancy = *RBAC + namespace + NetworkPolicy + PSS + Quota* 종합.**

---

## 1. RBAC 복습 (요약)

[`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) 참조. 핵심:

### 4가지 자원
- **Role**: namespace 안 권한
- **ClusterRole**: cluster-wide 또는 namespace 횡단
- **RoleBinding**: Role을 namespace의 누구에게
- **ClusterRoleBinding**: ClusterRole을 cluster 전체로

### 주체 (subjects)
- **User**: 사람
- **Group**: User 묶음
- **ServiceAccount**: Pod의 신원

### 동사 (verbs)
- `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### 디버깅
```bash
kubectl auth can-i create pods --as=alice
kubectl auth can-i --list --as=system:serviceaccount:dev:my-sa
```

---

## 2. ServiceAccount 깊이

### 자동 생성·자동 마운트
- 각 namespace에 *`default` ServiceAccount 자동*
- Pod 명시 안 하면 *`default` SA 사용*
- Pod에 *SA token 자동 mount* (`/var/run/secrets/kubernetes.io/serviceaccount/`)

### `automountServiceAccountToken: false`
```yaml
spec:
  automountServiceAccountToken: false
```
- 앱이 K8s API 안 부르면 *token 노출 불필요*
- 보안 강화

### Projected SA Token (1.22+) — *현대 표준*
- *짧은 유효기간* (예: 1시간)
- *audience 명시* — 특정 service에만 유효
- *자동 회전* (만료 전 갱신)

### 옛 legacy token (1.24부터 자동 발급 X)
- *영구 token* — Secret으로 저장
- *유출 시 영구 위험*
- → *projected token으로 대체*

---

## 3. *IRSA* (IAM Roles for Service Accounts) — AWS EKS

### 발상
- Pod이 *AWS API* (S3, DynamoDB, SQS 등)에 접근 필요
- 옛 방식: *EC2 instance profile* — 같은 노드의 *모든 Pod이 같은 권한*
- → *Pod별 최소 권한* 어떻게?

### IRSA 흐름
```
1. EKS 클러스터에 *OIDC Provider 활성화*
2. AWS IAM에 *IAM Role 생성*
   - Trust policy: "이 OIDC issuer의 *특정 SA*만 assume 허용"
3. K8s ServiceAccount에 *annotation*
   - eks.amazonaws.com/role-arn: arn:aws:iam::...:role/myrole
4. Pod 시작 시 EKS Admission이 *환경변수 + projected SA token mount* 자동 주입
5. AWS SDK가 *그 token으로 STS AssumeRoleWithWebIdentity* 호출
6. STS가 *trust policy 검증 후 임시 credentials* 발급 (1시간)
7. SDK가 그 credentials로 AWS API 호출
```

### Setup 예시

**OIDC Provider** (Terraform)
```hcl
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

**IAM Role with trust**
```hcl
data "aws_iam_policy_document" "trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
    }
    
    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values   = ["system:serviceaccount:default:my-app"]   # ← namespace:SA name
    }
  }
}

resource "aws_iam_role" "app" {
  name               = "my-app-role"
  assume_role_policy = data.aws_iam_policy_document.trust.json
}
```

**ServiceAccount with annotation**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

**Pod에서 사용**
```yaml
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: myapp
    # AWS SDK가 환경변수 (AWS_WEB_IDENTITY_TOKEN_FILE 등) 자동 사용
```

→ **정적 AccessKey 0개**. Pod이 *자기 SA의 IAM Role만 사용*.

### 보안 함의
- *Trust policy의 *sub claim 조건*이 *경계** — *너무 헐겁게 (`*`)* 두면 *다른 SA가 가로채기 가능*
- *namespace:SA 단위로 정확히* 명시

---

## 4. *Workload Identity* (GCP, Azure)

### GCP Workload Identity
- IRSA와 *같은 발상*
- K8s SA ↔ GCP Service Account 매핑
- *workloadIdentityConfig* 활성화

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    iam.gke.io/gcp-service-account: my-gcp-sa@project.iam.gserviceaccount.com
```

### Azure Workload Identity
- AKS의 OIDC + Azure AD
- 비슷한 흐름

→ *클라우드별 다른 구현, 같은 원리* — *OIDC trust + 단기 token + 정적 키 0*.

---

## 5. K8s 외부 IdP (OIDC) — *사람 user*

### 발상
- *K8s는 사람 user DB 없음* — 외부 IdP 의존
- 옛: client cert (mTLS)
- 현대: *OIDC* (Okta, Google, Azure AD)

### Setup
```yaml
# /etc/kubernetes/config.yaml (kube-apiserver flags)
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=...
--oidc-username-claim=email
--oidc-groups-claim=groups
```

### kubectl에서
- `kubectl` config에 OIDC token 설정
- `kubectl oidc-login` plugin
- *user의 email·groups가 RBAC subjects로*

### EKS·GKE·AKS의 IAM 통합
- EKS: *AWS IAM Identity Center 또는 IAM user → aws-auth ConfigMap 매핑*
- GKE: *Google Group → K8s RBAC*
- AKS: *Azure AD*

→ *사람은 IdP를 통한 인증, K8s는 RBAC으로 권한 결정*.

---

## 6. Audit Logging — *누가 무엇 했나*

### 정책 파일
```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse                  # 요청 + 응답 모두 로깅
  resources:
  - group: ""
    resources: ["secrets"]
  
- level: Metadata                         # metadata만 (효율)
  verbs: ["get", "list", "watch"]
  
- level: Request
  resources:
  - group: ""
    resources: ["*"]

- level: None                             # 로깅 안 함
  users: ["system:serviceaccount:kube-system:*"]
```

### Level
- **None**: 로깅 X
- **Metadata**: *요청 메타만* (user·verb·resource·timestamp)
- **Request**: + *요청 body*
- **RequestResponse**: + *응답 body*

### kube-apiserver 설정
```yaml
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

### Audit log 활용
- *컴플라이언스 audit* — 누가 prod secret 접근했나
- *사고 조사* — incident 시 *직전 변경 추적*
- *권한 misuse 감지* — 비정상 패턴 (예: 야간 secret list)
- *SIEM 통합* (Splunk, Sumo Logic, Datadog) — *중앙 검색·알람*

### 매니지드 K8s
- **EKS**: CloudWatch Logs로 *audit/authenticator/controllerManager/scheduler 등 로깅 활성화*
- **GKE**: Cloud Logging 자동
- **AKS**: Log Analytics

---

## 7. *Multi-Tenancy* — *cluster를 여러 팀이*

### Soft Multi-Tenancy
- *namespace 분리*
- *RBAC + NetworkPolicy + PSS + ResourceQuota*
- *같은 cluster, 같은 노드 가능*

### Hard Multi-Tenancy
- *클러스터 자체 분리* (팀별 클러스터)
- *완전 격리*
- *비용·운영 부담 ↑*

### Soft Multi-Tenancy의 필수 요소
| Layer | 도구 |
|---|---|
| **API 권한** | RBAC (namespace-scoped Role) |
| **자원 제한** | ResourceQuota + LimitRange |
| **네트워크 격리** | NetworkPolicy (namespace 간 차단) |
| **Pod 권한** | PSS (restricted profile) |
| **노드 격리** | taint + toleration, runtimeClass (Kata) |
| **Audit** | namespace별 audit logging |

### ResourceQuota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "100"
    persistentvolumeclaims: "20"
    requests.storage: "1Ti"
```

→ namespace의 *총 자원 한도*. 초과 자원 생성 *차단*.

### LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: tenant-a
spec:
  limits:
  - default:                # 명시 안 한 container의 limits
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

→ *Pod이 requests/limits 명시 안 해도 자동 부여*. *bare Pod도 자원 보호*.

---

## 8. *RBAC 운영 best practices*

### 1. *Default SA에 권한 부여 금지*
- 모든 새 Pod이 자동 받음 → *권한 폭발*
- *명시적 SA 만들고 그것에만*

### 2. *Wildcard 피하기*
```yaml
# 나쁨
verbs: ["*"]
resources: ["*"]

# 좋음
verbs: ["get", "list"]
resources: ["pods"]
```

### 3. *ClusterRoleBinding 신중*
- cluster 전체 권한 — 한 번 실수면 영향 큼
- *대부분은 RoleBinding으로 namespace-scoped*

### 4. *Built-in ClusterRole 활용*
- `view` (read-only), `edit` (write 가능, RBAC 제외), `admin` (RBAC 포함, namespace 안 모든), `cluster-admin` (전체)
- 새로 만들기 전 *이미 있는 것 활용*

### 5. *Aggregated ClusterRoles*
- 같은 *label로 자동 합쳐지는* ClusterRole
- *모듈식 권한 조립*

```yaml
kind: ClusterRole
metadata:
  name: monitoring-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheuses", "servicemonitors"]
  verbs: ["get", "list", "watch"]
```

→ 기본 `view` ClusterRole이 *자동으로 이 권한 흡수*.

### 6. *Group 활용*
- *팀별 group 부여 (OIDC)* → *그 group에 권한*
- 사람 단위 RoleBinding은 *유지 보수 지옥*

### 7. *RBAC 자체 변경 권한*도 최소
- *Role/RoleBinding 만들 권한* = *권한 escalation 가능*
- *cluster-admin만* RBAC 자원 변경

---

## 9. *RBAC 디버깅*

### `kubectl auth can-i`
```bash
# 내 권한
kubectl auth can-i create pods
kubectl auth can-i delete deployments

# 다른 SA의 권한
kubectl auth can-i create deployments \
  --as=system:serviceaccount:dev:my-sa \
  --namespace=dev

# 전체 가능한 verb 매트릭스
kubectl auth can-i --list --as=system:serviceaccount:dev:my-sa --namespace=dev
```

### Binding 찾기
```bash
# SA의 bindings
kubectl get rolebindings,clusterrolebindings -A \
  -o jsonpath='{range .items[?(@.subjects[*].name=="my-sa")]}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}'

# 또는 도구: rbac-lookup, rakkess
rbac-lookup my-sa --kind serviceaccount
rakkess --as=system:serviceaccount:dev:my-sa
```

### `audit2rbac` — *실제 사용된 권한*에서 *최소 RBAC 생성*
- audit log 분석 → *실제 호출된 API* → *그것만 허용하는 RBAC 자동 생성*
- *과도한 권한 자동 trim*에 유용

---

## 10. 자주 헷갈리는 것

### 10-1. *IRSA가 *적용 안 됨**
- ServiceAccount annotation 누락
- Pod이 *그 SA 사용 안 함*
- IAM Role trust policy의 *sub claim 오타* (`namespace:sa-name` 정확히)
- OIDC provider 미설정

### 10-2. *Audit log 너무 커짐*
- *모든 자원 RequestResponse* level — *수십 GB/일*
- *전략*: 중요 자원 (Secret, RBAC)만 RequestResponse, 나머지 Metadata, system SA 제외

### 10-3. *projected SA token 재사용*
- token은 *짧은 유효기간 + 자동 회전*
- 앱이 *시작 시 한 번 읽고 캐시*하면 만료 후 401
- → *주기적 재읽기* (대부분 SDK 자동 처리)

### 10-4. *kubectl 사용자가 *cluster-admin**
- 모든 권한 = 한 번 실수면 cluster 망함
- *명시적 namespace-scoped Role* + *cluster-admin은 별도 binding으로 minimal*

### 10-5. *Multi-tenancy에서 *node 격리 안 함**
- RBAC·Network·Quota 다 있어도 *같은 노드에 다른 tenant Pod 있으면 *kernel exploit으로 escape 가능*
- 진짜 격리는 *Kata Containers* 또는 *별도 노드 그룹 + taint*

### 10-6. *EKS aws-auth ConfigMap*
- IAM user/role을 K8s user/group에 매핑
- *수동 관리는 위험* — *Terraform 또는 EKS Access Entries (1.27+)*

---

## 🎤 면접 빈출 Q&A

### Q1. IRSA의 동작 원리는?
> Pod이 *K8s ServiceAccount → AWS IAM Role 받는 방식*. 흐름: (1) EKS에 OIDC Provider 활성화, (2) IAM Role의 trust policy에 *"이 OIDC issuer의 특정 SA"*, (3) K8s SA에 *role ARN annotation*, (4) Pod 시작 시 EKS Admission이 *projected token + 환경변수 자동 주입*, (5) AWS SDK가 *STS AssumeRoleWithWebIdentity* 호출, (6) STS가 trust 검증 후 *임시 credentials* (1h). **정적 AccessKey 0개**.

### Q2. IRSA의 trust policy에서 *주의해야 할 것*?
> `sub` claim 조건이 *경계*. `repo:my-org/*` 같이 *너무 헐겁게* 두면 *다른 SA가 가로채기 가능*. **`system:serviceaccount:namespace:sa-name` 단위로 정확히**. namespace·SA name 오타도 흔한 함정. `aud` claim도 `sts.amazonaws.com` 정확히.

### Q3. Multi-tenancy 격리에 필요한 6요소?
> (1) **RBAC** (namespace-scoped Role), (2) **ResourceQuota + LimitRange** (자원 한도), (3) **NetworkPolicy** (namespace 간 차단), (4) **PSS** (restricted profile), (5) **Audit logging** (namespace별), (6) **노드 격리** (taint+toleration + Kata Containers for hard isolation). **6 다 같이 적용**해야 의미. *어느 하나 빠지면 escape 가능*.

### Q4. K8s Audit logging의 4 level?
> **None** (로깅 X), **Metadata** (user/verb/resource/timestamp), **Request** (+ 요청 body), **RequestResponse** (+ 응답 body). 전략: *중요 자원 (Secret, RBAC)만 RequestResponse*, *읽기 (get/list)는 Metadata*, *system SA 제외*. SIEM (Splunk·Datadog) 통합으로 *중앙 검색·알람*.

### Q5. RBAC 디버깅 도구?
> **`kubectl auth can-i`** — 내·다른 SA의 권한 확인. `--list --as=...` 로 *전체 verb 매트릭스*. **rbac-lookup / rakkess** — *SA의 모든 binding 찾기*. **audit2rbac** — *실제 사용된 권한에서 *최소 RBAC 자동 생성** (audit log 분석). RBAC 사고 시 *first step은 `can-i`*.

### Q6. ServiceAccount의 *legacy token vs projected token*?
> **Legacy** (1.24까지 자동 발급, 이후 X): *영구 token*, Secret으로 저장, 유출 시 영구 위험. **Projected** (1.22+ 표준): *짧은 유효기간 (1h)*, *audience 명시*, *자동 회전*. Pod의 `/var/run/secrets/...` 에 mount. **앱은 *주기적으로 token 재읽기*** (대부분 SDK 자동).

### Q7. K8s 사람 user는 *어떻게 인증*?
> K8s는 *user DB 없음* — 외부 IdP. 옛: *client cert (mTLS)*. 현대: **OIDC** (Okta, Google, Azure AD). kube-apiserver의 `--oidc-*` flags 설정. EKS/GKE/AKS는 *cloud IAM 통합* — EKS aws-auth ConfigMap (또는 1.27+ Access Entries), GKE는 Google Group, AKS는 Azure AD. **사람·service 둘 다 *정적 키 0개***이 현대 표준.

---

## 🔗 Cross-reference

- **RBAC 기본 (Role/Binding 4자원)** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)
- **API server 3단계 (AuthN/AuthZ/Admission)** → [`kubernetes-basics/01`](../kubernetes-basics/01-architecture.md)
- **OIDC trust = GitHub Actions OIDC와 같은 원리** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **NetworkPolicy (multi-tenancy 필수)** → [02-network-policy.md](02-network-policy.md)
- **PSS (multi-tenancy 필수)** → [01-pod-security.md](01-pod-security.md)
- **Secret 관리** → [05-secrets-and-encryption.md](05-secrets-and-encryption.md)
- **AWS IAM·STS·KMS** → `aws-basics/` (예정)

---

## 📝 3줄 요약

1. *RBAC = cluster 내 API 권한*, *IRSA/Workload Identity = Pod → cloud API 권한*. **정적 키 0개** = 현대 표준. *OIDC trust*가 양쪽 베이스.
2. *Audit logging* (4 level)으로 *누가 무엇 했나 추적*. SIEM 통합으로 *컴플라이언스·사고 조사·misuse 감지*.
3. *Multi-tenancy = RBAC + Quota + NetworkPolicy + PSS + Audit + Node 격리* 6요소 종합. *어느 하나 빠지면 escape 가능*.

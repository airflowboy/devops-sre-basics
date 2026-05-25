# 07 — RBAC와 Security 입문

> **공식 근거:** [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/), [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
>
> **선행:** [01-architecture.md](01-architecture.md) (API server 흐름), [06-api-and-controllers.md](06-api-and-controllers.md) (Admission)
>
> *깊이 보안은 [`k8s-security-basics/`](예정) 에서 다룸. 이 파일은 *입문*.*

---

## 🎯 한 문장

> **모든 API 요청은 *Authentication (누구?)* → *Authorization (RBAC 등, 무엇을 할 수 있어?)* → *Admission (정책 위반?)* 3단계를 거친다. *RBAC 기본 단위는 ServiceAccount + Role/ClusterRole + Binding*. Pod 보안의 기본 강제는 *Pod Security Standards (PSS)*.**

---

## 1. 3단계 흐름 — Auth*N* / Auth*Z* / Admission

```
요청 → ① Authentication: "누구야?"
       (mTLS / Bearer token / OIDC / Webhook)
       → identity (User / Group / ServiceAccount)

     → ② Authorization: "이거 할 권한 있어?"
       (RBAC / ABAC / Webhook / Node)
       → allow / deny

     → ③ Admission: "정책 위반 아냐?"
       (Mutating + Validating)
       → 변형 / 거부

     → ④ etcd 저장
```

→ **한 단계라도 거부하면 etcd에 안 박힘.**

---

## 2. Authentication — *누구인지 확인*

K8s는 *사용자 DB가 없음*. 외부 신원 정보에 의존.

### 신원 종류

| 종류 | 용도 | 인증 방법 |
|---|---|---|
| **User** | *사람*. 외부에서 발급된 신원 | Client cert (mTLS), OIDC token, static token 등 |
| **Group** | User들의 묶음 | OIDC claim 등에서 |
| **ServiceAccount** | *Pod이 K8s API에 접근할 때* | JWT token (자동 발급) |

### 인증 메커니즘

| 방법 | 비고 |
|---|---|
| **X.509 client cert** | mTLS. kubeadm 기본 (kubelet, controller-manager 등). 갱신·revoke 어려움 |
| **Bearer Token (정적)** | 옛 방식, 비추 |
| **ServiceAccount Token (JWT)** | Pod의 기본. 자동 mount |
| **OIDC** | Okta, Auth0, Google, Azure AD 등 통합 |
| **Webhook** | 임의의 외부 IdP |

### ServiceAccount Token의 진화
- **legacy** (자동 mounted, *영구 token*) → 1.24부터 *기본 X*
- **Projected ServiceAccount Token** (1.22+) — *짧은 유효기간*, *audience 명시*, *자동 회전*
  ```yaml
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: vault
          expirationSeconds: 3600
  ```

### *EKS의 IRSA* — Pod이 AWS IAM Role 받기
- K8s ServiceAccount에 IAM Role ARN annotation
- Pod이 *Projected SA Token*을 *AWS STS에 제시* → *임시 AWS credentials* 받음
- *정적 AccessKey 0개*
- → `aws-basics/` (예정) 에서 더.

---

## 3. Authorization — *권한 확인*

### 인가 모드 (`--authorization-mode`)
순서대로 평가, *하나라도 allow면 통과*:
- **Node** — kubelet 전용 (자기 Pod·Secret·ConfigMap만 접근)
- **RBAC** — *표준*
- **ABAC** — 옛날 (deprecated)
- **Webhook** — 외부 인가 서비스
- **AlwaysAllow** / **AlwaysDeny** — 테스트용

→ 운영은 **Node + RBAC** 조합.

---

## 4. RBAC — 핵심 4가지 자원

### 4-1. Role (namespace-scoped)
*한 namespace 안에서 무슨 자원·verb 허용?*
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### 4-2. ClusterRole (cluster-scoped)
*클러스터 전체 또는 namespace 횡단 권한.*
```yaml
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

### 4-3. RoleBinding — *Role을 누구에게*
```yaml
kind: RoleBinding
metadata:
  namespace: default
  name: read-pods
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4-4. ClusterRoleBinding — *ClusterRole을 누구에게 (전 클러스터)*
```yaml
kind: ClusterRoleBinding
metadata:
  name: read-secrets-cluster
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 매트릭스
| | Role (namespace) | ClusterRole (cluster) |
|---|---|---|
| RoleBinding (namespace) | *그 ns에서 그 권한* | *그 ns에서 ClusterRole의 권한* |
| ClusterRoleBinding (cluster) | ❌ (의미 없음) | *전 클러스터에서 그 권한* |

→ ClusterRole + RoleBinding 조합이 **"공통 권한 정의 + 여러 namespace에 각각 부여"** 패턴.

---

## 5. *RBAC 점검 — `kubectl auth can-i`*

```bash
# 내가 이거 할 수 있어?
kubectl auth can-i create pods

# 특정 사용자/SA가 할 수 있어?
kubectl auth can-i create deployments \
  --as=system:serviceaccount:dev:my-sa \
  --namespace=dev

# 가능한 모든 verb 보기
kubectl auth can-i --list --as=system:serviceaccount:dev:my-sa
```

→ *디버깅 정석*. RBAC 문제 의심 시 첫 명령.

---

## 6. ServiceAccount — *Pod의 신원*

### 자동 생성
- 각 namespace에 `default` SA가 *자동 생성*
- Pod의 `spec.serviceAccountName` 안 명시하면 *default 사용*
- *default SA는 권한 거의 없음* (좋은 일)

### Pod에서 사용
```yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa     # ← 명시
  containers:
  - name: app
    image: myapp
```

### Pod 안에서 SA token 활용
- 자동으로 `/var/run/secrets/kubernetes.io/serviceaccount/` 에 마운트
  - `token` — JWT
  - `ca.crt` — API server CA
  - `namespace` — 자기 namespace
- 앱이 K8s API 호출 시 *이 token으로 인증*

### *automountServiceAccountToken: false* — token 자동 마운트 안 함
- 앱이 K8s API 안 부르면 *token mount 필요 없음*
- 보안 권장 (불필요한 신원 노출 X)
```yaml
spec:
  automountServiceAccountToken: false
```

---

## 7. Pod Security Standards (PSS) — *Pod 보안의 기본*

K8s 1.25+에서 `PodSecurity` admission plugin이 *내장*. 3단계 profile:

| Profile | 의미 |
|---|---|
| **privileged** | 거의 제한 없음 |
| **baseline** | 알려진 *위험* 차단 (hostPath, hostNetwork, hostPID, privileged 등) |
| **restricted** | *모범 보안 강제* — runAsNonRoot, cap-drop ALL, seccomp, RO root fs 권장 |

### Namespace에 라벨로 적용
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

`enforce` — *위반 시 Pod 생성 거부*
`audit` — *위반 시 audit log*
`warn` — *위반 시 사용자에게 warning*

### restricted profile이 강제하는 것
- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- *모든 capability drop ALL* + 추가 시 `NET_BIND_SERVICE`만
- *seccompProfile: RuntimeDefault*
- *읽기 전용 root fs 권장* (강제 X)
- *hostPath/hostNetwork/hostPID/hostIPC 금지*
- *privileged container 금지*

→ [`docker-basics/06`](../docker-basics/06-security.md) 의 *최소 권한 securityContext* 와 동일. *K8s가 namespace 단위로 강제*.

→ 더 깊이는 `k8s-security-basics/` (예정).

---

## 8. *etcd Encryption at Rest* — Secret 진짜 암호화

기본 etcd 저장은 *plaintext*. Secret base64는 *인코딩일 뿐*. 진짜 암호화는 *etcd encryption at rest*:

```yaml
# EncryptionConfiguration (kube-apiserver --encryption-provider-config)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: [secrets]
  providers:
  - kms:
      name: aws-kms
      endpoint: unix:///tmp/kms.sock
      ...
  - identity: {}    # fallback (no encryption)
```

- KMS provider 권장 (AWS KMS / GCP KMS / Vault KMS)
- *기존 secret도 *재암호화* 필요*: `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`

---

## 9. 흔한 RBAC 모범

### 9-1. *default SA에 권한 부여 금지*
- 모든 새 Pod이 *자동으로* 그 권한 받음 → 의도치 않은 권한 폭발
- 항상 *명시적 SA* 만들고 *그것에만* 부여

### 9-2. *ClusterRoleBinding 신중하게*
- 클러스터 전체 권한 부여 → 한 번 실수면 *전체 영향*
- *대부분의 권한은 namespace-scoped로* 충분

### 9-3. *aggregated ClusterRoles*
- K8s 내장 `view`, `edit`, `admin`, `cluster-admin` 활용
- 또는 *aggregation rule*로 *자동 합쳐지는 ClusterRole* 만들기

### 9-4. *Wildcard 피하기*
```yaml
# 나쁨
verbs: ["*"]
resources: ["*"]

# 좋음
verbs: ["get", "list", "watch"]
resources: ["pods", "services"]
```

### 9-5. *RBAC 자체 변경 권한*도 *최소화*
- *Role/RoleBinding 만들 권한*이 있으면 *권한 escalation 가능*
- *cluster-admin 사람만* RBAC 자원 변경 가능하도록

---

## 10. 자주 헷갈리는 것

### 10-1. *Role과 ClusterRole의 차이*는 *scope*만이 아님
- ClusterRole은 *cluster-scoped 자원* (Node, PV, Namespace 자체)도 *대상으로 가능*. Role은 *namespace 자원만*.
- *non-resource URL* (`/healthz`, `/metrics`)도 ClusterRole로만.

### 10-2. *RoleBinding이 ClusterRole 참조*
- 가능. 의미: *ClusterRole에 정의된 권한*을 *그 RoleBinding의 namespace*에서만 부여.
- 흔한 패턴: 공용 `pod-reader` ClusterRole + 각 namespace에서 RoleBinding으로 부여.

### 10-3. *system:masters group* = *cluster-admin*
- kubeadm의 *admin.conf*가 이 그룹에 속함. *모든 권한*.
- 잘못 다루면 *전체 클러스터 위험*

### 10-4. *ServiceAccount token 갱신*
- 옛 *legacy SA token*은 *영구* — secret으로 저장
- *Projected token*은 자동 회전 + audience 명시
- 1.24부터 *legacy 자동 발급 안 함* — 명시적으로 만들어야

### 10-5. *Pod이 K8s API 호출하는데 401·403*
- `kubectl auth can-i ... --as=system:serviceaccount:ns:sa-name` 로 확인
- 보통: *SA에 Role/RoleBinding 없음*, 또는 *automountServiceAccountToken: false 인데 인증 시도*

---

## 🎤 면접 빈출 Q&A

### Q1. K8s 요청의 3단계 처리는?
> **Authentication** (누구야? — mTLS·OIDC·SA token) → **Authorization** (이거 할 수 있어? — 보통 RBAC) → **Admission** (정책 위반? — Mutating·Validating). 한 단계라도 거부면 *etcd에 안 박힘*.

### Q2. Role / ClusterRole / RoleBinding / ClusterRoleBinding 매트릭스?
> Role = *namespace-scoped 권한 정의*. ClusterRole = *cluster-scoped 또는 namespace 횡단 권한 정의*. RoleBinding = *Role을 namespace의 누구에게*. ClusterRoleBinding = *ClusterRole을 cluster 전체로 누구에게*. *RoleBinding이 ClusterRole 참조*도 가능 (특정 ns에서만 적용).

### Q3. ServiceAccount는 무엇이고 default SA와의 관계는?
> ServiceAccount = *Pod의 신원*. K8s API 호출 시 *그 SA의 token으로 인증*. namespace마다 *default SA가 자동 생성* — Pod이 명시 안 하면 default 사용. *default SA에 권한 부여하면 모든 Pod이 받음* → 위험. 명시적 SA 만들고 그것에만 부여.

### Q4. Pod Security Standards `restricted` profile은 무엇을 강제?
> *runAsNonRoot*, *allowPrivilegeEscalation: false*, *cap-drop ALL + 최소만 add*, *seccompProfile RuntimeDefault*, *host namespace·privileged 금지*. K8s 1.25+ 내장 `PodSecurity` admission plugin이 *namespace 라벨로 적용*. baseline·restricted·privileged 3단계.

### Q5. Secret이 etcd에서 *진짜 안전*하려면?
> base64는 *인코딩일 뿐*. 진짜 보안은 *etcd encryption at rest* 활성화. EncryptionConfiguration + KMS provider (AWS KMS / Vault / GCP KMS). *기존 secret*도 *재암호화 필요* (replace). 추가로 *RBAC 빡세게* + *외부 secret manager 통합* (External Secrets Operator).

### Q6. EKS의 IRSA란?
> Pod이 *K8s ServiceAccount → AWS IAM Role*을 받는 방식. SA에 *Role ARN annotation*. Pod의 *Projected SA Token*을 *AWS STS의 AssumeRoleWithWebIdentity* 에 제시 → *임시 AWS credentials*. *정적 AccessKey 0개*. SA마다 다른 IAM Role 부여 가능 (least privilege per workload).

### Q7. RBAC 디버깅 절차?
> `kubectl auth can-i {verb} {resource} --as=system:serviceaccount:{ns}:{sa}` 로 *현재 권한* 확인. `--list` 옵션으로 *전체 가능한 verb 매트릭스*. SA의 binding 찾기: `kubectl get rolebindings,clusterrolebindings -A -o yaml | grep {sa-name}`. *system:* group들 권한도 의식.

---

## 🔗 Cross-reference

- **3단계 처리 흐름** → [01-architecture.md](01-architecture.md)
- **Admission Controllers** → [06-api-and-controllers.md](06-api-and-controllers.md)
- **컨테이너 capability·seccomp·USER** → [`docker-basics/06`](../docker-basics/06-security.md)
- **NetworkPolicy, PSS 깊이, Trivy, Cosign, Sigstore policy-controller** → `k8s-security-basics/` (예정)
- **IRSA, IAM, KMS, AWS Secrets Manager** → `aws-basics/` (예정)
- **External Secrets Operator, Vault** → `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *모든 API 요청 = AuthN → AuthZ → Admission*. RBAC 기본 단위 = ServiceAccount + Role/ClusterRole + Binding.
2. *default SA에 권한 부여 금지*. *명시적 SA + 최소 권한*. `kubectl auth can-i`로 RBAC 디버깅.
3. *Pod Security Standards*가 K8s 내장 Pod 보안 강제 (1.25+). *etcd encryption at rest*로 Secret 진짜 암호화. EKS는 *IRSA*로 정적 키 0개.

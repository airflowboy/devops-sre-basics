# 05 — Secrets & Encryption: etcd encryption · External Secrets · Vault · Sealed Secrets

> **공식 근거:** [Encryption at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/), [External Secrets Operator](https://external-secrets.io/), [Sealed Secrets](https://sealed-secrets.netlify.app/), [SOPS](https://github.com/getsops/sops)

---

## 🎯 한 문장

> **K8s Secret의 *base64는 암호화 아님*. 진짜 보안 = (1) **etcd encryption at rest (KMS)** + (2) **RBAC 빡세게** + (3) **외부 secret manager 통합** (Vault·AWS Secrets Manager·External Secrets Operator). GitOps 친화는 *Sealed Secrets / SOPS*. *Pod 안엔 *언제든 회전 가능한 단기 token*만***.

---

## 1. K8s Secret의 *실체*

### Secret resource
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: bXl1c2Vy        # base64
  password: bXlwYXNzd29yZA==
```

### 위협
- **base64 ≠ 암호화** — `echo bXlwYXNzd29yZA== | base64 -d` → 평문
- **etcd 평문** — etcd 직접 접근하면 *모든 secret 보임*
- **RBAC 부족** — secret read 권한 가진 모든 user/SA
- **Pod 안 환경변수** — `/proc/{pid}/environ` 로 *다른 프로세스에서 보임*
- **dump 위험** — `kubectl get secret -o yaml` 누구나

### Secret types
| Type | 용도 |
|---|---|
| `Opaque` (기본) | 임의 key-value |
| `kubernetes.io/dockerconfigjson` | image registry credential |
| `kubernetes.io/tls` | TLS 인증서·키 |
| `kubernetes.io/service-account-token` | SA token (자동 생성) |
| `kubernetes.io/ssh-auth` | SSH key |
| `kubernetes.io/basic-auth` | basic auth |

---

## 2. etcd Encryption at Rest

### 설정
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - kms:                            # ← KMS provider 권장
      apiVersion: v2
      name: aws-kms
      endpoint: unix:///tmp/kms.sock
      cachesize: 1000
      timeout: 3s
  - aescbc:                         # fallback (less secure)
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}                    # plain text (last resort)
```

### kube-apiserver 설정
```yaml
- --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Provider 종류

| Provider | 특징 |
|---|---|
| **kms (v2)** | *외부 KMS 위임* (AWS KMS, GCP KMS, Vault) — **권장** |
| **kms (v1)** | 옛 API, 1.28+에서 deprecated |
| **aescbc / aesgcm / secretbox** | 자체 키 (config에 박힘) — *키 관리 부담* |
| **identity** | 암호화 X (default) |

### KMS provider plugin
- AWS: `aws-encryption-provider` (sidecar)
- GCP: `gcp-kms-encryption-provider`
- Vault: `vault-kubernetes-mutator` 또는 Bank-Vaults

### 기존 secret 재암호화
- *encryption config 적용 후 기존 secret*은 *옛 방식* (또는 평문)
- *재암호화 필요*:
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### 매니지드 K8s
- **EKS**: *envelope encryption + KMS* 클릭 한 번 (cluster 생성 시)
- **GKE**: *application-layer secrets encryption*
- **AKS**: *KMS V2 (preview)*

---

## 3. *RBAC 빡세게* — 누가 secret read 가능?

### 위험
- `view` ClusterRole = *secret read 가능* (기본). 모든 user가 view 받으면 → secret 노출.
- `default` SA에 *secret 권한* 있으면 → 모든 Pod이 모든 secret 접근

### 권장
- *secret read는 *명시적 RoleBinding으로 *특정 SA·user에만**
- *cross-namespace secret read는 *cluster-admin만**
- *audit logging level RequestResponse*로 *모든 secret 접근 추적*

### Check
```bash
# 누가 secret 읽을 수 있나?
kubectl auth can-i list secrets --as=system:serviceaccount:default:my-sa
kubectl auth can-i list secrets --as=alice

# 모든 binding 검사
kubectl get rolebindings,clusterrolebindings -A \
  -o yaml | grep -B 5 secrets
```

---

## 4. Secret을 *Pod에 주입*하는 방법

### 1. 환경변수
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```
- 함정: `/proc/{pid}/environ`으로 *다른 프로세스에서 볼 수 있음*
- *Logging에 env dump 사고* 흔함

### 2. Volume mount
```yaml
volumes:
- name: secrets
  secret:
    secretName: db-credentials
    defaultMode: 0400          # read-only owner

containers:
- name: app
  volumeMounts:
  - name: secrets
    mountPath: /etc/secrets
    readOnly: true
```
- *파일로 마운트* (`/etc/secrets/password`)
- *tmpfs* (메모리만, 디스크 안 닿음)
- *Secret 변경 시 자동 갱신* (수십 초 안)

### 3. Init container — secret fetch + 처리
```yaml
initContainers:
- name: fetch-secret
  image: vault-cli
  command: ["sh", "-c", "vault read ... > /vault/db-creds"]
  volumeMounts:
  - name: vault-secrets
    mountPath: /vault
```
- *런타임에 *외부에서 fetch*
- K8s Secret을 *전혀 안 만듦*

### 4. Sidecar (Vault Agent Injector)
- *Vault Agent가 sidecar로 들어가서 *주기적으로 secret 갱신**
- *앱은 그냥 파일 읽음*

→ 권장 우선순위: *4 > 3 > 2 > 1*. 환경변수는 *마지막*.

---

## 5. External Secrets Operator (ESO)

### 발상
- *Secret을 K8s에 *manual 만들면 git에 못 commit + 동기화 부담*
- 해결: *외부 secret manager가 source of truth + ESO가 자동 동기화*

### Architecture
```
┌─────────────────────────┐
│ AWS Secrets Manager /   │  source of truth
│ HashiCorp Vault /        │
│ GCP Secret Manager /     │
│ Azure Key Vault          │
└─────────┬───────────────┘
          │ ESO가 polling 또는 watch
          ▼
┌─────────────────────────┐
│ ExternalSecret (CRD)     │  ← git에 commit (참조만)
└─────────┬───────────────┘
          │ ESO가 fetch
          ▼
┌─────────────────────────┐
│ K8s Secret               │  ← 자동 생성 (Pod이 mount)
└─────────────────────────┘
```

### ExternalSecret 예시
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials        # 생성될 K8s Secret 이름
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: myapp/db
      property: username
  - secretKey: password
    remoteRef:
      key: myapp/db
      property: password
```

### SecretStore
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:                    # IRSA 활용
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

→ *IRSA로 정적 키 0*. ESO Pod이 *IAM role assume → Secrets Manager fetch*.

### 장점
- *git에 secret value 없음* (참조만)
- *secret manager가 source of truth*
- *자동 회전* (refreshInterval)
- *audit는 secret manager에서*

### 단점
- *외부 의존* — secret manager down 시 ESO도 fail
- *cache + 회전 지연* (refreshInterval)

→ **운영 표준** — secret 관리의 정답.

---

## 6. Sealed Secrets — *git에 commit 가능한 암호화 Secret*

### 발상
- *git이 source of truth*고 싶다 (GitOps)
- 단 secret을 *평문으로 못 commit*
- 해결: *cluster 안 controller가 *public key 발급* → 그 키로 *클라이언트 측 암호화* → git에 commit*

### 흐름
```
1. 운영자 노트북:
   kubectl create secret ... --dry-run -o yaml | kubeseal > sealed-secret.yaml
   (kubeseal 도구가 controller의 public key로 암호화)

2. git commit sealed-secret.yaml

3. ArgoCD/kubectl이 sealed-secret을 apply

4. Cluster의 *Sealed Secrets controller*가:
   - sealed-secret을 *복호화* (자기 private key로)
   - K8s Secret 생성
```

### SealedSecret 예시
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  encryptedData:
    password: AgB+vM8a9...HEAVILY_ENCRYPTED_BASE64...
    username: AgCx2bN3...
```

### kubeseal CLI
```bash
# fetch controller's public cert
kubeseal --fetch-cert > pub-cert.pem

# encrypt
echo -n 'mypassword' | kubectl create secret generic db --dry-run=client --from-file=password=/dev/stdin -o yaml | kubeseal --cert pub-cert.pem -o yaml > sealed.yaml
```

### 장점
- *git에 commit OK* (암호화됨)
- *GitOps 완전 친화*
- *외부 secret manager 불필요*

### 단점
- *Controller의 private key 분실 시 모든 sealed-secret 복호화 불가*
- *cluster별 다른 key* → *multi-cluster 운영 복잡*
- *secret 자체는 cluster에 K8s Secret으로 존재* (etcd encryption 별도 필요)

---

## 7. SOPS — *파일 단위 암호화*

### 발상
- *YAML/JSON/.env 파일 자체를 부분 암호화*
- *git diff 가독성 유지* (key는 평문, value만 암호화)

### 사용
```bash
# AWS KMS 사용
sops --encrypt --kms arn:aws:kms:... secrets.yaml > secrets.enc.yaml

# 결과
# password: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
# username: alice    ← key, 평문 value도 가능

# 복호화
sops --decrypt secrets.enc.yaml > secrets.yaml
```

### Backend
- AWS KMS
- GCP KMS
- Azure Key Vault
- HashiCorp Vault
- PGP / age (offline keys)

### Helm·Terraform 통합
- **helm-secrets** plugin — Helm install 시 SOPS 자동 복호화
- **terraform-provider-sops** — Terraform에서 SOPS 파일 read

### vs Sealed Secrets
| | Sealed Secrets | SOPS |
|---|---|---|
| Cluster 의존 | 필요 (controller) | 필요 X |
| 복호화 | cluster 안에서 | 운영자·CI에서 (deploy 전) |
| 키 관리 | controller private key | KMS / PGP / age |
| 도구 | kubeseal CLI | sops CLI + plugin |
| GitOps 친화 | ✓ (ArgoCD가 직접 apply) | △ (Helm 사용 시 plugin 필요) |

---

## 8. Vault — *secret 관리의 *깊은 도구**

### 기능
- **KV store** — 정적 secret 저장
- **Dynamic secrets** — *런타임에 새 credential 생성* (DB user, AWS STS 등)
- **PKI** — TLS 인증서 발급·갱신
- **Transit** — *암호화 service* (key는 Vault, data는 외부)
- **SSH CA** — SSH 인증서 발급
- **Encryption** — *application-level encryption*

### K8s 통합 패턴

**(1) ExternalSecret + Vault** — 가장 흔함
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: external-secrets-sa
```

**(2) Vault Agent Injector** — Sidecar 패턴
- Pod annotation에 *Vault 정보* 명시
- Vault Agent가 *sidecar로 자동 주입*
- *fileshare로 secret 전달, 주기적 갱신*

```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/myapp/db"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "secret/data/myapp/db" -}}
      DB_USER={{ .Data.data.username }}
      DB_PASS={{ .Data.data.password }}
      {{- end -}}
```

**(3) CSI Secret Store Driver** — *직접 mount*
- *K8s Secret을 안 만들고* Pod이 *직접 Vault에서 fetch*
- 가장 secure (etcd에 없음)

### Dynamic Database Credentials
- 옛: *DB password 영구 + 회전 부담*
- Vault Dynamic: 앱이 시작 시 *Vault에 요청 → Vault가 *그 자리에서 DB user 생성* → 1시간 유효 → 만료 시 자동 삭제*
- *유출돼도 1시간만 위험*

---

## 9. Secret *회전* 자동화

### 옛 패턴
- *분기 1회 수동 회전*
- 회전 중 *앱 재시작 필요*
- *DB에 옛 + 새 password 둘 다 valid한 짧은 윈도우*

### 현대 패턴
- *외부 secret manager가 회전* (예: AWS Secrets Manager의 *Lambda 자동 회전*)
- ESO가 *refreshInterval마다 K8s Secret 갱신*
- 앱이 *Secret 변경 감지 + reload* 또는 *Pod 자동 재시작*

### Pod 재시작 trigger 패턴 (Helm)
```yaml
metadata:
  annotations:
    checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```
→ Secret 내용 변경 → annotation 변경 → Pod template 변경 → rolling update

### Reloader operator
- *Secret/ConfigMap 변경 감지 시 자동 Pod 재시작*
- annotation 박으면 작동
```yaml
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "db-credentials"
```

---

## 10. 자주 헷갈리는 것

### 10-1. *base64 = 암호화*로 착각
- 누구나 디코딩
- 진짜 보안은 *etcd encryption + RBAC + 외부 secret manager*

### 10-2. *환경변수로 secret 주입 + log에 env dump*
- 사고 흔함
- *Volume mount 우선*, 환경변수는 *log에 dump 안 함* 확인

### 10-3. *ESO Pod이 *너무 많은 IAM 권한**
- ESO가 *모든 secret 접근* — 그 Pod 침해 = 큰 사고
- *namespace별 SecretStore + 최소 권한 IAM Role* 패턴

### 10-4. *Sealed Secrets controller 키 분실*
- 모든 sealed-secret 복호화 불가
- *controller key를 별도 안전한 곳에 백업*

### 10-5. *etcd encryption 활성화했지만 *기존 secret*이 평문*
- 활성화 *후* 새로 만든 것만 암호화
- *기존은 `kubectl get secrets -A -o json | kubectl replace -f -`로 재암호화*

### 10-6. *Secret 회전 했는데 Pod 안 재시작*
- env로 inject면 *Pod 재시작 전엔 옛 값*
- volume mount면 *자동 갱신* (수십 초)
- *checksum annotation* 또는 *Reloader operator*

---

## 🎤 면접 빈출 Q&A

### Q1. K8s Secret이 *진짜 안전*하려면?
> base64는 *암호화 아님*. 4단계: (1) **etcd encryption at rest (KMS provider)** — 디스크 도난 시도 보호. (2) **RBAC 빡세게** — secret read 권한 최소화. (3) **외부 secret manager** (Vault, AWS Secrets Manager) **+ External Secrets Operator** — git에 평문 없음, 자동 회전. (4) **Pod 주입은 Volume mount 우선** (환경변수는 log dump 위험). EKS는 *envelope encryption + KMS* 클릭 한 번.

### Q2. External Secrets Operator는 무엇? 왜?
> *외부 secret manager (Vault, AWS Secrets Manager 등)를 K8s에 동기화*하는 operator. *ExternalSecret CRD*에 *외부 경로 참조만* → ESO가 *fetch → K8s Secret 자동 생성*. **git에 secret value 없음**, **secret manager가 source of truth**, *자동 회전 (refreshInterval)*. IRSA로 *정적 키 0*. **운영 표준**.

### Q3. Sealed Secrets vs SOPS 비교?
> **Sealed Secrets**: *cluster controller의 public key로 client-side 암호화* → git commit OK → controller가 *복호화하여 K8s Secret 생성*. GitOps 친화. **Single point of failure: controller key**. **SOPS**: *파일 단위 부분 암호화* (key 평문, value 암호화). KMS/PGP/age 활용. *cluster 의존 X* — 운영자·CI가 복호화. Helm-secrets·Terraform plugin 통합.

### Q4. Vault의 *Dynamic Secrets*이란?
> 옛: *DB password 영구* + 회전 부담. Vault Dynamic: *런타임에 DB user 생성* — 앱이 *Vault에 요청 → Vault가 *그 자리에서 DB user 생성* → 1시간 유효 → 만료 시 자동 삭제*. **유출돼도 1시간만 위험**. AWS STS도 비슷 — *영구 IAM user 대신 1시간 token*. 회전 자동화의 정수.

### Q5. Secret을 *Pod에 주입*하는 4 방법?
> (1) **환경변수** — 간단하나 `/proc/{pid}/environ` 노출, log dump 사고. (2) **Volume mount** — 파일로, tmpfs, 자동 갱신. (3) **Init container** — 외부에서 fetch 후 처리. (4) **Sidecar (Vault Agent Injector)** — 주기적 갱신. **권장 우선순위**: Sidecar > Init > Volume > 환경변수.

### Q6. Secret 회전 후 Pod이 *옛 값 사용*하는 문제?
> *환경변수로 주입*된 경우 Pod 재시작 전까지 옛 값. *Volume mount*는 *수십 초 안에 파일 갱신* (앱이 *파일 watch + reload* 해야). 해결: (1) **Helm `checksum/secret` annotation** — Secret 변경 시 Pod template hash 변화 → rolling update. (2) **Reloader operator** — annotation 박으면 *자동 Pod 재시작*. (3) **앱이 *주기적 secret reload* 구현**.

### Q7. etcd encryption at rest의 *KMS provider* vs *aescbc* 차이?
> **KMS** (v2): *외부 KMS에 위임* (AWS KMS, GCP KMS, Vault) — **권장**. 키 관리·회전·HSM 통합 자동. **aescbc/aesgcm/secretbox**: *키가 config 파일에 hardcode* — 키 관리 부담, 회전 어려움. **운영은 KMS**. 옛 KMS v1은 1.28+ deprecated.

---

## 🔗 Cross-reference

- **K8s Secret 기본** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md)
- **IRSA (ESO가 cloud secret manager 접근)** → [03-identity-and-rbac.md](03-identity-and-rbac.md)
- **RBAC** → [03-identity-and-rbac.md](03-identity-and-rbac.md), [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)
- **Helm `checksum/secret` 패턴** → [`helm-basics/02`](../helm-basics/02-templating.md)
- **ArgoCD + Sealed Secrets / ESO / SOPS** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)
- **AWS KMS / Secrets Manager / IAM** → `aws-basics/` (예정)
- **Terraform secret 관리** → [`terraform-basics/04`](../terraform-basics/04-variables-and-outputs.md)

---

## 📝 3줄 요약

1. *base64는 암호화 아님*. 진짜 보안 = **etcd encryption (KMS) + RBAC + 외부 secret manager (Vault/AWS SM) + ESO**. *Pod 주입은 Volume > env*.
2. **External Secrets Operator**가 *운영 표준* — git에 평문 없음, secret manager source of truth, 자동 회전. **Sealed Secrets / SOPS**는 GitOps에서 git-only 패턴.
3. *Vault Dynamic Secrets*로 *런타임 credential 생성* — 영구 password 회전 부담 제거. **회전 후 Pod 자동 재시작**은 *checksum annotation* 또는 *Reloader operator*.

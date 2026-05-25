# Kubernetes Security — 동작 원리 중심

> *"K8s 보안은 어디부터 어디까지?"*
> *"PSS / NetworkPolicy / RBAC / Admission / etcd encryption — 각각 어떤 위협 막나?"*
> *"공급망 보안 (cosign·SBOM·admission)을 어떻게 구성?"*

---

## 🎯 한 문장 큰 그림

> **K8s 보안 = *4 + 2 축*. **4 in-cluster**: Pod Security (runtime 권한 제한) / Network Policy (Pod 간 통신 제어) / RBAC (API 권한) / Secret 관리 (etcd encryption + 외부 매니저). **2 supply chain**: Image 신뢰 (cosign + SBOM + Trivy) / Admission policy (Kyverno/Gatekeeper). **defense-in-depth** — 한 layer 뚫려도 다음 layer 차단.**

---

## 🧩 한 페이지 요약

```
┌────────────────────────────────────────────────────────────┐
│  Supply Chain (배포 전)                                       │
│  ├─ Image scan (Trivy/Snyk) — CVE 차단                       │
│  ├─ SBOM 생성 (Syft) — 공급망 가시성                          │
│  └─ Image 서명 (Cosign keyless) — 출처 검증                   │
└────────────────────────────────────────────────────────────┘
                       │ kubectl apply / argocd sync
                       ▼
┌────────────────────────────────────────────────────────────┐
│  K8s API server (admission 단계)                              │
│  ├─ Authentication — 누구야? (mTLS/OIDC/SA token)             │
│  ├─ Authorization — RBAC (무엇 할 수 있어?)                    │
│  ├─ Mutating Admission — sidecar 자동 주입 등                  │
│  ├─ Validating Admission — PSS, Kyverno, Sigstore policy ←서명 검증│
│  └─ etcd (encrypted at rest)                                  │
└────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────┐
│  Cluster runtime                                              │
│  ├─ Pod Security: runAsNonRoot, cap drop, seccomp, RO rootfs │
│  ├─ Network Policy: default-deny + 명시 allow                 │
│  ├─ Secret 주입: External Secrets Operator (Vault/AWS SM)     │
│  ├─ Identity: ServiceAccount → IRSA (cloud IAM)               │
│  └─ Runtime detection: Falco (선택)                            │
└────────────────────────────────────────────────────────────┘
```

---

## 🌪 자주 헷갈리는 7가지

### 1. *컨테이너 ≠ VM 보안 경계*
- *호스트 커널 공유* — 커널 exploit 하나로 호스트 탈출 가능
- → **defense-in-depth 필수** — 한 layer만 의존 X
- 진짜 강한 격리는 *gVisor / Kata Containers* (마이크로 VM)

→ [`docker-basics/06`](../docker-basics/06-security.md) 의 *container escape* 와 같은 베이스.

### 2. *Default가 *insecure**
- *PSS 안 설정* → privileged 컨테이너 가능
- *NetworkPolicy 없음* → *모든 Pod 간 자유 통신*
- *RBAC default SA* → namespace 권한 있을 수 있음
- *Secret base64* — *암호화 아님*
- → **명시적 hardening 필수**.

### 3. *Pod Security Standards (PSS) ≠ Pod Security Policy (PSP)*
- *PSP*는 *deprecated* (1.21+), *제거* (1.25+)
- *PSS*는 *내장 admission plugin* (1.25+) — *namespace label로 enforce*
- 마이그레이션: PSP → PSS or Kyverno/Gatekeeper

### 4. *NetworkPolicy는 *CNI가 구현**
- K8s는 *spec만* 정의 — *실제 시행은 CNI*
- *Flannel 기본*은 NetworkPolicy 지원 X — Calico/Cilium 필요
- *AWS VPC CNI*는 *별도 add-on*

→ [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md) 의 CNI 책임 분리와 일관.

### 5. *Secret base64는 *암호화 아님**
- 누구나 `base64 -d`로 복호화
- 진짜 보안: *etcd encryption at rest + RBAC + 외부 secret manager*

### 6. *IRSA = K8s ServiceAccount + AWS IAM Role*
- Pod이 *정적 AWS AccessKey 안 받고 *IAM Role assume*
- *Projected SA Token (JWT)* → AWS STS → 임시 credentials
- *cloud별 비슷한 메커니즘* — GCP Workload Identity, Azure Workload Identity

→ [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) 의 IRSA와 같음.

### 7. *Admission Controller 가 *공급망 보안의 *Enforcement Point**
- CI에서 *cosign 서명*해도 admission이 *검증 안 하면* 의미 X
- Kyverno / Gatekeeper / Sigstore policy-controller가 *배포 직전 검증*

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-pod-security.md](01-pod-security.md) | PSS (privileged/baseline/restricted), securityContext, capability, seccomp, gVisor/Kata |
| [02-network-policy.md](02-network-policy.md) | default-deny + 명시 allow, ingress/egress, namespaceSelector, CNI 책임, Cilium L7 |
| [03-identity-and-rbac.md](03-identity-and-rbac.md) | ServiceAccount, RBAC 깊이, **IRSA / Workload Identity**, audit log, multi-tenancy |
| [04-supply-chain.md](04-supply-chain.md) | Trivy 스캔, Cosign keyless 서명, SBOM, Admission (Kyverno/Gatekeeper/Sigstore), SLSA |
| [05-secrets-and-encryption.md](05-secrets-and-encryption.md) | etcd encryption at rest, External Secrets Operator, Vault, Sealed Secrets, SOPS |

---

## 🔗 Cross-reference

- **컨테이너 격리 베이스** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md), [`docker-basics/06`](../docker-basics/06-security.md)
- **K8s RBAC 기본** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)
- **API server 흐름 (Auth/Admission)** → [`kubernetes-basics/01`](../kubernetes-basics/01-architecture.md)
- **Admission webhook 메커니즘** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **CI에서 cosign 서명** → [`cicd-gitops-basics/05`](../cicd-gitops-basics/05-build-and-registry.md)
- **AWS IAM / KMS / Secrets Manager** → `aws-basics/` (예정)

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| Pod Security Standards가 강제하는 것? | [01](01-pod-security.md) |
| NetworkPolicy default-deny 패턴? | [02](02-network-policy.md) |
| IRSA 동작 원리? | [03](03-identity-and-rbac.md) |
| Cosign keyless가 안전한 이유? | [04](04-supply-chain.md) |
| Image 무결성 강제 (admission)? | [04](04-supply-chain.md) |
| Secret을 진짜 안전하게? | [05](05-secrets-and-encryption.md) |
| External Secrets Operator vs Sealed Secrets? | [05](05-secrets-and-encryption.md) |
| K8s audit log를 어떻게? | [03](03-identity-and-rbac.md) |
| 멀티 테넌시 cluster의 보안 격리? | [02](02-network-policy.md) + [03](03-identity-and-rbac.md) |

---

## 📐 학습 순서 권장

1. **01 pod-security** — *Pod 단위 베이스*
2. **02 network-policy** — *통신 통제*
3. **03 identity-and-rbac** — *누가 무엇 할 수 있나*
4. **04 supply-chain** — *배포 전 신뢰*
5. **05 secrets-and-encryption** — *민감 정보 보호*

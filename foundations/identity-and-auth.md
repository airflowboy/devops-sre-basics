# Identity & Auth — uid부터 OIDC federation까지

> [[tls-and-pki]]가 *어떻게 안전하게 통신하는가* 라면, 이 문서는 *누가 누구로 행동하는가*.
> linux uid, container user namespace, k8s ServiceAccount, RBAC, AWS IAM, STS, IRSA, OIDC federation —
> 모두 같은 질문에 대한 다른 레이어의 답.

---

## 1. Linux의 신원 — uid/gid

### 가장 근본

Linux 커널은 *문자열 username을 모른다*. 모든 권한 결정은 `uid` (정수)로 한다. `/etc/passwd` 는 uid ↔ name 매핑일 뿐.

```
USER=alice (이건 shell의 환상)
   └─ uid=1000 (이게 진짜)
```

container 안의 `root` 도 *uid=0*. host에 매핑되지 않으면 host의 root와 *같은* uid를 갖는다 → user namespace가 없으면 container root가 host root와 동급.

### Process credential

각 프로세스는 credential 묶음을 들고 다닌다:

- **Real UID/GID** — 누가 이 프로세스를 시작했나
- **Effective UID/GID** — *지금 권한 결정에 쓰이는 값*
- **Saved UID/GID** — setuid에서 원래 값 복원용
- **Supplementary groups** — 추가 그룹 목록
- **Capabilities** (Linux 2.2+) — root 권한을 *조각* 으로 나눈 것

### setuid — *임시 권한 승격*

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root ... /usr/bin/passwd
      ^ setuid bit
```

`passwd` 는 누가 실행해도 *effective uid가 0* 이 된다. 그래야 `/etc/shadow` 를 수정 가능. setuid는 "필요한 동안만 root" 의 고전적 메커니즘 — 위험하지만 강력.

### Capability — root의 *분할*

```
CAP_NET_BIND_SERVICE   80 같은 well-known port 바인딩
CAP_NET_ADMIN          네트워크 설정 (iptables, route)
CAP_SYS_ADMIN          "거의 root" — 너무 광범위해서 비판 받음
CAP_DAC_OVERRIDE       파일 권한 무시
CAP_CHOWN              파일 소유자 변경
```

container security는 *불필요한 capability 제거* 가 시작점. k8s `securityContext.capabilities.drop: ["ALL"]` + 필요한 것만 add.

### PAM — *어떻게 인증하는가* 의 추상화

PAM (Pluggable Authentication Modules) = "패스워드 확인하는 방법은 여러 가지가 있다" 의 답.

```
/etc/pam.d/sshd
auth required pam_unix.so       # /etc/shadow 검사
auth required pam_google_authenticator.so  # 2FA
account required pam_nologin.so # /etc/nologin 있으면 거부
```

sshd·sudo·login·su 가 다 PAM을 통해 인증. 새 인증 방식 (LDAP, Kerberos, SSO) 추가는 PAM 모듈 하나 작성 = *시스템 전체에 적용*.

---

## 2. Container의 신원 — User Namespace

### 문제

container 안의 `root` (uid 0)가 host에서도 *uid 0* 이면, container 탈출 시 host root. → 매우 위험.

### User namespace

커널 기능: container 안 uid를 host의 *다른 uid* 로 매핑.

```
Container 안:    uid 0 (root)
                   ↓ 매핑
Host에서는:      uid 100000 (권한 없는 일반 사용자)
```

container 안에서 `root` 권한 쓰지만, host에서는 *100000번 사용자* 일 뿐.

### 현실의 docker/k8s

- docker는 user namespace를 *기본으로 안 쓴다* (`--userns-remap` 옵션).
- rootless docker / podman은 *기본* 사용.
- k8s는 v1.25+에서 alpha, v1.30+에서 beta. 아직 광범위하지 않음.

대안: container에서 *애초에 root로 안 돈다*.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

이미지가 root 가정으로 쓰여있으면 (`USER` 미지정) 깨진다. → distroless·alpine 이미지의 *nonroot* 변종이 인기.

→ [[shell-scripting]] §3 PID 1 문제와 합쳐 보면, container의 신원·시그널 모델이 *host와 같지 않다* 는 일관된 그림.

---

## 3. k8s ServiceAccount — Pod의 정체

### SA = Pod이 *apiserver에게 자기 자신을 증명하는 방법*

각 Pod은 SA를 가진다 (기본은 namespace의 `default` SA). Pod 시작 시 kubelet이 *projected JWT* 를 Pod 안에 마운트:

```
/var/run/secrets/kubernetes.io/serviceaccount/
  ├─ token   ← JWT
  ├─ ca.crt  ← apiserver를 검증할 CA
  └─ namespace
```

이 token으로 apiserver에 호출 → apiserver가 "이건 namespace `foo`의 SA `bar`" 라고 인식.

### v1.22 전후로 *완전히 다른 모델*

| 버전 | 모델 |
|---|---|
| v1.21 이전 | SA마다 *영구 token* 이 Secret으로 저장. Pod이 그 Secret을 mount. 노출 시 *영구히 유효*. |
| v1.22+ | **Projected SA token** — kubelet이 *Pod 시작 시* 짧은 만료 (`expirationSeconds`, 기본 1시간) JWT를 발급. 자동 갱신. |

projected token이 *audience* 까지 지정 가능 → IRSA·workload identity가 이 위에 쌓임.

### JWT 내용물

```json
{
  "iss": "https://kubernetes.default.svc",
  "sub": "system:serviceaccount:my-ns:my-sa",
  "aud": ["api"],
  "exp": 1234567890,
  "kubernetes.io": {
    "namespace": "my-ns",
    "pod": { "name": "my-pod", "uid": "..." },
    "serviceaccount": { "name": "my-sa", "uid": "..." }
  }
}
```

→ [[tls-and-pki]] §6에서 본 JWT 구조 그대로.

---

## 4. RBAC — *그래서 뭐 할 수 있는데*

§3의 SA는 *정체* 만 알려준다. *권한* 은 RBAC이 따로 정한다.

### 모델

```
Subject  →  Verb  →  Resource
(누가)      (뭘)      (무엇에)
```

| Subject | Verb | Resource |
|---|---|---|
| ServiceAccount, User, Group | get/list/watch/create/update/patch/delete | pods, services, configmaps, ... (+ subresource: `pods/log`, `pods/exec`) |

### 객체 4종

```
Role                 namespace-scoped 권한 묶음
ClusterRole          cluster-scoped 권한 묶음
RoleBinding          Subject ↔ Role 연결 (namespace 안)
ClusterRoleBinding   Subject ↔ ClusterRole 연결 (cluster 전체)
```

### AWS IAM과의 *동형성*

| k8s RBAC | AWS IAM |
|---|---|
| Subject (SA·user·group) | Principal (IAM user·role) |
| Verb (`get`, `create`) | Action (`s3:GetObject`) |
| Resource (Pod, ConfigMap) | Resource (`arn:aws:s3:::bucket/*`) |
| RoleBinding | IAM policy attachment |
| Aggregated ClusterRole | IAM managed policy |

**같은 모델, 다른 문법.** 한쪽 익히면 다른 쪽이 *놀랍게 비슷* 하다.

---

## 5. AWS IAM과 STS — *임시 자격증명* 의 발명

### IAM의 기본

```
Principal  →  Action  →  Resource  +  Condition
(누가)        (뭘)        (무엇에)      (언제·어떻게)
```

policy 예시:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": { "StringEquals": { "aws:RequestTag/env": "prod" } }
}
```

condition이 *시간·IP·tag·MFA 여부* 등으로 동적 제어 → "어떤 상황에서만" 권한 부여.

### IAM User vs IAM Role

| | User | Role |
|---|---|---|
| 자격증명 | 영구 access key | *임시* (STS가 발급) |
| 누가 사용 | 사람 | 사람·서비스·Pod 누구나 (assume) |
| 권장 패턴 | 거의 안 씀 (IAM Identity Center 권장) | 모든 자동화 |

### STS AssumeRole

```
Principal A
   ↓ sts:AssumeRole(role=B)
임시 자격증명 (access key + secret + session token, 만료 있음)
   ↓
B의 권한으로 행동
```

핵심: 영구 key 없이 *역할을 빌림*. 다음으로 이어짐 →

---

## 6. IRSA — k8s SA를 IAM Role로 *교환*

### 문제

Pod이 AWS API 호출하려면 자격증명이 필요. 옵션:

| 옵션 | 문제 |
|---|---|
| 영구 access key를 Secret으로 mount | 노출되면 영구히 위험, rotation 어려움 |
| 노드에 IAM instance profile | *그 노드의 모든 Pod이 같은 권한* — 격리 깨짐 |
| **IRSA (IAM Roles for ServiceAccounts)** | Pod마다 *다른* role, 임시 자격증명 |

### IRSA 흐름

```
1. EKS 클러스터의 OIDC issuer URL을 IAM에 등록 (한 번)
2. IAM Role의 trust policy에 "이 OIDC issuer가 발급한 JWT 중,
   sub='system:serviceaccount:my-ns:my-sa' 면 assume 허용" 추가
3. SA에 annotation: eks.amazonaws.com/role-arn: arn:aws:iam::...:role/my-role
4. Pod 시작 → kubelet이 projected JWT 발급 (aud=sts.amazonaws.com)
5. Pod 안의 AWS SDK가 JWT를 STS에 보내며 AssumeRoleWithWebIdentity 호출
6. STS가 JWT를 OIDC issuer JWKS로 검증 → 임시 자격증명 발급
7. SDK가 그 자격증명으로 AWS API 호출
```

### 핵심 통찰

IRSA는 *새 기술이 아니다*. 다음 셋의 조합:

- k8s SA의 projected JWT ([[tls-and-pki]] §6의 OIDC)
- OIDC discovery (JWKS endpoint)
- STS AssumeRoleWithWebIdentity (원래 외부 IdP용)

k8s apiserver가 *OIDC IdP 역할* 을 한다는 발상이 핵심. 이 패턴이 GKE workload identity, AKS workload identity, 그리고 GitHub Actions OIDC로도 *그대로* 확장.

---

## 7. GitHub Actions OIDC — *같은 패턴, 다른 IdP*

### 문제

CI/CD가 AWS·GCP·Azure에 deploy하려면 자격증명 필요. 영구 secret을 GitHub Secrets에 저장하는 게 표준이었지만 — 노출 시 위험.

### OIDC federation

```
1. GitHub Actions의 OIDC issuer: https://token.actions.githubusercontent.com
2. AWS IAM에 이 issuer를 OIDC provider로 등록
3. IAM Role trust policy: "repo:my-org/my-repo:ref:refs/heads/main 의 JWT면 assume 허용"
4. Workflow에서 OIDC token 요청:
   permissions: { id-token: write }
5. 워크플로우 step이 토큰으로 AWS STS AssumeRoleWithWebIdentity
6. 임시 자격증명으로 AWS API
```

**완벽히 IRSA와 같은 패턴**. IdP만 다름 (k8s apiserver → GitHub).

### JWT claim으로 *어디까지* 정밀하게 제어

```
"sub": "repo:my-org/my-repo:ref:refs/heads/main"
"sub": "repo:my-org/my-repo:pull_request"
"sub": "repo:my-org/my-repo:environment:production"
```

trust policy condition으로 "production 환경에서, main 브랜치에서만" 같은 조건 가능. PR에서 prod 배포 못 하게 막는 기본 방어.

---

## 8. Ansible Become / sudo — *권한 승격의 또 다른 레이어*

§1의 setuid가 *바이너리에 영구* 라면, sudo는 *호출 시점에 정책 확인*.

```
/etc/sudoers
alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

alice는 *systemctl restart nginx만* root로 실행 가능, 패스워드 없이.

### ansible become

```yaml
- hosts: web
  become: yes
  become_method: sudo
  become_user: root
  tasks:
    - name: restart nginx
      service: { name: nginx, state: restarted }
```

ansible이 SSH로 일반 사용자 로그인 후 *task마다* sudo로 권한 승격. PAM·sudoers가 *실제 정책*. ansible은 *호출자* 일 뿐.

→ [[shell-scripting]] §1의 process model + PAM이 결합된 그림.

---

## 9. 한 줄 정리 — 같은 패턴의 여러 얼굴

| 시나리오 | "정체" | "권한 부여 방식" |
|---|---|---|
| Linux user | uid | file permission + capability |
| Container user | (대부분) host uid 그대로 | + securityContext, user namespace |
| k8s Pod | ServiceAccount JWT | RBAC |
| AWS IAM user | IAM User ARN | IAM policy |
| AWS Role assumer | STS 임시 자격증명 | role의 attached policy |
| k8s Pod → AWS | SA JWT → STS 교환 (IRSA) | IAM role policy |
| CI → AWS | GitHub OIDC JWT → STS 교환 | IAM role policy |
| ansible task | SSH user + become | sudoers + PAM |

**모든 줄이 같은 구조**: *정체 증명* → *정책 평가* → *행동 허가*.

---

## 10. Cross-reference 지도

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| 파일 권한, capability, setuid | `linux-basics/03-filesystem-and-permissions.md` |
| Container securityContext, user namespace | `docker-basics/` 보안, `k8s-security-basics/` PSS |
| RBAC 실제 구성, aggregated ClusterRole | `kubernetes-basics/` security, `k8s-security-basics/` RBAC |
| IAM policy 문법, condition 활용 | `aws-basics/` IAM |
| IRSA 실제 terraform·k8s 매니페스트 | `aws-basics/` EKS, `k8s-security-basics/` |
| GitHub OIDC trust policy | `cicd-gitops-basics/` GitHub Actions |
| JWT 검증 메커니즘, JWKS | [[tls-and-pki]] §6 |
| ansible become, sudoers | `ansible-basics/` |

---

## 한 줄 요약

> **모든 인증/권한 시스템은 같은 골격이다 — 정체 → 정책 → 허가.**
> 다른 건 *정체를 무엇으로 증명하느냐* (uid · JWT · cert) 와 *정책을 어디에 적느냐* (file perm · IAM · RBAC).
> 이 골격이 보이면 새 시스템도 *어디에 무엇이 박혔는지* 빠르게 찾는다.

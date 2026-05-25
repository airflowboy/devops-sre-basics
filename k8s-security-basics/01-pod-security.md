# 01 — Pod Security: PSS · securityContext · runtime 격리

> **공식 근거:** [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/), [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/), [securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
>
> **선행:** [`docker-basics/06`](../docker-basics/06-security.md), [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)

---

## 🎯 한 문장

> **Pod Security = *Pod·컨테이너의 런타임 권한 제한*. *PSS 3 profile (privileged/baseline/restricted)*을 *namespace label로 enforce*. *securityContext*로 *runAsNonRoot + cap drop ALL + seccomp + read-only rootfs*가 *최소 권한 기본*. 진짜 격리 필요하면 *gVisor / Kata*.**

---

## 1. Pod Security Standards (PSS) — 3 Profile

### 발상
- *Pod 보안의 *공통 기준*이 필요*
- *옛 PSP (PodSecurityPolicy)*는 *deprecated → 1.25에서 제거*
- *PSS (Pod Security Standards) + Pod Security Admission* — K8s 1.25+ 내장

### 3 Profile

| Profile | 의미 |
|---|---|
| **privileged** | *거의 제한 없음* — 시스템 컴포넌트용 |
| **baseline** | *알려진 위험 차단* — hostPath, hostNetwork, hostPID, privileged 등 차단 |
| **restricted** | *모범 보안 강제* — non-root, cap drop, seccomp, read-only 권장 |

### Restricted profile이 *강제*하는 것
- `runAsNonRoot: true` — UID 0 (root) 금지
- `allowPrivilegeEscalation: false` — setuid로 권한 상승 금지
- *모든 capability drop ALL + 필요한 것만 add* (`NET_BIND_SERVICE` 만 허용)
- `seccompProfile.type: RuntimeDefault` 또는 `Localhost`
- `readOnlyRootFilesystem` — *권장* (강제 X)
- *Host namespace 공유 금지* (hostNetwork, hostPID, hostIPC)
- *privileged container 금지*
- *hostPath volume 금지*
- *Linux capabilities — drop ALL + add 명시*

### Namespace label로 적용
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

### Mode 3종
- **enforce** — 위반 Pod *생성 거부*
- **audit** — 위반 시 *audit log* (생성은 허용)
- **warn** — 위반 시 *사용자에게 warning* (생성은 허용)

→ 운영 흐름: *audit/warn으로 시작 → 위반 정리 → enforce로 전환*.

### exemptions
```yaml
# /etc/kubernetes/policy/podsecurity.yaml
apiVersion: pod-security.admission.config.k8s.io/v1
kind: PodSecurityConfiguration
defaults:
  enforce: "baseline"
  enforce-version: "latest"
exemptions:
  usernames: ["admin-user"]
  namespaces: ["kube-system", "ingress-nginx"]
  runtimeClasses: ["kata"]
```

→ 시스템 컴포넌트 (`kube-system`)는 *예외*. 일반 워크로드만 strict.

---

## 2. securityContext — *Pod·Container 단위*

### Pod 수준
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000              # volume의 ownership
    fsGroupChangePolicy: OnRootMismatch
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: [1001, 1002]
```

### Container 수준 (Pod 수준 override)
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
        add: [NET_BIND_SERVICE]   # 80 포트 bind 필요 시
      seccompProfile:
        type: RuntimeDefault
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}              # read-only root + /tmp만 writable
```

### 권장 *최소 권한 베이스라인*
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
  seccompProfile:
    type: RuntimeDefault
```

→ 이 설정이 PSS *restricted profile의 핵심*.

---

## 3. Capabilities — *root 권한 세분화*

Linux는 root 권한을 *30+ capability*로 분할 ([`linux-basics/03`](../linux-basics/03-filesystem-and-permissions.md)).

### Drop ALL + 필요한 것만
```yaml
securityContext:
  capabilities:
    drop: [ALL]
    add: [NET_BIND_SERVICE]    # 80/443 bind만
```

### 자주 add 하는 capability
| Cap | 의미 |
|---|---|
| `NET_BIND_SERVICE` | 1024 이하 포트 bind |
| `CHOWN` | 파일 소유자 변경 |
| `DAC_OVERRIDE` | DAC 권한 무시 (파일 접근) |
| `FOWNER` | 파일 소유 무시 (chmod 등) |
| `SETUID` / `SETGID` | uid/gid 변경 |
| `KILL` | 임의 프로세스 signal |
| `NET_ADMIN` | 네트워크 설정 (iptables 등) — *위험* |
| `SYS_ADMIN` | *광범위* — *거의 root* — 피할 것 |

→ Docker 기본은 *14개 capability* 부여. `drop: [ALL]` 로 *모두 제거 후 명시*가 정석.

### 흔한 함정
- `NET_BIND_SERVICE` 없이 *nginx가 80 포트 bind 시도* → fail
- 해결: (1) `add: [NET_BIND_SERVICE]`, (2) *unprivileged port* 사용 (8080) + Service NodePort/LoadBalancer

---

## 4. Seccomp — *시스템 콜 필터*

### 발상
- 컨테이너가 호출 가능한 *syscall 목록 제한*
- *위험한 syscall* (예: `mount`, `init_module`, `kexec_load`) 차단
- *최후 방어선* — 컨테이너 escape 시도 시 차단

### Profile
```yaml
seccompProfile:
  type: RuntimeDefault         # 컨테이너 런타임의 기본 (containerd/CRI-O 제공)
```

```yaml
seccompProfile:
  type: Localhost
  localhostProfile: profiles/audit.json   # 커스텀
```

```yaml
seccompProfile:
  type: Unconfined             # 비활성 — *비추*
```

### Docker default vs RuntimeDefault
- 거의 같음 — *위험한 syscall ~50개 차단*
- 일반 앱은 *RuntimeDefault*로 충분

### 커스텀 profile
- 앱의 *최소 syscall set*만 허용
- 도구: `falco-driver-loader`, `bpftrace`, *runtime profile 자동 생성*
- *과도한 strictness* — 앱 종속성 모르면 *fail 폭증*

---

## 5. AppArmor / SELinux — *MAC*

[`linux-basics/03`](../linux-basics/03-filesystem-and-permissions.md) 의 MAC을 K8s에 적용.

### AppArmor (Ubuntu·Debian)
```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container-name>: runtime/default
```

### SELinux (RHEL)
```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

→ *기본 docker/containerd 프로필*로 충분한 경우가 많음. 커스텀은 *전문 영역*.

---

## 6. *Read-only Root Filesystem*

```yaml
securityContext:
  readOnlyRootFilesystem: true

volumeMounts:
- name: tmp
  mountPath: /tmp
- name: cache
  mountPath: /var/cache

volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir:
    medium: Memory             # tmpfs (RAM)
```

### 효과
- 컨테이너 안에서 *`/` write 불가*
- 침해 시 *변조 어려움*
- *임시 파일*은 *명시적 volume mount* (`/tmp`, `/var/cache`)

### 앱이 *어디 쓰는지* 확인
- 시작 시 *write 시도 path*를 *log·strace*로 확인
- *각각 emptyDir volume*으로 mount
- *Logging은 stdout으로 (12-Factor)* — 파일 X

---

## 7. *Privileged Container의 *진짜* 의미*

```yaml
securityContext:
  privileged: true             # ⚠️ 거의 호스트 root
```

### 무엇이 풀리나
- *모든 capability* 부여
- *모든 host device 접근* (`/dev/*`)
- *AppArmor·seccomp 비활성*
- *cgroup 한도 무시 가능*
- → *호스트 fs mount, kernel module load, ptrace, ...*

### 진짜 필요한 경우 (드뭄)
- Storage driver (CSI node plugin)
- Network plugin (CNI)
- 일부 monitoring agent

### 대안
- `--cap-add=NET_ADMIN` 같이 *필요한 cap만*
- `--device=/dev/fuse` 같이 *필요한 device만*
- `hostPath` mount + `securityContext.privileged: false`

→ **PSS baseline·restricted가 privileged 차단**. 정당화 못 하면 *절대 사용 X*.

---

## 8. *Host Namespace 공유* — *위험*

### 위험한 옵션들
```yaml
spec:
  hostNetwork: true            # 호스트 NET namespace 공유 — 호스트 port 직접
  hostPID: true                # 호스트 PID namespace 공유 — 호스트 process 다 보임
  hostIPC: true                # 호스트 IPC namespace 공유
  
  containers:
  - volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:                  # 호스트 fs 접근 — *사실상 escape*
      path: /
```

### 정당화 가능한 경우
- *DaemonSet으로 node 모니터링/관리* (node-exporter, kube-proxy, CNI agent)
- 그 외엔 *거의 항상 위험*

### PSS baseline/restricted는 다 차단
- → *시스템 ns만 예외*, 일반 워크로드는 *명시 정당화 필요*.

---

## 9. *런타임 격리 강화* — gVisor / Kata

### 문제
- 컨테이너 = *호스트 커널 공유* — 커널 exploit으로 *호스트 escape* 가능
- *멀티 테넌시* (다른 고객의 코드 같이 실행)에선 *위험*

### gVisor
- *user-space 커널* — Go로 작성된 *syscall handler*
- 컨테이너의 syscall이 *gVisor 거쳐* 처리 → 호스트 커널과 *간접 접촉*
- *성능 일부 손해*
- Google이 *App Engine·Cloud Run에 사용*

### Kata Containers
- *경량 VM 안에 컨테이너* — *별도 게스트 커널*
- *호스트 커널과 *완전 격리**
- 실행 시간 약간 ↑ (VM 부팅)
- *진짜 격리* 필요한 워크로드

### K8s에서 사용 — RuntimeClass
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

```yaml
spec:
  runtimeClassName: kata       # ← 이 Pod은 Kata로 실행
  containers: ...
```

→ *FaaS, 멀티 테넌트 SaaS*에 적합. 일반 사내 워크로드는 *runc + PSS*로 충분.

---

## 10. 자주 헷갈리는 것

### 10-1. *PSS는 *namespace 단위***
- *cluster 전체*가 아니라 *namespace별로 label*
- → 새 namespace 만들 때마다 *label 박는 자동화* 필요 (Kyverno generate, GitOps template)

### 10-2. *PSS만으론 부족*
- *NetworkPolicy + RBAC + Secret 관리* + *공급망 보안* 다 같이
- *PSS는 *Pod 권한*만*

### 10-3. *runAsNonRoot + image의 USER 누락*
- `runAsNonRoot: true` + image Dockerfile에 `USER` 없음 → *Pod fail* (root로 시작)
- 해결: image에 `USER 1000` 명시 + `runAsUser: 1000`

### 10-4. *readOnlyRootFilesystem 후 *앱 fail**
- 앱이 *`/tmp`나 `/var/run` 등에 write* 시도
- 해결: *그 path만 volume mount* (emptyDir 또는 tmpfs)

### 10-5. *Seccomp profile 너무 strict — 앱 fail*
- 커스텀 profile에서 *필요한 syscall 누락*
- *RuntimeDefault 우선*. 커스텀은 *audit 후 점진 strict*

### 10-6. *Container security context vs Pod*
- 같은 필드 *Pod·Container 둘 다* 가능
- *Container가 override*
- *공통은 Pod, 특수만 Container*

---

## 🎤 면접 빈출 Q&A

### Q1. Pod Security Standards (PSS)는 무엇? 3 profile?
> K8s 1.25+ 내장 admission plugin이 강제. **privileged** (거의 제한 X — 시스템용), **baseline** (알려진 위험 차단 — hostPath, hostNetwork, privileged 등), **restricted** (모범 보안 — runAsNonRoot, cap drop ALL, seccomp, RO root 권장). *namespace label*로 enforce/audit/warn. **PSP (deprecated 1.25)의 후계**.

### Q2. 최소 권한 securityContext 베이스라인?
> `runAsNonRoot: true`, `runAsUser: 1000`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities: { drop: [ALL], add: [NET_BIND_SERVICE if needed] }`, `seccompProfile: { type: RuntimeDefault }`. **PSS restricted profile의 핵심**. 모든 신규 워크로드 *이 베이스라인*에서 시작.

### Q3. Capabilities를 *왜 drop ALL + add 명시*?
> Linux root 권한이 *30+ capability*로 분할. Docker 기본은 *14개 부여* — *불필요한 것까지*. **drop ALL + 필요한 것만 add** — *진짜 필요한 권한만*. 예: nginx → `NET_BIND_SERVICE` (80/443 bind). 불필요한 cap이 *공격 표면*. *SYS_ADMIN, NET_ADMIN, SYS_PTRACE는 거의 root* — 절대 피함.

### Q4. `privileged: true` 가 *왜 위험*?
> *모든 capability + 모든 device + AppArmor/seccomp 비활성 + cgroup 한도 무시 가능*. 컨테이너 안 root가 *사실상 호스트 root*. *kernel module load, 호스트 fs mount, ptrace 다 가능* → *escape 사실상 자유*. **PSS baseline/restricted 차단**. 정당화 가능한 경우는 *CSI/CNI agent* 등 *시스템 컴포넌트만*.

### Q5. `readOnlyRootFilesystem: true` 적용 시 함정?
> 앱이 *`/tmp`나 `/var/run` 등에 write* 필요할 때 fail. 해결: *그 path만 별도 volume mount* (emptyDir 또는 tmpfs). 시작 전 *strace로 write path 식별*. *Logging은 stdout으로* (12-Factor). *cache·temp는 명시적 volume*. **침해 시 변조 어렵게 — supply chain 보안 강화**.

### Q6. gVisor·Kata Containers는 *언제* 필요?
> *컨테이너 ≠ VM 보안 경계*. 호스트 커널 공유 → exploit으로 *escape 가능*. *멀티 테넌시* (다른 고객 코드)에서 위험. **gVisor**: *user-space 커널*로 syscall 가로채. **Kata**: *경량 VM*에 컨테이너 — *완전 커널 격리*. FaaS, 멀티 테넌트 SaaS에 적합. 일반 사내 워크로드는 *runc + PSS*로 충분.

### Q7. PSS의 *audit/warn/enforce* 차이? 도입 흐름?
> **enforce** — 위반 Pod *생성 거부*. **audit** — *audit log에 기록* (생성 허용). **warn** — *kubectl 사용자에 warning* (생성 허용). **도입 흐름**: (1) *모든 namespace에 audit + warn* 적용 → (2) 위반 *audit log·warn 모니터링* → (3) 정리 후 *enforce로 전환*. *바로 enforce하면 *기존 워크로드 폭망**.

---

## 🔗 Cross-reference

- **Linux capability·seccomp·AppArmor·SELinux 베이스** → [`linux-basics/03`](../linux-basics/03-filesystem-and-permissions.md)
- **Container 보안 (Docker 측면)** → [`docker-basics/06`](../docker-basics/06-security.md)
- **NetworkPolicy** → [02-network-policy.md](02-network-policy.md)
- **Admission webhook (Kyverno로 PSS 보완)** → [04-supply-chain.md](04-supply-chain.md)
- **K8s API server Admission 흐름** → [`kubernetes-basics/01`](../kubernetes-basics/01-architecture.md), [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)

---

## 📝 3줄 요약

1. *PSS 3 profile* (privileged/baseline/restricted)을 *namespace label로 enforce*. **restricted = 최소 권한 베이스라인** (runAsNonRoot + cap drop ALL + seccomp + RO root).
2. **`privileged: true` 절대 피함**. *capabilities drop ALL + 필요한 것만 add*. *Host namespace 공유 (hostNetwork/PID/IPC, hostPath) 정당화 필수*.
3. *진짜 격리는 *gVisor/Kata* (멀티 테넌트). *audit/warn → enforce* 점진 도입. *Image의 USER 누락 + runAsNonRoot* 충돌 흔한 함정.

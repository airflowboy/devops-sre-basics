# 06 — Security: 컨테이너 보안의 layer는 어디부터 어디까지인가

> **공식 근거:** [NIST SP 800-190 Application Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final), [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker), [OWASP Container Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Container_Security_Cheat_Sheet.html), [Sigstore docs](https://docs.sigstore.dev/)
>
> **선행:** [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md) (격리 메커니즘), [04-volumes-and-storage.md](04-volumes-and-storage.md) (read-only fs)

---

## 🎯 한 문장

> **컨테이너는 *VM 수준 보안 경계가 아니다*. *호스트 커널을 공유*하므로 *커널 exploit 하나로 탈출 가능*. 그래서 보안은 *layer 여러 겹* — *image 신뢰 + 런타임 권한 최소화 + 격리 강화 + 모니터링*.**

---

## 1. 컨테이너 보안의 7가지 layer

```
┌──────────────────────────────────────────────────┐
│  L7. 공급망: image 출처·서명·SBOM·CVE              │  ← supply chain
├──────────────────────────────────────────────────┤
│  L6. Registry·전송: 권한, mTLS, 스캐닝              │
├──────────────────────────────────────────────────┤
│  L5. Build: Dockerfile 검토, secret 누출 방지       │
├──────────────────────────────────────────────────┤
│  L4. 런타임 권한: USER, capability, privileged 금지 │  ← *가장 자주 빠짐*
├──────────────────────────────────────────────────┤
│  L3. 격리 강화: seccomp, AppArmor/SELinux, RO fs    │
├──────────────────────────────────────────────────┤
│  L2. 호스트: 커널 패치, docker daemon 권한         │
├──────────────────────────────────────────────────┤
│  L1. 네트워크: NetworkPolicy, secret in transit    │
└──────────────────────────────────────────────────┘
```

각 layer 하나가 뚫려도 *다음 layer가 막는* defense-in-depth.

---

## 2. L7. 공급망 — *내가 띄우는 이미지가 진짜 내 것인가*

### 위협
- *typosquatting*: `expres` (가짜) vs `express` (진짜)
- *baseimage 침해*: 베이스 이미지 자체에 백도어
- *registry 침해*: 누군가 내 image tag를 *바꿔치기*
- *의존성 침해*: npm/pip 패키지 maintainer 탈취

### 방어 도구

| 도구 | 역할 |
|---|---|
| **Trivy / Grype** | image 스캔 — 알려진 CVE 검출 |
| **Syft** | SBOM (Software Bill of Materials) 생성 — image에 *무슨 컴포넌트*가 들었는지 명세 |
| **Cosign (Sigstore)** | image *서명* — *내가 만든 게 맞는지* 검증 |
| **Sigstore Rekor** | 서명의 *변경 불가 로그* — 누가 언제 서명했는지 박제 |
| **Sigstore Fulcio** | *키 없이* OIDC 기반 단기 서명 (keyless signing) |
| **SLSA framework** | 공급망 보안의 *수준 표준* (Level 1~4) |

### Cosign keyless signing — *현대적 흐름*
```bash
# GitHub Actions에서 (OIDC 토큰으로)
cosign sign --yes ghcr.io/me/myapp@sha256:abc...
# → Fulcio가 *단기 인증서* 발급해서 서명, Rekor에 *영구 로그* 박힘

# 검증 (배포 직전)
cosign verify ghcr.io/me/myapp@sha256:abc... \
  --certificate-identity=https://github.com/me/myrepo/.github/workflows/build.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

→ *키 관리 부담 없이* 서명. *누가 어떤 워크플로에서 빌드했는지*가 인증서에 박힘.

K8s에서는 *Admission Controller* (Kyverno, Sigstore policy-controller, OPA Gatekeeper)가 *배포 직전에 검증* → 서명 안 된 이미지는 *거부*.

→ K8s 보안 더 깊이: `k8s-security-basics/` (예정).

---

## 3. L5/L6. Build와 Registry — *secret 누출 방지*

### 흔한 사고
- `ARG GITHUB_TOKEN=...` 으로 빌드 → image history에 그대로 남음
- `.env` 파일을 `.dockerignore` 안 하고 `COPY . .` → image에 포함
- 빌드 로그에 secret 출력 → CI 로그에 남음
- 잘못 build한 image를 *public registry에 push*

### 방어
- **BuildKit secret** (`--mount=type=secret`) — image에 안 박힘 ([05 참조](05-dockerfile-and-build.md))
- **`.dockerignore`** 에 `.env`, `*.key`, `*.pem`, `credentials.json` 등
- **CI에서 secret masking** — GitHub Actions·GitLab CI는 secret 자동 마스킹
- **빌드 후 image 스캔** — Trivy로 secret 패턴 검출 (`trivy image --scanners secret`)
- **Registry 권한 분리** — push는 CI만, pull은 K8s service account만

---

## 4. L4. 런타임 권한 — *가장 자주 빠지는 부분*

### 4-1. USER — *root로 실행하지 마라*

기본적으로 컨테이너는 *root (uid 0)* 로 실행. 호스트의 root와 *같은 uid* 라 (USER namespace 안 쓰면) 사실상 동일 권한.

```dockerfile
# ❌ 기본 — root
FROM alpine
RUN apk add nginx
CMD ["nginx", "-g", "daemon off;"]

# ✅ non-root
FROM alpine
RUN apk add nginx && \
    addgroup -S nginx && adduser -S -G nginx nginx && \
    mkdir -p /var/log/nginx /var/lib/nginx /run/nginx && \
    chown -R nginx:nginx /var/log/nginx /var/lib/nginx /run/nginx
USER nginx
CMD ["nginx", "-g", "daemon off;"]
```

K8s에서 강제:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

→ image가 root로 실행되려고 하면 *Pod 자체가 fail*.

### 4-2. Capabilities — *root여도 권한 세분화*

Linux는 root 권한을 *여러 capability*로 쪼개 놓음. *전체 root가 아니라 필요한 capability만* 줄 수 있음.

```bash
# 모든 capability 제거 + 필요한 것만 추가
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
# → 80번 포트 bind만 허용, 다른 root 권한 없음
```

흔한 capability:
| Capability | 의미 |
|---|---|
| `NET_BIND_SERVICE` | 1024 이하 포트 bind (예: 80, 443) |
| `NET_ADMIN` | 네트워크 설정 변경 (iptables 등) |
| `SYS_ADMIN` | *광범위한 시스템 관리* (사실상 root 수준) |
| `SYS_PTRACE` | 다른 프로세스 추적 (gdb 등) |
| `SYS_MODULE` | 커널 모듈 로드 |
| `SYS_TIME` | 시스템 시간 변경 |
| `DAC_OVERRIDE` | 파일 권한 무시 |

K8s에서:
```yaml
securityContext:
  capabilities:
    drop: [ALL]
    add: [NET_BIND_SERVICE]  # 필요 시만
```

→ Pod Security Standards `restricted` 가 이걸 *강제*함. `k8s-security-basics/` (예정).

### 4-3. `--privileged` — *절대 안 됨* (특수 사정 제외)

`--privileged` = *거의 모든 capability + 모든 device 접근 + AppArmor/seccomp 해제*.
컨테이너 안 root가 *사실상 호스트 root*. 격리 *거의 무력화*.

흔한 변명과 대안:
| 변명 | 대안 |
|---|---|
| "Docker-in-Docker 필요" | DinD 대신 *kaniko*, *buildah*, *외부 빌드 서비스* |
| "GPU 써야 함" | `--device=/dev/nvidia0` + `nvidia-container-runtime` |
| "tcpdump 해야 함" | `--cap-add=NET_ADMIN --cap-add=NET_RAW` 만 |
| "FUSE 마운트" | `--cap-add=SYS_ADMIN --device=/dev/fuse` |
| "sysctl 변경" | `--sysctl` 옵션 (필요한 것만) |

→ 면접 빈출: *"`--privileged` 가 왜 위험한가, 대안은?"*

---

## 5. L3. 격리 강화 — *seccomp / AppArmor / SELinux / RO fs*

### Seccomp — *시스템 콜 필터*
- 컨테이너가 *호출할 수 있는 syscall 목록 제한*
- Docker 기본 프로필은 *위험한 syscall* (예: `mount`, `pivot_root`, `init_module`) 차단
- K8s에서는 `securityContext.seccompProfile`

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault   # docker 기본 프로필 사용
    # 또는 Localhost + 커스텀 프로필
```

### AppArmor / SELinux — *MAC* (Mandatory Access Control)
- 파일·네트워크·capability 접근의 *세부 정책*
- AppArmor = Ubuntu·Debian 표준
- SELinux = RHEL·CentOS·Rocky 표준
- docker는 *기본 프로필* 제공 + 컨테이너별 override 가능

### Read-only Root Filesystem
```yaml
securityContext:
  readOnlyRootFilesystem: true
```
- 컨테이너 안에서 `/` 가 read-only → 침해 시 *변조 어려움*
- 앱이 *어딘가에 쓰긴 해야 함* → 명시적 volume (`/tmp` tmpfs 등)

→ ([04 참조](04-volumes-and-storage.md) `--read-only` 패턴)

### 종합 권장 — *최소 권한 보안 컨텍스트*
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false      # setuid 비트로 권한 상승 금지
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
    # add: [NET_BIND_SERVICE]   # 필요 시만
  seccompProfile:
    type: RuntimeDefault
```

→ 이것이 K8s *Pod Security Standards `restricted` profile* 의 핵심.

---

## 6. L2. 호스트 — *docker daemon 자체의 권한*

### Docker socket 의 *진짜 의미*
- `docker.sock` = docker daemon API. *socket에 접근 가능 = root 권한* (사실상)
- 컨테이너 안에 `docker.sock` 마운트하면 → *그 컨테이너에서 호스트 root* (다른 컨테이너 띄울 수 있고, host 디렉터리 mount 가능)
```bash
# ❌ 사실상 root 부여
docker run -v /var/run/docker.sock:/var/run/docker.sock alpine
```

### 호스트 커널 패치
- 컨테이너 격리의 *근본*은 *Linux 커널*. 커널 취약점은 *모든 컨테이너에 영향*.
- *호스트 커널 정기 업데이트*가 컨테이너 보안의 베이스
- Live patching (kpatch, kgraft) 활용 시 재부팅 최소화

### Rootless Docker / Podman
- *docker daemon 자체를 일반 사용자로* 실행 (root 아님)
- 컨테이너 탈출해도 *호스트에선 일반 사용자*
- 일부 제약 (privileged ports 80 사용 불가 등)

---

## 7. L1. 네트워크 — *컨테이너 간 통신 통제*

### Docker
- Docker만으로는 *컨테이너 간 통제 도구 약함*
- user-defined network로 격리만 (다른 네트워크와 분리)

### K8s NetworkPolicy
- *어떤 Pod가 어떤 Pod와 통신할 수 있는지* 선언적 정책
- 기본은 *모두 허용* → `default-deny + 명시적 allow` 패턴
- 구현은 CNI 플러그인 (Calico, Cilium 등)

→ `k8s-security-basics/` (예정) 에서 더.

### Secrets in transit
- Pod 간 통신 *암호화* — Service Mesh (Istio mTLS) 또는 *앱 레벨 TLS*
- 외부와의 통신은 당연히 *TLS*

---

## 8. Container Escape — *실제 사고 시나리오*

### 시나리오 A. `--privileged` + `cgroup mount`
- privileged 컨테이너에서 *호스트의 cgroup을 직접 마운트* → release_agent 활용해 *호스트에서 임의 명령 실행*
- Defense: `--privileged` 금지

### 시나리오 B. Docker socket 마운트
- `docker.sock` 마운트한 컨테이너 → `docker run -v /:/host` 새 컨테이너 띄움 → *호스트 fs 전체 접근*
- Defense: socket 마운트 금지. 꼭 필요하면 *socket proxy* (필터링)

### 시나리오 C. 커널 취약점 (예: Dirty Pipe, runC CVE 2024)
- 컨테이너 안에서 *호스트 커널 취약점 exploit*
- Defense: 호스트 커널 패치, seccomp/AppArmor 추가 방어

### 시나리오 D. capability `SYS_ADMIN` + 호스트 디바이스
- SYS_ADMIN으로 *호스트 디바이스 mount* → 호스트 fs 접근
- Defense: cap-drop ALL

### 시나리오 E. *gVisor/Kata*로 *진짜 격리* — 컨테이너 ≠ 보안 경계 해결책
- gVisor: *user-space 커널*이 syscall 가로채 (격리 강함)
- Kata: 컨테이너 = *경량 VM*. 호스트 커널 공유 X.
- 비용: 성능 일부 손해. 보안이 중요한 멀티테넌트 환경 (FaaS 등) 채택.

---

## 9. Image Scanning — *CI 게이트로*

### Trivy 흐름
```bash
# 로컬 스캔
trivy image nginx:latest

# CI 게이트 (CVE 발견 시 fail)
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
```

### 무엇을 검출하나
- *알려진 CVE* (NVD, GitHub Advisory, vendor DB)
- *secret 누출* (API key 패턴 매칭)
- *misconfiguration* (Dockerfile best practice 위반)
- *라이선스* (특정 라이선스 차단)

### SBOM 생성 (Syft)
```bash
syft myapp:latest -o spdx-json > sbom.json
# 또는 Trivy
trivy image --format spdx-json myapp:latest > sbom.json
```
→ 나중에 *새 CVE 발견* 시 SBOM 검색해 *영향받는 image 빠르게 찾음*.

→ 운영에서 정석: *CI에서 build 후 → Trivy 스캔 → 통과 시 push → Cosign 서명 → admission controller가 검증 후 K8s에 배포*.

---

## 10. 흔한 함정 정리

### 10-1. *베이스 이미지 정기 업데이트 안 함*
- 베이스에 CVE 누적 → 우리 image도 취약
- → CI에 *주기적 rebuild* (예: 매주) + Trivy 스캔

### 10-2. *Dockerfile에 root 안 바꿈*
- 별일 없어 보이지만 *침해 시 피해 막대*
- → `USER` 명시. K8s에서 `runAsNonRoot: true` 로 강제

### 10-3. *docker socket 마운트*
- "편의를 위해" 한 번 마운트하면 *호스트 권한*. 절대 안 됨.

### 10-4. *image scanning 만 하고 *실제 조치 안 함*
- Scan 결과 *수십 개 CRITICAL*인데 *그냥 push*
- → CI 게이트로 *자동 차단*. 또는 *exception 명시적 등록* 프로세스.

### 10-5. *secrets를 environment variable로*
- `ENV API_KEY=xxx` 박으면 image에 남음
- 환경 변수 자체도 `/proc/{pid}/environ` 으로 *다른 프로세스에서 볼 수 있음*
- → BuildKit secret + 런타임 시 Vault/Secrets Manager에서 *fetch*

### 10-6. *공급망 검증 없음*
- public registry image를 *그냥 신뢰*
- → 최소한 *digest pin*, 가능하면 *내부 mirror + 서명 검증*

---

## 🎤 면접 빈출 Q&A

### Q1. 컨테이너 보안에서 *가장 자주 빠지는* 항목은?
> *Container를 root로 실행하는 것*. Dockerfile에 `USER` 명시 없으면 기본 root, 그리고 USER namespace 안 쓰면 *호스트의 root와 동일 uid*. 침해 시 피해 막대. → Dockerfile에 `USER`, K8s에서 `runAsNonRoot: true` 강제.

### Q2. `--privileged` 가 위험한 이유와 대안?
> *거의 모든 capability + 모든 device 접근 + AppArmor/seccomp 해제*. 컨테이너 안 root가 *사실상 호스트 root*. 격리 무력화. 대안: 필요한 *capability만 add* (`--cap-add=NET_BIND_SERVICE` 등), 필요한 *device만 마운트* (`--device=...`), 필요한 *sysctl만 변경* (`--sysctl`).

### Q3. Container escape 시나리오 하나 설명?
> docker socket 마운트가 대표 사례. 컨테이너 안에 `/var/run/docker.sock` 마운트하면 그 컨테이너에서 *docker daemon API* 사용 가능 → `docker run -v /:/host` 새 컨테이너 띄워 *호스트 fs 전체 접근*. 사실상 호스트 root. → 절대 마운트 금지, 꼭 필요하면 *socket proxy로 API 필터링*.

### Q4. 이미지의 무결성·출처를 어떻게 보장?
> *Cosign으로 서명* — Sigstore keyless 방식이 현대적. CI에서 build 후 OIDC 토큰으로 Cosign 서명 → Sigstore Rekor에 *변경 불가 로그*. 배포 시 K8s admission controller (Kyverno / Sigstore policy-controller) 가 *서명 검증* → 통과한 image만 띄움. + *Digest pin*으로 tag 위변조 방어.

### Q5. SBOM은 왜 필요한가요?
> *Image에 무슨 컴포넌트·버전이 들었는지* 명세. *나중에 새 CVE 발견* (예: log4shell) 시 SBOM 검색해 *어느 image가 영향받는지 즉시 파악*. 컴플라이언스 감사에서도 *공급망 가시성*의 증거.

### Q6. Pod Security Standards `restricted` profile이 강제하는 것은?
> *runAsNonRoot*, *allowPrivilegeEscalation: false*, *capabilities drop ALL + 최소만 add*, *seccompProfile RuntimeDefault*, *읽기 전용 root FS 권장*, *host namespace 공유 금지*, *hostPath volume 금지* 등. 컨테이너 보안의 *최소 권한 베이스라인을 K8s가 강제*.

---

## 🔗 Cross-reference

- **격리 메커니즘 자체** → [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md)
- **Read-only fs / volume** → [04-volumes-and-storage.md](04-volumes-and-storage.md)
- **Build에서 secret 안 박히게** → [05-dockerfile-and-build.md](05-dockerfile-and-build.md)
- **K8s Pod Security Standards / NetworkPolicy / Admission** → `k8s-security-basics/` (예정)
- **CI에서 Trivy 게이트 / Cosign 서명** → `cicd-gitops-basics/` (예정)
- **Vault·Secrets Manager** (런타임 secret 관리) → `aws-basics/` (예정)

---

## 📝 3줄 요약

1. *컨테이너 ≠ VM 보안 경계*. 호스트 커널 공유 → defense-in-depth 필수.
2. 가장 자주 빠지는 것 — *root 실행, `--privileged`, docker socket 마운트, secret을 ARG/ENV로*. 이 4개만 안 해도 절반 안전.
3. 현대적 흐름 = *Trivy 스캔 → Cosign keyless 서명 → admission controller 검증* + *minimum securityContext* (runAsNonRoot + cap-drop ALL + seccomp + read-only fs).

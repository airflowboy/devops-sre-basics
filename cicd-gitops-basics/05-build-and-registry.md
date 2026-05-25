# 05 — Build & Registry: 이미지 생성·전송·검증의 공급망 보안

> **공식 근거:** [BuildKit](https://docs.docker.com/build/buildkit/), [Kaniko](https://github.com/GoogleContainerTools/kaniko), [Sigstore](https://docs.sigstore.dev/), [SLSA framework](https://slsa.dev/), [OCI Distribution Spec](https://github.com/opencontainers/distribution-spec)
>
> **선행:** [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md), [`docker-basics/06`](../docker-basics/06-security.md)

---

## 🎯 한 문장

> **Build = *Dockerfile (또는 buildpack) → OCI image*, Registry = *image 저장·인증·서명 검증의 HTTP 서비스*. 현대 공급망 보안 = *BuildKit/Kaniko로 안전한 빌드 + cosign으로 서명 + SBOM 생성 + admission controller로 배포 시 검증* 4박자.**

---

## 1. CI에서 image 빌드 — 3가지 방식

| 방식 | Docker daemon 필요? | 격리 | 비고 |
|---|---|---|---|
| **dockerd + docker build** | ✅ | 약 | 옛 방식. CI runner에 docker.sock 마운트 필요 (privileged 경향) |
| **BuildKit (buildx)** | ✅ (또는 standalone buildkitd) | 중 | Docker의 *현대 builder*. cache/parallel/secret 강력. |
| **Kaniko** | ❌ | 강 | *컨테이너 안에서 daemon 없이* 빌드. K8s/CI에 적합. |
| **Buildah / Podman** | ❌ | 강 | Red Hat 진영. *rootless 가능*. |
| **img / ko / Jib** | ❌ | 강 | 언어 특화 (Jib = JVM, ko = Go) — Dockerfile 없이 빌드 |

### Why "daemon 없이"가 중요한가
- CI runner에 dockerd 띄우면 *그 컨테이너가 사실상 root* (docker socket = host root와 동급)
- *K8s Pod 안에서 build* 시 *privileged* 필요 → 보안 사고
- Kaniko·Buildah는 *rootless / unprivileged*로 build → CI 보안 강함

### Jib 예시 (JVM)
```
mvn jib:build → 이미지 빌드 + registry push
              (Dockerfile X, daemon X)
```
- 같은 base 위에 *변경된 layer만* push → 빠름
- *PayLane/Korbit 같은 한국 핀테크가 자주 채택* (Kotlin/Spring 표준)

→ 이미지 빌드 깊이는 [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md).

---

## 2. Registry — *HTTP API 서버*

OCI Distribution Spec 따르는 모든 registry가 *같은 API*. ECR / GHCR / Docker Hub / Harbor / Artifactory / Nexus.

### 인증 — *Bearer Token 패턴*
```
Client → GET /v2/myrepo/manifests/v1 (no auth)
Registry → 401 + WWW-Authenticate: Bearer realm="...", service="...", scope="repository:myrepo:pull"
Client → 인증 서버에 토큰 요청 (basic auth 등)
Authenticator → JWT token (scoped to 그 repo, pull only)
Client → GET ... with Bearer {token} → 200
```

### 클라우드 registry의 *임시 자격증명* 패턴
- **AWS ECR**: `aws ecr get-login-password` → 12시간 유효 token. CI에선 OIDC + IAM Role.
- **GCR / Artifact Registry**: Workload Identity Federation
- **GHCR**: workflow의 `GITHUB_TOKEN` 또는 PAT

→ *정적 registry 비밀번호 박지 말 것*. OIDC가 정답.

### Mirror / Proxy / Pull-through Cache
- public registry 직접 의존 → *rate limit, network outage, supply chain* 문제
- 사내 *mirror* 둠 (Harbor·Nexus·ECR pull-through)
- 사내 mirror에서만 pull → *외부 의존 차단 + 캐싱*
- *모든 base image도 사내 mirror 거치게* — 정책 (admission controller로 강제 가능)

---

## 3. Build Cache 전략 — *CI 시간이 곧 비용*

### 4가지 cache 모드 (BuildKit)
| 모드 | 위치 | 비고 |
|---|---|---|
| `type=inline` | image 안 metadata | 가장 단순. 같은 image를 다시 빌드 시 layer 재사용. |
| `type=registry,ref=...` | 별도 registry tag | 가장 일반적. CI runner 간 공유 가능. |
| `type=local,dest=...` | 로컬 디스크 | self-hosted runner에 적합 (영구 보존) |
| `type=gha` | GitHub Actions cache | GitHub Actions 전용. 무료 quota. |

```yaml
# GitHub Actions에서
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/me/myapp:${{ github.sha }}
    cache-from: type=registry,ref=ghcr.io/me/myapp:buildcache
    cache-to:   type=registry,ref=ghcr.io/me/myapp:buildcache,mode=max
```

- `mode=max` — 모든 중간 layer 캐시 (multi-stage 빌드에 핵심)
- `mode=min` — 최종 layer만

### Cache hit 안 될 때
- Dockerfile 순서 (자주 안 바뀌는 명령이 위에 — [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md))
- `COPY` 한 파일이 매번 *timestamp 다름* → 해시 다름 → cache miss
- `.dockerignore`로 *불필요 파일 제외*

---

## 4. SBOM (Software Bill of Materials) — *image에 무엇이 들었나*

### 왜
- 새 CVE 발견 (예: log4shell) → *어느 image가 영향*인가
- *수동으로 모든 image 스캔*은 *너무 늦음*
- *사전에 SBOM 생성·저장*해 두면 *grep 한 번*으로 영향 범위 파악

### 생성 도구
- **Syft** — SBOM 생성 표준. SPDX·CycloneDX format.
  ```bash
  syft myapp:latest -o spdx-json > sbom.json
  ```
- **Trivy** — 스캔 + SBOM 한 번에
  ```bash
  trivy image --format spdx-json myapp:latest > sbom.json
  ```
- **Docker buildx** — `--sbom=true` 옵션으로 image attestation에 포함

### 저장
- SBOM 자체를 *registry의 attestation*으로 push (cosign으로)
- 또는 별도 SBOM 저장소 (DependencyTrack 등)

### 활용
- 새 CVE → *영향받는 image 검색* → *cluster의 해당 image 사용처* 파악 → *우선순위 패치*
- 컴플라이언스 감사 — *공급망 가시성 증거*

---

## 5. Vulnerability Scanning — *CI 게이트로*

### Trivy
```bash
# 로컬
trivy image myapp:latest

# CI 게이트 — HIGH/CRITICAL 발견 시 fail
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# 보조 스캐너
trivy image --scanners vuln,secret,misconfig myapp:latest
```

### CI 통합 모범
```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: 'HIGH,CRITICAL'
    exit-code: '1'
    ignore-unfixed: true   # 패치 없는 CVE는 무시 (잡음 줄임)
    format: 'sarif'
    output: 'trivy-results.sarif'

- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

→ SARIF로 *GitHub Security 탭에 표시* — 가시성·tracking.

### Suppress / Allowlist
- false positive 또는 *해결 불가한 CVE* → `.trivyignore`로 *명시적으로* 제외 (정당화 commit message 필수)

---

## 6. Image 서명 — Cosign (Sigstore)

### 옛 방식 (Notary v1, DCT)
- *별도 키쌍 관리*
- *키 분실·유출 위험*
- 복잡한 키 회전

### 현대 — Sigstore Cosign (Keyless)
```
CI workflow가 OIDC ID Token 받음 (예: GitHub Actions, GitLab CI)
       ↓
Cosign이 그 토큰을 Fulcio (Sigstore CA)에 제시
       ↓
Fulcio가 *짧은 인증서* 발급 (예: 10분 유효)
       ↓
Cosign이 그 인증서로 image digest 서명
       ↓
서명·인증서·timestamp가 Rekor (변경 불가 로그)에 박힘
       ↓
인증서는 *바로 폐기* (재사용 안 함)
```

```bash
# 서명 (GitHub Actions에서, 자동 OIDC)
cosign sign --yes ghcr.io/me/myapp@sha256:abc...

# 검증 (배포 직전)
cosign verify ghcr.io/me/myapp@sha256:abc... \
  --certificate-identity-regexp='^https://github.com/me/myrepo/' \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

### 키 없는 게 왜 안전한가
- 키가 없으면 *키 유출 불가능*
- *각 서명마다 새 인증서* — 한 서명 침해 ≠ 다른 서명 침해
- *서명의 *맥락 (subject claim)*이 인증서에 박힘* — "이 image는 *어느 repo의 어느 workflow에서* 만들어졌나"

→ [`docker-basics/06`](../docker-basics/06-security.md) 의 supply chain security 와 직결.

---

## 7. Admission에서 *서명 검증* — 배포 시 강제

K8s admission controller (Sigstore policy-controller, Kyverno, OPA Gatekeeper) 가 *배포 시 image 서명 확인*. 통과 못 하면 *Pod 생성 거부*.

### Kyverno 예시
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
  - name: verify-cosign-signature
    match:
      any:
      - resources:
          kinds: [Pod]
    verifyImages:
    - imageReferences:
      - "ghcr.io/me/*"
      attestors:
      - entries:
        - keyless:
            subject: "https://github.com/me/myrepo/.github/workflows/build.yml@refs/heads/main"
            issuer: "https://token.actions.githubusercontent.com"
```

→ *모든 ghcr.io/me/\* 이미지*는 *위 workflow에서 서명된 것*만 허용. 다른 건 차단.

### Sigstore Policy Controller
- Kyverno보다 *서명 검증 전용*. 간단한 ClusterImagePolicy로.
- Sigstore 생태계와 가장 native 통합.

→ 면접 빈출: *"image 무결성·출처를 어떻게 보장하나요"* — CI에서 cosign 서명 + cluster에서 admission 검증.

---

## 8. SLSA Framework — *공급망 보안의 수준 모델*

| Level | 요구 |
|---|---|
| **L1** | 빌드 프로세스 *자동화 + provenance 생성* |
| **L2** | + 빌드 서비스가 *호스팅된 환경* (Hosted Runner). version-controlled build. |
| **L3** | + *Build 환경 비공개* (다른 사람 못 봄), *provenance 인증 가능* (서명) |
| **L4** | + *완전 재현성* (hermetic build, parameterless) — 매우 빡셈 |

### Provenance란
- *어떻게 이 image가 만들어졌나*의 metadata
- builder identity, source repo + commit, build params, materials (input)
- *SLSA Provenance v1* spec
- 보통 *attestation*으로 image와 함께 registry에 push

```bash
# Cosign attestation
cosign attest --predicate provenance.json myapp:latest
```

→ 운영에선 *SLSA L2~L3*가 현실적 목표. L4는 매우 어려움.

---

## 9. 자주 헷갈리는 것

### 9-1. *Docker daemon 없이 빌드* — Kaniko·Buildah가 그 답
- CI에서 *privileged 필요 없음*. K8s Pod 안에서 안전하게 빌드.
- *Docker-in-Docker* 패턴은 *피해야 할 것*.

### 9-2. *서명·SBOM 만들었는데 *검증 안 함* — 의미 X
- 만들기만 하고 admission에서 *안 막으면* 효과 0
- *cluster의 admission policy*까지 갖춰야 사이클 완성

### 9-3. *Cosign keyless가 *키 0개*가 아님*
- 인증서는 *짧게 발급되는 임시 키*
- *영구 키쌍이 없다*는 뜻. *서명 자체는 인증서로*.
- "*keyless*"가 *signature 없다*는 의미 X

### 9-4. *Image digest 사용 안 하면 서명 검증 무력*
- `myapp:v1.0` (tag) 으로 검증하면 *그 사이 다른 image로 swap 가능*
- 항상 *digest pin* (`@sha256:abc...`)

### 9-5. *Registry에서 image *삭제* 안 됨*
- digest는 *불변* — 한 번 push되면 *garbage collection까지 영구*
- *옛 image 정리*는 *retention policy* (lifecycle rule) — ECR의 *N개 이상 / N일 이상*

### 9-6. *Multi-arch build*
- ARM·x86 동시 빌드 — `docker buildx build --platform linux/amd64,linux/arm64 ...`
- *각 arch별로 별도 layer* — image manifest list가 *arch별 image 가리킴*

---

## 🎤 면접 빈출 Q&A

### Q1. CI에서 docker daemon 없이 image build 어떻게?
> **Kaniko** (Google), **Buildah** (Red Hat), **Jib** (Google, JVM 전용). 모두 *daemon 없이 컨테이너 안에서 build* + push. CI runner에 *privileged 필요 없음*. K8s Pod 안에서 안전한 빌드. Docker-in-Docker는 *피해야 할 패턴*.

### Q2. Cosign keyless 서명 원리는?
> *영구 키쌍 없음*. CI workflow가 *OIDC ID Token 받아 Fulcio (Sigstore CA)에 제시* → *짧은 임시 인증서 발급* (예: 10분) → 그 인증서로 image digest 서명 → 서명·인증서가 *Rekor 변경 불가 로그에 박힘* → 인증서 바로 폐기. *키 유출 불가능* (없으니), *각 서명마다 새 인증서*, *맥락 (어느 repo의 어느 workflow)*이 인증서에 박힘.

### Q3. SBOM은 왜 필요? 어떻게 활용?
> *image에 무엇이 들었는지 명세*. 새 CVE 발견 (log4shell 같은) 시 *영향받는 image 즉시 검색* — 수동 스캔으론 너무 늦음. *Syft·Trivy로 생성*, registry attestation으로 저장, 컴플라이언스 감사에 *공급망 가시성 증거*. SPDX / CycloneDX 표준 format.

### Q4. Image 무결성·출처를 cluster에서 강제하려면?
> CI에서 cosign 서명 + cluster에 *admission controller* (Sigstore policy-controller, Kyverno) → *배포 시 서명 검증* → 통과 못 하면 *Pod 생성 거부*. trust 조건은 *어느 repo의 어느 workflow까지*까지 빡세게.

### Q5. SLSA framework란?
> *공급망 보안의 수준 모델* (Level 1~4). L1: 자동화 + provenance. L2: hosted runner + version control. L3: 빌드 환경 격리 + provenance 인증. L4: 완전 재현성 (hermetic). 보통 운영 목표는 *L2~L3*. L4는 매우 어려움 (timestamp·build id 등 비결정성 제거).

### Q6. Image 빌드 cache 전략은?
> BuildKit `cache-from / cache-to`. (1) `inline` — image 안에 (간단). (2) `registry` — 별도 tag로 push (가장 일반적, CI runner 공유). (3) `local` — self-hosted runner 영구 디스크. (4) `gha` — GitHub Actions cache. `mode=max`로 *multi-stage 모든 layer* 캐시.

### Q7. 사내 image registry mirror가 필요한 이유?
> public registry 직접 의존 = *rate limit, outage, supply chain* 위험. 사내 mirror (Harbor/Nexus/ECR pull-through) 두면 *외부 의존 차단 + 캐싱*. *모든 base image도 mirror 거치게 강제* — admission policy로 *외부 직접 pull 차단*. 컴플라이언스에서도 *어떤 image가 들어왔는지 통제 가능*.

---

## 🔗 Cross-reference

- **Dockerfile / BuildKit / multi-stage** → [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md)
- **컨테이너 보안·escape·SBOM 개념** → [`docker-basics/06`](../docker-basics/06-security.md)
- **K8s admission controller (서명 검증 강제)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md), [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)
- **OIDC (CI ↔ cloud + Cosign 서명의 베이스)** → [02-github-actions.md](02-github-actions.md)
- **K8s NetworkPolicy / Pod Security Standards / Kyverno 정책 깊이** → `k8s-security-basics/` (예정)
- **AWS ECR / IAM** → `aws-basics/` (예정)

---

## 📝 3줄 요약

1. *Docker daemon 없는 빌드 (Kaniko·Buildah·Jib)*가 CI 보안의 베이스. Docker-in-Docker는 피할 것.
2. 공급망 보안 4박자: **BuildKit/Kaniko 안전 빌드** + **Cosign keyless 서명** + **SBOM 생성/저장** + **Admission controller 배포 시 검증**.
3. *SLSA L2~L3 현실 목표*. *Registry mirror로 외부 의존 차단*, *digest pin으로 서명 검증 무력화 방지*.

# 04 — Supply Chain Security: Trivy · Cosign · SBOM · Admission

> **공식 근거:** [Sigstore docs](https://docs.sigstore.dev/), [SLSA framework](https://slsa.dev/), [Kyverno docs](https://kyverno.io/), [Trivy docs](https://aquasecurity.github.io/trivy/)
>
> **선행:** [`docker-basics/06`](../docker-basics/06-security.md), [`cicd-gitops-basics/05`](../cicd-gitops-basics/05-build-and-registry.md)

---

## 🎯 한 문장

> **공급망 보안 4박자 = *Image scan (Trivy)* + *SBOM 생성 (Syft)* + *Image 서명 (Cosign keyless)* + *Admission 검증 (Kyverno/Gatekeeper/Sigstore policy-controller)*. CI에서 *서명*해도 cluster admission이 *검증 안 하면 무의미*. SLSA framework가 *수준 모델*.**

---

## 1. *공급망 위협 모델*

### 위협 시나리오
1. **Typosquatting** — `expres` (가짜) vs `express` — npm/pip 패키지
2. **Dependency confusion** — public registry의 *동일 이름 패키지*가 사내 mirror 가로채기
3. **Compromised maintainer** — 의존성 maintainer 계정 탈취 → 백도어 PR
4. **Base image 침해** — `nginx:latest`에 malware 주입
5. **Registry swap** — 누가 같은 tag로 다른 image push
6. **Build environment 침해** — CI runner에 백도어
7. **Insecure base image** — 오래된 CVE 다수

### Defense layers
```
1. Source code   — git signing, SAST, dependency audit
2. Dependencies  — SCA (Trivy, Snyk), Dependabot, SBOM
3. Build         — BuildKit secret, ephemeral runner, build provenance
4. Artifact      — image signing (cosign), SBOM attestation
5. Registry      — mTLS, scan-on-push, immutable tags
6. Deploy        — admission policy (signed only), digest pin
7. Runtime       — Falco, runtime CVE alerts
```

---

## 2. Image Vulnerability Scanning — *Trivy*

### Trivy 사용
```bash
# 로컬 image
trivy image myapp:latest

# severity 필터 + CI 게이트
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# 보조 스캐너
trivy image --scanners vuln,secret,misconfig myapp:latest

# fixed CVE 만 (실제 패치 가능)
trivy image --ignore-unfixed myapp:latest

# JSON 출력
trivy image --format json -o report.json myapp:latest

# SARIF (GitHub Security)
trivy image --format sarif -o report.sarif myapp:latest
```

### 검출 대상
- **알려진 CVE** — NVD, GitHub Advisory, vendor DB
- **Secret 누출** — API key 패턴 매칭
- **Misconfiguration** — Dockerfile/K8s/Terraform best practice 위반
- **License** — 특정 license 차단 가능

### CI 통합 (GitHub Actions)
```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: 'HIGH,CRITICAL'
    exit-code: '1'
    ignore-unfixed: true
    format: 'sarif'
    output: 'trivy-results.sarif'

- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

→ GitHub Security 탭에 *발견사항 자동 표시* — 가시성·tracking.

### 정기 rescan
- *image 한 번 빌드 후 *새 CVE 발견* 시 영향
- *주기적 rescan* (예: nightly) → 새 CVE 발견 시 알람
- *rebuild + redeploy* cycle 자동화

---

## 3. SBOM — *Software Bill of Materials*

### 왜
- *image에 무엇이 들었나* 명세
- 새 CVE 발견 (log4shell 같은) 시 *영향받는 image 즉시 검색*
- 컴플라이언스 — *공급망 가시성 증거*

### 생성
```bash
# Syft (Anchore)
syft myapp:latest -o spdx-json > sbom.json
syft myapp:latest -o cyclonedx-json > sbom.json

# Trivy 도
trivy image --format spdx-json myapp:latest > sbom.json

# Docker buildx
docker buildx build --sbom=true --provenance=true -t myapp:v1 .
```

### Format
- **SPDX** (Linux Foundation) — 표준화
- **CycloneDX** (OWASP) — 더 풍부한 metadata, 보안 focus
- 둘 다 *주요 도구 지원*

### 저장
- *registry attestation*으로 push (Cosign attest)
- *DependencyTrack* 등 SBOM 관리 도구
- *git에 같이 commit* (audit trail)

### 활용
```bash
# 새 CVE 발견 시 SBOM 검색
grep "log4j-core@2.14" sbom.json   # 영향 받는 image 즉시 식별
```

---

## 4. Cosign — *Image 서명*

[`cicd-gitops-basics/05`](../cicd-gitops-basics/05-build-and-registry.md) Section 6 참조. 핵심:

### Keyless signing (현대 표준)
- *영구 키쌍 없음*
- *CI workflow가 OIDC ID Token → Fulcio (Sigstore CA) → 짧은 인증서 (10분) → 서명*
- *서명·인증서·timestamp가 Rekor (변경 불가 log)에 박힘*
- *인증서 즉시 폐기*

```bash
# 서명 (GitHub Actions OIDC 자동)
cosign sign --yes ghcr.io/me/myapp@sha256:abc...

# 검증
cosign verify ghcr.io/me/myapp@sha256:abc... \
  --certificate-identity-regexp='^https://github.com/me/myrepo/' \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

### 왜 *키 없는* 게 안전한가
- 키 없음 → *유출 불가능*
- 각 서명마다 *새 임시 인증서* — 한 서명 침해 ≠ 다른 침해
- 서명의 *맥락* (어느 repo의 어느 workflow)이 *인증서에 박힘*

### SBOM·Provenance attestation
```bash
# SBOM을 attestation으로 push
cosign attest --predicate sbom.json myapp:latest

# Provenance (SLSA)
cosign attest --predicate provenance.json --type slsaprovenance myapp:latest
```

---

## 5. Admission Policy — *Cluster의 Enforcement*

### 발상
- CI에서 *서명·SBOM·스캔* 다 해도 *cluster admission이 검증 안 하면 무의미*
- → **admission policy가 *enforcement point**

### 도구 비교

| 도구 | 특징 |
|---|---|
| **Kyverno** | YAML로 정책 정의 (Rego 안 배움), K8s native, validate + mutate + generate |
| **OPA Gatekeeper** | Rego 정책, 복잡한 logic, 많은 community 정책 |
| **Sigstore policy-controller** | *cosign 서명 검증 전용*, 가장 native |
| **Pod Security Admission** | PSS 강제 (1.25+ 내장) |

→ 보통 *Kyverno*가 *진입 장벽 낮음*. *복잡한 정책은 Gatekeeper*.

---

## 6. Kyverno — *YAML 기반 정책*

### 정책 예시 — 서명된 image만 허용
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
  - name: verify-cosign
    match:
      any:
      - resources:
          kinds: [Pod]
    verifyImages:
    - imageReferences:
      - "ghcr.io/my-org/*"
      attestors:
      - entries:
        - keyless:
            subject: "https://github.com/my-org/my-repo/.github/workflows/build.yml@refs/heads/main"
            issuer: "https://token.actions.githubusercontent.com"
```

→ ghcr.io/my-org/* image는 *위 workflow에서 서명된 것만* 허용.

### 정책 예시 — 모든 image digest pin 강제
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-digest
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Image must use digest (@sha256:...) not tag"
      pattern:
        spec:
          containers:
          - image: "*@sha256:*"
```

### 정책 예시 — 모든 Pod에 standard labels 강제
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-standard-labels
spec:
  rules:
  - name: require-labels
    match:
      any:
      - resources:
          kinds: [Pod, Deployment, StatefulSet]
    validate:
      message: "Must have app.kubernetes.io/{name,owner} labels"
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
            app.kubernetes.io/owner: "?*"
```

### Mutating
```yaml
# 모든 Pod에 자동 sidecar 주입
- name: inject-sidecar
  match: [...]
  mutate:
    patchStrategicMerge:
      spec:
        containers:
        - name: log-shipper
          image: log-shipper:v1
```

### Generate
```yaml
# 새 namespace 생성 시 default NetworkPolicy 자동
- name: generate-default-deny
  match:
    any:
    - resources:
        kinds: [Namespace]
  generate:
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    name: default-deny
    namespace: "{{ request.object.metadata.name }}"
    data:
      spec:
        podSelector: {}
        policyTypes: [Ingress]
```

→ *Multi-tenancy 자동화*에 매우 유용.

---

## 7. OPA Gatekeeper — *Rego 정책*

### Constraint Template + Constraint
```yaml
# Template — 정책 logic
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: {type: string}
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      
      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing labels: %v", [missing])
      }

---
# Constraint — 실제 적용
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-labels
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: [Pod]
  parameters:
    labels: [app, owner]
```

### Kyverno vs Gatekeeper

| | Kyverno | Gatekeeper |
|---|---|---|
| 언어 | YAML | Rego (학습 곡선 ↑) |
| 표현력 | 중 | 강 (Rego는 임의 logic) |
| 진입 장벽 | 낮음 | 높음 |
| Mutate / Generate | ✓ | Mutate만 (alpha) |
| Community 정책 | 많음 | 많음 |

→ 보통 *Kyverno로 시작*, *복잡한 정책은 Gatekeeper*.

---

## 8. Sigstore Policy Controller — *서명 검증 전용*

### 특징
- *cosign 서명 검증에 특화*
- Sigstore 생태계와 *native 통합*
- 더 단순 (Kyverno는 *모든 종류 정책*, 이건 *서명만*)

### ClusterImagePolicy
```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: signed-by-github-actions
spec:
  images:
  - glob: "ghcr.io/my-org/**"
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://token.actions.githubusercontent.com
        subject: "https://github.com/my-org/my-repo/.github/workflows/build.yml@refs/heads/main"
      ctlog:
        url: https://rekor.sigstore.dev
```

→ Kyverno보다 *간결*. 서명만 필요한 환경에 적합.

---

## 9. SLSA Framework — *공급망 보안 수준 모델*

### 4 Level
| Level | 요구 |
|---|---|
| **L1** | 빌드 프로세스 *자동화 + provenance 생성* |
| **L2** | + 빌드 서비스가 *hosted (단일 사용자가 위변조 못 함)* + version-controlled |
| **L3** | + *빌드 환경 비공개 (격리)*, *provenance 인증 가능 (서명)* |
| **L4** | + *완전 재현성 (hermetic)*, *parameterless* — 매우 어려움 |

### Provenance
- *어떻게 이 image가 만들어졌나*의 metadata
- *builder identity, source repo + commit, build parameters, materials* (input)
- *SLSA Provenance v1* spec

```bash
# Cosign으로 provenance attestation
cosign attest --predicate provenance.json --type slsaprovenance myapp:latest

# 검증
cosign verify-attestation --type slsaprovenance --certificate-identity ... myapp:latest
```

### 운영 현실
- *SLSA L2~L3가 현실적 목표*
- L4는 매우 어려움 (timestamp·build id 비결정성 제거)
- GitHub Actions + cosign + provenance = *자연스럽게 L3*

---

## 10. 자주 헷갈리는 것

### 10-1. *서명만 하고 *검증 안 함**
- CI에서 cosign 서명 → admission policy 없음 → *서명 의미 X*
- *cluster의 Kyverno/policy-controller 까지 갖춰야 사이클 완성*

### 10-2. *Cosign keyless가 *키 0개*가 아님*
- 인증서는 *짧게 발급되는 임시 키*
- *영구 키쌍이 없다*는 뜻
- "keyless"가 *signature 없다*는 의미 X

### 10-3. *Tag로 검증*
- `cosign verify myapp:v1.0` — *그 사이 다른 image로 swap 가능*
- *항상 digest pin*: `myapp@sha256:abc...`

### 10-4. *Trivy ignore-unfixed 안 씀 → 노이즈*
- *패치 없는 CVE*도 차단 → *영원히 push 불가*
- `--ignore-unfixed` 로 *fixable 만 차단*

### 10-5. *Kyverno 정책 *audit → enforce 단계 건너뜀**
- 바로 enforce → *기존 워크로드 다 차단*
- 정책: `validationFailureAction: Audit` → 정리 → `Enforce`

### 10-6. *Sigstore의 *public good infra 의존**
- Fulcio/Rekor는 *공개 서비스*
- 사내 *air-gapped 환경*에선 자체 운영 필요 (Fulcio·Rekor self-host)

---

## 🎤 면접 빈출 Q&A

### Q1. 공급망 보안의 *4박자*는?
> (1) **Image scan (Trivy)** — 알려진 CVE 검출, CI 게이트. (2) **SBOM 생성 (Syft)** — image 컴포넌트 명세, 새 CVE 발견 시 영향 image 식별. (3) **Image 서명 (Cosign keyless)** — 출처·무결성 검증. (4) **Admission 검증 (Kyverno/Sigstore policy-controller)** — *cluster에서 서명 안 된 image 거부*. **4박자 모두 갖춰야 의미**.

### Q2. Cosign keyless가 *키 0개가 아닌* 이유는?
> *영구 키쌍이 없다*는 뜻. 서명 시 *Fulcio가 짧은 임시 인증서 (예: 10분) 발급* → 그걸로 서명 → 인증서 즉시 폐기. 서명·인증서·timestamp는 *Rekor 변경 불가 log에 박힘*. 키 *유출 불가능* (없으니), *각 서명마다 새 인증서*. "keyless"는 *영구 키 관리 부담 없음*의 의미.

### Q3. SBOM은 *왜 중요*한가?
> *image에 무엇이 들었는지 명세*. **새 CVE 발견 시** (log4shell 같은) *영향받는 image 즉시 검색*. 수동 스캔으론 너무 늦음. 컴플라이언스 audit에서 *공급망 가시성 증거*. 생성: **Syft / Trivy**. Format: **SPDX / CycloneDX**. 저장: *registry attestation* (cosign attest) 또는 *DependencyTrack*.

### Q4. Kyverno와 OPA Gatekeeper 비교?
> **Kyverno**: *YAML 기반 정책*, 학습 곡선 낮음, K8s native, *validate + mutate + generate* 모두. **Gatekeeper**: *Rego 정책*, 표현력 강 (임의 logic 가능), 학습 곡선 높음, mutate는 alpha. **시작은 Kyverno**, 복잡한 정책은 Gatekeeper. *Pod Security Admission*은 PSS 전용으로 K8s 내장.

### Q5. Admission policy로 *서명된 image만* 어떻게 강제?
> Kyverno의 `ClusterPolicy` + `verifyImages`. `attestors.keyless.subject + issuer` 조건으로 *어느 repo·workflow의 서명만 허용*. **Sigstore policy-controller**가 더 native — `ClusterImagePolicy`로 *cosign 검증 전용*. 더 simple, 서명만 필요하면 권장.

### Q6. SLSA framework란?
> *공급망 보안 수준 모델* (L1~L4). **L1**: 자동화 + provenance. **L2**: hosted builder + version-controlled. **L3**: 빌드 환경 격리 + provenance 인증 (서명). **L4**: 완전 재현성 (hermetic, parameterless) — 매우 어려움. **운영 목표는 L2~L3**. GitHub Actions + cosign + provenance attestation = 자연스럽게 *L3 접근*.

### Q7. *CI에서 서명했는데 cluster가 실수로 *unsigned image 배포*했다*. 어떻게 막나?
> **Admission policy로 *enforce***. (1) Kyverno/Sigstore policy-controller 설치, (2) `ClusterImagePolicy` 또는 `ClusterPolicy verifyImages`로 *trusted issuer + subject* 명시, (3) *validationFailureAction: Enforce*. **+ digest pin 강제** (tag 대신 `@sha256:...`) — `image must match *@sha256:*` 정책. 둘 다 있어야 *완전한 무결성*.

---

## 🔗 Cross-reference

- **Docker image 보안 (cap, USER, scanning)** → [`docker-basics/06`](../docker-basics/06-security.md)
- **Cosign + GitHub Actions OIDC** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md), [`cicd-gitops-basics/05`](../cicd-gitops-basics/05-build-and-registry.md)
- **K8s Admission Controller 메커니즘** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **Pod Security Admission (PSS)** → [01-pod-security.md](01-pod-security.md)
- **RBAC + IRSA** → [03-identity-and-rbac.md](03-identity-and-rbac.md)
- **Secret 관리** → [05-secrets-and-encryption.md](05-secrets-and-encryption.md)

---

## 📝 3줄 요약

1. *공급망 보안 4박자* — **Trivy 스캔 + Syft SBOM + Cosign keyless 서명 + Kyverno/Sigstore policy-controller admission**. *4 다 있어야 의미*.
2. *Cosign keyless* = OIDC + Fulcio 임시 인증서 + Rekor 영구 log. *영구 키 X, 유출 불가능*. **항상 digest pin** + admission *Enforce*.
3. *SLSA L2~L3가 운영 목표*. *Kyverno (YAML)로 시작*, 복잡 정책은 Gatekeeper (Rego). Sigstore policy-controller가 서명 전용에 native.

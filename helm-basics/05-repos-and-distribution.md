# 05 — Repos & Distribution: helm repo·OCI registry·dependency 관리

> **공식 근거:** [Helm Repository Guide](https://helm.sh/docs/topics/chart_repository/), [OCI-based registries](https://helm.sh/docs/topics/registries/)

---

## 🎯 한 문장

> **Helm chart도 *패키지* — 빌드·서명·배포·버저닝 흐름이 *컨테이너 이미지와 비슷*. 전통은 *HTTP-based chart repository* (index.yaml), 현대는 *OCI registry* (image와 같은 곳)에 push. 의존성은 *semver constraint*로 lock.**

---

## 1. Chart Repository — *전통 HTTP 방식*

### 구조
```
https://my-org.github.io/charts/
├── index.yaml            ← 모든 chart 메타데이터 (이름·버전·URL·해시)
├── myapp-0.1.0.tgz       ← 패키징된 chart
├── myapp-0.1.1.tgz
├── myapp-0.2.0.tgz
└── ...
```

### Repository 등록
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update              # index 새로고침
helm repo list
helm search repo nginx        # 검색
```

### 흔한 chart repo
- **Bitnami** (https://charts.bitnami.com/bitnami)
- **prometheus-community** (https://prometheus-community.github.io/helm-charts)
- **argo** (https://argoproj.github.io/argo-helm)
- **ingress-nginx** (https://kubernetes.github.io/ingress-nginx)
- **jetstack** (cert-manager) (https://charts.jetstack.io)

### 사내 chart repo 만들기
**옵션 1: GitHub Pages**
```bash
helm package mychart -d packaged/
helm repo index packaged/ --url https://my-org.github.io/charts/
# packaged/ 디렉터리를 GitHub Pages branch에 push
```

**옵션 2: ChartMuseum** (사내 서버)
- 자체 hosting. UI 제공. 권한 관리.

**옵션 3: Harbor / Nexus / Artifactory**
- 사내 registry가 *image + chart 동시 host*

---

## 2. OCI Registry — *컨테이너 이미지와 같은 곳*

### Helm 3.8+ 부터 *OCI 표준 지원 안정화*

Image와 chart를 *같은 registry*에 push:
```
ghcr.io/my-org/myapp:v1.0.0       ← container image
oci://ghcr.io/my-org/charts/myapp ← Helm chart
```

### Push / Pull
```bash
# 패키징
helm package mychart -d packaged/

# OCI registry에 push
helm push packaged/myapp-0.1.0.tgz oci://ghcr.io/my-org/charts

# pull
helm pull oci://ghcr.io/my-org/charts/myapp --version 0.1.0

# 직접 install
helm install myrelease oci://ghcr.io/my-org/charts/myapp --version 0.1.0
```

### 인증
```bash
helm registry login ghcr.io --username myuser --password $TOKEN
# 또는 docker login 한 것 그대로 활용
```

### OCI의 *장점*
- *image와 같은 인프라* (registry, 권한, 스캐닝, 서명)
- *별도 chart repo 호스팅 불필요*
- *Cosign 서명* image와 같은 방식
- *Trivy 스캔* (chart 안 image까지)

→ 새 chart는 *OCI 우선* 권장. 전통 HTTP repo는 *legacy 유지보수* 외엔 deprecated 추세.

---

## 3. Chart Versioning — semver

```yaml
# Chart.yaml
version: 1.2.3
```

`MAJOR.MINOR.PATCH` (Semantic Versioning):
- **MAJOR** — *호환 깨지는* 변경
- **MINOR** — *호환되는 기능 추가*
- **PATCH** — *호환되는 bug fix*

### Chart 변경 시
- *항상 version ↑*
- *같은 버전 재push* → registry가 거부 (또는 *덮어쓰지 말 것* 정책)
- 운영에선 *immutable versions* — 같은 버전이면 *내용 같음을 보장*

### Pre-release
- `1.0.0-beta.1`, `1.0.0-rc.1` 등
- *helm search* 기본 제외 (`--devel` 옵션 시 포함)

---

## 4. Dependency 관리

### Chart.yaml 의 dependencies
```yaml
dependencies:
- name: postgresql
  version: "12.x.x"               # ← semver constraint
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
- name: redis
  version: "~17.0.0"              # ← patch 까지만
  repository: oci://registry-1.docker.io/bitnamicharts
- name: common
  version: 1.0.0
  repository: file://../common-chart
- name: external-secrets
  version: ">=0.5.0,<1.0.0"
  repository: https://charts.external-secrets.io
```

### Semver Constraint 문법
| 패턴 | 의미 |
|---|---|
| `1.2.3` | 정확히 1.2.3 |
| `>=1.2.0` | 1.2.0 이상 |
| `>=1.2.0,<2.0.0` | 범위 |
| `~1.2.3` | `>=1.2.3, <1.3.0` (patch 까지) |
| `^1.2.3` | `>=1.2.3, <2.0.0` (minor 까지) |
| `1.x.x` | `>=1.0.0, <2.0.0` |
| `*` | 어떤 버전이든 (위험) |

### Lock 파일
```bash
helm dependency update
# 결과:
# - charts/ 에 .tgz 다운로드
# - Chart.lock 파일 생성

cat Chart.lock
# digest: sha256:abc...
# generated: "2026-05-26T..."
# dependencies:
# - name: postgresql
#   version: 12.5.6
#   ...
```

- *lock 파일이 정확한 resolved 버전* 기록
- *재현성 보장* — `helm dependency build`로 lock 기반 fetch

### Best practice
- *constraint에 `^` 또는 `~` 사용* — patch/minor 자동 업그레이드
- *lock 파일을 git commit* — 재현성
- *주기적 update* + *changelog 검토*

---

## 5. CI/CD에서 chart 관리

### Lint
```bash
helm lint ./mychart
helm lint ./mychart -f values-prod.yaml      # 특정 values로 검증
```

### Template (시뮬레이션)
```bash
helm template ./mychart -f values-prod.yaml > /tmp/manifests.yaml
# 결과 review, kubectl apply --dry-run 가능
```

### Test 자원 (`helm test`)
- `templates/tests/` 디렉터리의 *test 자원 실행*
- 자세히 [03-release-lifecycle.md](03-release-lifecycle.md) Section 7

### CI 흐름 예
```yaml
# .github/workflows/chart-ci.yml
- run: helm lint ./mychart
- run: helm template ./mychart > /tmp/manifests.yaml
- run: kubectl --dry-run=server apply -f /tmp/manifests.yaml

# release on tag
- run: |
    helm package ./mychart -d /tmp/
    helm push /tmp/myapp-*.tgz oci://ghcr.io/${{ github.repository_owner }}/charts
```

### 사내 표준 lint plugin
- **chart-testing (ct)** — Helm community 권장 CI tool
  - `ct lint` — 변경된 chart 자동 lint
  - `ct install` — 변경된 chart를 *kind cluster에 install* 후 test
- Github Action: `helm/chart-testing-action`

---

## 6. Chart 서명 / Provenance

### `helm package --sign` (GPG)
```bash
helm package mychart --sign --key 'mykey' --keyring ~/.gnupg/secring.gpg
# 결과:
# - myapp-0.1.0.tgz
# - myapp-0.1.0.tgz.prov     ← provenance 파일 (GPG 서명)
```

### 검증
```bash
helm verify myapp-0.1.0.tgz
helm install --verify myrelease ./myapp-0.1.0.tgz
```

### Cosign으로 OCI chart 서명 (현대)
```bash
cosign sign --yes oci://ghcr.io/my-org/charts/myapp:0.1.0
cosign verify oci://ghcr.io/my-org/charts/myapp:0.1.0 \
  --certificate-identity=https://github.com/my-org/my-chart-repo/.github/workflows/release.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

→ *OCI 통합의 장점* — image와 *같은 cosign 사용*. 별도 GPG 키 관리 불필요.

---

## 7. ArgoCD에서 chart 사용

### Git 안 chart
```yaml
# Application
spec:
  source:
    repoURL: https://github.com/my-org/myapp-manifests
    path: charts/myapp
    targetRevision: main
    helm:
      valueFiles:
      - values-prod.yaml
```

### Remote chart (HTTP repo)
```yaml
spec:
  source:
    chart: postgresql
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: 12.5.6
    helm:
      valueFiles:
      - $values/values-prod.yaml       # multi-source 패턴
```

### OCI registry chart
```yaml
spec:
  source:
    chart: myapp
    repoURL: ghcr.io/my-org/charts
    targetRevision: 0.1.0
    helm:
      values: |
        replicaCount: 3
```

→ 자세히 [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md).

---

## 8. *Subchart override* — alias·import-values

### alias — *같은 chart를 여러 번 의존*
```yaml
dependencies:
- name: postgresql
  version: 12.x.x
  repository: https://charts.bitnami.com/bitnami
  alias: primary-db
- name: postgresql
  version: 12.x.x
  repository: https://charts.bitnami.com/bitnami
  alias: replica-db
```

→ 두 개 다른 DB 인스턴스. values에선 `primary-db.auth.password`, `replica-db.auth.password` 분리.

### import-values — *subchart 값을 parent로 export*
```yaml
dependencies:
- name: subchart
  version: 1.0.0
  repository: ...
  import-values:
  - child: data.db.host
    parent: dbHost
```

→ subchart의 `.Values.data.db.host` 를 parent의 `.Values.dbHost` 로 가져옴.

---

## 9. *Chart 운영 패턴*

### 패턴: *app chart vs umbrella chart*
- **app chart**: 단일 마이크로서비스 (예: `myapp`)
- **umbrella chart**: 여러 chart 묶음 (예: `production-stack`)
- 운영 흐름:
  - 개발팀이 *app chart* 유지
  - 플랫폼팀이 *umbrella chart*로 *환경별 묶음*

### 패턴: *chart registry 통합*
- 사내 모든 chart를 *한 OCI registry*에 (예: GHCR, ECR)
- *image와 같은 권한*·*같은 스캔*·*같은 서명*
- chart 사용자: `oci://ghcr.io/my-org/charts/...` 일관

### 패턴: *chart 자동 dependency 업데이트*
- Renovate / Dependabot이 *Chart.yaml의 dependency version* 자동 PR
- *patch는 자동 머지*, *major는 사람 검토*

---

## 10. 자주 헷갈리는 것

### 10-1. *같은 version 재push*
- 기본적으로 *허용*되지만 *재현성 깨짐*
- registry 정책으로 *immutable tag* 강제 (ECR `IMMUTABLE`, Harbor 정책)
- *항상 새 version*

### 10-2. *Chart.lock 안 commit*
- `Chart.lock` git ignore 하면 *재현성 깨짐*
- *반드시 commit*

### 10-3. *`helm dependency update` vs `helm dependency build`*
- `update` — *constraint 다시 resolve* + lock 갱신
- `build` — *기존 lock 기반 fetch* (constraint 안 봄)
- CI에선 `build` (lock 신뢰), 의도적 업데이트 시 `update`

### 10-4. *OCI chart vs HTTP chart 차이*
- 사용자 입장 *거의 같음* — `repoURL` 형식만 다름
- 단, *HTTP chart는 `helm repo add` 필요*, *OCI는 `helm registry login`*만

### 10-5. *대규모 chart의 *index.yaml*이 *너무 큼**
- Bitnami repo는 *수백 chart × 수백 version* → index.yaml 거대
- `helm repo update`가 *느림*
- 해결: *OCI로 전환*, 또는 *필요한 chart만 별도 사내 repo로 mirror*

---

## 🎤 면접 빈출 Q&A

### Q1. Helm chart repository는 어떻게 구성?
> 전통: *HTTP-based* — `index.yaml` (메타데이터) + `*.tgz` (패키징된 chart). GitHub Pages, ChartMuseum, Harbor/Nexus/Artifactory. 등록은 `helm repo add`. 현대: **OCI registry** (Helm 3.8+) — *컨테이너 이미지와 같은 곳* (GHCR, ECR 등). `helm push oci://...` + `helm install oci://...`. *별도 chart 호스팅 불필요*.

### Q2. Chart dependency 관리는?
> Chart.yaml `dependencies` 에 *semver constraint* (`^1.2.3`, `~1.2.0`, `>=1.0.0,<2.0.0`). `helm dependency update` → `charts/` 에 `.tgz` 다운로드 + `Chart.lock` 생성. **Chart.lock을 git commit** — 재현성. CI에선 `helm dependency build` (lock 기반 fetch). 자동화는 Renovate/Dependabot.

### Q3. OCI registry로 chart 배포가 *전통 HTTP 대비* 장점?
> (1) *image와 같은 인프라* — 별도 chart 호스팅 X. (2) *권한·스캔·서명 통합* — image와 같은 cosign. (3) *deprecate되지 않을* 표준. (4) Bitnami·prometheus-community 등 대형 chart 제공자도 *OCI 전환 중*. → *새 chart는 OCI 우선*.

### Q4. Subchart에 *같은 chart 두 번 dependency*?
> `alias` 사용. 예:
> ```yaml
> dependencies:
> - name: postgresql
>   alias: primary-db
> - name: postgresql
>   alias: replica-db
> ```
> values에선 `primary-db.auth.password`, `replica-db.auth.password` 분리. 같은 chart 두 인스턴스를 *다른 설정으로*.

### Q5. Cosign으로 chart 서명?
> OCI chart라면 *image와 동일한 cosign*. `cosign sign --yes oci://...`. *keyless* (GitHub Actions OIDC) — 별도 GPG 키 관리 X. 검증: `cosign verify ... --certificate-identity ...`. 전통 GPG `helm package --sign` 도 있으나 키 관리 부담 — OCI + cosign이 현대 표준.

### Q6. CI에서 chart 변경을 어떻게 검증?
> (1) `helm lint` — chart 구조 검증. (2) `helm template --debug` — render 결과 확인. (3) `kubectl --dry-run=server apply` — K8s 서버 측 검증. (4) **chart-testing (ct)** — 변경된 chart 자동 lint + *kind cluster에 install + test*. (5) values.schema.json 검증. Github Action `helm/chart-testing-action` 표준.

### Q7. ArgoCD에서 chart 사용 *3가지 방식*?
> (1) **Git 안 chart** — `repoURL`은 git, `path`로 chart 경로. (2) **HTTP repo chart** — `repoURL`은 chart repo URL, `chart` 필드. (3) **OCI registry chart** — `repoURL`은 OCI URL. 모두 `helm.valueFiles` / `helm.values` / `helm.parameters` 로 values override.

---

## 🔗 Cross-reference

- **chart 구조·dependency 자체** → [01-chart-structure.md](01-chart-structure.md)
- **values override 자체** → [04-values-design.md](04-values-design.md)
- **OCI image / cosign / SBOM** → [`docker-basics/01`](../docker-basics/01-image-and-layers.md), [`cicd-gitops-basics/05`](../cicd-gitops-basics/05-build-and-registry.md)
- **GitHub Actions에서 helm package + push** → [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **ArgoCD의 chart 사용** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. *Chart 배포 = 전통 HTTP repo (index.yaml + tgz) vs 현대 OCI registry (image와 같은 곳)*. 새 chart는 *OCI 우선*.
2. *Dependency*는 Chart.yaml + *semver constraint* (`^`, `~`). **Chart.lock commit** + 자동 업데이트 (Renovate/Dependabot).
3. CI는 *lint + template + chart-testing (ct)*. 서명은 *OCI라면 cosign keyless*가 표준 (GPG 키 관리 X).

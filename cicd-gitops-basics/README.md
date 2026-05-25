# CI/CD & GitOps — 동작 원리 중심

> *"GitOps는 무엇이고 Pull-based가 왜 안전한가요?"*
> *"ArgoCD의 sync는 어떻게 작동하나요?"*
> *"GitHub Actions OIDC로 AWS access하는 원리는?"*

---

## 🎯 한 문장 큰 그림

> **CI = *코드 → 검증된 산출물*. CD = *산출물 → 운영 환경*. GitOps = *Git을 single source of truth로 두고, 운영 환경이 Git과 항상 일치하도록 controller가 reconcile* 하는 패턴.**

→ K8s의 *controller reconciliation* 사상을 *배포에도 적용*한 게 GitOps. ArgoCD가 *K8s API를 통해 git의 desired state를 cluster actual state에 강제*.

---

## 🧩 큰 그림 — 한 페이지 요약

```
   ┌─────────────────────────────────────────────────────────────┐
   │ Developer                                                    │
   │   git push  →  Source Repo (코드)                            │
   └────────────────────┬─────────────────────────────────────────┘
                        ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ CI (GitHub Actions / Jenkins / GitLab CI)                    │
   │   1. lint / test                                             │
   │   2. SAST·SCA·secret scan                                    │
   │   3. build image (BuildKit / Kaniko)                         │
   │   4. push to registry (digest로 박제)                         │
   │   5. (선택) cosign sign + SBOM                                │
   └────────────────────┬─────────────────────────────────────────┘
                        ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ CD — *Push-based* OR *Pull-based(GitOps)*                    │
   │                                                              │
   │ Push-based (옛 방식):                                          │
   │   CI가 직접 `kubectl apply` / `helm upgrade`                 │
   │   - CI가 cluster credential 가짐 (위험)                       │
   │                                                              │
   │ Pull-based (GitOps, 현대):                                    │
   │   1. CI가 manifest repo의 image tag만 commit                  │
   │   2. ArgoCD가 manifest repo *watch*                          │
   │   3. ArgoCD가 cluster에 *reconcile* (current vs git)         │
   │   - cluster credential은 ArgoCD만 가짐                        │
   └────────────────────┬─────────────────────────────────────────┘
                        ▼
                ┌─────────────────┐
                │  Kubernetes      │
                │  (actual state)  │
                └─────────────────┘
```

---

## 🌪 자주 헷갈리는 6가지

### 1. *CI와 CD는 다른 것*
- **CI** (Continuous Integration) — *코드 → 검증된 산출물 (image, package)*
- **CD** (Continuous Delivery / Deployment) — *산출물 → 운영*
- 한 도구가 둘 다 할 수 있어도 *책임 분리*가 좋음 (보안, 권한, 롤백)

### 2. *GitOps != ArgoCD*
- GitOps = *원칙*: Git을 진실 공급원, 자동 reconcile, declarative
- ArgoCD / Flux = *도구*
- 옛 Jenkins로도 GitOps 비슷하게 가능 (단, Jenkins 자체가 cluster 변경 — push-based)

### 3. *Push vs Pull의 진짜 차이는 *권한 방향**
- Push: *CI가 cluster credential 갖고 변경* → *CI 침해 = cluster 침해*
- Pull: *cluster 안 controller가 git을 fetch* → *CI는 manifest repo write 권한만*, *cluster credential 없음*
- → 보안 강함 + 외부 노출 표면 줄어듦

### 4. *Image tag로 배포하지 마라*
- `image: myapp:latest` 또는 `myapp:v1.2.3` → *tag는 mutable*. 누가 같은 태그로 다른 이미지 push 가능.
- *digest pin* (`myapp@sha256:abc...`) — 변경 불가
- ArgoCD Image Updater는 *새 digest 감지 시 manifest repo에 commit*하는 패턴

### 5. *GitOps의 sync가 *실패하면* 어떻게 되는가*
- ArgoCD는 *그냥 status에 OutOfSync / Degraded 표시*. cluster를 *깨지지 않음*.
- *self-heal* 기본 ON이면 *수동 변경 자동 복원*. OFF면 *git이 진실*이라는 원칙만 유지하고 *수동 변경 허용*.

### 6. *secret을 git에 안 박는다*
- 평문은 당연히 X
- *Sealed Secrets* — 공개 키로 암호화해서 git에 (controller가 cluster에서 복호화)
- *External Secrets Operator* — git에 *Vault path 참조*만, 실제 값은 Vault에서 fetch
- *SOPS + age/KMS* — git에 암호화된 YAML, controller가 복호화

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-ci-fundamentals.md](01-ci-fundamentals.md) | CI 파이프라인 설계 원칙, build/test/scan/artifact, 캐시·병렬화 |
| [02-github-actions.md](02-github-actions.md) (예정) | workflow/job/step, runner, OIDC, secrets, reusable workflows |
| [03-argocd.md](03-argocd.md) (예정) | Application·AppProject·sync wave, self-heal, ApplicationSet, Image Updater |
| [04-deployment-strategies.md](04-deployment-strategies.md) (예정) | RollingUpdate, Blue/Green, Canary, Argo Rollouts |
| [05-build-and-registry.md](05-build-and-registry.md) (예정) | BuildKit, Kaniko, registry workflow, 공급망 보안 (cosign·SBOM) |

---

## 🔗 Cross-reference

- **선언적 reconciliation 패턴** = [`kubernetes-basics/06-api-and-controllers.md`](../kubernetes-basics/06-api-and-controllers.md). ArgoCD는 *K8s controller pattern을 git에 적용*.
- **OIDC trust boundary** = `aws-basics/` (IRSA의 OIDC와 *같은 원리, 다른 IdP*). GitHub Actions OIDC도 같은 패턴.
- **Image build** = [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md) (BuildKit, multi-stage, .dockerignore).
- **Image scan + cosign + SBOM** = [`docker-basics/06`](../docker-basics/06-security.md).
- **Helm chart deploy** = `helm-basics/` (예정).
- **K8s 자원 생성 시 admission 정책 (서명 검증 등)** = [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md).

---

## 🎤 면접 빈출 (전체 폴더 횡단)

| 질문 | 답이 있는 파일 |
|---|---|
| CI와 CD의 차이는? | [01](01-ci-fundamentals.md) |
| GitOps는 무엇이고 push와 pull의 차이는? | [03](03-argocd.md) (예정) |
| GitHub Actions에서 AWS access 어떻게? OIDC 원리? | [02](02-github-actions.md) (예정) |
| ArgoCD의 sync 동작과 self-heal은? | [03](03-argocd.md) (예정) |
| Canary 배포를 K8s에서 어떻게 구현? | [04](04-deployment-strategies.md) (예정) |
| Docker daemon 없이 컨테이너 안에서 image build? | [05](05-build-and-registry.md) (예정) |
| 공급망 보안 — cosign keyless·SBOM | [05](05-build-and-registry.md) (예정) |

---

## 📐 학습 순서 권장

1. **01 ci-fundamentals** — *CI*가 *왜·무엇을* 검증하는가
2. **02 github-actions** — *대표 도구의 실제 동작* + OIDC 깊이
3. **03 argocd** — *CD의 현대적 표준* (GitOps)
4. **04 deployment-strategies** — *배포 전략 깊이*
5. **05 build-and-registry** — *image 생성·전송·검증의 보안*

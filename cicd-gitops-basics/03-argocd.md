# 03 — Argo CD: GitOps의 표준

> **공식 근거:** [Argo CD docs](https://argo-cd.readthedocs.io/), [OpenGitOps Principles](https://opengitops.dev/)
>
> **선행:** [01-ci-fundamentals.md](01-ci-fundamentals.md), [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md) (controller pattern)

---

## 🎯 한 문장

> **Argo CD = *Git을 single source of truth로 두고, K8s cluster의 actual state를 Git의 desired state에 reconcile하는 K8s controller*. *K8s controller pattern을 git에 확장*한 도구.**

---

## 1. GitOps 4원칙 (OpenGitOps)

1. **Declarative** — 시스템 상태를 *선언적으로 명세* (YAML 등)
2. **Versioned and immutable** — *git에 저장*, 변경 이력 추적
3. **Pulled automatically** — *cluster의 agent가 git에서 fetch* (push 아님)
4. **Continuously reconciled** — agent가 *영원히 desired vs actual 비교 + 강제*

→ ArgoCD가 *이 4원칙을 K8s에 구현*. *K8s controller pattern + git*.

---

## 2. Push vs Pull — *권한 방향이 결정*

### Push-based (옛)
```
CI → kubectl apply → cluster
     (CI가 cluster credential 가짐)
```
- *CI 침해 = cluster 침해*
- credential 회전·관리 부담
- 외부 (CI 서버) 가 *cluster API에 접근 권한*

### Pull-based (GitOps, 현대)
```
CI → git push manifest →  git repo
                              ▲
                              │ (watch + pull)
                              │
                      Argo CD (cluster 안)
                              │
                              ▼
                          cluster
```
- CI는 *manifest repo write 권한*만
- *cluster credential은 ArgoCD만* (cluster 안)
- *외부 노출 표면 줄어듦*
- 침해 격리: CI 침해 시 *git는 오염*되지만 *cluster API 직접 변경 X*

→ **이게 GitOps의 *진짜* 보안 가치**.

---

## 3. Argo CD의 핵심 컴포넌트

```
┌────────────────────────────────────────────┐
│ Argo CD (cluster 안 namespace: argocd)     │
│                                            │
│  ├─ API server   (UI + CLI + REST)          │
│  ├─ Repository server  (git clone + render) │
│  ├─ Application controller  (sync engine)   │
│  └─ Redis (캐시)                            │
└────────────────────────────────────────────┘
            │
            ├─ git repo 1 (manifest)
            ├─ git repo 2 (Helm chart values)
            └─ git repo 3 (...)
                          │
                          ▼
            ┌──────────────────────────┐
            │ Target cluster (자기 자신, │
            │  또는 다른 cluster들)      │
            └──────────────────────────┘
```

### Repository server의 역할
- git repo *clone + render*
- *plain YAML*, *Helm chart*, *Kustomize*, *Jsonnet* 모두 지원
- 렌더링된 manifest를 controller에 전달

### Application controller의 역할
- *Application 자원 watch*
- 주기적으로 *git의 desired state 가져옴* → *cluster actual state* 비교
- diff 있으면 *sync* (또는 OutOfSync 표시만)

---

## 4. Application — *배포 단위*

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/manifests
    targetRevision: main
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 의미
- *git의 `manifests/apps/myapp` 디렉터리* 의 *모든 YAML* 을
- *destination cluster의 `production` namespace* 에 적용
- *변경 시 자동 sync*, *수동 변경은 self-heal로 복원*, *git에서 사라진 자원은 자동 prune*

### Sync 상태
- **Synced** — git == cluster
- **OutOfSync** — git에 변화 있으나 아직 cluster에 안 적용 (auto-sync OFF이거나 sync 진행 중)
- **Healthy** — 자원이 정상 동작 (Deployment의 모든 Pod Ready 등)
- **Degraded** — 자원이 비정상 (Pod CrashLoop 등)

---

## 5. SyncPolicy 옵션

### `automated`
- `prune: true` — *git에서 삭제된 자원* 자동 cluster에서 삭제. *기본 false* (안전).
- `selfHeal: true` — 누군가 *수동으로 cluster 자원 변경* → *git 상태로 복원*. *기본 false*.

### `syncOptions`
- `CreateNamespace=true` — destination namespace 없으면 자동 생성
- `Validate=false` — kubectl validation skip
- `PrunePropagationPolicy=foreground` — prune 시 cascade 정책
- `ServerSideApply=true` — kubectl의 server-side apply 사용 (충돌 해결 우수)

### Sync 단계
1. git fetch + render → desired manifest
2. cluster API에서 actual 상태 가져옴
3. diff 계산
4. 차이 부분만 apply (또는 ServerSideApply)

---

## 6. Sync Waves & Hooks — *순서 제어*

### Wave (자원 간 순서)
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```
- 자원별 wave 번호 → *오름차순으로 순차 sync*
- 예: *Namespace (wave -1) → ServiceAccount + Role (wave 0) → Deployment (wave 1) → Ingress (wave 2)*
- 의존성 있는 자원에 유용

### Hooks (sync 전후 작업)
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

Hook 종류:
| Hook | 시점 |
|---|---|
| `PreSync` | sync 시작 전 |
| `Sync` | sync 중 (이 hook 끝나야 다음 wave) |
| `PostSync` | sync 완료 후 |
| `SyncFail` | sync 실패 시 |

### 흔한 활용
- *PreSync*: DB 마이그레이션 (Job)
- *PostSync*: smoke test (Job)
- *SyncFail*: 알림 전송

---

## 7. ApplicationSet — *여러 Application 자동 생성*

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/my-org/services
      directories:
      - path: services/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      source:
        repoURL: https://github.com/my-org/services
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

→ git repo의 *각 서비스 디렉터리마다 Application 자동 생성*.

### Generator 종류
- **list** — 명시적 목록
- **git** — git의 디렉터리 또는 파일 기반
- **cluster** — 등록된 모든 cluster에 같은 app 배포 (multi-cluster)
- **matrix** — 두 generator 조합
- **scm** — GitHub/GitLab org의 모든 repo

→ *수십~수백 개 app·env·cluster*를 *선언적으로 한 번에* 관리.

---

## 8. AppProject — *RBAC + 정책*

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  description: Backend team
  sourceRepos:
    - https://github.com/my-org/backend-*
  destinations:
    - namespace: backend-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
  roles:
    - name: developer
      policies:
        - p, proj:team-backend:developer, applications, sync, team-backend/*, allow
```

→ *어느 repo / 어느 cluster / 어느 namespace / 어느 자원 종류* 허용하는지 *프로젝트별 격리*.

---

## 9. Image Updater — *이미지 자동 갱신*

문제: CI가 새 image push → manifest repo의 image tag는 *수동*으로 갱신해야?

해결: **Argo CD Image Updater**가 *registry watch* → 새 image 발견 → *manifest repo에 자동 commit + ArgoCD sync*.

```yaml
# Application에 annotation
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=ghcr.io/me/myapp
    argocd-image-updater.argoproj.io/update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git
```

### 흐름
```
1. CI build & push image → ghcr.io/me/myapp:v1.2.3
2. Image Updater (5분마다) → registry watch → 새 tag 발견
3. Image Updater → manifest repo에 commit ("update myapp to v1.2.3")
4. ArgoCD → manifest repo 변화 감지 → sync → cluster 업데이트
```

→ *완전 자동화*. 개발자는 *코드만 push*. 배포는 자동.

### 주의
- *Image Updater가 manifest repo write 권한* 가짐 — SSH key 또는 PAT
- *update strategy* — `semver`, `latest`, `digest`, `name` 등
- *digest 권장* — tag mutable 위험 회피

---

## 10. Argo CD vs Flux — 간단 비교

| | Argo CD | Flux |
|---|---|---|
| 첫 인상 | *UI 강력* | *CLI·CRD 중심* |
| 학습 곡선 | 낮음 (UI) | 중간 |
| Application 모델 | `Application` CRD | `Kustomization`, `HelmRelease` CRD |
| Image automation | Image Updater (별도) | 내장 |
| Multi-cluster | ApplicationSet · cluster generator | 직접 controller deploy |
| Argo 생태계 | Rollouts·Workflows·Events | 단독 |

→ 둘 다 GitOps 표준. *팀 취향·UI 중요도*에 따라.

---

## 11. 자주 헷갈리는 것

### 11-1. *ArgoCD가 git 변화를 *얼마나 빨리* 감지*
- 기본은 *3분마다 git poll*
- *webhook 설정*하면 *git push 즉시* 감지

### 11-2. *OutOfSync인데 cluster는 정상* — 무엇이 잘못?
- git에 *최신 변경 안 적용*된 상태
- *auto-sync OFF*면 *수동 sync 대기*
- *sync 했는데도 OutOfSync*면 — *immutable 필드 변경* (예: Service의 selector) 또는 *admission webhook가 변형* 등

### 11-3. *self-heal이 ON인데도 수동 변경이 유지됨*
- self-heal은 *반복적*. 변경 직후 한순간은 유지 가능하나 *다음 reconcile에서 복원*
- *3분 안에* 복원 (또는 webhook 즉시)

### 11-4. *prune이 *원치 않게 자원 삭제**
- git에서 *실수로 manifest 삭제* → cluster 자원 삭제됨
- 방어: *important 자원에 `argocd.argoproj.io/sync-options: Prune=false` annotation*
- 또는 *project에서 cluster 자원 종류 화이트리스트 제한*

### 11-5. *Application이 너무 큰 단위*
- 한 Application에 *수십 개 service* → diff 어려움, sync 실패 시 *전체 영향*
- *서비스별 Application 분리* + *ApplicationSet으로 일괄 관리*

### 11-6. *sync에서 *immutable 필드 충돌**
- 일부 K8s 자원의 필드는 *변경 불가* (예: Service의 `clusterIP`, Job의 `selector`)
- ArgoCD가 변경 시도 → fail
- 해결: *Replace=true* sync option (자원 삭제 후 재생성) — 단점: *downtime + 데이터 손실 위험*
- 또는 *처음부터 immutable 필드 안 변경*

---

## 🎤 면접 빈출 Q&A

### Q1. GitOps란 무엇이고 push와 pull의 진짜 차이는?
> GitOps = *Git을 single source of truth로 두고 cluster를 git과 일치시키는 패턴* (declarative, versioned, pulled, continuously reconciled). **Push** = CI가 cluster credential 갖고 직접 변경. *CI 침해 = cluster 침해*. **Pull** = cluster 안 controller (ArgoCD)가 git fetch + reconcile. *CI는 manifest repo write 권한만*, cluster credential은 ArgoCD만. *외부 노출 표면·침해 격리* 모두 강함.

### Q2. Argo CD의 sync는 어떻게 작동?
> Application 자원에서 *git repoURL + path + targetRevision* 지정. Application controller가 *3분마다 git poll* (또는 webhook 즉시) → manifest 렌더 → cluster actual 상태 비교 → diff 있으면 apply (또는 OutOfSync 표시). *auto-sync*면 자동 적용, *self-heal*이면 수동 변경 복원, *prune*이면 git에서 삭제된 자원 자동 삭제.

### Q3. Sync wave와 hook의 차이?
> **Wave** = *자원 간 순서*. annotation 번호 오름차순으로 sync. 예: Namespace → SA → Deployment → Ingress. **Hook** = *sync 단계의 특정 시점에 실행되는 작업*. PreSync (DB 마이그레이션 Job), PostSync (smoke test), SyncFail (알림). 둘 다 *복잡한 배포 워크플로*를 선언적으로.

### Q4. ApplicationSet의 가치는?
> *여러 Application 자동 생성*. generator (list/git/cluster/matrix/scm)로 *동적 목록*에서 template으로 Application 생성. → *수십~수백 개 app·env·cluster*를 *선언적*으로 일괄 관리. 예: git의 각 서비스 디렉터리마다 자동 Application, 또는 등록된 모든 cluster에 같은 app 배포 (multi-cluster).

### Q5. Image Updater의 흐름은?
> *Registry watch* (5분마다) → 새 image tag 발견 → *manifest repo에 commit* (image tag 갱신) → ArgoCD가 manifest 변화 감지 → sync → cluster 업데이트. → *완전 자동 CD*. 개발자는 코드만 push, image 빌드는 CI, 배포는 Image Updater + ArgoCD. *update strategy* (semver/digest/latest) 명시.

### Q6. *OutOfSync인데 sync 했는데도 안 됨* — 디버깅?
> 흔한 원인: (1) *immutable 필드 변경* (Service `clusterIP`, Job `selector`) — `Replace=true` 또는 *변경 안 함*. (2) *Mutating admission webhook이 자원 변형* — git에 *변형 후 형태* 반영. (3) *RBAC 부족* — ArgoCD SA가 *해당 자원 변경 권한 없음*. (4) *Helm chart의 dynamic 값* — render 결과가 계속 달라짐 (random suffix 등).

### Q7. Argo CD를 *직접 cluster에 띄울 때 보안* 고려는?
> (1) *cluster-admin 권한 신중* — ArgoCD가 *모든 namespace·자원에 접근* 가능. *AppProject로 격리*. (2) *git repo credential 보안* — SSH key·PAT를 *Sealed Secret 또는 External Secrets*. (3) *Web UI 노출 보안* — Ingress + Okta SSO + RBAC. (4) *Image Updater write-back* 권한 — *제한된 PAT*. (5) *Argo CD 자체도 GitOps로 관리* (App of Apps 패턴).

---

## 🔗 Cross-reference

- **K8s controller pattern (ArgoCD의 베이스)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **CI 단계 (manifest repo에 commit)** → [01-ci-fundamentals.md](01-ci-fundamentals.md), [02-github-actions.md](02-github-actions.md)
- **배포 전략 (Canary 등)** → [04-deployment-strategies.md](04-deployment-strategies.md)
- **Helm chart로 Application source** → `helm-basics/` (예정)
- **Sealed Secrets / External Secrets — secret을 git에** → `k8s-security-basics/` (예정)
- **OpenGitOps 원칙** = K8s declarative reconciliation의 *외부 git 확장*

---

## 📝 3줄 요약

1. *Argo CD = K8s controller pattern을 git에 확장한 도구*. Application 자원이 *git의 desired state를 cluster actual state로 reconcile*.
2. **Pull-based의 진짜 가치는 *권한 격리*** — cluster credential이 *ArgoCD만*, CI는 *manifest repo write*만. CI 침해 ≠ cluster 침해.
3. *ApplicationSet으로 다수 app 일괄 관리*, *Sync Wave/Hook으로 순서 제어*, *Image Updater로 image 자동 배포 루프 완성*. **OutOfSync 디버깅**은 *immutable 필드·admission webhook·RBAC* 의심.

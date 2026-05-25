# Helm — 동작 원리 중심

> *"Helm chart는 무엇이고 ArgoCD와 어떻게 같이 쓰나요?"*
> *"values.yaml override 우선순위는?"*
> *"helm upgrade가 어떻게 rollback 가능한가요?"*

---

## 🎯 한 문장 큰 그림

> **Helm = *K8s manifest를 templating + parameterizing + release 단위로 lifecycle 관리*하는 패키지 매니저. chart는 *재사용 가능한 K8s app template*, release는 *cluster에 install된 chart 인스턴스*.**

→ K8s를 *yaml 더미*가 아니라 *재사용 가능한 패키지*로 다루게 해주는 도구.

---

## 🧩 한 페이지 요약

```
   ┌─────────────────────────────────────────────────────────┐
   │              Chart (재사용 가능 패키지)                   │
   │  Chart.yaml        ← 메타데이터 (이름·버전·의존성)        │
   │  values.yaml       ← 기본값 (override 가능)              │
   │  templates/        ← Go template로 작성된 K8s manifest   │
   │  charts/           ← sub-chart 의존성                    │
   │  README.md, NOTES.txt, _helpers.tpl                     │
   └────────────────────────┬────────────────────────────────┘
                            │  helm install/upgrade
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │      Release (cluster에 install된 chart 인스턴스)         │
   │  - chart + values를 *렌더링한 결과*                       │
   │  - 결과 manifest를 K8s API에 *apply*                      │
   │  - release 정보 → cluster의 Secret으로 저장              │
   │  - 같은 chart 여러 번 install 가능 (release name으로 구분) │
   └─────────────────────────────────────────────────────────┘
```

---

## 🌪 자주 헷갈리는 6가지

### 1. Helm은 *render + apply* 도구. *런타임 관여 없음*.
- `helm install` → chart + values 렌더링 → 결과를 `kubectl apply` 와 비슷하게 적용
- *그 후엔 관여 X* — 그냥 *static manifest*가 cluster에 박힘
- → 런타임 reconcile은 *K8s controller가 함*. Helm은 *deploy 시점만*.

### 2. *Operator와의 차이*
- Helm = *templating + 한 번 적용*
- Operator = *영원히 watch + reconcile* (CRD + controller)
- *한 번 만들고 끝*이면 Helm. *런타임 변화 대응* 필요하면 Operator. 둘 결합도 흔함 (Helm으로 Operator 설치).

→ [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md) Operator 섹션과 같은 구분.

### 3. *Helm 2 vs Helm 3*
- Helm 2: *Tiller* (server-side component) 필요. cluster-admin 권한. 보안 우려.
- Helm 3: *Tiller 제거*. client-side rendering. release 정보는 *cluster Secret으로* 저장.
- *현재 표준은 Helm 3*. Helm 2는 EOL.

### 4. *Release 정보는 *cluster의 Secret*에 저장*
- `kubectl get secret -n {namespace} | grep sh.helm.release`
- 압축된 release manifest + values + history
- *etcd 안에 있음* — etcd 백업이 release 백업도 겸함

### 5. *values 우선순위*가 헷갈림
- 낮음 → 높음: *chart의 values.yaml* → *parent chart values* → `-f values-prod.yaml` → `--set key=value`
- 명시적으로 *override*. 자세히 [04 참조](04-values-design.md).

### 6. *ArgoCD와 같이 쓸 때*
- ArgoCD가 *helm template을 내부적으로 호출*해서 manifest 렌더링
- ArgoCD가 *helm install/upgrade를 안 함* (release 자원 없음) — 그냥 *rendering만*
- → `helm list` 에 안 보일 수 있음. *ArgoCD가 관리*하는 거지 *Helm CLI가 아님*.

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-chart-structure.md](01-chart-structure.md) | Chart.yaml, values.yaml, templates/, _helpers, charts/ subchart |
| [02-templating.md](02-templating.md) | Go template, sprig, .Values/.Release/.Chart, if/range/with, named template, checksum/config |
| [03-release-lifecycle.md](03-release-lifecycle.md) | install / upgrade / rollback / uninstall, history, hooks (pre/post-install/upgrade), tests |
| [04-values-design.md](04-values-design.md) | 우선순위, schema, override 패턴, library chart, multi-env |
| [05-repos-and-distribution.md](05-repos-and-distribution.md) | helm repo, OCI registry, ChartMuseum, dependency 관리, version constraint |

---

## 🔗 Cross-reference

- **K8s 자원 자체** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **ConfigMap·Secret + checksum/config 패턴** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md)
- **Operator (Helm과의 분담)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **ArgoCD가 chart 렌더링** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)
- **CI에서 chart lint/test/publish** → [`cicd-gitops-basics/01`](../cicd-gitops-basics/01-ci-fundamentals.md)

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| Helm chart의 구조와 각 파일의 역할? | [01](01-chart-structure.md) |
| values 우선순위? | [04](04-values-design.md) |
| `checksum/config` annotation 패턴이 왜 필요? | [02](02-templating.md) |
| helm hook으로 *DB migration* 어떻게? | [03](03-release-lifecycle.md) |
| Helm rollback 가능 원리? release 정보 어디 저장? | [03](03-release-lifecycle.md) |
| Sub-chart 의존성 관리? | [01](01-chart-structure.md), [05](05-repos-and-distribution.md) |
| ArgoCD + Helm 같이 쓸 때 주의점? | [03](03-release-lifecycle.md) + [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md) |

---

## 📐 학습 순서 권장

1. **01 chart-structure** — *Helm chart가 무엇인지*
2. **02 templating** — *Go template 동작*
3. **04 values-design** — *override 패턴* (운영 정수)
4. **03 release-lifecycle** — *install/upgrade/rollback/hook*
5. **05 repos-and-distribution** — *chart 배포·의존성*

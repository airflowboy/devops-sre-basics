# 03 — Release Lifecycle: install·upgrade·rollback·hooks·tests

> **공식 근거:** [Helm Commands](https://helm.sh/docs/helm/helm/), [Chart Hooks](https://helm.sh/docs/topics/charts_hooks/)
>
> **선행:** [01-chart-structure.md](01-chart-structure.md), [02-templating.md](02-templating.md)

---

## 🎯 한 문장

> **Release = *cluster에 install된 chart 인스턴스*. install/upgrade/rollback/uninstall이 라이프사이클. *release 정보는 cluster의 Secret에 저장* (etcd) — 그래서 rollback이 가능. *hook*으로 install/upgrade 단계 *전후*에 *별도 자원 (Job 등)* 실행.**

---

## 1. install — *처음 설치*

```bash
# 기본
helm install myrelease ./mychart

# values 파일 + override
helm install myrelease ./mychart -f values-prod.yaml --set replicaCount=5

# remote chart
helm install myrelease bitnami/postgresql -n database --create-namespace

# dry-run (시뮬레이션)
helm install myrelease ./mychart --dry-run --debug

# 대기 (모든 자원 Ready까지)
helm install myrelease ./mychart --wait --timeout 5m
```

### `--wait` 옵션
- 자원이 *Ready 상태*까지 대기
- *Deployment*: replicas 모두 Ready
- *Service*: External IP 받음 (LoadBalancer 타입)
- *Pod*: containers Ready
- 타임아웃 (`--timeout 5m`) 안에 못 되면 *fail*
- CI에서 *deploy 성공 확인* 용

### `--atomic`
- 실패 시 *자동 rollback*
- 부분 적용 상태 안 남김
- 운영 권장

```bash
helm install myrelease ./mychart --atomic --wait --timeout 10m
```

---

## 2. upgrade — *기존 release 변경*

```bash
helm upgrade myrelease ./mychart -f values-prod.yaml --set image.tag=v1.2.3

# install if not exists, upgrade if exists
helm upgrade --install myrelease ./mychart -f values-prod.yaml
```

`--install` 패턴 = CI에서 *idempotent*. *처음이면 install, 이후엔 upgrade*.

### upgrade의 동작
1. *새 manifest 렌더링*
2. *3-way merge*: *기존 cluster 상태* + *이전 release manifest* + *새 manifest*
3. *변경분만 patch* (전부 재생성 아님)
4. *새 revision 기록* (release Secret에 history 추가)

### 3-way merge의 함의
- *cluster에서 수동 변경한 부분*도 *고려*
- 단, *Helm이 모르는 자원* (다른 도구가 만든 것)은 무시

### 흔한 함정
- *`--force`* 옵션은 *delete + recreate* — Pod 죽음. 보통 안 씀.
- *immutable field 변경* (예: Service의 selector, PV의 storageClass) → upgrade fail. 수동 delete 또는 recreate 필요.

---

## 3. rollback — *이전 revision으로*

```bash
# history 보기
helm history myrelease

# REVISION  UPDATED                  STATUS      CHART          APP VERSION
# 1         Mon May 1 10:00:00 ...  superseded  myapp-0.1.0    1.0.0
# 2         Tue May 2 14:30:00 ...  superseded  myapp-0.1.1    1.1.0
# 3         Wed May 3 09:00:00 ...  deployed    myapp-0.1.2    1.2.0

# 직전으로
helm rollback myrelease

# 특정 revision으로
helm rollback myrelease 1

# 대기 / atomic
helm rollback myrelease 1 --wait --timeout 5m
```

### Rollback 가능 원리
- *모든 revision의 manifest + values가 cluster Secret에 저장됨*
- secret 이름: `sh.helm.release.v1.{release}.v{revision}`
- 압축됨 (gzip + base64)
- → 그 secret 읽어 *그 시점의 manifest 다시 적용*

### History 한도
- 기본 `--history-max 10` — *최근 10개 revision만 보존*
- 옛 revision은 *자동 정리*
- 확인: `helm history myrelease`

```bash
# CI에서 history 크기 조정
helm upgrade myrelease ./mychart --history-max 5
```

---

## 4. uninstall — *release 삭제*

```bash
helm uninstall myrelease
```

- *모든 release manifest 삭제*
- *history도 함께 삭제* (default)
- *PVC는 보통 안 사라짐* (StatefulSet의 `volumeClaimTemplates`)
- *CRD도 안 사라짐* (Helm은 CRD 안 건드림)

```bash
# history만 보존
helm uninstall myrelease --keep-history
```

→ 이후에 *같은 이름 install* 시 *history 이어짐*. *재설치 후 rollback* 으로 옛 revision 복귀 가능.

---

## 5. Release 정보 — *cluster Secret에*

```bash
# release secret 확인
kubectl get secret -n {namespace} -l owner=helm
# NAME                          TYPE
# sh.helm.release.v1.myrelease.v1   helm.sh/release.v1
# sh.helm.release.v1.myrelease.v2   helm.sh/release.v1
# sh.helm.release.v1.myrelease.v3   helm.sh/release.v1

# 한 secret 보기 (압축됨)
kubectl get secret sh.helm.release.v1.myrelease.v3 -n {namespace} -o jsonpath='{.data.release}' | base64 -d | gunzip -
```

→ 안에 *해당 revision의 manifest + values + chart metadata* JSON.

### Storage 백엔드 변경
- 기본: Secret (`HELM_DRIVER=secret`)
- 옵션: ConfigMap, SQL 등
- 보통 Secret이 표준 (encrypted at rest 적용 가능)

---

## 6. Hooks — *install/upgrade 단계 전후 실행*

특정 시점에 *별도 Job 등 자원 실행*. 흔한 활용: *DB migration*, *secret pre-fetch*, *post-install validation*.

### Hook annotation
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: myapp:{{ .Values.image.tag }}
        command: ["./migrate.sh"]
```

### Hook 종류
| Hook | 언제 |
|---|---|
| `pre-install` | install 직전 |
| `post-install` | install 후 자원 Ready 된 후 |
| `pre-upgrade` | upgrade 직전 |
| `post-upgrade` | upgrade 후 |
| `pre-delete` | uninstall 직전 |
| `post-delete` | uninstall 후 |
| `pre-rollback` | rollback 직전 |
| `post-rollback` | rollback 후 |
| `test` | `helm test` 실행 시 |

### Hook weight
- 같은 hook 단계에 *여러 자원* 있으면 *weight 순서대로* (낮은 weight 먼저)

### Hook delete policy
| 정책 | 의미 |
|---|---|
| `before-hook-creation` (기본) | 새 hook 생성 *전* 기존 같은 이름 삭제 |
| `hook-succeeded` | hook 성공 시 *삭제* (Job 정리) |
| `hook-failed` | hook 실패 시 *삭제* |

→ 둘 다 (`before-hook-creation,hook-succeeded`) 가 흔한 패턴 — 성공 시 자동 정리, 실패 시 보존 (디버깅).

### 함정
- Hook 자원은 *release manifest에 포함 안 됨* → `helm rollback`이 *hook 자원은 안 되돌림*
- Hook Job 실패 → *전체 helm operation 실패*
- *DB migration 실패 시 release 자체가 fail* → atomic 옵션이면 자동 rollback

---

## 7. helm test — *설치 검증*

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
  - name: wget
    image: busybox
    command: ["wget"]
    args: ["{{ include "mychart.fullname" . }}:{{ .Values.service.port }}"]
```

```bash
helm test myrelease
```

- *test annotation 있는 Pod 실행*
- *성공/실패* 보고
- *post-install verification* 용
- CI에서 *deploy 성공 검증* 도구

---

## 8. ArgoCD + Helm — *유의점*

### ArgoCD의 helm 사용 방식
- *helm CLI 안 씀* (release Secret 안 만듦)
- 내부적으로 `helm template` 호출해 *manifest 렌더링*
- 그 manifest를 *직접 K8s에 apply*

### 함의
- `helm list` 에 *안 나타남*
- `helm rollback` *작동 안 함* — ArgoCD가 *git 기준으로 reconcile*
- *rollback*은 *git commit revert*로
- *hook 자원*은 ArgoCD가 *일반 자원처럼 처리*
- *hook weight·delete policy 일부 무시* — *sync wave* (`argocd.argoproj.io/sync-wave`)로 대체

### Argo CD 패턴 권장
```yaml
# Application
spec:
  source:
    helm:
      valueFiles:
      - values-prod.yaml
      parameters:
      - name: image.tag
        value: v1.2.3
```

→ values 파일 + parameters로 *git에 명세*. *helm CLI 없이* 같은 효과.

→ 자세히 [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md).

---

## 9. helm secret 자체의 보안

- Release Secret = *manifest + values 모두 포함*
- *values에 secret 박혀 있으면* release Secret에도 있음
- → release Secret 자체 *RBAC 빡세게*
- → *values에 평문 secret 안 박기* — 외부 secret manager + External Secrets Operator ([`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md))

---

## 10. 자주 헷갈리는 것

### 10-1. *upgrade 후 *변경 안 됐는데 새 revision* 생김*
- *content 변경 없어도* `helm upgrade` 호출 시 *revision ↑*
- 의도된 동작 — *history 추적용*

### 10-2. *옛 revision rollback 후 다시 upgrade* 시 revision 번호
- rollback도 *새 revision 번호*. 절대 *옛 번호 재사용 안 함*.
- 예: 1→2→rollback to 1 → revision 3 (= 1과 같은 내용)

### 10-3. *History full → 옛 revision 사라짐 → rollback 불가*
- `--history-max` 기본 10
- 그 이상 upgrade 시 *옛 것 자동 삭제*
- 운영 critical chart는 `--history-max 30` 등 늘리거나, 별도 *git에 manifest 백업*

### 10-4. *Hook Job이 *영원히 실행 중*에서 stuck*
- 보통 *script 안에서 무한 loop* 또는 *DB 응답 대기 무한*
- `helm install --timeout 5m` 으로 강제 한도
- `kubectl delete job/{hook-job}` 로 정리

### 10-5. *helm install/upgrade 도중 *ctrl+c**
- 일부 자원만 적용된 상태 남을 수 있음
- `--atomic` 으로 *자동 rollback*
- 안 쓰면 `helm rollback` 또는 `helm uninstall` 후 재시도

### 10-6. *Subchart의 hook도 같이 실행*
- 의도와 다를 수 있음 — sub-chart의 hook이 *parent install 단계에 실행*
- *DB migration sub-chart 만들었더니 parent install마다 migrate*

---

## 🎤 면접 빈출 Q&A

### Q1. Helm rollback이 가능한 원리는?
> 모든 revision의 *manifest + values가 cluster의 Secret으로 저장됨* (`sh.helm.release.v1.{name}.v{N}`). 압축됨. `helm rollback` 시 *그 secret 읽어 그 시점의 manifest 다시 apply*. `--history-max` (기본 10)만큼만 보존 — 그 이상은 자동 정리.

### Q2. `helm install` vs `helm upgrade --install` ?
> `install` = release 없을 때만. 이미 있으면 fail. `upgrade --install` = *idempotent* — 없으면 install, 있으면 upgrade. CI에서 *항상 후자* — 처음/이후 구분 없이 같은 명령.

### Q3. `--atomic` 옵션은?
> 실패 시 *자동 rollback*. 부분 적용 상태 안 남김. 운영 권장. `--wait --timeout` 과 함께 — wait 안에 자원 Ready 안 되면 fail → atomic이 rollback.

### Q4. Helm hook으로 DB migration 어떻게?
> `pre-upgrade,pre-install` annotation 있는 *Job 자원* 정의. install/upgrade 직전 실행. 실패 시 *전체 helm operation 실패*. `helm.sh/hook-weight` 로 *여러 hook 순서*, `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded` 로 *성공 시 정리, 실패 시 보존*. 단, hook 자원은 release manifest에 없어서 *rollback 안 됨* — DB migration 자체의 idempotency·forward-compatible 설계 필수.

### Q5. ArgoCD + Helm 같이 쓸 때 주의?
> ArgoCD는 *helm CLI 안 씀* — 내부적으로 `helm template`만 호출해 manifest 렌더링 후 *직접 apply*. → *release Secret 안 만듦*, `helm list` 안 보임, `helm rollback` 작동 X. *rollback은 git revert*. *hook weight 일부 무시* — ArgoCD `sync-wave` 로 대체.

### Q6. Release Secret이 *압축*된 이유? 어떻게 보나?
> manifest + values 크면 Secret 크기 초과 (etcd value 1MB 한도). gzip 압축 + base64 인코딩. `kubectl get secret {name} -o jsonpath='{.data.release}' | base64 -d | gunzip -` 로 풀어볼 수 있음. 내용은 JSON — chart 정보 + values + 모든 manifest.

### Q7. *upgrade가 immutable field 변경* 시 어떻게?
> Service의 selector·PV의 storageClass 같은 *immutable field 변경* → upgrade fail. *helm이 patch 못 함*. 해결: (1) *수동 delete + helm install*, (2) `--force` (delete + recreate, downtime), (3) *자원 이름 변경 + 옛 자원 수동 정리*. 대규모 변경은 *블루-그린 패턴*도 옵션.

---

## 🔗 Cross-reference

- **chart 구조** → [01-chart-structure.md](01-chart-structure.md)
- **template 동작** → [02-templating.md](02-templating.md)
- **values 우선순위** → [04-values-design.md](04-values-design.md)
- **K8s Secret · etcd encryption** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md)
- **ArgoCD가 Helm chart 다루는 법** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. Release = *cluster에 install된 chart 인스턴스*. 모든 revision의 manifest+values가 *cluster Secret에 저장* → rollback 가능. `--history-max` (기본 10).
2. `helm upgrade --install --atomic --wait --timeout`이 CI 표준 패턴. *idempotent + 실패 시 자동 rollback + Ready 대기*.
3. Hook으로 DB migration 같은 *전후 작업* 가능. ArgoCD는 *helm CLI 안 씀* — manifest 렌더링만, rollback은 *git revert*.

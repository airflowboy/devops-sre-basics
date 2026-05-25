# 06 — API와 Controller: K8s가 *영원히 reconcile*하는 메커니즘

> **공식 근거:** [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/), [API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/), [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
>
> **이 파일이 *K8s를 K8s답게 만드는 메타 개념*.** 다른 모든 자원은 이 패턴의 인스턴스.

---

## 🎯 한 문장

> **K8s = *선언적 API + Controller가 영원히 reconcile하는 framework*. 모든 내장 자원 (Deployment·Service·…) 도 *이 패턴의 인스턴스*. CRD + 직접 만든 controller = Operator. K8s 자체가 *operator framework의 reference 구현*.**

---

## 1. *선언적*과 *명령적* — 왜 선언적인가

| | 명령적 (imperative) | 선언적 (declarative) |
|---|---|---|
| 표현 | "이 명령을 *실행해라*" | "*결과가 이렇게 돼야 한다*" |
| 예 | `kubectl create deployment ...` | `kubectl apply -f deploy.yaml` |
| 재현성 | 명령 히스토리 필요 | YAML 보관만 |
| 일관성 | 누가 무엇 실행했는지 추적 어려움 | git이 single source of truth |
| K8s 친화성 | 낮음 | 높음 (모든 controller가 선언적) |

→ K8s의 *모든 컴포넌트가 reconciliation으로 동작*하니 *선언적이 자연스러움*.
→ GitOps (ArgoCD)·Terraform·Helm 다 *같은 사상*.

---

## 2. API의 구조 — GVK (Group / Version / Kind)

K8s 자원은 *3계층 식별자*로 명확히 분리:

```yaml
apiVersion: apps/v1   ←  Group: apps, Version: v1
kind: Deployment      ←  Kind
metadata:
  name: myapp
  namespace: default
spec: {...}
status: {...}
```

| 부분 | 의미 |
|---|---|
| **Group** | 자원 묶음 (`apps`, `batch`, `networking.k8s.io`, ...) |
| **Version** | API 버전 (`v1alpha1` → `v1beta1` → `v1`) — *deprecation 사이클* |
| **Kind** | 자원 종류 (`Deployment`, `Service`, ...) |

### Core Group
- `apiVersion: v1` 처럼 *group 생략 = core group* (Pod, Service, ConfigMap 등 가장 기본)
- 다른 group은 `apiVersion: <group>/<version>` 형식

### API Discovery
- 클러스터가 어떤 *GVK 지원*하는지 *런타임에 알 수 있음*
- `kubectl api-resources` — 전체 list
- `kubectl explain deployment.spec` — 특정 필드 문서

### Version Skew
- *같은 Kind*에 *여러 Version 동시 지원* 가능 (v1alpha1, v1beta1, v1)
- API server가 *internal version으로 변환* 후 처리
- *Deprecation cycle* — v1beta1 → v1 *최소 N 릴리즈 유지* 후 제거

---

## 3. *spec vs status* — 한 자원의 *두 얼굴*

```yaml
spec:        # ← 사용자가 *원하는 상태* (desired state)
  replicas: 3
  template: {...}
status:      # ← 컨트롤러가 *현재 상태* 보고 (actual state)
  readyReplicas: 2
  conditions: [...]
```

- **사용자는 spec만 변경** (`kubectl apply`, `kubectl edit`)
- **controller는 status만 변경** (자신이 관찰한 결과 보고)
- *spec vs status*의 차이가 *reconcile할 일*

→ 이게 *선언적 시스템의 핵심 메커니즘*.

---

## 4. Controller Pattern — *영원한 loop*

### 의사 코드
```python
while True:
    desired = api_server.get(MyResource)         # spec 읽기
    actual = api_server.get(RelatedResources)    # 현재 상태 관찰
    
    diff = compute_diff(desired, actual)
    if diff:
        api_server.patch(...)                     # 조치
        api_server.update_status(MyResource, ...)
    
    wait_for_change()                             # Watch API로 다음 변화 대기
```

### 핵심 특징
1. **Idempotent** — 같은 desired 상태에서 *몇 번 reconcile해도 같은 결과*
2. **Eventually consistent** — 즉시 X. *결국* desired로 수렴.
3. **Watch-driven** — polling 아니라 *변화 push*받음 → 빠르고 효율
4. **Self-healing** — 누가 actual을 *직접 바꿔도* (예: 수동 `kubectl delete pod`) 다음 reconcile에서 *원래대로 복원*

### 예시: ReplicaSet controller
```
ReplicaSet:
  spec.replicas: 3
  selector: {app: web}

actual: matching Pods = [pod1, pod2]   ← 2개만 있음 (3 < spec)

→ controller: Pod 1개 더 생성

actual: matching Pods = [pod1, pod2, pod3]
→ diff 0 → 다음 변화 대기
```

만약 `pod2`를 *수동으로 delete* 하면:
```
actual: [pod1, pod3]   ← 2개로 줄어듦

→ controller가 watch로 감지
→ 또 Pod 1개 생성
```
이게 *self-heal*. 내가 수동 delete를 *영원히 못 막음* — controller가 매번 복원.

→ Deployment·StatefulSet·Job·Service·Endpoint·... *모든 내장 자원이 같은 패턴*.

---

## 5. Watch API — *push-based의 베이스*

### Watch 흐름
```
client → GET /api/v1/pods?watch=true&resourceVersion=12345
       ← (HTTP/2 long-lived stream)
         {type: ADDED,    object: pod1, resourceVersion: 12346}
         {type: MODIFIED, object: pod2, resourceVersion: 12347}
         {type: DELETED,  object: pod3, resourceVersion: 12348}
         ...
```

- *resourceVersion* 으로 *어디부터 이어볼지* 명시
- 연결 끊김 → 마지막 RV부터 *재시작* → 놓치는 이벤트 없음
- API server는 *etcd watch*를 통해 변화 수신, *모든 watcher에 push*

### Informer / Reflector — Controller가 watch 활용하는 표준
```
Reflector → Watch (long-lived) → Local Cache (in-memory)
                                       │
                                       └─→ Event Handler (controller logic)
```

- *Local cache*에 *클러스터 상태 mirror*
- controller logic은 *cache만 읽음* (API server 안 부름) → 빠르고 부하 없음
- 변화는 *event handler*로 push

→ Operator 만드는 사람이 *직접 watch 처리하지 않음*. 표준 client-go 라이브러리가 다 처리.

---

## 6. CRD (Custom Resource Definition) — *내 자원 만들기*

### 왜
- *내 도메인 자원*도 K8s 자원으로 다루고 싶다 (kubectl·RBAC·watch 다 활용)
- 예: `PostgresCluster`, `ReplicaQueue`, `RedisFailover`

### 정의 — CRD 자원
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.acid.zalan.do
spec:
  group: acid.zalan.do
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              numberOfInstances: { type: integer, minimum: 1 }
              postgresql:
                type: object
                properties:
                  version: { type: string }
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: postgrescluster
    shortNames: [pgcluster]
```

CRD 적용 후:
```bash
kubectl get crd
kubectl get postgresclusters     # ← 새 자원 종류로 쿼리 가능
```

### CRD만으로는 *아무 일도 안 일어남*
- CRD는 *스키마 등록*일 뿐. API server가 *저장·조회 가능하게* 함.
- 실제 *PostgresCluster를 보고 Pod 만드는 controller*가 따로 있어야.
- → CRD + controller = **Operator**

---

## 7. Operator Pattern — *Domain-specific Controller*

### 정의
- *CRD로 도메인 자원 정의*
- *그 자원을 watch하고 reconcile하는 controller 직접 작성*
- *운영자(operator)의 지식을 자동화*

### 예시: PostgreSQL Operator
- `PostgresCluster` 자원 정의 (replicas, version, backup schedule, ...)
- Operator가 watch:
  - 새 PostgresCluster 등장 → *StatefulSet + Service + ConfigMap + Secret + PVC* 자동 생성
  - replicas 변경 → 안전하게 *scale* (DB 안전 절차 따름)
  - 백업 schedule → CronJob 생성
  - 장애 → primary 승격, replica 재생성

### 실제 Operator들
- **Strimzi** — Kafka
- **Postgres-Operator (Zalando)** — PostgreSQL HA
- **Cert-manager** — TLS 인증서 자동 관리
- **Prometheus Operator** — Prometheus·Alertmanager·ServiceMonitor 자원
- **ArgoCD** — Application·AppProject (CRD + controller)
- **External Secrets Operator** — 외부 secret manager 통합
- **AWS Load Balancer Controller** — Ingress·Service → AWS ALB/NLB

→ *K8s 생태계의 거의 모든 도구*가 Operator 형태.

### Operator를 *어떻게 만드나*
- **Kubebuilder** / **Operator SDK** — Go 기반 framework
- **Metacontroller** — Python·Bash 등 다른 언어 가능
- **client-go** 직접 사용 — 가장 raw

→ 면접 빈출: *"Operator를 직접 만들어본 적 있나요?"* 모르면 *개념 정확히* 라도 답.

---

## 8. Admission Controllers — *etcd 저장 전 마지막 관문*

API server가 *etcd 저장 직전*에 자원을 *검사·변형*하는 단계. [01 architecture](01-architecture.md) 의 Section 3 흐름 복습:

```
요청 → 인증 → 인가 → Mutating Admission → 검증 → Validating Admission → etcd 저장
                       ▲                              ▲
                       │                              │
                  (자원 변형)                    (자원 검증)
```

### Mutating vs Validating

**Mutating** — *자원을 수정해서 다음 단계로 넘김*
- 예: sidecar 자동 주입 (Istio, Linkerd)
- 예: default value 채우기
- 예: image registry 자동 변경 (mirror로)

**Validating** — *정책 위반 검사. 위반이면 거부*
- 예: image가 *허가된 registry*에서 오나
- 예: Pod이 *서명된 image*만 쓰나 (Cosign 검증)
- 예: 모든 Pod에 *requests/limits 설정 강제*
- 예: privileged container 차단

### 종류
1. **Built-in admission plugins** — API server에 *컴파일된* 플러그인 (예: `NamespaceLifecycle`, `LimitRanger`, `ResourceQuota`, `PodSecurity`)
2. **Webhook admission** — *외부 서비스 호출* — Kyverno, OPA Gatekeeper, Sigstore policy-controller 등

### Webhook 동작
```
API server → POST /validate (또는 /mutate) → 외부 admission webhook
              ↑                                       │
              └──────── allowed/denied + 메시지 ──────┘
```

### 흔한 활용
- **Pod Security Standards** — `PodSecurity` admission plugin (1.25+ 내장)
- **Image 서명 검증** — Sigstore policy-controller
- **정책 강제** — Kyverno (선언적) / OPA Gatekeeper (Rego)
- **자원 라벨 강제** — 모든 Deployment에 `app.kubernetes.io/owner: team-x` 자동 추가

→ `k8s-security-basics/` (예정) 에서 더 깊이.

---

## 9. API Aggregation — *외부 API 서버를 K8s API에 통합*

- 새 API group을 *K8s API 위에* 얹기
- API server가 *aggregator로 동작* — 특정 path는 *외부 API server에 위임*
- 예: `metrics.k8s.io/v1beta1` (metrics-server), `custom.metrics.k8s.io` (custom metrics for HPA)

CRD와 비교:
| | CRD | API Aggregation |
|---|---|---|
| 저장 | etcd | 외부 (별도) |
| 구현 노력 | 적음 (스키마만) | 큼 (API server 작성) |
| 유연성 | 제한 (CRUD + status) | 자유 (임의 verb, 커스텀 동작) |
| 사용 빈도 | 매우 흔함 | 드뭄 (metrics 등 특수) |

→ 대부분은 CRD로 충분. API Aggregation은 *특수 케이스* (metric API 등).

---

## 10. 자주 헷갈리는 것

### 10-1. CRD를 만들었지만 *아무 일도 안 일어남*
- CRD = 스키마만. *controller가 따로 있어야* 실제 동작.
- 자원 만들어 etcd에 저장은 되나, *그걸 보고 뭘 하는 사람*이 없음.

### 10-2. *Operator vs Helm chart*
- *Helm chart* = *static rendering* (template → YAML). *배포 시점 후엔 관여 X*.
- *Operator* = *영원히 watch + reconcile*. *런타임 변화에도 대응*.
- → 자원을 *한 번 만들고 끝*이면 Helm. *지속적 운영 관리* 필요하면 Operator.
- 둘 결합도 흔함 (Helm으로 Operator를 *설치*, 그 다음 Operator가 *도메인 자원 관리*).

### 10-3. *Watch가 끊겼는데 모르고* — 상태 stale
- 네트워크 일시 끊김·API server restart → watch 끊김
- Reflector가 *재연결* + last RV부터 *재시작*
- 그러나 *너무 옛 RV*면 *full LIST → relist*. *대규모 클러스터*에선 *부하 폭증*.

### 10-4. *Spec 변경했는데 status 안 바뀜*
- controller가 *spec watch* 하나? (CRD라면 직접 만든 controller가 watching인지)
- controller logic에 *bug* 가능성
- *eventually consistent* — *시간이 좀 지나야* 반영될 수도 (특히 *exponential backoff* 중)

### 10-5. *Admission webhook이 fail* → *모든 자원 생성 차단*
- Validating webhook의 *외부 서비스가 down*이면 *모든 자원 생성 fail* (default behavior)
- → `failurePolicy: Ignore` 설정 (webhook fail 시 *허용*) — 보안 떨어지나 *서비스 다운 막음*
- *namespace selector*로 *특정 네임스페이스만* 적용 — 시스템 ns 제외

---

## 🎤 면접 빈출 Q&A

### Q1. K8s controller pattern을 설명해주세요.
> *Desired state (spec)* 와 *Current state (status)* 를 *영원히 비교*하며 *current를 desired로 수렴*시키는 loop. *Watch API*로 push-driven, *idempotent*, *self-healing* (수동 변경도 다음 reconcile에서 복원). ReplicaSet controller가 spec.replicas 만큼 Pod 유지하는 게 가장 단순한 예. 모든 내장 자원이 이 패턴.

### Q2. CRD와 Operator의 관계?
> CRD = *내 도메인 자원의 스키마 등록*. K8s API server가 *저장·조회 가능하게* 함. **CRD만으로는 아무 일도 안 일어남**. 그 자원을 *watch하고 reconcile하는 controller*를 직접 작성하면 **Operator**. 운영자의 지식을 자동화. Strimzi (Kafka), Postgres-Operator, cert-manager 등이 대표.

### Q3. Admission Controller의 역할?
> API server가 *etcd 저장 직전* 자원을 *검사·변형*. **Mutating**: 자원 수정 (sidecar 자동 주입, default 채우기). **Validating**: 정책 위반 검사·거부 (서명되지 않은 image 차단, requests/limits 강제). *Webhook*으로 외부 서비스 호출 가능 (Kyverno, OPA Gatekeeper, Sigstore policy-controller).

### Q4. Watch API와 polling 차이?
> *Watch*는 *long-lived stream*으로 *변화 push*. *polling*은 주기적 fetch. K8s controller들은 *전부 watch* (Informer/Reflector 표준 라이브러리). 장점: *즉시 반응, 부하 적음, scale 좋음*. 연결 끊겨도 *마지막 resourceVersion*으로 *재시작* — 놓치는 이벤트 없음.

### Q5. spec과 status는 누가 변경하나요?
> *spec은 사용자만* (apply/edit). *status는 controller만* (관찰한 결과 보고). 분리 이유: *single writer 원칙*으로 conflict 줄임. 사용자가 status를 직접 수정하면 controller가 다음 reconcile에서 *덮어씀*. 이 분리가 *선언적 시스템의 핵심*.

### Q6. Helm chart와 Operator의 차이? 언제 무엇?
> **Helm**: *template → static YAML rendering*. 배포 시점 후엔 관여 X. *한 번 만들고 끝나는* 자원에 적합. **Operator**: *영원히 watch + reconcile*. *런타임 변화* (자동 failover, 백업, scale 안전 절차)에 대응. → DB·메시지 큐 같이 *지속적 운영* 필요하면 Operator. 단순 앱 배포면 Helm.

### Q7. CRD 만들었는데 *자원 생성해도 아무것도 안 만들어진다*. 왜?
> CRD는 *스키마 등록만*. *그 자원을 보고 뭘 만드는 controller*가 별도 필요. → Operator를 *설치*했나 확인. 설치했다면 *Operator Pod 떴나*, *로그에 watch 시작 메시지* 있나. Operator가 *RBAC 권한 부족* 으로 watch 실패도 흔함.

---

## 🔗 Cross-reference

- **API server·etcd·Watch 흐름** → [01-architecture.md](01-architecture.md)
- **내장 자원들 (Deployment·StatefulSet 등)이 이 패턴의 인스턴스** → [02-workloads.md](02-workloads.md)
- **Admission webhook으로 *서명 검증* / *PSS 강제*** → [07-rbac-and-security.md](07-rbac-and-security.md), `k8s-security-basics/` (예정)
- **선언적 reconciliation의 *외부 버전*** — ArgoCD (GitOps), Terraform (IaC) → `cicd-gitops-basics/`, `terraform-basics/`
- **Operator 사례: Prometheus Operator (ServiceMonitor·PrometheusRule)** → `observability-basics/` (예정)
- **Cert-Manager·External Secrets Operator** → `aws-basics/`, `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *K8s = 선언적 API + Watch-driven controller reconciliation framework*. spec(원하는 상태) vs status(현재) 차이를 *영원히 메움*.
2. *CRD + 직접 작성한 controller = Operator*. 운영자의 지식을 자동화. K8s 생태계 거의 모든 도구가 Operator.
3. *Admission Controller*는 etcd 저장 직전 *검사·변형*. Webhook으로 *정책 강제* (PSS, 서명 검증, label 강제 등).

# 02 — Workloads: Pod, Deployment, StatefulSet, DaemonSet, Job

> **공식 근거:** [Workloads](https://kubernetes.io/docs/concepts/workloads/), [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
>
> **선행:** [01-architecture.md](01-architecture.md) (controller pattern 이해)

---

## 🎯 한 문장

> **Pod = K8s의 *가장 작은 배포 단위* (= namespace 공유하는 컨테이너 1+개). 그 위에 *목적별로 다른 controller* (Deployment / StatefulSet / DaemonSet / Job …) 가 *Pod 개수·이름·라이프사이클*을 관리.**

---

## 1. Pod — *컨테이너 묶음, 격리의 단위*

### Pod = *함께 살고 함께 죽는 컨테이너들*
- 같은 Pod의 컨테이너들은:
  - 같은 **네트워크 namespace** → `localhost`로 통신, 같은 IP, 포트 공유
  - 같은 **IPC namespace** → shared memory
  - 같은 **UTS namespace** → 같은 hostname
  - *분리된* **MNT namespace** (각자 자기 rootfs) → 파일은 안 공유 (단, *Volume* 으로 공유 가능)
  - *분리된* **PID namespace** (기본). `shareProcessNamespace: true` 로 공유 가능.

→ [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 의 *Pod 격리* 와 정확히 같은 원리.

### 한 Pod에 *왜* 컨테이너 여러 개?
- **Sidecar 패턴** — 로그 수집기, sync agent, proxy 등
  - 예: 메인 앱 + Fluent Bit (로그 수집)
  - 예: 메인 앱 + Envoy (sidecar proxy)
- **Init container** — 메인 컨테이너 시작 *전*에 실행, 모두 성공해야 메인 시작
  - 예: 마이그레이션 실행, 의존 서비스 대기, secret fetch
- **Ambassador 패턴** — 외부 서비스 추상화

### Pod이 *사실상 단독 Pod로 쓰이지 않는 이유*
- *재시작·재배치 안 됨* (naked Pod). 죽으면 *그냥 죽음*.
- → 항상 *Deployment·StatefulSet·DaemonSet 같은 *상위 controller*에 의해 만들어짐.

### Pod Lifecycle
```
Pending  →  Running  →  Succeeded
                   ↘
                    Failed
                   ↘
                    Unknown
```

세부 단계 (특히 Running 안):
- **PodScheduled** condition — scheduler가 노드 결정함
- **Initialized** — 모든 init container 성공
- **ContainersReady** — 모든 컨테이너의 *readiness probe* 통과
- **Ready** — Pod 자체가 Ready (Service의 Endpoint에 추가됨)

상태 확인: `kubectl describe pod` 의 *Conditions* 섹션.

### Probe 3종
| Probe | 목적 | 실패 시 |
|---|---|---|
| **liveness** | "살아있나?" | 컨테이너 재시작 |
| **readiness** | "트래픽 받을 준비?" | Service Endpoint에서 *제외* (재시작 X) |
| **startup** | "시작 다 됐나?" (느린 앱) | 통과 전까진 liveness/readiness 미실행 |

흔한 함정:
- *liveness만 설정* → 트래픽 받을 준비 안 됐는데 Endpoint에 들어가 *503 폭주*
- *readiness만 설정* → 죽어가는 컨테이너 *영원히 살아있음* (재시작 안 됨)
- *둘 다 같은 endpoint* → readiness 잠깐 fail → liveness도 fail → *재시작* (의도 아닐 수 있음)

권장: *readiness는 가볍게 (load 받을 준비)*, *liveness는 무겁게 (진짜 죽은 거 확실할 때만)*.

---

## 2. ReplicaSet — *Pod 개수 보장*

- *직접 만들지 않음.* Deployment가 만들어줌.
- 역할 단순: *spec.replicas 만큼 Pod 유지*. 모자라면 생성, 남으면 *삭제* (어느 걸 죽일지는 PodDisruptionBudget·priority 고려).
- *selector* 로 *어느 Pod이 자기 소속인지* 식별.

→ ReplicaSet 자체를 *직접 만들 일은 거의 없음*. Deployment가 표준.

---

## 3. Deployment — *가장 흔히 쓰는 workload*

### 책임
- *ReplicaSet을 만들고 관리*
- 새 image·spec 변경 시 *RollingUpdate* (또는 Recreate)
- *Revision history* 보존 (옛 RS들이 replicas=0으로 남음)
- `rollout undo`, `rollout pause`, `rollout status` 지원

### RollingUpdate 동작
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # 새 Pod를 desired보다 최대 몇 % 더 만들 수 있나
    maxUnavailable: 25%  # 동시에 unavailable일 수 있는 최대 %
```

흐름 (Replicas: 4, maxSurge: 25%(=1), maxUnavailable: 25%(=1)):
```
초기:  [old1] [old2] [old3] [old4]                  Available: 4
1단계: [old1] [old2] [old3] [old4] [new1]           새 RS 1개 추가 (surge)
2단계: [old1] [old2] [old3]        [new1]           old4 종료
3단계: [old1] [old2] [old3]        [new1] [new2]
... 점진적으로 ...
최종:  [new1] [new2] [new3] [new4]
```

→ *zero-downtime* 보장 (readiness probe 통과 후 다음 Pod 교체).

### Recreate vs RollingUpdate
| | Recreate | RollingUpdate (기본) |
|---|---|---|
| 동작 | 모두 죽이고 모두 새로 | 점진 교체 |
| Downtime | 있음 | 없음 |
| 자원 | 최소 | 일시적으로 surge 필요 |
| 용도 | 호환 안 되는 큰 변경 (예: DB schema migration 전후 같은 시점 운영 불가) | 일반적 (기본) |

### Rollback
```bash
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp                 # 직전 revision
kubectl rollout undo deployment/myapp --to-revision=3 # 특정 revision
```
- 옛 RS가 *replicas=0으로 남아있어서* 가능 (revisionHistoryLimit 만큼)

---

## 4. StatefulSet — *순서·이름·storage가 *Pod별로 고유*

### 언제 쓰나
- **상태있는 워크로드** — DB (Cassandra·MongoDB), 메시지 큐 (Kafka, RabbitMQ), 분산 캐시
- *각 Pod이 *고유한 이름·storage·네트워크 ID*를 가져야 할 때

### Deployment와의 차이

| | Deployment | StatefulSet |
|---|---|---|
| Pod 이름 | 랜덤 (`myapp-7f5b9c-abc123`) | 순서 있는 고정 (`mydb-0`, `mydb-1`, `mydb-2`) |
| Storage | 공유 또는 없음 | *각 Pod별 독립된 PVC* |
| 생성·삭제 순서 | 동시 (병렬) | *순서대로* (0 → 1 → 2 ...) |
| 네트워크 ID | 안 고정 | *headless service*로 `pod-name.service.ns.svc` 고정 |
| 업데이트 | 한 번에 여러 개 | *기본 RollingUpdate도 1개씩 순서대로* |

### Headless Service와 짝
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  clusterIP: None     # ← Headless
  selector:
    app: mydb
  ports: [...]
```
- *ClusterIP 안 만듦* → DNS는 *Pod IP 목록 그대로* 반환
- 결과: `mydb-0.mydb.default.svc.cluster.local` 로 *특정 Pod에 직접* 접근 가능

### 함정
- StatefulSet은 *update가 느림* (1개씩) — 의도된 안전장치
- *삭제 순서가 역순* (N-1 → 0)
- PV는 *StatefulSet 삭제해도 안 사라짐* (의도). PVC 직접 삭제 필요.

---

## 5. DaemonSet — *노드당 1개*

### 언제 쓰나
- *모든 노드에 정확히 1개씩* 필요한 워크로드
- 예: log shipper (Fluent Bit), 모니터링 agent (node-exporter), CNI (kube-proxy도 사실 DaemonSet), storage driver

### 특징
- 새 노드 추가 → DaemonSet controller가 *자동으로 그 노드에 Pod 추가*
- 노드 삭제 → 자동 정리
- *nodeSelector·affinity·toleration* 으로 *특정 노드만* 가능 (예: GPU 노드만)
- `hostNetwork: true` 같이 *호스트 자원* 사용하는 경우 많음 (특수 권한)

---

## 6. Job & CronJob — *완료되어야 끝나는 작업*

### Job
- 컨테이너 *성공 종료(exit 0) 까지 실행*
- `completions: 5` — 5번 성공해야 끝
- `parallelism: 3` — 동시에 3개까지 병렬
- *실패 시 재시도* (backoffLimit)

### CronJob
- *cron schedule*에 따라 Job *생성*
- `schedule: "0 2 * * *"` — 매일 02:00에 Job 생성

### 함정
- *concurrencyPolicy* — 이전 Job 아직 도는데 다음 trigger 오면? `Allow`(병행) / `Forbid`(skip) / `Replace`(이전 죽이고 새로)
- *startingDeadlineSeconds* — controller 다운 후 깨어났을 때 *얼마나 오래 지난 schedule까지 catch up*
- *successfulJobsHistoryLimit / failedJobsHistoryLimit* — 옛 Job 정리. *기본값 작음*. 디버깅 시 옛 Job 못 보면 이걸 늘려야.

---

## 7. Init Container — *순차 실행되는 사전 작업*

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  - name: run-migration
    image: myapp:latest
    command: ['./migrate']
  containers:
  - name: app
    image: myapp:latest
```

- *순서대로* (위에서 아래로)
- *각자 종료해야 다음*
- *모두 성공해야 주 컨테이너 시작*
- 실패 시 → Pod 재시작 (restartPolicy에 따라)

### 흔한 활용
- 의존 서비스 ready 대기 (DB·메시지 큐)
- DB 마이그레이션 실행
- 비밀 fetch
- volume에 초기 파일 복사

---

## 8. PodDisruptionBudget — *동시에 죽일 수 있는 한도*

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mydb-pdb
spec:
  selector:
    matchLabels: { app: mydb }
  minAvailable: 2
  # 또는 maxUnavailable: 1
```

- *voluntary disruption* (노드 drain, 업그레이드 등)에서 *동시에 죽일 수 있는 한도* 강제
- *non-voluntary* (노드 hardware 실패 등)에는 영향 X — *말 그대로 보장* 아님
- StatefulSet (특히 DB) 에 *반드시* 추가하는 게 운영 모범

---

## 9. 라벨·셀렉터 — *모든 게 이걸로 묶임*

- Pod의 `metadata.labels` → 자유로운 key=value
- 다른 자원(Service, Deployment, NetworkPolicy 등)이 *selector* 로 *어느 Pod이 자기 대상인지* 식별

### 컨벤션 (`app.kubernetes.io/`)
- `app.kubernetes.io/name` — 앱 이름
- `app.kubernetes.io/instance` — release 인스턴스
- `app.kubernetes.io/version` — 버전
- `app.kubernetes.io/component` — 어떤 컴포넌트 (db, frontend, ...)
- `app.kubernetes.io/part-of` — 어느 시스템에 속하나
- `app.kubernetes.io/managed-by` — 관리 도구 (Helm 등)

→ Helm·ArgoCD가 자동으로 이런 라벨 박음.

---

## 10. 자주 헷갈리는 것

### 10-1. Pod 죽으면 *같은 Pod이 다시 뜨는가*?
- *naked Pod*은 안 뜸 (controller 없으니).
- Deployment/ReplicaSet 안 Pod이면 *새 Pod (다른 이름·UID)* 가 만들어짐.
- StatefulSet은 *같은 이름의 새 Pod*. PV도 *재마운트*.

### 10-2. *Deployment에서 image 안 바꾸고 환경변수만 바꾸면?*
- Pod template 변경 → *새 ReplicaSet 생성* → rolling update.
- 환경변수만 바꿔도 *Pod 재생성됨*. (ConfigMap·Secret 의존 변경은 *재시작 안 됨* — 4 참조)

### 10-3. *ConfigMap·Secret 갱신해도 Pod이 자동 재시작 안 됨*
- volume으로 마운트된 경우 → 컨테이너 안 파일은 *수십 초 안에 갱신* (kubelet sync interval)
- 환경 변수로 inject된 경우 → *Pod 재시작 전엔 옛 값*
- → 흔한 패턴: *ConfigMap 해시를 annotation에 박아 자동 재시작 트리거* (Helm `checksum/config` 패턴)

### 10-4. *Deployment의 rolling update가 stuck*
- 새 RS가 *replicas만큼 안 늘어남*. 보통 readiness probe fail.
- `kubectl describe deployment` → Events
- `kubectl rollout status` → 진행 상황
- *수동 멈춤*: `kubectl rollout pause`
- *되돌리기*: `kubectl rollout undo`

### 10-5. StatefulSet의 *순서가 stuck*
- `mydb-0` 안 ready → `mydb-1` *영원히 시작 안 됨*
- *podManagementPolicy: Parallel* 옵션으로 동시 시작도 가능 (순서 보장 포기)

---

## 🎤 면접 빈출 Q&A

### Q1. Deployment / StatefulSet / DaemonSet 차이?
> **Deployment**: stateless. Pod 이름 랜덤, 동시 생성/삭제. *웹·API 서버* 등 일반 워크로드. **StatefulSet**: stateful. Pod 이름 고정 순서 (`mydb-0,1,2`), 각자 PV, 순서대로 생성/삭제. *DB·Kafka 등 *. **DaemonSet**: *노드당 1개*. log shipper·모니터링 agent.

### Q2. Pod이 죽으면 같은 Pod이 다시 뜨나요?
> 단독 Pod (naked)은 *안 뜸*. controller (Deployment·RS·StatefulSet) 안 Pod이면 *새 Pod이 생성됨*. Deployment는 *새 이름·UID*, StatefulSet은 *같은 이름·새 UID + 같은 PV*. → naked Pod은 운영에서 거의 안 씀.

### Q3. liveness vs readiness probe 차이?
> **liveness**: "살아있나?" 실패 시 *컨테이너 재시작*. **readiness**: "트래픽 받을 준비?" 실패 시 *Service Endpoint에서 제외* (재시작 X). 둘 다 *같은 endpoint*로 설정하면 *의도하지 않은 재시작* 발생 가능. *readiness는 가볍게, liveness는 무겁게*.

### Q4. ConfigMap 변경 시 Pod이 자동 재시작 되나요?
> Volume mount면 *컨테이너 안 파일 갱신* (수십 초). 환경변수면 *Pod 재시작 전엔 옛 값*. *자동 재시작 안 됨*. 해결: Helm `checksum/config` annotation 패턴 — ConfigMap 내용 해시를 Pod template annotation에 박아두면 ConfigMap 바뀔 때 *Pod template 해시 변화 → rolling update*.

### Q5. RollingUpdate가 stuck됐을 때 어떻게 진단?
> `kubectl rollout status` 로 진행 멈춘 지점 확인. `kubectl describe deployment` Events 섹션에서 *왜 멈춘지* (readiness probe fail, image pull fail 등). 새 RS의 Pod `describe`로 자세히. *RollingUpdate paused* 면 `rollout resume`. 진짜 망했으면 `rollout undo`.

### Q6. PodDisruptionBudget이 필요한 이유?
> 노드 drain·upgrade·spot interruption 같은 *voluntary disruption* 시 *동시에 죽일 수 있는 Pod 한도* 강제. DB 같이 *과반 살아있어야 quorum* 인 워크로드에 필수. *non-voluntary* (hardware fail)에는 보장 X — *말 그대로 보장*은 아님.

---

## 🔗 Cross-reference

- **Pod = namespace 공유 컨테이너 묶음** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)
- **Service / Endpoint** (Pod이 *어떻게 외부에 노출되는가*) → [03-networking.md](03-networking.md)
- **PV / PVC** (StatefulSet의 storage) → [04-config-and-storage.md](04-config-and-storage.md)
- **scheduler** (어느 노드에 들어가나) → [05-scheduling.md](05-scheduling.md)
- **controller pattern** (모든 workload의 베이스) → [06-api-and-controllers.md](06-api-and-controllers.md)

---

## 📝 3줄 요약

1. *Pod = 함께 사는 컨테이너 묶음* (namespace 공유). 운영에선 *naked Pod 안 씀* — 항상 controller가 만들어줌.
2. 목적별 controller: *Deployment(stateless), StatefulSet(stateful + PV), DaemonSet(노드당 1개), Job/CronJob(완료까지)*.
3. *Probe 잘 설계*, *PDB로 동시 죽일 한도 강제*, *ConfigMap 변경 시 checksum annotation*으로 자동 rollout.

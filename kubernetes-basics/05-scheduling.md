# 05 — Scheduling: Pod이 어느 노드에 들어가는가

> **공식 근거:** [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/), [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/), [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
>
> **선행:** [01-architecture.md](01-architecture.md) (scheduler 위치)

---

## 🎯 한 문장

> **Scheduler = *unscheduled Pod를 watch* → *filter (들어갈 수 있는 노드)* → *score (가장 좋은 노드)* → *binding API로 Pod에 노드 박음*. 사용자는 *nodeSelector·affinity·taint/toleration·topology spread*로 의도 표현.**

---

## 1. Scheduler의 2단계

### 단계 1: Filter (Predicate)
*"이 Pod이 들어갈 수 *없는* 노드 제외"*

체크 항목:
- 자원 부족 (`NodeResourcesFit`) — CPU/Memory/PIDs/storage requests 충족?
- nodeSelector 매칭
- node affinity *required* 조건 매칭
- taint를 *tolerate* 하나
- volume topology (EBS는 같은 AZ만 등)
- Pod affinity *required* 매칭

→ Filter 끝나면 *통과한 노드 후보 list*.

### 단계 2: Score (Priority)
*"통과한 노드 중 *가장 좋은* 곳"*

각 노드에 0~100 점수. 합산해서 최고점:
- *자원 여유* (덜 빡빡한 노드 선호)
- node affinity *preferred* 조건 일치 여부
- Pod affinity preferred 일치
- *이미지 locality* (해당 노드에 이미지가 이미 있으면 +점)
- topology spread 균등도

최고점 1개 → *Binding API* 호출 → Pod의 `spec.nodeName` 박힘 → 해당 노드의 kubelet이 *watch로 감지* → 실행.

---

## 2. nodeSelector — *가장 단순한 배치 룰*

```yaml
spec:
  nodeSelector:
    disktype: ssd
    zone: us-west-2a
```
- *모든 라벨 매칭 필수* (AND)
- *strict* — 매칭 노드 없으면 Pod *Pending*
- 더 풍부한 표현 필요하면 → *affinity*

---

## 3. Node Affinity — *유연한 배치 룰*

### required vs preferred
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 필수
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values: [nvidia, amd]
      preferredDuringSchedulingIgnoredDuringExecution:  # 선호
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-west-2a]
```

| 종류 | 의미 |
|---|---|
| `requiredDuringScheduling...` | *필수* — 안 맞으면 Pending |
| `preferredDuringScheduling...` | *선호* — weight 합산 score에 반영 |
| `...IgnoredDuringExecution` | *Pod 떠 있는 동안 노드 라벨 바뀌어도 evict 안 됨* |

연산자: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

### `RequiredDuringExecution`은 *아직 없음*
- "노드 라벨 바뀌면 evict" 는 *Beta·실험적*. 운영에선 *Ignored*만 사실상 사용.

---

## 4. Pod Affinity / Anti-affinity — *다른 Pod 위치 기반*

```yaml
# anti-affinity: 같은 노드에 같은 앱 다른 Pod이 있으면 *피함*
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels: { app: web }
      topologyKey: kubernetes.io/hostname    # 노드 단위
```

`topologyKey`가 핵심 — *어느 도메인 단위로 묶는가*:
- `kubernetes.io/hostname` — 노드 단위
- `topology.kubernetes.io/zone` — AZ 단위
- `topology.kubernetes.io/region` — 리전 단위

→ 흔한 사용:
- *DB replicas*는 *서로 다른 노드/AZ*에 (HA)
- *cache + 그 cache 쓰는 app*은 *같은 노드*에 (locality)

### *비용*
- Pod affinity는 *계산 복잡도 ↑* — scheduler가 *기존 Pod들과 비교*
- 대규모 클러스터에선 *scheduling latency 증가* 가능
- → *반드시 필요한 경우만* `required` 사용. 가능하면 `preferred`.

---

## 5. Taint / Toleration — *노드가 Pod을 *거부*하는 방식*

### 발상
- *nodeSelector·affinity*는 *Pod이 노드를 고름*
- *Taint*는 반대 — *노드가 "이런 Pod만 허용"*

### Taint 종류
```bash
kubectl taint nodes gpu-node1 dedicated=gpu:NoSchedule
# key=value:effect
```

| Effect | 의미 |
|---|---|
| `NoSchedule` | toleration 없으면 *새로 안 들어옴* (기존 Pod은 둠) |
| `PreferNoSchedule` | 가급적 안 들어옴 (선호) |
| `NoExecute` | toleration 없으면 *기존 Pod도 evict* |

### Toleration
Pod 쪽에 명시:
```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```
→ 이 toleration 있는 Pod만 그 taint 노드에 *들어갈 수 있음*. *반드시 들어간다는 보장 X* (scheduler가 다른 노드도 가능하면 그쪽 갈 수 있음). 같이 보내려면 *nodeSelector·affinity와 조합*.

### 흔한 활용
- **전용 노드** — GPU·고메모리 노드를 *특정 워크로드에만*
- **Master 노드 보호** — 기본 taint `node-role.kubernetes.io/control-plane:NoSchedule`
- **Node maintenance** — `kubectl drain` 이 `NoExecute` taint로 *Pod evict*

### 시스템 taint
- `node.kubernetes.io/not-ready` — 노드 NotReady 시 자동 taint
- `node.kubernetes.io/unreachable` — 노드 unreachable
- *기본 toleration 5분* — 5분 후 evict (수정 가능)

---

## 6. Topology Spread — *AZ/노드별 균등 분포*

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: web }
```
의미: *각 AZ별 web Pod 수 차이가 1을 넘으면 안 됨*. 어기면 *Pending* (또는 `ScheduleAnyway`).

→ 모든 zone에 *고르게 분포* → 한 AZ 장애에 대비.

Pod anti-affinity와 비슷하나 *더 정확한 표현* (균등 분포 vs 단순 "다른 곳").

---

## 7. Priority & Preemption — *중요한 Pod 우선*

### PriorityClass
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Mission critical"
```

Pod 쪽:
```yaml
spec:
  priorityClassName: high-priority
```

### Preemption
- *고우선 Pod이 Pending* (자원 부족) → scheduler가 *저우선 Pod 죽여서 자리 확보*
- 죽이는 순서: 우선순위 낮은 것부터, *PDB 고려*
- 시스템 Pod의 PriorityClass: `system-cluster-critical`, `system-node-critical` — *절대 죽지 말아야* 하는 Pod

### 흔한 활용
- *결제 Pod*는 *우선순위 높게* — 자원 경합 시 *다른 거 죽여서라도 살림*
- *batch Job*은 *낮게* — 자원 부족하면 *기다리거나 evict*

---

## 8. Cluster Autoscaler / Karpenter

### 자원 부족 → 노드 추가
- 단순 Pending Pod 검출 → *클라우드 API로 노드 추가 요청*
- 추가된 노드 join → scheduler가 Pending Pod 배치

### Cluster Autoscaler (CA)
- *고정된 노드 그룹* (ASG / MIG)에서 노드 수 조정
- *그룹 안에서 instance type 통일*

### Karpenter (AWS)
- *유연한 노드 provisioning* — Pod 요구에 맞춰 *동적으로 instance type 선택*
- *consolidation* — 비효율적 배치 발견 시 *기존 노드 정리·재배치*
- 더 빠름, 더 효율적
- *AWS 종속* (현재). 다른 클라우드 버전은 발전 중

---

## 9. *Descheduler* — *이미 배치된 Pod을 재배치*

- Scheduler는 *처음 배치* 만. 한 번 들어가면 *그대로*.
- 시간 지나면 *불균형* 발생 (노드 추가/제거, Pod churn 등)
- Descheduler — 주기적으로 *재배치 후보 찾고 evict* → scheduler가 다시 *더 좋은 노드*에 배치
- 운영 모범: cluster-autoscaler / Karpenter와 함께 *주기적*으로

---

## 10. 자주 헷갈리는 것

### 10-1. *Pod이 Pending* — 가장 흔한 이유 순서
1. *자원 부족* (`kubectl describe pod` → `Insufficient cpu/memory`)
2. *nodeSelector·affinity 매칭 안 됨*
3. *모든 노드에 toleration 안 맞는 taint*
4. *PVC bind 안 됨* (StorageClass 문제, AZ 불일치)
5. *image pull secret 없음* (private registry)

### 10-2. *requests vs limits 의미*
- `requests` — *scheduler가 보고 자원 예약*. 노드의 *requests 합 ≤ allocatable*.
- `limits` — *실행 시 cgroup 한도*. 초과 시 *throttle (CPU) 또는 OOMKill (Memory)*.
- **scheduler는 requests만 봄.** limits는 *런타임 강제*.

### 10-3. *모든 Pod에 requests 설정 안 함* → *대형 사고 가능성*
- requests 없는 Pod = scheduler가 *최소만 예약* → 노드 *오버커밋*
- 부하 폭증 시 *OOM 폭풍·노드 다운*
- → *반드시 requests·limits 설정*. LimitRange로 *기본값 강제* 가능.

### 10-4. *Pod affinity 너무 빡세면* — 모든 Pod이 한 노드에 몰림
- *PodAffinity가 너무 강하면* 모든 Pod이 *한 노드 폭주*
- → AntiAffinity와 *조합*해서 *분산도 보장*

### 10-5. *Karpenter가 노드 못 만듦* — IAM·SG·subnet 확인
- Karpenter는 *클라우드 API 호출* — 권한 부족 시 fail
- *EC2 instance type quotas* 도 자주 부딪힘
- `kubectl get events -n karpenter` 로 fail 이유

---

## 🎤 면접 빈출 Q&A

### Q1. K8s scheduler는 어떻게 Pod의 노드를 결정하나요?
> 2단계. **Filter**: 이 Pod이 들어갈 *없는* 노드 제외 (자원·nodeSelector·affinity required·taint·volume topology). **Score**: 남은 노드에 점수 (자원 여유·affinity preferred·image locality·topology spread). 최고점 노드 → Binding API → Pod에 nodeName 박힘 → kubelet이 watch로 감지.

### Q2. nodeSelector vs nodeAffinity vs taint/toleration 차이?
> **nodeSelector**: 단순 라벨 매칭, AND, strict. **nodeAffinity**: 풍부한 표현 (`In/NotIn/Exists`), `required`/`preferred` 분리. **taint/toleration**: *반대 방향* — 노드가 거부, toleration 있어야 들어옴. → 일반: nodeAffinity. *전용 노드*는 taint + toleration + nodeAffinity 조합.

### Q3. *모든 Pod에 requests/limits를 명시*해야 하는 이유?
> *scheduler는 requests만 봄*. requests 없으면 *최소만 예약* → 노드 *오버커밋* → *부하 폭증 시 OOM·노드 다운*. limits는 *cgroup 강제* — 초과 시 CPU throttle·메모리 OOMKill. 운영에선 *LimitRange로 namespace 기본값 강제* + *ResourceQuota로 총량 제한*.

### Q4. *Pod이 Pending* 일 때 디버깅 순서?
> `kubectl describe pod` → *Events 섹션*. 가장 흔한 원인: (1) 자원 부족 (`Insufficient cpu/memory`), (2) nodeSelector 매칭 X, (3) toleration 부족, (4) PVC bind 안 됨 (특히 EBS *AZ 불일치*), (5) imagePullSecret 누락. `kubectl get nodes -o wide` 로 *노드 자원 여유* 확인.

### Q5. Topology Spread Constraints가 왜 필요한가요? Anti-Affinity와 차이?
> *AZ별 균등 분포 보장*. Anti-Affinity는 "다른 곳에 있어라" *정성적*. Topology Spread는 *maxSkew*로 *정량적* — "각 zone별 Pod 수 차이가 1 이하여야". *Anti-Affinity는 1개씩만 보장* 가능 (3 AZ에 4 Pod → 어디 1개 더 갈지 모름). Topology Spread는 *정확한 분포 제어*.

### Q6. PriorityClass와 Preemption은 어떻게 작동?
> PriorityClass = *Pod의 우선순위 정의* (value 정수). 고우선 Pod이 Pending인데 자원 부족 → scheduler가 *저우선 Pod evict해서 자리 확보* (preemption). 죽이는 순서: 우선순위 낮은 것부터, *PDB 고려*. *시스템 critical Pod*은 `system-node-critical` 등 *절대 죽지 말아야* 할 클래스 사용.

### Q7. Karpenter와 Cluster Autoscaler 차이?
> Cluster Autoscaler = *고정 노드 그룹 (ASG)*에서 노드 수만 조정. Karpenter = *동적으로 instance type 선택*해서 노드 provisioning. Karpenter는 *Pending Pod 요구에 정확히 맞는* 노드를 *빠르게* 띄움. *consolidation*도 가능 (비효율 배치 발견 시 재배치). 단점: *AWS 종속* (현재).

---

## 🔗 Cross-reference

- **Pod 생성 흐름·kubelet 위치** → [01-architecture.md](01-architecture.md)
- **PV의 AZ 묶임 (volume topology)** → [04-config-and-storage.md](04-config-and-storage.md)
- **PodDisruptionBudget** (preemption·evict 시 고려) → [02-workloads.md](02-workloads.md)
- **AWS instance types·spot** → `aws-basics/` (예정)
- **FinOps — bin packing·spot strategy** → `aws-basics/` (예정)

---

## 📝 3줄 요약

1. Scheduler 2단계 — *Filter (들어갈 수 있는 노드) + Score (가장 좋은 노드)*. 결과는 binding API로 Pod에 nodeName 박힘.
2. 의도 표현 4가지 — *nodeSelector (단순), nodeAffinity (풍부), pod affinity/anti (다른 Pod 기반), taint/toleration (노드가 거부)* + *topology spread (균등 분포) + priority (우선순위)*.
3. *requests/limits 필수 설정* — 안 하면 노드 오버커밋·OOM 폭풍. *Pending 디버깅은 describe Events*부터.

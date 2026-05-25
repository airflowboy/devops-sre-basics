# Kubernetes — 동작 원리 중심

> *"K8s가 Pod를 어떻게 띄우는지 처음부터 설명해주세요"* 면접 빈출. *"kubectl apply 누른 순간부터 Pod이 Ready 되기까지"* 의 데이터 흐름을 *컴포넌트 단위로* 설명할 수 있어야.

---

## 🎯 한 문장 큰 그림

> **K8s = *선언적 상태(YAML)를 etcd에 저장* → *controller들이 *현재 상태 vs 원하는 상태*를 비교하며 *영원히 reconcile*하는 시스템.**

**핵심 키워드:**
- **선언적** — *"이렇게 되어야 한다"* (절차 X, 결과 명세 O)
- **Reconciliation loop** — controller가 *영원히 desired vs actual* 비교해서 메우는 동작
- **etcd = 단일 진실 공급원** — 모든 상태가 거기 한 곳에
- **kube-apiserver = 유일한 출입구** — 모든 컴포넌트가 API server 거쳐서만 etcd 접근

→ docker가 *컨테이너 하나*의 추상화면, K8s는 *컨테이너 떼*의 추상화.

---

## 🧩 K8s의 7가지 핵심 컴포넌트 (한 페이지 요약)

```
   ┌────────────────────────────────────────────────────┐
   │              Control Plane (보통 3+ 노드, HA)         │
   │                                                    │
   │  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
   │  │kube-apiserver│  │   etcd     │  │  scheduler   │  │
   │  │(REST 출입구) │  │ (KV 저장소) │  │ (Pod ↔ Node) │  │
   │  └────────────┘  └────────────┘  └──────────────┘  │
   │  ┌────────────────────────────────┐                │
   │  │   controller-manager           │                │
   │  │  (Deployment/RS/Endpoint/...)  │                │
   │  └────────────────────────────────┘                │
   └────────────────────────────────────────────────────┘
                       │ (kubelet ↔ apiserver)
   ┌───────────────────┼───────────────────┐
   ▼                   ▼                   ▼
┌──────────┐       ┌──────────┐       ┌──────────┐
│ Node 1   │       │ Node 2   │       │ Node 3   │
│ kubelet  │       │ kubelet  │       │ kubelet  │
│ kube-proxy│      │ kube-proxy│      │ kube-proxy│
│ runtime  │       │ runtime  │       │ runtime  │  ← CRI (containerd 등)
│ ┌──┐ ┌──┐│       │ ┌──┐ ┌──┐│       │ ┌──┐    │
│ │P1│ │P2││       │ │P3│ │P4││       │ │P5│    │  ← Pod = container 묶음
│ └──┘ └──┘│       │ └──┘ └──┘│       │ └──┘    │
└──────────┘       └──────────┘       └──────────┘
        Data Plane (워크로드가 실제로 도는 곳)
```

| 컴포넌트 | 역할 | 한 줄 |
|---|---|---|
| **kube-apiserver** | 모든 통신의 *유일한 출입구* (REST). 인증·인가·admission 처리 | *길*. 모든 길은 여기로. |
| **etcd** | 분산 KV 저장소. *모든 상태*가 여기 | *기억*. K8s의 single source of truth. |
| **scheduler** | 새로 만들어진 Pod의 *어느 노드에 둘지* 결정 | *부동산 중개*. Pod에 노드 배정. |
| **controller-manager** | Deployment·ReplicaSet·Job 등 *내장 controller들의 묶음* | *영원한 정리꾼*. 원하는 상태 강제. |
| **kubelet** | 각 노드의 *에이전트*. Pod spec 받아서 *컨테이너 실행 + 상태 보고* | *현장 작업자*. |
| **kube-proxy** | 각 노드의 *Service → Pod 라우팅* (iptables/IPVS/eBPF) | *교환원*. |
| **container runtime** | 실제 컨테이너 실행 (containerd / CRI-O). CRI로 kubelet과 통신 | *손*. |

상세는 [`01-architecture.md`](01-architecture.md).

---

## 🌪 자주 헷갈리는 7가지

### 1. *"K8s는 컨테이너를 직접 다룬다"* — 아니다
- K8s가 다루는 *가장 작은 단위는 Pod* (컨테이너 1+개의 묶음)
- 같은 Pod의 컨테이너들은 *NET·IPC·UTS namespace 공유* → `localhost`로 통신
- 컨테이너 자체는 *CRI를 통해 런타임(containerd)에 위임*
- → K8s는 *컨테이너의 상위 추상화*, 직접 실행은 런타임이 함

→ 이게 [`docker-basics/02-namespace-and-cgroup.md`](../docker-basics/02-namespace-and-cgroup.md) 의 *Pod = namespace 공유* 와 같은 원리.

### 2. *Service IP* 는 *진짜 IP가 아니다*
- ClusterIP = *가상 IP* (kube-proxy가 만들어내는 환상)
- 호스트의 라우팅 테이블에도 없음, 어떤 인터페이스에도 안 바인딩됨
- 패킷이 ClusterIP로 가면 *kube-proxy의 iptables/IPVS 룰*이 *진짜 Pod IP로 DNAT*
- → ping은 안 됨 (가상 IP라), curl은 됨 (TCP라 iptables가 잡음)

→ 이건 [`docker-basics/03-networking.md`](../docker-basics/03-networking.md) 의 *iptables DNAT* 와 같은 원리. K8s가 *클러스터 전체에 같은 트릭*을 적용한 것.

### 3. *Pod IP* 는 *재시작 시 바뀐다*
- Pod 죽고 새로 생기면 *다른 IP*
- → 직접 의존 X. 항상 *Service*로 추상화.
- StatefulSet도 *Pod 이름은 유지*되지만 *IP는 바뀜* (DNS 헤드리스 서비스 활용)

### 4. *kubectl apply* 는 *서버에서 일어나는 일이 거의 다*
- kubectl은 *YAML을 HTTP API에 전송*만 함 — 매우 dumb한 클라이언트
- 진짜 일은 *API server → etcd → controllers → scheduler → kubelet* 흐름에서
- kubectl 종료해도 *진행 중인 reconciliation은 계속*

### 5. *Deployment* 는 Pod을 *직접 만들지 않는다*
- Deployment → *ReplicaSet 생성* → ReplicaSet이 *Pod 생성*
- Rolling update = *새 RS 만들고 점진 전환*, 옛 RS 줄임
- 히스토리 보존도 *과거 RS들이 남아있어서* 가능 (`kubectl rollout undo`)

### 6. *모든 게 etcd 거친다* — 통신은 *직접 X*
- kubelet ↛ scheduler 직접 통신 ❌
- 모든 컴포넌트가 *API server의 Watch API*로 *원하는 자원 변화 구독*
- API server는 *etcd의 변화를 watch*해서 *구독자에게 push*
- → 분산 시스템 표준 패턴 (event-driven)

### 7. *Operator* 는 *내가 만든 controller*
- K8s 내장 controller가 Deployment·StatefulSet 등 *기본 자원*을 reconcile
- *내 도메인 자원*(예: `PostgresCluster`) 도 같은 방식으로 만들고 싶다 → CRD 정의 + 내가 만든 controller (= Operator)
- 결국 *K8s는 거대한 controller framework*

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 질문 |
|---|---|
| [01-architecture.md](01-architecture.md) | "kubectl apply 누른 순간부터 Pod이 Ready 되기까지 무슨 일이 일어나는가" |
| [02-workloads.md](02-workloads.md) | "Pod, Deployment, StatefulSet, DaemonSet, Job 차이는?" |
| [03-networking.md](03-networking.md) | "Service IP는 어떻게 작동하나? CNI·kube-proxy·CoreDNS의 역할" |
| [04-config-and-storage.md](04-config-and-storage.md) | "ConfigMap/Secret, PV/PVC/StorageClass/CSI의 분리 이유" |
| [05-scheduling.md](05-scheduling.md) | "Pod이 *어느 노드에* 들어갈지 어떻게 결정되나" |
| [06-api-and-controllers.md](06-api-and-controllers.md) | "Reconciliation loop 동작, CRD/Operator의 의미" |
| [07-rbac-and-security.md](07-rbac-and-security.md) | "ServiceAccount/Role/RoleBinding, Pod Security 입문" |

---

## 🔗 Cross-reference

- **Pod = namespace 공유 컨테이너 묶음** ([`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md))
- **CNI (Calico/Flannel/Cilium)** = `docker-basics/03` 의 veth/bridge/iptables의 *K8s 차원 일반화* → [03 networking](03-networking.md), [`network-basics/06`](../network-basics/06-kubernetes.md)
- **CSI** = `docker-basics/04` 의 volume driver의 *K8s 표준 인터페이스* → [04 config-and-storage](04-config-and-storage.md)
- **kube-proxy iptables 모드** = `docker-basics/03` 의 DNAT의 *Service 차원 일반화* → [03](03-networking.md)
- **RBAC** = aws IAM과 *같은 원리, K8s 안에서 자체 구현* → [07](07-rbac-and-security.md), `aws-basics/` (예정)
- **Pod Security Standards** = `docker-basics/06` 의 보안 권장사항을 *K8s가 선언적으로 강제* → [07](07-rbac-and-security.md), `k8s-security-basics/` (예정)
- **선언적 reconciliation** 패턴 = ArgoCD·Terraform과 *같은 원리* → [06](06-api-and-controllers.md), `cicd-gitops-basics/` (예정), `terraform-basics/` (예정)

---

## 🎤 면접 빈출 (전체 폴더 횡단)

| 질문 | 답이 있는 파일 |
|---|---|
| kubectl apply 후 무슨 일이 일어나나요? | [01](01-architecture.md) |
| Deployment / StatefulSet / DaemonSet 차이? | [02](02-workloads.md) |
| Service의 ClusterIP/NodePort/LoadBalancer 차이? | [03](03-networking.md) |
| Pod 간 통신은 어떻게? | [03](03-networking.md) |
| ConfigMap과 Secret 차이? Secret은 안전한가요? | [04](04-config-and-storage.md) |
| Pod이 어느 노드에 들어갈지 어떻게 결정? | [05](05-scheduling.md) |
| Controller pattern·Operator·CRD 이해? | [06](06-api-and-controllers.md) |
| RBAC: ServiceAccount, Role, ClusterRole 차이? | [07](07-rbac-and-security.md) |
| Pod이 갑자기 Restart Loop. 디버깅 순서? | [01](01-architecture.md) + [02](02-workloads.md) |
| etcd 백업 / 복구 절차 아시나요? | [01](01-architecture.md) |

---

## 📐 학습 순서 권장

1. **01 architecture** — 큰 그림. 모든 토픽의 베이스.
2. **02 workloads** — *내가 띄우는 것들*.
3. **03 networking** — *그것들이 어떻게 통신하나*.
4. **04 config-and-storage** — *그것들이 어떤 데이터·설정과 함께 사나*.
5. **05 scheduling** — *어디에 배치되나*.
6. **06 api-and-controllers** — *왜 이 모든 게 일관되게 작동하나* (메타 개념).
7. **07 rbac-and-security** — *누가 무엇을 할 수 있나*.

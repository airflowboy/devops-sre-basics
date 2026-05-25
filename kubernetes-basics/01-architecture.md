# 01 — Architecture: kubectl apply 후 Pod이 Ready되기까지

> **공식 근거:** [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/), [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/), [kubelet docs](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
>
> **이 파일이 K8s 전체의 베이스.** 다른 모든 파일이 이걸 가정한다.

---

## 🎯 한 문장

> **K8s는 *모든 통신이 kube-apiserver를 거치고*, *모든 상태가 etcd에 저장되며*, *각 컴포넌트는 자기 관심사만 watch해서 reconcile*하는 *event-driven 분산 시스템*이다.**

---

## 1. Control Plane vs Data Plane

```
┌─────────────────────────────────────────────────────────┐
│  Control Plane  (보통 3+ 노드 HA, 매니지드면 클라우드 운영)│
│                                                          │
│  ├─ kube-apiserver  (전 유저·전 컴포넌트의 출입구)        │
│  ├─ etcd            (모든 상태)                          │
│  ├─ scheduler       (Pod → Node 배정)                    │
│  ├─ controller-manager (built-in controller 묶음)        │
│  └─ cloud-controller-manager (클라우드 통합 — LB, Volume) │
└─────────────────────────────────────────────────────────┘
                              ▲
                              │ (kubelet ↔ apiserver)
                              ▼
┌─────────────────────────────────────────────────────────┐
│  Data Plane  (= 워커 노드들)                              │
│                                                          │
│  각 노드:                                                 │
│  ├─ kubelet       (노드의 K8s 에이전트)                   │
│  ├─ kube-proxy    (Service 라우팅)                       │
│  └─ container runtime (containerd / CRI-O — CRI로 통신)  │
│                                                          │
│  └─ Pods (워크로드 = 실제 컨테이너들)                     │
└─────────────────────────────────────────────────────────┘
```

매니지드 K8s (EKS, GKE, AKS) 에서는 **Control Plane을 클라우드 사업자가 운영** → 사용자는 Data Plane (워커 노드)만 신경. self-hosted (kubeadm)에서는 *둘 다 직접 운영*.

---

## 2. 핵심 컴포넌트 자세히

### 2-1. kube-apiserver — *유일한 출입구*
- HTTP/2 + gRPC REST API 서버
- *모든 요청은 여기로* (kubectl, controllers, kubelets, dashboards 다)
- 처리 순서:
  ```
  요청 → 인증(Authentication) → 인가(Authorization, RBAC) → Admission Controller → etcd 저장/조회
  ```
- **stateless** — 여러 인스턴스 실행 가능. 상태는 etcd에.
- LoadBalancer 뒤에 여러 개 띄우는 게 HA 패턴.

### 2-2. etcd — *분산 KV 저장소*
- *모든 K8s 자원의 진실*이 여기 (Pod·Deployment·Secret·...)
- *strong consistency* (Raft 합의 알고리즘) — 분산 노드 간 *항상 같은 값* 보장
- 노드 수는 *홀수* (3·5·7) — quorum 위해
- 손실 시 *클러스터 사실상 잃음* → **백업 필수**:
  ```bash
  ETCDCTL_API=3 etcdctl snapshot save backup.db \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  ```
- 매니지드 K8s는 *사업자가 알아서 백업*. self-hosted는 *내 책임*.

### 2-3. scheduler — *Pod → Node 배정*
- 새 Pod 발견 (status.nodeName이 비어있는 Pod) → *어느 노드에 둘지 결정* → API server에 *바인딩 요청*
- 결정 단계:
  1. **Filter** — Pod가 *들어갈 수 없는 노드 제거* (자원 부족, taint, nodeSelector 안 맞음 등)
  2. **Score** — 남은 노드들에 점수 매김 (자원 여유, affinity, locality 등)
  3. **Bind** — 최고점 노드에 바인딩
- 스케줄러는 *Pod 실제 실행은 안 함*. *어디로 갈지만 결정.* 실행은 kubelet이.
- 자세히 [`05-scheduling.md`](05-scheduling.md).

### 2-4. controller-manager — *built-in controller들의 묶음*
- *수십 개 controller가 한 프로세스에서* 동작
  - Deployment controller — Deployment 변경 → 새 ReplicaSet 만들거나 RS 개수 조정
  - ReplicaSet controller — RS.spec.replicas 만큼 Pod 유지
  - Node controller — 노드 health 모니터링, NotReady 시 Pod evict
  - Endpoint controller — Service ↔ Pod IP 매핑 자동 갱신
  - Job controller, CronJob controller, Namespace controller, ...
- 각 controller가 *각자의 자원 변화를 watch* + *reconcile*

### 2-5. cloud-controller-manager — *클라우드 통합*
- Cloud-provider별 controller들 (CCM)
- Node controller — 노드의 *cloud meta* 동기화 (가용존, 인스턴스 타입)
- Route controller — VPC 라우팅 자동 (일부 CNI)
- Service controller — `type: LoadBalancer` Service → *클라우드 LB 자동 생성·연결*
- (EKS면 AWS Load Balancer Controller가 이걸 보강)

### 2-6. kubelet — *각 노드의 에이전트*
- API server의 *자기 노드용 Pod spec*을 watch
- 새 Pod 도착 → *CRI를 통해 컨테이너 런타임에 명령*
  - 이미지 pull
  - Pod sandbox(=pause container) 생성 → namespace 셋업
  - 그 sandbox 안에 컨테이너들 실행
- 정기적으로 *node status + pod status*를 API server에 보고
- liveness/readiness probe 실행
- volume mount, secret 주입

### 2-7. kube-proxy — *Service 라우팅*
- API server의 *Service + Endpoint 변화*를 watch
- 각 노드의 *iptables*(또는 IPVS, eBPF)에 룰 박음
- Pod의 ClusterIP 접근 → kube-proxy가 박은 룰이 *실제 Pod IP로 DNAT*
- 자세히 [`03-networking.md`](03-networking.md).

### 2-8. Container Runtime — *실제 컨테이너 실행*
- **CRI (Container Runtime Interface)** 표준으로 kubelet과 통신
- 표준 런타임:
  - **containerd** (현재 표준, CNCF graduated)
  - **CRI-O** (Red Hat 진영, K8s 전용)
- 구버전 *dockershim* (K8s가 dockerd를 거치는 shim) — *K8s 1.24부터 제거*
- containerd가 결국 *runc*를 호출해 컨테이너 실행 ([`docker-basics/01`](../docker-basics/01-image-and-layers.md))

---

## 3. `kubectl apply -f deploy.yaml` — *전 과정 추적*

가장 자주 묻는 면접 질문. 단계별로:

### Step 1. kubectl이 YAML 읽음
- 클라이언트 측에서 *YAML → JSON 변환*
- *kubeconfig*에서 클러스터 endpoint + 인증 정보 읽음
- *HTTPS POST*로 API server에 전송:
  ```
  POST /apis/apps/v1/namespaces/default/deployments
  {
    "kind": "Deployment",
    "spec": { "replicas": 3, "template": {...} }
  }
  ```

### Step 2. API server에서 인증·인가·admission
1. **Authentication** — 누구야? (mTLS·token·OIDC)
2. **Authorization** — 너 이거 할 권한 있어? (RBAC)
3. **Mutating Admission Webhooks** — 자원을 *변형* (예: sidecar 자동 주입, default value 채움)
4. **Validation** — schema·CRD 검증
5. **Validating Admission Webhooks** — 자원이 *정책 위반인지* (예: 보안 정책)
6. **etcd에 저장**

→ 6번까지 통과해야 *etcd에 기록됨*. 한 단계라도 fail이면 *에러 반환*, 자원 안 만들어짐.

### Step 3. Deployment controller가 Watch로 감지
- controller-manager의 *Deployment controller*가 API server에 *Watch 요청* 걸어둠
- 새 Deployment 등장 → Notify 받음
- 비교: 이 Deployment의 *replicas: 3* 만큼 *ReplicaSet*이 있나?
- 없음 → ReplicaSet 자원 *생성* (다시 API server → etcd로)

### Step 4. ReplicaSet controller가 감지
- 새 ReplicaSet 등장 → Notify
- 비교: 이 RS의 *spec.replicas: 3* 만큼 *Pod*가 있나?
- 없음 → Pod 3개 *생성* (다시 API server → etcd로). 이 시점 Pod.status.nodeName 비어 있음.

### Step 5. Scheduler가 감지
- *unscheduled Pod* watch 중
- 새 Pod 3개 발견 → 각각 *filter + score* → *node 결정* → *binding API* 호출:
  ```
  POST /api/v1/namespaces/default/pods/{name}/binding
  { "target": { "kind": "Node", "name": "node-1" } }
  ```

### Step 6. 해당 노드의 kubelet이 감지
- kubelet이 *자기 nodeName의 Pod* watch 중
- 새 Pod 발견 (status.nodeName 자기 것) → 처리:
  1. *Sandbox(pause container) 생성* — namespace 셋업
  2. 이미지 *CRI에 pull 요청*
  3. *볼륨 마운트* (PV/ConfigMap/Secret)
  4. *init container 순서대로 실행 후 종료*
  5. *주 컨테이너들 실행*
  6. *probe 시작* (readiness/liveness)
- 각 단계 결과를 *API server에 status update*

### Step 7. Endpoint controller가 감지 (Service가 있다면)
- Service의 selector에 매칭되는 Pod이 *Ready* 됨 → Endpoint(또는 EndpointSlice)에 *Pod IP 추가*
- kube-proxy가 *Endpoint 변화 watch* → iptables 룰 *갱신*
- 이제 Service IP로 들어오는 트래픽이 *이 Pod까지 도달*

### Step 8. (Service type=LoadBalancer면) cloud-controller-manager가
- 클라우드 API 호출해서 *실제 LB 생성*, ALB/NLB Listener·Target Group 셋업
- LB DNS를 Service.status.loadBalancer.ingress에 기록

→ **이 8단계를 면접에서 *순서대로* 설명할 수 있으면 architecture 마스터.**

---

## 4. *모든 통신이 API server 거치는 이유*

### 장점
- *단일 인증·인가 지점* — 보안 강함
- *단일 audit 로그* — *누가 무엇을 했는지* 한 곳에
- *etcd 추상화* — controller가 etcd 직접 안 봄. API server가 추상화 (CRUD + Watch).
- *버전 호환* — apiserver가 *v1 vs v1beta1 변환* 처리

### 단점·고려
- API server *부하 집중*. 대규모 클러스터에선 *Watch 부하 폭증* 가능 → bookmarks·resource version 활용
- 모든 controller가 *비동기적으로 reconcile* → *eventually consistent*. *즉시 반영*을 기대하면 안 됨.

---

## 5. Watch API — *event-driven의 핵심*

K8s의 *모든 controller가 작동하는 메커니즘*.

```
client → GET /api/v1/pods?watch=true&resourceVersion=12345
       ← (stream)
         {type: ADDED,    object: pod1}
         {type: MODIFIED, object: pod2}
         {type: DELETED,  object: pod3}
         ...
```

- API server가 *etcd watch*를 통해 변화 감지
- 모든 watcher에 *push*
- 각 controller는 *자기 관심사*만 watch (예: ReplicaSet controller는 ReplicaSet + Pod만)
- 연결 끊김 → 마지막 resourceVersion부터 *재구독*

→ **Polling이 아니라 push**. 그래서 K8s가 *빠르게 변화에 반응*.

---

## 6. *Pod이 Restart Loop* — 디버깅 순서 (인터뷰 빈출)

흔한 시나리오: `kubectl get pods` → `mypod  0/1  CrashLoopBackOff  5  3m`

### 단계별 디버깅
1. **`kubectl describe pod mypod`** — Events 섹션 확인
   - `Failed to pull image` → image 이름·태그·registry 권한
   - `Insufficient cpu` → 자원 부족, scheduler 못 배치
   - `Liveness probe failed` → probe 설정 잘못됐거나 앱 진짜 안 응답
   - `OOMKilled` → 메모리 limit 부족 (cgroup이 죽임)
2. **`kubectl logs mypod`** — 현재 컨테이너 로그
3. **`kubectl logs mypod --previous`** — *직전 종료된* 컨테이너 로그 (CrashLoop에서 가장 유용)
4. **`kubectl get pod mypod -o yaml`** — 자원·노드·conditions 전체 상태
5. *Pod 자체는 떴는데 traffic 안 가는 경우*:
   - `kubectl get endpoints {service-name}` — 이 Pod IP 등록됐나? 안 됐으면 readiness probe 실패 가능성
6. *Image pull 실패*가 잦다면:
   - private registry credential (imagePullSecrets) 확인
   - K8s ServiceAccount에 secret 바인딩됐나
   - 노드에서 직접 `crictl pull {image}` 시도 (containerd 직접)

→ 면접 답: *"Events → logs → describe → endpoints 순서로 좁힌다"*.

---

## 7. 자주 헷갈리는 것

### 7-1. *kubelet은 API server 에이전트지 controller가 아니다*
- kubelet은 *자기 노드 Pod spec*만 받아 실행. *다른 노드 Pod 모름*.
- 노드 간 협상·재배치는 *controller-manager + scheduler*가.

### 7-2. *etcd에 직접 쓰면 안 된다*
- API server *우회* → admission·validation 안 거침. 데이터 망가짐.
- 항상 *API 통해서*.

### 7-3. *kubectl edit / kubectl patch* 도 결국 API 호출
- *etcd에 직접 박는 게 아님*. 다 API server 거침.
- *Mutating webhook이 변경*할 수도 있음 → `kubectl get -o yaml` 로 *실제 저장된 형태* 확인 필요.

### 7-4. *Deployment 변경 ≠ 즉시 새 Pod*
- Deployment update → *새 ReplicaSet 생성* → *기존 RS는 surge/unavailable 한도 내에서 축소* → *새 RS는 점진 증가*
- 한 번에 *모든 Pod이 바뀌지 않음*. RollingUpdate 전략 default.

### 7-5. *Pod 한 번 죽으면 같은 Pod이 다시 뜬다* — 아니다
- *Deployment·ReplicaSet 안에서면* RS controller가 *새 Pod 생성*. 같은 *이름·UID*가 아님.
- StatefulSet에선 *같은 이름의 새 Pod* (UID는 새 것).
- *naked Pod* (단독 Pod)은 *재생성 안 됨* (그래서 운영에선 거의 안 씀).

---

## 🎤 면접 빈출 Q&A

### Q1. kubectl apply 누른 순간부터 Pod이 Ready 되기까지 설명해주세요.
> (위 Section 3의 8단계 답)

### Q2. 모든 통신이 API server를 거치는 이유?
> *단일 인증·인가·admission 지점*으로 보안·정책 일관성. *etcd 추상화*로 controller가 etcd 직접 안 봄. *단일 audit 로그*로 누가 무엇을 했는지 추적. 단점은 *부하 집중*이라 대규모에선 *Watch 효율*이 중요.

### Q3. etcd가 죽으면 K8s는 어떻게 되나요?
> *모든 상태가 etcd에 있음*. etcd 손실 = *클러스터 상태 손실*. 이미 떠 있던 Pod은 *kubelet이 계속 운영*하므로 *당장 죽진 않음*, 그러나 *새 Deploy·변경 불가*, *Endpoint 갱신 X*, *scheduler 결정 X*. 복구는 *etcd snapshot에서 restore*. 그래서 *백업 필수*.

### Q4. dockershim이 deprecated 됐는데, 무엇이 바뀌나요?
> kubelet이 dockerd를 *shim 통해* 호출하는 구조 → containerd를 *CRI로 직접* 호출. *image 호환성은 그대로* (OCI 표준). 변화는 *내부 컴포넌트 단순화*. 사용자 입장에선 거의 차이 없음. K8s 1.24부터 dockershim 제거됨.

### Q5. Pod이 CrashLoopBackOff일 때 디버깅 순서?
> `describe pod` → *Events 섹션*에서 *왜 fail인지* (image pull / probe / OOM / 자원 부족). `logs --previous` 로 *직전 컨테이너 로그*. `get pod -o yaml` 로 *전체 상태*. *traffic 안 가는 경우*는 `get endpoints` 로 *readiness 통과 여부*. *image pull*은 *imagePullSecrets*·*노드 직접 pull* 시도.

### Q6. K8s의 *선언적 vs 명령적* 의 의미는?
> 선언적 = *원하는 상태를 명세* (`kubectl apply`, YAML). 명령적 = *어떻게 변경할지 명시* (`kubectl create deployment ...`). K8s의 *모든 컴포넌트는 reconciliation으로 동작* → 선언적이 *fit*. 명령적은 빠른 prototyping엔 좋으나 *재현성·GitOps에 안 맞음*.

### Q7. controller pattern을 직접 설명해주세요.
> *원하는 상태(spec)* 와 *현재 상태(status)* 를 *영원히 비교*하며 *현재를 spec에 맞추는 loop*. ReplicaSet controller: `spec.replicas: 3` 이면 *Pod 수를 3개로 맞춤* (모자라면 만들고, 남으면 죽임). 같은 패턴이 *모든 K8s 자원*에 적용. Custom Resource + 직접 만든 controller = *Operator*.

---

## 🔗 Cross-reference

- **워크로드 종류** → [02-workloads.md](02-workloads.md)
- **Service IP / kube-proxy / CoreDNS** → [03-networking.md](03-networking.md)
- **CRD / Operator / Admission** → [06-api-and-controllers.md](06-api-and-controllers.md)
- **RBAC (인증·인가)** → [07-rbac-and-security.md](07-rbac-and-security.md)
- **CRI / runc / containerd** → [`docker-basics/01`](../docker-basics/01-image-and-layers.md), [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)
- **Service Mesh** (Istio 등) — control plane 위에 *또 다른 control plane 얹기* → 예정

---

## 📝 3줄 요약

1. K8s = *kube-apiserver를 유일한 출입구로 두고*, *etcd에 모든 상태 저장*, *controller들이 watch + reconcile* 하는 event-driven 분산 시스템.
2. *kubectl apply* 의 8단계: 인증·인가·admission → etcd → Deployment ctrl → RS ctrl → Pod 생성 → scheduler 배정 → kubelet 실행 → Endpoint·LB.
3. 디버깅은 *Events → logs → describe → endpoints* 순서로 좁힌다. *직접 etcd 안 만지고, 항상 API 통해서*.

# 03 — Networking: Service IP는 어떻게 작동하는가

> **공식 근거:** [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/), [CNI spec](https://github.com/containernetworking/cni)
>
> **선행:** [`network-basics/`](../network-basics/) (IP·NAT·iptables), [`docker-basics/03`](../docker-basics/03-networking.md) (veth/bridge)

---

## 🎯 한 문장

> **K8s 네트워킹 = *모든 Pod이 라우팅 가능한 IP를 가진* *flat network* + *Service*라는 *가상 IP 추상화* + *kube-proxy*가 *iptables/IPVS로 가상 IP를 실제 Pod로 라우팅*하는 구조.**

---

## 1. K8s 네트워킹의 4가지 통신 모델

```
1. Container ↔ Container  (같은 Pod 안)        → localhost (NET namespace 공유)
2. Pod ↔ Pod              (같은/다른 노드)      → Pod IP 직접 (CNI가 라우팅 보장)
3. Pod ↔ Service          (Service IP로)        → kube-proxy가 DNAT
4. External ↔ Service     (외부 → Service)      → NodePort / LoadBalancer / Ingress
```

각각을 차례로.

---

## 2. Container ↔ Container (같은 Pod 안)

- Pod = *NET namespace를 공유*하는 컨테이너 묶음
- → 같은 Pod의 컨테이너는 *같은 IP, 같은 포트 공간*
- 통신: `localhost:포트`
- 포트 충돌: 같은 Pod에서 두 컨테이너가 *같은 포트* 사용 불가

→ [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 의 *Pod = namespace 공유 컨테이너 묶음* 의 응용.

---

## 3. Pod ↔ Pod — *CNI가 보장*

### K8s 네트워크 모델의 *기본 원칙*
1. *모든 Pod은 고유한 IP를 가짐* (NAT 없이)
2. *모든 Pod은 다른 Pod과 *직접* IP로 통신 가능* (같은 노드든 다른 노드든)
3. *각 노드의 agent (kubelet)도 그 Pod들과 통신 가능*

→ **NAT 없이 flat network처럼 동작**. 이 *원칙을 구현하는 책임*은 *CNI 플러그인*에 있음.

### CNI (Container Network Interface) 란?
- K8s가 *네트워크 구현을 외주*하는 인터페이스 표준
- K8s는 *플러그인 binary를 호출*하면서 *Pod 생성·삭제 이벤트* 전달
- 플러그인이 *네트워크 인터페이스 생성 + IP 할당 + 라우팅 셋업*

### 주요 CNI 플러그인

| 플러그인 | 라우팅 방식 | 비고 |
|---|---|---|
| **Flannel** | VXLAN (UDP overlay) 기본 | 가장 단순. 소규모 |
| **Calico** | *BGP 라우팅* (overlay 없음) 또는 *VXLAN/IPIP* | NetworkPolicy 강력. 가장 흔한 표준. |
| **Cilium** | *eBPF* — 커널 hook로 패킷 직접 처리 | 성능 최강, observability 강력. 모던 표준 후보. |
| **AWS VPC CNI** | *VPC subnet의 ENI/IP를 Pod에 직접* | EKS 기본. *VPC IP 소진 위험* 단점. |
| **Weave** | mesh + VXLAN | 작은 클러스터 |

### 핵심 비교: Overlay vs Native

**Overlay** (Flannel VXLAN, Calico IPIP)
- 패킷을 *UDP 안에 캡슐화*해서 다른 노드로
- 장점: *underlying network에 의존 X*. 어떤 환경에서도 동작.
- 단점: *캡슐화 비용* (MTU, CPU, 약간의 latency).

**Native routing** (Calico BGP, AWS VPC CNI)
- *underlying network*가 Pod IP를 *직접 라우팅 가능*
- 장점: 캡슐화 비용 X. 성능 좋음.
- 단점: underlying network 설정 필요 (BGP peering, VPC 라우팅).

→ EKS의 AWS VPC CNI는 *각 Pod이 VPC의 ENI IP를 직접 할당* → *Pod IP가 VPC 내에서 그대로 라우팅 가능*. 클라우드 native 환경의 장점.

---

## 4. Pod ↔ Service — *가상 IP의 마법*

### 문제
- Pod IP는 *재시작 시 바뀜*. 직접 의존 불가.
- 같은 역할의 Pod이 *여러 개* (load balancing 필요).
- → *안정적 endpoint*가 필요.

### 해결: Service
- Pod 집합 (selector로 매칭) 에 *고정된 IP + 이름* 부여
- 이름으로 lookup → 가상 IP → 실제 Pod 중 하나로 라우팅

### Service 타입 4종

| 타입 | 역할 | IP/엔드포인트 |
|---|---|---|
| **ClusterIP** (기본) | 클러스터 *내부*에서만 접근 | 가상 IP (ClusterIP) |
| **NodePort** | *모든 노드의 특정 포트* (30000~32767)로 접근 | 노드IP:포트 |
| **LoadBalancer** | *외부 LB* (클라우드 LB 자동 생성) → NodePort → Pod | LB의 외부 DNS/IP |
| **ExternalName** | *DNS CNAME alias* (Pod 없이 외부 DNS 매핑) | DNS |

### ClusterIP의 *실체*
- *진짜 인터페이스에 바인딩된 IP가 아님*. 어떤 호스트의 NIC에도 없음.
- *iptables (또는 IPVS) 룰의 trigger*. 패킷이 *그 IP로 가는 순간* kube-proxy 룰이 잡아 *진짜 Pod IP로 DNAT*.
- → *ping은 안 됨* (가상 IP라 ICMP 응답 줄 게 없음), *TCP/UDP만 작동*.

→ 이게 [`docker-basics/03`](../docker-basics/03-networking.md) 의 *iptables DNAT* 와 *같은 원리, 더 큰 스케일*.

### kube-proxy 모드 3종

| 모드 | 어떻게? | 비고 |
|---|---|---|
| **iptables** (기본) | iptables 룰 *Pod 수만큼 추가* | 단순. 대규모 (Pod 만 단위)면 룰 평가 비용 ↑ |
| **IPVS** | Linux 커널의 IPVS (전용 LB 모듈) | hash table 기반. 대규모에 효율적. |
| **eBPF** (Cilium 등) | eBPF 프로그램이 kube-proxy *대체* | 가장 모던. kube-proxy 자체를 제거 가능. |

### Endpoint vs EndpointSlice
- 옛날: 한 Service 당 *Endpoint 자원 1개*에 *모든 Pod IP 나열*
- 문제: Pod 수천 개면 *Endpoint 자원이 거대해짐* → 갱신 비용 폭증
- 해결: **EndpointSlice** — Pod IP를 *여러 작은 슬라이스*로 분할 (K8s 1.21+ 기본)

### DNS와 Service
- 클러스터 안 DNS는 **CoreDNS** (옛날엔 kube-dns)
- 모든 Pod의 `/etc/resolv.conf`가 *CoreDNS Service IP* 가리킴
- Service 이름 → *ClusterIP 반환*
  - `myservice` (같은 ns)
  - `myservice.ns2` (다른 ns)
  - `myservice.ns2.svc.cluster.local` (FQDN)
- *Headless Service*는 *Pod IP 목록 그대로* 반환 (StatefulSet 등)

---

## 5. Pod이 Service로 통신하는 *흐름 전체*

`curl http://myservice` (Pod 안에서) 실행 시:

```
1. Pod 안 DNS lookup: myservice → CoreDNS Service IP
2. CoreDNS Service IP로 DNS 요청 → 라우팅:
   ├─ CoreDNS도 결국 Service → kube-proxy iptables 룰 → 실제 CoreDNS Pod
3. CoreDNS가 `myservice` 검색 → ClusterIP 반환 (예: 10.96.10.1)
4. Pod이 10.96.10.1:80 으로 HTTP 요청
5. 호스트의 iptables 룰이 PREROUTING에서 잡음:
   "10.96.10.1:80 → DNAT to {랜덤 Pod IP}:8080"
6. 패킷이 그 Pod로 (같은 노드면 docker0 bridge 통해, 다른 노드면 CNI 라우팅)
7. 응답 패킷 → conntrack이 *원래 ClusterIP에서 온 것처럼* 보이게 SNAT → 요청 Pod에 도달
```

→ **이 흐름을 면접에서 그릴 수 있으면 네트워킹 마스터.**

---

## 6. External ↔ Service — *외부에서 들어오는 방법*

### Option 1. NodePort
- *모든 노드의 같은 포트* 열림 (예: 30080)
- 외부에서 `노드IP:30080` 접근 → kube-proxy → ClusterIP → Pod
- 단점: *포트 범위 제한* (30000~32767), *어느 노드 IP* 신경써야 함
- 보통 *개발용* 또는 *바로 위에 LB 두는 베이스*

### Option 2. LoadBalancer
- 클라우드의 *실제 LB 자동 생성* (cloud-controller-manager)
- AWS: ELB/NLB/ALB. GCP: GCLB. Azure: Azure LB.
- LB → NodePort → ClusterIP → Pod
- Service 당 LB 1개 → *비용·자원 누적*
- → *여러 서비스를 하나의 진입점으로* 묶으려면 **Ingress**

### Option 3. Ingress
- *L7 라우팅* (HTTP path / Host 기반)
- Ingress 자체는 *선언적 spec*. *실제로 트래픽 처리하는 건 Ingress Controller* (nginx-ingress, AWS Load Balancer Controller, Traefik 등)
- 패턴:
  ```
  Internet → ALB (1개) → Ingress Controller Pod
                              ├─ /api/* → api-service
                              ├─ /web/* → web-service
                              └─ Host: admin.* → admin-service
  ```
- 장점: *LB 1개로 여러 서비스* (비용 절감)
- 단점: HTTP/HTTPS 외 (raw TCP/UDP)는 다른 방법

### Option 4. Gateway API (차세대)
- Ingress의 *진화 버전*. *더 표현력 있고 *역할 분리* (Cluster Operator / App Owner / Developer)
- *role-based RBAC*에 더 친화적
- 점진 보급 중. 새 클러스터엔 *Gateway API 우선 검토* 권장.

---

## 7. NetworkPolicy — *Pod 간 통신 제어*

### 기본
- *기본은 모두 허용* (no policy = all open)
- NetworkPolicy 적용 시 → *해당 Pod에 들어오는 트래픽이 *whitelist*로 동작*

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-web
  namespace: default
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: { app: web }
    ports:
    - protocol: TCP
      port: 8080
```
의미: *api Pod*에 *web Pod에서 오는 TCP 8080*만 허용. 나머지 *전부 차단*.

### 구현
- *NetworkPolicy 자체는 spec*. *실제 시행은 CNI 플러그인 책임*.
- Calico·Cilium은 지원. Flannel 기본은 *지원 X* (Calico와 함께 써야).
- AWS VPC CNI는 *별도 add-on 필요* (Calico 또는 Cilium 추가).

### default-deny 패턴
```yaml
# 모든 ingress 차단부터 시작
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
```
그 다음 *명시적 allow* 정책 추가. *whitelist 방식*.

→ `k8s-security-basics/` (예정) 에서 더 깊이.

---

## 8. CoreDNS — *클러스터 DNS*

- *DNS plugin chain* 구조
- 기본 plugin들:
  - `kubernetes` — Service/Pod 이름 lookup
  - `forward` — 외부 DNS로 forward
  - `cache` — TTL 캐시
  - `prometheus` — 메트릭 노출
- Pod의 `/etc/resolv.conf`:
  ```
  nameserver 10.96.0.10              ← CoreDNS Service IP
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  ```
- *`ndots:5`*: dot이 5개 이하면 *search 도메인 순회*. 외부 도메인 lookup이 *느려질 수 있음* — `ndots:1` 또는 FQDN 사용으로 최적화.

---

## 9. *Service Mesh* — L7 레이어 위에 *또 다른 control plane*

- Istio·Linkerd 같은 게 *Sidecar로 Envoy proxy 주입* → 모든 Pod 간 트래픽이 *Envoy 거침*
- 기능:
  - mTLS 자동 (Pod 간 암호화)
  - L7 라우팅 (canary, A/B, traffic shifting)
  - 관측성 (자동 metric/tracing)
  - 정책 (retry, timeout, circuit breaker)
- 비용: *Pod마다 sidecar 1개 추가* (메모리·CPU)
- → 모든 클러스터에 *필요한 건 아님*. *마이크로서비스 수십 개 이상*일 때 ROI.

---

## 10. 자주 헷갈리는 것

### 10-1. *Service IP는 ping이 안 된다*
- 가상 IP이므로 ICMP 응답할 *프로세스가 없음*
- TCP/UDP는 iptables 룰이 잡아 DNAT
- → "Service IP에 ping 안 되는데 통신은 되는" 정상

### 10-2. *Pod이 외부로 나갈 때 SNAT*
- Pod IP는 *클러스터 내부 망*. 외부에서 라우팅 안 됨
- → 노드의 *MASQUERADE*로 *노드 IP로 SNAT*. AWS VPC CNI는 *Pod IP 그대로* 나갈 수 있음 (VPC라우팅).

### 10-3. *NodePort 30080 못 들어옴*
- 보안 그룹·방화벽이 30000~32767 막을 가능성
- 클러스터 외부 LB가 NodePort 가리키는 게 일반적인데 *LB 헬스체크 fail* 가능성도

### 10-4. *Pod이 외부 DNS lookup 느림*
- `ndots:5` 설정이 *외부 도메인을 search 도메인으로 먼저 시도* → fail → 본 도메인 → 5번 추가 lookup
- 해결: `ndots:1` 또는 FQDN 사용 (`google.com.` 끝에 `.`)

### 10-5. *Service Endpoint에 Pod이 안 들어감*
- *readiness probe 실패* 가능성 가장 높음
- `kubectl get endpoints {service}` 로 *현재 등록된 Pod IP* 확인
- 0개면 → `kubectl describe pod` 로 readiness 실패 이유 추적

---

## 🎤 면접 빈출 Q&A

### Q1. Service의 ClusterIP / NodePort / LoadBalancer 차이?
> **ClusterIP**: 클러스터 *내부*만 접근 가능, 가상 IP. **NodePort**: 모든 노드의 *30000~32767 포트* 열림, 외부에서 노드 IP로 접근. **LoadBalancer**: 클라우드 LB 자동 생성, *외부 DNS/IP* 제공. 보통 *ClusterIP는 내부 서비스*, LoadBalancer는 *외부 노출*, NodePort는 *개발용·LB 베이스*.

### Q2. Pod 간 통신은 어떻게? CNI 역할은?
> K8s 네트워크 모델은 *모든 Pod이 라우팅 가능한 IP* + *NAT 없이 flat network*. 이 *구현 책임이 CNI 플러그인*. Flannel은 VXLAN overlay, Calico는 BGP 라우팅 또는 VXLAN, AWS VPC CNI는 *VPC IP 직접*, Cilium은 eBPF. K8s는 *CNI 인터페이스만 정의*, 구현은 외주.

### Q3. ClusterIP에 ping이 왜 안 되나요? 그런데 curl은 됩니까?
> ClusterIP는 *가상 IP*. 어떤 NIC에도 바인딩 안 됨, 어떤 프로세스도 listen 안 함. ICMP에 응답할 게 없으니 *ping fail*. 그런데 TCP/UDP는 *kube-proxy iptables 룰*이 *Service IP 향한 패킷을 잡아 실제 Pod IP로 DNAT*. → curl은 됨, ping은 안 됨.

### Q4. Ingress와 LoadBalancer Service의 관계?
> LoadBalancer Service는 *Service당 LB 1개* (비싸고 IP 누적). Ingress는 *L7 라우팅 (path/host)* — *LB 1개 뒤에 여러 Service*를 묶을 수 있음. 단, Ingress는 *spec*이고 *실제 처리하는 Controller* (nginx-ingress, AWS Load Balancer Controller 등) 필요. 보통 운영은 *Ingress + Controller가 LoadBalancer Service로 노출*.

### Q5. NetworkPolicy는 어떻게 작동하나요?
> *기본은 모두 허용* (no policy). NetworkPolicy 자원을 생성하면 *그 자원이 매칭하는 Pod에 한해 whitelist 동작*. 즉 *정책에 명시되지 않은 트래픽은 차단*. *구현은 CNI 책임* — Calico·Cilium 지원, Flannel은 별도. 운영 정석은 *default-deny 정책으로 시작 + 명시적 allow*.

### Q6. CoreDNS의 역할? `ndots:5` 문제는?
> 클러스터 내 DNS. Service 이름 → ClusterIP 매핑. `/etc/resolv.conf`의 `ndots:5` 옵션은 *dot이 5개 미만이면 search 도메인 먼저 시도*. 외부 도메인 (`google.com`, dot 1개) 호출 시 *클러스터 search 도메인들로 먼저 시도 → fail → 본 도메인*. → *외부 lookup 느려짐*. 해결: `ndots:1`로 줄이거나 *FQDN 끝에 `.`* 붙임.

### Q7. kube-proxy를 *대체*하는 게 있다고요?
> Cilium 같은 *eBPF 기반 CNI*는 kube-proxy 없이 *eBPF 프로그램이 Service 라우팅 직접* 처리. 장점: iptables 룰 수만 개 평가하는 비용 X, 더 빠르고 더 풍부한 관측성. 단점: *eBPF·Linux 커널 새 버전* 필요.

---

## 🔗 Cross-reference

- **veth/bridge/iptables의 *컨테이너 1개* 버전** → [`docker-basics/03`](../docker-basics/03-networking.md)
- **IP·라우팅·NAT 기본** → [`network-basics/01,02`](../network-basics/)
- **AWS VPC + LB** → [`network-basics/04,05`](../network-basics/), `aws-basics/` (예정)
- **NetworkPolicy 깊이** → `k8s-security-basics/` (예정)
- **Helm·ArgoCD가 Service·Ingress 어떻게 다루나** → `helm-basics/`, `cicd-gitops-basics/` (예정)

---

## 📝 3줄 요약

1. K8s 네트워킹의 4모델: *컨테이너↔컨테이너 (localhost), Pod↔Pod (CNI), Pod↔Service (kube-proxy DNAT), External↔Service (NodePort/LB/Ingress)*.
2. *Service ClusterIP는 가상 IP* — kube-proxy iptables/IPVS가 트리거로 *진짜 Pod로 DNAT*. ping은 안 되고 TCP/UDP만.
3. NetworkPolicy는 *whitelist*, *default-deny + 명시 allow* 패턴이 정석. 구현은 CNI 책임 (Calico/Cilium).

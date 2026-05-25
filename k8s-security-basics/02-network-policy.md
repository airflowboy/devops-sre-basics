# 02 — Network Policy: default-deny + 명시 allow

> **공식 근거:** [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [Calico docs](https://docs.tigera.io/calico/latest/about/), [Cilium docs](https://docs.cilium.io/)
>
> **선행:** [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md), [`network-basics/`](../network-basics/)

---

## 🎯 한 문장

> **NetworkPolicy = *K8s가 정의하는 Pod 간 통신 정책 spec*, *실제 시행은 CNI* (Calico/Cilium). *기본은 모두 허용* → *NetworkPolicy 적용 시 그 Pod에 whitelist 동작*. 운영 정석 = *default-deny + 명시 allow*. Cilium은 *eBPF로 L7 정책*도 가능.**

---

## 1. K8s 네트워크 *기본*은 모두 허용

### 함의
- 새 K8s 클러스터 → *모든 Pod이 모든 Pod과 통신 가능*
- *namespace 간*에도 자유
- *외부 인터넷*에도 자유 (kube-proxy + NAT)

### 위험
- 한 namespace의 *침해된 Pod*이 *다른 namespace의 DB* 접근 가능
- *멀티 테넌시*에선 *완전 노출*
- *최소 권한 원칙 위반*

### 해결: NetworkPolicy
- 특정 Pod에 *whitelist 정책* 정의
- 정책 매칭하는 Pod엔 *명시된 트래픽만 허용*

---

## 2. NetworkPolicy 기본 syntax

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-from-web
  namespace: default
spec:
  podSelector:                       # 정책 적용 대상 Pod
    matchLabels:
      app: api
  policyTypes:
  - Ingress                          # ingress 정책 활성
  - Egress                           # egress 정책 활성
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### 의미
- *namespace `default`의 `app: api`* Pod에 적용
- ingress: *같은 namespace의 `app: web`*에서 *TCP 8080*만 허용
- egress: *같은 namespace의 `app: postgres`*로 *TCP 5432*만 허용
- *그 외 모든 ingress/egress는 차단*

### 동작 규칙
- *NetworkPolicy가 없는 Pod*: *기본 (모두 허용)*
- *NetworkPolicy가 적용된 Pod*: *그 정책에 명시된 것만 허용*
- 여러 정책 매칭 시 *합집합* (OR)

---

## 3. Default-Deny 패턴 — *시작점*

### 모든 ingress 차단부터 시작
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
  namespace: prod
spec:
  podSelector: {}                    # 모든 Pod
  policyTypes:
  - Ingress
```

→ namespace의 *모든 Pod에 ingress 차단*. 이후 *명시적 allow 정책* 추가.

### Egress도
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-egress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

→ *모든 egress 차단 + DNS만 허용*. *DNS 없으면 cluster 통신 자체 안 됨*.

### 점진 도입
1. *audit 모드* — 정책 적용하나 차단 없이 *로깅만* (Calico 등 CNI 지원)
2. 모든 통신 *식별*
3. *명시 allow 정책 작성*
4. *실제 enforce*

---

## 4. Selector 종류

### podSelector
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: web
```
→ *같은 namespace의 `app: web`*.

### namespaceSelector
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: prod
```
→ *`environment: prod` label 있는 namespace의 모든 Pod*. (namespace 라벨링 필수)

### podSelector + namespaceSelector — *AND*
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: prod
    podSelector:
      matchLabels:
        app: web
```
→ *prod namespace의 `app: web`*만.

### 두 selector — *OR*
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: web
  - namespaceSelector:
      matchLabels:
        environment: prod
```
→ *같은 ns의 `app: web`* OR *prod ns의 모든 Pod*.

### ipBlock — *외부 IP 범위*
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.5.0/24            # 이 범위 제외
  ports:
  - protocol: TCP
    port: 443
```

→ 외부 (cluster 밖) IP 범위. 사내 corporate IP 만 허용 등.

---

## 5. Egress 정책 — *외부 통신 통제*

### 가장 흔한 정책
- **DNS만** (53/UDP, 53/TCP) — *기본 필수*
- **K8s API server** (443/TCP) — *Pod이 K8s API 호출 시*
- **사내 service mesh** — *control plane 통신*
- **외부 API** — *특정 IP/도메인만*

### 외부 도메인 통제는 *어려움*
- NetworkPolicy는 *IP/CIDR* 기반 — *DNS 이름 X*
- 외부 도메인의 IP는 *바뀜·여러 개*
- 해결:
  - **Cilium**: *FQDN 기반 정책* (L7)
  - **Service Mesh** (Istio): mTLS + egress 정책
  - **Egress Gateway**: 모든 외부 통신을 *gateway 거치기*

### DNS allow 정책 (필수)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## 6. CNI 책임 분리 — *NetworkPolicy 구현*

### K8s는 *spec만*
- NetworkPolicy CRD 정의
- *실제 시행은 CNI 책임*

### CNI별 지원

| CNI | NetworkPolicy 지원 |
|---|---|
| **Calico** | 완전 지원 + 확장 (GlobalNetworkPolicy 등) |
| **Cilium** | 완전 지원 + **eBPF L7 (HTTP path 기반 정책)** |
| **Flannel** | *기본은 X* → *Calico와 함께* (canal) |
| **AWS VPC CNI** | *별도 add-on* (Calico 또는 Cilium 추가) |
| **Weave** | 완전 지원 |

→ EKS에선 *VPC CNI + Calico/Cilium 추가* 가 흔함.

### 구현 방식
| CNI | 구현 |
|---|---|
| Calico | *iptables 룰* (또는 eBPF) |
| Cilium | *eBPF 프로그램* — 커널 hook |
| Flannel + Calico (canal) | iptables |

→ *eBPF가 성능·기능면에서 우월*. 점차 표준화 추세.

---

## 7. *Cilium L7 정책* — 더 풍부한 제어

### L3/L4 (표준 NetworkPolicy)
- 5-tuple (src IP, dst IP, src port, dst port, protocol) 기반
- "TCP 8080 허용" 수준

### L7 (Cilium 확장)
- *HTTP method·path 기반*
- 예: "GET /api/* 만 허용, POST 차단"

### CiliumNetworkPolicy 예시
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: web
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/.*"
        - method: POST
          path: "/api/orders"
```

### L7 정책의 가치
- *마이크로서비스의 *지나친 권한*을 제한*
- *internal endpoint 보호*
- *Service Mesh (Istio)와 유사* — Cilium은 *별도 sidecar 없이*

---

## 8. *Namespace 격리* 패턴

### Multi-tenancy
- 각 *팀·고객별 namespace*
- *namespace 간 통신 차단* (기본)

### 정책
```yaml
# 1. 모든 namespace간 통신 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-ns
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: tenant-a   # 같은 ns만

# 2. 시스템 namespace는 예외
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
```

### Calico GlobalNetworkPolicy
- 모든 namespace에 *cluster-wide 적용*
- 운영 표준 정책 (예: *kube-system 외 모든 namespace에 deny-cross-ns*)

---

## 9. *Audit / Observability*

### NetworkPolicy 효과 검증
- *적용 전*: `netshoot` Pod에서 `nc` / `curl` 로 *모든 endpoint 접근 시도* → 다 됨
- *적용 후*: 같은 시도 → *명시된 것만 됨*

### Calico Flow Logs
- 각 Pod의 *허용·차단 트래픽 로깅*
- *어떤 정책이 누구를 막았는지* 추적
- Enterprise 기능 (또는 *Calico Cloud*)

### Cilium Hubble
- *eBPF 기반 observability*
- *Pod 간 통신 시각화 + 정책 평가 결과*
- *오픈소스* — Cilium 사용 시 *큰 가치*

```bash
hubble observe --from-label app=web --to-label app=api
# Service flow log
```

---

## 10. 자주 헷갈리는 것

### 10-1. *NetworkPolicy 적용했는데 *효과 X**
- CNI가 *지원 안 함* (Flannel 기본 등)
- → CNI 확인. EKS면 Calico/Cilium 추가.

### 10-2. *Egress 차단 후 *모든 통신 끊김**
- *DNS 차단* — Pod의 hostname lookup 실패 → *모든 통신 실패*
- → *DNS allow 정책 *먼저**

### 10-3. *외부 도메인 정책*
- NetworkPolicy는 *IP/CIDR 기반*. 도메인 X.
- → Cilium FQDN 또는 *egress gateway*

### 10-4. *namespaceSelector가 *빈 label namespace에 매칭 안 됨**
- K8s 1.22+ — 모든 namespace에 *`kubernetes.io/metadata.name` label 자동* 부여
- 옛 클러스터에선 *수동 label 필요*

### 10-5. *podSelector 의 *AND vs OR**
```yaml
ingress:
- from:
  - podSelector: {...}        # OR
  - podSelector: {...}        # OR

ingress:
- from:
  - podSelector: {...}        # AND
    namespaceSelector: {...}
```
→ 같은 *from item 안 selectors는 AND*, 다른 from item은 *OR*.

### 10-6. *Allow rule이 *deny를 override 안 함**
- 정책은 *additive (whitelist)*
- *deny rule은 없음* (표준 NetworkPolicy)
- Calico/Cilium의 *확장 CRD*는 deny 가능

---

## 🎤 면접 빈출 Q&A

### Q1. K8s의 기본 네트워크 정책은? NetworkPolicy 적용 시 동작?
> 기본은 **모두 허용** — 모든 Pod이 모든 Pod과 통신. NetworkPolicy 적용 시 *그 정책 매칭 Pod에 *whitelist 동작** — 명시되지 않은 트래픽은 차단. 적용 안 된 Pod은 *기본 (모두 허용)* 유지. 운영 정석은 **default-deny + 명시 allow** 패턴.

### Q2. NetworkPolicy의 *실제 구현*은?
> K8s는 *spec만 정의*. **실제 시행은 CNI 책임**. *Calico* (iptables 또는 eBPF), *Cilium* (eBPF), *Weave* (지원). **Flannel은 기본 미지원** — Calico와 함께 (canal). **AWS VPC CNI는 별도 add-on 필요** (Calico/Cilium). CNI 선택이 *NetworkPolicy 기능에 직접 영향*.

### Q3. default-deny 패턴은 어떻게?
> `podSelector: {}` (모든 Pod) + `policyTypes: [Ingress]` (ingress 차단). egress도 별도. **단 *DNS allow 정책 먼저***. DNS 차단되면 *hostname lookup 실패 → 모든 통신 실패*. 도입 흐름: *audit mode → 위반 식별 → 명시 allow → enforce*.

### Q4. Egress 정책에서 *외부 도메인 통제* 어떻게?
> 표준 NetworkPolicy는 *IP/CIDR 기반* — 도메인 X. 외부 도메인 IP는 *바뀜·여러 개* → 정책 어려움. **해결**: (1) **Cilium FQDN 정책** (L7), (2) **Service Mesh (Istio) egress 정책**, (3) **Egress Gateway** — 모든 외부 통신을 gateway 거치게. Cilium L7이 가장 modern.

### Q5. Cilium의 *L7 정책*은 무엇?
> 표준 NetworkPolicy는 *L3/L4 (5-tuple)*. Cilium은 *eBPF로 L7 (HTTP method/path) 정책*. 예: "GET /api/* 만 허용, POST 차단". *Service Mesh와 유사하나 sidecar 없이*. **마이크로서비스의 *과도한 권한 제한*에 핵심**.

### Q6. *Multi-tenancy* 격리 어떻게?
> *팀·고객별 namespace* + *namespace 간 통신 차단* NetworkPolicy. `namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: tenant-a } }` 패턴. *시스템 namespace 예외* (kube-system DNS 등). Calico **GlobalNetworkPolicy**로 *cluster-wide 적용*. + *RBAC* + *PSS* + *Resource Quota* 다 같이.

### Q7. NetworkPolicy 효과를 *어떻게 observability*?
> **Calico Flow Logs** (Enterprise) — 허용·차단 트래픽 로깅. **Cilium Hubble** (오픈소스) — eBPF 기반 *Pod 간 통신 시각화 + 정책 평가 결과*. `hubble observe`로 *실시간 flow*. 적용 전후 *test Pod에서 nc/curl로 *모든 endpoint 시도* → 차이 확인*.

---

## 🔗 Cross-reference

- **K8s 네트워킹 베이스** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md)
- **CNI 종류·동작** → [`network-basics/06`](../network-basics/06-kubernetes.md)
- **Service Mesh (Istio) — egress 통제 + mTLS** → 외부 자료
- **K8s RBAC** → [03-identity-and-rbac.md](03-identity-and-rbac.md)
- **AWS VPC + Security Group** (네트워크의 클라우드 layer) → [`network-basics/05`](../network-basics/05-aws-vpc.md)

---

## 📝 3줄 요약

1. *K8s 기본 = 모두 허용*. NetworkPolicy 적용 시 *그 Pod에 whitelist*. **default-deny + 명시 allow**가 운영 정석. **DNS allow 먼저** 필수.
2. *K8s는 spec만, 실제 시행은 CNI*. **Calico (iptables/eBPF) / Cilium (eBPF + L7) / Flannel (미지원)**. EKS는 *VPC CNI + Calico/Cilium 추가*.
3. *외부 도메인 통제는 Cilium FQDN* 또는 *egress gateway*. *Cilium Hubble*로 observability. *Multi-tenancy*는 *namespace + 정책 + RBAC + PSS + Quota* 종합.

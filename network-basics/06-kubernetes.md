# 06. Kubernetes 네트워킹 — Pod IP / Service / Ingress / NetworkPolicy / CNI

> K8s 네트워킹은 *겹쳐있는 추상*이 많아 헷갈림. 한 번에 정리.

## 목차
- [4 모델 동시에 이해](#4-모델-동시에-이해)
- [Pod IP — 임시 책상번호](#pod-ip--임시-책상번호)
- [Service — 부서 대표번호](#service--부서-대표번호)
- [Ingress — 안내데스크](#ingress--안내데스크)
- [NetworkPolicy — 동별 출입표](#networkpolicy--동별-출입표)
- [CNI — 단지 내부 도로 시공](#cni--단지-내부-도로-시공)
- [클라우드 LB와의 관계](#클라우드-lb와의-관계)
- [치트시트](#치트시트)

---

## 4 모델 동시에 이해

K8s에는 *네트워크 추상이 4겹*. 이해하면 다 풀림:

```
1. 컨테이너 ↔ 컨테이너  (한 Pod 안)
   → localhost 통신. 같은 네임스페이스 공유.

2. Pod ↔ Pod  (클러스터 안)
   → 각 Pod IP 직접 통신. CNI(Calico)가 라우팅.
   → 모든 Pod이 모든 Pod 볼 수 있음 (NetworkPolicy로 제한 가능).

3. Pod ↔ Service  (안정된 endpoint)
   → Service의 ClusterIP를 호출하면 백엔드 Pod 중 하나로.
   → kube-proxy가 iptables로 LB.

4. 외부 ↔ Service  (인그레스)
   → NodePort / LoadBalancer / Ingress(L7) 방식.
```

→ "이게 어느 모델 이야기인가" 의식하면 헷갈림 ↓.

---

## Pod IP — 임시 책상번호

> 각 Pod이 받는 *클러스터 내부* IP. 사람(Pod) 바뀌면 번호도 바뀜.

### 특성
- **CNI가 할당** — Pod CIDR 안에서 (홈랩 `10.244.0.0/16`, AWS EKS는 VPC CIDR 안 — *Pod이 VPC IP를 직접 받음*).
- **임시** — Pod 재생성되면 새 IP. 그래서 *직접 Pod IP를 호출하지 말 것*. Service를 호출.
- **클러스터 안에서만 유효** — 외부에서 Pod IP로 접속 X.

### Pod끼리 통신
- 같은 노드: 가상 인터페이스로 직접.
- 다른 노드: CNI가 *오버레이 네트워크*(VXLAN 등)로 터널링 — 또는 underlay 라우팅.

### EKS의 특이점 — Pod이 VPC IP
- EKS 기본 CNI(VPC CNI)는 *Pod에 VPC 안의 IP를 직접 부여*.
- 결과: Pod이 *AWS 자원과 같은 네트워크*. SG·라우팅이 Pod 단위로 적용 가능.
- 단점: 노드당 Pod 수가 IP 수에 제한 (작은 인스턴스는 Pod 적게).
- 다른 CNI (Calico VXLAN 등)는 별도 Pod CIDR — 우리 홈랩 패턴.

### 우리 프로젝트
- 홈랩: Pod CIDR `10.244.0.0/16`. Pod IP는 `10.244.x.x`.
- AWS EKS (Ch 10): VPC CNI라 Pod이 `10.0.11.x` / `10.0.12.x` (프라이빗 서브넷의 IP).

---

## Service — 부서 대표번호

> Pod이 바뀌어도 *고정된 endpoint*. 클러스터 안의 추상.

### 왜 필요?
- Pod IP는 임시. 죽었다 살아나면 바뀜.
- → Service가 *Pod 그룹*에 안정된 *ClusterIP* + *DNS 이름* 부여.
- 다른 Pod이 Service 이름·IP로 호출 → kube-proxy가 살아있는 Pod 중 하나로 분배.

### Service 종류
| 종류 | 동작 | 사용처 |
|------|------|--------|
| **ClusterIP** (기본) | 클러스터 내부 가상 IP | Pod간 통신 |
| **NodePort** | 노드의 30000-32767 포트 노출 | 개발/테스트 외부 접속 |
| **LoadBalancer** | 클라우드 LB(NLB) 자동 생성 | TCP·gRPC 외부 노출 |
| **ExternalName** | DNS CNAME만 (Pod 없음) | 외부 서비스 별칭 |
| **Headless** (`clusterIP: None`) | ClusterIP 없음, DNS가 Pod IP 직접 | StatefulSet (DB 등) |

### 작동 원리 — ClusterIP의 실체
```
Client Pod → curl http://web:80
   │
   │ DNS: web → CoreDNS → 10.96.42.17 (ClusterIP)
   │
   │ kube-proxy가 노드의 iptables에 깐 룰 발동:
   │   목적지 10.96.42.17:80 → 10.244.1.5:8080 (Pod A)
   │                        or 10.244.2.7:8080 (Pod B)
   │                        or 10.244.3.9:8080 (Pod C)
   │ → 라운드 로빈(또는 IPVS면 다양한 알고리즘)
   ▼
Pod A의 nginx
```

→ ClusterIP는 *진짜로 존재하는 IP가 아님*. iptables 룰 안에만 있음. 그래서 `ping`은 안 됨, `curl`만.

### Service DNS
- `<service>.<namespace>.svc.cluster.local`
- 같은 ns 안: `web` 만 써도 됨 (CoreDNS가 자동 보완)
- 다른 ns: `web.production.svc.cluster.local`

### 흔한 함정
- "Pod에서 Service IP로 안 통함" → kube-proxy 죽었거나 NetworkPolicy 차단
- "Service Endpoint가 비어있음" → label selector가 Pod와 매칭 안 됨 (`kubectl describe svc <name>`)
- "Pod 직접 IP로는 되는데 Service IP로 안 됨" → Service의 `targetPort`가 Pod의 실제 포트와 다를 수 있음

---

## Ingress — 안내데스크

> HTTP(S) 트래픽을 *호스트·경로 기반*으로 Service에 라우팅.

### Service vs Ingress 차이
- **Service (LoadBalancer)**: 한 서비스 = 한 LB. 비싸고 단순.
- **Ingress**: 한 LB로 *여러 서비스*에 라우팅. 경로/호스트별 분기.

### 흔한 패턴
```yaml
rules:
  - host: api.example.com
    http:
      paths:
        - path: /users
          backend:
            service: { name: user-api, port: 80 }
        - path: /orders
          backend:
            service: { name: order-api, port: 80 }
  - host: web.example.com
    http:
      paths:
        - path: /
          backend:
            service: { name: web-frontend, port: 80 }
```

→ 한 ALB로 두 도메인 + 여러 경로 라우팅.

### Ingress *Controller*가 실제 일을 함
- `Ingress` 객체는 *희망사항*만 명시.
- **Ingress Controller**가 그걸 보고 실제 LB/proxy 설정:
  - ingress-nginx (홈랩 Ch 04에서 사용) — nginx Pod이 트래픽 받음
  - AWS Load Balancer Controller (EKS Ch 10) — ALB 자동 생성
  - Traefik / Istio / Contour 등

### TLS
- Ingress에 `tls:` 섹션 + 시크릿(인증서) 참조
- 홈랩: cert-manager가 자체 CA로 자동 발급 (Ch 04-C)
- AWS: ACM 인증서 + ALB가 종단

### 우리 프로젝트
- 홈랩: `nginx.<IP>.nip.io`, `argocd...`, `grafana...` 모두 ingress-nginx 통과
- AWS: 캡스톤에서 ALB가 `webapp.<도메인>` → issue-api Service로

---

## NetworkPolicy — 동별 출입표

> Pod간 통신을 *허용/거부* 명시. Ch 09 Phase A에서 학습.

### 기본
- NetworkPolicy 없으면 → 모든 Pod이 모든 Pod 볼 수 있음 (default allow)
- 하나라도 매칭되면 → 그 Pod은 *명시된 것만* 허용 (default deny + selective allow)

### Ch 09 Phase A 핵심 패턴
```yaml
# default-deny: 이 ns의 모든 Pod ingress 거부
spec:
  podSelector: {}              # 모든 Pod
  policyTypes: [Ingress]
  # ingress 룰 없음 = 다 거부

---
# allow: client → web 만 허용
spec:
  podSelector:
    matchLabels: { app: web }   # 이 정책의 대상
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: client }   # client에서만 OK
      ports:
        - protocol: TCP
          port: 80
```

### CNI가 NetworkPolicy 지원해야 동작
- **Calico** ✅ (우리 홈랩)
- **Cilium** ✅
- **AWS VPC CNI**: 자체로 NetworkPolicy 미지원 → **Calico 사이드카** 깔거나 Cilium 도입
- **Flannel** ❌

→ EKS 캡스톤(Phase I)에서: VPC CNI + Calico addon, 또는 Cilium 도입 결정 필요 (ADR 후보).

### SG vs NetworkPolicy
| | **SG** | **NetworkPolicy** |
|---|---|---|
| 단위 | 인스턴스/ENI | Pod (라벨 기반) |
| 범위 | AWS 자원 | K8s 자원 |
| 정의자 | 인프라 팀 | 앱 팀 |
| 적층 | NetworkPolicy 위에 SG | SG 안에 NetworkPolicy |

→ 둘 다 *defense in depth*. SG로 노드 격리, NetworkPolicy로 Pod 격리.

---

## CNI — 단지 내부 도로 시공

> **Container Network Interface** — 컨테이너에 네트워크 붙여주는 *플러그인 표준*.

### 역할
- Pod 생성 시: Pod에 IP 할당, 가상 NIC 만들고 호스트 네트워크에 연결
- Pod 삭제 시: IP 회수
- 네트워크 정책 구현 (NetworkPolicy → iptables 룰)

### 주요 CNI 비교
| CNI | 데이터 평면 | NetworkPolicy | 특징 |
|-----|------------|:------------:|------|
| **Calico** | iptables/eBPF (옵션 VXLAN) | ✅ | 가장 흔함, BGP 라우팅 가능 |
| **Cilium** | eBPF (커널) | ✅ + L7 | 성능·관찰성 ↑, 학습 곡선 |
| **Flannel** | VXLAN | ❌ | 가장 단순, 작은 클러스터 |
| **AWS VPC CNI** | AWS ENI (Pod이 VPC IP) | ❌ (Calico와 같이) | EKS 기본 |
| **Weave** | VXLAN | ✅ | 옛 인기, 요즘 사용 ↓ |

### 우리 홈랩
- **Calico** (Ch 03에서 설치)
- Pod CIDR `10.244.0.0/16`
- VXLAN 모드 (overlay)
- NetworkPolicy 지원 ✅

### AWS EKS (Ch 10)
- 기본 **VPC CNI** — Pod이 VPC IP
- NetworkPolicy 필요하면 → Calico addon 같이
- 또는 Cilium 도입 (ADR 후보)

### "이거 안 통하는데 CNI 문제?"
- 대부분 CNI는 무관. 통신 안 됨은 *위 층(SG/NetworkPolicy/Service/DNS)*이 더 흔한 원인.
- CNI 문제 = Pod이 *IP 자체를 못 받음* (`kubectl describe pod`에 "no IP assigned" 류), 또는 *전 Pod이 서로 통신 안 됨*.

---

## 클라우드 LB와의 관계

K8s Service 종류와 클라우드 자원 매핑:

| K8s 리소스 | AWS 자원 (EKS) | 트래픽 흐름 |
|-----------|----------------|------------|
| Service `ClusterIP` | (없음, 내부만) | Pod → kube-proxy iptables → Pod |
| Service `NodePort` | (없음, 노드 IP:포트) | 외부 → 노드 IP:30000+ → kube-proxy → Pod |
| Service `LoadBalancer` | **NLB** (자동 생성) | 외부 → NLB → 노드 IP → kube-proxy → Pod |
| `Ingress` (ALB Controller) | **ALB** (자동 생성) | 외부 → ALB → 노드(Target Group) → kube-proxy → Pod |

### AWS Load Balancer Controller
- EKS에 설치하면, Ingress 리소스를 보고 ALB 자동 생성
- Service type=LoadBalancer 보면 NLB 자동 생성
- 만들 LB의 세부 사항은 *annotation*으로 (인증서 ARN, target type, scheme 등)

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip      # Pod IP 직접 (VPC CNI라서)
    alb.ingress.kubernetes.io/certificate-arn: <ACM ARN>
```

→ Phase D에서 본격 활용.

---

## 치트시트

### "이거 어느 추상?"
- "Pod에서 다른 Pod 호출" → Service ClusterIP
- "외부에서 웹 접속" → Ingress + Ingress Controller
- "외부에서 DB 접속" → Service LoadBalancer (NLB) — 보통 안 함 (DB는 private)
- "Pod이 외부 인터넷" → SNAT (Calico/VPC CNI가 노드 IP로 변환)

### Service 디버깅 4단계
```
1. Service 존재? → kubectl get svc <name>
2. Endpoint에 Pod 있나? → kubectl get endpoints <name>
   (없으면 selector·label 매칭 실패)
3. Pod 자체 살아있나? → kubectl get pods -l <selector>
4. Pod IP 직접 호출됨? → curl <PodIP>:<port> (Pod 안에서)
```

### Ingress 디버깅
```
1. Ingress 객체 존재? → kubectl get ingress -A
2. Ingress Controller Pod 살아있나? → kubectl get pods -n ingress-nginx (or aws-load-balancer-controller)
3. ALB 만들어졌나? (AWS) → aws elbv2 describe-load-balancers
4. DNS가 ALB로? → dig <도메인>
5. Target Group이 healthy? → AWS Console "Target Groups"
```

### NetworkPolicy 디버깅
```
1. Pod이 통신 안 됨 → kubectl get networkpolicy -n <ns>
2. 어느 정책이 매칭? → kubectl describe networkpolicy <name>
3. CNI가 지원? → Calico면 OK, AWS VPC CNI는 별도 addon 필요
4. curl exit code:
   28 = timeout (NetworkPolicy 가능성 ↑)
   7  = refused (서버 안 떠있음, 정책 무관)
```

### 한 줄 정리
> **Pod IP는 임시, Service는 안정된 가상 IP, Ingress는 L7 라우팅, NetworkPolicy는 Pod 단위 방화벽.**
> **CNI가 이 모든 데이터 평면을 깐다.**

---

## 다 읽었다면 — 자가 진단 답

[README의 5개 질문](README.md#-자가-진단--이-5개-답할-수-있으면-80-잡힘):

1. **`10.0.1.5`로 우리 집 컴퓨터에서 핑 안 가는 이유?**
   → 사설 IP. 다른 단지(나의 집 네트워크)에서 라우팅 안 됨. 그 IP는 *각 단지마다* 다른 것 가리킴.

2. **EC2 SSH(22)는 되는데 HTTP(80)는 안 되는 이유?**
   → SG에 22는 inbound 있는데 80 없음. 또는 nginx 안 돔(L7). 순서로 확인: SG → 호스트 방화벽 → 프로세스 listen.

3. **프라이빗 EC2가 `apt update` 안 되는 이유?**
   → NAT GW 없거나, private RT에 `0.0.0.0/0 → NAT` 룰 없음. outbound 길이 안 깔림.

4. **`0.0.0.0/0`이 뭘 의미?**
   → "모든 IP". 라우트 테이블에선 *default route* — "다른 룰 안 매칭되면 이쪽으로".

5. **Pod이 Service IP 부르면 실제 Pod까지 어떻게 도착?**
   → kube-proxy가 노드의 iptables(또는 IPVS)에 룰을 깔아둠. 목적지 ClusterIP:port → 백엔드 Pod 중 하나의 PodIP:targetPort로 DNAT. CNI가 그 Pod IP로 라우팅.

---

## 끝.

이 6개 파일을 다 읽었으면 — DevOps/SRE 일상에서 마주치는 *어떤 네트워크 상황도* 비유와 함께 풀어낼 수 있습니다.

여전히 헷갈리면 **다시 돌아와도 됩니다.** 네트워크는 한 번에 안 잡혀요. *몇 번 돌아오는 게 정상*. 우리 프로젝트 코드(`terraform/ch10/network.tf` 같은) 보면서 이 문서랑 매칭하면 더 빨리 잡힙니다.

```
실제 코드 ←→ 이 문서 ←→ 비유
이 셋이 다 매칭되면 그 개념은 "내 것"이 됨.
```

🚀 화이팅.

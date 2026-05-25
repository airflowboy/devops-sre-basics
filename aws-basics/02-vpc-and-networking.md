# 02 — VPC & Networking: Subnet · Route · SG · NACL · PrivateLink

> **공식 근거:** [VPC docs](https://docs.aws.amazon.com/vpc/latest/userguide/), [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
>
> **선행:** [`network-basics/`](../network-basics/), [`network-basics/05-aws-vpc.md`](../network-basics/05-aws-vpc.md)

---

## 🎯 한 문장

> **VPC = *AWS의 가상 사설 네트워크*. *Subnet (AZ별)* + *Route Table* + *Internet Gateway (외부 양방향)* / *NAT Gateway (외부 단방향)* + *Security Group (stateful)* + *NACL (stateless)*. 클라우드 네트워크의 *모든 자원이 살 곳*.**

---

## 1. VPC 기본 — *가상 사설 네트워크*

### VPC = 한 region 안 *격리된 네트워크*
- CIDR 정의 (예: `10.0.0.0/16` — 65k IP)
- *그 안에 subnet, EC2, RDS, EKS Pod 등 모든 자원*
- *기본 VPC*가 자동 생성 (account 만들 때)

### Subnet
- *VPC의 *분할* — 각 subnet은 *하나의 AZ에만**
- CIDR (예: `10.0.1.0/24` — 256 IP)
- *Public vs Private* — 라우팅에 따라 결정

### 흔한 패턴 (멀티 AZ)
```
VPC: 10.0.0.0/16
├── Subnet (Public, AZ-a):   10.0.1.0/24
├── Subnet (Public, AZ-c):   10.0.2.0/24
├── Subnet (Private, AZ-a):  10.0.11.0/24
├── Subnet (Private, AZ-c):  10.0.12.0/24
├── Subnet (DB, AZ-a):       10.0.21.0/24
└── Subnet (DB, AZ-c):       10.0.22.0/24
```

→ 3 tier (public/private/db) × 2 AZ = 6 subnet.

---

## 2. Route Table — *어디로 갈지*

### 기본
- 각 subnet은 *하나의 route table에 연결*
- 매번 패킷마다 *route table 조회* → 다음 hop 결정

### Public Subnet의 라우트 테이블
```
Destination       Target
10.0.0.0/16       local            ← 같은 VPC 안
0.0.0.0/0         igw-12345         ← 그 외 모든 곳 → Internet Gateway
```

→ *`0.0.0.0/0 → IGW` 가 있으면 public subnet*.

### Private Subnet의 라우트 테이블
```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         nat-12345         ← 외부는 NAT GW 거쳐서
```

→ *외부 갈 수 있되 외부에서 들어올 수 없음* (NAT의 단방향).

### DB Subnet의 라우트 테이블
```
Destination       Target
10.0.0.0/16       local             ← VPC 안만
                                    ← 0.0.0.0/0 없음 = 외부 완전 차단
```

→ *완전 격리* — 외부와 단방향도 안 됨.

---

## 3. Internet Gateway (IGW) vs NAT Gateway

### IGW — *양방향 외부 통신*
- VPC에 하나만
- *Public IP 가진 인스턴스가 외부 통신*
- *외부도 그 public IP로 접근 가능*

### NAT Gateway — *내→외 단방향*
- Private subnet의 인스턴스가 *외부 갈 때 거침*
- *외부에서 들어오는 건 불가* (NAT의 본질)
- *AZ에 묶임* — 보통 *AZ별로 하나씩* (HA)
- *시간당 + 데이터 transfer 비용* — *비쌈*

→ [`network-basics/02-routing-and-nat.md`](../network-basics/02-routing-and-nat.md) 의 NAT 개념과 일치.

### 비용 절감 — *VPC Endpoint*
- *AWS service (S3, DynamoDB, etc.) 접근*은 *NAT 거치지 않고 *VPC endpoint*로*
- *NAT 비용 절감 + 보안*

```
Pod → S3 → 옛 방식: NAT GW → 외부 (인터넷) → S3
       → 새 방식: VPC Endpoint → S3 (AWS 내부망 직접)
```

---

## 4. Security Group (SG) — *Stateful 방화벽*

### 특징
- *EC2/RDS/ALB 등 자원에 *attach**
- *Stateful* — outbound 요청의 *return traffic 자동 허용*
- *Whitelist 만 — 명시한 것 외 차단*
- *Allow 만, Deny 없음*

### 규칙 예시
```
Inbound:
  Type: SSH (TCP 22)   Source: 10.0.0.0/16   (VPC 안 SSH 허용)
  Type: HTTPS (443)    Source: 0.0.0.0/0     (어디서나 HTTPS)

Outbound:
  Type: All            Destination: 0.0.0.0/0   (모든 outbound 허용)
```

### *Source*가 *다른 SG*도 가능 (강력)
```
Source: sg-12345 (ALB의 SG)
```

→ *ALB에서 오는 트래픽만 허용* — 동적 IP에도 작동. *micro-service 통신의 표준 패턴*.

### 자원당 *최대 SG 수*
- *기본 5개*, *최대 16개*
- 너무 많으면 *유지보수 복잡*

---

## 5. NACL (Network ACL) — *Stateless 방화벽*

### 특징
- *subnet 단위* (인스턴스 X)
- *Stateless* — *return traffic도 명시 필요*
- *Allow + Deny 둘 다 가능*
- *번호 순서대로 평가* (낮은 번호 먼저)

### 규칙 예시
```
Inbound:
  100  Allow  TCP  443        Source: 0.0.0.0/0
  110  Allow  TCP  1024-65535 Source: 0.0.0.0/0   ← ephemeral port (return)
  *    Deny   ALL             Source: 0.0.0.0/0

Outbound:
  100  Allow  TCP  ALL        Destination: 0.0.0.0/0
  *    Deny   ALL
```

### *Ephemeral port 함정*
- Outbound 요청 → return traffic은 *random high port* (32768-60999)
- *NACL inbound에 그 range 허용 필수*
- 안 하면 *outbound 요청에 응답 못 받음*

### SG vs NACL

| | Security Group | NACL |
|---|---|---|
| Scope | 인스턴스 단위 | Subnet 단위 |
| State | Stateful | Stateless |
| Allow/Deny | Allow만 | 둘 다 |
| 평가 | 모든 규칙 OR | 번호 순서 |
| Default | 모두 차단 | (기본 NACL) 모두 허용 |

→ **운영은 SG 위주**. NACL은 *subnet 전체 차단* 같은 *coarse 정책*에만.

---

## 6. PrivateLink — *VPC 간 / AWS service 비공개 접근*

### 발상
- *외부 인터넷 거치지 않고 다른 VPC·service 접근*
- *PrivateLink endpoint*가 *VPC 안에 ENI로 생김*

### VPC Endpoint 종류
| | 용도 |
|---|---|
| **Gateway Endpoint** | S3, DynamoDB (route table에 entry) |
| **Interface Endpoint** | 그 외 service (ENI 형태) |

### Interface Endpoint
- VPC 안에 *private IP* 받음
- *그 IP로 service API 호출* → AWS 내부망 직접
- DNS도 *VPC private DNS*로 매핑

### 예시
```
EC2 (private subnet) → secretsmanager.ap-northeast-2.amazonaws.com
                       ↓ (private DNS 해석)
                     10.0.11.50 (Interface Endpoint IP)
                       ↓ (AWS 내부망)
                     Secrets Manager service
```

→ *NAT GW 불필요*, *외부 노출 X*, *비용 절감*.

### 자주 Endpoint 거는 service
- S3, DynamoDB (Gateway — 무료)
- ECR, Secrets Manager, KMS, STS, SSM (Interface — 시간당 비용)

### VPC Peering vs PrivateLink
- **VPC Peering**: VPC 간 *전체 라우팅* — *모든 자원 접근 가능* (CIDR 충돌 시 불가)
- **PrivateLink**: *특정 service만* 노출 — *CIDR 충돌 OK*, *방향 단방향*

---

## 7. Transit Gateway — *Hub-and-Spoke*

### 발상
- VPC 수십 개 → *Peering 메시* — N(N-1)/2 연결 = 폭증
- *Transit Gateway가 *hub** → 모든 VPC가 *TGW에 연결*만

### 구조
```
        Transit Gateway (hub)
         /    |    \    \
       VPC1  VPC2  VPC3  on-prem (VPN/Direct Connect)
```

### 장점
- *연결 단순화*
- *on-prem 통합*
- *cross-region peering*
- *route table per attachment*

### 비용
- *시간당 + 데이터 transfer 비용*
- 작은 환경엔 *오버킬*

---

## 8. Load Balancer

### ALB (Application Load Balancer) — *L7*
- HTTP/HTTPS
- *Path / Host 기반 라우팅*
- *WebSocket / gRPC*
- *target group*에 EC2·IP·Lambda
- *EKS Ingress*의 표준 backend

### NLB (Network Load Balancer) — *L4*
- TCP/UDP
- *극단 성능* (수백만 connection)
- *static IP* (또는 EIP)
- *TLS termination*

### CLB (Classic) — 옛 — 사용 X

### *EKS와의 통합*
- **AWS Load Balancer Controller** (Helm install)
- K8s `Service type=LoadBalancer` → NLB 자동
- K8s `Ingress` → ALB 자동
- *target type IP*면 *Pod IP 직접* (NLB가 Pod 직접 가리킴, *AWS VPC CNI 활용*)

---

## 9. Route 53 — *DNS*

### 자주 쓰는 record
- **A** — IPv4
- **AAAA** — IPv6
- **CNAME** — alias to another domain
- **ALIAS** — AWS service에만 (ALB·CloudFront·S3 등) — *root domain CNAME 같이 동작*
- **MX** — mail
- **TXT** — verification·SPF

### Routing Policy
| Policy | 의미 |
|---|---|
| **Simple** | 한 record |
| **Weighted** | 비율 (canary 등) |
| **Latency** | 가장 가까운 region |
| **Geolocation** | 사용자 region별 |
| **Failover** | primary fail 시 secondary |
| **Multi-value** | 여러 IP 응답 + health check |

### Health Check
- Route 53이 *endpoint 주기 체크*
- fail 시 *DNS 응답에서 제외* (또는 failover trigger)

---

## 10. 자주 헷갈리는 것

### 10-1. *NAT 비용 폭증*
- private subnet의 모든 외부 트래픽 *NAT 거침*
- AWS service (S3, DynamoDB)도 *외부로 인식* → NAT 비용
- **해결**: VPC Endpoint (Gateway 무료, Interface 저렴)

### 10-2. *NACL ephemeral port 누락*
- Outbound 응답이 *random high port*로 옴
- NACL inbound에 *32768-60999 허용 필수*

### 10-3. *SG 너무 많이 attach*
- 16개 한도 + *유지보수 복잡*
- *role 단위로 SG 통합* — 한 인스턴스에 SG 2-3개 권장

### 10-4. *Public IP vs Elastic IP*
- *Public IP*: instance stop·start 시 *변경*
- *Elastic IP (EIP)*: *고정 public IP*. unattached 시 *시간당 비용*

### 10-5. *VPC CIDR 너무 작게*
- `/24` (256 IP) — Pod 많이 띄우면 *고갈*
- 권장 `/16` 또는 `/18` — *충분한 여유*
- *나중에 확장 가능*하나 *복잡* (secondary CIDR)

### 10-6. *VPC Peering이 *transitive 안 됨**
- A↔B, B↔C peering → A↔C *자동 X*
- A↔C도 *별도 peering* 필요
- *Transit Gateway가 해결*

---

## 🎤 면접 빈출 Q&A

### Q1. Public Subnet vs Private Subnet 차이?
> **Public**: route table에 `0.0.0.0/0 → IGW` 있음 → *직접 외부 통신*, *public IP로 외부 도달 가능*. **Private**: `0.0.0.0/0 → NAT GW` 또는 없음 → *외부 갈 때 NAT 거침 (단방향)*, *외부에서 접근 불가*. **DB tier**는 *0.0.0.0/0 없음* — 완전 격리.

### Q2. Security Group과 NACL 차이?
> **SG**: *인스턴스 단위*, *stateful* (return traffic 자동 허용), *allow만*. **NACL**: *subnet 단위*, *stateless* (return traffic 명시), *allow + deny*. **운영은 SG 위주**. NACL은 *subnet 전체 차단* 같은 coarse 정책에만. **Ephemeral port (32768-60999)** NACL에서 자주 함정.

### Q3. NAT Gateway 비용 절감?
> *NAT는 시간당 + 데이터 transfer 비용 — 비쌈*. **VPC Endpoint**로 *AWS service (S3, DynamoDB 등) NAT 거치지 않고 직접*. *Gateway Endpoint* (S3, DynamoDB — 무료), *Interface Endpoint* (ECR, Secrets Manager 등 — 시간당 비용). *NAT GW 사용량 분석 후 자주 호출되는 service에 endpoint*.

### Q4. PrivateLink vs VPC Peering 차이?
> **VPC Peering**: VPC 간 *전체 라우팅* — 모든 자원 접근, *CIDR 충돌 시 불가*. **PrivateLink**: *특정 service만* 노출 (Interface Endpoint), *CIDR 충돌 OK*, *방향 단방향*. *SaaS service 통합*은 PrivateLink, *내부 VPC 통합*은 Peering 또는 Transit Gateway.

### Q5. Transit Gateway가 *왜 필요*?
> VPC 수십 개면 *Peering 메시 폭증* (N(N-1)/2 연결). **TGW = hub-and-spoke** — 모든 VPC가 TGW에 연결만. *on-prem 통합 (VPN/Direct Connect)*, *cross-region peering*도 지원. *시간당 + transfer 비용* — 작은 환경은 오버킬.

### Q6. ALB와 NLB 차이? 언제 무엇?
> **ALB**: *L7* (HTTP/HTTPS), path/host 라우팅, WebSocket·gRPC. *EKS Ingress 표준*. **NLB**: *L4* (TCP/UDP), 극단 성능, static IP, TLS termination. EKS에선 *Ingress = ALB, Service type=LoadBalancer = NLB* 일반적. **AWS Load Balancer Controller**가 K8s 자원 → AWS LB 자동 생성.

### Q7. EKS의 Pod IP가 *VPC IP를 직접 사용*하는데 어떻게?
> **AWS VPC CNI**가 각 Pod에 *VPC subnet의 ENI 보조 IP 부여*. *VPC 안에서 라우팅 가능한 IP*. NAT 거치지 않고 *AWS service에 직접 접근*. **장점**: 성능, *AWS LB가 Pod IP 직접 가리킬 수 있음* (NLB target type IP). **단점**: VPC IP 소진 위험 — 큰 subnet 또는 *Custom Networking* + *prefix delegation*.

---

## 🔗 Cross-reference

- **네트워크 베이스** → [`network-basics/`](../network-basics/), 특히 [`network-basics/05-aws-vpc.md`](../network-basics/05-aws-vpc.md)
- **K8s Pod 네트워킹 (VPC CNI)** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md)
- **EKS deep** → [03-compute-and-eks.md](03-compute-and-eks.md)
- **NetworkPolicy 추가 (CNI에 위임)** → [`k8s-security-basics/02`](../k8s-security-basics/02-network-policy.md)
- **Terraform VPC module** → [`terraform-basics/03`](../terraform-basics/03-modules.md)

---

## 📝 3줄 요약

1. *VPC + Subnet (AZ별) + Route Table*이 베이스. **IGW (양방향) vs NAT (단방향)**. **VPC Endpoint로 NAT 비용 절감**.
2. **SG (stateful, 인스턴스, allow만)** vs **NACL (stateless, subnet, allow/deny, ephemeral port 주의)**. 운영은 SG 위주.
3. *VPC Peering* (메시) → *Transit Gateway* (hub-spoke). *PrivateLink*로 service 단위 노출. *AWS LB Controller*로 K8s ↔ ALB/NLB 통합.

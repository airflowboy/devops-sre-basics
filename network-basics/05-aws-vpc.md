# 05. AWS VPC — Ch 08·10에서 우리가 만든 것

> "VPC가 뭐냐"는 추상으로 배우면 항상 새로움. **우리가 실제 만든 것을 그림으로 보고 *왜 그렇게 만들었나* 짚는 게 가장 빠름**.

## 목차
- [VPC의 핵심 발상](#vpc의-핵심-발상)
- [VPC를 구성하는 7가지](#vpc를-구성하는-7가지)
- [퍼블릭/프라이빗 서브넷의 본질](#퍼블릭프라이빗-서브넷의-본질)
- [Ch 08 단일 AZ 학습용 → Ch 10 운영급 진화](#ch-08-단일-az-학습용--ch-10-운영급-진화)
- [멀티 AZ가 필요한 이유](#멀티-az가-필요한-이유)
- [VPC Peering / Endpoint / Transit Gateway (참고)](#vpc-peering--endpoint--transit-gateway-참고)
- [치트시트](#치트시트)

---

## VPC의 핵심 발상

> **AWS 안에 내 *전용* 가상 네트워크를 만든다.** 다른 AWS 고객과 격리.

비유: 같은 거대한 건물(AWS data center)에 여러 회사가 입주. 회사마다 *전용 층*(VPC)을 받음. 내 층 안에서는 자유롭게, 다른 회사 층과는 *명시적 허용* 시에만 통신.

### VPC 안에서
- 사설 IP (`10.0.0.0/16` 같은) 자유롭게 사용
- 인스턴스끼리 자유롭게 통신 (기본은)
- 외부 인터넷 접속은 *명시적으로 길을 만들어야* 가능

### VPC 밖과는
- IGW로 인터넷
- VPC Peering / Transit Gateway로 다른 VPC
- VPC Endpoint로 AWS 서비스 (S3·DynamoDB 등) — 인터넷 안 거치고

---

## VPC를 구성하는 7가지

운영급 VPC의 *필수 부속*. 우리 Ch 10 Phase A에 다 있음:

| # | 자원 | 역할 | 비유 |
|:-:|------|------|------|
| 1 | **VPC** | 전체 컨테이너 (`10.0.0.0/16` CIDR) | 단지 전체 |
| 2 | **Subnet** ×N | VPC를 작게 쪼갬, AZ별 분리 | 단지 안의 동 |
| 3 | **Internet Gateway (IGW)** | VPC ↔ 인터넷 (양방향) | 단지 정문 |
| 4 | **NAT Gateway** | 프라이빗 서브넷 outbound | 정문 옆 우편 대행 |
| 5 | **Route Table** | 어디로 갈지 결정표 | 우편 분류표 |
| 6 | **Security Group** | 인스턴스 단위 방화벽 | 호실 문지기 |
| 7 | **(선택) VPC Endpoint** | AWS 서비스 직접 접근 (S3 등) | 단지 내 우체국 |

→ 1~5만 있어도 *동작*. 6은 보안용. 7은 비용 최적화·보안 강화.

---

## 퍼블릭/프라이빗 서브넷의 본질

> **단어가 헷갈리지만 본질은 *라우트 테이블* 하나.**

```
서브넷 자체엔 "public/private" 속성 없음.
→ 그 서브넷의 라우트 테이블에 0.0.0.0/0 → IGW 있으면 = "퍼블릭"
→ 0.0.0.0/0 → NAT GW 있으면 = "프라이빗"
→ 0.0.0.0/0 룰 없으면 = "isolated" (외부 인터넷 X)
```

### 퍼블릭 서브넷 4요소 (반복!)
인스턴스가 인터넷되려면 *4가지 동시*:
1. VPC에 IGW attach
2. 서브넷의 라우트 테이블에 `0.0.0.0/0 → IGW`
3. 라우트 테이블이 *그 서브넷에 연결*
4. 인스턴스가 *공인 IP 가짐* (서브넷의 `map_public_ip_on_launch=true` 또는 EIP)

→ Ch 08-B에서 학습. Ch 10 Phase A에선 *멀티 AZ*로 확장.

### 프라이빗 서브넷 4요소
1. NAT GW가 *퍼블릭* 서브넷에 있음 + EIP 부여
2. 서브넷의 라우트 테이블에 `0.0.0.0/0 → NAT GW`
3. 라우트 테이블이 *그 서브넷에 연결*
4. (인스턴스는 공인 IP 없음 — 그게 "프라이빗"의 의미)

→ Ch 10 Phase A 추가 학습.

### 둘이 함께
```
퍼블릭 서브넷: ALB·NAT GW·bastion (외부 직접 노출 OK)
프라이빗 서브넷: 백엔드·DB·캐시 (외부에서 직접 못 들어옴, 응답만 가능)
```

---

## Ch 08 단일 AZ 학습용 → Ch 10 운영급 진화

```
┌─────────────────────────────────────────┐
│  Ch 08 Phase B (학습용)                  │
│                                         │
│  VPC 10.0.0.0/16                        │
│  └─ public-a (10.0.1.0/24)              │
│     └─ EC2 (공인 IP, SSH로 접속)         │
│                                         │
│  자원: 1 VPC + 1 서브넷 + 1 IGW + 1 SG  │
│  자원 합: 4~5개. 시간당 비용: $0         │
└─────────────────────────────────────────┘

                  ↓ "진짜 운영" 패턴으로 진화

┌─────────────────────────────────────────────────────────┐
│  Ch 10 Phase A (운영급)                                  │
│                                                         │
│  VPC 10.0.0.0/16                                        │
│  │                                                      │
│  ├─ public-a  (10.0.1.0/24)  ─┐                        │
│  ├─ public-c  (10.0.2.0/24)  ─┤  ← ALB·NAT GW 자리     │
│  ├─ private-a (10.0.11.0/24) ─┤                        │
│  └─ private-c (10.0.12.0/24) ─┘  ← EKS·DB·Redis 자리   │
│                                                         │
│  IGW 1개 + NAT GW 1개 (+EIP) + 2 RT + 4 RTA            │
│  자원 합: 14개. 시간당 비용: ~$0.045 (NAT)              │
│                                                         │
│  + EKS 자동발견 태그                                     │
│    public:  kubernetes.io/role/elb=1                    │
│    private: kubernetes.io/role/internal-elb=1           │
│    all:     kubernetes.io/cluster/<name>=shared         │
└─────────────────────────────────────────────────────────┘
```

### 무엇이 추가됐나
1. **멀티 AZ** — 1개 AZ → 2개 (a + c)
2. **퍼블릭/프라이빗 분리** — 인터넷 직접 노출 vs 격리
3. **NAT GW** — 프라이빗에서 outbound 가능
4. **EKS 호환 태그** — 매니지드 K8s가 자기 자원 자동 인식

### 왜 운영급?
- 한 AZ 장애에도 다른 AZ 살아있음 (가용성)
- DB·앱 노드가 *프라이빗*에 → 외부 공격 표면 ↓
- ALB가 2 AZ에 걸쳐 → ALB 자체도 HA

### Ch 10에서 결정한 단순화 (ADR-003·004)
- NAT GW 단일 (운영은 AZ별 ×2)
- 4 서브넷 (운영은 DB 전용 서브넷 분리해 6으로)
- → 학습 환경 비용·복잡도 ↓, 운영 시 *변수 하나로 전환* 가능 (ADR-003 Decision의 실천)

---

## 멀티 AZ가 필요한 이유

### AZ가 뭐길래
- **Availability Zone** = 한 리전 안의 *물리적으로 격리된* 데이터센터.
- `ap-northeast-2` (서울)은 a, b(없음), c, d 등 여러 AZ.
- AZ 간엔 *전용 회선*으로 빠르게 연결 (수 ms 지연), 하지만 *전기·냉방·네트워크가 독립*.

### 그래서 필요한 이유
- 한 AZ가 정전 / 광케이블 단선 → 그 AZ 자원 다 죽음
- 다른 AZ에 같은 서비스가 있으면 → 자동 페일오버 가능
- AWS SLA도 *멀티 AZ 배포* 가정

### "1 AZ로 충분 안 됨?"
- 학습/개발 환경엔 충분. 운영엔 부족.
- **AZ 장애는 진짜 일어남.** 2017 AWS S3 us-east-1 장애가 대표적.
- ALB도 *2 AZ 최소 요구* (헬스 체크 가능한 구조 위해).

### 운영 패턴
```
EKS 노드: AZ a, c에 자동 분산 (Managed Node Group)
RDS: Multi-AZ 활성화 (primary AZ-a, standby AZ-c, 장애 시 60초 페일오버)
ElastiCache: Replication group (primary + replica 다른 AZ)
S3: 자동 11개 AZ 복제 (사용자 신경 X)
```

---

## VPC Peering / Endpoint / Transit Gateway (참고)

운영 들어가면 만나게 되는 것들 — 캡스톤 범위는 아니지만 *큰 그림으로 알아두면 좋은*:

### VPC Peering
- 두 VPC를 *1:1*로 연결. 같은 회사 다른 VPC, 또는 파트너 VPC와.
- 양쪽 라우트 테이블에 상대 CIDR → peering connection 추가.
- 비용: 데이터 전송 비용만.
- **단점**: peering 수가 늘면 mesh가 복잡 (전 mesh = N×(N-1)/2 연결)

### Transit Gateway (TGW)
- *VPC들의 허브*. 모든 VPC가 TGW에 연결 → 서로 통신.
- Peering의 mesh 문제 해결.
- 비용: $0.05/hr + 데이터 → peering보다 비쌈. 대규모 multi-VPC에 가치.

### VPC Endpoint
- AWS 서비스(S3, DynamoDB, Secrets Manager 등)에 *인터넷 안 거치고* 접근.
- **Gateway Endpoint** (S3/DynamoDB만, 무료) vs **Interface Endpoint** (대부분 서비스, 시간당 과금).
- 효과:
  - 보안: 트래픽이 VPC 안에 머무름 (AWS 백본)
  - 비용: NAT GW 데이터 비용 ↓ (특히 ECR·S3 트래픽이 큼)
- → ADR-003에서 언급한 *NAT 비용 줄이는 정공법*.

### 캡스톤 적용 가능
- S3 / ECR / Secrets Manager / CloudWatch Logs → VPC Endpoint 도입 시 NAT 트래픽 ↓
- 시간 되면 Phase J 직전에 도입 검토 가치.

---

## 치트시트

### VPC 시작 시 체크리스트
- [ ] VPC CIDR (회사 다른 VPC·홈랩과 안 겹치게)
- [ ] AZ 최소 2개
- [ ] 서브넷: 각 AZ에 public + private (+옵션 db-private)
- [ ] IGW + VPC attach
- [ ] NAT GW (프라이빗 outbound 필요하면)
- [ ] 라우트 테이블 2개: public RT, private RT
- [ ] RT-서브넷 연결 (까먹기 쉬움)
- [ ] EKS 쓸 거면 *서브넷 태그* 미리 (Ch 10 Phase A 패턴)
- [ ] DB·Cache subnet group (필요하면)

### "안 통함" 디버깅 — VPC 측에서
```
1. 같은 VPC 안: ping 안 가면 → SG·NACL
2. 인터넷 → VPC: 안 들어오면 → 퍼블릭 서브넷 4요소
3. VPC → 인터넷 (outbound): 안 나가면 → 프라이빗 서브넷 NAT 4요소
4. VPC ↔ VPC: 안 통하면 → peering / TGW 라우트
```

### 비용 우선순위 (큰 것부터)
1. NAT GW ($32/월 + 데이터)
2. EKS Control Plane ($72/월) — Ch 10에서 신규
3. EC2 / RDS / ElastiCache (Free Tier 12개월)
4. ALB ($18/월 + LCU)
5. EIP unattached ($3.6/월 each — 정리 안 하면 새어나감)
6. VPC / Subnet / IGW / RT / SG = **무료**

### "destroy 안 하면 새어나가는 것"
- NAT GW (가장 큼)
- EIP (NAT에서 분리되면)
- ALB
- EC2 (terminate 안 하고 stop만 하면 EBS는 계속 과금)
- EBS 볼륨 (delete_on_termination=false면)

### 한 줄 정리
> **VPC는 단지, 서브넷은 동, IGW는 정문, NAT는 우편 대행.**
> **퍼블릭/프라이빗의 본질 = `0.0.0.0/0` 라우트가 IGW냐 NAT냐.**
> **운영급 = 멀티 AZ + 퍼블릭/프라이빗 분리 + NAT.**

---

## 다음 읽을 거
- [06-kubernetes.md](06-kubernetes.md) — K8s 네트워킹 (Pod IP / Service / Ingress / NetworkPolicy / CNI). Ch 03·04·09 복습 + 캡스톤 Phase B·D 준비.

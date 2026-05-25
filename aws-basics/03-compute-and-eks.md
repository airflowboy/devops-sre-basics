# 03 — Compute & EKS: EC2 · EKS · Fargate · Lambda · Karpenter

> **공식 근거:** [EC2 docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/), [EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/), [Karpenter](https://karpenter.sh/)
>
> **선행:** [`kubernetes-basics/`](../kubernetes-basics/), [01-iam-and-identity.md](01-iam-and-identity.md), [02-vpc-and-networking.md](02-vpc-and-networking.md)

---

## 🎯 한 문장

> **EC2 = *가상 머신*, EKS = *managed K8s* (control plane은 AWS, 노드는 사용자), Fargate = *serverless container* (노드 X), Lambda = *event-driven 함수* (런타임 매니지드). EKS 노드는 *Karpenter*로 동적 provisioning이 현대 표준.**

---

## 1. EC2 — *가상 머신*

### Instance type 명명 규칙
```
t3.medium
│ │  │
│ │  └─ size (nano/micro/small/medium/large/xlarge/...)
│ └──── generation (3세대)
└────── family (t=burstable, m=balanced, c=compute, r=memory, x=high-mem, ...)
```

### 주요 family
| Family | 용도 |
|---|---|
| **t** | Burstable (낮은 baseline + credit) — *web·dev* |
| **m** | Balanced (CPU·Mem 균형) — *일반 workload* |
| **c** | Compute optimized — *batch·HPC* |
| **r** | Memory optimized — *DB·캐시* |
| **x / u** | High memory — *SAP HANA* |
| **i / d** | Storage optimized — *NoSQL·HDFS* |
| **g / p** | GPU — *ML·rendering* |

### Spot Instance
- *AWS 여유 capacity* 사용
- *최대 90% 할인*
- 단점: *AWS가 *2분 notice로 회수*
- → *interruption-tolerant workload* (batch, CI runner, K8s worker)

### Reserved Instance / Savings Plan
- *1년/3년 약정* → *30~70% 할인*
- *Savings Plan*이 더 유연 (instance type 변경 OK)
- 자세히 [06-cost-and-finops.md](06-cost-and-finops.md)

### Auto Scaling Group (ASG)
- *N개 인스턴스 유지* (replicas 비슷)
- *Launch Template* 정의 (AMI, type, SG, user_data)
- *health check fail 시 자동 replace*
- *scaling policy* (CPU 기반, schedule 기반 등)

### user_data — *부팅 시 script*
```bash
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
```

→ *간단 부트스트랩*. 복잡하면 *Packer로 AMI 빌드* 또는 *Ansible*.

### EC2 Instance Profile
- *EC2가 IAM Role 받는 방식*
- *metadata service (IMDSv2)*로 *임시 credential 자동*

```bash
# EC2 안에서
aws sts get-caller-identity
# 자동으로 instance profile의 Role 사용
```

→ **정적 AccessKey 사용 X**. *EC2는 항상 instance profile*.

---

## 2. EKS — *Managed Kubernetes*

### Control Plane vs Data Plane

| | 누가 관리 | 무엇 |
|---|---|---|
| **Control Plane** | AWS | kube-apiserver, etcd, scheduler, controller-manager |
| **Data Plane** | 사용자 | 노드 (EC2 또는 Fargate), 그 위의 Pod |

→ EKS의 *가치 = control plane 운영 부담 0*. *upgrade, HA, backup도 AWS*.

### Cluster 생성
- *Console / Terraform / eksctl*
- *VPC 지정* (이미 있는 것 활용)
- *Endpoint access* — public / private / 둘 다
- *Encryption* — KMS for secrets

### Node Groups — *3가지 옵션*

**(1) Managed Node Group**
- AWS가 *EC2 launch template + ASG 관리*
- *upgrade·draining 자동* (PodDisruptionBudget 존중)
- 일반적 선택

**(2) Self-managed Node Group**
- 사용자가 *직접 ASG·launch template 관리*
- *더 자유롭지만 운영 부담 ↑*
- 특수한 경우 (custom AMI, GPU 드라이버 등)

**(3) Fargate**
- *노드 자체가 없음*
- *Pod 단위 serverless*
- *AWS가 Pod별 micro-VM 띄움*
- 비싸나 *운영 부담 0*, *capacity planning 불필요*

### EKS Add-ons — *AWS 관리 K8s 컴포넌트*
- **VPC CNI** — Pod 네트워킹
- **CoreDNS** — DNS
- **kube-proxy** — Service 라우팅
- **EBS CSI Driver** — EBS volume
- **EFS CSI Driver** — EFS volume
- *Version + 자동 upgrade*

→ 사용자가 *Helm으로 직접 install*도 가능하나 *Add-on이 더 편함*.

---

## 3. IRSA — K8s ServiceAccount → IAM Role

[`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), [`k8s-security-basics/03`](../k8s-security-basics/03-identity-and-rbac.md), [01-iam-and-identity.md](01-iam-and-identity.md) 참조.

### 핵심 flow
```
1. EKS cluster 생성 시 *OIDC Provider 활성화*
2. IAM Role 생성, trust policy에 *OIDC issuer + SA sub*
3. K8s ServiceAccount에 *role-arn annotation*
4. Pod 시작 시 EKS Admission이 *Projected SA Token + 환경변수 자동 주입*
5. AWS SDK가 *STS AssumeRoleWithWebIdentity* 호출
6. STS가 trust 검증 후 *임시 credentials* 발급
```

→ **정적 AccessKey 0개**. Pod별 *최소 권한 IAM Role*.

---

## 4. *Karpenter* — *동적 노드 provisioning*

### vs Cluster Autoscaler (CA)

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| 방식 | *고정 노드 그룹 (ASG) scale* | *동적으로 instance type 선택* |
| 응답 속도 | 분 단위 | *수십 초* |
| Instance type | ASG가 정함 | *Pending Pod 요구에 정확* |
| Consolidation | 약함 | *비효율 배치 자동 재배치* |
| Multi-AZ | ASG별 | *Pod별 zone 선택* |

### 동작
```
1. Pending Pod 감지 (자원 부족)
2. Provisioner / NodePool 정의 확인 (allowed instance types, zones, ...)
3. 가장 *적합한 instance* 선택 (Pod 요구 + spot 우선 등)
4. EC2 launch
5. 노드 join → Pod scheduling

+ Consolidation (주기적):
1. 노드별 utilization 검사
2. *empty 또는 underutilized 노드 발견*
3. *Pod 재배치 → 노드 삭제*
4. 비용 절감
```

### NodePool 예시
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: [spot, on-demand]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: [m6i.large, m6i.xlarge, m6i.2xlarge, c6i.large, c6i.xlarge]
      - key: topology.kubernetes.io/zone
        operator: In
        values: [ap-northeast-2a, ap-northeast-2c]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

→ Karpenter가 *위 조건 안에서 *가장 효율적인 instance* 선택*.

### 장점
- *Pod 요구 정확 매치* — 작은 Pod에 큰 노드 안 띄움
- *Spot 자동 활용* — 비용 절감
- *Consolidation* — 운영 중 효율 개선

### 단점
- *AWS 종속* (현재) — 다른 클라우드 버전 발전 중
- *학습 곡선* — CA보다 옵션 많음
- *Spot 회수 대응* 필요 (PDB + 정상 종료)

---

## 5. ECS / Fargate — *EKS 대안*

### ECS (Elastic Container Service)
- *AWS 자체 컨테이너 오케스트레이션*
- *K8s보다 단순* — AWS service들과 *깊은 통합*
- *Task Definition (Pod 비슷) + Service (Deployment 비슷)*

### Fargate
- *ECS와 EKS 둘 다에서 사용 가능*
- *노드 자체 없음* — *AWS가 micro-VM 띄움*
- *Pod/Task 단위 과금*
- 비싸나 *운영 부담 0, autoscaling 자동*

### ECS vs EKS

| | ECS | EKS |
|---|---|---|
| 표준 | AWS 자체 | K8s (CNCF) |
| 학습 곡선 | 낮음 | 높음 |
| AWS 통합 | 깊음 | 약간 |
| Vendor lock-in | 강 | 약 |
| 생태계 | AWS만 | 광범위 (Helm, Operator, ArgoCD, ...) |
| 추천 | *AWS-only + 단순* | *멀티 클라우드·복잡* |

→ 한국 대기업 대부분 *EKS* (multi-cloud 옵션 + 광범위 생태계).

---

## 6. Lambda — *Serverless Functions*

### 특징
- *event-driven* — HTTP, S3 event, SQS, EventBridge 등
- *Runtime 매니지드* — Node, Python, Java, Go, .NET, Ruby, 또는 Custom
- *0~수천 req/s 자동 스케일*
- *과금 = *요청 수 + 실행 시간 (ms 단위)**

### 한계
- *최대 15분 실행*
- *Memory 10GB까지*
- *Cold start* (특히 Java)
- *VPC 안 연결 시 ENI overhead*

### 흔한 활용
- HTTP API backend (+ API Gateway)
- 이벤트 처리 (S3 → Lambda → 처리)
- ETL
- CronJob 대체 (EventBridge schedule)
- 사내 자동화 (CloudWatch alarm → Lambda 자동 조치)

### Cold start 최적화
- *Provisioned Concurrency* — *항상 N개 warm* (비용 ↑)
- *Runtime 가벼운 것* (Go·Node) > 무거운 것 (Java·.NET)
- *VPC 회피* (가능하면) — ENI overhead

---

## 7. EKS 운영 *Best Practices*

### 1. *Multi-AZ Control Plane + Worker*
- AWS가 *control plane 3 AZ 자동*
- Worker도 *AZ별 node group* — Pod scheduling에 *topology spread*

### 2. *IRSA 모든 Pod*
- EC2 instance profile 권한 *너무 광범위* (모든 Pod이 받음)
- *Pod별 SA + IRSA*로 *최소 권한*

### 3. *NetworkPolicy + Calico/Cilium*
- AWS VPC CNI는 *기본 NetworkPolicy 미지원* (1.27+ alpha)
- Calico/Cilium add-on 또는 *Cilium 완전 교체*

### 4. *Karpenter or CA + Spot*
- *Spot으로 70% 비용 절감*
- *PDB 필수* — spot 회수 시 안전

### 5. *EKS Auto Mode* (2024+)
- AWS가 *node + add-on 자동 운영*
- *managed Karpenter + Fargate-like UX*
- 신규 환경에 *고려*

### 6. *Cluster autoscaler*는 *Karpenter로 교체* 추천
- Karpenter가 *모든 면에서 우월*

### 7. *Cilium 도입 고려*
- VPC CNI 한계 (IP 소진, NetworkPolicy 약함) → Cilium 완전 교체
- *eBPF로 성능·observability·L7 정책*

---

## 8. 자주 헷갈리는 것

### 8-1. *EC2 stop·start vs reboot*
- **reboot**: *같은 호스트* 유지
- **stop + start**: *다른 호스트로 이동 가능* (instance store 데이터 손실)
- *EBS 데이터는 유지*

### 8-2. *Instance store vs EBS*
- *Instance store*: *호스트 디스크 직접* — 빠름, 단 *stop 시 데이터 손실*
- *EBS*: *네트워크 storage* — 영구, AZ에 묶임

### 8-3. *Public IP vs DNS*
- public IP: stop·start 시 *바뀜*
- *Public DNS도 같이 바뀜*
- *EIP 사용 또는 Route 53*

### 8-4. *EKS upgrade*
- control plane → node → addon 순서
- *minor version 하나씩만* 가능 (1.27 → 1.28, 1.27 → 1.29 X)
- *deprecated API 사용 check 필수*

### 8-5. *Pod이 *외부 AWS API 호출 실패**
- IRSA 누락 — `aws sts get-caller-identity` 로 확인
- *VPC CNI 문제* — Pod IP가 VPC 안인가
- *VPC Endpoint or NAT 없음* — outbound 외부 통신 불가

### 8-6. *Karpenter consolidation으로 *production Pod evict**
- consolidation이 *underutilized 노드 정리*
- *critical Pod에 PDB 필수* — 동시 evict 한도

---

## 🎤 면접 빈출 Q&A

### Q1. EKS의 control plane은 누가 관리?
> **AWS가 관리** — kube-apiserver, etcd, scheduler, controller-manager. *upgrade·HA·backup 다 AWS*. **사용자는 data plane** (노드 + Pod). *EKS의 가치 = control plane 운영 부담 0*. cluster당 *시간당 비용* (현재 ~$0.10).

### Q2. EKS Node Group 3옵션?
> (1) **Managed Node Group**: AWS가 *EC2 launch template + ASG 관리*, *upgrade·draining 자동* — 일반적 선택. (2) **Self-managed**: *직접 ASG*, 자유 ↑ 부담 ↑ — custom AMI/GPU. (3) **Fargate**: *노드 없음*, *Pod 단위 serverless*, 비싸나 *운영 부담 0*. 현대는 *Karpenter (dynamic)*가 사실상 표준.

### Q3. Karpenter vs Cluster Autoscaler?
> **CA**: *고정 ASG scale*, 분 단위, instance type 고정. **Karpenter**: *동적 instance type 선택* (Pod 요구에 정확), *수십 초*, *Spot 자동 활용*, *Consolidation* (비효율 배치 재배치). **모든 면에서 Karpenter 우월** — 단 AWS 종속 (현재). 신규 EKS는 Karpenter 권장.

### Q4. IRSA 동작 + EKS와의 관계?
> EKS cluster에 *OIDC Provider 활성화* → IAM Role의 *trust policy*에 *OIDC issuer + SA sub claim* → K8s SA에 *role-arn annotation* → Pod 시작 시 EKS Admission이 *Projected SA Token + 환경변수 자동 주입* → AWS SDK가 *STS AssumeRoleWithWebIdentity* → 임시 credentials. **정적 AccessKey 0**. Pod별 *최소 권한 IAM Role*.

### Q5. Fargate가 *언제 좋고 언제 비싸나*?
> **좋은 경우**: *capacity planning 어려움 (burst)*, *운영 부담 최소화*, *짧은 batch job*. **비싼 경우**: *steady-state workload* (CPU·Mem 항상 사용) — EC2 + Reserved/Savings Plan이 *훨씬 저렴*. *cold start* 약간 (수십 초). EKS Fargate는 *Pod 단위*, ECS Fargate는 *Task 단위*.

### Q6. Spot Instance를 *안전하게* 사용?
> *2분 회수 notice* 대응: (1) **`SpotInterruptionHandler`** — Spot 회수 알림 → *그 노드 cordon + drain*. (2) **PodDisruptionBudget** — 동시 evict 한도. (3) *replica 충분히* + *multi-AZ multi-instance-type* (Karpenter가 자동). (4) *Critical workload (DB·stateful)는 On-Demand* — Spot은 stateless·interruption-tolerant만.

### Q7. EKS upgrade 절차?
> *control plane → node → addon* 순서. **minor version 하나씩** (1.27 → 1.28, jump 불가). **deprecated API 사용 check 필수** — `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis`. *Pluto / kubent* 같은 도구로 *manifest의 deprecated API 검출*. ArgoCD라면 *manifest 업데이트 → re-sync* → 안전.

---

## 🔗 Cross-reference

- **K8s control plane/data plane 깊이** → [`kubernetes-basics/01`](../kubernetes-basics/01-architecture.md)
- **IRSA 깊이** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), [`k8s-security-basics/03`](../k8s-security-basics/03-identity-and-rbac.md)
- **K8s scheduling (Karpenter와 연결)** → [`kubernetes-basics/05`](../kubernetes-basics/05-scheduling.md)
- **K8s networking (VPC CNI)** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md)
- **IAM Role + Trust Policy** → [01-iam-and-identity.md](01-iam-and-identity.md)
- **VPC + LB** → [02-vpc-and-networking.md](02-vpc-and-networking.md)
- **Terraform으로 EKS 자원** → [`terraform-basics/03`](../terraform-basics/03-modules.md)

---

## 📝 3줄 요약

1. *EKS = managed K8s — control plane 운영 부담 0*. **Managed Node Group / Self-managed / Fargate** 3 옵션. **Karpenter**가 현대 노드 provisioning 표준 (CA보다 우월).
2. **IRSA**로 *Pod별 최소 권한 IAM Role + 정적 키 0*. EKS Add-ons (VPC CNI / CoreDNS / kube-proxy / EBS CSI)로 *기본 컴포넌트 매니지드*.
3. *Spot + Karpenter + PDB*로 *비용 70% 절감 + 안전*. EKS upgrade는 *minor 하나씩 + deprecated API check 필수*.

# 06 — Cost & FinOps: Cost Explorer · Budget · Savings Plan · Spot

> **공식 근거:** [AWS Cost Management](https://docs.aws.amazon.com/cost-management/), [FinOps Foundation](https://www.finops.org/framework/)

---

## 🎯 한 문장

> **FinOps = *운영의 *cost 가시성 + 책임 + 최적화*. *Cost Explorer (분석) + Budget (알람) + Tag (분류) + Savings Plan/Reserved (할인) + Spot (90% 할인)* 조합. **opex의 30~50% 절감 가능**. *문화*가 도구보다 중요.**

---

## 1. FinOps의 *3 phase*

```
Inform (가시성)        Optimize (최적화)        Operate (운영)
├ Cost Explorer       ├ Right-sizing          ├ Anomaly detection
├ Budget              ├ Savings Plan          ├ Tag enforcement
├ Tag 분석            ├ Reserved              ├ Continuous review
├ Showback/Chargeback ├ Spot                  ├ FinOps culture
                      ├ Auto-scaling
                      ├ Storage tier
```

→ *Inform → Optimize → Operate 반복*. *시작은 가시성*.

---

## 2. *비용 분류* — Tag

### Tag 전략
- *모든 자원에 *최소 tag 강제**:
  - `Environment` (prod/stg/dev)
  - `Team` 또는 `Owner`
  - `Service` 또는 `Application`
  - `CostCenter`

### Tag Enforcement
- **SCP** (Organization 단위) — *tag 없는 자원 생성 거부*
- **IAM Policy** — 특정 tag 요구
- **AWS Config rule** — 위반 자원 감지·알람

### Tag 활용
- **Cost Allocation Tag** 활성 → *Cost Explorer에서 tag별 비용 breakdown*
- *Showback*: 각 팀에 *비용 보고* (의식)
- *Chargeback*: 각 팀에 *비용 청구* (책임)

---

## 3. Cost Explorer — *비용 분석 도구*

### 기능
- *시계열 비용 추이*
- *Service / Region / Tag / Account 별 breakdown*
- *Forecasting* (다음 N개월 예측)
- *Anomaly detection*

### Group By
- Service (가장 흔함)
- Tag (Environment, Team 등)
- Account (Organization)
- Region
- Instance Type

### Filter
- *period* (last 7d, last month, custom)
- *service*
- *linked account*
- *tag value*

→ *월 1회 회의*에서 *팀별 cost review*가 운영 표준.

---

## 4. AWS Budgets — *알람*

### 종류
- **Cost Budget**: *지출 한도*
- **Usage Budget**: *사용량* (예: EC2 시간)
- **Reservation Budget**: RI/SP 사용률
- **Savings Plans Budget**: SP coverage

### 알람
- *80% 도달 → Email/SNS*
- *예상 100% 초과 → 알람*
- **Action**: *Budget actions* — *IAM policy 자동 적용* (예: deny 새 자원 생성)

```yaml
Budget:
  Amount: 10000
  Currency: USD
  Period: MONTHLY
Notifications:
- Threshold: 80%
  ComparisonOperator: GREATER_THAN
  Subscribers: [team@example.com, slack-webhook]
- Threshold: 100%
  ComparisonOperator: FORECASTED
  Subscribers: [...]
```

→ *발견하면 늦음* — *예측 알람 활용*.

---

## 5. *EC2 비용 모델 4가지*

| 모델 | 할인 | 약정 | 유연성 |
|---|---|---|---|
| **On-Demand** | 0 | 없음 | 최대 |
| **Reserved Instance (RI)** | 30~70% | 1년/3년 | 적음 (instance type 고정) |
| **Savings Plan** | 30~70% | 1년/3년 | 중간 (compute SP는 region·family 변경 OK) |
| **Spot** | 최대 90% | 없음 | 회수 risk |

### Reserved Instance — *옛 방식*
- *Standard RI*: instance type 변경 X
- *Convertible RI*: 같은 family 안 변경 OK
- *Scheduled RI*: 특정 시간대만

### Savings Plan — *권장*
- **Compute SP**: *instance family·region·tenancy 자유*
  - 가장 유연. Lambda·Fargate 포함.
- **EC2 Instance SP**: 특정 family만, 더 큰 할인
- **SageMaker SP**: ML 용

→ *모르겠으면 Compute Savings Plan*.

### 권장 전략
- *Baseline (변하지 않는 사용량)* → **Savings Plan 1년 또는 3년**
- *Burst (변동)* → **On-Demand**
- *Interruptible* → **Spot**

→ 보통 *baseline 60-70% SP + 나머지 OD/Spot* 조합.

---

## 6. Spot Instance — *90% 할인*

### 특징
- *AWS 여유 capacity*
- *최대 90% 할인*
- *2분 notice로 회수*

### Use case
- *Stateless workload* — web, API
- *Batch job* — CI runner, video encoding
- *K8s worker* (PDB + graceful 처리)
- *Test/dev*

### 안 쓰는 case
- DB·stateful
- Long-running (24h+) — 회수 확률 ↑
- Critical real-time

### Best practices
- *Multi-instance type + Multi-AZ* — 회수 시 다른 type/AZ 활용
- **Spot Fleet** 또는 **Karpenter** — 자동 diversification
- **SpotInterruptionHandler** (K8s) — 회수 시 *graceful node drain*
- *checkpoint·resume* — 작업 도중 회수 대응

### 비용 절감 예
```
On-Demand m5.large: $0.096/hour
Spot m5.large: $0.029/hour (70% 할인)
3개 노드 × 24h × 30d = $69 vs $209/month
```

---

## 7. *Right-sizing* — *과도한 자원 줄이기*

### Compute Optimizer
- *AWS의 ML 기반 추천*
- *EC2 instance 사용률 분석 → 더 작은 type 추천*
- *EBS volume IOPS/throughput 분석 → gp2 → gp3 추천*
- *Lambda memory 추천*

### 흔한 패턴
- **CPU 평균 10% → 한 size down**
- **Memory 평균 30% → 다른 family**
- **gp2 → gp3** (저렴 + 더 빠름)

### K8s에서
- **VPA (Vertical Pod Autoscaler)** — Pod 자원 추천·자동 조정
- **Goldilocks** — VPA 결과 시각화
- **Karpenter consolidation** — 노드 단위 최적화

---

## 8. *Storage 비용 최적화*

### S3
- **Lifecycle**: 30d → IA, 90d → Glacier, 365d → delete
- **Intelligent-Tiering**: AWS가 자동 tier (모니터링 무료, transition만 비용)
- **versioning 정리**: 옛 version 자동 삭제
- **VPC Endpoint**: NAT 비용 회피

### EBS
- **gp2 → gp3** (대부분 더 저렴 + 빠름)
- **Snapshot 정리**: *옛 snapshot lifecycle*
- **Volume 사용률 모니터링**: 30% 미만 → resize

### RDS
- **Reserved Instance / Aurora I/O optimized**
- **Storage auto-scaling**
- **dev/stg는 *야간 stop***

---

## 9. *Network 비용 — *조용한 폭증**

### 가장 비싼 항목
- **Inter-AZ traffic**: $0.01/GB
- **Internet egress**: $0.09/GB (대량은 할인)
- **NAT Gateway processing**: $0.045/GB
- **CloudFront**: edge 비용

### 절감
- **VPC Endpoint** — S3·DynamoDB·ECR 등 *NAT 비용 회피*
- **Single-AZ 배치** (가능하면) — inter-AZ 회피
- **CloudFront** — egress 비용 절감 (CDN)
- **Direct Connect** — on-prem ↔ AWS 대량 transfer

### Egress 추적
- *Cost Explorer + Usage Type filter*: `DataTransfer-Out-Bytes`
- *어느 service / region이 *원인*인지 식별*

---

## 10. 자주 헷갈리는 것

### 10-1. *Reserved Instance vs Savings Plan*
- RI = *instance type 고정* (옛)
- SP = *유연 (compute SP)* — *family 변경 OK*
- **새로 살 거면 Savings Plan**

### 10-2. *Tag 안 한 자원이 비용 분석 안 됨*
- *un-tagged*도 *Cost Explorer에서 별도 표시*
- 운영 초기 *tag enforcement 정책 필수*

### 10-3. *Budget 알람이 *늦음**
- *AWS billing data는 *24시간 지연*
- *예측 알람* 활용 (forecasted)
- *real-time은 *Cost Anomaly Detection**

### 10-4. *Free tier 12개월 후 *과금 시작**
- 잊고 *EC2 t2.micro 켜둠* → 과금
- *Budget 알람 필수*

### 10-5. *Cross-region transfer가 *비용 큰지 모름**
- Inter-region: $0.02/GB
- 모르고 cross-region 호출 → 대량 비용
- *VPC Peering·Transit Gateway transfer*도 마찬가지

### 10-6. *Idle resource (잊혀진 자원)*
- 옛 EBS volume·snapshot·EIP (unattached)
- *AWS Trusted Advisor* 또는 *cost-anomaly-detection*으로 식별
- 매월 정리

---

## 🎤 면접 빈출 Q&A

### Q1. FinOps의 *3 phase*는?
> **Inform** (가시성): Cost Explorer, Budget, Tag로 *어디 얼마 쓰는지*. **Optimize**: Right-sizing, Savings Plan, Reserved, Spot, auto-scaling, storage tier로 *효율화*. **Operate**: Anomaly detection, tag enforcement, continuous review, *FinOps culture* — 매월 회의·책임 분담. *시작은 가시성*, 가장 어려운 건 *culture*.

### Q2. Reserved Instance vs Savings Plan?
> **RI** (옛): *instance type 고정*. Standard/Convertible/Scheduled. **Savings Plan** (권장): *유연*. **Compute SP** = *family·region·tenancy 자유*, Lambda·Fargate 포함. **EC2 Instance SP** = 특정 family만, 더 큰 할인. 새로 살 거면 *Compute SP*. *baseline은 SP 1-3년, burst는 OD, interruptible은 Spot*.

### Q3. Spot Instance를 *안전하게* 사용?
> *2분 회수 notice*. **Multi-instance type + Multi-AZ diversification** (Spot Fleet, Karpenter 자동). **SpotInterruptionHandler** (K8s) — 회수 알림 → *graceful drain*. **PDB** 필수 — 동시 evict 한도. *stateless workload만*: web, CI runner, batch. *DB·stateful은 On-Demand*. Use case 적절하면 *70-90% 절감*.

### Q4. EC2 Right-sizing 어떻게?
> **AWS Compute Optimizer** — ML 기반 추천 (instance type, EBS, Lambda memory). 패턴: *CPU 평균 10% → 한 size down*, *memory 30% → 다른 family*, **gp2 → gp3** (저렴 + 빠름). K8s는 **VPA** (Vertical Pod Autoscaler) + **Karpenter consolidation** (노드 단위). 주기적 review 필수.

### Q5. NAT Gateway 비용 *왜 폭증*하나? 절감?
> *시간당 ($0.045/h) + processing ($0.045/GB) + transfer*. 모든 private subnet outbound가 NAT 거침. **AWS service (S3, DynamoDB) 호출도 NAT 비용** — *모르는 경우 많음*. **해결**: **VPC Endpoint** — *Gateway Endpoint (S3, DynamoDB) 무료*, *Interface Endpoint* (ECR, Secrets Manager, KMS, STS, SSM) 시간당 비용이나 *NAT보다 저렴*.

### Q6. S3 비용 최적화?
> **Lifecycle policy** — 30d → IA, 90d → Glacier, 365d → delete. **Intelligent-Tiering** — AWS 자동 tier (접근 패턴 모름 시). **versioning 정리** — 옛 version 자동 삭제 (Lifecycle expiredObjectDeleteMarker). **VPC Endpoint** — NAT 회피. **S3 Bucket Key** — SSE-KMS *KMS API 호출 99% 감소*.

### Q7. *Tag 강제*가 *왜 중요*?
> 비용 분석의 *베이스*. Un-tagged 자원은 *cost breakdown 안 됨* — 어느 팀·service의 비용인지 모름. **SCP로 tag 없는 자원 생성 거부**, **IAM Policy로 특정 tag 요구**, **AWS Config rule로 위반 감지·알람**. *Showback/Chargeback*의 전제. **운영 초기 enforcement** — 늦게 도입하면 *기존 자원 다 untagged*.

---

## 🔗 Cross-reference

- **IAM (SCP)** → [01-iam-and-identity.md](01-iam-and-identity.md)
- **VPC (NAT, VPC Endpoint)** → [02-vpc-and-networking.md](02-vpc-and-networking.md)
- **Compute (EC2, Spot, Karpenter)** → [03-compute-and-eks.md](03-compute-and-eks.md)
- **Storage (S3, EBS gp3)** → [04-storage-and-database.md](04-storage-and-database.md)
- **K8s VPA·Karpenter** → [`kubernetes-basics/05`](../kubernetes-basics/05-scheduling.md), [03-compute-and-eks.md](03-compute-and-eks.md)
- **Terraform on-demand apply/destroy 패턴** → [`terraform-basics/02`](../terraform-basics/02-state-and-backends.md), [`terraform-basics/05`](../terraform-basics/05-workflow-and-best-practices.md)

---

## 📝 3줄 요약

1. *FinOps 3 phase = Inform (가시성 — Cost Explorer/Budget/Tag) → Optimize (SP/Spot/Right-sizing) → Operate (Anomaly·문화)*. **opex 30~50% 절감 가능**.
2. *Compute Savings Plan = 권장 (유연 + 할인)*, *Spot = 90% 할인 (stateless + PDB)*, *RI는 옛 방식*. *Baseline SP + Burst OD + Interruptible Spot* 조합.
3. *NAT Gateway·Inter-AZ·Internet egress가 *조용한 비용**. **VPC Endpoint로 NAT 회피**. *Tag enforcement는 운영 초기 필수*. **gp2 → gp3는 거의 항상 win**.

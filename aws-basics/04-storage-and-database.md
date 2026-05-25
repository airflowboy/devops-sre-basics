# 04 — Storage & Database: S3 · EBS · EFS · RDS · Aurora · DynamoDB

> **공식 근거:** [S3 docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/), [RDS docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/), [DynamoDB docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)

---

## 🎯 한 문장

> **AWS 데이터 *4 분류*: *Object* (S3 — unlimited, eventually consistent → 2020년부터 strongly consistent), *Block* (EBS — AZ에 묶임), *File* (EFS — NFS, multi-AZ), *DB* (RDS·Aurora·DynamoDB). 선택은 *접근 패턴 + 일관성 + 비용* 균형.**

---

## 1. S3 — *Object Storage*

### 특징
- *Unlimited capacity*
- *Region service* (단 bucket 이름은 *global 유일*)
- *Strong read-after-write consistency* (2020년 12월+)
- *Durability 99.999999999%* (11 nines) — *AZ 3개에 자동 replication*
- *Availability 99.99%*

### Storage Class

| Class | 용도 | 비용 (대략 GB-month) |
|---|---|---|
| **Standard** | 자주 접근 | $0.023 |
| **Intelligent-Tiering** | 접근 패턴 모름 — AWS 자동 tier | 약간 ↑ |
| **Standard-IA** | 자주 안 봄, 빠른 접근 필요 | $0.0125 |
| **One Zone-IA** | 단일 AZ — 비싼 backup의 backup | $0.01 |
| **Glacier Instant** | 즉시 접근, 옛 데이터 | $0.004 |
| **Glacier Flexible** | 분~시간 retrieval | $0.0036 |
| **Glacier Deep Archive** | 12시간 retrieval | $0.00099 |

→ *Lifecycle policy*로 *자동 tier 이동*. 1년 안 본 데이터 → Glacier 등.

### Versioning
```
PUT object → version 1
PUT (overwrite) → version 2 (옛 version 1 보존)
DELETE → delete marker (실제 데이터 보존)
```

- *삭제 사고 복구*
- *MFA Delete* — version 영구 삭제 시 MFA 필수
- *비용*: 모든 version의 storage

### Lifecycle Rule
```yaml
# 30일 후 IA, 90일 후 Glacier, 365일 후 삭제
Rules:
- Transition:
  - Days: 30
    StorageClass: STANDARD_IA
  - Days: 90
    StorageClass: GLACIER
  Expiration:
    Days: 365
```

### Cross-Region Replication (CRR)
- *다른 region에 자동 복제*
- *durability + DR*
- 양쪽 *versioning 활성화 필수*

### S3 Event
- Object 생성/삭제 → *Lambda·SQS·SNS·EventBridge trigger*
- *데이터 파이프라인의 흔한 시작점*

### Bucket Policy + IAM
- Bucket Policy = *resource-based* — *누가 이 bucket에 무엇*
- IAM Policy = *identity-based* — *그 user/role이 어느 자원*
- *Cross-account는 양쪽 모두 Allow*

### Public Access Block — *실수 방지*
- *기본은 모든 public 접근 차단* (account 단위)
- 그래도 public 필요하면 *명시적 unblock*
- **데이터 유출 사고 #1 원인 = public bucket** → AWS가 *기본 차단*으로 강화

---

## 2. S3 운영 모범

### 1. *Encryption at rest 강제*
- SSE-S3 (AWS 키), SSE-KMS (KMS 키 — audit), SSE-C (customer 키)
- *Bucket Policy로 *암호화 안 된 PUT 거부**

### 2. *Versioning + MFA Delete*
- 사고 복구 + 변조 방지

### 3. *Lifecycle로 비용 자동 최적화*
- 30 days → IA, 90 → Glacier, 365 → 삭제

### 4. *Access logging + CloudTrail Data Events*
- 누가 무엇 read·write 추적

### 5. *Public Access Block* 활성
- 모든 account-level + bucket-level

### 6. *VPC Endpoint (Gateway)*
- 무료. *NAT 비용 회피 + 보안*

### 7. *S3 Object Lock* (WORM)
- *Compliance mode*: *지정 기간 동안 수정·삭제 불가* (root도 불가)
- 컴플라이언스 (HIPAA, SEC 17a-4 등)

---

## 3. EBS — *Block Storage*

### 특징
- *AZ에 묶임* — *그 AZ EC2에만 attach 가능*
- *Single instance attach (기본)*, *Multi-Attach (특정 type만)*
- *snapshot으로 백업* (S3 저장, region-wide)

### Volume Type

| Type | 용도 | 특징 |
|---|---|---|
| **gp3** | 일반 (default 권장) | IOPS·Throughput 독립 설정 |
| **gp2** | 옛 default | 크기 비례 IOPS |
| **io2 / io2 Block Express** | 고성능 DB | 256K IOPS, 99.999% durability |
| **st1** | Throughput-optimized HDD | big data, 로그 |
| **sc1** | Cold HDD | infrequent access |

### gp3 vs gp2 (가장 흔한 결정)
- gp3: *기본 3000 IOPS + 125 MB/s*, *추가 IOPS·Throughput만 결제*
- gp2: *크기에 비례 IOPS* (1GB당 3 IOPS, 100~16000)
- *gp3가 더 저렴 + 더 유연*. *gp2 사용 중이면 gp3로 마이그레이션 권장*.

### Snapshot
- *증분 백업* (변경된 block만)
- *S3에 저장*
- *다른 AZ·region 복사 가능*
- *AMI*의 베이스 (root volume snapshot)

---

## 4. EFS — *Managed NFS*

### 특징
- *Multi-AZ* (POSIX file system)
- *동시 수천 EC2 마운트*
- *Pay-per-use* (사용 GB만, capacity planning 0)
- *Throughput mode*: Bursting (default) / Provisioned / Elastic

### vs EBS
| | EBS | EFS |
|---|---|---|
| Type | Block | File (NFS) |
| AZ | 묶임 | Multi-AZ |
| Attach | 보통 single | Multi |
| Performance | 매우 빠름 | 보통 |
| 비용 | 저렴 | 비쌈 (3-5x EBS) |
| 용도 | DB, single instance | 공유 fs, content management |

### K8s에서 사용
- **EBS CSI Driver** — Pod에 EBS volume mount (*RWO만*)
- **EFS CSI Driver** — Pod에 EFS mount (*RWX*)

→ K8s에서 *여러 Pod 공유*가 필요하면 EFS. *DB·single Pod*는 EBS.

---

## 5. RDS — *Managed Relational DB*

### 지원 엔진
- **Aurora** (MySQL/PostgreSQL 호환) — AWS 자체 *최강*
- MySQL, MariaDB, PostgreSQL
- Oracle, SQL Server (license 별도)

### Multi-AZ
- *Primary AZ-a + Standby AZ-c (synchronous replication)*
- *Primary fail 시 60~120초 안 standby로 자동 failover*
- *standby는 read 불가* (대기만)

### Read Replica
- *비동기 replication*
- *read traffic 분산*
- *region 간 가능* (cross-region replica)
- *최대 5개 (MySQL/MariaDB)*

### Multi-AZ vs Read Replica
| | Multi-AZ | Read Replica |
|---|---|---|
| 목적 | *HA* (failover) | *read scale* |
| Replication | Synchronous | Asynchronous |
| Read 가능? | Standby는 X | ✓ |
| Failover | 자동 | 수동 (replica를 primary로) |
| 비용 | 인스턴스 2배 | 인스턴스 N배 |

→ *진짜 HA + read 분산*은 *Multi-AZ + Read Replica 조합*.

### Backup
- *자동 backup* (기본 7일, 최대 35일)
- *Snapshot* — 수동, 영구 (delete 안 하면)
- *Point-in-Time Recovery (PITR)* — 5분 단위 복구

### Performance Insights
- *DB 부하 시각화*
- *느린 query 식별*
- *대시보드에서 즉시 확인*

---

## 6. Aurora — *AWS의 최강 RDS*

### 발상
- *MySQL·PostgreSQL 호환 + AWS가 reimagine한 storage layer*
- *Storage가 6 copy across 3 AZ* — 자동
- *Compute (DB instance)와 Storage 분리*
- *Compute 추가만으로 read replica 가능* — 데이터 복제 X

### 성능
- MySQL의 *5x*, PostgreSQL의 *3x*
- *최대 128TB*
- *Replica 15개*

### Aurora Serverless v2
- *Capacity (ACU)를 자동 조정* — 0.5~256
- *burst 트래픽에 자동 scale*
- *idle 시 cost ↓* (단 cold start 약간)

### Aurora Global Database
- *Cross-region replication* (1초 미만)
- *DR + global read*
- *Failover (region 단위)*

### Aurora vs RDS
| | RDS | Aurora |
|---|---|---|
| Storage | EBS | *Aurora distributed storage* |
| Replication | binlog | *storage-level* (빠름) |
| Replica 수 | 5 | 15 |
| Failover | 60-120s | *수십 초* |
| 비용 | 저렴 | 약간 비쌈 |
| 성능 | 보통 | *5x MySQL* |

→ *prod는 거의 Aurora*. RDS는 *옛 호환·특수 엔진*.

---

## 7. DynamoDB — *NoSQL*

### 특징
- *Fully managed, serverless*
- *Single-digit ms latency at any scale*
- *Auto-scaling* (또는 *On-Demand*)
- *Multi-region (Global Tables)*

### Data Model
- **Table** ↔ Item (row) ↔ Attribute (column)
- *Partition Key* (필수) — *데이터 분산의 기준*
- *Sort Key* (옵션) — *같은 partition 안 정렬*

### Read Consistency
- **Eventually Consistent** (기본) — *최대 1초 stale*
- **Strongly Consistent** — 명시 옵션 (read 2x 비용)
- **Transactions** — *여러 item 원자적 (ACID)* — 2x 비용

### Capacity Mode
- **Provisioned**: RCU/WCU 명시 + auto-scaling
- **On-Demand**: *요청별 과금*, capacity planning 0

### 흔한 함정
- **Hot partition** — *한 partition key에 트래픽 집중* → throttle
  - 해결: *random suffix* 또는 *재설계*
- **Item size 400KB 한도**
- **Query는 *partition key 필수***, 그 외엔 *Scan (느림·비쌈)*
- **GSI** (Global Secondary Index) — 다른 *partition/sort key*로 query

### Use case
- *Game leaderboard*
- *Session store*
- *IoT 데이터*
- *Real-time 분석*

→ *Schema-flexible + 극단 scale* 필요. *Complex JOIN 필요*는 RDS.

---

## 8. ElastiCache — *Managed Redis/Memcached*

### Redis
- *In-memory data store*
- *Pub/Sub, sorted sets, streams 등 풍부*
- *Replication + clustering*

### Memcached
- *단순 KV*
- *multi-threading*
- *옛 시스템 호환*

→ 현대는 *거의 Redis*. Memcached는 *옛 호환*.

### Cluster Mode
- **Disabled**: 1 master + N replica — 작은 데이터
- **Enabled**: *shard 별 master+replica* — *큰 데이터·고 throughput*

### Use case
- DB query cache (write-through, lazy load)
- Session store
- Rate limiting (sorted set, atomic increment)
- Leaderboard (sorted set)
- Pub/Sub

---

## 9. 자주 헷갈리는 것

### 9-1. *S3 *eventual vs strong consistency**
- 옛: *PUT 후 GET이 옛 값 가능*
- 2020년 12월+: *Strong read-after-write consistency* (자동)
- 단 *concurrent PUT*은 여전히 *last-write-wins*

### 9-2. *EBS는 AZ에 묶임*
- 다른 AZ EC2에 mount X
- 해결: *snapshot → 다른 AZ에서 새 volume*

### 9-3. *RDS Multi-AZ standby는 read 불가*
- standby = *failover 용*만
- read scale은 *Read Replica*

### 9-4. *DynamoDB hot partition*
- 모든 트래픽이 한 partition → throttle
- partition key 설계 — *균등 분포 보장*
- *user_id*는 OK (다양), *date*는 위험 (오늘만 hot)

### 9-5. *S3 versioning 비용*
- 모든 version의 *storage* 비용
- *delete marker도 마찬가지*
- *Lifecycle로 옛 version 자동 정리*

### 9-6. *Aurora vs RDS 가격*
- Aurora 약간 *비싸나* — *성능·HA·storage cost가 보통 더 좋아 net positive*
- 작은 환경엔 *RDS PostgreSQL*도 충분

---

## 🎤 면접 빈출 Q&A

### Q1. S3의 storage class와 lifecycle?
> **Standard** (자주 접근) → **Standard-IA** (자주 안 봄) → **Glacier** (옛 데이터) → **Glacier Deep Archive** (12h retrieval). **Lifecycle rule**로 *자동 tier 이동* + *expiration*. 예: 30일 → IA, 90일 → Glacier, 365일 → 삭제. **Intelligent-Tiering**은 *AWS가 접근 패턴 분석해 자동 tier*.

### Q2. S3 versioning + MFA Delete의 가치?
> Versioning = *삭제 사고 복구* (모든 version 보존). **MFA Delete**: *version 영구 삭제 시 MFA 필수* → *비인가 삭제 차단*. 비용: 모든 version storage. *Lifecycle로 옛 version 자동 정리* 필수. **데이터 유출 사고 #1 원인 = public bucket** → AWS *Public Access Block* 기본 활성.

### Q3. RDS Multi-AZ vs Read Replica?
> **Multi-AZ**: *HA용*. Primary AZ-a + Standby AZ-c, *synchronous replication*, *standby는 read 불가*. Primary fail 시 *60-120초 자동 failover*. **Read Replica**: *read scale용*. *asynchronous replication*, *read 가능*, *수동 promote*. **진짜 HA + read 분산은 Multi-AZ + Read Replica 조합**.

### Q4. Aurora가 *왜 RDS보다 좋은가*?
> *MySQL/PostgreSQL 호환 + AWS reimagine한 storage*. **Storage 6 copy across 3 AZ 자동**, *compute와 storage 분리*. 성능 **MySQL 5x / PostgreSQL 3x**, *replica 15개*, failover *수십 초*. **Aurora Serverless v2**로 *capacity 자동 조정*. **Global Database**로 cross-region replication. *prod는 거의 Aurora* — RDS는 옛 호환·특수 엔진만.

### Q5. DynamoDB의 *hot partition* 문제?
> 모든 트래픽이 *한 partition key에 집중* → throttle (capacity 한도 초과). **원인**: partition key 설계 잘못 (예: *오늘 date를 key*로 — 오늘만 hot). **해결**: *균등 분포 partition key* (user_id 같이), *write-sharding* (random suffix 추가), *재설계*. **GSI** 활용으로 *다른 query pattern* 지원.

### Q6. EBS와 EFS 차이? 언제 무엇?
> **EBS**: *block storage*, *AZ에 묶임*, *single instance attach* (보통), *매우 빠름*. DB·single Pod에 적합. **EFS**: *NFS file system*, *multi-AZ*, *수천 동시 mount*, *비싸나 (3-5x EBS)*. K8s RWX (여러 Pod 공유), content management. **EBS CSI driver = RWO, EFS CSI driver = RWX**.

### Q7. ElastiCache Redis의 활용?
> *In-memory data store* — single-digit ms latency. **Use case**: (1) **DB cache** (write-through, lazy load), (2) **Session store**, (3) **Rate limiting** (sorted set + atomic increment), (4) **Leaderboard** (sorted set), (5) **Pub/Sub**. *Cluster Mode Enabled*로 *shard별 master+replica* — 큰 데이터·고 throughput. *Memcached는 옛 단순 KV* — 현대는 거의 Redis.

---

## 🔗 Cross-reference

- **K8s PV/PVC/StorageClass/CSI** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md)
- **Docker volume vs K8s PV** → [`docker-basics/04`](../docker-basics/04-volumes-and-storage.md)
- **IAM policy (S3 bucket policy)** → [01-iam-and-identity.md](01-iam-and-identity.md)
- **VPC endpoint (S3·DynamoDB)** → [02-vpc-and-networking.md](02-vpc-and-networking.md)
- **KMS encryption (S3·EBS·RDS)** → [05-secrets-and-kms.md](05-secrets-and-kms.md)
- **Terraform S3·RDS module** → [`terraform-basics/03`](../terraform-basics/03-modules.md)

---

## 📝 3줄 요약

1. *S3 (object, unlimited, lifecycle) / EBS (block, AZ 묶임) / EFS (NFS, multi-AZ, RWX)*. **Public Access Block + versioning + KMS encryption** 운영 필수.
2. *RDS Multi-AZ (HA) vs Read Replica (scale)*. **Aurora가 prod 표준** — storage 6-copy + 5x MySQL 성능 + 빠른 failover. Aurora Serverless v2로 burst.
3. *DynamoDB = serverless NoSQL, single-digit ms*. **Hot partition 주의 — 균등 분포 partition key**. ElastiCache Redis는 *cache + session + rate limit + leaderboard*.

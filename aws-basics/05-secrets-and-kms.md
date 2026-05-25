# 05 — Secrets & KMS: envelope encryption · Secrets Manager · Parameter Store

> **공식 근거:** [KMS docs](https://docs.aws.amazon.com/kms/latest/developerguide/), [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/), [Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)

---

## 🎯 한 문장

> **KMS = *AWS의 *암호화 백본**. *직접 데이터 암호화 X — Data Key만 발급*. *Envelope Encryption*으로 *큰 데이터도 효율*. Secrets Manager (회전·통합 강) / Parameter Store (config + secret, 무료). 모든 *S3·EBS·RDS·Secrets Manager*가 KMS 위에 얹혀 있음.**

---

## 1. KMS — *Key Management Service*

### Customer Master Key (CMK) / KMS Key
- *암호화의 *최상위 키**
- *AWS가 *키 자체를 안전하게 보관* (HSM 기반)
- *원본 키는 *export 불가* — 항상 *AWS 안에서만 사용*

### Key 종류
| Type | 특징 |
|---|---|
| **AWS-managed key** | AWS service가 자동 생성 (`aws/s3`, `aws/ebs` 등). 무료. |
| **Customer-managed key (CMK)** | *내가 생성·관리*. *Key Policy·rotation·audit 완전 통제*. 시간당 $1. |
| **Customer-supplied key** | *내가 import한 key material*. 매우 드뭄. |
| **AWS-owned key** | AWS 내부용, *사용자 안 보임* |

→ 운영에선 **CMK 권장** — *policy·rotation·audit 통제*.

### Key Policy
- *Resource Policy* — *누가 이 key 사용 가능*
- *IAM Policy*와 *결합* — 둘 다 Allow 필요 (대부분의 경우)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Root admin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow app use",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111:role/my-app-role" },
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

### Rotation
- *Automatic rotation 활성* — *연 1회 자동* (CMK만, AWS-managed는 자동)
- *옛 key version도 *decrypt에 사용 가능* (사용자 데이터 재암호화 필요 X)
- *Manual rotation* — *새 CMK 생성 + alias 이동*

### Multi-region key
- *여러 region에 같은 key material*
- *DR·global service*에 활용

---

## 2. Envelope Encryption — *KMS의 핵심 패턴*

### 발상
- *큰 데이터를 *KMS로 직접 암호화*는 비효율* (API 호출 + 크기 한도)
- 해결: *Data Key (DEK)는 *클라이언트가 생성·관리*, KMS는 *DEK를 암호화*만**

### 흐름
```
1. 데이터 암호화:
   - 앱이 KMS에 GenerateDataKey 호출
   - KMS가 *DEK (Data Encryption Key) 생성*
     - plaintext DEK (메모리에서만 사용 후 폐기)
     - encrypted DEK (KMS의 CMK로 암호화됨, 영구 저장 가능)
   - 앱이 *plaintext DEK로 데이터 암호화* (AES-GCM)
   - 결과 = encrypted data + encrypted DEK 함께 저장
   - plaintext DEK 메모리에서 폐기

2. 데이터 복호화:
   - encrypted DEK + encrypted data 읽음
   - 앱이 KMS에 Decrypt 호출 (encrypted DEK 전달)
   - KMS가 *CMK로 encrypted DEK 복호화* → plaintext DEK 반환
   - 앱이 plaintext DEK로 *데이터 복호화*
```

### 장점
- *큰 데이터도 효율적* — KMS 호출은 *작은 DEK만*
- *CMK는 *항상 KMS 안에만* — 절대 외부에 안 나옴
- *각 데이터에 *unique DEK* 가능 — 한 DEK 침해 ≠ 전체 영향

### S3·EBS·RDS의 *envelope encryption*
- AWS service들이 *내부적으로 자동* — 사용자는 *KMS key만 명시*
- *encrypted DEK가 metadata에 저장*, KMS로 복호화

---

## 3. KMS의 *API*

| API | 의미 |
|---|---|
| **Encrypt** | 작은 데이터 직접 암호화 (4KB까지) |
| **Decrypt** | 복호화 |
| **GenerateDataKey** | *DEK 생성* (plaintext + encrypted 둘 다 반환) — **envelope의 핵심** |
| **GenerateDataKeyWithoutPlaintext** | encrypted DEK만 (plaintext 안 받음) — *replicate 시* |
| **ReEncrypt** | 한 key로 암호화된 데이터를 *다른 key로 재암호화* (KMS 안에서) |
| **EnableKey / DisableKey** | key 활성/비활성 |
| **ScheduleKeyDeletion** | *최소 7일 후* 삭제 (cancel 가능) |

### Aliasing
```bash
aws kms create-alias --alias-name alias/my-app-key --target-key-id arn:...:key/abc
```

→ *key ID 대신 alias 사용*. *key rotation 시 alias만 이동* — 코드 변경 X.

---

## 4. KMS의 *CloudTrail 통합*

- 모든 KMS API 호출이 *CloudTrail에 기록*
- *누가 언제 무엇 암호화·복호화*
- *컴플라이언스 audit의 핵심*
- *비정상 패턴 감지* — 새 IP의 대량 Decrypt 호출 등

→ **KMS = 보안 + audit 의 베이스**.

---

## 5. Secrets Manager — *secret 중앙 관리*

### 특징
- *Secret 저장 + 회전 + 통합*
- *RDS·DocumentDB·Redshift password 자동 회전* (Lambda)
- *cross-region replication*
- *audit (CloudTrail)*
- *KMS encryption at rest*

### Secret 종류
- *generic key-value JSON*
- *RDS credentials* (자동 회전 가능)
- *OAuth token*
- *API key*

### 사용
```bash
# 저장
aws secretsmanager create-secret --name myapp/db \
  --secret-string '{"username":"alice","password":"secret"}'

# 조회
aws secretsmanager get-secret-value --secret-id myapp/db
```

### 비용
- *secret 1개당 $0.40/월* + *API 호출 $0.05/10k*
- 100 secret = $40/월

### 자동 회전
- *Lambda function 자동 생성* — RDS·DocumentDB·Redshift
- *N일 (예: 30일)마다 자동 회전*
- *옛 password와 새 password 잠시 둘 다 valid*

### K8s 통합
- **External Secrets Operator** ([`k8s-security-basics/05`](../k8s-security-basics/05-secrets-and-encryption.md))
- *ExternalSecret CRD에 *Vault 대신 AWS Secrets Manager 명시**

### Resource Policy
- *Cross-account 접근* 가능 (Bucket Policy 비슷)

---

## 6. Parameter Store (SSM) — *config + secret*

### 특징
- *Systems Manager의 *Parameter Store**
- *4 tier*: Standard (무료) / Advanced (8KB+, policy 가능) / Intelligent-Tiering
- *String / StringList / SecureString (KMS 암호화)*
- *hierarchy* — `/myapp/prod/db/password` 같은 path

### vs Secrets Manager

| | Secrets Manager | Parameter Store |
|---|---|---|
| 비용 | $0.40/secret/월 | *Standard 무료* (Advanced $0.05) |
| 자동 회전 | ✓ (RDS 등) | ✗ |
| Cross-region | ✓ | △ (수동) |
| 통합 | 깊음 (RDS·DocumentDB) | 일반적 |
| 4 KB 한도 | 64 KB | Standard 4KB, Advanced 8KB |
| 사용 빈도 | 적음 | 많음 (config) |

→ **Secret + 자동 회전 = Secrets Manager**. **일반 config = Parameter Store** (무료).

### SecureString
```bash
aws ssm put-parameter --name "/myapp/prod/db/password" \
  --value "secret" --type SecureString --key-id alias/myapp
```

→ *KMS로 암호화 저장*. read 시 자동 복호화 (권한 있으면).

---

## 7. *Cross-account secret 공유*

### Pattern
- Account A의 secret을 Account B의 Pod이 접근

### 흐름
1. Account A의 Secrets Manager에 *Resource Policy*:
   ```json
   {
     "Principal": { "AWS": "arn:aws:iam::B:role/my-app-role" },
     "Action": "secretsmanager:GetSecretValue",
     "Resource": "*"
   }
   ```

2. Account B의 IAM Role에 *secret read 권한*:
   ```json
   {
     "Action": "secretsmanager:GetSecretValue",
     "Resource": "arn:aws:secretsmanager:region:A:secret:myapp-*"
   }
   ```

3. **KMS key도 cross-account 정책 필요** (KMS encrypted secret이므로):
   ```json
   {
     "Principal": { "AWS": "arn:aws:iam::B:role/my-app-role" },
     "Action": "kms:Decrypt",
     "Resource": "*"
   }
   ```

→ *3가지 정책 다* 있어야 작동.

---

## 8. S3·EBS·RDS의 *KMS encryption*

### S3 SSE-KMS
```yaml
# Bucket 설정 또는 PUT 시
x-amz-server-side-encryption: aws:kms
x-amz-server-side-encryption-aws-kms-key-id: arn:aws:kms:...:key/abc
```

- *각 object의 DEK*는 *KMS로 암호화되어 metadata에*
- *read 시 KMS Decrypt 자동 호출*

### EBS Encryption
- *Volume 생성 시 *KMS key 명시**
- *snapshot도 자동 암호화*
- *불-encrypted volume → encrypted snapshot* 마이그레이션 가능

### RDS Encryption
- *생성 시 *Enable encryption + KMS key 명시**
- *automated backup·snapshot·read replica 모두 자동 암호화*
- **생성 후 변경 불가** — 변경하려면 *snapshot → 암호화된 새 instance 복원*

### Bucket / Account-level Default Encryption
- *모든 새 object를 *기본 암호화**
- *Bucket Policy로 *암호화 안 된 PUT 거부* 강제*

---

## 9. *비용·운영 best practices*

### 1. *Customer-managed CMK + Auto-rotation*
- AWS-managed key는 *audit·policy 제약*
- CMK는 *완전 통제 + auto-rotation*

### 2. *Multi-region key for DR*
- *region 별 별도 key 관리는 부담*
- multi-region key로 *DR 시 같은 key 활용*

### 3. *Key alias 사용*
- *alias로 abstract* — *key rotation 시 코드 변경 X*

### 4. *KMS API 호출 비용*
- *Decrypt $0.03/10k 호출*
- *대량 호출 시 비용 폭증* — *클라이언트 caching* (5분 등)

### 5. *Secret rotation 자동화*
- *manual 회전 = 영원히 안 함* + 사고
- *Secrets Manager 자동 회전* 활성

### 6. *Resource Policy + IAM Policy 둘 다*
- KMS는 *둘 다 Allow 필요*
- *cross-account 시* 특히

### 7. *Audit*
- *CloudTrail로 모든 KMS·Secrets Manager API 추적*
- *SIEM 통합*

---

## 10. 자주 헷갈리는 것

### 10-1. *KMS key 직접 export*
- *불가능* — key material은 *항상 KMS 안*
- 그래서 *secure*

### 10-2. *Decrypt 시 *어느 key 사용한지 명시 안 함**
- KMS가 *ciphertext metadata에서 자동 판단*
- 단 *cross-region·imported key는 명시 필요*

### 10-3. *Key 삭제 *7일 cooldown**
- *즉시 삭제 X* — 최소 7일 (최대 30일) waiting period
- 그 사이 *cancel 가능*
- *데이터 영구 손실 방지*

### 10-4. *Secrets Manager의 *secret JSON 전체 read***
- 한 secret = 여러 key-value (JSON)
- 한 API 호출로 *전체 read*
- *RBAC로 key 단위 제한 불가* — *secret 단위 분리*

### 10-5. *Parameter Store SecureString *복호화 권한 분리**
- read 권한 있어도 *KMS Decrypt 권한 없으면 plaintext 못 봄*
- *권한 분리* 가능

### 10-6. *KMS API rate limit*
- *region별 *기본 rate limit* (10k req/s 등)
- *대량 호출* (예: S3 SSE-KMS 매 object) 시 *throttle*
- *clientside cache + S3 Bucket Key* 옵션 활용

---

## 🎤 면접 빈출 Q&A

### Q1. KMS Envelope Encryption은?
> *큰 데이터를 KMS로 직접 암호화는 비효율* (API 호출 + 4KB 한도). 해결: *KMS는 Data Key (DEK)만 발급*, *클라이언트가 DEK로 실제 암호화* (AES-GCM). encrypted DEK + encrypted data 함께 저장. **CMK는 항상 KMS 안에만**. **각 데이터에 unique DEK** — 한 DEK 침해 ≠ 전체 영향. S3·EBS·RDS가 *내부적으로 자동*.

### Q2. AWS-managed key vs Customer-managed CMK?
> **AWS-managed** (`aws/s3`, `aws/ebs`): service가 자동 생성, 무료. *Policy·rotation 통제 X*. **CMK** ($1/월): *내가 생성·관리*, **Key Policy + auto-rotation (연 1회) + CloudTrail audit + key alias** 완전 통제. **운영은 CMK 권장** — *audit·정책 통제 필수*.

### Q3. Secrets Manager vs Parameter Store?
> **Secrets Manager** ($0.40/secret/월): *자동 회전 (RDS·DocumentDB)*, *cross-region*, *64KB 한도*, *통합 깊음*. **Parameter Store** (Standard 무료): *config + secret*, *4KB*, *자동 회전 X*. 사용: **secret + 자동 회전 = Secrets Manager**, **일반 config = Parameter Store** (무료). 둘 다 *KMS encryption*.

### Q4. Cross-account secret 공유?
> *3가지 정책 다* 필요: (1) Account A의 **Secret Resource Policy**에 *B의 Role 허용*. (2) Account B의 **IAM Policy**에 *secretsmanager:GetSecretValue 권한*. (3) **KMS key policy + IAM Policy**에 *Decrypt 권한* (KMS encrypted secret이라). 셋 중 하나 빠지면 *접근 불가*.

### Q5. KMS key를 *export*할 수 있나요?
> **불가능**. Key material은 *항상 KMS 내부* (HSM 기반). 그래서 *secure*. 외부에선 *encrypt/decrypt API 호출*만 — 데이터를 KMS에 보내거나 (4KB까지), *DEK를 받아서 클라이언트가 처리*. **Custom key import** (BYOK)는 *내가 만든 key material을 import* 가능하나 export는 여전히 불가.

### Q6. KMS *Decrypt 호출 비용*과 최적화?
> *Decrypt $0.03/10k 호출* — *대량 호출 시 비용 폭증*. 예: S3 SSE-KMS로 *수백만 object read* → *수백만 Decrypt 호출*. **최적화**: (1) **S3 Bucket Key** — bucket 단위 DEK *client-side cache* (호출 *최대 99% 감소*). (2) *Application clientside cache* (5분 등). (3) *Envelope encryption custom* — *적은 DEK + 자주 재사용*.

### Q7. Secret 자동 회전이 *왜 중요*?
> Manual 회전 = *영원히 안 함* + 사고. Secret 유출 시 *영구 위험*. **Secrets Manager 자동 회전**: *Lambda function이 N일 (예: 30일)마다 새 password 생성 → DB에 적용 → Secret 갱신*. 옛 + 새 password 잠시 둘 다 valid — *graceful*. *External Secrets Operator + AWS Secrets Manager*로 *K8s Secret 자동 갱신* + *Pod 자동 재시작 (Reloader)*.

---

## 🔗 Cross-reference

- **IAM Policy + Resource Policy** → [01-iam-and-identity.md](01-iam-and-identity.md)
- **K8s Secret + etcd encryption (KMS provider)** → [`k8s-security-basics/05`](../k8s-security-basics/05-secrets-and-encryption.md)
- **External Secrets Operator** → [`k8s-security-basics/05`](../k8s-security-basics/05-secrets-and-encryption.md)
- **S3 SSE-KMS / EBS·RDS encryption** → [04-storage-and-database.md](04-storage-and-database.md)
- **Terraform secret 관리** → [`terraform-basics/04`](../terraform-basics/04-variables-and-outputs.md)
- **CloudTrail audit** → [01-iam-and-identity.md](01-iam-and-identity.md)

---

## 📝 3줄 요약

1. *KMS = 암호화 백본*. **Envelope Encryption** — KMS는 *Data Key (DEK)만 발급*, 클라이언트가 DEK로 데이터 암호화. CMK는 *항상 KMS 안*.
2. *Customer-managed CMK + auto-rotation + key alias + CloudTrail audit* 운영 표준. *KMS Decrypt 비용 최적화 (S3 Bucket Key, clientside cache)*.
3. *Secrets Manager (자동 회전·통합) vs Parameter Store (무료·일반)*. **Cross-account는 Secret Resource Policy + IAM + KMS 3정책 모두 필요**.

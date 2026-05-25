# 04 — Config와 Storage: ConfigMap·Secret·PV·PVC·StorageClass·CSI

> **공식 근거:** [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/), [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), [Storage](https://kubernetes.io/docs/concepts/storage/)
>
> **선행:** [`docker-basics/04`](../docker-basics/04-volumes-and-storage.md) (Volume 개념)

---

## 🎯 한 문장

> **Config는 *ConfigMap/Secret 으로 코드와 분리*해서 *환경별 변동 흡수*. Storage는 *PV로 실제 스토리지 추상화* + *PVC로 Pod의 요청* + *StorageClass로 자동 provisioning* + *CSI로 외부 시스템 통합*.**

---

## 1. ConfigMap — *코드와 환경 분리*

### 왜
- 12-Factor App: *환경별 설정을 image에 박지 말 것*
- dev/stg/prod가 *같은 image*를 쓰되 *다른 설정*

### 만들기
```bash
# 파일에서
kubectl create configmap app-config --from-file=config.yaml

# 리터럴
kubectl create configmap app-config --from-literal=LOG_LEVEL=info

# YAML
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  DB_HOST: postgres
  application.yaml: |
    server:
      port: 8080
```

### Pod에서 쓰기 — 3가지 방법

**(A) 환경 변수**
```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```
- 단점: *ConfigMap 변경해도 Pod 재시작 전엔 옛 값*

**(B) `envFrom` — 모든 키를 env로**
```yaml
envFrom:
- configMapRef:
    name: app-config
```

**(C) Volume mount — 파일로**
```yaml
volumes:
- name: config
  configMap:
    name: app-config
containers:
- name: app
  volumeMounts:
  - name: config
    mountPath: /etc/app
```
- 장점: *ConfigMap 변경 → 수십 초 안에 컨테이너 안 파일 갱신* (kubelet sync)
- 단점: 앱이 *파일 변경 감지·재로드*해야 의미. 안 그러면 *프로세스 재시작* 필요.

### 흔한 함정
- ConfigMap *변경해도 자동 rollout 없음*
- → 해결: *ConfigMap 내용 해시*를 Deployment Pod template annotation에 박기 (Helm `checksum/config` 패턴):
  ```yaml
  metadata:
    annotations:
      checksum/config: "{{ include (print $.Template.BasePath \"/configmap.yaml\") . | sha256sum }}"
  ```
  → ConfigMap 변경 → annotation 해시 변화 → Pod template 변화 → rolling update 자동.

---

## 2. Secret — *민감 정보 (단, base64 != 암호화)*

### Secret vs ConfigMap
- *구조 거의 같음*. 다만:
  - data가 **base64 인코딩**
  - kubectl 출력 시 *기본 숨김*
  - 메모리 *tmpfs로 마운트* (디스크에 안 닿음, kubelet)
  - *RBAC을 별도로 더 빡세게* 가능
- **base64는 *암호화가 아님*** — 디코딩 자유롭게 가능. 누구나 `base64 -d`.

### 진짜 보안을 원하면
- **Encryption at rest** — etcd에 *암호화해서 저장* (KMS 통합)
  ```yaml
  # EncryptionConfiguration
  resources:
  - resources: [secrets]
    providers:
    - kms:
        name: myKMS
        endpoint: unix:///tmp/socketfile.sock
  ```
- **외부 secret manager 통합** — Vault, AWS Secrets Manager, etc.
- **External Secrets Operator** — *외부 매니저의 secret*을 *K8s Secret으로 자동 동기화*
- **CSI Secret Store Driver** — Pod이 *직접 외부 매니저에서 secret fetch* (K8s Secret 안 거침)

### Secret 사용 패턴 — 가장 흔한
```yaml
# DB 비밀번호 같은 케이스
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64

---
# Pod에서
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

### Secret 타입
- `Opaque` (기본) — 임의 key-value
- `kubernetes.io/dockerconfigjson` — image registry credential
- `kubernetes.io/tls` — TLS 인증서·키
- `kubernetes.io/service-account-token` — ServiceAccount의 token (자동 생성)

### ImagePullSecret
private registry에서 image pull 시 필요:
```yaml
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
  - name: my-registry-secret
  containers:
  - image: private.registry.io/myapp:latest
```
또는 *ServiceAccount에 박아*두면 그 ServiceAccount 쓰는 모든 Pod에 자동 적용.

---

## 3. Volume — *Pod이 데이터 보존·공유하는 방법*

### Volume의 *수명*
- Volume은 *Pod에 속함*. Pod 죽으면 *일부는 사라짐*.
- *어떤 종류*인지에 따라 다름.

### Volume 종류 (자주 쓰는 것)

| 종류 | 수명 | 용도 |
|---|---|---|
| `emptyDir` | *Pod 죽으면 사라짐* | 컨테이너 간 공유 임시 공간 |
| `hostPath` | *호스트 노드의 디렉터리* | 노드의 *호스트 파일* 접근 (DaemonSet 등). *Pod 이동 시 다른 데이터* |
| `configMap` / `secret` | ConfigMap/Secret 갱신 시 즉시 | 설정·secret 파일로 마운트 |
| `projected` | 위 여러 개를 *하나로* 묶음 | ServiceAccount token + downward API 등 |
| `persistentVolumeClaim` | *Pod 죽어도 데이터 유지* | *상태있는 워크로드* (DB, 큐) |
| `csi` | CSI driver가 결정 | 외부 스토리지 (EBS, EFS, Ceph 등) |

→ *데이터 영속화*는 항상 *PV/PVC* (또는 CSI 직접). 그 외는 *임시·설정·노드 접근*.

---

## 4. PV / PVC / StorageClass — *Storage 추상화 3층*

### 문제
- 운영자는 *Ceph 클러스터 / AWS EBS / NFS*를 운영
- 개발자는 *"100GB SSD 필요"* 라고만 명세하고 싶음
- *구현 변경*해도 *Pod spec 안 바뀌어야*

### 해결: 3계층 추상화
```
┌─────────────────────────────────────┐
│  PVC (PersistentVolumeClaim)        │  ← 개발자: "100GB SSD 필요"
│  - storage: 100Gi                   │
│  - storageClassName: fast-ssd       │
│  - accessModes: [ReadWriteOnce]     │
└─────────────────────────────────────┘
              ▲ (bind)
              │
┌─────────────────────────────────────┐
│  PV (PersistentVolume)              │  ← 운영자: "여기 실제 storage 있다"
│  - capacity: 100Gi                  │
│  - awsElasticBlockStore: vol-abc... │
│  ...                                │
└─────────────────────────────────────┘
              ▲ (provisioned by)
              │
┌─────────────────────────────────────┐
│  StorageClass                       │  ← *자동 provisioning* 정의
│  - provisioner: ebs.csi.aws.com     │
│  - parameters: { type: gp3 }        │
└─────────────────────────────────────┘
```

### Static vs Dynamic Provisioning

**Static**: 운영자가 *PV 미리 수동 생성* → PVC 요청 오면 match해서 bind
- 단점: *수동* — 새 PVC 올 때마다 PV 만들어야

**Dynamic** (현대 표준): StorageClass 만 정의 → PVC 요청 오면 *자동으로 PV 생성*
- StorageClass의 provisioner가 *클라우드 API 호출 → 실제 EBS 볼륨 생성 → PV 자원 만듦*

### accessModes
| Mode | 의미 |
|---|---|
| **ReadWriteOnce (RWO)** | *한 노드*에서만 read/write 마운트 | 대부분 블록 스토리지 (EBS 등) |
| **ReadOnlyMany (ROX)** | *여러 노드*에서 read-only | 정적 자원 공유 |
| **ReadWriteMany (RWX)** | *여러 노드*에서 read/write | NFS·EFS·Ceph CephFS |
| **ReadWriteOncePod (RWOP)** | *한 Pod*만 (K8s 1.22+) | 더 엄격한 RWO |

**중요:** EBS·GCE PD 같은 *블록 스토리지는 RWO만*. *RWX* 필요하면 EFS·NFS·CephFS 같은 *파일 스토리지*.

### Reclaim Policy
- **Retain** — PVC 삭제해도 PV·데이터 *유지* (수동 정리). 운영 DB 추천.
- **Delete** — PVC 삭제 시 PV·실제 스토리지 *함께 삭제*. 개발용.
- **Recycle** (deprecated)

---

## 5. CSI (Container Storage Interface) — *플러그인 표준*

### 왜
- 옛날엔 *스토리지마다 K8s 코어에 in-tree driver* 추가 (AWS·GCP·Azure 등). K8s 코드베이스 비대화·release 결합.
- 해결: *플러그인 표준*. K8s는 *CSI 인터페이스만 정의*. 각 storage 벤더가 *driver 따로 배포*.

### CSI driver 예시
- **EBS CSI** — AWS EBS
- **EFS CSI** — AWS EFS (RWX 가능)
- **Ceph CSI** — Ceph 블록/파일
- **NFS CSI** — NFS
- **Longhorn** — 분산 블록 (소규모 self-hosted에 인기)
- **Rook** — Ceph operator

### 구조
- *controller plugin* — Provisioning (PV 생성·삭제·snapshot)
- *node plugin* — DaemonSet으로 각 노드에 — 실제 마운트

→ `docker-basics/04` 의 *volume driver* 와 *같은 아이디어, K8s 표준화*.

---

## 6. *StatefulSet과 PVC* — VolumeClaimTemplate

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mydb
spec:
  serviceName: mydb
  replicas: 3
  selector:
    matchLabels: { app: mydb }
  template:
    metadata: { labels: { app: mydb } }
    spec:
      containers:
      - name: db
        image: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # ← 각 Pod별 PVC 자동 생성
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
```

→ Pod별로 *독립된 PVC* (`data-mydb-0`, `data-mydb-1`, `data-mydb-2`) *자동 생성*. *Pod 재시작 시에도 같은 PVC 재마운트* (같은 데이터).

### StatefulSet 삭제해도 PVC는 안 사라짐 (의도)
- 수동 정리:
  ```bash
  kubectl delete pvc data-mydb-0 data-mydb-1 data-mydb-2
  ```
- *데이터 실수 삭제 방지*. 의도된 안전장치.

---

## 7. Downward API — *Pod 자기 정보를 컨테이너에 주입*

Pod의 metadata (이름·namespace·IP·label 등) 를 *컨테이너 안 환경변수·파일*로:

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
```

→ *로깅에 Pod 이름 박기, 모니터링 라벨링* 등.

---

## 8. 자주 헷갈리는 것

### 8-1. Secret base64 = 암호화 아님
- 누구나 `echo Y... | base64 -d` 로 디코딩
- *진짜 보안*은 etcd encryption at rest + RBAC + 외부 secret manager

### 8-2. ConfigMap·Secret *Volume mount 시* 자동 갱신
- *환경변수로 inject*면 *Pod 재시작 전엔 옛 값*
- 환경별로 *어느 방식*인지 의식

### 8-3. PVC와 PV의 *바인딩이 한 번 일어남*
- 한 번 bind된 PVC는 *다른 PV로 못 옮김*
- *데이터 마이그레이션*은 *새 PVC 생성 후 데이터 복사*

### 8-4. EBS는 *Availability Zone에 묶임*
- EBS는 *특정 AZ의 자원*. Pod이 다른 AZ로 이동 시 *마운트 못 함*
- → topology-aware StorageClass + Pod 스케줄링 조정 (또는 *zone에 묶인 Pod*)

### 8-5. RWX 필요한데 *RWO만 지원하는 storage* 쓰면
- *PV 마운트 fail* (다른 노드에선)
- → EFS·NFS·CephFS 같은 *파일 스토리지*로

### 8-6. *Pod이 PVC 마운트 안 됨* 디버깅
- `kubectl describe pod` Events: *FailedMount*
- 원인: AZ 불일치, PV-PVC 매칭 안 됨, CSI driver fail, IAM 권한 부족 (CSI controller가 클라우드 API 권한 부족)
- `kubectl get events --sort-by=.metadata.creationTimestamp` 로 최근 events

---

## 🎤 면접 빈출 Q&A

### Q1. ConfigMap과 Secret 차이는?
> 구조 거의 같음. *Secret은 base64 인코딩 + kubectl 출력 시 숨김 + tmpfs 마운트 + RBAC 더 빡세게 설정 권장*. **단 base64는 *암호화가 아님*** — 진짜 보안이 필요하면 *etcd encryption at rest* + *외부 secret manager (Vault, AWS Secrets Manager) + External Secrets Operator*.

### Q2. ConfigMap 변경하면 Pod이 자동 재시작 되나요?
> *환경변수로 inject*면 *재시작 전엔 옛 값*. *Volume mount*면 *수십 초 안에 파일 갱신* (kubelet sync). 그러나 *앱 자체가 파일 watch + reload* 해야 의미. 그래서 흔한 패턴은 *ConfigMap 해시를 Pod template annotation*에 박기 — `checksum/config: {sha256}` — 그러면 ConfigMap 바뀔 때 *Pod template 변화 → rolling update 자동*.

### Q3. PV / PVC / StorageClass 의 분리 이유?
> *추상화 분리*. PVC는 *개발자의 요청* ("100GB SSD 필요"), PV는 *운영자가 제공하는 실제 storage 자원*, StorageClass는 *자동 provisioning 정의*. *Dynamic provisioning*에선 PVC 요청 → StorageClass의 provisioner가 *자동으로 PV + 실제 storage 생성*. 운영자는 *storage 구현 바꿔도 Pod spec 안 바꿔도 됨*.

### Q4. RWO와 RWX 차이? 어떤 storage가 어느 걸?
> **RWO** (ReadWriteOnce): *한 노드*에서만 마운트. **RWX** (ReadWriteMany): *여러 노드*에서 동시 마운트. *블록 스토리지* (AWS EBS, GCE PD)는 *RWO만*. *RWX*는 *파일 스토리지* (NFS, AWS EFS, CephFS) 필요. → DB는 보통 RWO (단일 노드 마운트), 공유 자원은 RWX.

### Q5. CSI는 무엇이고 왜 필요했나요?
> *Container Storage Interface*. K8s가 *storage 구현을 외주*하는 표준 인터페이스. 옛날엔 *모든 storage driver가 K8s 코어에 in-tree* → 코드베이스 비대화·release 결합. CSI 도입 후 *벤더가 driver 따로 배포*, K8s는 *인터페이스만 정의*. EBS·EFS·Ceph 등 다 CSI driver로.

### Q6. StatefulSet에서 *PVC가 자동으로 생기는* 메커니즘은?
> `volumeClaimTemplates` 사용. StatefulSet의 *각 Pod별로 PVC 자동 생성* (`data-mydb-0`, `-1`, `-2`). Pod 죽고 재생성돼도 *같은 PVC 재마운트* (= 같은 데이터). StatefulSet 삭제해도 *PVC는 안 사라짐* (의도 — 실수 데이터 삭제 방지). 정리는 *PVC 명시적 삭제*.

### Q7. EBS PV가 Pod에 *마운트 fail* 디버깅?
> `describe pod` Events에서 *FailedMount* 메시지 확인. 흔한 원인: (1) *AZ 불일치* — EBS는 *AZ에 묶임*, Pod이 다른 AZ로 이동하면 fail → *topology-aware StorageClass* 또는 *zone affinity*. (2) *CSI controller IAM 권한 부족* — clouds API 호출 실패. (3) *PV-PVC 매칭 안 됨* — accessMode·size·storageClass 검토.

---

## 🔗 Cross-reference

- **Volume 개념·OverlayFS** → [`docker-basics/04`](../docker-basics/04-volumes-and-storage.md)
- **StatefulSet** → [02-workloads.md](02-workloads.md)
- **External Secrets Operator·Vault·KMS** → `aws-basics/`, `k8s-security-basics/` (예정)
- **Helm `checksum/config` 패턴** → `helm-basics/` (예정)
- **etcd encryption at rest** → [01-architecture.md](01-architecture.md), `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *ConfigMap·Secret으로 코드와 환경 분리*. Secret base64는 *암호화 아님* — etcd encryption + 외부 매니저 통합 필수. *변경 시 자동 rollout 위해 `checksum/config` 패턴*.
2. *Storage 3계층*: PVC(요청) ↔ PV(실제) ↔ StorageClass(자동 provisioning). Dynamic이 현대 표준. CSI가 *벤더 driver 외주* 인터페이스.
3. *EBS는 AZ 묶임·RWO만*. RWX 필요하면 *EFS/NFS/CephFS*. StatefulSet은 `volumeClaimTemplates`로 Pod별 PVC 자동.

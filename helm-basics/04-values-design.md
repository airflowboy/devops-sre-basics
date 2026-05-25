# 04 — Values Design: 우선순위·schema·multi-env

> **공식 근거:** [Helm Values Files](https://helm.sh/docs/chart_template_guide/values_files/), [JSON Schema](https://helm.sh/docs/topics/charts/#schema-files)

---

## 🎯 한 문장

> **Values 설계가 chart의 *유지보수성·재사용성·운영 안전*을 결정. *override 우선순위 4단계*, *JSON Schema로 검증*, *환경별 values 분리*, *_helpers.tpl로 반복 제거*가 핵심.**

---

## 1. Values 우선순위 — *낮음 → 높음*

```
1. chart의 values.yaml                              (기본값, 가장 낮음)
2. parent chart의 values.yaml (sub-chart의 경우)     (위에서 override)
3. -f values-prod.yaml (여러 파일 가능, 나중 것 우선)
4. --set / --set-file / --set-string (CLI 마지막에 override, 가장 높음)
```

### 예시
```bash
helm install myrelease ./mychart \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag=v1.2.3
```

- chart의 `values.yaml` (chart 안)
- + `values.yaml` 파일 (현재 디렉터리, 같은 이름이지만 *다른 파일*)
- + `values-prod.yaml` (덮어쓰기)
- + `image.tag=v1.2.3` (가장 우선)

### `--set` 문법
```bash
--set image.tag=v1.2.3                # 점 표기 (nested)
--set image.tag=v1.2.3,replicaCount=3 # 콤마로 여러 개
--set 'env={dev,prod}'                # 배열
--set 'env[0]=dev,env[1]=prod'        # 배열 인덱스
--set 'annotations.team=payments'     # nested
--set "key=value with spaces"         # 공백 처리
```

→ 복잡하면 *values 파일이 깔끔*. `--set` 은 *한두 개 override*만.

---

## 2. Multi-Environment 패턴

### 패턴 1: 환경별 values 파일
```
mychart/
├── values.yaml             # 기본값 (안전한 default)
├── values-dev.yaml          # dev 차이만
├── values-stg.yaml          # stg 차이만
└── values-prod.yaml         # prod 차이만
```

설치:
```bash
helm install myapp ./mychart -f values-prod.yaml
```

각 환경 파일은 *base values를 override*만. *전체 다시 안 적음*.

```yaml
# values-prod.yaml
replicaCount: 5
image:
  tag: v1.0.0          # prod-pinned
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
ingress:
  enabled: true
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix
```

### 패턴 2: 환경별 별도 directory (외부)
```
infra/
├── charts/myapp/             # 단일 chart
└── envs/
    ├── dev/values.yaml
    ├── stg/values.yaml
    └── prod/values.yaml
```

→ ArgoCD ApplicationSet과 잘 맞음. *envs/{env}/values.yaml* 패턴으로 자동 순회.

### 패턴 3: Umbrella chart (의존성 패턴)
```
production-stack/
├── Chart.yaml
└── values.yaml
```

```yaml
# Chart.yaml
dependencies:
- name: myapp
  version: 0.1.0
  repository: file://../myapp
- name: postgresql
  version: 12.x.x
  repository: https://charts.bitnami.com/bitnami
```

→ *여러 chart 묶어서 하나의 release로*. 마이크로서비스 모음 패턴.

---

## 3. values.schema.json — *검증의 정석*

`values.yaml` 옆에 `values.schema.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "pattern": "^[a-z0-9./-]+$"
        },
        "tag": { "type": "string" },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "limits": { "$ref": "#/$defs/resourceSpec" },
        "requests": { "$ref": "#/$defs/resourceSpec" }
      }
    }
  },
  "$defs": {
    "resourceSpec": {
      "type": "object",
      "properties": {
        "cpu": { "type": ["string", "number"] },
        "memory": { "type": "string", "pattern": "^[0-9]+[KMG]i?$" }
      }
    }
  }
}
```

→ `helm install` 자동 검증. *잘못된 type / 누락 / pattern 위반* 즉시 fail.

### 효과
- `--set replicaCount=abc` → 즉시 *integer 아님* fail
- `image: { tag: 1.0 }` (숫자) → *string 아님* fail
- prod에 잘못된 값 박힘 사고 *원천 차단*

---

## 4. 명명 컨벤션 (Bitnami 기준)

Bitnami가 사실상 *de facto 표준*:

```yaml
# 이미지
image:
  registry: docker.io           # 별도 명시 가능
  repository: bitnami/nginx
  tag: ""                       # 빈 값 → .Chart.AppVersion
  pullPolicy: IfNotPresent
  pullSecrets: []

# 이름
nameOverride: ""                # chart name override
fullnameOverride: ""            # release-name + chart-name 패턴 override

# 공통 라벨/어노테이션
commonLabels: {}
commonAnnotations: {}

# 자원
resources: {}                   # 기본 빈 — 사용자가 명시
podLabels: {}
podAnnotations: {}

# 보안
podSecurityContext: {}
containerSecurityContext: {}

# Service
service:
  type: ClusterIP
  ports:
    http: 80                    # 명명된 포트 (다중 포트 지원)
  annotations: {}

# Ingress
ingress:
  enabled: false
  hostname: chart-example.local
  ingressClassName: ""
  annotations: {}
  tls: false

# Persistence
persistence:
  enabled: true
  storageClass: ""              # "" = default storage class
  accessModes: [ReadWriteOnce]
  size: 8Gi

# 스케줄링
nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPU: 80

# Probes
livenessProbe:
  enabled: true
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  enabled: true
  initialDelaySeconds: 5
  periodSeconds: 10
```

→ *이 컨벤션 따르면 사용자가 익숙*. 자기만의 *특이 구조* 만들면 사용자 *학습 비용*.

---

## 5. *Library Chart* — *type=library*

```yaml
# Chart.yaml
apiVersion: v2
name: common
type: library                   # ← 핵심
version: 1.0.0
```

- *직접 install 불가* (실행 가능한 자원 없음)
- *named template만* 제공 — 다른 chart가 *import 해서 사용*
- *공통 logic 재사용* — _helpers.tpl 의 cross-chart 버전

### 활용
```yaml
# my-app-chart/Chart.yaml
dependencies:
- name: common
  version: 1.0.0
  repository: https://my-org.github.io/common-charts
  import-values:
  - common-defaults
```

```yaml
# my-app-chart/templates/deployment.yaml
{{ include "common.deployment" . }}
```

→ 사내에 *common chart 표준화*하면 *각 팀의 chart 작성 부담 감소*. Bitnami common chart가 모범.

---

## 6. *Schema-First 설계*

### 일반 흐름
1. *requirements 모음* (어떤 옵션 필요한가)
2. **values.schema.json 먼저 작성** (구조·type·제약)
3. *values.yaml 작성* (기본값)
4. *templates 작성*

### 왜?
- schema 없이 시작하면 *values 구조가 들쭉날쭉*
- schema-first → *구조 일관*, *팀 communication 명확*
- *사용자 입장에서 schema = 문서*

---

## 7. *민감 정보* 처리

### values.yaml에 *평문 secret 박지 말기*
```yaml
# ❌ 절대 안 됨
database:
  password: "supersecret"
```

→ release Secret에 *그대로* 저장. *RBAC 가진 누구나 보임*. Git에 commit하면 *영구 노출*.

### 패턴
1. **External Secrets Operator (ESO)** — values에 *Vault path만* 명세, 실제 값은 *cluster에서 fetch*
```yaml
externalSecrets:
- name: db-credentials
  vaultPath: kv/myapp/db
  keys: [password]
```

2. **Sealed Secrets** — *공개키로 암호화*된 secret을 git에. cluster controller가 *복호화*.

3. **SOPS + helm-secrets plugin** — values 일부를 *암호화* (kms/age/pgp)
```yaml
# values-prod.yaml
database:
  password: ENC[AES256_GCM,data:abc...,iv:def...,tag:ghi...,type:str]
```
```bash
helm secrets install myapp ./mychart -f secrets/values-prod.yaml
```

4. **CI에서 --set 으로 주입** — secret은 CI secret store에서, *git에 박지 않음*
```yaml
- run: |
    helm upgrade myapp ./mychart \
      -f values-prod.yaml \
      --set database.password=${{ secrets.DB_PASSWORD }}
```
*단, 환경변수로 노출 위험 — log 마스킹 확인.*

---

## 8. *조건부 자원 생성* — `enabled` flag 패턴

```yaml
# values.yaml
ingress:
  enabled: false
  ...

serviceAccount:
  create: true
  ...

autoscaling:
  enabled: false
  ...
```

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

→ 환경별로 *기능 on/off*. dev 에선 Ingress 없고 prod 에선 있음.

---

## 9. *Common values 외부 file*

대규모 환경에서 *전사 공통값* 분리:

```
infra/
├── charts/myapp/
├── values/
│   ├── company-defaults.yaml    # 전사 공통 (registry, monitoring labels)
│   ├── team-defaults.yaml       # 팀 공통
│   └── env/
│       ├── dev.yaml
│       ├── stg.yaml
│       └── prod.yaml
```

```bash
helm upgrade myapp ./charts/myapp \
  -f values/company-defaults.yaml \
  -f values/team-defaults.yaml \
  -f values/env/prod.yaml \
  --set image.tag=$VERSION
```

→ *공통값 한 번에 변경* (예: registry 변경) → *모든 chart 일괄 적용*.

---

## 10. 자주 헷갈리는 것

### 10-1. *`--set replicaCount=0`* — *기본값 적용 안 됨*
- helm은 *명시 값을 그대로 사용*. `default 1`로 fallback 안 됨.
- → schema로 `minimum: 1` 강제

### 10-2. *Empty string vs null*
- `image.tag: ""` = 빈 string, 명시된 값
- `image.tag: null` 또는 키 자체 없음 = *기본값 적용*
- `default` 함수는 *빈 값에도 적용* (조심)

### 10-3. *Subchart values override*
```bash
# 잘못된 경로
--set postgresql.auth.password=xxx     # OK (subchart-name.path)
--set auth.password=xxx                # 의도 X (parent values로 감)
```

### 10-4. *YAML 1.2 vs 1.1 함정 — Norway problem*
- `country: NO` → *Boolean false* 로 해석됨 (옛 YAML)
- *항상 quote*: `country: "NO"`
- 같은 함정: `yes`, `on`, `off`, `null`

### 10-5. *values 파일 *나중에 명시한 것 우선**
- `-f base.yaml -f override.yaml` → override가 우선
- `-f override.yaml -f base.yaml` → base가 우선 (의도 X)

### 10-6. *helm show values* 로 *현재 values 확인*
```bash
helm show values bitnami/postgresql       # remote chart의 기본 values
helm get values myrelease                  # 설치된 release의 *override된 부분*
helm get values myrelease --all            # 설치된 release의 *모든 values*
```

---

## 🎤 면접 빈출 Q&A

### Q1. Helm values 우선순위는?
> 4단계 (낮음 → 높음): (1) chart의 *values.yaml*, (2) parent chart의 values (sub-chart인 경우), (3) `-f values-prod.yaml` (여러 개면 *나중 것 우선*), (4) `--set` / `--set-file` / `--set-string` (CLI 마지막 override, 가장 우선). 운영에선 *values 파일이 깔끔*, `--set`은 *한두 개 override만*.

### Q2. Multi-environment 어떻게?
> *환경별 values 파일 분리* — `values-dev.yaml`, `values-prod.yaml`. base는 `values.yaml`, 각 env 파일은 *차이만 override*. ArgoCD에선 *ApplicationSet*과 결합 — `envs/{env}/values.yaml` 패턴 자동 순회. 대규모면 *umbrella chart* 또는 *common values 외부 file*.

### Q3. values.schema.json 의 효과?
> *JSON Schema*로 values.yaml 검증. *helm install 시 자동* — type·required·minimum·pattern·enum 위반 즉시 fail. 효과: (1) 잘못된 값 *런타임 사고 차단* (예: `replicaCount: abc`), (2) *팀 communication 명확*, (3) *문서 역할*. **Schema-first 설계** 권장.

### Q4. 민감 정보는 values에 어떻게?
> *values에 평문 secret 박지 말기* — release Secret에 그대로 저장, git commit 시 영구 노출. 패턴: (1) **External Secrets Operator** — values에 *Vault path만*, cluster에서 fetch. (2) **Sealed Secrets** — 공개키 암호화. (3) **SOPS + helm-secrets** — values 일부 암호화 (kms/age). (4) **CI --set 주입** — secret은 CI secret store에서, git 0건. 운영 표준은 ESO + cloud secret manager.

### Q5. Library chart는 무엇? 왜?
> `type: library` chart — *직접 install 불가*, *named template만 제공*. 다른 chart가 *dependency로 import* 해서 _helpers.tpl 처럼 사용. *사내 공통 logic 표준화* 용. Bitnami common chart가 모범. *각 팀이 같은 패턴 반복 안 함*.

### Q6. *Norway problem*?
> YAML 1.1: `NO`, `yes`, `on`, `off` 등을 *Boolean으로 해석*. `country: NO` → `country: false`. 해결: *항상 quote* (`country: "NO"`). 비슷한 사고: `version: 1.0` → float `1.0` (string 아님), `tag: 01` → octal 또는 int. Helm chart 값은 *quote 적극적으로*.

### Q7. `helm get values` 와 `helm show values` 차이?
> `helm show values {chart}` — chart의 *기본 values.yaml 내용* (설치 전 확인). `helm get values {release}` — *설치된 release의 override된 부분만*. `helm get values {release} --all` — *설치된 release의 모든 values* (기본 + override). 디버깅 시 *실제로 적용된 값 확인*에 `--all`.

---

## 🔗 Cross-reference

- **chart 구조** → [01-chart-structure.md](01-chart-structure.md)
- **template 동작·sprig** → [02-templating.md](02-templating.md)
- **release lifecycle·history** → [03-release-lifecycle.md](03-release-lifecycle.md)
- **External Secrets Operator** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md), `k8s-security-basics/` (예정)
- **ArgoCD Application의 helm.valueFiles·parameters** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. *Values 우선순위 4단계* — chart values.yaml → parent values → `-f` (나중 우선) → `--set` (가장 우선). 운영은 *values 파일 분리* + `--set` 최소.
2. **values.schema.json으로 검증** + **Bitnami 컨벤션 따르기** + **`enabled` flag로 조건부 자원** + **민감정보는 ESO/Sealed/SOPS로 git에 박지 말기**.
3. 환경별 *values-dev/stg/prod.yaml*. 대규모면 *umbrella chart* 또는 *common values 외부 디렉터리*. *library chart*로 공통 logic 표준화.

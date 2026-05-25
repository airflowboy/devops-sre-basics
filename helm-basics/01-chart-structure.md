# 01 — Chart Structure: Chart.yaml·values.yaml·templates·_helpers

> **공식 근거:** [Helm Charts](https://helm.sh/docs/topics/charts/), [Chart File Structure](https://helm.sh/docs/topics/charts/#the-chart-file-structure)

---

## 🎯 한 문장

> **Chart = *디렉터리 구조의 패키지*. Chart.yaml(메타) + values.yaml(기본값) + templates/(K8s manifest template) + charts/(sub-chart 의존성)이 핵심. *`helm create`로 시작하지만 *제공된 nginx 예시는 거의 다 지우고* 자기 앱에 맞게 작성*.**

---

## 1. 디렉터리 구조

```
mychart/
├── Chart.yaml                ← 메타데이터 (필수)
├── values.yaml               ← 기본값 (사실상 필수)
├── values.schema.json        ← JSON Schema (선택, 강력 권장)
├── README.md                 ← 사용법 (관습)
├── LICENSE                   ← 라이선스 (관습)
├── charts/                   ← sub-chart 의존성 (helm dep update로 채움)
│   └── postgresql-12.1.0.tgz
├── templates/                ← K8s manifest 템플릿
│   ├── _helpers.tpl          ← named template (helper)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt             ← 설치 후 출력 메시지
│   └── tests/                ← `helm test`용
│       └── test-connection.yaml
├── crds/                     ← CRD (선택, helm이 lifecycle 관리 안 함)
└── .helmignore               ← 패키지에서 제외할 파일
```

---

## 2. Chart.yaml — *메타데이터*

```yaml
apiVersion: v2                  # v2 = Helm 3, v1 = Helm 2 (deprecated)
name: myapp
description: A Helm chart for my app
type: application               # 또는 'library' (코드만 제공, 직접 install 불가)
version: 0.1.0                  # chart version (semver) — 변경 시 매번 ↑
appVersion: "1.16.0"            # 실제 앱 버전 (사람용 표시)
icon: https://...               # UI 표시
keywords: [web, api]
home: https://my-org.com/myapp
sources:
- https://github.com/my-org/myapp
maintainers:
- name: Alice
  email: alice@example.com
dependencies:                   # sub-chart 의존성
- name: postgresql
  version: "12.x.x"             # semver constraint
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled # values.postgresql.enabled가 true일 때만 설치
  alias: db                     # 다른 이름으로 사용 (옵션)
  tags: [database]              # 그룹 enable/disable
```

### chart version vs app version
- **chart version** — *chart 자체의 버전*. chart 구조·template 변경 시 ↑
- **appVersion** — *chart가 패키징한 앱의 버전*. 사람용 표시. 보통 image tag와 매칭.
- → *둘 독립적으로 관리*. chart 구조 그대로인데 *image만* 새 버전이면 *chart version은 그대로*, *appVersion만* 변경.

### dependencies (sub-charts)
- 다른 chart를 *의존성으로* 포함 (예: 내 앱 + DB)
- `helm dependency update` 또는 `helm dep up` → `charts/` 에 *tgz로 다운로드*
- *Helm Charts Repository* 또는 *OCI registry*에서 fetch ([05 참조](05-repos-and-distribution.md))

---

## 3. values.yaml — *기본값*

```yaml
# 컨벤션 (Bitnami 패턴이 많은 곳에서 사용됨)
replicaCount: 1

image:
  repository: nginx
  tag: ""                       # 빈 값이면 .Chart.AppVersion 사용 (관습)
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podSecurityContext:
  fsGroup: 1000
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop: [ALL]
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
  - host: chart-example.local
    paths:
    - path: /
      pathType: ImplementationSpecific
  tls: []

resources: {}
# 예시:
#   limits:
#     cpu: 100m
#     memory: 128Mi
#   requests:
#     cpu: 100m
#     memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# Sub-chart values (의존성 chart에 전달)
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: ""                # 운영에선 외부 secret manager
```

### 좋은 values.yaml 의 원칙
- **모든 값에 *기본값* 제공** — 사용자가 *최소 명세*로 동작 가능
- ***깊이 너무 안 함*** — 3~4단계까지. 그 이상은 사용 복잡
- ***주석으로 *예시와 의미** 명시
- ***enabled flag* 패턴** — `feature.enabled: false` → 비활성 시 자원 안 만듦

---

## 4. templates/ — *K8s manifest의 Go template*

### 파일 명명 컨벤션
- `deployment.yaml`, `service.yaml`, `ingress.yaml`, `configmap.yaml`, `hpa.yaml`, ...
- *underscore로 시작하는 파일* (`_helpers.tpl`) 은 *render 대상 X* — named template만 정의
- `NOTES.txt` — install 후 *출력 메시지*
- `tests/*.yaml` — `helm test`로 실행

### Render 동작
```bash
helm template mychart .            # 렌더링 결과만 출력 (apply X)
helm install myrelease . --dry-run --debug    # 시뮬레이션
helm install myrelease .           # 실제 cluster에 적용
```

→ Helm은 *template 디렉터리의 모든 .yaml 파일을 rendering* → *한 큰 YAML stream*으로 합쳐 *kubectl apply*와 비슷하게 적용.

### 자주 보는 deployment.yaml 패턴
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.port }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

→ Go template 문법은 [02-templating.md](02-templating.md).

---

## 5. _helpers.tpl — *Named Template*

여러 manifest에서 *반복되는 logic*을 *함수처럼* 정의:

```yaml
{{/*
fullname 패턴 — release name + chart name 조합
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
공통 라벨
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
selector 전용 라벨 (Deployment·Service selector에 일관 사용)
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

호출:
```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

### `include` vs `template`
- `template`은 *값 반환 X* (그냥 출력만, pipe에 못 씀)
- `include`는 *값 반환* (pipe `nindent`·`toYaml` 등 조합 가능)
- → **항상 `include`** 권장

---

## 6. sub-charts (`charts/` 디렉터리)

### Dependencies 추가
```yaml
# Chart.yaml
dependencies:
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

```bash
helm dependency update           # repository에서 fetch → charts/ 에 tgz
helm dependency list             # 의존성 목록
```

### Sub-chart에 values 전달
parent chart의 values.yaml:
```yaml
postgresql:                      # ← sub-chart 이름
  enabled: true
  auth:
    database: myapp
    username: myapp
  primary:
    persistence:
      size: 100Gi
```

→ `postgresql` chart 내부에선 `.Values.auth.database` 로 접근. parent의 `.Values.postgresql.auth.database` 는 *sub-chart 입장에선 `.Values.auth.database`*.

### Global Values
- *모든 sub-chart에 공통* 전달
```yaml
global:
  imageRegistry: registry.internal.com
  storageClass: gp3
```
→ 각 sub-chart에서 `.Values.global.imageRegistry`.

### condition / tags
- *condition*: 특정 values가 true일 때만 sub-chart 설치
- *tags*: *그룹화*. tag로 여러 chart 한꺼번에 enable/disable
```yaml
tags:
  database: true                 # tag: database 인 모든 dep enable
```

---

## 7. CRD 처리 — `crds/` 디렉터리 함정

```
mychart/
└── crds/
    └── mycrd.yaml
```

**중요한 동작:**
- `helm install` 시 *한 번 install* (helm이 *생성만*)
- `helm upgrade` 시 *기존 CRD 안 건드림* (Helm이 *수정·삭제 안 함*)
- `helm uninstall` 시 *삭제 안 함*
- → CRD *업그레이드·삭제는 수동* 또는 별도 도구

이유: CRD 삭제 = *그 CRD의 모든 인스턴스 삭제*. 위험해서 *Helm이 손 안 댐*.

→ 새 CRD 추가는 OK, *기존 CRD 업그레이드는* `kubectl apply -f mychart/crds/` 직접.

---

## 8. NOTES.txt — *install 후 출력*

```
{{- define "mychart.notes" -}}
1. Get the application URL:
{{- if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "mychart.fullname" . }})
  export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- end }}
{{- end }}
```

→ install 끝나면 *친절한 안내*. *어떻게 접근하는지·환경변수 등*.

---

## 9. .helmignore — *패키지에서 제외*

```
# 패턴은 .gitignore 와 비슷
.DS_Store
.git/
.gitignore
.idea/
*.swp
*.tmp
README.md.tmp
tests/
docs/
```

→ `helm package` 할 때 *최종 tgz에 안 포함*.

---

## 10. 자주 헷갈리는 것

### 10-1. *`helm create` 가 만든 기본 chart는 *예시*. 실제 운영용은 *대부분 새로 작성*.
- 기본은 *nginx 가정* — 자기 앱에 맞춰 *templates/ 거의 새로*

### 10-2. *_helpers.tpl 의 `define` 이름은 *chart 안에서 유일*해야*
- 같은 이름 두 번 define → *마지막 것만* 적용

### 10-3. *Subchart name 충돌*
- parent와 sub-chart의 fullname이 *겹치면 자원 충돌*
- → fullname 패턴에 *release name* 포함이 표준

### 10-4. *Sub-chart의 values에 *.Values.global* 접근 가능*
- *global* key는 *모든 chart에서 공유*
- 다른 sub-chart의 values는 *서로 접근 못 함*

### 10-5. *Chart.yaml `appVersion`은 *quoted string* 권장*
- `appVersion: 1.0` → *float로 해석되어 1*이 됨
- `appVersion: "1.0"` 로 *명시적 string*

### 10-6. *대용량 chart는 *.helmignore* 안 하면 패키지 거대화*
- test fixtures, docs, screenshots 등 *반드시 제외*

---

## 🎤 면접 빈출 Q&A

### Q1. Helm chart의 구조와 각 파일의 역할?
> Chart.yaml (메타: 이름·버전·의존성), values.yaml (기본값), templates/ (K8s manifest의 Go template), _helpers.tpl (named template, 함수처럼 재사용), charts/ (sub-chart 의존성), crds/ (CRD — helm은 *생성만* 함, upgrade·delete 안 함), NOTES.txt (install 후 메시지), .helmignore (패키지 제외), values.schema.json (JSON Schema 검증).

### Q2. chart version과 appVersion 차이?
> *chart version* = chart 자체 버전 (구조·template 변경 시 ↑). *appVersion* = chart가 패키징한 *앱의 버전* (image tag와 보통 매칭). 둘 독립적. 예: 앱 image만 새 버전 (chart 구조 동일)이면 *appVersion만* ↑, chart version은 그대로.

### Q3. _helpers.tpl 의 `include` vs `template` 차이?
> `template` 은 *값 반환 X* — 그냥 출력만. pipe에 못 씀. `include` 는 *값 반환* — `nindent`·`toYaml` 등 *pipe 조합 가능*. **항상 `include` 권장**.

### Q4. sub-chart에 values 어떻게 전달?
> parent values.yaml의 `{subchart-name}:` 키 아래 sub-chart의 values 구조 그대로. 예: postgresql sub-chart라면 parent에 `postgresql: { auth: { database: ... } }`. sub-chart 내부에선 `.Values.auth.database` 로 접근. *모든 sub-chart 공통값*은 `global:` 으로.

### Q5. CRD를 chart에 어떻게 포함하나요? helm upgrade 시 동작?
> `crds/` 디렉터리에. **helm은 install 시 *한 번 생성*, upgrade·uninstall 시 *건드리지 않음***. 이유: CRD 삭제 = *모든 인스턴스 삭제* 위험. 새 CRD 추가는 OK, *기존 CRD 업그레이드는 수동* (`kubectl apply -f`) 또는 *별도 자동화 (Operator)*. 또는 chart 안 `templates/` 에 넣으면 *일반 자원처럼 관리* (그러나 위험).

### Q6. *.helmignore* 안 하면?
- *모든 파일이 chart 패키지 (.tgz)에 포함*
- README screenshots, test fixtures, docs로 *패키지 거대화* + 옛 *secret 파일이 실수로 포함될 수도*
- 최소: `.git/`, `.idea/`, `*.swp`, `tests/` (실제 helm test 자원과 다른 *test fixtures* 라면)

### Q7. 좋은 values.yaml 설계 원칙?
> (1) *모든 값에 기본값* — 사용자가 *최소 명세*로 동작. (2) *깊이 3~4단계*까지. 그 이상은 사용 복잡. (3) *주석으로 예시·의미 명시*. (4) *`enabled` flag 패턴* — `feature.enabled: false`로 *비활성 시 자원 안 만듦*. (5) *Bitnami 컨벤션 참고* — image / service / ingress / resources / autoscaling / podSecurityContext 등 표준 구조.

---

## 🔗 Cross-reference

- **Go template 문법·sprig** → [02-templating.md](02-templating.md)
- **values 우선순위 + override** → [04-values-design.md](04-values-design.md)
- **dependency 관리·OCI repo** → [05-repos-and-distribution.md](05-repos-and-distribution.md)
- **K8s 자원 종류** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **PodSecurityContext (securityContext의 chart 패턴)** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)

---

## 📝 3줄 요약

1. Chart = *Chart.yaml + values.yaml + templates/ + charts/(sub-chart) + _helpers.tpl + crds/* 구조의 패키지.
2. *chart version (chart 구조) vs appVersion (앱)* 독립. CRD는 `crds/` — *helm이 생성만 하고 upgrade·delete는 안 함*.
3. _helpers.tpl 의 named template은 *`include`로 호출* (pipe 조합용). sub-chart는 *parent values의 `{subchart}:` 아래*에 values 전달. *global:* 은 모든 sub-chart 공통.

# 02 — Templating: Go Template + Sprig

> **공식 근거:** [Go text/template](https://pkg.go.dev/text/template), [Sprig functions](https://masterminds.github.io/sprig/), [Helm Template Guide](https://helm.sh/docs/chart_template_guide/)

---

## 🎯 한 문장

> **Helm template = *Go text/template* + *Sprig 함수 라이브러리* + *Helm 내장 객체* (`.Values`, `.Release`, `.Chart`, `.Files`, ...). 결과는 *최종 YAML*. *공백·들여쓰기에 매우 민감*.**

---

## 1. 기본 문법 — `{{ ... }}` 블록

```yaml
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
```

- `{{ }}` 안 = template 표현식
- `{{-` (왼쪽에 `-`) = *왼쪽 공백 제거*
- `-}}` (오른쪽에 `-`) = *오른쪽 공백 제거*
- *공백 제거 안 하면 YAML 들여쓰기 깨짐*

---

## 2. Helm 내장 객체

### `.Values`
- values.yaml 에서 정의된 값 + `-f` / `--set` override
- 가장 자주 사용

### `.Release`
- `.Release.Name` — release 이름 (e.g. `myapp-prod`)
- `.Release.Namespace` — 설치 namespace
- `.Release.IsInstall` / `.Release.IsUpgrade` — install/upgrade 여부
- `.Release.Service` — *항상 "Helm"*
- `.Release.Revision` — 현재 revision 번호

### `.Chart`
- `.Chart.Name` — Chart.yaml의 name
- `.Chart.Version` — chart version
- `.Chart.AppVersion` — app version

### `.Capabilities`
- `.Capabilities.KubeVersion` — cluster K8s 버전
- `.Capabilities.APIVersions.Has "apps/v1"` — 특정 API 사용 가능 여부

### `.Files`
- `.Files.Get "config.yaml"` — chart 안 파일 읽기
- `.Files.Glob "configs/*.yaml"` — 패턴 매칭
- 흔한 활용: *ConfigMap 안에 chart 디렉터리의 외부 파일 임베드*

### `.Template`
- `.Template.Name` — 현재 template 이름
- `.Template.BasePath` — chart의 templates/ 경로

---

## 3. 자주 쓰는 *Action* (제어 구조)

### `if / else if / else`
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- end }}
```

### `with` — *컨텍스트 변경*
```yaml
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 2 }}    # . = .Values.podAnnotations
{{- end }}
```

- `with`는 *그 블록 안에서 . 의 의미*를 바꿈
- 외부 . 접근하려면 `$` 또는 위 변수 캡처

### `range` — *반복*
```yaml
{{- range .Values.hosts }}
- host: {{ .host }}
  paths:
  {{- range .paths }}
  - path: {{ .path }}
    pathType: {{ .pathType }}
  {{- end }}
{{- end }}
```

```yaml
# 인덱스도 받기
{{- range $i, $host := .Values.hosts }}
- index: {{ $i }}
  host: {{ $host.host }}
{{- end }}
```

### `define` / `include` (named template, [01 참조](01-chart-structure.md))

### 변수 (`$`로 시작)
```yaml
{{- $name := "myapp" }}
{{- $port := .Values.service.port }}
metadata:
  name: {{ $name }}
```

### `$` (root context)
- `range` / `with` 안에서 *외부 `.` 접근* 어려움 → `$`로 *항상 root 접근*
```yaml
{{- range .Values.containers }}
- name: {{ .name }}
  image: "{{ .image }}:{{ $.Values.global.tag }}"     # $로 root.Values.global.tag 접근
{{- end }}
```

---

## 4. *공백·들여쓰기* — 가장 자주 깨지는 부분

### `nindent` vs `indent`
- `indent N` — 각 줄 앞에 *N칸 공백*
- `nindent N` — *앞에 줄바꿈 추가 + N칸 공백*

```yaml
spec:
  template:
    metadata:
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
```

→ 보통 *`nindent`* 가 안전. 윗 줄과 *분리되어 깔끔*.

### `-` 의 정확한 의미
```yaml
foo: bar
{{- if .Values.x }}             # ← 위 줄 끝 공백 + 줄바꿈 제거
baz: qux
{{- end -}}                     # ← 오른쪽 공백·줄바꿈도 제거 (다음 줄 붙음)
next: value
```

→ 너무 많이 `-` 쓰면 *YAML 한 줄로 합쳐져 깨짐*. 보통 *왼쪽만* `{{-`.

### 결과 확인
```bash
helm template . --debug | less
```
→ *렌더링 결과* 확인. *공백 사고* 잡기.

---

## 5. *파이프* (`|`) — 함수 chain

```yaml
labels:
  app: {{ .Values.appName | quote }}
  hash: {{ .Values.config | sha256sum | trunc 8 }}
```

- 왼쪽 → 오른쪽으로 *값 흘려보내기*
- `quote` — 문자열 quote (`"`)
- `upper` / `lower` — 대소문자
- `trunc N` — N자 까지
- `trimSuffix` / `trimPrefix`
- `default` — 빈 값 시 기본값

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

→ `image.tag` 빈 값이면 `Chart.AppVersion` 사용.

---

## 6. Sprig — *함수 라이브러리*

Helm 은 *Sprig 의 거의 모든 함수* 사용 가능.

### 문자열
- `upper`, `lower`, `title`, `quote`, `squote`
- `trim`, `trimAll`, `trimPrefix`, `trimSuffix`
- `contains`, `hasPrefix`, `hasSuffix`
- `replace`, `regexReplaceAll`, `splitList`
- `printf "%s-%s" .Release.Name .Chart.Name`

### 숫자
- `add`, `sub`, `mul`, `div`, `mod`
- `min`, `max`
- `int`, `float64`

### 컬렉션
- `list 1 2 3`
- `dict "key" "value"`
- `keys`, `values`, `has`, `get`
- `pluck "key" $list` — list의 각 dict에서 그 key 추출
- `concat` — 합치기
- `uniq` — 중복 제거

### Date / Time
- `now`
- `dateInZone "2006-01-02" .now "UTC"`

### Encoding
- `b64enc` / `b64dec` — base64
- `sha256sum` — SHA-256 해시
- `toJson` / `fromJson`
- `toYaml` / `fromYaml`

### 흐름·논리
- `eq`, `ne`, `lt`, `gt`, `le`, `ge`
- `and`, `or`, `not`
- `empty` — 빈 값 체크

```yaml
{{- if and .Values.ingress.enabled (not .Values.ingress.tls) }}
# ...
{{- end }}
```

### `lookup` — *cluster 자원 조회* (위험·드뭄)
```yaml
{{- $existingSecret := lookup "v1" "Secret" "default" "my-secret" -}}
{{- if $existingSecret }}
# 기존 secret 활용
{{- else }}
# 새로 생성
{{- end }}
```
- 단점: *idempotent 깨짐* (cluster 상태에 따라 결과 변동) — *render-time 결정*. **`helm template` 만 하면 lookup 결과 비어 있음**.

---

## 7. *`required` / `fail`* — 명시적 검증

```yaml
{{- if not .Values.database.password }}
{{- fail "database.password is required" }}
{{- end }}

# 또는 한 줄
password: {{ required "database.password is required" .Values.database.password }}
```

→ values 안 줘서 *조용히 빈 string*으로 렌더링되는 사고 방지.

### values.schema.json — *더 정석*
JSON Schema로 values.yaml 검증:
```json
{
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" }
      }
    },
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    }
  }
}
```

→ `helm install` 자동 검증. *잘못된 type / 누락된 필드* 즉시 차단.

---

## 8. `checksum/config` 패턴 — *ConfigMap 변경 시 자동 rollout*

문제: ConfigMap 변경해도 *Pod 자동 재시작 안 됨* ([`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md))

해결: ConfigMap 내용 해시를 *Pod template annotation에 박기*:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
      labels:
        ...
```

→ ConfigMap 변경 → 그 파일 *render 결과* 해시 변경 → Pod template annotation 변경 → Deployment의 *template hash 변화* → *RollingUpdate 자동*.

→ Helm 운영의 *가장 흔한 패턴 중 하나*.

---

## 9. *외부 파일 포함* — `.Files`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    {{- .Files.Get "configs/nginx.conf" | nindent 4 }}
```

```bash
mychart/
├── configs/
│   └── nginx.conf       # 일반 파일
└── templates/
    └── configmap.yaml
```

→ 큰 설정 파일을 *template 안에 inline 안 쓰고* 외부 파일로 분리. *읽기·유지보수 쉬움*.

```yaml
# 여러 파일을 한 ConfigMap에
data:
{{- range $path, $_ := .Files.Glob "configs/*" }}
  {{ base $path }}: |
    {{- $.Files.Get $path | nindent 4 }}
{{- end }}
```

---

## 10. 자주 헷갈리는 것

### 10-1. *`{{- }}` 의 왼쪽 / 오른쪽 효과*
- 왼쪽 `{{-` = *앞 공백·줄바꿈 제거*
- 오른쪽 `-}}` = *뒤 공백·줄바꿈 제거*
- 양쪽 다 쓰면 *다음 줄과 합쳐짐* → 자주 깨짐
- **보통 왼쪽만** `{{-`, 오른쪽은 그냥 `}}`

### 10-2. *`if` 안에서 `with` 컨텍스트 변경*
```yaml
{{- with .Values.podAnnotations }}
{{- if eq .key "value" }}        # ← . 은 podAnnotations
{{- end }}
{{- end }}
```
- `with` 안에선 `.` 의미가 *그 값*으로 바뀜
- 외부 접근하려면 `$` 사용

### 10-3. *`range` 안에서 변수 capture*
```yaml
{{- range $i, $host := .Values.hosts }}
# 여기선 $i, $host로 접근
{{- end }}
```
- 인덱스 + 값 한꺼번에
- 외부 `.` 접근은 `$`

### 10-4. *`toYaml` 시 들여쓰기*
```yaml
# 잘못된 예
spec:
  template:
    metadata:
      annotations:
        {{ toYaml .Values.podAnnotations }}    # 들여쓰기 깨짐
```
```yaml
# 올바른
spec:
  template:
    metadata:
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
```

### 10-5. *`default`는 *false / 0 / 빈 string* 도 default 적용*
```yaml
replicas: {{ .Values.replicaCount | default 1 }}
# replicaCount: 0 이면 → 1 됨 (의도 아닐 수도)
```
→ *명시적으로 nil 체크* 필요할 때 `nil` 비교.

### 10-6. *`include` 안에 `.` 전달 안 하면 빈 결과*
```yaml
{{ include "mychart.labels" }}              # 잘못 — . 누락
{{ include "mychart.labels" . }}             # 올바름
```
- named template 안에서 `.` 사용하므로 *호출 시 반드시 전달*

---

## 🎤 면접 빈출 Q&A

### Q1. Helm template의 기본 객체는?
> `.Values` (values.yaml + override), `.Release` (Name·Namespace·Revision·IsUpgrade), `.Chart` (Name·Version·AppVersion), `.Capabilities` (cluster K8s 버전·API 사용 가능), `.Files` (chart 안 외부 파일 읽기), `.Template` (현재 template 이름·경로).

### Q2. `checksum/config` annotation 패턴이 왜 필요?
> ConfigMap·Secret 변경 시 *Pod 자동 재시작 안 됨* (K8s 기본 동작). 해결: ConfigMap/Secret의 *render 결과 해시*를 Pod template annotation에 박음 → 내용 변경 시 *Pod template hash 변화* → RollingUpdate 자동 trigger. `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}` 패턴.

### Q3. `include` vs `template` 차이?
> `template` 은 *값 반환 X* (출력만, pipe 못 씀). `include` 는 *값 반환* — `nindent`·`toYaml` 등 pipe 조합 가능. **항상 `include` 권장**. 호출 시 *컨텍스트 `.` 반드시 전달*.

### Q4. `with` 안에서 외부 `.` 접근하려면?
> `$` (root context) 또는 외부에서 변수로 캡처 후 사용. `{{- with .Values.x }}{{ $.Values.global.y }}{{- end }}`. `range` 안에서도 마찬가지.

### Q5. values 잘못된 type / 누락 검증은?
> (1) `required "msg" .Values.x` — 빈 값이면 fail. (2) `values.schema.json` — JSON Schema로 type·required·minimum 등 *helm install 시 자동 검증*. 더 정석은 schema. 잘못된 type / 누락 즉시 차단 → *런타임 사고 예방*.

### Q6. `lookup` 함수는 무엇? 왜 위험?
> 렌더링 시점에 *cluster 자원 조회*. 기존 secret 활용·기존 자원 참조 등. **위험**: (1) *idempotent 깨짐* — cluster 상태에 따라 결과 변동. (2) `helm template` 만 하면 *lookup 결과 빈 값* (cluster 접근 X). (3) *ArgoCD 등 GitOps 도구와 잘 안 맞음* (manifest 일관성 깨짐). 가능하면 회피, 진짜 필요할 때만.

### Q7. 외부 큰 설정 파일을 chart에 어떻게 포함?
> `.Files.Get "configs/nginx.conf"` — chart 디렉터리의 외부 파일 읽어 *ConfigMap data에 inline*. 큰 nginx config·SQL 초기화 스크립트 등 *template 안에 inline 안 하고 외부 파일 분리* → 읽기·유지보수 쉬움. `.Files.Glob "configs/*"` 로 *여러 파일 한 번에*.

---

## 🔗 Cross-reference

- **chart 구조·_helpers.tpl** → [01-chart-structure.md](01-chart-structure.md)
- **values 우선순위·schema·override** → [04-values-design.md](04-values-design.md)
- **ConfigMap 변경 자동 rollout 패턴** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md)
- **ArgoCD가 helm template 내부 호출** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. *Go template + Sprig + Helm 내장 객체*. `.Values` `.Release` `.Chart` `.Capabilities` `.Files` `.Template`.
2. *공백·들여쓰기 매우 민감*. `{{-` (왼쪽 trim) + `nindent N` 패턴이 안전. `helm template --debug` 로 확인.
3. *`checksum/config` 패턴*으로 ConfigMap 변경 자동 rollout. *`required` / `values.schema.json`*으로 검증. `lookup` 은 idempotent 깨므로 회피.

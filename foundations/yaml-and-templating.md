# YAML & Templating — 설정의 공용 언어

> k8s manifest, helm chart, ansible playbook, docker-compose, GitHub Actions workflow,
> Prometheus rule, ArgoCD Application — 전부 YAML.
> 그리고 그중 다수가 *템플릿 위의 YAML*. 두 레이어의 함정이 곱해진다.

---

## 1. YAML 자체의 함정

### YAML 1.1 vs 1.2 — "Norway problem"

```yaml
country: NO   # 1.1에서 boolean false. 1.2에서 string "NO". 노르웨이가 사라진다.
enabled: yes  # 1.1: true. 1.2: string "yes".
power: on     # 1.1: true. 1.2: string "on".
```

대부분의 도구가 YAML 1.1 parser (go-yaml v2, PyYAML 기본). 의도가 string이면 **따옴표로 감싸라**:

```yaml
country: "NO"
enabled: "yes"
version: "1.10"   # 따옴표 없으면 float 1.1
```

`1.10` 같은 버전 문자열이 *float 1.1로 파싱* 되는 게 helm·k8s에서 자주 일어남.

### Block scalar — `|` vs `>`

```yaml
script: |              # literal: 줄바꿈 보존
  #!/bin/sh
  echo hello

description: >         # folded: 줄바꿈을 스페이스로
  this is
  a long line          # → "this is a long line\n"

# 끝 줄바꿈 제어
keep: |+               # +: 마지막 줄바꿈 *모두 보존*
strip: |-              # -: 마지막 줄바꿈 *제거*
```

container `command:` 나 ConfigMap의 script를 적을 때 `|` vs `>` 를 헷갈리면 script가 *한 줄로 합쳐져* 작동 안 함.

### Multi-document

```yaml
---
kind: ConfigMap
...
---
kind: Service
...
```

`---` 가 *문서 구분자*. `kubectl apply -f file.yaml` 이 여러 리소스 한 번에 처리하는 방식.

### YAML이 *자주* 헷갈리는 또 다른 것들

- `null`, `Null`, `NULL`, `~`, 빈 값 모두 null
- `0x10` → 16, `010` → 8 (octal, 1.1만), `1_000_000` → 1000000 (1.2+)
- `[]` 와 `[ ]` 와 `\n  -` 가 다 빈 list 또는 list로 파싱 (flow vs block style)
- 들여쓰기는 *스페이스만*. 탭은 에러.

---

## 2. Anchor·Alias·Merge — *재사용*

### Anchor (`&`) + Alias (`*`)

```yaml
defaults: &defaults
  retries: 3
  timeout: 30

job-a:
  <<: *defaults    # merge (1.1+)
  name: a

job-b:
  <<: *defaults
  name: b
  retries: 5       # override
```

`<<` (merge key)는 YAML 1.1 spec. 1.2에서 *공식 제거* 됐지만 대부분의 parser가 여전히 지원.

### 어디서 자주 보나

- docker-compose의 service 정의 재사용
- gitlab-ci의 job 공통 설정
- Prometheus alert rule의 공통 label

### 한계

- k8s가 직접 anchor를 *지원하지 않는다*. `kubectl apply -f` 가 *parsing은* 하지만, 그 결과를 etcd에 저장할 때 anchor는 *풀려서 평탄화* 됨. 의도한 "한 곳 수정으로 여러 곳 반영"이 *클러스터에 적용된 후엔* 안 됨.
- → 진짜 재사용은 **helm·kustomize** 같은 *외부 템플릿/오버레이* 로.

---

## 3. 두 가지 큰 템플릿 엔진 — Go template vs Jinja2

### 누가 어디서 쓰나

| 엔진 | 사용 도구 |
|---|---|
| **Go template** (`{{ }}`) | helm, kubectl `--template`, Hugo, terraform `templatefile`, Prometheus alert label 일부 |
| **Jinja2** (`{{ }}`, `{% %}`) | ansible, salt, FastAPI templates, Flask, mkdocs |

문법이 *비슷해 보이지만* 의미가 다르다. 한쪽 익혀서 다른 쪽 가면 미묘하게 깨진다.

### 변수 scope

**Go template**: 명시적으로 *현재 컨텍스트* 라는 객체가 있음 (`.`).

```gotemplate
{{ .Values.image.repository }}
{{ range .Values.servers }}
  {{ .name }}           # range 안에서는 . 이 element
{{ end }}
{{- $top := . -}}       # 변수로 저장 가능
```

**Jinja2**: Python처럼 *플랫* 한 변수 공간.

```jinja2
{{ image.repository }}
{% for server in servers %}
  {{ server.name }}
{% endfor %}
```

→ helm 작성자가 ansible 가면 `.` 빼는 걸 잊고, ansible 작성자가 helm 가면 `.` 빠뜨려서 깨진다.

### Whitespace 제어

**Go template**: `{{- ` 와 ` -}}` 가 *주변 공백·줄바꿈 trim*.

```gotemplate
{{- range .items }}
- {{ . }}
{{- end }}
```

YAML 들여쓰기를 맞추려면 *공백 제어가 거의 필수*. 빠뜨리면 rendering 결과가 *invalid YAML*.

**Jinja2**: `{%- %}` 로 비슷한 제어. 또한 ansible은 *Python 객체* 를 그대로 다루기 때문에 string vs list 혼동이 덜함.

### 기본값·조건

```gotemplate
{{ .Values.image.tag | default "latest" }}
{{ if .Values.ingress.enabled }}...{{ end }}
{{ with .Values.podAnnotations }}...{{ end }}
```

```jinja2
{{ image_tag | default('latest') }}
{% if ingress_enabled %}...{% endif %}
```

### Sprig — Go template의 *확장 함수*

Go template은 *기본 함수가 빈약*. helm·terraform은 [sprig](https://masterminds.github.io/sprig/) 라이브러리를 추가:

```gotemplate
{{ .Values.name | upper | quote }}
{{ randAlphaNum 16 }}
{{ now | date "2006-01-02" }}
{{ .Values.password | b64enc }}    # k8s Secret data에 필수
```

sprig 없이는 helm이 사실상 불가능.

---

## 4. *템플릿 위의 YAML* 의 함정

### 들여쓰기가 *런타임에* 결정된다

```yaml
spec:
  containers:
    - name: app
      env:
        {{- range .Values.envVars }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
```

`{{ range }}` 가 *어디서부터 어디까지의 공백* 을 잡아먹는지에 따라 결과가 *invalid YAML*. 디버깅:

```bash
helm template my-release ./chart > rendered.yaml
yamllint rendered.yaml
kubectl apply --dry-run=client -f rendered.yaml
```

*템플릿 자체* 를 보지 말고 *렌더 결과* 를 봐라.

### 멀티라인 값

```yaml
data:
  config.yaml: |
    {{- .Values.config | nindent 4 }}
```

`nindent N` = "N 스페이스로 들여쓰기한 후 앞에 줄바꿈 추가". sprig 함수. 멀티라인 값 삽입에는 거의 필수.

### Quote 함정

```yaml
name: {{ .Values.name }}        # name이 "yes" 면 boolean true로 파싱
name: {{ .Values.name | quote }} # 안전 — "yes" 로 감쌈
```

helm의 `| quote` 는 *디폴트로 켜둬라* 가 안전한 권장.

---

## 5. Schema 검증

YAML이 *문법적으로 valid* 라도 *의미적으로 invalid* 일 수 있음 (오타, 잘못된 field 이름).

### 도구

| 도구 | 검증 대상 |
|---|---|
| `yamllint` | YAML 문법, 들여쓰기, line length |
| `kubeconform` | k8s manifest의 schema (OpenAPI 기반, kubeval의 fork) |
| `helm lint` | helm chart 자체 |
| `kubectl apply --dry-run=server` | apiserver 측 admission 포함 검증 |
| `conftest` (OPA Rego) | 정책 검증 ("모든 Pod은 limits 있어야 함") |
| `datree`, `polaris` | best practice 검증 |

CI 파이프라인에서 `helm template` → `kubeconform` → `conftest` 가 *manifest를 cluster 닿기 전에* 거르는 표준 조합.

### OpenAPI = k8s의 *진실*

k8s의 모든 리소스는 OpenAPI v3 스키마를 갖는다. apiserver가 `/openapi/v3` 로 노출. CRD도 자기 스키마를 *반드시* 제공해야 함 (v1+). 모든 *유효한 field 이름·타입* 은 거기서 옴.

`kubectl explain pod.spec.containers.resources.limits` → 그 스키마를 인간 친화적으로 읽어주는 명령.

---

## 6. Cross-reference 지도

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| Helm chart 구조, _helpers.tpl | `helm-basics/` chart·template |
| Ansible playbook의 Jinja2, filter | `ansible-basics/` playbook |
| K8s manifest의 각 field 의미 | `kubernetes-basics/` workload·service |
| Kustomize의 patch·overlay (anchor의 대안) | `kubernetes-basics/` 또는 `cicd-gitops-basics/` ArgoCD |
| Prometheus rule의 label template | `observability-basics/` Prometheus |
| GitHub Actions workflow YAML 구조 | `cicd-gitops-basics/` |
| 멱등한 template render = 선언형의 전제 | [[declarative-and-reconciliation]] §2 |

---

## 한 줄 요약

> **YAML은 *쉬워 보이는데 함정이 많은* 포맷이다.** 1.1/1.2 차이, boolean trap, block scalar, anchor의 한계.
> 템플릿 위의 YAML은 *두 레이어의 함정이 곱해진다*. 렌더 결과를 검증하지 않으면 cluster에 닿아서야 깨진다.

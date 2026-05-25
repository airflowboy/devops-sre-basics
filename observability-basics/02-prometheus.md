# 02 — Prometheus: pull model · PromQL · exporter

> **공식 근거:** [Prometheus docs](https://prometheus.io/docs/), [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/), [Prometheus Operator](https://prometheus-operator.dev/)

---

## 🎯 한 문장

> **Prometheus = *pull 기반 시계열 DB + scraper + alerting*. *target의 `/metrics` endpoint를 주기적으로 scrape* → 자체 TSDB 저장 → PromQL로 쿼리. *K8s에선 Prometheus Operator (ServiceMonitor/PrometheusRule)*로 선언적 관리.**

---

## 1. Pull Model — *왜 pull인가*

### 동작
```
                  Prometheus
                  ────────────
                  │  every 15s:  GET http://app:8080/metrics  │
                  ────────────
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
   ┌─────┐         ┌─────┐         ┌─────┐
   │ App │         │ App │         │ App │
   │/metrics│      │/metrics│      │/metrics│
   └─────┘         └─────┘         └─────┘
```

- Prometheus가 *target에 HTTP GET*
- target은 *plain text format으로 metric 응답*
- 보통 *15s ~ 60s 간격*

### 응답 format (text-based)
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1024
http_requests_total{method="GET",status="500"} 5
http_requests_total{method="POST",status="200"} 256

# HELP cpu_usage_percent Current CPU usage
# TYPE cpu_usage_percent gauge
cpu_usage_percent 42.5
```

### Pull의 장점
- **Target liveness 자동 체크** — scrape 실패 = `up{...}=0` (자동 알람)
- **방화벽 친화** — Prometheus *한 방향*만 (Prometheus → target)
- **service discovery 친화** — target 목록 *동적 갱신* (K8s, EC2, Consul 등에서)
- **재구성 쉬움** — *target 추가/제거* 가 *Prometheus config만 변경*

### Pull의 단점
- **short-lived job** — 작업이 끝나면 metric 못 받음 → *Pushgateway* 사용
- **NAT 뒤 target** — Prometheus가 도달 못 함 → 같은 해결 (Pushgateway)
- **client-side aggregation 없음** — high cardinality 사고 위험

### Pushgateway — *예외*
- short-lived job 전용
- job이 *Pushgateway에 push* → Prometheus가 *Pushgateway를 scrape*
- *항상 last value 유지* (자동 만료 X) — 주의

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│              Prometheus Server                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Service  │  │ Scraper  │  │   TSDB           │  │
│  │Discovery │→ │ (HTTP    │→ │ (local on-disk)  │  │
│  │ (K8s API)│  │  GET)    │  │                  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                       │              │
│  ┌──────────────────────────┐         ▼              │
│  │ PromQL engine (queries)  │ ◄───────┘              │
│  └──────────────────────────┘                        │
│                                       │              │
│  ┌──────────────────────────┐         ▼              │
│  │ Alerting Rules           │ → Alertmanager         │
│  └──────────────────────────┘                        │
└─────────────────────────────────────────────────────┘
                        ▲
                        │ HTTP API (Grafana, kubectl 등)
```

---

## 3. Service Discovery — *target 자동 발견*

### Static
```yaml
# prometheus.yml
scrape_configs:
- job_name: 'myapp'
  static_configs:
  - targets: ['app1:8080', 'app2:8080']
```

### K8s SD (가장 흔함)
```yaml
scrape_configs:
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    target_label: __metrics_path__
    regex: (.+)
```

→ *Pod에 annotation 박으면 자동 scrape*:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

### EC2 / GCE / Azure SD
- *cloud API*에서 인스턴스 자동 발견
- tag로 필터링

### Consul / Eureka / Nomad SD
- service discovery 도구 통합

---

## 4. Exporter — *외부 시스템의 metric을 Prometheus format으로*

### 발상
- 모든 시스템이 *Prometheus 형식 X*
- *Exporter = 그 시스템 metric을 Prometheus format으로 변환*하는 *adapter*

### 자주 쓰는 exporter
| Exporter | 대상 |
|---|---|
| **node_exporter** | Linux 호스트 (CPU, memory, disk, network) |
| **kube-state-metrics** | K8s 자원 상태 (Pod count, Deployment status, etc.) |
| **cAdvisor** | container metric (kubelet에 내장) |
| **blackbox_exporter** | 외부 endpoint probe (HTTP/HTTPS/TCP/ICMP/DNS) |
| **postgres_exporter** | PostgreSQL |
| **mysql_exporter** | MySQL |
| **redis_exporter** | Redis |
| **nginx_exporter** | Nginx |
| **JMX exporter** | Java JMX |
| **AWS CloudWatch exporter** | CloudWatch → Prometheus |

→ *사실상 모든 인기 시스템에 exporter 존재*. Awesome Prometheus Exporters 목록.

### Direct instrumentation
- 앱 코드에 *Prometheus client library*로 *직접 expose*
- Go·Python·Java·Node·Ruby·Rust 등 *공식 라이브러리*
- *exporter 불필요* — 앱이 직접 `/metrics`

```python
from prometheus_client import Counter, Histogram, start_http_server

requests_total = Counter('http_requests_total', 'Total requests', ['method', 'status'])
request_duration = Histogram('http_request_duration_seconds', 'Request duration')

@request_duration.time()
def handle_request():
    ...
    requests_total.labels(method='GET', status='200').inc()

start_http_server(8000)   # /metrics endpoint 자동 노출
```

---

## 5. TSDB (Time-Series DB)

### 저장 단위
- **Sample**: (timestamp, value) — 8 bytes 정도
- **Time series**: *같은 metric name + label 조합*의 sample 시퀀스
- **Block**: 2시간치 sample (default) — `data/blocks/<block-id>/`

### Retention
```yaml
# prometheus.yml
storage:
  tsdb:
    retention.time: 15d         # 기본 15일
    retention.size: 50GB        # 또는 크기 제한
```

### Compaction
- block은 시간 지나면 *압축·병합*
- *옛 block일수록 더 큰 단위로*

### 한계
- *단일 노드*. *수억 series*면 메모리 부족.
- *장기 보관* 어려움 — 15일이 기본.
- **해결: Remote Storage** (Thanos, Cortex, Mimir, VictoriaMetrics, Grafana Cloud)
  - Prometheus가 *remote_write*로 외부 저장소에 push
  - 장기 보관·query federation 가능

---

## 6. PromQL — *시계열 쿼리 언어*

### 기본 selector
```promql
# 모든 series
up

# label 필터
http_requests_total{method="GET", status="200"}

# regex
http_requests_total{status=~"5.."}     # 5xx 모두
http_requests_total{status!~"2.."}     # 2xx 아닌 것

# offset (5분 전)
http_requests_total offset 5m
```

### Range vector — `[5m]`
```promql
http_requests_total[5m]    # 최근 5분간의 모든 sample (range vector)
```

→ *range vector는 *집계 함수에 입력*. instant vector와 다름.

### `rate` / `irate` — *Counter 의 초당 증가율*
```promql
rate(http_requests_total[5m])     # 5분 평균 초당 증가율 (대시보드용)
irate(http_requests_total[1m])    # 마지막 2 sample 기반 (변화 감지용)
```

- **rate** = (last - first) / time span — *부드러운 평균*
- **irate** = (last - prev) / time gap — *반응 빠름, 노이즈 많음*

→ **대시보드는 `rate`**, **알람은 `rate`**, *순간 spike 감지엔 `irate`*.

### `increase` — counter 의 *증가량*
```promql
increase(http_requests_total[1h])   # 1시간 동안 증가량
```

→ `rate(...) * time_span` 과 같은 값. *해석이 직관적*.

### 집계 함수
```promql
# 합산
sum(rate(http_requests_total[5m]))

# label별 집계
sum by (service) (rate(http_requests_total[5m]))
sum without (instance) (rate(http_requests_total[5m]))   # instance 무시하고 합산

# 평균·최대·최소
avg by (service) (cpu_usage_percent)
max by (instance) (cpu_usage_percent)
min(memory_available_bytes)

# top N
topk(10, rate(http_requests_total[5m]))
bottomk(5, up)

# count
count by (service) (up == 1)         # service별 active 인스턴스 수
```

### Histogram quantile
```promql
# p99 latency
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# service별 p95
histogram_quantile(0.95,
  sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

→ `_bucket` suffix 가 *histogram의 raw 데이터*. `le` (less than or equal) label로 bucket 구분.

### 산술 / 비교
```promql
# 에러율
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# 알람 조건
rate(http_requests_total{status=~"5.."}[5m]) > 0.1

# 두 metric 결합
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

→ *벡터 매칭* — label이 *완전히 일치해야* 결합 가능. 아니면 `on(...)`, `ignoring(...)`, `group_left(...)` 활용.

### Time / Range 함수
- `predict_linear(metric[1h], 4*3600)` — 1시간 데이터로 4시간 뒤 *선형 예측* (디스크 가득 차는 시점 예측 등)
- `deriv(metric[5m])` — 미분 (gauge의 변화율)
- `delta(metric[5m])` — gauge의 변화량
- `changes(metric[5m])` — 값 변경 횟수

---

## 7. Recording Rules — *pre-aggregate*

### 발상
- 같은 쿼리를 *대시보드 여러 곳에서 반복* → 매번 계산
- *복잡한 쿼리는 느림* — 매 grafana refresh마다 부담
- **Recording rule**: *쿼리 결과를 *새 metric으로 저장**

```yaml
# rules.yml
groups:
- name: my_rules
  interval: 30s
  rules:
  - record: job:http_requests:rate5m
    expr: sum by (job) (rate(http_requests_total[5m]))
  
  - record: job:http_error_rate:ratio5m
    expr: |
      sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
      /
      sum by (job) (rate(http_requests_total[5m]))
```

→ Grafana는 `job:http_error_rate:ratio5m` 직접 query → 빠름.

### 명명 컨벤션
- `level:metric:operation` (예: `job:http_requests:rate5m`)
- *aggregation level (job, instance...) : metric 이름 : 연산자_window*

---

## 8. Prometheus Operator (K8s)

### 발상
- Prometheus를 K8s에서 *수동 운영*은 *config 변경마다 reload* 등 번잡
- **Prometheus Operator** = *Prometheus·Alertmanager·ServiceMonitor·PrometheusRule을 *CRD로 관리***

### 주요 CRD
| CRD | 역할 |
|---|---|
| **Prometheus** | Prometheus instance |
| **Alertmanager** | Alertmanager instance |
| **ServiceMonitor** | *어떤 Service의 어떤 endpoint를 scrape* |
| **PodMonitor** | Service 없는 Pod scrape |
| **PrometheusRule** | recording rule + alerting rule |
| **AlertmanagerConfig** | Alertmanager 라우팅 (per-namespace) |

### ServiceMonitor 예시
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    release: prometheus              # Prometheus가 watch할 selector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http                       # Service의 명명된 port
    path: /metrics
    interval: 30s
```

→ Operator가 *ServiceMonitor watch* → *Prometheus config 자동 갱신* → reload.

### PrometheusRule 예시
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-rules
  labels:
    release: prometheus
spec:
  groups:
  - name: my-app
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: |
        sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
        / sum by (service) (rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.service }}"
        description: "Error rate is {{ $value | humanizePercentage }}"
```

→ 자세히 [03-alerting.md](03-alerting.md).

### kube-prometheus-stack
- Helm chart `kube-prometheus-stack` — *Prometheus + Operator + Grafana + Alertmanager + node_exporter + kube-state-metrics* 모두 한 번에
- *K8s 모니터링의 사실상 표준*

---

## 9. Federation & Long-term Storage

### Federation
- 여러 Prometheus 인스턴스의 metric을 *상위 Prometheus가 scrape*
- 멀티 클러스터 / 멀티 DC 환경

```yaml
# 상위 Prometheus
scrape_configs:
- job_name: 'federate'
  scrape_interval: 15s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job="kubernetes-pods"}'
  static_configs:
  - targets:
    - 'cluster1.example.com:9090'
    - 'cluster2.example.com:9090'
```

### Long-term storage
| 도구 | 특징 |
|---|---|
| **Thanos** | sidecar pattern, object storage (S3), global query |
| **Cortex / Mimir** | multi-tenant, horizontally scalable |
| **VictoriaMetrics** | high-performance, lower resource |
| **Grafana Cloud** | SaaS |

→ Prometheus는 *15일 정도 local*, *장기는 별도*. *Thanos가 가장 흔한 오픈소스 선택*.

---

## 10. 자주 헷갈리는 것

### 10-1. *rate vs increase*
- `rate(counter[5m])` = *초당 증가율*
- `increase(counter[5m])` = *5분간 증가량* (= rate × 300)
- 알람 조건: `rate(...) > 0.1` 같은 *초당 비율*이 자연스러움

### 10-2. *Range vector를 *graph에 직접* 쓰면 안 됨*
- `http_requests_total[5m]` 은 *range vector* — Grafana가 못 그림
- 반드시 `rate(...)` / `avg_over_time(...)` 등 *집계 함수에 감싸기*

### 10-3. *Counter reset*
- 앱 재시작 시 counter는 *0부터 다시*
- `rate`/`increase`가 *알아서 처리* — 음수 감지 시 reset으로 간주

### 10-4. *Histogram quantile 정확도*
- bucket 경계에 가까운 quantile은 *부정확*
- *latency 분포에 맞는 bucket 설계 필수* — 기본 bucket이 *0.005, 0.01, ..., 10* 인데 *내 service가 100ms~500ms*면 *그 영역에 bucket 더 빡빡하게*

### 10-5. *cardinality 폭증 흔한 사고*
- label에 `user_id`, `path` (with id), `version` 등 *고유 값* 박음
- 시계열 100만+ → Prometheus 메모리 폭증
- 발견: `topk(10, count by (__name__)({}))` (metric별 시계열 수)

### 10-6. *Prometheus는 HA가 어렵다*
- *각 인스턴스가 독립 scrape* — *분산 합의 X*
- 표준 패턴: *2개 인스턴스 모두 같은 target scrape*, 그 위에 *load balancer*
- 진짜 HA·중복 제거는 *Thanos / Cortex 등 추가*

---

## 🎤 면접 빈출 Q&A

### Q1. Prometheus가 pull model을 쓰는 이유?
> (1) **Target liveness 자동 체크** — scrape 실패 = down. (2) **방화벽 친화** — 단방향 (Prometheus → target). (3) **service discovery 친화** — target 목록 동적 갱신. (4) **재구성 쉬움** — config만 변경. 단점: *short-lived job* 어려움 → *Pushgateway* 예외 사용.

### Q2. Counter / Gauge / Histogram / Summary 차이?
> **Counter**: 증가만 하는 누적값 (request 수, error 수). **Gauge**: 증감하는 현재값 (memory 사용량, queue 크기). **Histogram**: 값 분포 — *bucket별 카운트*, *수평 집계 가능*. **Summary**: client에서 직접 quantile 계산, *합산 불가*. **분산 시스템은 Histogram 권장**.

### Q3. `rate` vs `irate` 차이? 언제 무엇?
> `rate(counter[5m])` = *5분 윈도우 평균 초당 증가율* — 부드러움, *대시보드·알람 표준*. `irate(counter[1m])` = *마지막 2 sample 기반* — 반응 빠름, 노이즈 많음 — *순간 spike 감지용*. **대시보드는 거의 항상 `rate`**.

### Q4. Cardinality 폭증을 어떻게 방지?
> *label은 유한한 값만* (status code, region, env, service name 등). *user_id, IP, trace_id, full URL 등 high-cardinality 값 금지*. 발견: `topk(10, count by (__name__)({}))` — metric별 시계열 수. *recording rule로 pre-aggregate*해서 *대시보드는 적은 cardinality 사용*. label 추가 전 *시계열 수 계산*.

### Q5. Recording rule은 무엇? 언제?
> *쿼리 결과를 *새 metric으로 저장*. 같은 복잡한 쿼리를 *대시보드 여러 곳에서 반복* 시 *매번 계산 vs 1회 계산 후 재사용* — 후자가 압도적 빠름. 명명 컨벤션: `level:metric:operation` (예: `job:http_requests:rate5m`). SLO 알람에선 *burn rate*도 recording rule로 미리 계산.

### Q6. K8s에서 Prometheus 어떻게 운영?
> **Prometheus Operator** + CRD. **ServiceMonitor** (어떤 Service scrape), **PodMonitor** (Pod 직접), **PrometheusRule** (recording + alerting rule), **AlertmanagerConfig** (라우팅). **kube-prometheus-stack** Helm chart로 *Prometheus + Operator + Grafana + Alertmanager + node_exporter + kube-state-metrics* 한 번에. 사실상 표준.

### Q7. Prometheus의 *장기 보관* 어떻게?
> 로컬 TSDB는 *15일 기본*. 장기는 **remote storage**: **Thanos** (sidecar + object storage), **Cortex / Mimir** (multi-tenant), **VictoriaMetrics** (고성능), **Grafana Cloud** (SaaS). Prometheus가 *remote_write* 로 외부에 push. Global query·downsampling·multi-cluster federation 지원.

---

## 🔗 Cross-reference

- **3 pillars 개념** → [01-three-pillars.md](01-three-pillars.md)
- **Alertmanager + alert 라우팅** → [03-alerting.md](03-alerting.md)
- **Grafana 시각화** → [04-grafana-and-dashboards.md](04-grafana-and-dashboards.md)
- **SLO·burn rate 알람** → [05-slo-and-error-budget.md](05-slo-and-error-budget.md)
- **Prometheus Operator (CRD 패턴)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **node_exporter + Linux 메트릭** → [`linux-basics/02`](../linux-basics/02-process-and-signals.md)

---

## 📝 3줄 요약

1. *Pull model + service discovery + 자체 TSDB + PromQL*. K8s에선 *Prometheus Operator + ServiceMonitor/PrometheusRule* 표준.
2. PromQL의 핵심: *Counter는 `rate`*, *Histogram은 `histogram_quantile`*, *집계는 `sum by/without`*, *복잡한 쿼리는 recording rule로 pre-aggregate*.
3. *Cardinality가 가장 흔한 사고* — label은 유한한 값만. *장기 보관은 Thanos/Mimir/VictoriaMetrics*. HA는 *2 인스턴스 + 외부 dedup*.

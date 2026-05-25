# 06 — Logs & Traces: 깊이 디버깅 도구

> **공식 근거:** [Loki docs](https://grafana.com/docs/loki/latest/), [OpenTelemetry docs](https://opentelemetry.io/docs/), [Jaeger docs](https://www.jaegertracing.io/docs/)

---

## 🎯 한 문장

> **Log = *사후 분석의 핵심* (자세한 context), Trace = *분산 시스템 디버깅의 핵심* (요청 전체 경로). 도구 선택은 *cost vs UX trade-off* — Loki(저렴·label 기반), Elasticsearch(강력·비쌈), SaaS(통합·매우 비쌈). OpenTelemetry가 *현대 표준*.**

---

## 1. Log Pipeline — 4단계

```
   Application
       │ stdout
       ▼
   Container/Node  ←── log shipper (Fluent Bit, Promtail, Vector)
       │ ship
       ▼
   Aggregator  ←── 중앙 수집 (Fluentd, Logstash, Loki)
       │ store
       ▼
   Storage     ←── 검색 가능한 store (Elasticsearch, Loki TSDB, Cortex)
       │ query
       ▼
   UI          ←── 검색·시각화 (Kibana, Grafana)
```

### 1단계: Application
- *stdout에 출력* (12-Factor)
- *JSON structured 권장* — `{"ts":"...", "level":"...", "msg":"...", "trace_id":"...", "user_id":...}`
- log level: DEBUG / INFO / WARN / ERROR / FATAL

### 2단계: Log Shipper (각 노드)
- *컨테이너 stdout 또는 file 읽음*
- *parse + enrich + forward*
- DaemonSet 형태 (K8s)

| Shipper | 특징 |
|---|---|
| **Fluent Bit** | C 기반, 가볍고 빠름. K8s 표준. |
| **Fluentd** | 더 풍부한 plugin. Ruby. 무거움. |
| **Vector** | Rust, 빠름, 통합 (metric도 가능). 신흥. |
| **Promtail** | Loki 전용. Grafana Labs. |

### 3단계: Aggregator (선택)
- 여러 shipper 결과 *중간 집계·라우팅*
- 필수 아님 — 작은 환경에선 shipper가 직접 store에

### 4단계: Storage + UI
- **Elasticsearch + Kibana** (EFK) — full-text search 강력, 비쌈
- **Loki + Grafana** — label 기반 (Prometheus 사상), 저렴
- **CloudWatch Logs** — AWS native
- **Datadog Logs / NewRelic / Coralogix** — SaaS

---

## 2. Loki — *Prometheus의 log 버전*

### 발상
- Elasticsearch는 *모든 log content를 index* — 비쌈
- **Loki**: *label만 index, content는 압축 저장* — 매우 저렴
- *Prometheus와 같은 label 사상*

### 구조
```
Log line: "2026-05-26 12:00:00 ERROR User auth failed (id=123)"

Labels (index됨):
  app=auth-service
  namespace=production
  pod=auth-7f5b9c-abc

Content (압축 저장, index X):
  "2026-05-26 12:00:00 ERROR User auth failed (id=123)"
```

→ *content 안에서 grep은 *full scan* (느림). label 필터링으로 *범위 좁힘 후* grep.

### LogQL — *PromQL 비슷*
```logql
# label 필터
{app="auth-service", namespace="production"}

# + 텍스트 필터 (grep)
{app="auth-service"} |= "ERROR"
{app="auth-service"} != "DEBUG"

# regex
{app="auth-service"} |~ "User .* failed"

# JSON parsing
{app="auth-service"} | json | level="error"

# pattern parsing
{app="nginx"} | pattern `<ip> - - [<timestamp>] "<method> <path> HTTP/<protocol>" <status> <size>`

# metric 변환 (log → metric)
sum by (app) (rate({namespace="production"} |= "ERROR" [5m]))
```

### 비용 비교 (대략)
- Elasticsearch: TB당 *수백 $ / 월*
- Loki + S3: TB당 *수십 $ / 월*
- → *대규모 log은 Loki가 *10x 저렴**

### 단점
- *full-text search 약함* — label로 좁힐 수 없는 검색은 매우 느림
- *Elasticsearch의 풍부한 query DSL 없음*

→ *label 설계가 잘 된 환경*에서 Loki, 그렇지 않으면 Elasticsearch.

---

## 3. Elasticsearch (EFK/ELK)

### 강점
- *Full-text search 강력* (Apache Lucene 기반)
- *분석·집계 풍부* (aggregation, term, histogram)
- *Kibana UI 강력* — discovery, visualization, lens

### 약점
- *비쌈* — 모든 content index, RAM 많이 필요
- *운영 복잡* — cluster 관리, shard, replica
- *log volume에 비례 비용 폭증*

### 구조
```
Log → Fluent Bit/Logstash → Elasticsearch cluster (data nodes) → Kibana
                                                                  │
                                                                  ├── Discovery (검색)
                                                                  ├── Dashboard
                                                                  └── Alert
```

### Index Lifecycle Management (ILM)
- *최신 index*: hot (빠른 SSD, 빈번 접근)
- *옛 index*: warm → cold → frozen (HDD/object storage)
- *자동 rotation* — 비용 절감

---

## 4. Cloud-native Log

### CloudWatch Logs (AWS)
- *AWS 자원에서 *log group + log stream*
- *Insights query* (SQL-like)
- *retention period* per log group
- *Lambda·EC2·ECS·EKS 등과 통합*

### Stackdriver / Cloud Logging (GCP)
### Azure Monitor Logs (KQL)

### 장점·단점
- 장점: *agent 거의 없음, 자동 통합*
- 단점: *비싸짐 (특히 query)*, *vendor lock-in*

---

## 5. Tracing — Distributed Tracing 깊이

### 왜 필요한지 (다시)
- 한 user request → *5~20 마이크로서비스 호출*
- 각 서비스 *독립 log* — 어디서 느려졌는지 추적 어려움
- → **trace로 *요청 전체의 시각화***

### Trace 구조 (다시)
```
Trace (1개 user request)
└── Span (각 서비스의 작업)
    ├── Span ID
    ├── Parent Span ID
    ├── Trace ID (모든 span 동일)
    ├── Operation Name
    ├── Start time / Duration
    └── Tags (key-value: http.method, http.status_code, db.statement, ...)
    └── Events (timeline 내 timestamped 메시지)
```

### Trace Context 전파 — *W3C Trace Context*
- HTTP header `traceparent`: `00-{trace-id}-{parent-span-id}-{flags}`
- 받는 서비스가 *그 trace-id로 새 span 시작* (parent-span-id는 자기 부모)
- 모든 span이 *같은 trace로 묶임*

```
Client → API Gateway:        traceparent: 00-abc...-span1-01
       API Gateway → Auth:    traceparent: 00-abc...-span2-01
       API Gateway → Order:   traceparent: 00-abc...-span3-01
                Order → DB:    traceparent: 00-abc...-span4-01
```

---

## 6. OpenTelemetry — *현재 표준*

### 역사
- *OpenTracing* (2016) + *OpenCensus* (2018) → **OpenTelemetry** (2019) 통합
- *CNCF 표준* — 모든 벤더 지원
- *trace + metric + log 통합 framework*

### Architecture
```
┌──────────────┐
│ Application   │
│ (OTel SDK     │── 자동 또는 수동 instrument
│  installed)   │
└──────┬───────┘
       │ OTLP (gRPC/HTTP)
       ▼
┌──────────────┐
│  OTel        │── exporter로 *여러 backend*에 보냄
│  Collector   │   - Jaeger
│              │   - Tempo
│              │   - Datadog
│              │   - Cloud trace
└──────────────┘
```

### Auto-instrumentation
- *코드 거의 안 바꾸고* 자동 trace 생성
- 언어별 agent / SDK: Java agent, Python `opentelemetry-instrument`, Node `@opentelemetry/auto-instrumentations-node`
- HTTP 클라이언트·서버, DB driver, message queue 등 *모두 자동 span*

```python
# Python — auto-instrument
# opentelemetry-instrument python app.py
```

### Manual instrumentation
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.add_event("Validating order")
        # ... business logic ...
        span.set_attribute("order.total", 1234.56)
```

### OTel Collector
- *중앙 처리·routing 단위*
- *receivers* (어디서 받나) — OTLP, Jaeger, Zipkin, Prometheus
- *processors* (어떻게 변환) — batch, sampling, attributes
- *exporters* (어디로 보내나) — Jaeger, Tempo, Cloud, Prometheus, Loki

→ *Application은 OTLP 하나만 지원*, *Collector가 vendor 변환* — vendor lock-in 회피.

---

## 7. Trace Backend

| | 특징 |
|---|---|
| **Jaeger** (CNCF) | 오픈소스, 단독 또는 K8s. Elasticsearch/Cassandra 백엔드. UI 풍부. |
| **Tempo** (Grafana Labs) | object storage (S3) 백엔드 — *저렴*. Loki/Prometheus와 통합 잘 됨. |
| **Zipkin** | 옛 표준, 단순. |
| **AWS X-Ray** | AWS native. Lambda·ECS·EKS 통합. |
| **Datadog APM** | SaaS. metric·log·trace 통합. |
| **NewRelic APM** | SaaS. 같음. |
| **Honeycomb** | observability 전문 SaaS. high-cardinality 친화. |

---

## 8. Sampling — *모든 trace 저장 불가*

### 발생량
- 1M req/s 서비스 → 1M trace/s
- *전부 저장 = 비용 폭증, query 느림*

### Head-based sampling
- *요청 시작 시 *주사위* — keep / drop 결정*
- 단순. 일관 (한 trace 안 모든 span 같이 결정)
- 단점: *errored / slow trace 놓침* (낮은 비율로)

```yaml
# OTel SDK
sampler: parentbased_traceidratio
sampling_ratio: 0.1     # 10%
```

### Tail-based sampling
- *trace 완료 후 결정* — *errored / slow 트래픽은 keep*
- 더 똑똑. *디버깅 가치 높은 trace 보존*
- 단점: *전체 trace 메모리에 buffer* — 메모리 ↑

OTel Collector의 *tail_sampling processor*:
```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
    - name: errors
      type: status_code
      status_code: {status_codes: [ERROR]}
    - name: slow
      type: latency
      latency: {threshold_ms: 1000}
    - name: random
      type: probabilistic
      probabilistic: {sampling_percentage: 1}
```

→ error trace는 100%, slow trace 100%, 정상 1%만 keep. *디버깅 가치 vs 비용 균형*.

### Adaptive sampling
- 부하 따라 *동적 비율 조정*
- Honeycomb refinery 등 도구

---

## 9. Log·Metric·Trace 통합

### Correlation IDs
- 모든 log에 *trace_id 포함*
```json
{"ts":"...", "msg":"User auth failed", "trace_id":"abc123...", "span_id":"def..."}
```
- Grafana에서 trace 보다 *그 trace의 log* 자동 filter

### Exemplars
- Histogram bucket에 *대표 trace_id 첨부*
- 대시보드에서 metric 클릭 → *해당 trace 점프*

### Grafana 통합
- Grafana + Loki + Tempo = *log·trace 직접 점프 가능*
- Loki log line의 *trace_id 클릭 → Tempo에서 그 trace*
- Tempo span의 *log link → 해당 시간대 log*

→ **3 pillar 통합 운영** — 한 화면에서 디버깅 완결.

---

## 10. 자주 헷갈리는 것

### 10-1. *log에 *PII 박지 말기**
- email, phone, 카드번호, 주소 등
- *컴플라이언스 위반 + 유출 위험*
- 앱에서 *masking* 또는 *별도 audit log*

### 10-2. *trace_id 가 *log에 없음**
- log·trace 연결 안 됨 → 통합 디버깅 불가
- *로깅 library에서 자동 inject* (OTel logger integration)

### 10-3. *Synchronous logging이 *성능 영향**
- log write가 *blocking → 응답 지연*
- *async logging* (Logback async, zap async) 또는 *batch flush*

### 10-4. *sampling rate 너무 낮음 → 사고 trace 없음*
- 1% sampling이면 *드문 사고는 catch 못 함*
- *tail-based*로 *error는 100%* 보장

### 10-5. *Elasticsearch index 폭증*
- *log 형식 변경 시* 새 mapping → shard 폭증
- *ILM + index template + 적절한 mapping*

### 10-6. *Loki에 *high-cardinality label***
- Prometheus와 같은 함성 — *user_id, path with id 등 박으면 폭증*
- *label은 유한한 값만*

---

## 🎤 면접 빈출 Q&A

### Q1. EFK / ELK 와 Loki 차이? 언제 무엇?
> **EFK/ELK** (Elasticsearch + Fluent Bit/Logstash + Kibana): *full-text search 강력, 분석 풍부*, *비쌈* (모든 content index). **Loki + Grafana**: *label만 index, content는 압축 저장* — *10x 저렴*. 단 *full-text search 약함*. → *label 설계 잘 된 환경*은 Loki, *복잡한 검색 필요*는 Elasticsearch.

### Q2. OpenTelemetry란? 왜 표준?
> OpenTracing + OpenCensus 통합 → CNCF 표준 (2019). *trace + metric + log 통합 framework*. *언어별 SDK + Collector + OTLP protocol*. **Auto-instrumentation**: 코드 거의 안 바꾸고 trace 자동 생성. **Collector가 vendor 변환** — 앱은 OTLP 하나만, vendor 교체 자유. *현재 trace 표준은 OTel*.

### Q3. Distributed tracing의 *context propagation*은?
> **W3C Trace Context** 표준. HTTP header `traceparent: 00-{trace-id}-{parent-span-id}-{flags}`. 받는 서비스가 *그 trace-id로 새 span 시작* (parent-span-id는 자기 부모). 모든 span이 *같은 trace ID로 묶임*. SDK가 *HTTP 클라이언트·서버에서 자동 처리* — 개발자가 manual 안 해도 됨.

### Q4. Head-based vs Tail-based sampling?
> **Head**: 요청 시작 시 *주사위* — 단순·일관, *낮은 비율로 errored 놓침*. **Tail**: trace 완료 후 결정 — *errored / slow 100% keep*. *디버깅 가치 vs 비용* 우월. 단 *전체 trace 메모리 buffer 필요*. OTel Collector의 *tail_sampling processor*가 표준.

### Q5. Log·Metric·Trace 통합 운영?
> **Correlation IDs** — 모든 log에 *trace_id 포함*. **Exemplars** — Histogram bucket에 *대표 trace_id 첨부*. *Grafana + Loki + Tempo* — 대시보드 metric → 대표 trace → 그 trace의 log *한 화면에서 점프*. **3 pillar 통합 디버깅 흐름** — Metric으로 발견, Trace로 좁히기, Log로 깊이.

### Q6. Logging best practice?
> (1) **stdout** (12-Factor). 파일 X. (2) **Structured JSON** — `{"ts":"...", "level":"...", "trace_id":"..."}`. (3) **PII masking** — email·phone 등 안 박음. (4) **Async logging** — blocking 회피. (5) **Sensible level** — DEBUG는 dev만, prod는 INFO+. (6) **trace_id 자동 inject** — OTel logger integration.

### Q7. Sampling rate를 *어떻게 결정*?
> *Cost·디버깅 가치 trade-off*. 시작은 *head-based 1~10%*, **errored / slow 100%** (tail-based). 비용 ok면 head 비율 ↑, 비용 압박이면 ↓ 단 tail 100% 유지. *high-traffic critical path*는 더 낮게, *low-traffic but critical*은 더 높게.

---

## 🔗 Cross-reference

- **3 pillars + Exemplars** → [01-three-pillars.md](01-three-pillars.md)
- **Prometheus와의 통합 (Loki, Tempo)** → [02-prometheus.md](02-prometheus.md)
- **Grafana 통합 시각화** → [04-grafana-and-dashboards.md](04-grafana-and-dashboards.md)
- **OpenTelemetry Operator (K8s)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **Sidecar 패턴 (logging agent)** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md), [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)

---

## 📝 3줄 요약

1. *Log pipeline 4단계*: app stdout → shipper (Fluent Bit/Promtail) → aggregator → storage (Elasticsearch/Loki/SaaS) → UI (Kibana/Grafana). *Loki가 *10x 저렴*, full-text 약함*.
2. **OpenTelemetry**가 trace 표준. *Auto-instrumentation으로 코드 안 바꿈*. *Collector가 vendor 변환* — lock-in 회피. *W3C Trace Context로 HTTP header 전파*.
3. *Tail-based sampling*으로 errored/slow 100% 보존. **Correlation IDs + Exemplars**로 *3 pillar 통합 디버깅* (metric → trace → log 한 화면).

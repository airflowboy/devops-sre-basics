# 01 — Three Pillars: Metrics, Logs, Traces

> **공식 근거:** [Google SRE Book](https://sre.google/books/), [CNCF Observability Whitepaper](https://github.com/cncf/tag-observability), [OpenTelemetry docs](https://opentelemetry.io/docs/)

---

## 🎯 한 문장

> **Metrics = *시계열 수치* (대시보드·알람), Logs = *시점별 텍스트 이벤트* (자세한 사후 분석), Traces = *요청의 전체 경로* (분산 시스템 디버깅). 셋이 *상호 보완 — 어느 하나만으론 부족*.**

---

## 1. Metrics — *시계열 수치*

### 특징
- *작고 빠름* — 보통 sample 8 bytes
- *집계 친화* (sum, avg, percentile)
- *장기 보관 가능*
- *대시보드·알람의 베이스*

### 단점
- *카디널리티 한계* — label 조합이 *수십~수백만* 넘으면 시스템 망함
- *상세 정보 X* — 어떤 요청이 느렸는지는 *알 수 없음* (그건 log/trace 영역)

### 자주 보는 metric 종류
| 종류 | 의미 | 예 |
|---|---|---|
| **Counter** | *증가만* 하는 누적값 | `http_requests_total{status="200"}` |
| **Gauge** | *증감*하는 현재값 | `cpu_usage`, `memory_usage_bytes` |
| **Histogram** | *값 분포* (bucket별 카운트) | `http_request_duration_seconds_bucket{le="0.5"}` |
| **Summary** | 분포 (client에서 quantile 계산) | `http_request_duration_seconds{quantile="0.99"}` |

→ Histogram이 *수평 집계 가능* (여러 인스턴스 합쳐 quantile 계산). Summary는 X.

---

## 2. RED Method — *서비스 측면*

[Tom Wilkie 제안](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/). 모든 서비스가 *최소 이 3가지*는 측정:

| | 의미 | PromQL 예 |
|---|---|---|
| **R**ate | 초당 요청 수 | `sum(rate(http_requests_total[5m])) by (service)` |
| **E**rrors | 초당 실패 요청 수 | `sum(rate(http_requests_total{status=~"5.."}[5m]))` |
| **D**uration | 응답 시간 분포 | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` |

→ 이 3개 + 옆 *availability* 만 *대시보드 1장*으로 *서비스 건강 90% 진단*.

---

## 3. USE Method — *자원 측면*

[Brendan Gregg 제안](https://www.brendangregg.com/usemethod.html). *각 자원에 대해 3가지*:

| | 의미 | 예 |
|---|---|---|
| **U**tilization | 사용률 (%) | CPU usage, memory used |
| **S**aturation | 포화도 (queue 길이, wait time) | run queue length, disk await |
| **E**rrors | 에러 카운트 | NIC errors, disk read errors |

→ 자원: CPU·Memory·Disk·Network·File descriptor 등.

→ RED + USE 를 *서비스 + 인프라 각자* 적용 = *기본 대시보드 완성*.

---

## 4. Logs — *시점별 텍스트 이벤트*

### 특징
- *자세함* — request body, stack trace, debug info
- *검색 친화* — grep, full-text search
- *큼·느림* — TB 단위 누적
- *상관관계 추적 어려움* (한 요청이 여러 서비스 거치면 log 흩어짐)

### 형식
- **Unstructured** — 자유 텍스트. `2026-05-26 12:00:00 INFO User logged in (id=123)`
- **Structured** — JSON. `{"ts":"...", "level":"info", "msg":"User logged in", "user_id":123, "trace_id":"abc..."}`

→ **현대는 structured 우선** — 검색·집계·trace 연동 쉬움.

### 12-Factor: *logs as event streams*
- 컨테이너는 *log를 파일에 쓰지 말고 stdout으로*
- 외부 시스템 (kubelet → log shipper)이 *수집·중앙 집계*
- 이유: 컨테이너 *fs 비영속*, 멀티 인스턴스에서 *log 흩어짐*

→ [`docker-basics/04`](../docker-basics/04-volumes-and-storage.md) 의 영속성 원칙과 일관.

### 흔한 log stack
| | 구성 |
|---|---|
| **EFK** | Elasticsearch + Fluent Bit/Fluentd + Kibana |
| **ELK** | + Logstash |
| **Loki** | Grafana Loki + Promtail (label 기반, 가벼움, 검색 약함) |
| **Coralogix / Datadog / NewRelic** | SaaS — 운영 부담 0, 비용 ↑ |
| **CloudWatch / Stackdriver** | 클라우드 native |

---

## 5. Traces — *요청의 전체 경로*

### 왜 필요?
- 마이크로서비스 환경: 한 요청이 *5~20개 서비스 거침*
- log 흩어짐 — *어디가 느렸나, 어디서 fail했나 추적 어려움*
- → **Trace로 *요청 단위 전체 path 시각화***

### 구조
```
Trace (전체 요청)
├── Span 1: API Gateway (10ms)
│   ├── Span 2: Auth Service (3ms)
│   ├── Span 3: Order Service (45ms)         ← 느림! 여기 zoom in
│   │   ├── Span 4: DB query (35ms)          ← DB 병목
│   │   └── Span 5: Cache lookup (1ms)
│   └── Span 6: Notification Service (5ms)
```

각 span = *한 서비스의 작업 단위*. 시작·종료 시각 + metadata (tag).

### Trace context 전파
- 요청 들어옴 → *Trace ID 생성*
- 다음 서비스 호출 시 *HTTP header에 Trace ID 전달* (`traceparent`)
- 받는 서비스가 *그 Trace ID로 새 span 시작*
- 모든 span이 *같은 Trace ID* 로 묶임 → 시각화

### 표준
- **OpenTelemetry** (CNCF) — *현재 표준*. SDK + collector + protocol.
- 옛 표준: OpenTracing + OpenCensus → 통합되어 OpenTelemetry.
- *언어별 auto-instrumentation* 라이브러리 — 코드 거의 안 바꾸고 trace 자동 생성.

### Trace 백엔드
| | 특징 |
|---|---|
| **Jaeger** (CNCF) | 오픈소스, Elasticsearch/Cassandra 백엔드 |
| **Tempo** (Grafana Labs) | object storage 백엔드 (저렴) |
| **Zipkin** | 옛 표준, 단순 |
| **Datadog APM** | SaaS, log·metric과 통합 |
| **NewRelic APM** | SaaS |
| **AWS X-Ray** | AWS native |

### Sampling — *모든 trace 저장 못 함*
- 대규모 시스템: 매 요청에 trace = *데이터 폭증*
- *sampling*: 일부만 보관
- 전략:
  - **Head-based**: 요청 시작 시 *주사위* (예: 10%)
  - **Tail-based**: 요청 완료 후 *errored / slow만 보관*
  - **Adaptive**: 부하 따라 비율 조정

---

## 6. 셋의 *상호 보완* — 통합 운영

### 시나리오: *p99 latency가 튀었다*

1. **Metric (alert)**: SLO 위반 알람 → 대시보드 확인 → 특정 service의 p99가 *5분간 spike*
2. **Trace (분석)**: 그 시간대의 slow trace 검색 → *Order Service의 DB query가 50ms → 500ms*로 튐
3. **Log (깊이)**: Order Service의 그 시간 log → *"slow query: SELECT * FROM orders WHERE ..."* + stack trace

→ **Metric으로 발견, Trace로 좁히기, Log로 깊이 분석.**

### Exemplars — *metric에서 trace로 점프*
- Histogram bucket에 *예시 trace ID 첨부*
- Grafana에서 metric 클릭 → *대표 trace 자동 열기*
- *3 pillar 통합의 핵심*

### Correlation IDs
- 모든 log에 *trace ID·request ID* 박기
- 한 요청의 모든 log를 *그 ID로 grep*
- 분산 추적의 *log 버전*

---

## 7. *어떤 데이터를 어떤 pillar에?* — 결정 가이드

| 질문 | 적합 |
|---|---|
| "초당 요청 수" | Metric |
| "지난 1시간 5xx 비율 추이" | Metric |
| "어떤 user가 어떤 endpoint 호출" | Log |
| "특정 요청의 SQL stack trace" | Log |
| "이 느린 요청이 어느 서비스의 어느 호출에서 느려졌나" | Trace |
| "user_id별 요청 수" | **Log** (metric에 user_id 박으면 cardinality 폭증) |
| "지난 1년 SLO 추적" | Metric (장기 보관·집계) |
| "OOM 직전 어떤 메모리 할당이 있었나" | Log + Metric |

### *Metric에 절대 안 박을 것*
- user_id, email, IP address, trace_id, request_id, URL의 query string
- *무한 cardinality* — Prometheus 메모리 폭증
- → 이건 *log에*

---

## 8. *Profiling* — 4번째 pillar?

최근 추가 논의. **Continuous profiling** = *CPU·메모리·flame graph 시계열 수집*.

- *어느 코드 라인이 CPU 잡고 있나*
- 옛 ad-hoc `perf` / `pprof` → *상시 수집·시계열 분석* (Pyroscope, Parca, Datadog Continuous Profiler)
- Metric으론 "CPU 80%" — *어디서?* 는 모름. Profiling이 그 답.

→ Observability의 *확장 영역*. 아직 표준화 진행 중.

---

## 9. *Cost 관리*

| Pillar | Cost driver |
|---|---|
| Metrics | *카디널리티* (label 조합 수) |
| Logs | *volume* (line 수 × 평균 크기) |
| Traces | *sampling rate* × span 수 |

### 절감 전략
- **Metric**: 카디널리티 제한 — *label 신중하게*, *recording rule로 pre-aggregate*
- **Log**: *level filter* (DEBUG 안 보관), *sampling*, *cold storage tier*
- **Trace**: *tail-based sampling* (errored/slow만), *낮은 head sampling rate*

→ 운영에서 *관측성 비용이 인프라 비용의 30% 차지*도 흔함. 의식적 관리 필수.

---

## 10. 자주 헷갈리는 것

### 10-1. *log를 metric 대체로 쓰면 비용 폭증*
- "Error log 1개 → +1 카운트" — log aggregator로 metric 만들면 *log 비용*
- *앱 자체에서 metric 직접 expose* (Prometheus client library)

### 10-2. *trace 모든 요청 trace = 비현실*
- *cardinality·volume 폭증*
- *sampling 필수* (head-based 또는 tail-based)

### 10-3. *Grafana ≠ Prometheus*
- Grafana = *시각화·대시보드* (여러 datasource 지원: Prometheus, Loki, Tempo, MySQL, ...)
- Prometheus = *metric DB + scraper + alerting*
- 둘이 *같이 쓰임*: Prometheus가 데이터, Grafana가 보여줌

### 10-4. *NewRelic·Datadog = 전부*
- SaaS는 *metric + log + trace + alerting 통합*
- 단점: *vendor lock-in, 비용 ↑* (특히 카디널리티 비용)
- 자체 운영 (Prometheus + Loki + Tempo + Grafana) vs SaaS — *trade-off*

### 10-5. *3 pillar는 *기술 분류*가 아니라 *데이터 분류**
- 한 시스템이 *셋 다 backend*로 보내도 OK (예: OpenTelemetry collector)
- 또는 *통합 backend* (Datadog 등)

---

## 🎤 면접 빈출 Q&A

### Q1. Monitoring과 Observability 차이?
> *Monitoring* = *알려진 unknowns 추적* (CPU 80% 알람 등 — 질문이 정해짐). *Observability* = *알려지지 않은 unknowns 탐색* (예상 못 한 사고 디버깅). Observability가 *monitoring을 포함*. 단순 알람 시스템 아님. *Metrics + Logs + Traces 3 pillars로 구성*.

### Q2. 3 pillars (Metrics·Logs·Traces) 각각 언제?
> **Metrics**: 작고 빠름, 시계열 집계, 대시보드·알람·SLO. 단 *cardinality 한계*. **Logs**: 자세함, 사후 분석·디버깅·audit. 큼·느림. **Traces**: 분산 시스템의 *요청 전체 경로 추적*. 마이크로서비스 *어디가 느렸나*. → **셋이 상호 보완**, 한 사고 디버깅에 *셋 다 사용*.

### Q3. RED vs USE method?
> **RED** (서비스 측면): **R**ate (초당 요청), **E**rrors (실패), **D**uration (응답 시간). 서비스 모니터링 표준. **USE** (자원 측면): **U**tilization, **S**aturation, **E**rrors. 각 자원 (CPU·Memory·Disk·Network)에 적용. → 서비스는 RED, 인프라는 USE.

### Q4. Histogram과 Summary 차이?
> Histogram: *bucket별 카운트* (`le="0.5"`, `le="1.0"` 등) — *server 측 집계, 수평 집계 가능* (여러 인스턴스 합쳐 quantile). Summary: *client에서 직접 quantile 계산* (`quantile="0.99"`) — 정확하나 *합산 불가*. *대규모 분산 시스템은 Histogram* 거의 표준.

### Q5. Cardinality 폭증이 *왜 문제*?
> Prometheus는 *각 label 조합을 별도 시계열로 저장*. label에 *user_id*, *trace_id*, *IP* 같은 *high-cardinality 값* 박으면 → 시계열 *수십~수백만*. 메모리·디스크·쿼리 시간 폭증. *label은 유한한 값만* (status code, region, env 등). user_id·request_id는 *log에*.

### Q6. Distributed tracing이 *왜 필요*한가?
> 마이크로서비스 환경: 한 요청이 *5~20개 서비스 거침*. log·metric만으로는 *어느 호출이 느렸나·실패했나* 추적 어려움. Trace로 *요청의 전체 경로 + 각 span의 시간 + 호출 관계* 시각화. *Trace ID를 HTTP header로 전파* — 받는 서비스가 같은 trace에 span 추가. **OpenTelemetry**가 현재 표준.

### Q7. Exemplars란? 왜 유용?
> Histogram bucket에 *예시 trace ID 첨부*. Grafana에서 metric 보다 *클릭 → 대표 trace 자동 열기*. **3 pillar 통합의 핵심** — metric으로 *문제 발견*, exemplar로 *대표 trace 점프*, trace에서 *log·span 깊이 분석*. 디버깅 흐름이 *한 화면에서 완결*.

---

## 🔗 Cross-reference

- **Prometheus 깊이** → [02-prometheus.md](02-prometheus.md)
- **SLO·error budget** → [05-slo-and-error-budget.md](05-slo-and-error-budget.md)
- **Logs·Traces 도구** → [06-logs-and-traces.md](06-logs-and-traces.md)
- **K8s ServiceMonitor·PrometheusRule** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md) (Operator 패턴)
- **Linux 자원 모니터링 (LA, vmstat)** → [`linux-basics/02`](../linux-basics/02-process-and-signals.md)

---

## 📝 3줄 요약

1. *Observability = monitoring + 예상 못 한 질문에도 답할 능력*. *3 pillars (Metrics·Logs·Traces)* 가 데이터 분류.
2. *RED (서비스) + USE (자원)* method로 *어떤 지표를 측정할지* 결정. *Metric은 cardinality, Log는 volume, Trace는 sampling*이 비용 driver.
3. *셋이 상호 보완* — Metric (발견) → Trace (좁히기) → Log (깊이 분석). *Exemplars로 3 pillar 통합 디버깅*.

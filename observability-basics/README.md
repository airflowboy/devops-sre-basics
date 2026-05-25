# Observability — 동작 원리 중심

> *"Monitoring과 Observability 차이?"*
> *"SLO·error budget으로 운영을 어떻게 결정?"*
> *"Prometheus의 pull model 이유는?"*
> *"distributed tracing이 왜 필요한가?"*

---

## 🎯 한 문장 큰 그림

> **Observability = *시스템 내부 상태를 *외부 출력만으로 추론할 수 있는 정도**. *Monitoring*이 *예상한 질문에 답*이라면, *Observability*는 *예상 못 한 질문에도 답*. *Metrics + Logs + Traces 3 pillars* 위에 *SLO·error budget으로 운영 결정*.**

---

## 🧩 한 페이지 요약

```
                     ┌──────────────────────────────┐
                     │       애플리케이션             │
                     │  (RED metric / log / trace 발생)│
                     └──────┬─────────┬──────┬───────┘
                            │ pull    │ push │ push
              ┌─────────────▼──┐ ┌───▼───┐ ┌▼────────────┐
              │  Prometheus     │ │ Loki/ │ │  Jaeger /   │
              │  (metrics)      │ │ ELK  │ │  Tempo      │
              │                 │ │ (logs)│ │  (traces)   │
              └─────┬───────────┘ └───┬───┘ └─┬───────────┘
                    │                 │       │
                    ▼                 ▼       ▼
              ┌────────────────────────────────────┐
              │  Grafana / Datadog / NewRelic       │
              │  (대시보드 + 알람 + 탐색)             │
              └─────┬──────────────────────────────┘
                    │ alert
                    ▼
              ┌────────────────┐
              │  Alertmanager   │
              │  + PagerDuty    │
              └────────────────┘
                    │
                    ▼
                 oncall
```

---

## 🌪 자주 헷갈리는 7가지

### 1. *Monitoring ≠ Observability*
- **Monitoring** = *알려진 unknowns 추적* — "CPU 80% 넘으면 알람" (질문이 정해짐)
- **Observability** = *알려지지 않은 unknowns 탐색* — "오늘 새벽 *왜* 특정 endpoint p99 latency가 튀었지?" (질문이 *그때 떠오름*)
- → *Observability가 monitoring을 포함*. 단순 알람 시스템이 아님.

### 2. *3 Pillars: Metrics / Logs / Traces*
- **Metrics**: *시계열 수치*. 작고 빠름. 대시보드·알람에 적합.
- **Logs**: *시점별 텍스트 이벤트*. 자세하나 큼·느림.
- **Traces**: *요청의 전체 경로* (마이크로서비스 간 호출). 분산 시스템에 필수.
- 셋이 *상호 보완*. 어느 하나만으론 부족.

### 3. *Pull vs Push*
- **Prometheus = Pull** — Prometheus가 *target의 `/metrics` endpoint를 주기적으로 scrape*
- **Datadog/NewRelic = Push** — agent가 *서버에 push*
- Pull 장점: *target이 살아 있나도 자동 체크* (scrape 실패 = down), *방화벽 안 target 다루기 쉬움 (단방향)*
- Push 장점: *대규모에 scale*, *short-lived job 친화*, *방화벽 통과* (외부 SaaS)
- → Prometheus도 *push gateway* 사용 가능 (예외)

### 4. *RED vs USE method*
- **RED** (서비스/요청 측면): **R**ate, **E**rrors, **D**uration
- **USE** (자원 측면): **U**tilization, **S**aturation, **E**rrors
- → 서비스 모니터링은 RED, 인프라(노드/디스크) 모니터링은 USE.

### 5. *SLO는 *기술 지표*가 아니라 *비즈니스 약속**
- "p99 < 200ms" 는 기술 지표
- "*99.9%의 요청이 p99 < 200ms 안에 응답*" 가 SLO
- 측정 윈도우 + 목표 % + 영향 받는 사용자 정의가 *필수*
- *error budget = 100% - SLO* — 이 budget이 *기능 출시·운영 결정의 기준*

### 6. *Carddinality 폭증*이 *모든 시스템의 첫 사고*
- Prometheus label에 *user_id*, *email* 같은 *고유 값* 박으면 → 메트릭 *수십만 개*
- 메모리·스토리지·쿼리 시간 폭증
- → *label은 *유한한 값* 만 (status code, region, env 등). user_id는 *log*에.

### 7. *알람 노이즈는 *시스템 사용 안 되게* 만든다*
- *알람 너무 많음* → *팀이 알람 무시* → *진짜 사고 놓침*
- 해결: *SLO 기반 알람* (사용자 영향 있을 때만), *multi-burn-rate*, *inhibition·silence*.

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-three-pillars.md](01-three-pillars.md) | metrics/logs/traces 차이·언제 무엇 |
| [02-prometheus.md](02-prometheus.md) | pull model, exporter, PromQL, recording rules, federation |
| [03-alerting.md](03-alerting.md) | Alertmanager 라우팅·grouping·inhibition·silencing, PagerDuty |
| [04-grafana-and-dashboards.md](04-grafana-and-dashboards.md) | 데이터소스, variable, alerting, 대시보드 디자인 원칙 |
| [05-slo-and-error-budget.md](05-slo-and-error-budget.md) | SLI/SLO/SLA, error budget policy, multi-burn-rate alert |
| [06-logs-and-traces.md](06-logs-and-traces.md) | Loki/EFK/Coralogix, OpenTelemetry, Jaeger/Tempo, sampling |

---

## 🔗 Cross-reference

- **Linux의 LA·OOM·디스크 모니터링** → [`linux-basics/02`](../linux-basics/02-process-and-signals.md), [`linux-basics/05`](../linux-basics/05-package-and-troubleshooting.md)
- **K8s의 Pod·node 메트릭** → [`kubernetes-basics/`](../kubernetes-basics/)
- **Argo Rollouts의 metric-based promotion** → [`cicd-gitops-basics/04`](../cicd-gitops-basics/04-deployment-strategies.md)
- **Prometheus Operator (ServiceMonitor·PrometheusRule)** → [02-prometheus.md](02-prometheus.md) + [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| Monitoring vs Observability 차이? | README + [01](01-three-pillars.md) |
| RED와 USE method? | [01](01-three-pillars.md) |
| Prometheus pull model — 왜 pull? | [02](02-prometheus.md) |
| PromQL 기본 함수? rate vs irate? | [02](02-prometheus.md) |
| Alert noise를 어떻게 줄이나? | [03](03-alerting.md) + [05](05-slo-and-error-budget.md) |
| SLO를 어떻게 정의·운영? | [05](05-slo-and-error-budget.md) |
| Distributed tracing이 왜 필요? | [06](06-logs-and-traces.md) |
| Log·Metric·Trace를 *통합* 분석은? | [06](06-logs-and-traces.md) |

---

## 📐 학습 순서 권장

1. **01 three-pillars** — *큰 그림*
2. **02 prometheus** — *metric 깊이*
3. **05 slo-and-error-budget** — *운영 결정의 베이스*
4. **03 alerting** — *알람 어떻게 설계*
5. **04 grafana** — *시각화*
6. **06 logs-and-traces** — *깊이 디버깅*

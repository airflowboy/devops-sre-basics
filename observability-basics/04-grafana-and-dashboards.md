# 04 — Grafana & Dashboards

> **공식 근거:** [Grafana docs](https://grafana.com/docs/grafana/latest/), [Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)

---

## 🎯 한 문장

> **Grafana = *여러 데이터소스를 통합 시각화*하는 도구. Prometheus·Loki·Tempo·MySQL·CloudWatch 등 *수십 datasource 지원*. *대시보드 디자인 = 청중 + 질문 + 조직화*가 핵심. 알람도 가능 (Grafana Alerting).**

---

## 1. Datasource — *어디서 데이터 가져오나*

### 자주 쓰는 datasource
| Datasource | 용도 |
|---|---|
| **Prometheus** | metric (기본) |
| **Loki** | log (Grafana 진영) |
| **Tempo** | trace (Grafana 진영) |
| **Elasticsearch** | log (EFK) |
| **InfluxDB** | metric (다른 TSDB) |
| **MySQL / PostgreSQL** | DB query 직접 |
| **CloudWatch** | AWS native |
| **Stackdriver** | GCP native |
| **Azure Monitor** | Azure |
| **Datadog** | Datadog metric도 Grafana에서 |

### 추가
- UI에서 *Configuration > Data Sources* → 새 datasource
- *credentials + endpoint* 입력
- *test* 로 연결 확인

### Provisioning — *코드로 datasource 관리*
- UI 클릭 대신 *YAML 파일로 datasource 정의* → Git 관리·재현성
```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
- name: Prometheus
  type: prometheus
  url: http://prometheus:9090
  access: proxy
  isDefault: true
```

---

## 2. Dashboard 구조

```
Dashboard
├── Folder (조직화)
├── Tags
├── Variables (filter/dropdown)
├── Time range (전체 적용)
├── Panel (개별 시각화)
│   ├── Query (PromQL / LogQL / SQL etc.)
│   ├── Visualization (graph / stat / gauge / table / ...)
│   └── Field options (units, thresholds, color)
└── Annotation (이벤트 marker)
```

### 자주 쓰는 panel 종류
| Panel | 용도 |
|---|---|
| **Time series** | 시계열 line graph (가장 많이) |
| **Stat** | 단일 큰 숫자 (현재 값) |
| **Gauge** | 게이지 (0~max 비율) |
| **Bar chart** | 막대 |
| **Table** | 데이터 표 |
| **Heatmap** | 분포 시각화 (latency distribution 등) |
| **Logs** | log line 보기 (Loki·Elasticsearch) |
| **Trace** | trace span 시각화 (Tempo·Jaeger) |
| **State timeline** | 상태 변화 timeline |
| **Pie chart** | (가급적 회피 — 가독성 낮음) |

---

## 3. Variables — *filter/dropdown*

### 발상
- 한 대시보드를 *여러 service / cluster / env에 재사용*
- 매번 새 대시보드 만들지 말고 *변수로 필터*

### 변수 정의
```
Variable name: service
Type: Query
Datasource: Prometheus
Query: label_values(http_requests_total, service)
Multi-value: true
Include All: true
```

→ 대시보드 상단에 *service dropdown* 생김. 선택 시 *모든 panel에 적용*.

### 사용
```promql
sum by (instance) (rate(http_requests_total{service=~"$service"}[5m]))
```

→ `$service` 가 *선택된 값으로 치환*. `=~` 로 multi-value도 OK.

### 자주 쓰는 변수 패턴
- **service** — `label_values(metric, service)`
- **environment** — `label_values(metric, env)`
- **instance** — service에 따라 필터링
- **interval** — `1m, 5m, 1h` 선택지
- **datasource** — 여러 cluster의 대시보드 통합

### Chained variables
- `env` 선택 → `service` 가 *그 env의 service만*
```
Service query: label_values(http_requests_total{env="$env"}, service)
```

---

## 4. 좋은 Dashboard 디자인 원칙

### 1. *청중을 정해라*
- **Service team** — 자기 service만 (RED metric, dependency health)
- **SRE / Oncall** — 사고 시 *infra 상위 + 서비스 영향*
- **경영진** — SLO 추이, 사고 횟수, cost

→ 한 대시보드가 *모든 청중 만족*은 어려움. *청중별 분리*.

### 2. *질문을 정해라*
- "지금 서비스 건강한가?" — RED + SLO
- "어디서 사고났나?" — error rate by service, top error endpoints
- "성능 추이는?" — long-term latency, throughput

→ *대시보드가 *답할 질문 명확*해야*. 그게 아니면 *시각화 chaos*.

### 3. *왼쪽 위 = 가장 중요한 정보*
- 사람 시선은 *왼쪽 위에서 시작*
- 핵심 정보 (SLO 상태, error rate, availability) *최상단·왼쪽*
- 디테일·breakdown 은 *아래로*

### 4. *unit과 threshold 명시*
- *수치만 보면 모호*: "0.05"가 좋은 거? 나쁜 거?
- *Unit*: %, ms, req/s 명시
- *Threshold + color*: 0.01 이하 green, 0.05 이상 red

### 5. *너무 많이 panel 박지 말기*
- 한 화면에 *6~12 panel*이 적정
- 그 이상이면 *각 panel 작아 보기 어려움*
- *여러 dashboard로 분리* + *link로 연결*

### 6. *변수 활용 — 재사용*
- *service별 별도 dashboard* 안 만들고 *variable로 한 dashboard*

### 7. *Time range 일관*
- panel별로 time range 다르면 *혼란*
- 대시보드 *전체 time range* 사용 (`$__range`)

### 8. *Annotation — 사건 marker*
- deploy / incident / maintenance를 *graph에 marker로*
- "이 spike가 deploy랑 관련 있나?" *즉시 확인*

---

## 5. *RED dashboard*의 표준 layout

```
+------------------------------------------------------+
| [Service Health]      Variables: service env         |
|                                                       |
| ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
| │ Avail %  │ │ Req/s    │ │ Err %    │ │ p99 ms   │ │  ← Stat panels (현재 값)
| │ 99.95%   │ │ 1.2K     │ │ 0.05%    │ │ 145ms    │ │     큰 숫자
| └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
|                                                       │
| ┌────────────────────┐ ┌────────────────────┐        │
| │ Request Rate        │ │ Error Rate          │       │  ← Time series (추이)
| │ (graph)             │ │ (graph)             │       │
| └────────────────────┘ └────────────────────┘        │
|                                                       │
| ┌────────────────────┐ ┌────────────────────┐        │
| │ Latency (p50/95/99) │ │ Status code breakdown│      │
| └────────────────────┘ └────────────────────┘        │
|                                                       │
| ┌──────────────────────────────────────────────┐    │
| │ Top errors / Recent errors (table or logs)    │    │
| └──────────────────────────────────────────────┘    │
+------------------------------------------------------+
```

→ *RED + SLO + 추이 + breakdown* 의 표준.

---

## 6. Provisioning — *대시보드도 code로*

### 발상
- UI 클릭으로 만든 dashboard = *수동 관리*, *재현 어려움*
- *JSON으로 export → git 저장 → provisioning으로 자동 import*

```yaml
# /etc/grafana/provisioning/dashboards/main.yml
apiVersion: 1
providers:
- name: 'default'
  orgId: 1
  folder: 'Production'
  type: file
  options:
    path: /var/lib/grafana/dashboards
```

→ `/var/lib/grafana/dashboards/*.json` 자동 import + *git 변경 시 자동 갱신*.

### Grafana Operator (K8s)
- *Grafana Dashboard를 CRD로 관리*
- ArgoCD로 *dashboard 코드 변경 → 자동 배포*

### kube-prometheus-stack
- 25+ pre-built dashboards (K8s 노드·Pod·Deployment·etcd 등) 자동 import
- *시작 즉시 *광범위한 dashboard 활용*.

---

## 7. Grafana Alerting (vs Alertmanager)

### Grafana Alerting (Unified)
- *Grafana 자체의 alerting 엔진*
- *모든 datasource* 에 대해 알람 설정 가능 (Prometheus 외 MySQL, CloudWatch 등)
- *UI에서 alert 정의·관리*

### Alertmanager와 차이
- Prometheus alerting rule + Alertmanager: *코드 기반, GitOps 친화*
- Grafana Alerting: *UI 기반*, *non-Prometheus datasource 가능*
- → 운영에선 *Prometheus rule이 표준* (코드 관리·재현), Grafana는 *임시·특수 datasource*

### Grafana → Alertmanager 통합
- Grafana가 *외부 Alertmanager*에 alert 전송 가능
- *둘 다 사용* 시 routing 정책 통일

---

## 8. Datadog / NewRelic vs Grafana

### Grafana
- *오픈소스* (+ 유료 Enterprise / Cloud)
- *여러 datasource 통합 시각화*
- *self-hosted 가능*
- 학습 곡선·운영 부담 있음

### Datadog / NewRelic
- *SaaS — metric + log + trace + alerting 통합*
- *agent 한 번 깔면 끝* — 운영 부담 0
- *비용 ↑* (특히 cardinality)
- *vendor lock-in*

### 선택 기준
- *조직 작음 + 빠른 시작* → SaaS
- *규모 큼 + 비용 민감 + 자체 운영 가능* → Prometheus + Grafana + Loki/Tempo
- *하이브리드 흔함* — Datadog APM + 자체 Prometheus (cost 분산)

---

## 9. Performance — *느린 대시보드*

### 흔한 원인
- *쿼리가 너무 광범위* — `metric{}` (필터 없음)
- *복잡한 PromQL* — 매 refresh마다 계산
- *cardinality 큰 metric* — 쿼리 시간 ↑
- *time range 너무 큼* — 1년치 데이터

### 최적화
- **Recording rule** ([02 참조](02-prometheus.md)) — 복잡한 쿼리 pre-aggregate
- **변수 활용** — `service="$service"` 로 cardinality 좁히기
- **Time range 짧게** — 기본은 *지난 1시간*
- **Auto-refresh 끄기** — 항상 새로고침 X (특히 대시보드 30+)
- **Panel 적게** — 한 화면 12 이하

---

## 10. 자주 헷갈리는 것

### 10-1. *dashboard JSON import 시 datasource UID 충돌*
- Export된 JSON엔 *datasource UID* 박혀 있음
- 다른 환경에 import 시 UID 안 맞아서 *panel data 안 나옴*
- 해결: *변수로 datasource 명시* (`${DS_PROMETHEUS}`)

### 10-2. *대시보드 *변경했는데 git에 안 들어감**
- UI에서 *Save* 만 누름 → DB에 저장, 파일 X
- *Provisioning 파일이 source of truth* — UI 변경은 *임시*. JSON export 후 git commit 필요.

### 10-3. *변수 default 값*
- 변수 안 선택하면 *그 panel 비어 보임*
- `All` option 또는 *default value* 명시

### 10-4. *Panel time range가 *dashboard time range와 다름**
- panel별 *override* 가능 — 의도 아니면 *전체 사용*
- `Override panel timerange` 옵션 끄기

### 10-5. *Alerting이 *두 시스템에 분산**
- Prometheus rule + Grafana Alerting 동시 사용 시 *중복 알람*
- *한 곳으로 통일* (보통 Prometheus rule)

### 10-6. *NEW dashboard만들었는데 admin만 보임*
- Folder permission 설정 — *Team에 view/edit 권한 부여*
- Organization 단위 권한 관리

---

## 🎤 면접 빈출 Q&A

### Q1. Grafana의 역할? Prometheus와의 관계?
> Grafana = *시각화·대시보드 도구*. *여러 datasource* 통합 (Prometheus, Loki, Tempo, MySQL, CloudWatch 등). Prometheus는 *metric DB + scraper + alerting* — *데이터*. 보통 *함께 사용*: Prometheus가 데이터, Grafana가 보여줌. K8s에선 *kube-prometheus-stack* 으로 한 번에 설치.

### Q2. Variables는 언제·어떻게?
> *대시보드를 여러 service/env/cluster에 재사용*하려고. UI에 *dropdown* 생성, panel에서 `$service` 처럼 치환. **Query 변수** (label_values), **interval 변수**, **datasource 변수** (멀티 cluster 통합). chained variable로 *env → service*처럼 종속.

### Q3. 좋은 dashboard 디자인 원칙?
> (1) *청중·질문 명확*. (2) *왼쪽 위 = 가장 중요*. (3) *unit·threshold·color 명시*. (4) *panel 6~12개* — 그 이상은 분리. (5) *RED + SLO + 추이* 표준 layout. (6) *annotation으로 deploy/incident marker*. (7) *변수로 재사용*. (8) *Provisioning으로 코드 관리* (UI 클릭은 임시).

### Q4. Grafana Alerting vs Prometheus Alertmanager?
> Prometheus rule + Alertmanager: *코드 기반·GitOps 친화·표준*. Grafana Alerting: *UI 기반, 모든 datasource (Prometheus 외 MySQL·CloudWatch 등) 가능*. **운영 표준은 Prometheus rule**. Grafana Alerting은 *특수 datasource·임시*. *두 시스템 동시 사용 시 중복 알람 주의* — 한 곳으로 통일.

### Q5. Provisioning이 *왜 필요*한가?
> UI 클릭으로 만든 대시보드는 *수동·재현 어려움*. 운영에선 *대시보드를 코드 (JSON) 로 git 관리* → CI에서 *Grafana에 자동 import*. **Grafana Operator (K8s)** 로 *Dashboard CRD* + ArgoCD 통합도 가능. *Source of truth = git*.

### Q6. 대시보드가 *느릴 때* 어떻게?
> (1) **Recording rule**로 복잡한 쿼리 pre-aggregate. (2) **변수**로 cardinality 좁히기 — `service="$service"`. (3) **Time range 짧게** — 기본 지난 1시간. (4) **Auto-refresh 신중** — 30+ 대시보드 자동 refresh 부담. (5) **Panel 줄이기** — 화면 12 이하. (6) **PromQL 최적화** — 불필요한 high-cardinality 필터.

### Q7. Datadog/NewRelic 대신 *Grafana + Prometheus 자체 운영* 의 trade-off?
> **자체 운영 (Grafana + Prometheus + Loki + Tempo)**: *비용 통제, vendor lock-in 없음*, 단 *운영 부담 ↑*. **SaaS (Datadog/NewRelic)**: *agent 깔면 끝*, *통합 UX*, 단 *비용 ↑ (특히 cardinality)*, *vendor lock-in*. **하이브리드 흔함** — 일부 영역만 SaaS, 나머지 자체.

---

## 🔗 Cross-reference

- **Prometheus / PromQL** → [02-prometheus.md](02-prometheus.md)
- **Alertmanager** → [03-alerting.md](03-alerting.md)
- **SLO 대시보드 패턴** → [05-slo-and-error-budget.md](05-slo-and-error-budget.md)
- **Loki / Tempo / OpenTelemetry** → [06-logs-and-traces.md](06-logs-and-traces.md)
- **Grafana Operator (CRD 패턴)** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **ArgoCD로 dashboard 배포** → [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. *Grafana = 여러 datasource 통합 시각화*. Prometheus·Loki·Tempo·MySQL·CloudWatch 등 *수십 종*. K8s는 *kube-prometheus-stack*.
2. *Dashboard 디자인 = 청중 + 질문 + 조직화*. RED + SLO + 추이 + breakdown. Variables로 *재사용*. *Provisioning으로 코드 관리*.
3. *Alertmanager는 코드·GitOps 표준*, Grafana Alerting은 *특수·임시*. SaaS vs 자체 운영은 *비용 vs vendor lock-in* trade-off.

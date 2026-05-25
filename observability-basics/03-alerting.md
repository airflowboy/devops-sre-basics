# 03 — Alerting: Alertmanager · routing · grouping · inhibition · PagerDuty

> **공식 근거:** [Alertmanager docs](https://prometheus.io/docs/alerting/latest/alertmanager/), [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/), [PagerDuty docs](https://developer.pagerduty.com/)

---

## 🎯 한 문장

> **Prometheus가 *alert 생성*, Alertmanager가 *grouping + 중복 제거 + 라우팅 + inhibition + silencing*. 알람이 *팀에게 가서 oncall 호출되기까지의 모든 처리*가 Alertmanager 책임. 좋은 알람 = *actionable + symptom-based + multi-burn-rate*.**

---

## 1. 알람 흐름

```
┌─────────────┐  alert  ┌──────────────┐ notification ┌─────────────┐
│ Prometheus  │ ──────► │ Alertmanager  │ ──────────► │  PagerDuty  │
│ (rule eval) │         │  - group      │             │  Slack      │
│             │         │  - inhibit    │             │  Email      │
│             │         │  - silence    │             │  Webhook    │
│             │         │  - route      │             │             │
└─────────────┘         └──────────────┘             └─────────────┘
                                                              │
                                                              ▼
                                                          ┌────────┐
                                                          │ oncall │
                                                          │ engineer│
                                                          └────────┘
```

---

## 2. Alerting Rule — Prometheus 측

```yaml
# alerting-rules.yml
groups:
- name: critical_services
  interval: 30s
  rules:
  - alert: HighErrorRate
    expr: |
      sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
      / sum by (service) (rate(http_requests_total[5m]))
      > 0.05
    for: 5m                       # 5분 지속해야 firing
    labels:
      severity: critical
      team: payments
    annotations:
      summary: "High error rate on {{ $labels.service }}"
      description: |
        Service {{ $labels.service }} has error rate {{ $value | humanizePercentage }}
        for the last 5 minutes.
      runbook: "https://wiki.example.com/runbooks/high-error-rate"
      dashboard: "https://grafana.example.com/d/abc?var-service={{ $labels.service }}"
```

### Lifecycle
1. Prometheus가 *주기적으로 `expr` 평가*
2. *True가 되면* `pending` 상태로 진입
3. `for:` 기간 *내내 true 유지*되면 `firing` 으로 전환
4. Alertmanager에 *전송*
5. `expr` 가 *false*가 되면 *resolved* 알림 전송

### `for:` 의 의미
- *flapping (잠깐만 true)* 방지
- 짧은 spike에 알람 안 가게
- *너무 길면 사고 대응 늦음*, *너무 짧으면 노이즈*

### Labels vs Annotations
- **Labels**: *알람의 정체성* — 같은 label 조합 = 같은 알람. 라우팅·inhibition에 사용.
- **Annotations**: *사람이 읽는 정보* — summary, description, runbook URL, dashboard URL. 라우팅에 안 씀.

---

## 3. Alertmanager 설정

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  slack_api_url: 'https://hooks.slack.com/services/...'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
  - matchers:
    - severity = critical
    receiver: 'pagerduty-critical'
    group_wait: 10s
    repeat_interval: 1h
    
  - matchers:
    - severity = warning
    receiver: 'slack-warnings'
    
  - matchers:
    - team = payments
    - severity = critical
    receiver: 'pagerduty-payments'

receivers:
- name: 'default'
  slack_configs:
  - channel: '#alerts-general'

- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: '<integration-key>'
    description: '{{ .CommonAnnotations.summary }}'
    details:
      firing: '{{ .Alerts.Firing | len }}'

- name: 'slack-warnings'
  slack_configs:
  - channel: '#alerts-warnings'
    title: '{{ .CommonAnnotations.summary }}'
    text: '{{ .CommonAnnotations.description }}'

inhibit_rules:
- source_matchers:
  - severity = critical
  target_matchers:
  - severity = warning
  equal: [alertname, cluster, service]
```

---

## 4. Grouping — *비슷한 알람 묶기*

### 발상
- 같은 시점에 *비슷한 알람 100개* 발생 (예: 한 노드 down → 그 노드의 모든 Pod alert)
- 100개 페이지 = *PagerDuty 노이즈 + 비용 + 인지 부하*
- **Grouping**: 같은 *labels로 식별*되는 알람을 *한 notification으로 묶음*

### `group_by`
```yaml
group_by: ['alertname', 'cluster', 'service']
```
- 위 label들이 *같은 알람은 한 그룹*
- 한 notification 안에 *N개 alert 정보*

### Timing
- `group_wait`: 첫 알람 도착 후 *기다리는 시간* — 다른 비슷한 알람 모음
- `group_interval`: 같은 그룹에 *새 알람 추가* 발생 시 *다시 알림 보낼 간격*
- `repeat_interval`: *같은 알람이 계속 firing* 일 때 *재알림 간격*

### 예
- 12:00:00 — HighErrorRate (service=payments) firing
- 12:00:05 — HighErrorRate (service=auth) firing
- 12:00:10 — HighErrorRate (service=order) firing

`group_wait: 30s` → 12:00:30 에 *한 notification에 3 service 다* 묶어서 보냄.

→ 100개 알람 → *1 페이지에 100개 list* (PagerDuty incident도 1개).

---

## 5. Inhibition — *상위 알람이 fire하면 하위 알람 숨김*

### 발상
- *클러스터 down* → "CPU low", "memory low", "node down" 등 *수십 알람*
- 진짜 root cause = "cluster down"만 보고싶음
- **Inhibition**: 한 알람이 fire 시 *다른 알람 (같은 label 매칭) 무음*

```yaml
inhibit_rules:
- source_matchers:
  - alertname = ClusterDown
  target_matchers:
  - alertname =~ "PodDown|HighErrorRate|...
  equal: [cluster]
```

→ `ClusterDown` 알람 firing 시 *같은 cluster의 PodDown / HighErrorRate 알람 무음*.

### 흔한 패턴
- *node down → 그 node의 모든 Pod 알람 inhibit*
- *deploy 진행 중 → 일시적 error 알람 inhibit* (deploy 관련 알람은 안 보냄)
- *upstream service down → downstream 알람 inhibit*

---

## 6. Silencing — *수동 무음*

### 발상
- *예정된 maintenance* → 알람 발생 알지만 *지금은 알림 안 받고 싶음*
- *false alarm 조사 중* → 그 알람 잠시 무음

### 사용
- Alertmanager UI 또는 API에서 *silence 생성*
- *matcher* (label 조합) + *기간* + *작성자* + *코멘트*
- silence 활성 시 *해당 알람 notification 안 감*
- *기간 지나면 자동 해제*

```bash
amtool silence add alertname="HighErrorRate" service="payments" \
  --duration=2h \
  --comment="Investigating, see incident-1234"
```

### Silence vs Inhibition
- **Silence**: *수동, 시간 제한*. 한 번 만들고 잊음.
- **Inhibition**: *룰 기반, 자동*. 항상 적용.

→ Silence는 *임시 사고 대응*, Inhibition은 *영구 정책*.

---

## 7. Routing — *어느 receiver로*

### 트리 구조
- `route` 가 *root*
- *matcher 매칭 시 그 sub-route로 raisin*
- *no match 면 default*

```yaml
route:
  receiver: 'default'
  routes:
  - matchers: [severity = critical]
    receiver: 'pagerduty'
    routes:
    - matchers: [team = payments]
      receiver: 'pagerduty-payments'
    - matchers: [team = data]
      receiver: 'pagerduty-data'
  - matchers: [severity = warning]
    receiver: 'slack'
```

→ severity=critical + team=payments → `pagerduty-payments` (matchers 정확히 일치).
→ severity=critical + team=other → `pagerduty` (parent route).
→ severity=warning → `slack`.

### `continue:` — 매칭 후에도 *다음 route 확인*
```yaml
- matchers: [team = payments]
  receiver: 'slack-payments'
  continue: true                # 매칭됐어도 *다음 route 계속 확인*
```

→ 한 알람을 *Slack + PagerDuty 동시*에 보내고 싶을 때.

---

## 8. PagerDuty 통합

### 발상
- *Alertmanager의 receiver* 중 하나
- PagerDuty가 *escalation·oncall rotation·incident 관리* 전담

### 흐름
```
Alertmanager → PagerDuty Events API → Incident 생성
                                          │
                                          ▼
                                   PagerDuty Service
                                          │
                                          ▼
                                Escalation Policy
                                          │
                                          ▼
                                    Oncall Schedule
                                          │
                                          ▼
                                  oncall person 알림
                                  (SMS / phone call / app push)
```

### Service / Escalation Policy / Schedule
- **Service**: *알람의 그룹화 단위* (예: "payments-service")
- **Escalation Policy**: *알림이 *N분 안 ack 되면 다음 사람*에게* 룰
  - Level 1: primary oncall (5분 안 ack 안 되면)
  - Level 2: secondary oncall (10분 안 ack 안 되면)
  - Level 3: team lead
- **Schedule**: *주차별 누가 oncall* 정의 (rotation)

### `incident_key` — *중복 방지*
- 같은 알람 (`alertname + labels`) → *같은 incident에 update*
- *새 incident 안 만듦* — PagerDuty 입장에선 *한 사건의 update*
- Alertmanager가 자동 처리

### Resolved → auto-close
- 알람 resolved 시 Alertmanager가 PagerDuty에 *event_action: resolve* 전송
- PagerDuty incident *자동 close*

---

## 9. *Symptom-based vs Cause-based* — 좋은 알람의 원칙

### Cause-based (옛)
- "CPU 80% 넘음"
- "Memory 90% 넘음"
- "Disk I/O 높음"

→ 알람 *시끄러움*. *사용자에게 영향*인지 모름. *작업 끊김*.

### Symptom-based (현대 SRE)
- "사용자 요청의 *5xx 비율이 5% 넘음*"
- "*p99 latency가 SLO 위반*"
- "사용자가 *log in 못 함*"

→ *사용자가 체감하는 문제만* 알람. *진짜 사고만 oncall 호출*.

### 좋은 알람의 5조건
1. **Actionable** — 받으면 *바로 무엇을 해야 할지* 명확 (runbook 링크 필수)
2. **Symptom-based** — 사용자 영향 있을 때만
3. **Multi-burn-rate** — 빠른 + 느린 burn 동시 감지 ([05](05-slo-and-error-budget.md))
4. **Not flapping** — `for:` 적절히
5. **De-duplicated** — group/inhibition/silence로 노이즈 제거

---

## 10. 자주 헷갈리는 것

### 10-1. *알람 너무 많음 → 무시*
- *알람 노이즈 = 시스템이 사용 안 됨*
- 정기적 *알람 audit*: 지난 1개월 *각 알람이 몇 번 fire*, *몇 번 actionable* 이었나
- *actionable 0건* 인 알람은 *삭제 또는 룰 조정*

### 10-2. *Alertmanager가 dedupe 안 함 → 같은 알람 N번*
- *labels가 정확히 같아야* 같은 알람으로 인식
- target instance마다 다른 label이면 *각자 다른 알람* → group_by에 instance 빠뜨림 확인

### 10-3. *PagerDuty incident가 *자동 close 안 됨**
- Alertmanager에서 *resolved 전송 안 함*
- alertmanager.yml의 receiver 설정 `send_resolved: true` 확인

### 10-4. *Silence가 *resolved도 막음**
- silence 활성 시 *firing·resolved 둘 다 무음*
- 의도된 동작이나 *silence 끝나면 *복귀 알림* 안 옴*

### 10-5. *Group_wait 너무 길면 첫 알람 지연*
- 30s 기본 — *critical 알람은 더 짧게* (5~10s)
- *warning은 길게* (60s+) — 노이즈 줄임

### 10-6. *Inhibit가 의도와 달리 *너무 많이 막음**
- source/target matcher가 *너무 광범위*
- *equal:* 빠뜨려서 *다른 cluster까지 영향*

---

## 🎤 면접 빈출 Q&A

### Q1. Prometheus alert와 Alertmanager 차이?
> Prometheus는 *alerting rule을 평가해서 alert 생성*. Alertmanager는 *그 alert를 받아 grouping·deduplication·inhibition·silencing·routing*해서 *실제 notification 전송* (Slack·PagerDuty·Email·Webhook). **분리 이유**: Prometheus는 *데이터·룰*, Alertmanager는 *팀·통지 정책*. 여러 Prometheus가 같은 Alertmanager 사용 가능 (HA).

### Q2. Grouping이 왜 필요한가?
> 한 사고에 *비슷한 알람 수십~수백 개* 발생 (예: 노드 down → 그 노드의 모든 Pod). 한 번에 보내면 *PagerDuty 노이즈·비용·인지 부하*. **`group_by` 같은 label로 묶기** → 한 notification에 N alert. **`group_wait`** (첫 알람 후 다른 비슷한 알람 모음), **`group_interval`** (그룹에 새 alert 추가 시 재알림), **`repeat_interval`** (계속 firing 시 재알림).

### Q3. Inhibition과 Silencing 차이?
> **Inhibition**: *룰 기반 자동*. "한 alert (source)가 firing이면 다른 alert (target) 무음" — 예: ClusterDown → PodDown 무음. **Silencing**: *수동·시간 제한*. UI에서 matcher + 기간 + 코멘트로 생성. 예: 계획된 maintenance, 조사 중. 둘 다 *notification 막음*, 차이는 *자동 vs 수동*.

### Q4. Routing은 어떻게 작동?
> *Tree 구조*. root route → matcher 매칭되면 sub-route로 descent. *no match 면 default receiver*. `continue: true`로 매칭 후에도 *다음 route 계속 확인* (한 alert를 *Slack + PagerDuty 동시* 같은 경우). severity·team·environment 등 label로 라우팅.

### Q5. PagerDuty의 Escalation Policy는?
> *알림이 N분 안에 ack 안 되면 *다음 사람에게**. Level 1 (primary oncall, 5분) → Level 2 (secondary, 10분) → Level 3 (team lead). *Schedule*과 결합 — 누가 oncall인지 시간대별 정의 (rotation). PagerDuty가 *SMS·call·app push로 깨움*. Alertmanager는 *events API에 incident 생성*만.

### Q6. Symptom-based vs cause-based alert?
> *Cause-based* (옛): "CPU 80%", "memory 90%" — *사용자 영향 모름*, *시끄러움*. *Symptom-based* (현대 SRE): "5xx 비율 5% 넘음", "p99 latency SLO 위반", "사용자가 log in 못 함" — *사용자가 체감하는 문제만*. **진짜 사고만 oncall 호출**. cause-based는 *대시보드에서 디버깅 도구로*.

### Q7. Alert 노이즈를 줄이는 방법?
> (1) **Symptom-based** alert로 — cause는 dashboard에. (2) **Multi-burn-rate alert** — 짧은 + 긴 윈도우 동시 감지로 *flap 줄임* ([05](05-slo-and-error-budget.md)). (3) **`for:`** 적절히 — flapping 방지. (4) **Grouping**으로 묶기. (5) **Inhibition**으로 cascading 알람 차단. (6) 정기 **alert audit** — actionable 0건은 *삭제*. (7) **runbook 링크 필수** — runbook 없는 알람은 *알람 가치 X*.

---

## 🔗 Cross-reference

- **Prometheus alerting rule 작성** → [02-prometheus.md](02-prometheus.md)
- **SLO + multi-burn-rate alert** → [05-slo-and-error-budget.md](05-slo-and-error-budget.md)
- **Grafana alerting (Alertmanager 대안)** → [04-grafana-and-dashboards.md](04-grafana-and-dashboards.md)
- **K8s에서 AlertmanagerConfig CRD** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **K8s NetworkPolicy로 alert manager 보호** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md)

---

## 📝 3줄 요약

1. Prometheus *alert 생성*, Alertmanager *grouping/dedup/inhibition/silencing/routing*. 분리로 *데이터-정책 책임 분담*.
2. *Symptom-based + actionable + multi-burn-rate + grouping + inhibition* = 좋은 알람의 5조건. cause-based는 *대시보드에서*.
3. PagerDuty가 *Schedule + Escalation Policy*로 oncall 호출. 정기 *alert audit*으로 actionable 0건 *삭제*.

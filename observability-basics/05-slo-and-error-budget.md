# 05 — SLO & Error Budget: 현대 SRE의 운영 결정 베이스

> **공식 근거:** [Google SRE Book Ch.4 - SLO](https://sre.google/sre-book/service-level-objectives/), [Implementing SLOs (SRE Workbook)](https://sre.google/workbook/implementing-slos/), [The Site Reliability Workbook](https://sre.google/workbook/)
>
> **이 파일이 *현대 SRE의 핵심 사상*.** 모든 운영 결정의 베이스.

---

## 🎯 한 문장

> **SLI = *측정 가능한 사용자 경험 지표*, SLO = *그 SLI의 목표*, Error Budget = *100% - SLO* (*감당 가능한 사고 분량*). Budget 남으면 *기능 출시 OK*, 다 쓰면 *안정성 작업으로 전환*. 단순 알람이 아닌 *데이터 기반 의사 결정 프레임워크*.**

---

## 1. SLI / SLO / SLA — *위계*

| 용어 | 의미 | 누구 |
|---|---|---|
| **SLI** (Service Level Indicator) | *측정 가능한 지표* — "성공한 요청 수 / 총 요청 수" | 엔지니어 |
| **SLO** (Service Level Objective) | *내부 목표* — "그 SLI가 *30일 동안 99.9%* 여야" | 엔지니어 + product |
| **SLA** (Service Level Agreement) | *외부 약속* — 위반 시 *환불·penalty*. SLO보다 *느슨*. | 비즈니스 + 법무 |
| **Error Budget** | *100% - SLO* — *감당 가능한 사고 분량* | SRE 운영 도구 |

### 관계
```
SLI < SLO < SLA
실제 측정값  내부 목표  계약상 약속
   ↑       ↑         ↑
   99.95%  99.9%     99.5%
```

- SLO는 *SLA보다 항상 strict* — *내부 buffer*
- SLI가 SLO 위반해도 SLA 안 위반할 수 있음 — *조기 경고*

---

## 2. *좋은 SLI* 의 조건

### 1. *사용자가 체감하는 것*
- "*CPU 80%*" — 사용자 모름 ❌
- "*요청의 응답 시간 < 200ms 비율*" — 사용자 체감 ✅

### 2. *측정 가능*
- 정의가 명확 — 매번 같은 결과
- 시계열로 수집 가능 (Prometheus 등)

### 3. *집계 가능*
- *비율 (ratio)* 형태가 가장 좋음 — *0~1 범위*
- 예: 성공 요청 / 전체 요청

### SLI 예시
| 종류 | SLI |
|---|---|
| **Availability** | 성공 응답 수 / 전체 요청 수 |
| **Latency** | 응답 시간 < 200ms 인 요청 수 / 전체 요청 수 |
| **Throughput** | 처리한 jobs / 총 큐 길이 |
| **Quality** | 정상 응답 수 / *불완전한 응답 포함* 전체 |
| **Freshness** | 최신 데이터 (5분 이내) 응답 수 / 전체 |

### *Bad SLI 사례*
- "*p99 latency*" — *집계 어려움*, *outlier에 민감*
- 더 좋은 표현: "*p99가 200ms 이하인 시간의 비율*"

→ **SLI는 항상 *good event / total event 비율*로 표현**.

---

## 3. SLO 정의 — *5가지 컴포넌트*

```
"99.9% of API requests will return success
within 200ms over a rolling 30-day window."
 ↑       ↑                ↑       ↑           ↑
 목표    SLI              조건    측정 윈도우  
```

| 컴포넌트 | 예 |
|---|---|
| **목표 (목표 %)** | 99.9%, 99.95%, 99.99% |
| **SLI** | API 요청 성공률 |
| **조건** | success + latency < 200ms |
| **측정 윈도우** | 7일 / 28일 / 30일 / 90일 |
| **scope (영향 받는 user/요청)** | 모든 user / paid user만 / 특정 region 등 |

### 윈도우 선택
- **짧음** (7일) — *빠른 피드백, 변동성 크다*
- **김** (28~90일) — *안정적, 사고 잊혀짐*
- **표준** = *28일 rolling* (Google SRE)

### 목표 % 의 의미
| 목표 | 30일 budget |
|---|---|
| 99% | 7.2시간 |
| 99.5% | 3.6시간 |
| 99.9% | 43분 |
| 99.95% | 21.6분 |
| 99.99% | 4.3분 |
| 99.999% (5 9s) | 26초 |

→ **현실적으로 *99.9% 가 일반 서비스의 기본***. *99.999%는 매우 어렵다*. 비즈니스 가치 vs 비용 trade-off.

### *과도한 SLO 사고*
- "*100% availability* 목표" — *fail이 허용 안 됨 → 기능 출시 영원히 막힘*
- *과도한 SLO* = *팀이 무력해짐*
- → **달성 가능 + 비즈니스 가치** 양쪽 만족

---

## 4. Error Budget — *현대 SRE의 핵심 발명*

### 정의
- **Error Budget = 100% - SLO**
- *측정 윈도우 동안 *허용되는 실패 분량**

### 99.9% SLO + 30일 윈도우
```
Total requests in 30 days: 100,000,000
SLO: 99.9% success → 99,900,000 success
Error budget: 100,000 failures allowed

If 50,000 failed already → 50% budget consumed
If 100,000 failed → budget exhausted → freeze deploys
```

### Error Budget Policy — *팀 운영 규칙*
**Budget 남음 (예: < 50% 소비)**
- *기능 출시 자유*
- *실험 가능*
- *위험 감수 OK*

**Budget 부족 (50%~80% 소비)**
- *주의 단계*
- *큰 변경 review 강화*

**Budget 거의 소진 (80~95%)**
- *비기능 작업·안정성 작업 우선*
- *큰 변경 보류*

**Budget 소진 (100%)**
- **기능 deploy 동결** (또는 *매우 보수적*)
- *모든 팀 자원이 *안정화에 투입**
- *post-mortem 작성·재발 방지*

→ *Budget이 *정량적 *기능 vs 안정성 trade-off* 결정 도구*. *이게 SRE의 핵심 발명*.

---

## 5. PromQL로 SLO·Error Budget 구현

### SLI: Availability (request 성공률)
```promql
sum(rate(http_requests_total{status!~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

### SLO budget — 30일 윈도우
```promql
# 30일 동안 실제 SLI
slo_availability_30d:ratio_rate30d = 
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))

# Budget 남은 비율
slo_availability_30d:error_budget_remaining =
  (slo_availability_30d:ratio_rate30d - 0.999) / (1 - 0.999)
```

→ 결과:
- 1.0 = budget 100% 남음
- 0.0 = budget 정확히 소진
- < 0 = budget 초과 (SLO 위반)

### Recording rule로 pre-aggregate
```yaml
groups:
- name: slo_recording
  interval: 30s
  rules:
  - record: slo:requests_total:rate5m
    expr: sum by (service) (rate(http_requests_total[5m]))
  
  - record: slo:requests_errors:rate5m
    expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  
  - record: slo:availability:ratio5m
    expr: |
      1 - (slo:requests_errors:rate5m / slo:requests_total:rate5m)
```

---

## 6. Multi-Burn-Rate Alert — *현대 SLO 알람의 표준*

### 단순 알람의 문제
```yaml
# 단순 알람 (옛 방식)
- alert: HighErrorRate
  expr: error_rate > 0.01
  for: 5m
```

문제:
- *false positive 많음* — 5분 spike도 알람
- *진짜 SLO 위협 느림* — 천천히 budget 까먹는 건 감지 못 함

### Multi-burn-rate alert
**Burn rate** = *현재 속도로 가면 *N일 안에 한 달치 budget 다 씀**.

- *Burn rate 1.0* = 한 달치 budget을 한 달에 정확히 소진
- *Burn rate 14.4* = 한 달치 budget을 *2일 만에 다 씀*
- *Burn rate 36* = 한 달치 budget을 *6시간 안에 다 씀*

```promql
# Burn rate 계산 (5m 윈도우)
(1 - slo:availability:ratio5m) / (1 - 0.999)
```

### Google SRE 권장 룰
```yaml
# Page (긴급 — oncall 호출)
- alert: ErrorBudgetBurnRateHigh
  expr: |
    (
      (1 - slo:availability:ratio_rate5m) / (1 - 0.999) > 14.4
      and
      (1 - slo:availability:ratio_rate1h) / (1 - 0.999) > 14.4
    )
  for: 2m
  labels:
    severity: page
  annotations:
    summary: "Burning error budget too fast (2% in 1h)"

# Ticket (느림 — 시간 안에 처리)
- alert: ErrorBudgetBurnRateMedium
  expr: |
    (
      (1 - slo:availability:ratio_rate30m) / (1 - 0.999) > 6
      and
      (1 - slo:availability:ratio_rate6h) / (1 - 0.999) > 6
    )
  for: 15m
  labels:
    severity: ticket
```

### 왜 *두 윈도우 동시*
- *짧은 윈도우 (5m)*: *빠른 감지*
- *긴 윈도우 (1h)*: *flap 방지 (잠깐만 fail이면 1h 평균은 낮음)*
- **둘 다 임계 넘어야 firing** — false positive 대폭 감소

### 4-window 권장 룰
| 알람 | Burn rate | Short window | Long window | severity |
|---|---|---|---|---|
| Fast burn | 14.4 | 5m | 1h | page |
| Medium burn | 6 | 30m | 6h | page |
| Slow burn | 3 | 2h | 1d | ticket |
| Very slow | 1 | 6h | 3d | ticket |

→ *빠른 사고 + 천천히 까먹는 budget 둘 다 감지*.

---

## 7. SLO Dashboard

### 필수 panel
1. **현재 SLO 달성률** — Stat panel (큰 숫자)
2. **Error budget 남은 양** — Gauge (0~100%)
3. **Burn rate** — Time series (1.0 line + 현재 값)
4. **SLO 추이** — Time series (rolling 30d)
5. **Budget 소진 추이** — Time series (남은 budget %)
6. **Recent SLO violations** — Table (사고 시점 list)

### Annotation
- *Deploy 시점* — burn rate spike와 *관련 있나*
- *Incident 시작/종료* — budget 소진과 *관련 있나*

---

## 8. Error Budget Policy — *팀이 어떻게 사용?*

### Policy 예시 (Google SRE)
```
SLO: 99.9% availability over 28 days
Error budget: 0.1% (약 40분/month)

Budget remaining > 50%:
- Normal operation
- Feature deploys allowed
- Experiments allowed

Budget remaining 20-50%:
- Deploy review required
- Postmortems prioritized

Budget remaining < 20%:
- Deploy freeze for non-critical changes
- All team focuses on reliability
- Daily SLO review meeting

Budget exhausted:
- COMPLETE deploy freeze
- Mandatory postmortem for all incidents
- Director-level review required to unfreeze
```

### 운영의 변화
- 옛 모델: "*deploy 시 SRE 승인*"
- SLO 모델: "*Budget 있으면 자유, 없으면 동결*"
- → *데이터 기반*, *팀 자율성*, *책임 분담 명확*

### 비즈니스와의 대화
- Product manager: "*이번 분기 X 기능 빨리 출시해야*"
- SRE: "*Budget 70% 소진. 출시는 OK지만 *review 강화* 필요*"
- → *정성적 논쟁 X, *수치 기반 결정**.

---

## 9. *SLO 안티패턴* — 흔한 실수

### 9-1. *너무 높은 SLO*
- "99.999%" — *비현실적*, *budget 너무 작아 *모든 활동 위협**
- *시작은 *현재 달성 가능한 수준의 *약간 위***. 점진 향상.

### 9-2. *SLI가 *사용자 경험 아님**
- "CPU 80%" SLO — *사용자가 알 바 아님*
- *사용자 측면 지표*: 응답 성공, latency, freshness

### 9-3. *모든 endpoint를 같은 SLO*
- "Search" 와 "checkout" 은 *중요도 다름*
- *endpoint별 / journey별 SLO* 분리

### 9-4. *Budget 다 써도 *무시**
- Policy 없이 *알람만 발생*
- → *Budget exhaustion = 행동 변화*. 정책으로 강제.

### 9-5. *Window 너무 길게 (1년)*
- 사고 잊혀짐
- *28일 권장* — *최근성 + 안정성 균형*

### 9-6. *SLO를 *수동 측정**
- 매주 회의에서 *엑셀로 계산* — *시간 낭비, 부정확*
- *자동 계산·자동 대시보드·자동 알람*이 정석

---

## 10. 자주 헷갈리는 것

### 10-1. *SLO vs SLA 혼용*
- SLO = *내부 목표* (alerting·결정용)
- SLA = *외부 약속* (위반 시 환불)
- SLO < SLA — *내부 buffer*

### 10-2. *Error budget이 0이면 *서비스 down**?
- 아니. *서비스는 정상*, 단 *budget 다 썼다는 신호*.
- → *deploy 동결·안정화 우선* 등 *팀 행동 변화*.

### 10-3. *SLO 변경 = 영향 큼*
- SLO 변경 시 *모든 알람·대시보드·policy* 영향
- *분기별 review 권장*, 자주 바꾸지 말기

### 10-4. *Multi-burn-rate alert 의 *expr 복잡함**
- *recording rule로 burn rate pre-aggregate* 권장
- alert expr 단순화

### 10-5. *Synthetic monitoring vs Real User Monitoring*
- **Synthetic**: 자동 script가 *주기적 endpoint hit* — *availability SLO 측정*에 좋음 (트래픽 적은 시간도 가능)
- **RUM**: *실제 사용자 브라우저에서 metric* — *진짜 사용자 경험*. *Synthetic 보완*.

### 10-6. *SLO 측정에 *health check 만 사용**
- /healthz 만 측정 — *진짜 endpoint 동작은 모름*
- *실제 비즈니스 endpoint* 측정 또는 *synthetic monitoring*

---

## 🎤 면접 빈출 Q&A

### Q1. SLI / SLO / SLA / Error Budget 차이?
> **SLI** = *측정 지표* (성공 요청 / 전체 요청). **SLO** = *내부 목표* (99.9% over 28 days). **SLA** = *외부 약속* (위반 시 환불, SLO보다 느슨). **Error Budget** = *100% - SLO* (감당 가능한 사고 분량). Budget이 *기능 vs 안정성 trade-off 결정 도구*.

### Q2. Error budget policy는 무엇? 어떻게 운영?
> Budget 사용량에 따른 *팀 행동 규칙*. 예: <50% 소비 → 정상 운영, 50~80% → review 강화, 80~95% → 비기능 작업 우선, 100% 소진 → **deploy freeze**, 모든 팀 자원이 안정화에 투입, post-mortem 작성. *정량적 결정* — 정성적 논쟁 X.

### Q3. SLO를 정의할 때 *5가지 컴포넌트*?
> (1) **목표 %** (99.9%, 99.95%, ...), (2) **SLI** (어떤 측정 지표), (3) **조건** (success + latency < 200ms 등), (4) **측정 윈도우** (보통 28일 rolling), (5) **scope** (모든 user / paid user만 등). 예: "*99.9% of API requests will return success within 200ms over a rolling 28-day window for paid users*".

### Q4. Multi-burn-rate alert는 왜? 어떻게 작동?
> *단순 알람 (error rate > X)*은 *false positive 많음 + 천천히 까먹는 budget 감지 못 함*. **Burn rate** = *현재 속도로 한 달치 budget 소진까지 시간 비율*. **두 윈도우 동시** 조건 — 짧은 (5m, 빠른 감지) + 긴 (1h, flap 방지). Google SRE 권장: *14.4 burn rate + 5m/1h* (page), *6 + 30m/6h* (page), *3 + 2h/1d* (ticket), *1 + 6h/3d* (ticket).

### Q5. *너무 높은 SLO*가 *왜 나쁜가*?
> "99.999%" — *비현실적*. Budget 너무 작아 *모든 deploy·실험이 위협*. 팀이 *기능 출시 못 함 → 비즈니스 정체*. 반대로 *너무 낮은 SLO*는 *사용자 체감 나쁨*. **시작은 *현재 달성 가능한 수준의 약간 위***. 점진 향상. *비즈니스 가치 vs 운영 비용* 균형.

### Q6. SLO와 alert noise의 관계?
> *Symptom-based alert* (SLO 기반) = *사용자 체감 사고만 알람*. cause-based (CPU 80% 등) 의 *수십 알람 → SLO 위반 1 알람*. **Multi-burn-rate**로 *false positive 대폭 감소*. *알람 받으면 = 진짜 사고*. *Oncall 부담 ↓, 신뢰 ↑*.

### Q7. SLO를 *비즈니스와 어떻게 대화*?
> Product manager: "이번 분기 기능 빨리 출시" → SRE: "Budget 70% 소진, *출시 OK 단 review 강화 필요*". 옛 모델은 *정성적 논쟁* ("위험해요" vs "출시해야") → 자주 갈등. SLO 모델은 *데이터 기반* ("budget X% 남음"). **운영 결정의 *공통 언어***. 분기별 SLO 자체 review로 *기대치 조정*.

---

## 🔗 Cross-reference

- **Prometheus rule (SLO recording + multi-burn alert)** → [02-prometheus.md](02-prometheus.md), [03-alerting.md](03-alerting.md)
- **Argo Rollouts의 *metric-based promotion = 자동 SLO 검증*** → [`cicd-gitops-basics/04`](../cicd-gitops-basics/04-deployment-strategies.md)
- **K8s에서 PrometheusRule CRD** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **Postmortem 문화 (Blameless)** → 외부 SRE Workbook 참조
- **PagerDuty oncall** → [03-alerting.md](03-alerting.md)

---

## 📝 3줄 요약

1. *SLI (지표) < SLO (내부 목표) < SLA (외부 약속)*. **Error Budget = 100% - SLO** = *감당 가능한 사고 분량* — *기능 vs 안정성 결정의 정량적 도구*.
2. *SLO는 5컴포넌트* (목표%·SLI·조건·윈도우·scope). *28일 rolling 표준*. *99.9%가 일반 서비스 기본*, *99.999%는 매우 어려움*.
3. **Multi-burn-rate alert** (Google SRE) = *짧은 + 긴 윈도우 동시 조건* — false positive 대폭 감소. *Error Budget Policy*로 *팀 행동 규칙 자동화*.

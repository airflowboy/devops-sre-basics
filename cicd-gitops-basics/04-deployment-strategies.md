# 04 — Deployment Strategies: Rolling / Blue-Green / Canary

> **공식 근거:** [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-strategy), [Argo Rollouts docs](https://argo-rollouts.readthedocs.io/), Martin Fowler's "BlueGreenDeployment"
>
> **선행:** [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md) (Deployment, ReplicaSet), [03-argocd.md](03-argocd.md)

---

## 🎯 한 문장

> **배포 전략 = *위험을 *시간·트래픽·환경* 차원에서 어떻게 분산할 것인가*. RollingUpdate (기본, downtime 0) → Blue/Green (즉시 전환·즉시 rollback) → Canary (소수 트래픽 → 점진 확대 + 자동 분석).**

---

## 1. 왜 *배포 전략*이 필요한가

- 모든 배포는 *위험*. *잘 돌던 게 안 돌 수도*.
- 무전략 = "모두 한 번에 교체" → *문제 시 전체 영향, rollback 시간*
- 전략 = *위험을 분산·격리·빨리 감지·빨리 회복*

| 차원 | 분산 방법 |
|---|---|
| 시간 | Rolling — *점진 교체* |
| 환경 | Blue/Green — *별도 환경 띄우고 한 번에 전환* |
| 트래픽 | Canary — *소수 트래픽 → 점진 확대* |
| 사용자 | Feature flag — *코드 배포 ≠ feature 활성화* |

---

## 2. RollingUpdate (K8s 기본) — *점진 교체*

K8s Deployment의 *기본 전략*. [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md) 참조.

### 동작
- *새 ReplicaSet*에 Pod 추가 / *옛 RS*에서 Pod 제거 *동시*
- `maxSurge`, `maxUnavailable`로 속도 조절

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

### 장점
- *downtime 0*
- *추가 자원 적음* (surge 만큼만)
- *K8s 기본* — 별도 도구 X

### 단점
- *옛/새 버전이 *동시 운영*되는 시기 있음* — *데이터 호환성·세션 호환성* 주의
- *rollback도 점진* — 빠르지 않음
- *트래픽 분배가 *비율 기반이 아님** — Service가 *모든 ready Pod에 round-robin*. 새 버전 25%가 *항상 25% 트래픽 받는다는 보장 X*.
- *문제 감지 늦음* — readiness probe 통과면 *옛/새 둘 다 트래픽 받음*

→ **일반 워크로드에 가장 무난**. *진짜 안전한 배포*가 필요하면 Canary 또는 Blue/Green.

---

## 3. Recreate — *모두 죽이고 모두 새로*

```yaml
strategy:
  type: Recreate
```

### 동작
- 옛 Pod *모두 종료* → *새 Pod 시작*
- *downtime 있음* (옛 끝나고 새 ready되기 전 사이)

### 언제 쓰나
- *옛/새 동시 운영이 불가능*한 경우
  - DB schema 변경 (옛 코드와 새 코드가 다른 schema 기대)
  - *Singleton*이어야 하는 워크로드 (Cron-like 작업)
  - *기존 자원 (예: 외부 lock)이 한 instance만 쥘 수 있음*
- *짧은 downtime이 허용*되고 *복잡성 회피* 원할 때

---

## 4. Blue/Green — *별도 환경 + 한 번에 전환*

### 컨셉
```
환경 A (Blue, current prod)     환경 B (Green, new)
    Pods v1.0                       Pods v2.0
        ▲                                ▲
        │                                │
        └─── Service (selector=blue) ────┘
                     │
                     ▼
              user traffic
```

### 흐름
1. Blue (현 prod) 운영 중. *모든 트래픽 Blue*.
2. Green (새 버전) *별도로 띄움*. *트래픽 0* — *내부 테스트 가능*.
3. *Green 검증 완료* → *Service의 selector를 green으로 변경*.
4. *모든 트래픽 즉시 Green*. Blue는 *대기*.
5. 문제 발생 시 → *selector를 다시 blue로* (instant rollback).
6. 안정화 후 Blue 정리.

### 장점
- *즉시 전환·즉시 rollback*
- *옛/새 동시 운영 시간 거의 0* — 호환성 부담 적음
- *Green 환경에서 사전 smoke test 가능*

### 단점
- *자원 2배 필요* (Blue + Green 동시 운영)
- *DB는 공유* — schema 호환은 여전히 챙겨야
- *세션 처리 주의* — 전환 시점 진행 중 요청

### K8s 구현
- Deployment 2개 (blue·green) + Service의 `selector` 수동 변경
- 또는 *Argo Rollouts*의 BlueGreen strategy (자동화)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false   # 수동 promote
```

---

## 5. Canary — *소수 트래픽 → 점진 확대*

### 컨셉
```
                   ┌────── 90% ──→ Stable (v1.0) Pods
user ── LB / Ingress
                   └────── 10% ──→ Canary (v2.0) Pods

(점진: 10% → 25% → 50% → 75% → 100%)
```

### 흐름
1. v2.0 Pod 1~2개 만들고 *10% 트래픽* 받게
2. *메트릭·로그 모니터링* — error rate, latency 등이 *stable과 비교해 악화 없는가*
3. *통과* → 25%로 늘림
4. ... 점진 100%
5. *문제 발견* 즉시 → *canary 트래픽 0으로* (스케일 0 또는 weight 0)

### 장점
- *문제 감지 빠름* (실 트래픽으로)
- *영향 격리* — 10% 사용자만 영향
- *데이터 기반 promote 결정*

### 단점
- *복잡함* — LB·Ingress·Service Mesh 등의 *트래픽 분할 기능* 필요
- *메트릭·관측성 인프라*가 *전제*

### K8s에서 트래픽 분할 — 3가지 방법

**(A) 수동 — Pod 수 비율로**
- v1 Pod 9개, v2 Pod 1개 → Service round-robin이 *대략 10:1*
- 단점: *정확한 비율 X*, *세밀한 제어 X*

**(B) Ingress 또는 Service Mesh의 가중치 기반 분할**
- nginx Ingress: `nginx.ingress.kubernetes.io/canary-weight: "10"`
- AWS ALB: Target Group weight
- Istio: VirtualService weighted routing
- → *정확한 % 보장*

**(C) Argo Rollouts**
- *위 메커니즘들을 추상화* + *자동 promote 로직*
- *AnalysisTemplate*으로 *Prometheus·NewRelic 메트릭 기반 자동 판정*

---

## 6. Argo Rollouts — *K8s의 진보된 배포 controller*

K8s 내장 Deployment의 *상위 호환*. `Rollout` CRD가 *RollingUpdate / BlueGreen / Canary* 모두 지원.

### Canary 예시
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: myapp-canary   # canary용 Service
      stableService: myapp-stable
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress
      steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 25
      - pause: { duration: 5m }
      - setWeight: 50
      - pause: { duration: 5m }
      - setWeight: 100
```

### AnalysisTemplate — 자동 promote / abort
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 30s
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{job="myapp",status!~"5.."}[2m]))
          /
          sum(rate(http_requests_total{job="myapp"}[2m]))
```

→ *Prometheus 쿼리 결과로 자동 promote / abort*. *사람 개입 없이 안전한 canary*.

### Provider
- Prometheus, NewRelic, Datadog, CloudWatch, Wavefront, Web (HTTP), Kayenta (Spinnaker)
- *Job* — 임의 K8s Job 실행해서 결과 받음

---

## 7. Progressive Delivery — *Canary의 발전*

*Canary + Feature Flag + Observability* 결합:
- 코드 배포 ≠ feature 활성화
- *trunk-based*로 *모든 코드는 main에*, *feature flag로 일부만 켬*
- *user segment 별 노출* (beta 사용자, 특정 region, 1% 사용자)
- *문제 발견 → flag off로 즉시 비활성화* (deploy rollback 불필요)

### Feature Flag 도구
- LaunchDarkly, Unleash, GrowthBook (오픈소스), Split.io

### 패턴
```python
if feature_flags.is_enabled("new-checkout", user=user, default=False):
    return new_checkout_flow()
else:
    return old_checkout_flow()
```

→ *deploy와 release를 분리*. 안전성·실험 가능성 모두 ↑.

---

## 8. Shadow Deployment — *실 트래픽 *복사* 하여 새 버전에 실행*

```
user → LB → Stable (응답 반환)
              └─ 복사 → Canary (응답 무시)
```

- *Canary는 진짜 응답 안 함* — 사용자 영향 0
- *부하·로직 검증*에 유용
- *외부 부작용 (DB write 등) 주의* — *mocking* 또는 *read-only 검증*

→ *Istio mirroring*, 일부 ingress·LB가 지원.

---

## 9. Rollback — *모든 전략에서 *빠르고 안전하게**

### RollingUpdate
```bash
kubectl rollout undo deployment/myapp                  # 직전 revision
kubectl rollout undo deployment/myapp --to-revision=3  # 특정
```
- 옛 ReplicaSet이 *replicas=0*으로 남아 있어 가능
- `revisionHistoryLimit` 만큼만 보관

### Blue/Green
- *Service selector를 옛 환경으로* 즉시 전환

### Canary
- *canary weight를 0*으로 (또는 canary Deployment 삭제)

### *Rollback의 *데이터 측면* 주의*
- DB schema가 *forward-only 변경*이었으면 *코드 rollback 후에도 DB는 새 schema*
- → *backward-compatible migration* 패턴: *옛/새 코드 모두 동작*하는 schema 변경 절차 (expand → migrate → contract)

---

## 10. 자주 헷갈리는 것

### 10-1. *RollingUpdate가 *zero-downtime*인 *전제 조건**
- *readiness probe 정확히 설정* — 진짜 ready된 후만 트래픽 받음
- *graceful shutdown* — SIGTERM 받으면 *진행 중 요청 마치고* 종료
- `preStop` hook으로 *load balancer drain 시간* 확보 (보통 sleep 5~10초)

### 10-2. *Canary trapping*
- canary 시작 → 메트릭 *정상* → promote → *문제 발견 너무 늦음*
- 흔한 원인: *canary 트래픽이 *대표적이지 않음** (region·사용자 segment 편향)
- → *충분한 시간 + 대표 트래픽* 보장

### 10-3. *Blue/Green의 *DB 호환성**
- *Blue·Green이 같은 DB* — schema가 *둘 다 호환되야*
- *전환 시점 진행 중 요청*도 *어느 환경이 처리할지 모호*
- → expand→migrate→contract 패턴

### 10-4. *Deployment rollback이 *느림*
- RollingUpdate라서 *옛 RS로 점진 복귀*
- *빠른 rollback* 필요하면 Blue/Green 또는 Argo Rollouts abort

### 10-5. *Service의 트래픽 분배가 *정확한 비율이 아님**
- Service는 *Pod 수 기반 round-robin* (대략적)
- *세밀한 % 제어*는 Ingress·Service Mesh 가중치 기반 필요

### 10-6. *Argo Rollouts와 Deployment 혼동*
- `Rollout`은 *별도 CRD*. 기존 Deployment를 *Rollout으로 마이그레이션* 필요.
- *Workload reference* 패턴으로 *기존 Deployment를 점진 전환* 가능

---

## 🎤 면접 빈출 Q&A

### Q1. K8s 배포 전략 종류와 각각의 trade-off?
> **RollingUpdate** (기본): 점진 교체, downtime 0, 자원 적음, 옛/새 동시 운영 시기 있음. **Recreate**: 모두 죽이고 모두 새로, downtime 있음, 옛/새 호환 불가 시. **Blue/Green**: 별도 환경, 즉시 전환·rollback, 자원 2배. **Canary**: 트래픽 점진 확대, 실 데이터 기반 안전, 트래픽 분할 인프라 필요.

### Q2. Canary와 Blue/Green의 차이?
> **Blue/Green** = *환경 차원* 분리 (전체 새 환경 띄우고 한 번에 전환). **Canary** = *트래픽 차원* 분리 (소수 % 트래픽으로 시작 → 점진 확대). Canary는 *실 사용자 데이터로 검증*, Blue/Green은 *별도 환경에서 사전 검증 + 한 번에 전환*. Canary가 더 안전, Blue/Green이 더 단순·빠른 rollback.

### Q3. Argo Rollouts의 AnalysisTemplate 동작은?
> Canary 단계마다 *메트릭 쿼리 (Prometheus 등) → success/failure 판정*. `successCondition`이 true면 *다음 step으로*, false면 *abort + rollback*. *Prometheus·NewRelic·Datadog·CloudWatch·Web·Job* provider. → *사람 개입 없이* 메트릭 기반 *자동 promote*. *안전한 progressive delivery*.

### Q4. K8s에서 *트래픽 분할*은 어떻게? (정확한 % 보장)
> Service만 쓰면 *Pod 수 기반 round-robin* (대략적). *정확한 % 보장*은 (1) **Ingress controller의 가중치** — nginx-ingress canary annotation, AWS ALB target group weight. (2) **Service Mesh** — Istio VirtualService weighted routing, Linkerd traffic split. (3) **Argo Rollouts** — 위 메커니즘 추상화 + 자동 promote.

### Q5. RollingUpdate가 *zero-downtime*을 보장하려면?
> (1) **Readiness probe 정확히 설정** — 진짜 ready 후만 트래픽. (2) **Graceful shutdown** — SIGTERM 받고 *진행 중 요청 마치고* 종료. (3) **preStop hook** — `sleep 5~10` 으로 *LB drain 시간 확보* (Service Endpoint에서 제거되기 전에 종료되면 traffic loss). (4) **maxUnavailable / maxSurge** 설정.

### Q6. Feature flag와 progressive delivery의 관계는?
> *배포(deploy) 와 활성화(release) 분리*. *모든 코드는 main에 매일 머지* (trunk-based) → *feature flag로 일부만 활성화*. 장점: (1) *user segment 별 노출* (beta, 1%, region별), (2) *문제 시 flag off로 즉시 비활성화* (deploy rollback 불필요), (3) *Canary와 결합*해 *코드 + 활성화 모두 점진*.

### Q7. *DB schema 변경이 있는 배포*의 안전 패턴?
> **Expand → Migrate → Contract**. (1) **Expand**: 새 컬럼·테이블 *추가만* (옛 코드 영향 X). (2) **Migrate**: *데이터 이전 + 새 코드 배포* (둘 다 동작하는 dual-write 시기). (3) **Contract**: 옛 컬럼·테이블 *제거*. → *코드 rollback 시에도 DB는 호환*. *forward-only migration*은 *rollback 불가능* — 피해야.

---

## 🔗 Cross-reference

- **K8s Deployment / RollingUpdate** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **Probe 설정** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **Istio·Service Mesh** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md), 별도 mesh 학습
- **Argo Rollouts CRD가 controller pattern의 인스턴스** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md)
- **Prometheus 쿼리 / AnalysisTemplate** → `observability-basics/` (예정)
- **CI에서 image push → CD trigger** → [02-github-actions.md](02-github-actions.md), [03-argocd.md](03-argocd.md)

---

## 📝 3줄 요약

1. *위험 분산 차원*: 시간(Rolling), 환경(Blue/Green), 트래픽(Canary), 사용자(Feature flag). 일반은 Rolling, 안전 중요는 Canary, 즉시 rollback 중요는 Blue/Green.
2. *Argo Rollouts*가 K8s 표준 Deployment의 *상위 호환* — Canary/BlueGreen + AnalysisTemplate으로 *Prometheus 메트릭 기반 자동 promote/abort*. Progressive Delivery의 K8s native.
3. *DB schema는 expand→migrate→contract 패턴*. *Service의 트래픽 분배는 round-robin* — 정확한 % 보장은 Ingress 가중치 / Service Mesh / Argo Rollouts.

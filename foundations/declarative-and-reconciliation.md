# Declarative & Reconciliation — 모던 인프라의 메타 패턴

> k8s controller, terraform plan/apply, ansible idempotency, argocd sync, helm release, prometheus rule —
> 다 다른 이름이지만 *같은 패턴*이다. 이 문서가 그 패턴 자체.

이 문서는 `foundations/`의 **허브**다. 나머지 5개 foundation과 11개 챕터에서 마주칠 "원리"의 절반 이상이 여기서 갈라져 나온다.

---

## 1. 명령형 vs 선언형 — 왜 인프라는 선언형으로 갔나

### 정의

- **명령형 (imperative)**: *어떻게* 의 시퀀스를 적는다. "디렉토리 만들고, 패키지 설치하고, 서비스 시작해라."
- **선언형 (declarative)**: *무엇* 만 적는다. "이 서비스는 실행 중이어야 한다." 어떻게는 시스템이 정한다.

### 명령형이 인프라에서 깨지는 이유

인프라는 다음 세 가지 압력 아래 있다:

1. **부분 실패** — 10단계 중 7단계에서 실패하면 8~10은 *상태 미정*. 재실행하면 1~7이 다시 부작용을 일으킨다.
2. **동시 변경** — 사람·다른 도구·장애 복구가 동시에 시스템을 건드린다. "현재 상태"가 명령 작성 시점과 다르다.
3. **재시도** — 네트워크 일시 장애, 노드 재시작 등으로 *같은 작업을 여러 번* 실행해야 한다.

명령형 시퀀스는 위 셋 모두에 대해 *합성 불가능* 하다. "스크립트를 두 번 돌리면?"이라는 질문에 매번 손으로 답해야 한다.

선언형은 *목적 상태* 만 다루기 때문에 위 세 압력이 사라진다. 시스템이 알아서 *지금 상태* 와 *목적 상태* 의 차이만큼만 행동한다.

> 📚 참고: Borg 논문 (Verma et al., 2015) — Google이 워크로드를 선언형 모델로 다룬 첫 사례. k8s의 정신적 조상.

### 함수형과의 비유

선언형 ↔ 함수형 (referential transparency)은 형제 관계다.
- 함수형: "이 값은 이 식과 같다" — 평가 순서를 신경 쓰지 않아도 된다.
- 선언형 인프라: "이 시스템은 이 상태와 같다" — 실행 순서를 신경 쓰지 않아도 된다.

둘 다 *순서 의존성 제거* 가 본질이며, 그 대가로 **멱등성** (§4)이 요구된다.

---

## 2. Desired vs Observed — diff 계산이 핵심

모든 선언형 시스템은 두 가지 상태를 갖는다:

- **Desired state**: 사용자가 선언한 *목적*
- **Observed state**: 시스템이 관측한 *현재*

핵심 작업은 두 상태의 **차이 계산 (diff)** 과 그 차이만큼의 **행동 (apply)**.

### 도구별 매핑

| 도구 | Desired 저장소 | Observed 출처 | Diff 시점 |
|---|---|---|---|
| **k8s** | etcd의 object `.spec` | kubelet/컴포넌트의 `.status` report | controller가 watch event 받을 때 |
| **terraform** | `.tf` 파일 | `tfstate` + provider의 read API | `terraform plan` |
| **ansible** | playbook | 모듈의 check 단계 (예: `stat` → 파일 존재 확인) | task 실행 직전 |
| **argocd** | git repo의 manifest | live cluster에서 가져온 manifest | sync loop (기본 3분, watch 가능) |
| **helm** | chart의 rendered manifest | k8s API의 release 상태 | `helm upgrade` / `helm diff` |
| **prometheus alert** | rule 파일의 PromQL expression | TSDB의 현재 metric 값 | evaluation interval (기본 1분) |

### Diff 종류

| 종류 | 의미 | 예시 |
|---|---|---|
| **Structural diff** | 필드 단위 비교 | k8s의 `kubectl diff`, helm diff |
| **Semantic diff** | 의미 단위 비교 (순서 무시, default 적용 후 비교) | terraform의 `plan` (provider별로 normalize) |
| **State-machine diff** | 전이 가능 여부 판단 | k8s의 일부 immutable field (예: PVC `storageClassName`) |

### 왜 *observed 읽기* 가 어려운가

- 시스템은 분산되어 있다 → observed는 *여러 곳에 흩어진* 진실의 합성
- 외부 변경 (사람·다른 도구)이 언제든 일어난다 → §5 drift
- 일부 속성은 *읽을 방법이 없다* (예: 일부 cloud provider 리소스의 내부 상태)

terraform이 `refresh`를 별도 단계로 두는 이유, k8s controller마다 *informer cache* 를 두는 이유 모두 여기서 나온다.

---

## 3. Control Loop — 의사코드 한 덩어리

선언형 시스템의 심장:

```
for {
    desired  := readSpec()          // 사용자가 선언한 목적
    observed := readWorld()         // 시스템의 현재 상태
    actions  := diff(desired, observed)
    apply(actions)                  // 차이만큼만 행동
    waitForEvent() OR sleep(interval)
}
```

이 10줄이 다음 모두의 본질이다:

| 도구 | tick 트리거 | 비고 |
|---|---|---|
| **k8s controller** | watch event (event-driven) | `Reconcile(key)` 함수가 위 loop 한 tick |
| **argocd application-controller** | sync loop (3min default) + git webhook | desired=git, observed=cluster |
| **ansible** | 사람의 `ansible-playbook` 명령 | playbook 1회 실행 = tick 1회 |
| **terraform** | 사람의 `apply` | `plan` = diff 계산, `apply` = apply 단계 |
| **helm** | 사람의 `upgrade` | release 1회 |
| **prometheus alert** | evaluation interval | `for` 절이 *연속 tick에서 조건 유지* 를 요구 |

### terraform은 *왜* 연속 loop이 아닌가

terraform은 의도적으로 *사람이 loop을 돌린다*. 이유:

1. **인프라 변경은 비싸고 위험** — 자동 sync는 실수를 증폭시킨다. (EC2 인스턴스 100개를 잘못 destroy하면 복구가 어렵다.)
2. **`plan` 출력을 사람이 검토** 하는 단계가 필요하다. 변경 영향이 즉시 명백하지 않다.
3. **Cloud provider API rate limit** — 무한 loop이 비용 폭탄이 된다.

argocd는 *반대* 입장 — 자동 sync + self-heal로 사람을 빼낸다. 두 도구는 **같은 패턴, 다른 정책**.

| 정책 | 도구 | 트레이드오프 |
|---|---|---|
| 사람이 tick 돌림 | terraform, ansible, helm | 안전, 검토 가능, 느림 |
| 시스템이 자동 tick | k8s controller, argocd (auto-sync), prometheus | 빠른 수렴, 자동 복구, 폭주 위험 |

→ 이 control loop을 Go의 goroutine·channel·context로 구현하는 방식은 [[go-for-infra]] §controller pattern 참조.

### Event-driven vs Polling

위 의사코드의 `waitForEvent() OR sleep(interval)`은 *구현 선택*:

- **Polling**: 단순, idempotent, 그러나 reaction time이 interval에 묶임. terraform·ansible이 이 모델.
- **Event-driven**: 빠른 reaction, 그러나 event 유실 시 stale 가능 → k8s는 **둘 다** 쓴다 (watch + periodic resync). argocd도 webhook + 3분 polling.

"watch만 쓰면 안 되나?" → 안 된다. watch 연결이 끊겼다 재연결되는 사이 event가 유실될 수 있어서, *주기적 resync* 가 안전망.

---

## 4. Idempotency — 선언형의 필수조건

### 정의

함수 `f`가 멱등하다 = `f(f(x)) = f(x)`.
인프라 맥락: **같은 작업을 여러 번 실행해도 결과 상태가 같다**.

### 왜 선언형은 멱등성이 *필수* 인가

§3의 loop은 매 tick마다 *같은 desired state* 를 본다. observed가 이미 desired와 같다면 `actions = ∅` 여야 한다.

- 멱등하지 않다면: loop이 매 tick마다 *부작용을 일으킨다* → 시스템이 시끄럽고 비싸지고 불안정해진다.
- 멱등하다면: 수렴 후엔 loop이 *조용히* 돈다.

이것이 선언형 시스템이 사람에게 주는 핵심 가치 — **수렴 후 침묵 (silence at steady state)**.

### 도구별 멱등성 보장 위치

- **ansible**: *모듈 레벨*. 각 모듈이 내부에서 `check → change → execute` 를 분리. 예: `apt` 모듈이 패키지 이미 설치되어 있으면 `changed=false` 로 끝낸다. 책임이 *모듈 작성자* 에게 있다.
- **terraform**: *provider 레벨*. provider가 리소스마다 read/diff/create/update/delete를 분리 구현한다. terraform core는 diff에 따라 어느 함수를 호출할지만 정한다.
- **k8s**: *controller 레벨*. 각 controller가 `Reconcile()` 안에서 멱등성을 책임진다. `Reconcile()`은 *같은 key로 100번 불려도 같은 결과* 여야 한다는 게 controller-runtime의 계약.
- **helm**: *manifest 레벨*. 같은 values + 같은 chart = 같은 manifest = 같은 cluster 상태. 단, hook은 멱등하지 않을 수 있어 주의.

### Shell script가 기본적으로 멱등하지 않은 이유

```bash
mkdir foo              # 두 번째 실행: 에러
rm /tmp/state.json     # 두 번째 실행: 에러
echo "x" >> /etc/conf  # 두 번째 실행: x가 두 줄
```

멱등성을 *언어가 보장하지 않는다*. 그래서 ansible의 `shell:` / `command:` 모듈은 공식 문서가 "거의 항상 *다른* 모듈을 먼저 찾아라"라고 명시한다.

→ shell script에서 멱등성을 *수동으로* 만드는 패턴 (`mkdir -p`, `if ! grep -q ... >> file`, lock 파일 등)은 [[shell-scripting]] §idempotency 참조.

### 멱등성의 미묘한 함정

- **시간 의존**: `apt install nginx` 는 멱등하지만 *최신 버전* 으로 업그레이드되면 결과가 달라진다. → version pin이 필수.
- **외부 자원**: API 호출은 *서버 쪽 멱등성* 에 의존. HTTP `PUT`은 idempotent 명세지만 *실제 구현*은 그렇지 않을 수 있다. 그래서 RFC 7231은 idempotent를 "의도"로 정의.
- **순서**: 멱등 작업 A·B를 `[A,B]` 와 `[B,A]` 로 실행하면 *결과가 다를 수 있다* (commutativity는 별개 속성).

---

## 5. Drift — 현실의 균열

### 정의

**Drift**: observed state가 *외부 요인* 으로 desired state에서 벗어난 상태.

### 원인

- 사람이 손으로 변경 (`kubectl edit`, 콘솔에서 클릭)
- 다른 도구의 변경 (autoscaler가 replica 수를 변경, security agent가 securityContext 수정)
- 부분 장애 후 복구가 desired와 다른 상태로 끝남
- 시간에 따른 변화 (cert 만료, token 만료) — 사람이 안 건드려도 drift 발생

### 도구별 대응

| 도구 | Drift 탐지 | Drift 복구 |
|---|---|---|
| **k8s** | controller가 watch로 항상 감시 | **자동 복구 (그게 곧 존재 이유)** |
| **terraform** | `terraform plan` 시 `tfstate refresh` → drift 표시 | `apply` 로 사람이 결정 |
| **argocd** | `OutOfSync` 상태로 표시 | `self-heal: true` 면 자동, 아니면 사람 |
| **ansible** | `--check` 모드 (dry run) | 다음 실행이 곧 복구 |
| **prometheus rule** | 매 evaluation마다 재계산 | (alert는 drift 자체가 정상 — 알람으로 사람에게 위임) |

### Drift는 영원히 없어지지 않는다

선언형 시스템은 *closed world* 를 가정한다 — "내가 모르는 변경은 없다". 현실은 *open world*. 따라서:

1. drift 탐지·복구는 *연속적 활동* 이다 (한 번 해결 → 영원히 해결 X)
2. 모든 변경을 *반드시* desired state 통해서 하도록 강제하는 **문화**가 필요하다 → GitOps의 본질
3. IaC repo가 *git* 이어야 하는 이유 — desired state의 **유일한 진실 공급원 (single source of truth)** 을 확보하기 위해

### Self-heal의 두 얼굴

argocd `selfHeal: true`는 강력하지만 위험하다:
- ✅ 누가 손으로 만진 걸 자동 복구 → 의도한 상태 유지
- ❌ 정당한 hotfix (긴급하게 `kubectl edit` 한 변경)도 *지워버린다*
- ❌ HPA·VPA가 만진 replica/resource를 *되돌려서* 가용성을 깬다 (→ `ignoreDifferences` 필요)

자동 복구의 *경계* 를 어디에 그을지가 운영 정책의 핵심.

---

## 6. Cross-reference 지도

이 원리를 *실제 도구에서* 보고 싶다면:

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| Reconciler 실제 코드 구조 | `kubernetes-basics/` controller·CRD 토픽 |
| Plan/apply 모델, tfstate | `terraform-basics/` workflow 토픽 |
| Sync loop, self-heal, GitOps | `cicd-gitops-basics/` ArgoCD 토픽 |
| 모듈 레벨 멱등성 설계 | `ansible-basics/` idempotency 토픽 |
| Desired vs observed in alerting | `observability-basics/` PromQL·alert 토픽 |
| 선언형을 지탱하는 Go runtime | [[go-for-infra]] |
| 멱등성을 *수동으로* 만드는 shell 패턴 | [[shell-scripting]] |
| Desired state의 단일 출처 확보 | `cicd-gitops-basics/` GitOps 원칙 |
| 멱등성을 깨는 시간 요소 (cert 만료 등) | [[tls-and-pki]], [[time-and-clocks]] (예정) |

---

## 한 줄 요약

> **선언형** = 목적만 적기
> **Reconciliation** = 목적과 현실의 차이를 계속 메우기
> **Idempotency** = 그게 가능하게 하는 수학적 조건
> **Drift** = 그 가정을 깨는 현실의 압력

이 네 가지가 모던 인프라 도구를 *읽을 수 있게* 만든다. 도구가 바뀌어도 패턴은 안 바뀐다.

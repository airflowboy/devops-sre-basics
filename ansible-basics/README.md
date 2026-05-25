# Ansible — 동작 원리 중심

> *"Ansible은 agentless라는데 어떻게 원격 서버를 관리하나요?"*
> *"멱등성은 어떻게 보장되나요?"*
> *"같은 playbook을 100번 실행해도 안전한가요?"*

---

## 🎯 한 문장 큰 그림

> **Ansible = *agentless* (대상 서버에 별도 daemon 없음) + *push-based* (control node에서 SSH로 모듈 전송·실행) + *declarative + idempotent* (원하는 상태를 명세하면 *현재 상태 검사 후 변경 시에만 적용*)인 *구성 관리·자동화 도구*.**

→ Chef/Puppet (agent + pull) 과의 대비. *서버에 뭘 설치 안 해도 됨* (SSH + Python만 있으면).

---

## 🧩 한 페이지 요약

```
   ┌────────────────────────────────────┐
   │  Control Node (laptop / CI / AWX)   │
   │  - Ansible 설치                      │
   │  - Inventory (대상 서버 목록)         │
   │  - Playbook (할 일 명세, YAML)        │
   │  - Roles / Collections (재사용 단위)  │
   └─────────────┬──────────────────────┘
                 │ SSH (+ Python interpreter on target)
                 ▼
   ┌────────────────────────────────────┐
   │  Target Hosts (서버들)                │
   │  - SSH 접근만 있으면 됨               │
   │  - Python 3 (보통 기본 설치됨)        │
   │  - agent 불필요                       │
   └────────────────────────────────────┘
```

흐름:
1. Control node가 *playbook 파싱* → *각 task에 해당하는 module 결정*
2. 그 module 코드를 *SSH로 target에 전송*
3. Target의 Python이 module 실행 → *결과 JSON 반환*
4. Control node가 *결과 수집·집계*

---

## 🌪 자주 헷갈리는 7가지

### 1. *Ansible은 agentless* — 진짜?
- target에 *daemon 설치 X* (Chef/Puppet과 차이)
- 단 *Python 3은 필요* (대부분 기본 설치됨)
- SSH 연결만 가능하면 작동
- → *어떤 환경에도 빠르게 시작* 가능

### 2. *Push vs Pull*
- Ansible 기본 = **push** (control node → target)
- *ansible-pull*로 pull도 가능 (각 노드가 git에서 playbook fetch 후 self-apply) — 드물지만 *cattle 환경*에 적합

### 3. *멱등성 (Idempotency)*은 *module이 보장*
- Ansible 자체가 보장 X
- 각 *module이 *현재 상태 검사 후 변경 시에만 적용**
- → *잘 짠 module은 100번 실행해도 같은 결과*
- 단 *shell/command module은 멱등성 안 보장* — 직접 신경

### 4. *Playbook은 *순서대로* 실행*
- *task는 명세된 순서대로*
- *호스트는 *병렬* (forks)
- → 한 호스트에서 task 1 → task 2 → ... 순서. 여러 호스트는 *동시에* task 1 → 동시에 task 2 → ...

### 5. *변수 우선순위 22개 layer*
- Ansible의 *악명 높은* 부분
- 인벤토리 vars, group_vars, host_vars, play vars, role defaults, extra-vars 등
- *항상 동일 결과 보장 못 하면 디버깅 지옥*
- → [03-variables-and-templating.md](03-variables-and-templating.md)

### 6. *Ansible은 *순서 보장*하지만 *상태 보존 안 함**
- *현재 적용 상태를 어딘가에 저장 X* (Terraform과 차이)
- 다음 실행 시 *target의 현재 상태*를 *다시 검사*
- → *target 자체가 진실*. *재현성은 *playbook의 멱등성*에 의존*.

### 7. *Module vs Plugin vs Collection*
- **Module**: 한 task의 *실행 단위* (`apt`, `copy`, `service`, ...)
- **Plugin**: Ansible 자체 *확장* (connection, lookup, filter, callback, ...)
- **Collection**: *module + plugin + role*의 *배포 단위* (Ansible 2.10+)

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-inventory-and-modules.md](01-inventory-and-modules.md) | static/dynamic inventory, group, host_vars, 자주 쓰는 module 30개 |
| [02-playbook-and-tasks.md](02-playbook-and-tasks.md) | play / task / handler / loop / condition / block, idempotency |
| [03-variables-and-templating.md](03-variables-and-templating.md) | 변수 우선순위 22단계, Jinja2, vault, lookup, register / set_fact |
| [04-roles-and-collections.md](04-roles-and-collections.md) | role 구조, dependency, Galaxy, collection 작성·배포 |
| [05-execution-and-best-practices.md](05-execution-and-best-practices.md) | ansible-pull, AWX/Tower, error handling, debugging, async, check mode |

---

## 🔗 Cross-reference

- **Linux 베이스 (systemd·package)** → [`linux-basics/`](../linux-basics/) (Ansible의 *주요 대상*)
- **K8s 자원 생성**: Ansible *k8s module*로 가능하나 *Helm/kubectl이 더 흔함* → [`kubernetes-basics/`](../kubernetes-basics/)
- **Terraform과 비교** → `terraform-basics/` (예정). Terraform = *infra provisioning*, Ansible = *configuration management*. *Terraform으로 EC2 띄우고 Ansible로 설정* 패턴 흔함.
- **CI에서 ansible-lint·playbook 검증** → [`cicd-gitops-basics/01`](../cicd-gitops-basics/01-ci-fundamentals.md)

---

## 🎤 면접 빈출

| 질문 | 답이 있는 파일 |
|---|---|
| Ansible의 *agentless* 가 무슨 의미? | README + [01](01-inventory-and-modules.md) |
| 멱등성은 어떻게 보장? `shell` module은? | [02](02-playbook-and-tasks.md) |
| Inventory의 group / host_vars / group_vars 우선순위? | [01](01-inventory-and-modules.md) + [03](03-variables-and-templating.md) |
| Ansible vs Terraform — 차이와 같이 쓰는 패턴? | [05](05-execution-and-best-practices.md) |
| Role 구조와 dependency? | [04](04-roles-and-collections.md) |
| Secret을 playbook에 어떻게 안전하게? | [03](03-variables-and-templating.md) (Vault) |
| 대규모 인프라 자동화 시 *AWX/Tower* 가 필요한 이유? | [05](05-execution-and-best-practices.md) |

---

## 📐 학습 순서 권장

1. **01 inventory-and-modules** — *대상·도구 기본*
2. **02 playbook-and-tasks** — *playbook 작성*
3. **03 variables-and-templating** — *변수 (가장 복잡)*
4. **04 roles-and-collections** — *재사용*
5. **05 execution-and-best-practices** — *운영 패턴·디버깅*

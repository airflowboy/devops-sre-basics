# 05 — Execution & Best Practices: ansible-pull / AWX / debugging / 안전 패턴

> **공식 근거:** [Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html), [Async Tasks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_async.html), [AWX/Tower](https://www.ansible.com/products/awx-project)

---

## 🎯 한 문장

> **운영의 핵심: *디버깅 친화 (lint, check, diff, verbose)*, *안전 (serial, max_fail_percentage, --check)*, *재현성 (vault, requirements.yml, lock)*, *대규모 (AWX/Tower, ansible-pull)*. Ansible vs Terraform — *configuration management* vs *infra provisioning* 의 역할 분담.**

---

## 1. *Push (기본) vs Pull (ansible-pull)*

### Push (기본)
```
Control node (CI / 운영자) → SSH → Target servers
                              push
```

- *control node에서 trigger* → target에 *push*
- *중앙 제어 좋음*
- 단점: target 수 많으면 *push 부담 + 동시성 한도*

### Pull — `ansible-pull`
```
Target server ← (git pull) ← Repository
       │
       └→ ansible-pull 로 self-apply
```

- 각 target이 *git에서 playbook fetch* → *자신에게 apply*
- *cattle 환경* (Auto Scaling Group, K8s node) 에 적합
- *대규모* — 한 push로 1000+ host 비현실적

### 사용
```bash
# target에서 (보통 cron으로)
ansible-pull -U https://github.com/my-org/playbooks.git -i hosts.yml site.yml

# crontab
*/15 * * * * /usr/local/bin/ansible-pull -U ... -i hosts.yml site.yml 2>&1 | logger -t ansible-pull
```

→ 부팅 시 cloud-init으로 *처음 install + 정기 pull* 패턴.

---

## 2. AWX / Ansible Tower / Ansible Automation Platform

### 발상
- ansible-playbook CLI를 *팀에서 공유*하기 어려움
- *권한·실행 기록·schedule·UI*가 필요한 단계
- **AWX** (오픈소스) / **Tower** (Red Hat 유료 → AAP)

### 주요 기능
- **Job Template** — *playbook + inventory + credentials + extras*의 명명된 실행 단위
- **Workflow** — 여러 Job Template을 *DAG*로 묶기
- **Schedule** — cron-like
- **RBAC** — 누가 어느 inventory/job 접근 가능
- **Credentials Vault** — SSH key·cloud key·vault password 등 *중앙 관리*
- **Activity Stream** — *모든 실행·변경 audit*
- **Survey** — 실행 시 *사용자에게 변수 묻기*
- **Notification** — Slack/Email/Webhook
- **REST API** — *CI/CD 통합*
- **Execution Environment** — *컨테이너 안에서 playbook 실행* (의존성 격리)

### 운영 모범
- *팀에서 ansible 공유*하면 *반드시 AWX 정도*는 둠
- 개인 CLI 실행은 *audit·재현 어려움*
- *Schedule + Notification* 으로 *정기 자동화*

---

## 3. Check Mode + Diff — *dry-run + 변경 보기*

```bash
# 변경 없이 *시뮬레이션*
ansible-playbook playbook.yml --check

# 변경분도 보기 (diff)
ansible-playbook playbook.yml --check --diff

# 일부 task만 check (또는 일부만 실제 실행)
- name: ...
  template: ...
  check_mode: yes              # 항상 check만 (실제 실행 X)

- name: ...
  shell: ...
  check_mode: no               # check mode에서도 실제 실행 (read-only 명령)
```

### --check 의 한계
- *대부분 module 지원*, 일부 module은 *check mode 안 됨* (shell·command 등)
- *조건부 task* 가 *이전 task의 결과에 의존*하는 경우 — check mode에선 정확하지 않을 수 있음

→ check mode + diff가 *PR review의 핵심 도구*. *prod 적용 전 반드시*.

---

## 4. Verbose & Debug

### Verbose level
```bash
ansible-playbook playbook.yml -v       # 기본 verbose
ansible-playbook playbook.yml -vv      # 더
ansible-playbook playbook.yml -vvv     # 더 (SSH debug)
ansible-playbook playbook.yml -vvvv    # 최대
```

- `-vvv` 부터는 *SSH 명령·module 호출 내용* 그대로 보임 → *디버깅에 핵심*

### debug module
```yaml
- name: Show variable
  debug:
    var: my_variable
  
- name: Show message
  debug:
    msg: "Current state: {{ result.stdout }}"

- name: Show conditional info
  debug:
    msg: "Memory is low"
  when: ansible_memtotal_mb < 1024
```

### `assert` module — *전제 조건 검증*
```yaml
- name: Assert preconditions
  assert:
    that:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_version is version('22.04', '>=')
    fail_msg: "This playbook requires Ubuntu 22.04+"
    success_msg: "Preconditions met"
```

### `fail` — *명시적 실패*
```yaml
- name: Fail on bad input
  fail:
    msg: "app_version must be defined"
  when: app_version is not defined
```

### Pause — *수동 검토*
```yaml
- name: Manual check before continuing
  pause:
    prompt: "Press Enter to continue, or Ctrl-C to abort"
    seconds: 10
```

---

## 5. Async — *오래 걸리는 task 비동기*

### 문제
- `apt upgrade` 같은 task가 *10+ 분 소요*
- 그동안 *SSH 연결 timeout*?

### 해결: async + poll
```yaml
- name: Long task in background
  shell: /usr/local/bin/big-data-migration.sh
  async: 3600                  # 최대 1시간
  poll: 30                     # 30초마다 상태 확인
  register: migration_result

- name: Check final status
  debug:
    var: migration_result
```

### Fire-and-forget — poll 0
```yaml
- name: Fire and forget
  shell: /usr/local/bin/long-task.sh
  async: 3600
  poll: 0                      # 결과 안 기다림
  register: job_handle

- name: ... (다른 일)

- name: Check later
  async_status:
    jid: "{{ job_handle.ansible_job_id }}"
  register: result
  until: result.finished
  retries: 30
  delay: 30
```

---

## 6. Error Handling

### ignore_errors
```yaml
- name: Try something
  shell: maybe-fails.sh
  ignore_errors: yes
  register: result

- name: Continue based on result
  debug:
    msg: "It {{ 'worked' if result.rc == 0 else 'failed' }}"
```

### failed_when
```yaml
- name: Custom fail condition
  shell: check-status.sh
  register: result
  failed_when:
  - result.rc != 0
  - "'ignorable error' not in result.stderr"
```

### block / rescue / always
[02-playbook-and-tasks.md](02-playbook-and-tasks.md) Section 7 참조.

### `any_errors_fatal` 와 `max_fail_percentage`
```yaml
- hosts: webservers
  any_errors_fatal: true       # *단일 host fail = 전체 중단*
  # 또는
  max_fail_percentage: 10      # 10% 이상 fail = 중단
```

→ rolling deploy 안전성의 핵심.

---

## 7. ansible-lint — *정적 분석*

```bash
ansible-lint playbook.yml
ansible-lint roles/nginx
```

### 자주 잡히는 rule
- 모든 task에 `name:` (가독성)
- shell/command 대신 *전용 module* 사용
- `latest` 대신 *명시 버전*
- 변수 이름 컨벤션
- 권한 설정 (`mode:` 명시)
- handler 이름 컨벤션

### CI 통합
```yaml
- uses: ansible/ansible-lint@main
  with:
    args: "playbook.yml"
```

→ *PR 단위로 lint*. 코드 품질 게이트.

---

## 8. Best Practices — *운영 안전*

### 1. *항상 `--check --diff` 먼저*
```bash
ansible-playbook playbook.yml --check --diff -i prod
# 변경 사항 review 후
ansible-playbook playbook.yml -i prod
```

### 2. *Serial + max_fail_percentage*
```yaml
- hosts: webservers
  serial: 1
  max_fail_percentage: 0
  pre_tasks:
  - name: Remove from LB
    ...
  tasks:
  - ...
  post_tasks:
  - name: Add back to LB
    ...
```

→ *rolling update + early stop*.

### 3. *Secret은 vault 또는 외부 매니저*
- ansible-vault (소규모)
- HashiCorp Vault + lookup plugin (대규모)
- AWS Secrets Manager + `amazon.aws.aws_secret` lookup

### 4. *Inventory는 git*
- *환경별 directory* (`inventories/{dev,stg,prod}/`)
- group_vars/host_vars 함께
- *vault 파일도 git에* (암호화돼 있으니 안전)

### 5. *requirements.yml로 의존성 lock*
- role / collection 모두
- *CI에서 install* — 재현 보장

### 6. *Idempotency 테스트*
- 같은 playbook *2번 실행* → 두 번째는 *모두 ok (changed=0)* 여야
- CI에서 *자동 검증*

### 7. *Tags 적극*
- 큰 playbook의 *일부만 실행* 가능 (`--tags`, `--skip-tags`)
- 빠른 디버깅·반복

### 8. *Role로 재사용*
- *같은 logic 두 번 적으면 role로*
- 사내 *공통 role 저장소* (Galaxy / git)

### 9. *AWX/Tower로 audit*
- 누가 언제 무엇을 적용 — *기록 필수* (컴플라이언스)

### 10. *Ansible vs Terraform 역할 분담*

| Terraform | Ansible |
|---|---|
| **Infra provisioning** — EC2·VPC·RDS 생성 | **Configuration management** — 그 EC2에 nginx install·설정 |
| *State 파일에 현재 상태 저장* | *State 안 저장 — target이 진실* |
| Cloud API 호출 | SSH 통해 OS 명령 |
| `terraform plan/apply` | `ansible-playbook` |

흔한 패턴:
1. *Terraform으로 EC2 + ALB 생성*
2. *Terraform output → Ansible inventory*
3. *Ansible로 EC2 OS 설정·앱 배포*

→ `terraform-basics/` (예정) 에서 더.

---

## 9. CI/CD 통합

### 흐름
```
git push → CI:
  1. ansible-lint
  2. ansible-galaxy install -r requirements.yml
  3. ansible-playbook --syntax-check
  4. ansible-playbook --check --diff -i dev    (PR)
  5. (PR 머지 후) ansible-playbook -i dev
  6. (manual) ansible-playbook -i stg
  7. (manual + approval) ansible-playbook -i prod
```

### Molecule — *role 테스트 framework*
- 격리된 *docker / podman / vagrant / vm* 환경에서 role 실행·검증
- *idempotency 자동 테스트*
- *여러 OS에서 테스트* (matrix)
- CI 통합 표준

```bash
molecule init role my-role
molecule create        # 인스턴스 생성
molecule converge      # role 적용
molecule idempotence   # 멱등성 검증
molecule verify        # 검증
molecule destroy
molecule test          # 전체 사이클
```

---

## 10. 자주 헷갈리는 것

### 10-1. *--check 가 *모든 module에서 작동 안 함**
- `shell`, `command`, 일부 외부 module은 *check mode skip*
- 그래서 *--check 결과가 실제와 다를 수 있음*
- read-only shell 명령은 `check_mode: no`로 *항상 실행*

### 10-2. *Inventory와 -e 의 차이*
- inventory vars = *환경별 영구 설정*
- `-e` extra vars = *일회성 override*, 가장 우선
- 운영에선 *inventory가 진실*, `-e` 는 *임시 디버깅*

### 10-3. *ansible-vault password 관리 부담*
- 누가 password 알아? 어디 저장?
- 작은 팀: 1Password / Bitwarden
- 큰 팀: *외부 secret manager로 전환* 권장

### 10-4. *Self-hosted runner에서 ansible 실행 시 *SSH key 보안***
- runner host에 SSH key 영구 보관 → *유출 시 모든 target 위험*
- *ephemeral runner* + *credentials store에서 매번 fetch* 권장

### 10-5. *Ansible로 K8s 자원 관리* — 비효율
- `kubernetes.core.k8s` module 가능하나 *대부분 Helm/kubectl이 더 빠름*
- Ansible은 *node OS 설정·k8s 클러스터 부트스트랩에 강점*
- *클러스터 *위*의 자원은 Helm/ArgoCD에 맡기기*

---

## 🎤 면접 빈출 Q&A

### Q1. Ansible과 Terraform 차이? 같이 쓰는 패턴?
> **Terraform**: *infra provisioning* — EC2/VPC/RDS 생성·관리. State 파일에 *현재 상태 저장*. Cloud API 호출. **Ansible**: *configuration management* — 그 EC2에 *OS 설정·앱 install*. State 안 저장 (target이 진실). SSH로. 패턴: *Terraform으로 인프라 생성 → output을 ansible inventory로 → Ansible로 OS 설정·앱 배포*.

### Q2. Ansible 디버깅 도구는?
> (1) **`-vvv`** — SSH 명령·module 호출 verbose. (2) **`--check --diff`** — dry-run + 변경분. (3) **`debug` module** — 변수·메시지 출력. (4) **`assert` / `fail`** — 명시적 전제 조건 + 실패. (5) **`pause`** — 수동 검토 지점. (6) **`ansible-lint`** — 정적 분석. (7) **Molecule** — role 단위 격리 테스트.

### Q3. *오래 걸리는 task* (1시간+) 어떻게?
> `async: 3600 poll: 30` — *백그라운드 + 30초마다 상태 확인*. SSH timeout 회피. Fire-and-forget이면 `poll: 0` + 나중에 `async_status` 로 확인. 매우 긴 task는 *separate playbook* + AWX schedule로 분리 권장.

### Q4. 안전한 rolling deploy 패턴?
> `serial: 1` (또는 N%) + `max_fail_percentage: 0` + `pre_tasks` (LB에서 빼기) + `tasks` (실제 update) + `post_tasks` (LB에 다시 추가). 한 host fail = 전체 중단 → 사고 *전파 차단*. Health check + serial 조합이 *blue/green 비슷한 안전성*.

### Q5. AWX/Tower가 *왜 필요*한가?
> *팀에서 ansible 공유 시 필수 단계*. CLI 실행은 *audit·재현 어려움·권한 관리 X·schedule 없음*. AWX: **Job Template** (실행 단위 명명), **Workflow** (DAG), **Schedule** (cron), **RBAC**, **Credentials Vault** (중앙 관리), **Activity Stream** (audit), **Notification**, **REST API**, **Execution Environment** (컨테이너 격리). 컴플라이언스·운영 가시성에 필수.

### Q6. Idempotency를 *어떻게 검증*?
> 같은 playbook *2번 실행* → 두 번째는 *모두 ok, changed=0* 이어야. CI에 *자동 검증*:
> ```yaml
> - ansible-playbook playbook.yml
> - ansible-playbook playbook.yml | tee out.log
> - test "$(grep 'changed=' out.log | grep -v 'changed=0' | wc -l)" -eq 0
> ```
> **Molecule** 의 `molecule idempotence` 명령이 *role 단위로 자동*.

### Q7. ansible-pull은 *언제 push보다 좋은가*?
> *대규모 cattle 환경* (Auto Scaling Group, 수천 노드). *push로 모두 trigger 비현실적*. 각 node가 *git에서 fetch + self-apply* → 부담 분산. *cloud-init으로 부팅 시 install + cron으로 정기 sync*. 단 *중앙 audit 약함* — pull 실행 로그를 *별도 수집* 필요.

---

## 🔗 Cross-reference

- **playbook의 block/rescue, serial** → [02-playbook-and-tasks.md](02-playbook-and-tasks.md)
- **secret 관리** → [03-variables-and-templating.md](03-variables-and-templating.md)
- **role/collection 재사용** → [04-roles-and-collections.md](04-roles-and-collections.md)
- **CI/CD 통합** → [`cicd-gitops-basics/01`](../cicd-gitops-basics/01-ci-fundamentals.md), [`cicd-gitops-basics/02`](../cicd-gitops-basics/02-github-actions.md)
- **Terraform과 조합** → `terraform-basics/` (예정)
- **K8s 자원 → Helm/ArgoCD 더 효율** → [`helm-basics/`](../helm-basics/), [`cicd-gitops-basics/03`](../cicd-gitops-basics/03-argocd.md)

---

## 📝 3줄 요약

1. *Push (기본) vs Pull (ansible-pull) vs AWX/Tower (팀 운영)*. 대규모·audit·schedule 필요하면 AWX 필수.
2. 안전 패턴: `--check --diff` → *serial + max_fail_percentage* → *idempotency 검증 (Molecule)* → AWX audit.
3. *Ansible = configuration management, Terraform = infra provisioning*. 함께 쓰는 게 정석. *K8s 클러스터 위 자원은 Helm/ArgoCD가 더 효율*.

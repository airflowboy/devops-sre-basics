# 02 — Playbook & Tasks: play / task / handler / loop / block

> **공식 근거:** [Working with playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/), [Loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)

---

## 🎯 한 문장

> **Playbook = *YAML로 명세된 play의 리스트*. Play = *호스트 집합 + 실행할 task들*. Task = *module 호출 한 번*. Handler = *변경 시만 실행되는 특수 task* (notify로 trigger).**

---

## 1. Playbook 기본 구조

```yaml
---
# webserver-setup.yml
- name: Configure web servers
  hosts: webservers
  become: yes                        # sudo로 권한 상승
  vars:
    nginx_port: 8080
  tasks:
  - name: Ensure nginx is installed
    apt:
      name: nginx
      state: present
      update_cache: yes
    
  - name: Deploy nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      mode: '0644'
    notify: restart nginx            # ← handler trigger
    
  - name: Start nginx
    service:
      name: nginx
      state: started
      enabled: yes
  
  handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted

- name: Configure databases               # ← 두 번째 play (다른 호스트)
  hosts: databases
  become: yes
  tasks:
  - ...
```

### 실행
```bash
ansible-playbook -i inventory.yml webserver-setup.yml

# 특정 호스트만
ansible-playbook -i inventory.yml webserver-setup.yml --limit web1.example.com

# extra-vars
ansible-playbook -i inventory.yml webserver-setup.yml -e nginx_port=9090

# check mode (dry-run)
ansible-playbook -i inventory.yml webserver-setup.yml --check --diff

# 특정 task부터
ansible-playbook -i inventory.yml webserver-setup.yml --start-at-task="Deploy nginx config"

# 태그만 실행
ansible-playbook -i inventory.yml webserver-setup.yml --tags "config"
```

---

## 2. Play의 keys (자주 쓰는 것)

```yaml
- name: ...
  hosts: webservers                    # 대상
  remote_user: deploy                  # SSH user
  become: yes                          # privilege escalation
  become_user: root
  become_method: sudo
  gather_facts: yes                    # facts 자동 수집 (기본 yes)
  serial: 5                            # 한 번에 5 호스트씩 (rolling update)
  serial: "30%"                        # 또는 % 표현
  max_fail_percentage: 10              # 10% 이상 fail이면 중단
  any_errors_fatal: false              # 단일 host fail 시 전체 중단?
  strategy: linear                     # 또는 free (host 간 task 순서 비동기)
  vars: {...}                          # play 변수
  vars_files:                          # 외부 파일
  - secrets.yml
  pre_tasks: [...]                     # 본 tasks 전
  tasks: [...]
  post_tasks: [...]                    # 본 tasks 후
  handlers: [...]
  roles:
  - common
  - webserver
```

### Strategy
- `linear` (기본) — *모든 host가 한 task 완료 후 다음 task*
- `free` — *각 host가 *자기 속도로* 모든 task 진행* (병렬 비동기)
- `host_pinned` — host당 worker 고정

→ 대부분 `linear`. *오래 걸리는 task 많고 host 독립적*이면 `free`.

---

## 3. Task의 기본 구조

```yaml
- name: Install package         # 명확한 이름 (운영에서 중요)
  apt:                          # module 이름
    name: nginx                 # module 인자
    state: present
  become: yes                   # task별 override
  when: ansible_os_family == "Debian"   # 조건
  tags: [packages, web]         # 태그
  register: install_result      # 결과 변수에 저장
  ignore_errors: yes            # 실패 무시
  failed_when: install_result.rc != 0   # custom 실패 조건
  changed_when: false           # changed로 표시 안 함
  notify:                       # handler trigger
  - restart nginx
  - reload haproxy
```

### Task name이 *명확해야* 하는 이유
- 실행 출력에 *task name 그대로 표시*
- 디버깅·로그 검색에 *직접 영향*
- 자동화 시스템 (AWX 등)이 *task별 status 추적*

```
TASK [Install nginx] **************************
ok: [web1]
changed: [web2]
```

→ "Install nginx" 같은 *의미 있는 이름* 필수.

---

## 4. Handlers — *변경 시만 실행*

### 동작
- *handlers는 task처럼 정의*되지만 *기본적으론 실행 안 됨*
- *task가 `notify: handler-name` 하고 *그 task가 changed*인 경우만 trigger*
- *play 끝나면 trigger된 handler 한 번 실행* (여러 task가 같은 handler notify해도 *한 번*)

```yaml
tasks:
- name: Update config A
  template:
    src: config-a.j2
    dest: /etc/myapp/a.conf
  notify: restart myapp

- name: Update config B
  template:
    src: config-b.j2
    dest: /etc/myapp/b.conf
  notify: restart myapp

handlers:
- name: restart myapp
  service:
    name: myapp
    state: restarted
```

- a.conf·b.conf 둘 다 변경 → `restart myapp` *한 번만* 실행
- 둘 다 안 변경 → handler 실행 안 됨

### `meta: flush_handlers` — *지금 즉시 실행*
```yaml
- name: Update config
  template: ...
  notify: restart myapp

- meta: flush_handlers              # ← 여기서 trigger된 handler 모두 실행

- name: Test endpoint                # restart 후 수행할 작업
  uri: ...
```

### Handler가 *fail*하면
- 기본적으로 *play 중단*
- *이후 다른 task의 handler trigger도 실행 안 됨*
- `force_handlers: yes` 로 *실패에도 handler 실행 강제*

---

## 5. Conditionals — `when:`

```yaml
- name: Install on Debian
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Install on RedHat
  dnf:
    name: nginx
  when: ansible_os_family == "RedHat"

- name: Restart only on prod
  service:
    name: myapp
    state: restarted
  when:
  - inventory_hostname in groups['production']
  - app_version == '1.0.0'
  # list = AND 조건

- name: Conditional on previous result
  shell: echo "previous task did something"
  when: previous_result is changed       # 또는 is succeeded / is failed
```

### `register` + when 조합
```yaml
- name: Check if file exists
  stat:
    path: /opt/myapp/initialized
  register: init_check

- name: Initialize
  shell: /opt/myapp/init.sh
  when: not init_check.stat.exists
```

→ 멱등성을 *명시적*으로 표현.

---

## 6. Loops — `loop:`

### 단순 list
```yaml
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
  - nginx
  - postgresql
  - redis
```

### dict의 list
```yaml
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
  - { name: alice, groups: "admin,docker" }
  - { name: bob, groups: "users" }
```

### loop_control
```yaml
- name: ...
  loop: "{{ users }}"
  loop_control:
    label: "{{ item.name }}"          # 출력에 사용자 이름만 표시 (긴 dict 줄임)
    pause: 1                          # 각 iteration 사이 1초
    index_var: idx                    # 인덱스 변수
```

### with_* (구식)
```yaml
- name: 옛 방식
  apt:
    name: "{{ item }}"
  with_items:                         # ← 옛 문법, loop 권장
  - nginx
  - postgresql
```
- `with_items` ~ `with_file` ~ `with_sequence` 등 옛 plugin
- 현재는 *`loop:`* + lookup plugin 권장

### Nested loop
```yaml
- name: Nested
  debug:
    msg: "{{ item.0 }} - {{ item.1 }}"
  loop: "{{ users | product(databases) | list }}"
```

---

## 7. Blocks — *grouping + try/except*

```yaml
- name: Install and configure
  block:
  - name: Install package
    apt:
      name: nginx
  - name: Deploy config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
  - name: Start service
    service:
      name: nginx
      state: started
  rescue:                                # ← catch (위 block에서 fail 시)
  - name: Notify failure
    debug:
      msg: "Setup failed, rolling back"
  - name: Stop service
    service:
      name: nginx
      state: stopped
  always:                                # ← 항상 실행 (try/finally)
  - name: Log execution
    lineinfile:
      path: /var/log/ansible-runs.log
      line: "{{ ansible_date_time.iso8601 }}: setup attempted"
```

### Block-level 속성
```yaml
- block: [...]
  when: ansible_os_family == "Debian"   # 전체 block에 적용
  become: yes
  tags: [setup]
```

→ 같은 `when:` / `become:` 등 여러 task에 *반복 안 함*.

---

## 8. Tags — *선택 실행*

```yaml
tasks:
- name: Install package
  apt: name=nginx
  tags: [packages, web]

- name: Deploy config
  template: ...
  tags: [config, web]

- name: Restart
  service: ...
  tags: [service]
```

```bash
# 태그만 실행
ansible-playbook playbook.yml --tags "config"
ansible-playbook playbook.yml --tags "config,service"

# 태그 제외
ansible-playbook playbook.yml --skip-tags "packages"

# 사용 가능한 태그 보기
ansible-playbook playbook.yml --list-tags
```

### 예약 tag
- `always` — *항상 실행* (`--skip-tags` 로만 제외 가능)
- `never` — *명시적으로 호출해야만* (`--tags "never"`)

```yaml
- name: Debug task
  debug: var=facts
  tags: never                          # 평소엔 실행 안 됨, --tags 시만
```

---

## 9. Facts — *target의 자동 수집 정보*

```yaml
- hosts: all
  gather_facts: yes                    # 기본 yes
  tasks:
  - debug:
      var: ansible_facts
```

수집되는 facts (`ansible_facts.*`, 또는 직접 `ansible_*`):
- `ansible_os_family` — Debian/RedHat
- `ansible_distribution` — Ubuntu/CentOS/Rocky
- `ansible_distribution_version` — 22.04
- `ansible_kernel`, `ansible_architecture`
- `ansible_hostname`, `ansible_fqdn`
- `ansible_default_ipv4.address` — 기본 IPv4
- `ansible_memtotal_mb`, `ansible_processor_vcpus`
- `ansible_mounts`, `ansible_devices`

### Custom facts
- `/etc/ansible/facts.d/*.fact` 파일들이 *자동으로 facts에 포함*
- INI 또는 JSON 형식

### Fact caching
- 매번 *gather_facts*가 *수 초 소요*
- `ansible.cfg` 에서 *caching* 활성화 (jsonfile / redis / memcached)
- 한 번 수집 후 *TTL 동안 재사용*

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
```

### gather_facts 끄기
```yaml
- hosts: all
  gather_facts: no                     # 빠른 실행 (facts 필요 없으면)
  tasks: ...
```

→ 100+ host면 *gather_facts off로 수십 초 절감*.

---

## 10. 자주 헷갈리는 것

### 10-1. *task name 없음* → debugging 지옥
- task name 필수 — *출력에 그대로 표시*, *디버깅·로깅 핵심*

### 10-2. *YAML 들여쓰기 사고*
- *space 2 vs space 4* 혼용 → parse fail
- `ansible-lint` 로 *사전 검출*

### 10-3. *handler가 *실행 안 됨**
- *trigger task가 *changed*여야 함*. ok 면 trigger X.
- *play 끝나야 실행*. 중간에 보려면 `meta: flush_handlers`
- *playbook 자체가 중간 fail*하면 handler 실행 X — `force_handlers: yes`

### 10-4. *loop 안에서 register*
```yaml
- name: ...
  shell: ...
  loop: [a, b, c]
  register: result
# result.results 는 *list* — 각 iteration의 결과
```
- 일반 result처럼 다루면 *기대 안 함*. `result.results[0].stdout` 식.

### 10-5. *check mode*에서 *shell module 결과*
- `--check` 시 shell module은 *기본 skip* (변경 추정 불가)
- `check_mode: no` 로 *check mode에서도 실행 강제* (read-only 명령에 유용)

### 10-6. *serial: 1 + max_fail_percentage* — rolling update
```yaml
- hosts: webservers
  serial: 1                            # 1 host씩
  max_fail_percentage: 0               # 1 host fail = 전체 중단
```
→ 안전한 rolling deploy 패턴.

---

## 🎤 면접 빈출 Q&A

### Q1. Play / Task / Handler 차이?
> **Play** = *host 집합 + 실행할 task 들*. **Task** = *module 호출 한 번*. **Handler** = *변경 시만 실행되는 특수 task* — `notify: handler-name` 로 trigger되고 *play 끝에 한 번*만 실행 (여러 notify해도 합쳐서 한 번). 흔한 활용: *config 변경 → restart service* — 변경 없으면 restart 안 함.

### Q2. `when:` 으로 *멱등성 명시*하는 패턴?
> `stat` module로 *현재 상태 register* → `when: not result.stat.exists` 로 *없을 때만 실행*. 또는 `creates:`/`removes:` 인자 (shell·command module). 또는 `changed_when: false` 로 *읽기 전용 task 명시*. *멱등성 = module의 책임* 이지만 *조건부 실행으로 명시적 표현* 가능.

### Q3. Block의 rescue·always 의미?
> Python의 try/except/finally와 동일. **block**: 실행할 task들. **rescue**: block에서 *fail 시* 실행 (catch). **always**: 성공·실패 무관 *항상* 실행 (finally). 흔한 활용: 설치 fail 시 *rollback (rescue)*, 결과 *로깅 (always)*.

### Q4. Loop 안에서 *iteration별 결과* 어떻게?
> `register: result` 하면 `result.results` 가 *list* — 각 iteration의 *module 결과*. `loop_control.label`로 출력 *간결화*. nested loop는 `loop: "{{ list1 | product(list2) | list }}"` 패턴.

### Q5. Tags는 언제 유용?
> *큰 playbook의 일부분만 실행*하고 싶을 때. `--tags "config"` 로 config 관련 task만, `--skip-tags "packages"` 로 패키지 설치 빼고. 예약 tag: `always` (항상), `never` (명시 호출만). 빠른 반복 디버깅 시 *유용*.

### Q6. *Rolling update* 안전하게 어떻게?
> `serial: 1` (1 host씩) + `max_fail_percentage: 0` (한 host fail = 전체 중단) 조합. 또는 `serial: "10%"`. *load balancer에서 빼기 → update → health check → 다시 추가* 패턴은 `pre_tasks` / `post_tasks` 활용. *blue/green 비슷한 효과*.

### Q7. `gather_facts: no` 가 *성능에 미치는 영향*?
> facts 수집이 *각 host당 수 초*. 100 host면 수십 초~분. *facts 안 쓰는 playbook*은 `gather_facts: no` 로 *대폭 단축*. 또는 *fact caching* (jsonfile/redis) 활성화 — 한 번 수집 후 TTL 동안 재사용.

---

## 🔗 Cross-reference

- **Inventory + module** → [01-inventory-and-modules.md](01-inventory-and-modules.md)
- **변수 우선순위 (when의 변수 source)** → [03-variables-and-templating.md](03-variables-and-templating.md)
- **Role로 task 재사용** → [04-roles-and-collections.md](04-roles-and-collections.md)
- **CI에서 ansible-playbook 자동화** → [05-execution-and-best-practices.md](05-execution-and-best-practices.md)
- **K8s의 reconcile vs Ansible의 멱등성** → [`kubernetes-basics/06`](../kubernetes-basics/06-api-and-controllers.md) — *둘 다 desired state로 수렴, K8s는 영원히 / Ansible은 실행 시만*

---

## 📝 3줄 요약

1. *Playbook = play 리스트, play = host + task, task = module 호출*. Task name 필수. Handler는 *변경 시만, play 끝에 한 번*.
2. `when:`·`block/rescue/always`·`loop:`·`tags`로 *제어 흐름·재사용*. *register + when* 패턴이 멱등성의 friend.
3. `gather_facts: no` + *fact caching* + `serial: 1` + `max_fail_percentage` = 운영 환경의 *안전 + 성능* 조합.

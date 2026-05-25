# 03 — Variables & Templating: 22 layers 우선순위 + Jinja2 + Vault

> **공식 근거:** [Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html), [Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable), [Jinja2](https://jinja.palletsprojects.com/)

---

## 🎯 한 문장

> **변수는 Ansible에서 *가장 복잡한 영역*. 22단계 우선순위, Jinja2 templating, vault로 secret, lookup/filter plugin, register/set_fact로 동적. *이걸 정복하면 Ansible의 70%*.**

---

## 1. 변수 정의 위치 22단계 — *낮음 → 높음*

공식 문서 그대로:

1. command-line / role default values (defaults/main.yml)
2. inventory file or script group vars
3. inventory group_vars/all
4. playbook group_vars/all
5. inventory group_vars/*
6. playbook group_vars/*
7. inventory file or script host vars
8. inventory host_vars/*
9. playbook host_vars/*
10. host facts / cached_facts
11. play vars
12. play vars_prompt
13. play vars_files
14. role vars (vars/main.yml)
15. block vars (only for tasks in block)
16. task vars (only for the task)
17. include_vars
18. set_facts / registered vars
19. role (and include_role) params
20. include params
21. extra vars (`-e`, 가장 높음)
22. (위 21이 마지막)

→ *실용적으로 외우는 5가지*:
- **role defaults (가장 낮음)** — chart의 *기본값* 같은 역할
- **inventory group_vars / host_vars** — 환경별 override
- **play vars** — playbook 안 *전용 값*
- **set_facts / register** — *런타임 동적 값*
- **extra vars `-e` (가장 높음)** — CLI에서 *최종 override*

### 실용 권장
- *기본값은 role의 `defaults/main.yml`* — override 쉬움
- *환경별 차이는 inventory group_vars*
- *secret 같은 sensitive는 vault*
- *임시 override는 `-e key=value`*

---

## 2. 변수 정의 위치별 예시

### Role defaults — *가장 낮은 우선순위*
```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_processes: auto
nginx_keepalive_timeout: 65
```

### Role vars — *높은 우선순위*
```yaml
# roles/nginx/vars/main.yml
# 보통 *변경되면 안 되는 값*
nginx_user: www-data
nginx_log_format: combined
```

→ **defaults vs vars** 헷갈림: *defaults는 쉽게 override, vars는 거의 fixed.*

### Group vars / Host vars
```yaml
# inventory/group_vars/webservers.yml
nginx_port: 80
nginx_worker_processes: 4

# inventory/group_vars/all.yml
ansible_user: deploy
ntp_servers: [pool.ntp.org]

# inventory/host_vars/web1.example.com.yml
nginx_worker_processes: 8       # 특정 host만 override
```

### Play vars / vars_files
```yaml
- hosts: all
  vars:
    app_version: "1.2.3"
  vars_files:
  - secrets/credentials.yml
  tasks: ...
```

### set_fact — *런타임 동적*
```yaml
- name: Detect environment
  set_fact:
    is_prod: "{{ inventory_hostname.endswith('.prod.example.com') }}"
    deploy_dir: "/opt/myapp-{{ ansible_date_time.epoch }}"
```

### register — *task 결과 저장*
```yaml
- name: Get nginx status
  shell: systemctl is-active nginx
  register: nginx_status
  changed_when: false

- name: Show result
  debug:
    msg: "nginx is {{ nginx_status.stdout }}"
```

### Extra vars — *CLI 최종*
```bash
ansible-playbook playbook.yml -e "app_version=2.0.0 environment=prod"
ansible-playbook playbook.yml -e @vars.yml          # 파일에서
ansible-playbook playbook.yml -e '{"complex": ["list"]}'   # JSON
```

---

## 3. Jinja2 Templating — *값 참조 / 표현*

### 기본 syntax
```yaml
# 단순 참조
- debug:
    msg: "Welcome, {{ user_name }}!"

# 조건 표현 (filter)
- debug:
    msg: "Port: {{ port | default(80) }}"

# 산술
- debug:
    msg: "Total: {{ count * 10 }}"

# 비교 (when)
- name: ...
  when: ansible_memtotal_mb > 1024
```

### 자주 쓰는 filter
| Filter | 의미 |
|---|---|
| `default(val)` | 빈 값 시 기본값 |
| `mandatory` | 정의 안 되면 fail |
| `length` | 문자열·list 길이 |
| `upper` / `lower` / `title` / `capitalize` |
| `trim` / `replace('old', 'new')` |
| `split('/')` | string → list |
| `join(',')` | list → string |
| `unique` / `union` / `intersect` / `difference` | set 연산 |
| `to_json` / `from_json` |
| `to_yaml` / `from_yaml` |
| `b64encode` / `b64decode` |
| `hash('sha256')` |
| `regex_search('pat')` / `regex_replace('pat', 'rep')` |
| `bool` | "yes"/"true" → True |
| `int` / `float` / `string` |
| `selectattr('key', 'equalto', 'val')` | list of dict 필터링 |
| `map(attribute='key')` | list of dict → list of values |

### 예시
```yaml
- name: Filter users
  debug:
    msg: "{{ users | selectattr('admin', 'equalto', true) | map(attribute='name') | list }}"

- name: Build URL
  set_fact:
    api_url: "https://{{ inventory_hostname }}/{{ api_path | regex_replace('^/', '') }}"

- name: Generate hash
  set_fact:
    config_hash: "{{ lookup('file', 'config.yml') | hash('sha256') }}"
```

### Test (조건)
```yaml
- name: ...
  when: var is defined
  when: var is undefined
  when: var is none
  when: var is string / number / mapping / sequence
  when: file_result is changed / succeeded / failed / skipped
  when: ansible_distribution is search('Ubuntu')
```

→ `is defined` vs `is none` 헷갈림 주의.

---

## 4. Template Module — *Jinja2 파일을 target에 렌더링*

### 사용
```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes                  # 변경 시 옛 파일 backup
    validate: 'nginx -t -c %s'   # validate command (실패 시 deploy X)
  notify: restart nginx
```

### Template 예시
```jinja2
# nginx.conf.j2
worker_processes {{ nginx_worker_processes | default('auto') }};

events {
    worker_connections 1024;
}

http {
    keepalive_timeout {{ nginx_keepalive_timeout }};
    
    {% for upstream in nginx_upstreams %}
    upstream {{ upstream.name }} {
        {% for server in upstream.servers %}
        server {{ server.host }}:{{ server.port }};
        {% endfor %}
    }
    {% endfor %}
    
    server {
        listen {{ nginx_port }};
        server_name {{ inventory_hostname }};
        
        {% if ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};
        {% endif %}
    }
}
```

### Jinja2 control structures
```jinja2
{% for item in list %}
{{ item }}
{% endfor %}

{% if condition %}
...
{% elif other %}
...
{% else %}
...
{% endif %}

{% set my_var = "value" %}
{{ my_var }}

{# 주석 #}
```

### 공백 제어 (Helm과 비슷)
```jinja2
{%- for item in items %}      ← 왼쪽 공백 제거
- {{ item }}
{%- endfor %}
```

→ Helm template 의 `{{- }}` 과 *같은 발상*. ([`helm-basics/02`](../helm-basics/02-templating.md))

---

## 5. Magic Variables — *내장 변수*

| 변수 | 의미 |
|---|---|
| `inventory_hostname` | 현재 host의 inventory 이름 |
| `inventory_hostname_short` | FQDN의 첫 부분 |
| `groups` | 모든 그룹·hosts dict |
| `group_names` | 현재 host가 속한 그룹 list |
| `hostvars` | 모든 host의 vars dict (다른 host의 var 참조) |
| `ansible_facts` / `ansible_*` | gather_facts 수집 결과 |
| `playbook_dir` | playbook 파일의 디렉터리 |
| `inventory_dir` / `inventory_file` |
| `ansible_play_hosts` | 현재 play의 host list |
| `ansible_play_batch` | 현재 batch (serial 사용 시) |
| `ansible_run_tags` | `--tags` 로 지정된 태그 |

### `hostvars` — *다른 host의 var 참조*
```yaml
- name: Show DB master IP from web servers
  debug:
    msg: "DB master IP: {{ hostvars['db1.example.com']['ansible_default_ipv4']['address'] }}"
```

### `groups` — *그룹의 모든 host*
```yaml
- name: Generate /etc/hosts
  template:
    src: hosts.j2
    dest: /etc/hosts
```

```jinja2
{# hosts.j2 #}
{% for host in groups['all'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }} {{ host }}
{% endfor %}
```

---

## 6. Lookup Plugin — *외부에서 값 가져오기*

```yaml
- name: Read file content
  debug:
    msg: "{{ lookup('file', '/etc/hostname') }}"

- name: Read env var
  debug:
    msg: "{{ lookup('env', 'HOME') }}"

- name: Read from password manager
  debug:
    msg: "{{ lookup('community.hashi_vault.vault_kv2_get', 'kv/myapp/db', password=true) }}"

- name: Read from URL
  debug:
    msg: "{{ lookup('url', 'https://example.com/data.json') }}"

- name: Generate password
  set_fact:
    new_password: "{{ lookup('password', '/dev/null length=20 chars=ascii_letters,digits') }}"

- name: Multiple files
  debug:
    msg: "{{ lookup('fileglob', 'configs/*.conf', wantlist=True) }}"
```

### 자주 쓰는 lookup
- `file` — 파일 내용
- `env` — 환경변수
- `password` — 비밀번호 생성/조회
- `url` — URL fetch
- `pipe` — shell 명령 결과
- `csvfile`, `ini` — 구조화된 파일
- `vars` / `varnames`
- `community.hashi_vault.*` — Vault
- `amazon.aws.aws_secret`, `amazon.aws.aws_ssm` — AWS Secrets Manager / SSM

→ Lookup은 *control node에서 실행* (target 아님).

---

## 7. Ansible Vault — *secret 암호화*

### 발상
- secret을 *git에 평문으로 commit 안 됨*
- 해결: *값 자체를 AES-256으로 암호화*해서 git에 commit
- 실행 시 *passphrase 입력해 복호화*

### 사용
```bash
# 새 vault 파일 생성
ansible-vault create secrets.yml

# 편집
ansible-vault edit secrets.yml

# 비밀번호 변경
ansible-vault rekey secrets.yml

# 평문 → 암호화
ansible-vault encrypt vars.yml

# 암호화 → 평문 (조심)
ansible-vault decrypt vars.yml

# 보기 (편집 X)
ansible-vault view secrets.yml
```

### Playbook 실행 시
```bash
# 비밀번호 prompt
ansible-playbook playbook.yml --ask-vault-pass

# password 파일에서
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# script로 동적 password (Vault·KMS 통합)
ansible-playbook playbook.yml --vault-password-file ~/get_vault_pass.sh
```

### Single value 암호화 — *partial vault*
```bash
ansible-vault encrypt_string 'mysecretpassword' --name 'db_password'
```
출력:
```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  62313365396666...
```
→ 일반 vars.yml 안에 *한 값만 암호화*. 나머지는 git diff 보기 좋게 평문.

### Vault ID — *여러 vault*
```bash
# 환경별로 다른 password
ansible-vault encrypt --vault-id prod@~/.vault_prod_pass secrets-prod.yml
ansible-vault encrypt --vault-id dev@~/.vault_dev_pass secrets-dev.yml

# 실행 시 둘 다 제공
ansible-playbook playbook.yml --vault-id prod@~/.vault_prod_pass --vault-id dev@~/.vault_dev_pass
```

### 한계와 대안
- *passphrase 관리가 운영 부담* — Vault·1Password·AWS Secrets Manager 같은 *외부 secret manager 통합*이 현대적 대안
- ansible-vault는 *작은 규모·간단한 secret*에 적합
- 대규모면 *HashiCorp Vault / AWS Secrets Manager + lookup* 패턴

---

## 8. register vs set_fact 차이

### register — *task 결과 저장*
```yaml
- shell: cat /etc/myapp/version
  register: version_result
  changed_when: false

- debug:
    var: version_result      # 전체 결과 dict (.stdout, .stderr, .rc, .changed, ...)
```

### set_fact — *변수 명시적 설정*
```yaml
- set_fact:
    app_version: "{{ version_result.stdout }}"

- set_fact:
    is_prod: "{{ inventory_hostname.endswith('-prod') }}"
    deploy_dir: "/opt/myapp-{{ app_version }}"
```

### 차이
- `register` = *task 결과 그대로* (dict 구조)
- `set_fact` = *임의 표현식 → 새 변수*
- 둘 다 *현재 host의 facts에 추가*

### set_fact의 scope
- 기본: *그 host의 play 내 모든 task에서 사용 가능*
- `cacheable: yes` 옵션으로 *fact cache에 영구 저장*

```yaml
- set_fact:
    app_version: "1.2.3"
    cacheable: yes
```

---

## 9. when:과 변수 표현 — *문법 함정*

```yaml
# ✅ 올바름
- name: ...
  when: ansible_memtotal_mb > 1024
  when: my_var == "prod"
  when: my_list | length > 0
  when: my_var is defined

# ❌ 잘못 (Jinja2 brace 안 쓰기)
- when: "{{ ansible_memtotal_mb > 1024 }}"     # 작동하나 warning
```

### Boolean type 함정
```yaml
# inventory에서
my_flag: "yes"           # string!

# playbook에서
when: my_flag            # 항상 truthy (non-empty string)
when: my_flag | bool     # → True
```

→ inventory에서 `yes`/`no`/`true`/`false`는 *string*. *반드시 `| bool` 필터*.

---

## 10. 자주 헷갈리는 것

### 10-1. *role defaults vs role vars*
- defaults = *override 쉬움* (가장 낮은 우선순위)
- vars = *거의 fixed* (높은 우선순위)
- → *override 의도하면 defaults에 두기*

### 10-2. *변수 이름 *Python keyword 충돌**
- `name`, `class`, `type` 등 사용 시 *예약어 충돌*
- 안전한 prefix: `app_`, `role_`, `__`

### 10-3. *YAML의 `yes`/`no` → boolean 함정*
- `nginx_enabled: yes` → boolean True
- `nginx_version: "1.0"` → quote 안 하면 *float 1.0*
- → *항상 quote 권장*

### 10-4. *Vault 비밀번호 commit*
- `.vault_pass` 파일 *절대 git commit X*
- `.gitignore`에 추가

### 10-5. *Jinja2 template의 *공백·줄바꿈 사고**
- Helm template과 같은 함정. `{%- -%}` 신중하게.
- `lstrip_blocks: True` 옵션으로 *자동 lstrip*

### 10-6. *lookup vs query*
- `lookup('module', 'arg')` — *string 또는 list 반환*
- `query('module', 'arg')` — *항상 list 반환*
- list 처리할 때 `query` 가 안전

---

## 🎤 면접 빈출 Q&A

### Q1. Ansible 변수 우선순위는?
> 공식 22단계. 실용적으로 5개만 기억: **role defaults (가장 낮음)** → inventory group_vars / host_vars → play vars → role vars → set_facts / register → **extra vars `-e` (가장 높음)**. *기본값은 role defaults*, *환경별은 group_vars*, *최종 override는 `-e`*.

### Q2. role의 defaults/main.yml과 vars/main.yml 차이?
> *defaults* = 가장 낮은 우선순위 — *쉽게 override 가능*. role 사용자가 *원하는 값으로 변경*. *vars* = 높은 우선순위 — *거의 fixed*. role 내부에서 사용하는 *상수* 같은 값 (예: nginx 사용자명). *override 의도면 defaults에 두기*.

### Q3. Jinja2 template과 Helm template 비교?
> Jinja2는 *Ansible·Flask·Django 등 광범위 사용*. Helm은 *Go template + sprig*. 둘 다 *공백 제어* (`{%- -%}` vs `{{- -}}`), *if/for*, *filter pipe*, *named template/macro*. 차이: Jinja2는 *Python 생태계 친화*, Go template은 *strong typing*. 같은 *공백 사고* 함정.

### Q4. Ansible Vault는 무엇? 한계?
> AES-256으로 *값 자체를 암호화*해서 git에 commit. `ansible-vault encrypt/edit/view`. 실행 시 *passphrase 제공*. **한계**: passphrase 관리 자체가 부담. *대규모면 외부 secret manager* (HashiCorp Vault, AWS Secrets Manager) + *lookup plugin* 으로 *동적 fetch* 더 적합.

### Q5. register와 set_fact 차이?
> `register` = *task 결과 그대로* (dict — `.stdout`, `.rc`, `.changed` 등). `set_fact` = *임의 표현식 → 새 변수*. 둘 다 *그 host의 facts에 추가, 같은 play 내 사용 가능*. `set_fact: cacheable: yes` 로 *fact cache에 영구 저장*도 가능.

### Q6. *string "yes"가 boolean으로 평가되지 않는* 함정?
> YAML에서 `key: yes` → boolean True. `key: "yes"` → string. inventory에서 *quoted string* 으로 정의하면 *boolean 아님*. when 조건에서 *항상 truthy* 로 평가됨 (non-empty string). **`| bool` filter** 로 *명시적 변환* 필수: `when: my_var | bool`.

### Q7. *다른 host의 변수* 참조하려면?
> `hostvars` magic variable. `hostvars['db1.example.com']['ansible_default_ipv4']['address']`. 또는 그룹의 모든 host: `groups['databases']`. 흔한 활용: *web server의 nginx config에 *DB master IP 박기*, *모든 노드의 IP로 /etc/hosts 생성*.

---

## 🔗 Cross-reference

- **playbook의 when·register** → [02-playbook-and-tasks.md](02-playbook-and-tasks.md)
- **role의 defaults/vars** → [04-roles-and-collections.md](04-roles-and-collections.md)
- **Helm template과 비교** → [`helm-basics/02`](../helm-basics/02-templating.md) — *같은 공백 사고, 다른 도구*
- **K8s Secret과 비교** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md), *둘 다 secret 관리, 다른 추상화 수준*
- **HashiCorp Vault·AWS Secrets Manager** → `aws-basics/` (예정), `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. *변수 22단계 우선순위* — 실용 5개: role defaults (낮음) → group_vars/host_vars → play vars → set_facts → extra vars `-e` (가장 높음).
2. Jinja2 template으로 *값 참조·filter·loop*. 자주 쓰는 filter: `default`, `selectattr`, `map(attribute=)`, `to_json`, `hash('sha256')`.
3. *Vault*로 secret 암호화 (소규모). 대규모면 *외부 secret manager + lookup plugin*. `yes`/`no` quote 함정·`| bool` filter 주의.

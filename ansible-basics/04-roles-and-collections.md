# 04 — Roles & Collections: 재사용 단위

> **공식 근거:** [Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html), [Collections](https://docs.ansible.com/ansible/latest/collections_guide/), [Galaxy](https://galaxy.ansible.com/)

---

## 🎯 한 문장

> **Role = *task + handler + template + vars + defaults를 묶은 *재사용 단위**. Collection = *role + module + plugin 의 *배포 단위* (Ansible 2.10+). 둘 다 *Galaxy*에 publish.**

---

## 1. Role — *재사용 가능한 playbook 조각*

### 디렉터리 구조 (표준)
```
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml          ← 기본 변수 (override 쉬움)
    ├── vars/
    │   └── main.yml          ← role 내부 변수 (fixed)
    ├── tasks/
    │   ├── main.yml          ← 메인 task
    │   ├── install.yml       ← include용 sub task
    │   └── configure.yml
    ├── handlers/
    │   └── main.yml          ← handler
    ├── templates/
    │   └── nginx.conf.j2     ← Jinja2 template (template module의 src)
    ├── files/
    │   └── ssl-cert.pem      ← static 파일 (copy module의 src)
    ├── meta/
    │   └── main.yml          ← role 메타데이터 + dependency
    ├── tests/
    │   ├── inventory
    │   └── test.yml          ← role 테스트용
    └── README.md
```

### `ansible-galaxy role init` 으로 빠르게
```bash
ansible-galaxy role init nginx
# → 위 구조 자동 생성
```

---

## 2. Role 사용 방법

### Playbook에서 `roles:`
```yaml
- hosts: webservers
  become: yes
  roles:
  - common
  - nginx
  - { role: app, vars: { app_version: "1.2.3" } }
  - role: monitoring
    tags: [observability]
```

→ 명시된 *순서대로 실행*. 각 role의 `tasks/main.yml` 전체 실행.

### `tasks:` 안에서 `import_role` / `include_role`
```yaml
- hosts: webservers
  tasks:
  - name: Install common
    import_role:                  # 정적: parse time에 task 추가
      name: common
  
  - name: Install nginx conditionally
    include_role:                 # 동적: 실행 시점에 평가
      name: nginx
    when: install_nginx | bool
    
  - name: Run specific task from role
    include_role:
      name: nginx
      tasks_from: configure       # tasks/configure.yml만
```

### import vs include 차이
| | import_role | include_role |
|---|---|---|
| 처리 시점 | *parse time* (playbook 로딩 시) | *runtime* (실행 시점) |
| `when:` | task 단위로 평가 | 전체 role에 평가 |
| `loop:` | 안 됨 | 가능 |
| tag 상속 | 그대로 | 적용 |

→ 일반적으로 `import_role`, *조건부·loop 필요하면 `include_role`*.

---

## 3. tasks/main.yml + include

### 작은 role: tasks/main.yml 하나
```yaml
# roles/ntp/tasks/main.yml
- name: Install ntp
  package:
    name: ntp
    state: present

- name: Configure ntp
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  notify: restart ntp

- name: Start ntp
  service:
    name: ntp
    state: started
    enabled: yes
```

### 큰 role: 여러 파일로 분리
```yaml
# roles/myapp/tasks/main.yml
- import_tasks: install.yml
- import_tasks: configure.yml
- import_tasks: service.yml
```

```yaml
# roles/myapp/tasks/install.yml
- name: Install dependencies
  apt: ...

- name: Install myapp
  apt: ...
```

→ *큰 role은 의미별로 파일 분리*가 운영 friendly.

---

## 4. handlers/main.yml

```yaml
# roles/nginx/handlers/main.yml
- name: restart nginx
  service:
    name: nginx
    state: restarted

- name: reload nginx
  service:
    name: nginx
    state: reloaded

- name: validate nginx config
  command: nginx -t
  changed_when: false
```

→ task에서 `notify: restart nginx`.

### Handler chain
```yaml
# notify chain
- name: validate nginx config
  command: nginx -t
  changed_when: false
  notify: restart nginx      # validate 후 restart trigger
```

---

## 5. templates/ + files/

### templates/ — Jinja2 (template module)
```yaml
- template:
    src: nginx.conf.j2        # roles/nginx/templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

### files/ — static (copy module)
```yaml
- copy:
    src: ssl-cert.pem         # roles/nginx/files/ssl-cert.pem
    dest: /etc/ssl/cert.pem
    mode: '0600'
```

→ `src:` 에 *상대 경로*만 — Ansible이 *role의 templates/ 또는 files/ 에서 자동 검색*.

---

## 6. meta/main.yml — *role 메타데이터*

```yaml
# roles/nginx/meta/main.yml
galaxy_info:
  role_name: nginx
  author: Alice
  description: Install and configure nginx
  license: MIT
  min_ansible_version: "2.10"
  platforms:
  - name: Ubuntu
    versions: [20.04, 22.04]
  - name: EL
    versions: [8, 9]
  galaxy_tags:
  - web
  - nginx

dependencies:
- role: common               # 이 role 실행 전 common이 먼저
  vars:
    common_packages: [curl, vim]
- role: certbot
  when: ssl_enabled | bool
```

### dependency 처리
- 의존 role이 *현재 role 전에 실행*
- 중복 호출 방지 (allow_duplicates 옵션)
- *순환 의존 X*

---

## 7. defaults/main.yml vs vars/main.yml — *다시*

| | defaults/main.yml | vars/main.yml |
|---|---|---|
| 우선순위 | *가장 낮음* (1) | 높음 (14) |
| 의도 | 사용자가 *override 권장* | role *내부 상수* |
| 예시 | port, replicas, version | nginx user name, log format |

→ role 사용자가 *override 의도하는 값*은 *반드시 defaults에*. vars에 두면 *override 어렵고 헷갈림*.

---

## 8. Collection — *role + module + plugin의 배포 단위*

### 발상 (Ansible 2.10+)
- 옛날: module을 *Ansible 본체*에 포함. *core release 종속*.
- 문제: 모듈 수천 개, *release cycle 폭증*, *벤더 module update 느림*
- 해결: **module을 *collection*으로 분리** → *독립 release cycle*

### Collection의 구성
```
my-org/
└── my_collection/
    ├── galaxy.yml          ← 메타데이터
    ├── plugins/
    │   ├── modules/        ← module
    │   ├── filters/        ← filter plugin
    │   ├── lookup/         ← lookup plugin
    │   ├── inventory/      ← inventory plugin
    │   └── ...
    ├── roles/              ← role
    ├── playbooks/          ← playbook
    ├── README.md
    └── docs/
```

### galaxy.yml
```yaml
namespace: my_org
name: my_collection
version: 1.0.0
readme: README.md
authors:
- Alice <alice@example.com>
description: Custom Ansible collection
license:
- MIT
tags:
- infrastructure
- aws
dependencies:
  amazon.aws: ">=5.0.0"
  community.general: ">=6.0.0"
repository: https://github.com/my-org/my_collection
```

### 빌드 & publish
```bash
ansible-galaxy collection build
# → my_org-my_collection-1.0.0.tar.gz

ansible-galaxy collection publish my_org-my_collection-1.0.0.tar.gz --api-key=$GALAXY_TOKEN
```

### 사용
```bash
ansible-galaxy collection install my_org.my_collection:>=1.0.0
ansible-galaxy collection install -r requirements.yml
```

```yaml
# requirements.yml
collections:
- name: amazon.aws
  version: ">=5.0.0"
- name: community.general
- name: my_org.my_collection
  source: https://galaxy.ansible.com
```

### Playbook에서
```yaml
- hosts: all
  tasks:
  - name: Use module from collection
    amazon.aws.ec2_instance:        # ← FQCN (Fully Qualified Collection Name)
      name: my-server
      ...
  
  - name: Use role from collection
    import_role:
      name: my_org.my_collection.nginx
```

### FQCN — *Fully Qualified Collection Name*
- `{namespace}.{collection}.{module/role}`
- 예: `amazon.aws.ec2_instance`, `community.general.archive`
- *모호함 제거*. 권장.

### Built-in collections
- `ansible.builtin` — *core module* (apt, copy, service, ...) — FQCN으로 `ansible.builtin.apt`. *short name도 작동* (자동으로 ansible.builtin)
- `ansible.posix` — POSIX 전용
- `ansible.windows` — Windows
- `amazon.aws`, `community.aws`, `google.cloud`, `azure.azcollection` — 클라우드
- `kubernetes.core` — K8s
- `community.general` — 잡다 module 모음
- `community.crypto` — 암호화

---

## 9. Galaxy — *role / collection 저장소*

### 사용
```bash
# 검색
ansible-galaxy search nginx

# role 설치
ansible-galaxy role install geerlingguy.nginx
# → ~/.ansible/roles/ 또는 roles_path

# collection 설치
ansible-galaxy collection install amazon.aws
# → ~/.ansible/collections/ 또는 collections_path

# requirements.yml로 한 번에
ansible-galaxy install -r requirements.yml
```

### requirements.yml — *의존성 lock*
```yaml
# requirements.yml
roles:
- name: geerlingguy.nginx
  version: "3.1.0"
- name: my-role
  src: https://github.com/my-org/my-role
  version: "main"
  scm: git

collections:
- name: amazon.aws
  version: ">=5.0.0,<6.0.0"
- name: community.general
- name: my_org.my_collection
  source: https://galaxy.example.com  # 사내 galaxy
```

### 사내 Galaxy
- Ansible Galaxy 외에 *Pulp Galaxy* / *AWX 내장* 으로 사내 hosting
- *팀 collection / role 공유* 표준

---

## 10. 자주 헷갈리는 것

### 10-1. *role 안에서 `templates/x.j2` *상대 경로 작동***
- `src: x.j2` 만 적으면 *role의 templates/ 에서 자동 검색*
- 절대 경로도 가능하나 *재사용성 깎임*

### 10-2. *role의 defaults는 *role 안에서도 *override 가능**
```yaml
# playbook에서
- hosts: web
  roles:
  - role: nginx
    vars:
      nginx_port: 9090         # ← role defaults override
```

### 10-3. *role dependency 가 *항상 먼저 실행**
- 의도와 다를 수 있음 — `dependencies` 명시 시 *현재 role 전에* 실행
- 순서 중요하면 *명시적 import_role* 사용

### 10-4. *Collection install 시 *namespace 충돌***
- 다른 collection이 같은 module 이름 — *FQCN으로 모호함 제거*

### 10-5. *옛 module이 collection으로 이동*
- Ansible 2.10에서 *core 분리* — `ec2_instance` → `amazon.aws.ec2_instance`
- 옛 playbook에 *FQCN 추가* 또는 *short name으로 작동 (deprecation warning)*

### 10-6. *role의 tasks/main.yml 너무 크면*
- 100+ task 한 파일 — 운영·디버깅 어려움
- *기능별 import_tasks*로 분리 권장

---

## 🎤 면접 빈출 Q&A

### Q1. Role 구조? 각 디렉터리의 역할?
> `defaults/` (기본 변수, 가장 낮은 우선순위), `vars/` (role 내부 변수, 거의 fixed), `tasks/main.yml` (메인 task, sub 파일은 `import_tasks`), `handlers/main.yml` (변경 시만 실행되는 handler), `templates/` (Jinja2, *상대 경로*로 src), `files/` (static 파일), `meta/main.yml` (메타데이터 + dependency), `tests/` (role 자체 테스트).

### Q2. defaults vs vars 차이? 언제 어디에?
> *defaults* = 가장 낮은 우선순위, *role 사용자가 쉽게 override*. *vars* = 높은 우선순위, *role 내부에서 상수처럼 사용*. **override 의도하는 값은 defaults에**, role 외부에 노출 안 하고 싶은 내부 상수는 vars에. 운영에선 *대부분 defaults*.

### Q3. import_role vs include_role 차이?
> `import_role` = *parse time* — playbook 로딩 시 task 추가. `include_role` = *runtime* — 실행 시점 평가. **`include_role`만 *loop·동적 평가* 지원**. 일반적으로 `import_role`, *조건부·loop 필요하면 `include_role`*. 비슷한 차이: `import_tasks` vs `include_tasks`.

### Q4. Collection은 무엇? Role과의 관계?
> *Ansible 2.10+의 배포 단위*. *role + module + plugin* 묶음. 옛날엔 모든 module이 Ansible 본체에 — release 종속·벤더 update 느림. Collection 도입 후 *독립 release cycle*. **FQCN (Fully Qualified Collection Name)**: `amazon.aws.ec2_instance` 형식. 권장. *role도 collection 안에 포함* 가능.

### Q5. Galaxy는 무엇? requirements.yml 의 역할?
> *Galaxy* = role/collection의 *중앙 저장소* (galaxy.ansible.com). *Docker Hub의 ansible 버전*. *requirements.yml*에 *외부 role/collection 의존성 명세* + version constraint. CI에서 `ansible-galaxy install -r requirements.yml` → *재현 가능한 의존성*. 사내는 *Pulp Galaxy* 또는 *AWX 내장*으로 호스팅.

### Q6. Role dependency는 어떻게 동작?
> `meta/main.yml`의 `dependencies` — *현재 role 실행 전*에 dependency role *순서대로* 실행. 중복 방지 (한 번만), 순환 X. **명시적 순서 제어 필요하면 *dependency 대신 import_role 명시*** 권장 (더 명확).

### Q7. ansible-galaxy로 *사내 role 배포* 하려면?
> (1) **공개 Galaxy** — `ansible-galaxy role import` + `ansible-galaxy collection publish`. (2) **사내 Pulp Galaxy** — 사내 hosted, *권한 관리·CI 통합*. (3) **git URL 직접** — requirements.yml에 `src: https://github.com/...` + `scm: git`. (4) **AWX/Tower 내장 collection management**.

---

## 🔗 Cross-reference

- **task / handler 동작** → [02-playbook-and-tasks.md](02-playbook-and-tasks.md)
- **변수 우선순위 (defaults vs vars)** → [03-variables-and-templating.md](03-variables-and-templating.md)
- **AWX/Tower로 role 관리·실행** → [05-execution-and-best-practices.md](05-execution-and-best-practices.md)
- **CI에서 ansible-galaxy install + lint** → [`cicd-gitops-basics/01`](../cicd-gitops-basics/01-ci-fundamentals.md)
- **Helm chart의 dependency와 비교** → [`helm-basics/05`](../helm-basics/05-repos-and-distribution.md) — *같은 발상*

---

## 📝 3줄 요약

1. *Role = task/handler/template/vars/defaults 묶음의 재사용 단위*. `defaults/`는 *override 쉬움*, `vars/`는 *fixed*. `templates/`·`files/`는 *상대 경로로 자동 검색*.
2. *Collection = role + module + plugin의 배포 단위* (2.10+). FQCN (`amazon.aws.ec2_instance`) 권장. *core module도 `ansible.builtin.*`*.
3. *Galaxy*에서 role/collection 공유. *requirements.yml*로 *재현 가능한 의존성*. 사내는 *Pulp Galaxy* / *AWX 내장*.

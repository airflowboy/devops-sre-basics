# 01 — Inventory & Modules

> **공식 근거:** [Working with Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/), [Module Index](https://docs.ansible.com/ansible/latest/collections/index_module.html)

---

## 🎯 한 문장

> **Inventory = *대상 서버의 목록 + 그룹 + 변수*. Module = *task가 실제로 호출하는 단위* (`apt`, `copy`, `service` 등 수천 개). Ansible은 *playbook의 각 task → module 결정 → SSH로 전송·실행* 흐름.**

---

## 1. Inventory — *대상 서버 명세*

### Static Inventory — INI 형식
```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com ansible_host=192.168.1.20 ansible_port=2222

[databases]
db1.example.com
db2.example.com

[production:children]      # ← group of groups
webservers
databases

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key
```

### Static Inventory — YAML 형식 (선호)
```yaml
# inventory.yml
all:
  vars:
    ansible_user: deploy
    ansible_ssh_private_key_file: ~/.ssh/deploy_key
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
          ansible_host: 192.168.1.20
          ansible_port: 2222
    databases:
      hosts:
        db1.example.com:
        db2.example.com:
    production:
      children:
        webservers:
        databases:
```

### 호스트 패턴 — playbook의 `hosts:` 필드
```yaml
- hosts: webservers           # 그룹
- hosts: web1.example.com     # 단일
- hosts: webservers:databases # 합집합 (둘 다)
- hosts: webservers:!web1*    # 차집합 (web1로 시작하는 제외)
- hosts: webservers:&production  # 교집합 (둘 다 속한 것)
- hosts: all                  # 모든 호스트
- hosts: ungrouped            # 그룹에 속하지 않은
```

---

## 2. Dynamic Inventory — *클라우드 자동 발견*

매번 *static inventory 수정*하기 싫다. 클라우드의 *EC2 / GCE / Azure* 인스턴스를 *자동으로*:

### AWS EC2 예시
```yaml
# inventory/aws_ec2.yml  ← 파일 이름이 *.aws_ec2.yml로 끝나야
plugin: amazon.aws.aws_ec2
regions:
- ap-northeast-2
filters:
  tag:Environment: production
keyed_groups:
- key: tags.Role
  prefix: role            # role_web, role_db 등 group 자동 생성
- key: placement.region
- key: instance_type
hostnames:
- tag:Name
- private-ip-address
```

```bash
# 확인
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# 실행
ansible -i inventory/aws_ec2.yml role_web -m ping
```

### 다른 dynamic inventory plugin
- `google.cloud.gcp_compute` (GCP)
- `azure.azcollection.azure_rm` (Azure)
- `kubernetes.core.k8s` (K8s Pod·Node)
- `consul`, `nomad` 등 service discovery

### Inventory plugin vs script
- *옛 방식*: bash/python *script*가 JSON 출력 → Ansible이 호출
- *현재*: *plugin 파일* (`.yml`) — 더 단순하고 cache 활용

---

## 3. Group_vars / Host_vars — *대상별 변수*

### 디렉터리 구조
```
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml             # 모든 호스트 공통
│   ├── webservers.yml      # webservers 그룹
│   └── databases.yml
└── host_vars/
    └── db1.example.com.yml # 특정 호스트
```

### 예시
```yaml
# group_vars/webservers.yml
nginx_worker_processes: auto
nginx_listen_port: 80

# group_vars/databases.yml
postgres_version: "15"
postgres_max_connections: 200

# host_vars/db1.example.com.yml
postgres_max_connections: 500   # host별 override
```

### 우선순위
- 더 *specific*한 게 우선
- *host_vars > group_vars > inventory file*
- 자세히 [03-variables-and-templating.md](03-variables-and-templating.md)

### Group 계층
- `all` (built-in, 모든 호스트)
- `production` (사용자 정의 그룹)
- 같은 호스트가 *여러 그룹*에 속할 수 있음
- group var 우선순위: *자식 그룹이 부모 그룹 override*

---

## 4. Ad-hoc Command — *playbook 없이 한 줄*

```bash
# 모든 호스트에 ping
ansible all -m ping

# 특정 그룹에 명령
ansible webservers -m shell -a "uptime"

# root로 실행
ansible all -m apt -a "name=nginx state=present" --become

# 한 줄로 파일 복사
ansible webservers -m copy -a "src=/etc/nginx.conf dest=/etc/nginx/nginx.conf"
```

→ 일회성 작업·디버깅에 유용. 운영에선 *playbook이 표준*.

---

## 5. Module — *task의 실행 단위*

`man ansible-doc` 또는 `ansible-doc -l`로 *수천 개 module 검색*.

### 자주 쓰는 module 30개

**System / Package**
- `apt` / `yum` / `dnf` / `package` — 패키지 설치 (`package` 는 OS 자동 감지)
- `service` / `systemd` — 서비스 시작·정지·enable
- `cron` — cron job
- `user` / `group` — user/group 생성·관리
- `authorized_key` — SSH key 등록
- `hostname` / `timezone`

**File**
- `copy` — 로컬 → remote 파일 복사
- `fetch` — remote → 로컬
- `file` — 파일/디렉터리 생성·삭제·권한·소유자·심볼릭 링크
- `template` — Jinja2 template 렌더링 후 복사
- `lineinfile` — 파일의 한 줄 수정·삭제
- `blockinfile` — 파일의 한 블록 수정
- `replace` — regex 기반 치환
- `stat` — 파일 메타데이터 조회
- `synchronize` — rsync wrapper
- `archive` / `unarchive` — 압축·해제

**Command**
- `command` — 명령 실행 (shell 없음, redirect/pipe X)
- `shell` — shell 통한 명령 (redirect/pipe O)
- `raw` — Python 없는 target 대상 (네트워크 장비, 부팅 직후 등)
- `script` — 로컬 스크립트 → remote 실행

**Network / URL**
- `uri` — HTTP 요청 (REST API)
- `get_url` — 파일 다운로드
- `git` — git repo clone·update

**Cloud (collection 필요)**
- `amazon.aws.ec2_instance`, `amazon.aws.s3_bucket`, ...
- `google.cloud.gcp_compute_instance`, ...
- `community.aws.elb_classic_lb`, ...

**Database**
- `community.postgresql.postgresql_db`, `..._user`, ...
- `community.mysql.mysql_db`, ...

**K8s**
- `kubernetes.core.k8s` — K8s 자원 manage (apply 같은 효과)
- `kubernetes.core.helm` — helm install/upgrade
- `kubernetes.core.k8s_info` — 자원 조회

### Module 종료 상태
모든 module은 *4가지 결과* 중 하나 반환:
- **ok** — 이미 원하는 상태, 변경 안 함
- **changed** — 변경됨 (멱등성의 핵심 신호)
- **failed** — 실패
- **skipped** — 조건 미충족 (when: false 등)

`ansible-playbook` 실행 마지막에 *집계*:
```
PLAY RECAP *********************************
web1   : ok=10  changed=2  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

---

## 6. Module의 *멱등성* — 어떻게?

각 module이 *내부적으로*:
1. *현재 상태 검사* (예: nginx 설치됨?)
2. *원하는 상태와 비교*
3. *다르면 변경*, *같으면 skip*
4. *결과 보고* — changed 여부

### 예: `apt` module
```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
```
- nginx 이미 설치 → `ok` (changed=false)
- 설치 안 됨 → 설치 후 `changed`

### 예: `file` module
```yaml
- name: Ensure dir exists
  file:
    path: /opt/myapp
    state: directory
    mode: '0755'
    owner: deploy
```
- 디렉터리 이미 있고 권한·소유자 맞음 → `ok`
- 디렉터리 없음 → 생성, `changed`
- 디렉터리 있는데 권한 다름 → 권한 변경, `changed`

### *멱등성 안 보장* module
- **shell** / **command** — *Ansible이 결과 모름*
- **raw**
- → *항상 `changed_when`·`creates`·`removes`* 로 멱등성 표현 필요:

```yaml
- name: Initialize DB
  shell: /opt/myapp/init-db.sh
  args:
    creates: /var/lib/myapp/.initialized       # ← 이 파일 있으면 skip
```

또는
```yaml
- name: Run migration
  shell: /opt/myapp/migrate
  register: result
  changed_when: "'migrated' in result.stdout"
```

---

## 7. Connection — *어떻게 target에 접근*

기본은 `ssh`. 다른 옵션:

| Connection | 용도 |
|---|---|
| `ssh` (기본) | 대부분의 Linux 서버 |
| `local` | control node 자체에 실행 |
| `paramiko` | 옛 Python SSH (호환성 issue 있을 때) |
| `docker` | 실행 중인 컨테이너 내부 |
| `kubectl` | K8s Pod 내부 |
| `winrm` | Windows |
| `network_cli` | 네트워크 장비 (Cisco·Juniper 등) |

```yaml
- hosts: localhost
  connection: local            # SSH 없이 control node 자체에 실행
  tasks: ...
```

```ini
[mycontainers]
my-container ansible_connection=docker
```

---

## 8. SSH 동작 — *진짜 안에서 일어나는 일*

```
1. Control node가 module의 Python 코드 + 인자를 *임시 파일로 묶음*
2. SSH로 target에 *그 파일 전송* (보통 ~/.ansible/tmp/)
3. SSH로 target에서 *Python 실행*
4. Python이 module 실행 → JSON 결과 stdout
5. Control node가 stdout 회수 → 결과 보고
6. 임시 파일 정리
```

### ControlPersist — SSH 연결 재사용
- 같은 host에 *여러 task 실행*하면 *매번 SSH handshake* → 느림
- `ControlPersist` 옵션으로 *연결 유지*
- `ansible.cfg`:
```ini
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True              # SCP 없이 stdin으로 module 전송 (빠름)
```

→ 대규모 인프라에선 *pipelining 활성화 필수* — 5~10배 속도.

---

## 9. ansible.cfg — *설정 파일*

```ini
[defaults]
inventory = ./inventory
host_key_checking = False        # 첫 SSH host fingerprint prompt 안 함 (CI에서 유용)
remote_user = deploy
private_key_file = ~/.ssh/deploy_key
roles_path = ./roles
collections_path = ./collections
forks = 50                       # 병렬 호스트 수
gathering = smart                # facts 수집: smart(캐시) / explicit / implicit
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 3600
stdout_callback = yaml           # 출력 형식

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ServerAliveInterval=10
pipelining = True
control_path = ~/.ansible/cp/%%h-%%r

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

### 설정 검색 순서
1. `ANSIBLE_CONFIG` 환경변수
2. 현재 디렉터리 `./ansible.cfg`
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

---

## 10. 자주 헷갈리는 것

### 10-1. *inventory 그룹 이름에 `-` 못 씀*
- INI: `[web-servers]` → fail
- *underscore* 사용: `[web_servers]`

### 10-2. *inventory에 *변수 너무 많이* 박음*
- *복잡한 변수는 group_vars / host_vars 파일로 분리*
- inventory 파일 자체는 *간결하게*

### 10-3. *shell/command module의 *redirect 동작 다름**
```yaml
- shell: "ls /tmp > /tmp/listing.txt"      # 작동
- command: "ls /tmp > /tmp/listing.txt"    # > 가 인자로 해석됨 → fail
```
- `shell`은 shell 통과 (`/bin/sh -c "..."`), `command`는 직접 exec

### 10-4. *Dynamic inventory가 *느림**
- 매번 클라우드 API 호출 → 수십 초
- `cache=True` + `cache_timeout` 설정으로 *결과 캐시*

### 10-5. *control node에서 *local 실행* 잊음*
```yaml
- hosts: localhost            # ← 잊기 쉬움
  connection: local
  tasks:
  - name: Create S3 bucket
    amazon.aws.s3_bucket: ...
```
- 클라우드 API 호출은 *target에서 할 일 X* — control node에서.

### 10-6. *ad-hoc command에 sudo 안 됨*
```bash
ansible all -m apt -a "name=nginx state=present"            # ← 권한 부족 가능
ansible all -m apt -a "name=nginx state=present" --become   # ← OK
```

---

## 🎤 면접 빈출 Q&A

### Q1. Ansible의 *agentless*가 무슨 의미?
> target 서버에 *daemon 설치 안 함* (Chef/Puppet의 agent 대비). *SSH + Python 3* 만 있으면 작동. Control node가 *module 코드를 SSH로 전송 → target에서 Python 실행 → 결과 JSON 회수*. 장점: *어디서나 빠르게 시작*, *서버에 영구 설치 0*. 단점: *대규모 병렬 시 SSH 부하* — pipelining + ControlPersist로 완화.

### Q2. Static inventory vs Dynamic inventory?
> **Static**: INI 또는 YAML 파일에 *수동 명세*. 작은 인프라·간단. **Dynamic**: *클라우드 API에서 자동 발견* (EC2·GCE·K8s 등). plugin 형태. tag 기반으로 *자동 grouping*. 새 인스턴스 추가 시 *수동 갱신 없이* 자동 포함. 대규모·동적 환경 필수.

### Q3. Inventory의 group, group_vars, host_vars 우선순위?
> *더 specific 한 게 우선*. 우선순위 (낮음→높음): `all` 그룹 vars → 부모 그룹 vars → 자식 그룹 vars → host_vars → playbook vars → role/task vars → extra vars(`-e`). 같은 호스트가 *여러 그룹*에 속하면 *자식 그룹이 부모 override*. 전체 22 layer는 [03 참조](03-variables-and-templating.md).

### Q4. 멱등성은 *어떻게 보장*되나? `shell` module도 보장됨?
> 각 *module이 *현재 상태 검사 후 변경 시에만 적용**. `apt`는 이미 설치되면 skip, `file`은 권한 같으면 skip. **`shell`/`command`는 *멱등성 안 보장*** — Ansible이 결과 모름. 해결: `creates:`/`removes:` 인자 (특정 파일 있으면 skip), 또는 `changed_when:` 으로 *조건부 changed* 명시.

### Q5. ad-hoc command와 playbook의 차이?
> **ad-hoc**: `ansible {hosts} -m {module} -a "{args}"` 한 줄 — 일회성·디버깅. **playbook**: YAML 파일에 *여러 task 명세*, *재사용·버전 관리* 가능. 운영은 *playbook이 표준*, ad-hoc은 *조사·임시*.

### Q6. SSH 연결을 *최적화*하려면?
> (1) **ControlPersist** — 같은 host에 *연결 유지·재사용* (handshake 비용 줄임). (2) **pipelining = True** — SCP 없이 *stdin으로 module 전송* (5~10배 속도). (3) **forks** 조정 — 병렬 host 수 (기본 5, 대규모는 50~100). (4) *fact_caching* — gather_facts 캐시. (5) *raw connection* 회피 — Python 있는 한 일반 module이 빠름.

### Q7. shell module은 언제 OK·언제 피해야?
> *피해야*: 멱등성 안 됨, *변경 추적·rollback 어려움*. 대체 가능하면 *전용 module* (`apt`, `service`, `lineinfile` 등). *OK*: 전용 module이 없는 *특수한 명령* (예: 회사 CLI 도구). 단 *`creates:`/`changed_when:`* 으로 멱등성 표현 필수. *raw*는 Python 없는 target (네트워크 장비, 부팅 직후)에만.

---

## 🔗 Cross-reference

- **Playbook 작성** → [02-playbook-and-tasks.md](02-playbook-and-tasks.md)
- **변수 우선순위 22단계** → [03-variables-and-templating.md](03-variables-and-templating.md)
- **Role·Collection** → [04-roles-and-collections.md](04-roles-and-collections.md)
- **AWX/Tower·CI 통합** → [05-execution-and-best-practices.md](05-execution-and-best-practices.md)
- **SSH 자체** → [`linux-basics/01-systemd.md`](../linux-basics/01-systemd.md) (systemd로 sshd 관리)
- **Terraform과 조합** → `terraform-basics/` (예정)

---

## 📝 3줄 요약

1. *Inventory* = 대상 + 그룹 + 변수. Static (INI/YAML) vs Dynamic (클라우드 API plugin). group_vars/host_vars로 *대상별 변수 분리*.
2. *Module* = task 실행 단위. apt/copy/template/service/lineinfile 등 *수천 개*. 각 module이 *멱등성 보장* — *shell/command는 예외* → `creates:` / `changed_when:` 필수.
3. SSH + Python으로 작동. *pipelining + ControlPersist + forks* 가 *성능 핵심*. `ansible.cfg`로 한 번 설정.

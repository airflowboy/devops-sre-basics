# Linux — DevOps/SRE가 *진짜로* 알아야 하는 것들

> *"docker는 결국 Linux 프로세스다", "K8s는 결국 Linux 호스트 위에 도는 컨테이너다", "장애 디버깅의 90%가 결국 Linux"*. 이 폴더는 *그 베이스*를 다잡는 곳.

---

## 🎯 한 문장 큰 그림

> **Linux = *프로세스 + 파일시스템 + 네트워크 + 권한*의 4가지 추상화 위에서 *systemd가 서비스 관리*하고 *cgroup·namespace가 격리*를 제공하는 OS.**

DevOps 도구 거의 전부가 *이 위에 얹혀 있음*. 도구가 마법처럼 보일 때 *베이스로 내려가면* 보통 풀린다.

---

## 🧩 핵심 5영역 (한 페이지 요약)

| 영역 | 핵심 개념 | 자주 만지는 명령 |
|---|---|---|
| **systemd** | unit, service, timer, journal | `systemctl`, `journalctl` |
| **프로세스·시그널** | fork/exec, PID, signal, 좀비, OOM | `ps`, `top`, `htop`, `kill`, `strace` |
| **파일시스템·권한** | inode, mount, ACL, UID/GID, capabilities | `ls -l`, `stat`, `mount`, `df`, `du`, `chmod` |
| **cgroup·namespace** | 격리·자원 한도. *컨테이너의 베이스* | `lsns`, `nsenter`, `systemd-cgls` |
| **패키지·서비스 운영** | dnf/apt, service 관리, 부팅 흐름, journal | `dnf`, `apt`, `systemctl`, `journalctl` |

각 영역의 깊이는 토픽 파일.

---

## 🌪 자주 헷갈리는 6가지

### 1. *Linux는 모든 게 파일* — 실제 의미
- 디스크도 파일 (`/dev/sda1`)
- 네트워크 소켓도 파일 (`/proc/net/tcp`)
- 프로세스 정보도 파일 (`/proc/{pid}/...`)
- 디바이스도 파일 (`/dev/null`, `/dev/random`)
- → *fd (file descriptor)*가 핵심 추상화. `lsof`로 모든 열린 file descriptor 추적.

### 2. *PID 1*은 *진짜 init*
- PID 1은 *고아 프로세스 부모 인계 + 시그널 처리 + 좀비 회수*
- 컨테이너 안 *PID 1이 잘못되면* (shell script로 띄움) graceful shutdown 깨짐
- → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 의 *PID 1 함정*과 직결

### 3. *uid 0 = root* — 정확히 그게 다임
- root의 정체는 *그냥 uid 0*. 이름 'root'는 관습.
- *컨테이너 안 root* + (USER namespace 없으면) = *호스트의 root와 동일 uid*
- → [`docker-basics/06`](../docker-basics/06-security.md) 의 보안 함정

### 4. *systemd는 init 그 이상*
- 단순 *init system* 아님. *서비스 관리 + 의존성 + 소켓 활성화 + cgroup 관리 + journal + timer + ...* 다 함
- *systemd 모르면 RHEL/Ubuntu 운영 못 함*

### 5. *Load Average*는 *CPU 사용률이 아니다*
- *runnable + uninterruptible sleep 프로세스 수*의 평균
- D-state (I/O 대기)도 포함 → *CPU는 한가한데 LA 높음* 정상
- → [02-process-and-signals.md](02-process-and-signals.md)

### 6. *systemd journal vs syslog*
- 옛날: rsyslog (`/var/log/messages`)
- 현대: journald (`journalctl` 명령). binary log. 메타데이터 풍부.
- 보통 *journald 기본*, 필요시 rsyslog로 *외부 syslog server 전송*

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 |
|---|---|
| [01-systemd.md](01-systemd.md) | unit·service·timer·journal·target, 부팅 흐름, dependency |
| [02-process-and-signals.md](02-process-and-signals.md) | fork/exec, signal, PID 1, 좀비, OOM Killer, ps/top/htop, strace |
| [03-filesystem-and-permissions.md](03-filesystem-and-permissions.md) | mount, inode, UID/GID/ACL/capabilities, suid, /proc·/sys |
| [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md) | 7 namespace, cgroup v2, *컨테이너 격리의 베이스* |
| [05-package-and-troubleshooting.md](05-package-and-troubleshooting.md) | dnf/apt, journal, *장애 디버깅 12단계 체크리스트* |

---

## 🔗 Cross-reference

- **cgroup·namespace 깊이** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md). 이 폴더의 04는 *Linux 일반*, docker-basics/02는 *컨테이너 응용*.
- **네트워크 (ip route, iptables, NAT)** → [`network-basics/`](../network-basics/)
- **K8s node = Linux 호스트** → [`kubernetes-basics/01`](../kubernetes-basics/01-architecture.md)
- **firewalld vs ufw, SELinux** → 02·03에 포함. K8s NetworkPolicy와 비교는 *상위 추상화*.

---

## 🎤 면접 빈출 (전체 폴더 횡단)

| 질문 | 답이 있는 파일 |
|---|---|
| `ps aux`에서 STAT 컬럼 의미? | [02](02-process-and-signals.md) |
| 좀비 프로세스는 무엇이고 *왜* 생기나? | [02](02-process-and-signals.md) |
| OOM Killer는 어떻게 *어떤 프로세스를 죽일지* 결정? | [02](02-process-and-signals.md) |
| Load Average가 의미하는 것? CPU 사용률과 다른 이유? | [02](02-process-and-signals.md) |
| systemd unit의 종류와 dependency 방식? | [01](01-systemd.md) |
| journalctl로 *특정 unit의 *어제 4시* 로그 보기? | [01](01-systemd.md) |
| chmod 755·644 의미? | [03](03-filesystem-and-permissions.md) |
| Capability (CAP_NET_BIND_SERVICE 등) 와 root의 관계? | [03](03-filesystem-and-permissions.md) |
| Linux namespace 7종? | [04](04-cgroup-and-namespace.md) |
| 서버가 *갑자기 느려졌을 때* 디버깅 순서? | [05](05-package-and-troubleshooting.md) |

---

## 📐 학습 순서 권장

1. **02 process-and-signals** — *디버깅의 베이스*. 모든 장애 출발점.
2. **01 systemd** — *서비스 관리의 표준*. 안 알면 운영 불가.
3. **03 filesystem-and-permissions** — *권한 사고의 99%*.
4. **04 cgroup-and-namespace** — *컨테이너 이해의 베이스*. [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 의 전제.
5. **05 package-and-troubleshooting** — *체크리스트*. 실전 디버깅.

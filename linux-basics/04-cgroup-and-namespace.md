# 04 — cgroup & namespace: 컨테이너의 *진짜 베이스*

> **공식 근거:** `man 7 cgroups`, `man 7 namespaces`, [Kernel cgroup v2 docs](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
>
> **선행:** [02-process-and-signals.md](02-process-and-signals.md)
>
> **연결:** [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 가 *컨테이너 응용 편*. 이 파일은 *Linux 일반*.

---

## 🎯 한 문장

> **cgroup = *프로세스 그룹의 자원 한도* (CPU·메모리·I/O·PID 수). namespace = *프로세스 그룹의 시야 격리* (PID·NET·MNT·UTS·IPC·USER·CGROUP). 둘 다 Linux 커널 기능. *Docker/K8s는 이 둘의 포장지*.**

---

## 1. Cgroup — *Control Group*

### 발상
- 옛 Linux: *프로세스마다 nice·rlimit*은 있으나 *그룹 단위 자원 관리 부족*
- 해결: *프로세스들을 트리 구조의 cgroup에 묶고*, *cgroup 단위로 한도 적용*

### v1 vs v2

| | v1 | v2 |
|---|---|---|
| 계층 | *컨트롤러마다 별도 트리* (cpu, memory, blkio 따로) | *단일 통합 트리* |
| 일관성 | 컨트롤러 간 정책 불일치 가능 | 일관 |
| 사용 환경 | RHEL 7, 옛 K8s | RHEL 9+, K8s 1.25+, systemd 기본 |

**현재는 v2가 표준.** 이하 v2 기준.

### 구조
```
/sys/fs/cgroup/                    ← 루트
├── cgroup.controllers              ← 활성화된 컨트롤러 목록
├── cgroup.procs                    ← 이 cgroup의 PID들
├── memory.max                      ← 최대 메모리
├── cpu.max                         ← CPU quota
├── system.slice/                   ← 시스템 service들
│   ├── nginx.service/
│   │   ├── cgroup.procs
│   │   ├── memory.max
│   │   └── ...
│   └── ...
└── user.slice/                     ← user session들
    └── user-1000.slice/
```

systemd가 *모든 service를 자동으로 cgroup에 배치* → `systemd-cgls`로 트리 보기.

### 컨트롤러 (v2)

| 컨트롤러 | 무엇 제한 |
|---|---|
| **cpu** | CPU 시간 (quota, weight) |
| **memory** | 메모리 (max, high, swap) |
| **io** | block I/O (weight, latency) |
| **pids** | PID 수 (fork bomb 방어) |
| **cpuset** | CPU 집합 (어느 코어에만 실행) |
| **hugetlb** | huge page |
| **rdma** | RDMA 자원 |
| **misc** | 기타 |

### 자주 쓰는 설정

```bash
# memory.max = 512M
echo "536870912" > /sys/fs/cgroup/myapp/memory.max

# CPU 200% (= 2 core 분량)
echo "200000 100000" > /sys/fs/cgroup/myapp/cpu.max
# 형식: "quota period". 200000us / 100000us = 200%

# PID 최대 100
echo "100" > /sys/fs/cgroup/myapp/pids.max

# 프로세스를 cgroup에 추가
echo "$PID" > /sys/fs/cgroup/myapp/cgroup.procs
```

### systemd로 더 쉽게
```bash
systemd-run --slice=test --property=MemoryMax=100M sleep 1000
# 자동으로 cgroup 만들고 sleep을 그 안에 배치
```

또는 service unit에:
```ini
[Service]
MemoryMax=512M
CPUQuota=200%
TasksMax=100
IOWeight=500
```

### `systemd-cgtop` — *실시간 cgroup 자원 사용*
```bash
systemd-cgtop
# Control Group                          Tasks   %CPU   Memory  Input/s Output/s
# /                                      234    -      8.5G     -       -
# system.slice/nginx.service             4      12.3   23.4M    -       -
# system.slice/postgresql.service        15     45.1   1.2G     -       -
# ...
```

→ 어느 service가 *자원 많이 먹나* 즉시 가시화.

---

## 2. Memory cgroup — *OOM과의 관계*

### `memory.max`
- 초과 시 *그 cgroup 안 프로세스에 OOM Killer*
- 시스템 전체 OOM과 다름 (그 cgroup만)
- *시스템 전체 메모리는 남아도* cgroup 한도 초과면 OOM

### `memory.high`
- *soft limit* — 초과 시 *throttle* (강제 reclaim, 응답 느려짐)
- *OOM은 발생 안 함*
- → "*OOM 피하면서도 메모리 압박*" 표현

### `memory.swap.max`
- swap 사용 한도

### *cgroup OOM 로그*
```bash
dmesg | grep -i oom
# Memory cgroup out of memory: Killed process 1234 (java) ...
# ↑ 시스템 OOM이 아니라 *그 cgroup*의 OOM
```

→ K8s에서 *Pod OOMKilled* 가 *시스템 메모리 남는데도 발생*하는 이유.

---

## 3. CPU cgroup — *throttle 동작*

### `cpu.max`
- 형식: `{quota} {period}` (μs)
- *period 시간 동안 quota만큼만 사용*
- `100000 100000` = 1 core 100%
- `200000 100000` = 2 core 분량
- `50000 100000` = 0.5 core 분량

### CPU throttling
- quota 초과 시 *대기*. *period 끝나면 다시*.
- → *CPU 사용량 톱니*. 응답 latency *bumpy*.
- *진짜 CPU 부족* vs *cgroup throttle*인지 구분 필요

확인:
```bash
cat /sys/fs/cgroup/myapp/cpu.stat
# usage_usec ...
# user_usec ...
# system_usec ...
# nr_periods ...
# nr_throttled ...   ← *throttle 발생 횟수*
# throttled_usec ... ← throttle된 총 시간
```

→ K8s에서 *CPU limit으로 인한 throttling이 자주 사고 원인*. Node·Java 같은 *런타임이 cgroup-aware 아닐 때* 흔함.

---

## 4. Namespace — *시야 격리*

`man 7 namespaces`:
> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance.

### 7가지 namespace

| Namespace | 무엇 격리? | 예 |
|---|---|---|
| **PID** | 프로세스 ID 트리 | 컨테이너 안 첫 프로세스가 PID 1 |
| **NET** | 네트워크 (interface, route, iptables, port) | 자기만의 lo, 자기만의 eth0 |
| **MNT** | mount table | 자기만의 / (rootfs) |
| **UTS** | hostname, domain | 자기만의 hostname |
| **IPC** | System V IPC, POSIX 메시지 큐 | 다른 컨테이너의 shared memory 안 보임 |
| **USER** | uid/gid 매핑 | 컨테이너 root가 호스트 일반 user에 매핑 |
| **CGROUP** | cgroup 트리의 가시성 | 자기 cgroup만 보임 |

### 직접 만들어보기 — `unshare`

```bash
# 새 PID + UTS + MNT namespace에서 bash
sudo unshare --pid --uts --mount --fork bash

# 안에서:
hostname myhost            # ← 호스트 영향 X (UTS namespace)
ps aux                     # ← bash가 PID 1 (PID namespace)
mount --bind /tmp /opt     # ← 호스트 영향 X (MNT namespace)
```

### 현재 namespace 확인

```bash
# 프로세스의 namespace
ls -l /proc/{pid}/ns/
# lrwxrwxrwx 1 root root 0 May 26 12:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 ipc    -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 mnt    -> 'mnt:[4026531840]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 net    -> 'net:[4026531992]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 pid    -> 'pid:[4026531836]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 user   -> 'user:[4026531837]'
# lrwxrwxrwx 1 root root 0 May 26 12:00 uts    -> 'uts:[4026531838]'

# 모든 namespace 목록
lsns
```

같은 namespace inode → 같은 namespace. 다르면 다른.

### `nsenter` — 다른 프로세스의 namespace에 진입

```bash
# 컨테이너 안에 들어가 보기 (docker exec 없이)
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
nsenter -t $PID --pid --mount --uts --ipc --net -- bash
```

→ 디버깅 시 *root에서 컨테이너 내부 보기*에 유용.

---

## 5. PID Namespace 깊이 — *PID 1의 책임*

### 컨테이너 안 PID 1
- 첫 프로세스가 *항상 PID 1*
- *고아 프로세스 인계* — namespace 안에서 부모 잃은 자식의 새 부모
- *시그널 처리* — 컨테이너에 SIGTERM 가면 PID 1으로
- *좀비 회수* — wait()

### PID 1 죽으면 — namespace 전체 정리
- PID 1 종료 → *그 PID namespace 안 모든 프로세스 강제 종료*
- → 컨테이너의 *주 프로세스 죽으면* 컨테이너 *전체 종료*

### 함정: shell script로 PID 1
```bash
CMD ["./start.sh"]
# start.sh:
#   nginx &      ← 백그라운드
#   wait         ← 안 쓰면 좀비 + 시그널 안 됨
```
→ bash가 PID 1, nginx는 *그 자식*. SIGTERM이 bash에 와도 nginx에 *안 전파*.
→ 해결:
- `exec nginx` (bash 자리를 nginx가 교체)
- `tini` / `dumb-init` 같은 *진짜 init*
- Dockerfile `ENTRYPOINT ["nginx", "..."]` (exec form)

→ [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) Section 3 더 자세히.

---

## 6. NET Namespace 깊이

### 자기만의 네트워크 스택
- *interface 목록·라우팅 테이블·iptables 룰·소켓 풀* 다 독립
- *호스트와 분리* — 호스트의 eth0이 안 보임

### 호스트와 연결하는 방법 — veth pair
- *가상 케이블* (양 끝이 연결된 interface 한 쌍)
- 한 쪽은 호스트 namespace, 한 쪽은 *컨테이너 namespace*
- 호스트 쪽 veth를 *bridge* (예: docker0)에 attach

### 직접 만들어보기
```bash
# 새 net namespace
ip netns add myns

# veth pair 생성
ip link add veth0 type veth peer name veth1

# veth1을 myns로 이동
ip link set veth1 netns myns

# 호스트 쪽 veth0 up
ip link set veth0 up
ip addr add 10.0.0.1/24 dev veth0

# myns 안에서 veth1 셋업
ip netns exec myns ip link set veth1 up
ip netns exec myns ip addr add 10.0.0.2/24 dev veth1

# 통신 테스트
ping 10.0.0.2                          # 호스트 → myns
ip netns exec myns ping 10.0.0.1       # myns → 호스트
```

→ docker가 *내부적으로 하는 일*의 raw 버전. [`docker-basics/03`](../docker-basics/03-networking.md) Section 3 흐름 그대로.

---

## 7. USER Namespace — Rootless의 베이스

### 발상
- 컨테이너 안 root (uid 0) ← → 호스트의 일반 user (uid 100000) 매핑
- 컨테이너 안에선 *root처럼 보이나*, 호스트에선 *권한 없는 user*
- → 컨테이너 탈출해도 *호스트는 일반 user*

### 매핑 예시
```
컨테이너 uid 0    →  호스트 uid 100000
컨테이너 uid 1    →  호스트 uid 100001
...
컨테이너 uid 65535 →  호스트 uid 165535
```

### Rootless Docker / Podman
- *docker daemon 자체를 일반 user로* 실행
- 컨테이너 안 root = 호스트 일반 user
- 보안 강함. 단 일부 제약 (host port < 1024 안 됨 등)

### K8s에서
- *기본은 USER namespace 안 씀* (각 컨테이너 root = 호스트 root)
- *runAsNonRoot: true* 로 *non-root user* 강제 (이미지 자체가 USER 명시 필요)
- K8s 1.25+ alpha로 *Pod의 USER namespace* 도입 (점진 보급)

→ [`docker-basics/06`](../docker-basics/06-security.md), [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) 와 직결.

---

## 8. 컨테이너 = namespace + cgroup + rootfs

`docker run` 했을 때 일어나는 일 (커널 관점):

```
1. fork() — 새 프로세스
2. clone() with CLONE_NEW* flags — 새 namespace 7개
3. cgroup 생성 + 새 프로세스 attach + 한도 적용
4. pivot_root or chroot — rootfs 변경 (이미지의 OverlayFS merged 디렉터리)
5. setuid/setgid — USER 지시어 적용
6. execve() — 컨테이너 안 첫 명령 실행
```

→ docker는 이 모든 걸 *자동화*한 wrapper. K8s는 그 *위의 또 다른 wrapper*.

---

## 9. *cgroup-aware 안 한 앱*의 함정

### JVM
- 옛 JVM (Java 8 이전): *호스트 메모리 기준*으로 힙 잡음
- cgroup memory.max가 *512M*인데 JVM이 *호스트 32G 기준 8G 힙*
- → 메모리 초과 → cgroup OOM Kill
- 해결:
  - JDK 11+ — cgroup-aware (기본)
  - 옵션 명시: `-XX:MaxRAMPercentage=75.0`, `-Xmx 명시`

### Node.js
- `os.cpus().length` 가 *호스트 CPU 수* 반환
- cgroup CPU limit 무시한 *worker pool 크기*
- → throttling 폭증
- 해결: `--max-old-space-size`, *cgroup-aware libuv* (Node 16+ 일부)

### 일반적 함정
- *호스트 자원 기준으로 thread pool·캐시·버퍼* 설정한 앱이 cgroup 안에선 *과도 사용*
- → 모든 런타임에 *cgroup 인식 옵션* 명시 권장

---

## 10. 자주 헷갈리는 것

### 10-1. *systemd가 cgroup 관리* — 직접 만지면 충돌
- systemd는 *자기가 만든 cgroup만 관리*
- 외부에서 cgroup 만들고 *systemd가 reset*하면 사라짐
- → `systemd-run` 또는 *service unit*으로 만들기

### 10-2. *cgroup v1·v2 혼용*
- 일부 컨트롤러는 v1, 일부는 v2 — *unified vs hybrid 모드*
- K8s 1.25+ 는 *v2 전용 권장*
- containerd 등 *런타임이 어느 v 지원*하는지 확인

### 10-3. *namespace에서 *호스트 볼 수 있는 경우**
- `--pid=host` — 호스트 PID namespace 공유 (`ps`에 호스트 프로세스 다 보임)
- `--net=host` — 호스트 NET namespace 공유 (호스트 port 직접 사용)
- → 격리 약해짐. *모니터링 에이전트*에만 정당화.

### 10-4. *unshare로 namespace 만들어도 root 권한*
- USER namespace 없이 다른 namespace 만들려면 *root 필요* (CAP_SYS_ADMIN)
- USER namespace 통해서면 *unprivileged user*도 가능 (rootless 컨테이너의 베이스)

### 10-5. *cgroup hierarchy 정리 안 됨*
- 프로세스 다 죽어도 *cgroup 디렉터리는 남음*
- systemd가 보통 알아서 정리하나, *수동 cgroup은 직접 rmdir 필요*

---

## 🎤 면접 빈출 Q&A

### Q1. Linux namespace 7종 설명?
> **PID** (프로세스 ID 트리), **NET** (네트워크 스택), **MNT** (마운트 트리), **UTS** (hostname), **IPC** (System V IPC), **USER** (uid/gid 매핑), **CGROUP** (cgroup 트리 가시성). 각각 *전역 자원의 자기만의 인스턴스를 가진 것처럼 보이게* 만듦. *컨테이너 = 이 7개 namespace + cgroup + rootfs 적용된 Linux 프로세스*.

### Q2. cgroup v1과 v2 차이? 무엇이 표준?
> v1: *컨트롤러마다 별도 트리* (cpu, memory, blkio 따로). 일관성 약함. v2: *단일 통합 트리* — 모든 컨트롤러가 같은 계층. 정책 일관됨. **v2가 현재 표준** (RHEL 9+, K8s 1.25+, systemd 기본). 일부 환경은 hybrid 모드. K8s는 v2 전용 권장.

### Q3. *cgroup OOM*과 *시스템 OOM* 차이?
> *시스템 OOM*: 전체 메모리 + swap 고갈 시 OOM Killer가 점수 기반으로 누군가 죽임. *cgroup OOM*: *그 cgroup의 memory.max 초과* 시 *그 cgroup 안에서만* OOM Killer. 시스템 메모리 남아도 발생 — K8s Pod OOMKilled가 이 경우 흔함. `dmesg | grep oom`에 *cgroup path* 표시.

### Q4. CPU throttling이 왜 문제? 어떻게 진단?
> cgroup `cpu.max` 초과 시 *quota 다 쓰면 period 끝까지 대기* → 응답 latency *bumpy*. *진짜 CPU 부족이 아니라 *cgroup 제한*. `cat /sys/fs/cgroup/.../cpu.stat`의 `nr_throttled`, `throttled_usec`로 확인. Node·Java가 *cgroup-aware 아닐 때* 흔함.

### Q5. JVM 컨테이너의 *OOMKilled 잦음* — 의심해야 할 것?
> JVM이 *호스트 메모리 기준*으로 힙 잡음 (옛 JDK). cgroup이 512M인데 *호스트 32G 기준 8G 힙* → cgroup OOM. 해결: (1) **JDK 11+** (cgroup-aware 기본), (2) `-XX:MaxRAMPercentage=75.0`, (3) `-Xmx` 명시. Node.js도 비슷한 함정 (cpu count, max-old-space-size).

### Q6. `unshare`로 직접 namespace 만들어보기?
> `unshare --pid --uts --mount --net --fork bash` — 새 PID·UTS·MNT·NET namespace에서 bash. 안에서 `hostname myhost` 해도 호스트 영향 X (UTS 격리), `ps` 하면 bash가 PID 1 (PID 격리). *컨테이너의 raw 버전*. Docker가 자동화한 일을 *수동으로* 해보면 *마법이 아니게* 됨.

### Q7. USER namespace는 왜 *rootless 컨테이너의 베이스*?
> 컨테이너 안 uid 0 (root) ← → 호스트 uid 100000 (일반 user) 매핑. 컨테이너 안에선 root처럼 보이나, *호스트에선 권한 없음*. → 컨테이너 탈출해도 *호스트는 일반 user*. Podman 기본 사용, Docker는 옵션. K8s 1.25+ alpha로 Pod 차원 도입 중.

---

## 🔗 Cross-reference

- **컨테이너 응용 (Docker의 namespace·cgroup 활용)** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)
- **컨테이너 네트워킹 (veth/bridge/iptables)** → [`docker-basics/03`](../docker-basics/03-networking.md), [`network-basics/`](../network-basics/)
- **K8s Pod = namespace 공유 컨테이너 묶음** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **K8s OOMKilled / CPU throttling 디버깅** → [`kubernetes-basics/05`](../kubernetes-basics/05-scheduling.md)
- **systemd가 service별 cgroup 관리** → [01-systemd.md](01-systemd.md)
- **컨테이너 보안 (capability, USER namespace, seccomp)** → [`docker-basics/06`](../docker-basics/06-security.md)

---

## 📝 3줄 요약

1. cgroup = *프로세스 그룹의 자원 한도* (CPU·메모리·I/O·PID). v2 표준. namespace = *시야 격리* (PID·NET·MNT·UTS·IPC·USER·CGROUP).
2. *컨테이너 = namespace + cgroup + rootfs 적용된 Linux 프로세스*. Docker/K8s는 포장지. `unshare`로 raw 버전 직접 만들어볼 수 있음.
3. *cgroup-aware 안 한 앱 (옛 JVM, Node)* 이 *cgroup OOM·CPU throttling*의 흔한 원인. 런타임 옵션 (`MaxRAMPercentage` 등) 명시 필수.

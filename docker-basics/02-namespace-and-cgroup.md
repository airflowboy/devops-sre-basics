# 02 — Namespace와 Cgroup: 컨테이너는 어떻게 격리되는가

> **공식 근거:** Linux `man 7 namespaces`, `man 7 cgroups`, [Kernel docs — Control Groups v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
>
> **이 파일이 이 폴더에서 가장 중요하다.** Docker / Podman / containerd 모두 *이 두 가지 커널 기능의 포장지*다. 이걸 이해하면 컨테이너 90%가 풀린다.

---

## 🎯 한 문장

> **컨테이너 = (namespace로 *시야* 격리) + (cgroup으로 *자원* 한도) + (rootfs로 *파일시스템* 바꿔치기) 된 *그냥 Linux 프로세스*.**

---

## 1. 가상화 ≠ 컨테이너 — 결정적 차이

```
[ Virtual Machine ]                [ Container ]
┌──────────────────┐              ┌──────────────────┐
│   App            │              │   App            │
│   Bin / Libs     │              │   Bin / Libs     │
│   Guest OS       │              │   (호스트 커널   │
│   Guest Kernel   │              │    공유)         │
├──────────────────┤              ├──────────────────┤
│   Hypervisor     │              │   Host Kernel    │
│   Host Kernel    │              │   Host OS        │
│   Hardware       │              │   Hardware       │
└──────────────────┘              └──────────────────┘
```

- VM = *전체 OS + 커널*을 가상 하드웨어 위에 부팅
- 컨테이너 = *호스트 커널을 그대로 공유*, *애플리케이션 환경만 격리*

**증거:** 컨테이너 안에서 `uname -r` → 호스트 커널 버전 그대로. `nginx:alpine` 안에서도 호스트가 `5.15` 면 컨테이너도 `5.15`.

→ **격리는 *커널이 해주는 일*, *프로세스 입장에선 자기만의 세상처럼 보이는 일.***

---

## 2. Linux Namespace — *시야*를 격리

`man 7 namespaces` 정의:
> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.

총 **7가지 namespace** 가 있고, 컨테이너는 보통 *7개 모두* 새로 만든다.

| Namespace | 무엇을 격리? | 안에서 격리되면 보이는 변화 |
|---|---|---|
| **PID** | 프로세스 ID 트리 | 컨테이너 안 첫 프로세스가 PID 1. 호스트 프로세스는 `ps`에 안 보임. |
| **NET** | 네트워크 인터페이스·라우팅 테이블·iptables·소켓 | 자신만의 `lo`. 호스트 `eth0` 안 보임. 자신만의 라우팅. |
| **MNT** | 파일시스템 마운트 트리 | 자신만의 `/`. 호스트 `/home`·`/var` 안 보임. (`chroot`의 강화판) |
| **UTS** | hostname, domainname | 자신만의 `hostname`. 호스트 이름과 다르게 설정 가능. |
| **IPC** | System V IPC, POSIX 메시지 큐 | 다른 컨테이너의 shared memory 못 봄. |
| **USER** | UID·GID 매핑 | 컨테이너 안 root(uid 0) 가 호스트에선 일반 사용자로 매핑 가능 (*rootless 컨테이너의 베이스*). |
| **CGROUP** | cgroup 계층의 가시성 | 자기 cgroup만 `/sys/fs/cgroup`에서 보임. |

### 핵심 관찰
- *각 namespace는 독립적*. 일부만 격리 + 일부는 공유도 가능.
- **K8s의 Pod = *같은 NET + IPC + UTS namespace를 공유*하는 컨테이너들의 묶음**. → 같은 Pod 내 컨테이너끼리 `localhost`로 통신.
- *namespace로는 자원 양 제한 X*. 시야만 제한. 양 제한은 *cgroup*.

### 직접 만들어보기 — `unshare`
컨테이너 = namespace + cgroup의 조합. *수동으로* 만들어볼 수 있음:

```bash
# 새 PID + UTS + MNT + NET namespace에서 bash 실행 (root 필요)
sudo unshare --pid --uts --mount --net --fork bash

# 안에서 확인:
hostname myhost            # 호스트 영향 X
ps aux                     # bash가 PID 1
ip addr show               # 인터페이스 lo 뿐
```

→ docker가 하는 일의 *최소 단위*. 여기에 *rootfs 마운트 + cgroup + 네트워크 셋업*만 더하면 docker.

---

## 3. PID Namespace — *컨테이너 안 PID 1*이 특별한 이유

컨테이너 안에서 `ps aux` → 컨테이너 내부 프로세스만, *첫 프로세스가 PID 1*.

### PID 1의 책임 (init process)
- *고아 프로세스의 부모 인계* (parent reap)
- *시그널 처리* (SIGTERM 등)
- *좀비 프로세스 회수*

### 흔한 함정: shell 스크립트로 PID 1 띄우기
```dockerfile
CMD ["./start.sh"]
```
- `start.sh` 안에서 `nginx &` 같이 백그라운드 실행 → nginx의 부모가 *bash 스크립트*
- bash가 종료되면 nginx 고아됨, PID 1이 처리 안 함 → 좀비 누적
- 또 SIGTERM이 bash에 전달돼도 *자식 nginx에 전파 안 됨* → graceful shutdown 깨짐

→ **해결책:**
1. `exec` 사용 (`exec nginx`) — bash 자리를 nginx가 *교체* (PID 1이 nginx)
2. *진짜 init* 사용 (`tini`, `dumb-init`) — `--init` 플래그
3. Dockerfile에서 `ENTRYPOINT ["nginx", "-g", "daemon off;"]` 형식

면접 빈출: *"컨테이너 안 PID 1의 책임은? graceful shutdown이 안 될 때 원인은?"*

---

## 4. MNT Namespace — *chroot의 강화판*

- 컨테이너는 자신만의 rootfs를 *마운트해서* 그걸 `/`로 본다
- `chroot`와 비슷하지만 *마운트 테이블 전체*를 격리 → 호스트가 새 볼륨 마운트해도 컨테이너에 안 보임

### docker가 하는 일
1. OverlayFS로 image layer들 합친 결과를 `/var/lib/docker/overlay2/.../merged` 에 마운트
2. 컨테이너의 MNT namespace에서 *그 디렉터리를 `/`로* 마운트 변경 (`pivot_root` 또는 `chroot`)
3. 그 안에서 프로세스 실행

→ 이게 `04-volumes-and-storage.md` 의 OverlayFS와 *직접 연결*되는 부분.

---

## 5. NET Namespace — *별도 네트워크 스택*

- 각 컨테이너는 자기만의 *네트워크 인터페이스 목록·라우팅 테이블·iptables 룰·소켓 풀*
- docker는 *veth pair*(가상 케이블) 만들어 컨테이너 namespace ↔ 호스트 namespace 연결
- 호스트 쪽 veth는 *bridge(`docker0`)에 꽂힘*

→ 이게 `03-networking.md` 의 시작. 자세한 흐름은 거기서.

---

## 6. USER Namespace — *rootless 컨테이너*의 베이스

- 컨테이너 안 *root (uid 0)* 가 호스트 *uid 1000* 으로 매핑 가능
- 컨테이너 내부에선 root처럼 보이나 호스트에선 *권한 없음*
- → 컨테이너 탈출 사고 시 *피해 최소화*

```bash
# user namespace 매핑 예시
컨테이너 uid 0    →  호스트 uid 100000
컨테이너 uid 1    →  호스트 uid 100001
...
컨테이너 uid 65535 →  호스트 uid 165535
```

- Docker는 *기본적으로 USER namespace 안 씀* (옵션). Podman은 *기본 사용*.
- K8s에서는 `runAsUser` / `runAsNonRoot` 등으로 보안 강화 → ([06 참조](06-security.md))

---

## 7. Cgroup — *자원 한도*

`man 7 cgroups`:
> Control groups, usually referred to as cgroups, are a Linux kernel feature which allow processes to be organized into hierarchical groups whose usage of various types of resources can then be limited and monitored.

### 격리하는 자원
- **CPU** — 시간 비율 (CPU shares, quota)
- **Memory** — 최대 사용량, OOM 발생 시 동작
- **Block I/O** — 디스크 읽기/쓰기 속도
- **Network** — *직접 격리 X*, 다른 메커니즘 (tc, iptables)
- **PIDs** — 최대 프로세스 수
- **Devices** — /dev 접근 권한

### `docker run` 의 자원 옵션
| 옵션 | cgroup 동작 |
|---|---|
| `--memory=512m` | memory.max = 512M, 초과 시 OOMKill |
| `--cpus=1.5` | cpu.max = 150000 100000 (150% of 1 core period) |
| `--cpu-shares=512` | 가중치 (다른 컨테이너와 경합 시 비율) |
| `--pids-limit=100` | PID 최대 100개 (fork bomb 방어) |
| `--device-read-bps` | block I/O 제한 |

### Cgroup v1 vs v2
| 항목 | v1 | v2 |
|---|---|---|
| 계층 | *컨트롤러별로 별도 계층* (CPU/MEM 따로) | *단일 계층* (한 트리에서 모든 컨트롤러) |
| 일관성 | 컨트롤러 간 불일치 가능 | 일관된 정책 |
| 사용 | RHEL 7, 구버전 K8s | RHEL 9+, K8s 1.25+, systemd 기본 |

→ 컨테이너 OOM 디버깅: `dmesg | grep -i oom` 에서 *cgroup 경로*가 찍힘. 경로로 어느 컨테이너인지 추적 가능.

---

## 8. 모든 게 합쳐진 모습 — `docker run` 의 내부

```
사용자: docker run -d --memory=512m --cpus=1 -p 8080:80 nginx:latest

1.  daemon: image가 로컬에 있나?
    └─ 없으면 → registry에서 pull (manifest + layers)
    
2.  daemon: OverlayFS 마운트 준비
    ├─ image layer들 (read-only) lower
    ├─ 새 빈 디렉터리 upper (writable layer)
    └─ merged 디렉터리 = lower + upper 합쳐 보임
    
3.  daemon: 새 namespace 7개 생성 (clone(2) with CLONE_NEW* flags)
    ├─ PID, NET, MNT, UTS, IPC, USER, CGROUP
    
4.  daemon: 새 cgroup 생성 + 한도 적용
    ├─ /sys/fs/cgroup/.../docker-{id}/memory.max=512M
    └─ /sys/fs/cgroup/.../docker-{id}/cpu.max=100000 100000
    
5.  daemon: 네트워크 셋업
    ├─ veth pair 생성: veth0 (호스트) ↔ eth0 (컨테이너 namespace 안)
    ├─ veth0를 docker0 bridge에 attach
    ├─ 컨테이너 eth0에 IP 할당 (172.17.0.2 등)
    ├─ iptables PREROUTING에 DNAT 룰 (호스트 8080 → 172.17.0.2:80)
    
6.  daemon: 컨테이너 안에서 프로세스 실행
    ├─ pivot_root: merged 디렉터리를 / 로
    ├─ chdir to WORKDIR
    ├─ setuid to USER (보통 0)
    ├─ exec [ENTRYPOINT, CMD]  ← nginx 시작 (PID 1)
    
7.  daemon: stdin/stdout/stderr 연결, 종료 시 정리 등록
```

→ **이 흐름을 면접에서 *단계별로* 설명할 수 있으면 docker 작동 원리 마스터.**

---

## 9. *컨테이너 안에서* 호스트를 보는 법 (그리고 *왜 위험한가*)

```bash
# 컨테이너 안에서
ls /proc/1/root/        # PID 1의 rootfs 보기 — 자기 rootfs
ls /proc/{호스트PID}/root/  # 다른 namespace의 rootfs 보기 (권한 필요)

# 컨테이너 안에서 호스트 프로세스 트리 보기 (PID namespace 격리 우회)
# → 보통 안 됨 (PID namespace가 막음)
# → 단, --pid=host 옵션으로 띄우면 보임 (위험)
```

위험한 옵션들 (사용 시 격리 약해짐):
- `--pid=host` — 호스트 PID namespace 공유 (호스트 프로세스 다 보임)
- `--net=host` — 호스트 NET namespace 공유 (호스트 포트 직접 사용)
- `--privileged` — *거의 모든 격리 해제*. 사실상 root.
- `-v /:/host` — 호스트 루트를 마운트 (탈출 사실상 자유)

→ ([06-security.md](06-security.md) 에서 더 깊이)

---

## 10. 자주 헷갈리는 것

### 10-1. 컨테이너 안 *root*는 *진짜 root*인가?
- USER namespace 안 쓰면 **호스트에서도 root**. 매우 위험.
- USER namespace 쓰면 *컨테이너 안에서만 root처럼 보임*, 호스트에선 일반 사용자.
- *Best practice*: Dockerfile에 `USER 1000` 명시 + (가능하면) USER namespace.

### 10-2. `--privileged` 의 *진짜* 의미
- 거의 모든 *capability* 부여 + 모든 device 접근 + AppArmor·seccomp 해제 + cgroup 한도 해제 가능
- *컨테이너 ≠ VM 보안 경계*. `--privileged`는 *그 약한 경계마저 거의 없음*.
- "Docker-in-Docker 하려면 --privileged 필요" 라는 말 → *진짜인지 의심하고 *대안 검토* (예: socket mount, podman in container, kaniko)

### 10-3. *컨테이너가 죽었다 = PID 1이 종료됐다*
- 컨테이너의 PID 1이 종료되면 그 namespace 전체 정리됨
- → CMD가 *백그라운드로* 떠 있으면 *바로 죽음* (foreground로 실행 필수)
- 예: `CMD ["nginx", "-g", "daemon off;"]` — `daemon off;` 로 foreground 강제

### 10-4. *호스트 커널 업그레이드*는 컨테이너에 영향?
- 컨테이너는 *호스트 커널을 공유*. 호스트 커널 바뀌면 *모든 컨테이너가 새 커널 사용*.
- 호스트 커널 버그가 컨테이너 동작에 영향 줄 수 있음.
- *옛 커널에서만 동작하는 워크로드*는 호스트 커널을 고정해야 함.

### 10-5. cgroup 한도는 *컨테이너 안에서 보이는가*
- 부분적. `/proc/meminfo`는 *호스트 총 메모리* 를 보여줌 (v1에선).
- → JVM 같이 *호스트 메모리 기준으로 힙 잡는 앱*은 OOM 빈발 가능
- v2 / 최신 JDK는 cgroup-aware. 옛날 JDK는 `-XX:MaxRAMPercentage` 같은 옵션 필요.

---

## 🎤 면접 빈출 Q&A

### Q1. Docker는 어떻게 격리를 만드나요?
> Linux 커널의 *namespace*와 *cgroup* 기능을 활용한다. namespace는 *시야*를 격리 (PID·NET·MNT·UTS·IPC·USER·CGROUP 7종), cgroup은 *자원 한도*를 강제 (CPU·MEM·I/O·PID 수 등). 컨테이너는 결국 *이 두 가지가 적용된 Linux 프로세스*다. VM과 달리 *호스트 커널을 공유*하므로 부팅이 밀리초 단위.

### Q2. Docker와 VM의 차이는?
> VM은 하이퍼바이저가 *가상 하드웨어* 만들고 *전체 게스트 OS+커널*을 부팅. 컨테이너는 *호스트 커널을 그대로 공유*하고 *프로세스 격리*만 추가. 부팅 시간·디스크·메모리 모두 컨테이너가 훨씬 가볍지만, *보안 경계는 VM이 강함* (커널 공유 → 커널 exploit으로 탈출 가능).

### Q3. `docker run` 했을 때 내부에서 일어나는 일을 순서대로 설명해주세요.
> (1) image 없으면 registry에서 pull. (2) OverlayFS로 layer 합쳐 마운트. (3) 새 namespace 7개 생성. (4) cgroup 생성 + 자원 한도 적용. (5) 네트워크 셋업 — veth pair 만들어 docker0 bridge에 연결, iptables DNAT. (6) pivot_root로 rootfs 변경 후 ENTRYPOINT/CMD를 PID 1으로 exec. 종료 시 namespace·cgroup·writable layer 정리.

### Q4. K8s Pod에 컨테이너 여러 개가 들어가는데, 그들의 격리는?
> 같은 Pod의 컨테이너들은 *NET·IPC·UTS namespace를 공유*한다. 그래서 `localhost`로 통신 가능, 같은 hostname. 단 *PID·MNT·USER namespace는 각자 독립* (옵션으로 PID 공유도 가능). → Sidecar 패턴(예: log shipper)이 가능한 이유.

### Q5. 컨테이너 안에서 *graceful shutdown* 이 안 되는 이유와 해결책?
> CMD를 *shell 스크립트*로 띄우면 bash가 PID 1. SIGTERM은 PID 1에 가는데 bash는 *자식에게 전파 안 함*. 해결: (1) shell script 안에서 `exec`로 nginx를 PID 1으로 *교체*, (2) `tini`/`dumb-init` 같은 *진짜 init* 사용, (3) Dockerfile `ENTRYPOINT ["nginx", "..."]` 형식으로 *exec form* 사용.

### Q6. `--privileged` 가 *위험한* 이유는?
> *거의 모든 capability* 부여 + 모든 device 접근 + AppArmor·seccomp 해제. 컨테이너 안 root가 *사실상 호스트 root*. 커널 모듈 로드, 호스트 디스크 마운트, ptrace 등 다 가능 → *컨테이너 격리 사실상 무력화*. 꼭 필요하다면 *왜* 필요한지 정당화하고, 대안(특정 capability만 추가, socket mount 등) 검토.

### Q7. JVM 컨테이너가 OOMKill 자주 발생할 때 의심해야 할 것은?
> JVM이 *호스트 메모리 기준*으로 힙을 잡는 경우. cgroup 한도가 *예: 512MB*인데 JVM이 *호스트 32GB 기준 8GB 힙* 잡으려 함 → cgroup에서 OOMKill. 해결: (1) cgroup-aware JDK (11+), (2) `-XX:MaxRAMPercentage=75.0`, (3) `-Xmx` 명시.

---

## 🔗 Cross-reference

- **NET namespace** 의 구체적 활용 → [03-networking.md](03-networking.md)
- **MNT namespace** + OverlayFS → [04-volumes-and-storage.md](04-volumes-and-storage.md)
- **USER namespace** + capability + seccomp → [06-security.md](06-security.md)
- **K8s Pod 격리** (Pod = namespace 공유 컨테이너 묶음) → `kubernetes-basics/` (예정)
- **PSS (Pod Security Standards)** = K8s가 위 격리·capability·USER 등을 *선언적으로 강제* → `k8s-security-basics/` (예정)
- **Linux 프로세스·시그널·systemd** 기본 → `linux-basics/` (예정)

---

## 📝 3줄 요약

1. 컨테이너 = *namespace(시야 격리) + cgroup(자원 한도) + rootfs(파일시스템 교체) 된 Linux 프로세스*. VM 아님 — 호스트 커널 공유.
2. namespace 7종 (PID·NET·MNT·UTS·IPC·USER·CGROUP) — 어떤 *전역 자원*의 *자신만의 인스턴스를 가진 것처럼 보이게* 함.
3. K8s Pod 격리·CNI·PSS 등 *상위 추상화*가 결국 이 두 가지 기능 위에 얹혀 있다. 이 파일을 이해하면 K8s가 마법이 아니게 됨.

# 03 — Filesystem & Permissions: inode·UID/GID·capabilities

> **공식 근거:** `man 7 path_resolution`, `man 5 inode`, `man 7 capabilities`, `man chmod`, `man 5 fstab`
>
> **선행:** [02-process-and-signals.md](02-process-and-signals.md)

---

## 🎯 한 문장

> **Linux 파일시스템 = *inode (메타 + 데이터 포인터) + dentry (이름)* 구조. 권한은 *3단계 (owner/group/other) × 3비트 (rwx)* + *고급 (ACL, capability, SUID, sticky bit, SELinux/AppArmor)*. `/proc`·`/sys`가 *커널과의 인터페이스 가상 파일시스템*.**

---

## 1. inode — *파일의 진짜 정체*

### 정의
- *inode = 파일의 메타데이터 + 데이터 블록 포인터*
- 파일 이름은 *inode에 없음* — *디렉터리가 이름 → inode 매핑*
- `ls -li`로 *inode 번호* 볼 수 있음

```bash
ls -li
# 1234567 -rw-r--r-- 1 user user  4096 May 26 12:00 hello.txt
#  ^^^^^^^
#  inode #
```

### 의미
- *같은 inode = 같은 파일* (하드링크 = 같은 inode 가리키는 여러 이름)
- *다른 inode = 다른 파일* (이름 같아도 다른 파일시스템이면)

### inode 고갈
- *디스크 용량 남음*인데 *파일 생성 fail* — `df -i`로 *inode 사용량* 확인
- 흔한 원인: *작은 파일 너무 많음* (예: mail spool, 로그 파편)
- 해결: 옛 파일 정리, 또는 *파일시스템 재포맷* (mkfs.ext4의 `-N` 옵션으로 inode 수 늘림)

### Hard link vs Symlink
| | Hard link | Symlink |
|---|---|---|
| 구조 | 같은 inode | 새 inode, *경로 문자열 저장* |
| 생성 | `ln src dst` | `ln -s src dst` |
| 다른 파일시스템 | ❌ | ✅ |
| 원본 삭제 시 | *남은 링크가 데이터 유지* | *깨짐* (dangling) |
| 디렉터리 대상 | 불가 (특수 case 제외) | 가능 |

→ *symlink가 일반적*. hard link는 *백업 도구 (rsnapshot 등)*에 활용.

---

## 2. 마운트 — *파일시스템을 트리에 붙임*

### `mount` 명령
```bash
mount                       # 현재 모든 마운트
mount /dev/sdb1 /mnt/data
mount -t nfs server:/path /mnt/nfs
mount -o remount,ro /        # rootfs를 read-only로 remount
```

### `/etc/fstab` — *부팅 시 자동 마운트*
```
# <device>          <mount point>  <fs type> <options>          <dump> <fsck>
UUID=xxx-xxx        /              ext4      defaults           0      1
UUID=yyy-yyy        /var           xfs       defaults,noatime   0      2
/dev/sdb1           /data          ext4      defaults           0      2
tmpfs               /tmp           tmpfs     defaults,size=2G   0      0
```

### 옵션
| 옵션 | 의미 |
|---|---|
| `ro` / `rw` | read-only / read-write |
| `noatime` | 접근 시간 기록 안 함 (성능 ↑) |
| `nosuid` | suid bit 무시 (보안) |
| `nodev` | device file 무시 (보안) |
| `noexec` | 실행 금지 (보안, /tmp 등) |

### `df` / `du`
```bash
df -h                       # 마운트별 사용량
df -i                       # inode 사용량
du -sh /var/log/*           # 디렉터리별 사용량
du -sh * | sort -h          # 큰 순으로
```

### 디스크 공간 회복 *안 되는* 흔한 원인
- *프로세스가 삭제된 파일을 fd로 잡고 있음*
- `lsof | grep deleted` 로 *확인*
- 그 프로세스 *재시작*하면 해제

---

## 3. 권한 — *3 × 3 + 특수 비트*

### 기본 권한
```
-rwxr-xr--
^         
type     
 ^^^      
 owner perm
    ^^^   
    group perm
       ^^^
       other perm
```

| 비트 | 파일 | 디렉터리 |
|---|---|---|
| **r** (4) | 내용 읽기 | 목록 보기 (ls) |
| **w** (2) | 내용 쓰기 | 파일 생성·삭제 (디렉터리 안) |
| **x** (1) | 실행 | *진입* (cd) |

### chmod
```bash
chmod 755 script.sh         # rwxr-xr-x
chmod 644 config.txt        # rw-r--r--
chmod u+x script.sh         # 소유자에 실행권한 추가
chmod g-w file              # 그룹에 쓰기권한 제거
chmod -R 750 /opt/myapp     # 재귀
```

### 특수 비트
| 비트 | 의미 | 표시 |
|---|---|---|
| **SUID** (4000) | 실행 시 *소유자 권한*으로 동작 | `rws------` |
| **SGID** (2000) | 실행 시 *그룹 권한*으로 동작 + 디렉터리 안 생성된 파일이 *그 디렉터리 그룹 상속* | `---s------` 또는 디렉터리 |
| **Sticky bit** (1000) | 디렉터리 안 *파일은 owner만 삭제* (`/tmp` 등) | `------rwt` |

### SUID 위험
- `/usr/bin/passwd` 같이 *root 권한 필요한 명령*에 사용
- *임의의 실행 파일에 SUID 주면 위험* — *권한 escalation*
- 컨테이너에선 `nosuid` 마운트 권장 ([06 security 참조](../docker-basics/06-security.md))

```bash
find / -perm -4000 2>/dev/null    # SUID 파일 모두 찾기
```

### 숫자 권한 빠르게 읽기
- `r=4, w=2, x=1`
- `755` = `rwxr-xr-x` (소유자 7, 그룹 5, 기타 5)
- `644` = `rw-r--r--` (6, 4, 4)

---

## 4. 소유권 — UID/GID

### UID 0 = root
- 이름 'root'는 관습. 정체는 *uid 0*.
- *uid 0이면 거의 모든 권한* (capability로 세분화 가능)

### 일반 사용자
- `/etc/passwd`: `user:x:1000:1000:User:/home/user:/bin/bash`
- UID 1000부터 일반 user (RHEL/Ubuntu 관습)
- *system user*는 100~999 (또는 1~999)

### chown
```bash
chown user:group file
chown -R user:group /opt/myapp
chown :group file              # 그룹만
```

### `id` 명령
```bash
id                          # 현재 user
# uid=1000(user) gid=1000(user) groups=1000(user),27(sudo),998(docker)
```

→ docker 그룹에 속하면 *passwordless docker* 가능 — *docker socket = root와 동급*이라 *실질적 root 권한*. *멤버십 신중*.

---

## 5. ACL — *3×3 권한이 부족할 때*

기본 권한 한계: *3 group에만* 권한 부여 가능 (owner / group / other).
*사용자 A에만 read 추가* 같은 *세밀한 권한* → **ACL** (Access Control List).

```bash
# 현재 ACL 보기
getfacl file

# 사용자 alice에 rw 부여
setfacl -m u:alice:rw file

# 그룹 dev에 r 부여
setfacl -m g:dev:r file

# 디렉터리의 default ACL — 안에 만들어지는 파일이 상속
setfacl -d -m u:alice:rw /opt/shared

# 제거
setfacl -x u:alice file
```

→ ACL 적용된 파일은 `ls -l` 에서 *권한 끝에 `+`* 표시.

→ *컨테이너 안에선 거의 안 씀*. 일반 서버에서 *복잡한 권한 모델* 필요할 때.

---

## 6. Capabilities — *root를 세분화*

전통: *root = 모든 권한 / 일반 user = 거의 없음* — *너무 거침*.

해결: *root 권한을 30+ capability로 분할*. 필요한 것만 부여.

`man 7 capabilities` 일부:

| Capability | 의미 |
|---|---|
| `CAP_NET_BIND_SERVICE` | 1024 이하 포트 bind (예: 80, 443) |
| `CAP_NET_ADMIN` | 네트워크 설정 (iptables, route, etc.) |
| `CAP_NET_RAW` | raw socket (ping 등) |
| `CAP_SYS_ADMIN` | *광범위* (mount, swap, reboot 등) — *사실상 root* |
| `CAP_SYS_PTRACE` | 다른 프로세스 추적 (gdb) |
| `CAP_SYS_MODULE` | 커널 모듈 로드 |
| `CAP_SYS_TIME` | 시스템 시간 변경 |
| `CAP_DAC_OVERRIDE` | 파일 권한 무시 |
| `CAP_KILL` | 임의 프로세스에 신호 |
| `CAP_SETUID` / `CAP_SETGID` | uid/gid 변경 |

### 사용
```bash
# 바이너리에 capability 부여 (SUID 없이 80 포트 bind 가능)
setcap 'cap_net_bind_service=+ep' /usr/local/bin/myapp

# 현재 capability 확인
getcap /usr/local/bin/myapp
cat /proc/{pid}/status | grep Cap
```

### 컨테이너에서
- Docker는 *기본적으로 14개 capability만 부여* (위험한 거 drop)
- `--cap-add=NET_ADMIN`, `--cap-drop=ALL` 로 조절
- K8s `securityContext.capabilities`

→ [`docker-basics/06`](../docker-basics/06-security.md), [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md) 의 *최소 권한*과 직결.

---

## 7. `/proc`, `/sys`, `/dev` — *가상 파일시스템*

### `/proc` — *프로세스 + 커널 상태*
- `/proc/{pid}/...` — 프로세스별 정보
  - `/proc/{pid}/status` — uid/gid/state/메모리/capability
  - `/proc/{pid}/cmdline` — 실행 명령
  - `/proc/{pid}/environ` — 환경변수
  - `/proc/{pid}/fd/` — 열린 file descriptor (symlink)
  - `/proc/{pid}/maps` — 메모리 맵
  - `/proc/{pid}/cgroup` — *어느 cgroup에 속함*
  - `/proc/{pid}/ns/` — *어느 namespace에 속함*
- `/proc/meminfo` — 메모리 통계
- `/proc/cpuinfo` — CPU
- `/proc/net/...` — 네트워크 통계·연결
- `/proc/sys/...` — *커널 sysctl 변수* (read·write)

### `/sys` — *kernel objects (devices, modules, ...)*
- `/sys/class/net/eth0/` — 네트워크 인터페이스
- `/sys/block/sda/` — block device
- `/sys/fs/cgroup/` — cgroup 계층

### `/dev` — *device files*
- `/dev/null` — 휴지통 (쓰면 사라짐)
- `/dev/zero` — 0 무한 생성
- `/dev/random` / `/dev/urandom` — 난수
- `/dev/sda`, `/dev/sda1` — 디스크
- `/dev/pts/{N}` — pseudo terminal
- `udev`가 *동적으로 device file 생성·관리*

### sysctl
```bash
sysctl -a                       # 모든 변수
sysctl net.ipv4.ip_forward      # 특정 변수 읽기
sysctl -w net.ipv4.ip_forward=1 # 임시 변경
# 영구는 /etc/sysctl.d/ 에
```

→ K8s 노드는 *수많은 sysctl 변경* (net.bridge.bridge-nf-call-iptables 등) 필요.

---

## 8. *방화벽* — firewalld vs ufw vs iptables

### iptables — *근본*
- Linux 커널의 *netfilter 인터페이스*
- *모든 패킷 처리 룰*이 여기 (대부분의 방화벽 도구가 내부적으로 iptables)
- *직접 다루기 복잡* → 상위 도구 사용

### firewalld — RHEL·Rocky·Fedora 표준
```bash
firewall-cmd --get-default-zone
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
firewall-cmd --list-all
```
- *zone* 개념: *인터페이스마다 다른 정책*
- *runtime* + *permanent* 분리 — `--permanent` 안 붙이면 *재부팅 시 사라짐*

### ufw — Ubuntu 표준
```bash
ufw enable
ufw allow 22
ufw allow 80/tcp
ufw deny from 192.168.1.100
ufw status verbose
```

### iptables 직접
```bash
iptables -L -n -v               # 룰 보기
iptables -t nat -L -n -v        # NAT 테이블
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

→ K8s에서는 *kube-proxy*가 iptables 룰을 *자동 박음*. *수동 변경하면 깨짐*. → CNI·NetworkPolicy로 추상화.

---

## 9. SELinux / AppArmor — *MAC*

전통 권한 (DAC, Discretionary Access Control) 한계:
- *root는 모든 권한*
- *프로세스 권한 분리 약함*

해결: **MAC** (Mandatory Access Control)
- *모든 프로세스·파일에 *label/profile**
- *커널이 강제* — root도 못 우회
- *세밀한 정책* (예: nginx는 `/var/log/nginx`만 write 가능)

### SELinux (RHEL 진영)
| 모드 | 의미 |
|---|---|
| **Enforcing** | 정책 *강제* (위반 차단) |
| **Permissive** | 정책 *적용하나 차단 X* — 로그만 |
| **Disabled** | 끔 |

```bash
getenforce
setenforce 1                     # Enforcing
setenforce 0                     # Permissive (임시)
sestatus                         # 자세히
```

흔한 함정:
- *권한 충분한데 *접근 거부* → SELinux denial 의심
- `journalctl | grep -i denied`, `ausearch -m AVC`
- *Permissive로 잠시 전환해서 테스트*
- 해결: *맞는 context 부여*, audit2allow로 정책 생성

### AppArmor (Ubuntu·Debian 진영)
- *path-based* (SELinux는 *label-based*)
- 약간 단순. *프로필* 단위 적용.

```bash
aa-status
aa-enforce /etc/apparmor.d/usr.sbin.nginx
aa-complain /etc/apparmor.d/usr.sbin.nginx
```

→ docker도 *기본 AppArmor·SELinux 프로필* 적용.

---

## 10. 자주 헷갈리는 것

### 10-1. *파일 삭제했는데 *디스크 안 줄어듦**
- 프로세스가 *fd로 잡고 있음*
- `lsof | grep deleted` → 해당 프로세스 *재시작*

### 10-2. *디스크 남는데 *파일 못 만듦**
- *inode 고갈*. `df -i` 확인.
- 작은 파일 많은 디렉터리 정리

### 10-3. *chmod 777 — *모두에게 rwx**
- "권한 문제 해결되면" 좋은 게 *아님* — *보안 사고*
- 진짜 원인 (사용자·그룹·context 등) 찾아 해결

### 10-4. *수정 안 했는데 *mtime 바뀜**
- `mtime` = 내용 수정 시간
- `atime` = 접근 시간 (성능 위해 *noatime* 마운트 흔함)
- `ctime` = *메타데이터 변경 시간* (권한·소유자 변경 시 갱신)

### 10-5. *symlink 깨졌는데 *ls는 보임**
- `ls -l`은 *symlink의 타겟 존재 안 확인*
- `readlink -f symlink` 로 *실제 가리키는 경로 확인*
- `find . -xtype l` 로 *깨진 symlink 찾기*

### 10-6. *디렉터리에 x 없으면 *진입 불가**
- 디렉터리의 *x = "이 디렉터리로 cd 가능"*
- `r` 만 있으면 *목록 보기는 가능하나 진입 X*
- → 비밀 폴더는 *x만 줘서 *목록 가리고 진입 허용*

---

## 🎤 면접 빈출 Q&A

### Q1. `chmod 755`, `chmod 644` 의미?
> *r=4, w=2, x=1* 합산. *755* = owner rwx (7), group r-x (5), other r-x (5) — 실행 파일·디렉터리 기본. *644* = owner rw- (6), group r-- (4), other r-- (4) — 일반 파일 기본. *775* = group에도 쓰기. *600* = owner만 rw, secret 파일 등.

### Q2. Hard link와 Symlink 차이?
> *Hard link*: *같은 inode를 가리키는 추가 이름*. 다른 파일시스템 불가. 원본 삭제해도 *남은 링크가 데이터 유지*. *Symlink*: *경로 문자열을 저장한 새 파일*. 다른 파일시스템 가능. 원본 삭제 시 *깨짐 (dangling)*. 일반적으론 symlink 더 유연.

### Q3. 디스크 용량 남는데 *파일 생성 fail* — 가능한 원인?
> *inode 고갈* — `df -i`로 확인. 작은 파일 너무 많은 디렉터리 (mail spool, 로그 파편, /tmp 등). 정리하거나 *파일시스템 재포맷 시 inode 수 늘림* (mkfs `-N` 옵션).

### Q4. Capability와 root의 관계?
> 전통: root = 모든 권한, user = 거의 없음 (이분법). 해결: *root 권한을 30+ capability로 분할*. 각각 따로 부여 가능. 예: `CAP_NET_BIND_SERVICE` = 1024 이하 포트 bind. *non-root user에 capability만 부여*해서 *root 없이* 80 포트 listen 가능. 컨테이너에선 *cap-drop ALL + 필요한 것만 add*가 표준.

### Q5. *프로세스가 어느 파일을 열고 있나* 어떻게 확인?
> `lsof -p {pid}` — 특정 프로세스의 모든 fd. `lsof /var/log/myapp.log` — 그 파일을 누가 열고 있나. `lsof -i :8080` — 8080 포트 누가 listen. `lsof | grep deleted` — *삭제됐지만 fd로 잡힌* 파일 (디스크 공간 회복 안 됨 시 의심).

### Q6. SELinux Enforcing/Permissive/Disabled 차이? *권한 충분한데 접근 거부*?
> *Enforcing*: 정책 강제 (위반 차단). *Permissive*: 정책 적용하나 차단 X (로그만). *Disabled*: 끔. **DAC 권한 충분한데 접근 거부 → SELinux denial 의심**. `journalctl | grep -i denied`, `ausearch -m AVC`로 확인. *Permissive로 임시 전환* 테스트. 해결: *맞는 context 부여* (chcon, restorecon), 또는 *audit2allow*로 정책 생성.

### Q7. `/proc/{pid}/` 에서 어떤 정보를 볼 수 있나요?
> 프로세스별 *모든 상태*. `status` (uid/gid/state/메모리/capability), `cmdline` (실행 명령), `environ` (환경변수), `fd/` (열린 file descriptor), `maps` (메모리 맵), `cgroup` (소속 cgroup), `ns/` (소속 namespace), `limits` (rlimits). 디버깅의 *블랙박스 보기 창*.

---

## 🔗 Cross-reference

- **systemd unit의 보안 강화 옵션 (NoNewPrivileges, ProtectSystem)** → [01-systemd.md](01-systemd.md)
- **컨테이너의 capability·SUID·USER namespace** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md), [`docker-basics/06`](../docker-basics/06-security.md)
- **K8s Pod Security Standards** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md)
- **K8s NetworkPolicy (방화벽의 K8s 추상화)** → [`kubernetes-basics/03`](../kubernetes-basics/03-networking.md)
- **PV/Volume 마운트** → [`kubernetes-basics/04`](../kubernetes-basics/04-config-and-storage.md), [`docker-basics/04`](../docker-basics/04-volumes-and-storage.md)

---

## 📝 3줄 요약

1. *inode = 메타 + 데이터 포인터*. 이름은 디렉터리에. *디스크 남는데 파일 못 만들면 inode 고갈 의심*. *삭제 후 공간 안 줄면 fd로 잡혔는지*.
2. 권한 = *3×3 + SUID/SGID/sticky*. `chmod 777`은 *해결 아니라 사고*. ACL·Capability로 세밀한 권한, MAC (SELinux/AppArmor)로 강제 정책.
3. `/proc`·`/sys`·`/dev`가 *커널과의 인터페이스 가상 파일시스템*. 디버깅·튜닝의 *블랙박스 보기 창*.

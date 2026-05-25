# 05 — Package 관리 & 장애 디버깅 체크리스트

> **공식 근거:** `man dnf`, `man apt`, `man journalctl`, [Brendan Gregg — Linux Performance](https://www.brendangregg.com/linuxperf.html)
>
> **선행:** 이 폴더의 01~04 전부

---

## 🎯 한 문장

> **패키지 관리(dnf/apt)는 *재현 가능한 서버 상태*의 베이스. 장애 디버깅은 *위에서 아래로 좁히는* (시스템 → 자원 → 프로세스 → syscall) 일관된 흐름. *어디서 시작할지 모르면 마법, 알면 단순 작업*.**

---

## 1. 패키지 관리 — RPM (dnf) vs DEB (apt)

### RHEL 진영 (Rocky, AlmaLinux, RHEL, Fedora, Amazon Linux)
- *RPM* 패키지 (`.rpm`)
- 매니저: **dnf** (RHEL 8+, Fedora 22+). yum의 후계 (yum도 alias 동작)

### Debian 진영 (Ubuntu, Debian)
- *DEB* 패키지 (`.deb`)
- 매니저: **apt**, **apt-get** (구식)

### 명령 매트릭스

| 동작 | dnf (RHEL) | apt (Debian) |
|---|---|---|
| 업데이트 메타 | `dnf check-update` | `apt update` |
| 시스템 업그레이드 | `dnf upgrade` | `apt upgrade` |
| 패키지 설치 | `dnf install nginx` | `apt install nginx` |
| 패키지 제거 | `dnf remove nginx` | `apt remove nginx` (설정 유지) / `apt purge nginx` (완전) |
| 검색 | `dnf search nginx` | `apt search nginx` |
| 정보 | `dnf info nginx` | `apt show nginx` |
| 어느 패키지 소유? | `rpm -qf /usr/sbin/nginx` | `dpkg -S /usr/sbin/nginx` |
| 패키지의 파일 목록 | `rpm -ql nginx` | `dpkg -L nginx` |
| 설치된 모든 패키지 | `rpm -qa` | `dpkg -l` |
| 의존성 확인 | `dnf repoquery --requires nginx` | `apt depends nginx` |
| 의존성 역추적 | `dnf repoquery --whatrequires nginx` | `apt rdepends nginx` |

### Repository 관리

**dnf:**
```bash
# /etc/yum.repos.d/{name}.repo
# enabled=1, gpgcheck=1 권장
dnf repolist
dnf config-manager --add-repo https://...
dnf config-manager --set-disabled myrepo
```

**apt:**
```bash
# /etc/apt/sources.list 또는 /etc/apt/sources.list.d/*.list
# /etc/apt/keyrings/ 에 GPG key
apt policy nginx               # 어느 repo·버전이 후보
add-apt-repository ppa:...
```

### 버전 고정

**dnf:**
```bash
# /etc/dnf/dnf.conf 또는 명시:
dnf install nginx-1.20.1
echo "exclude=nginx*" >> /etc/dnf/dnf.conf   # 영구 제외
```

**apt:**
```bash
apt-mark hold nginx            # 업그레이드 시 보류
# 또는 /etc/apt/preferences.d/ 에 pinning
```

→ 운영에선 *불변 인프라*가 정답. *서버에 매번 install* 보다 *AMI/이미지에 미리 포함* → 변경 시 *새 이미지·새 인스턴스*.

---

## 2. 서비스 상태·로그 — `systemctl` + `journalctl` 통합

기본 흐름 ([01-systemd.md](01-systemd.md) 참조):

```bash
# 상태
systemctl status nginx

# 실시간 로그
journalctl -u nginx -f

# 어제부터 에러만
journalctl -u nginx --since yesterday -p err

# kernel 메시지 (OOM, hardware)
journalctl -k --since today
dmesg -T | tail -50
```

---

## 3. *장애 디버깅 — 12단계 체크리스트*

서버가 *이상해* 라고 들어왔을 때. 위에서 아래로:

### 1. *언제부터 / 무엇이 변했나*
- *최근 배포·인프라 변경·트래픽 변동* — 가장 흔한 원인
- *git log*, *Argo CD sync 이력*, *Terraform plan*, *팀 슬랙*

### 2. *시스템 상위 지표* — uptime / top
```bash
uptime              # load average
top -bn1 | head     # 또는 htop
```
- LA 폭증? 메모리 차 있음? D-state 프로세스?

### 3. *메모리* — free
```bash
free -h
# available 컬럼이 진짜 지표
```
- *available 매우 적음* → 메모리 압박. swap 발생 가능.
- `dmesg | grep -i oom` — OOM 사고 있었나

### 4. *디스크 — df / iostat*
```bash
df -h               # 사용량 (특히 / 와 /var)
df -i               # inode
iostat -x 1 5       # I/O 통계 (await, %util)
```
- 디스크 *100% 차면 대부분 망함*
- `await` 높음 = I/O 응답 느림
- `%util` 100% = 디스크 포화

### 5. *디스크 큰 거 찾기*
```bash
du -sh /var/* | sort -h | tail
du -sh /var/log/* | sort -h | tail
```
- 보통: 로그 폭증, 옛 백업, *deleted but fd-held 파일*
- `lsof +L1` — *삭제됐는데 fd로 잡힌 파일*

### 6. *네트워크 — ss / ping*
```bash
ss -tunlp           # listening + 프로세스
ss -tan | awk '{print $1}' | sort | uniq -c   # 상태별 카운트
ss -s               # 전체 통계

ping 8.8.8.8        # 외부 도달
ping {게이트웨이}    # 게이트웨이 도달
dig google.com      # DNS
```
- *TIME_WAIT 폭증* → port 고갈
- *CLOSE_WAIT 폭증* → 앱이 close 안 함
- *DNS 실패* → resolv.conf, systemd-resolved, /etc/hosts

### 7. *문제 프로세스 좁히기*
```bash
ps auxf             # 트리 형태
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```
- CPU·메모리 *상위 프로세스*가 *예상한 것?*
- D-state 프로세스? Z (zombie)?

### 8. *그 프로세스의 *행동* — strace / lsof*
```bash
strace -p {pid} -c     # 통계 (어떤 syscall 많이)
strace -p {pid} -e file,network
lsof -p {pid}
cat /proc/{pid}/status
cat /proc/{pid}/limits
```
- "*어디서 막혀 있나*" — strace가 *write/read에 멈춰* → I/O. *poll/epoll* → 정상 대기.
- `lsof | grep deleted` — 디스크 회복 안 됨 시

### 9. *kernel 메시지*
```bash
dmesg -T | tail -100
journalctl -k --since "1 hour ago"
```
- OOM Killer 발동? hardware error? *NIC reset*? *fs 에러*?

### 10. *컨테이너·K8s 컨텍스트면*
```bash
kubectl get pods -A | grep -v Running
kubectl describe pod {pod} -n {ns}
kubectl logs {pod} -n {ns} --previous
kubectl top pods -A --sort-by=cpu | head
kubectl top nodes
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -30
```

### 11. *외부 의존* — 다른 서비스 동작?
- 우리는 안 깨졌고 *DB 가 깨졌다* / *외부 API 응답 느림* / *CDN 장애*
- *upstream status page*, *Datadog/CloudWatch 상태*

### 12. *결정·기록·완화*
- 임시 완화 (재시작, 트래픽 차단, 캐시 비활성, 회로 차단기)
- *postmortem* 작성. *root cause 명확*해야 같은 사고 안 남.

---

## 4. *상위 지표 → 좁히기* 도구 정리

Brendan Gregg의 *USE Method* (Utilization, Saturation, Errors):

| 자원 | Util | Sat | Err |
|---|---|---|---|
| CPU | `top` `%CPU` | `vmstat 1` `r` 컬럼 | `dmesg` |
| Memory | `free -h` | `vmstat 1` `si/so` | `dmesg \| grep oom` |
| Disk | `iostat -x 1` `%util` | `iostat -x 1` `await` | `dmesg`, `smartctl` |
| Network | `sar -n DEV 1` | `ss -s` retrans | `ifconfig` errors |

→ *각 자원에 대해 U/S/E 한 번씩만* — 빠르게 *어디에 문제 있나* 파악.

---

## 5. *간단한 commands 자주 잊는 것*

```bash
# 현재 listening port
ss -tunlp

# 특정 포트 누가 listen
lsof -i :8080

# 외부 연결 상태별
ss -tan state established | wc -l
ss -tan state time-wait | wc -l

# 디렉터리 크기 정렬
du -sh /var/log/* 2>/dev/null | sort -h | tail

# 큰 파일 찾기
find / -type f -size +1G 2>/dev/null
find / -type f -mtime -1 2>/dev/null | head     # 24h 안 수정된

# /tmp 정리 후보
find /tmp -type f -atime +7 2>/dev/null

# 환경변수 export
export VAR=value; env | grep VAR

# 시간 차이
date -u                            # UTC
timedatectl                        # 시스템 시간/NTP 상태
chronyc tracking                   # NTP sync 상세

# DNS resolver
resolvectl status                  # systemd-resolved 활성 시
cat /etc/resolv.conf

# 부팅 시간 분석
systemd-analyze
systemd-analyze blame | head
systemd-analyze critical-chain
```

---

## 6. *vim·grep·find*은 *진짜* 알아야

```bash
# grep
grep -r "ERROR" /var/log/        # 재귀
grep -rn --include="*.py" "TODO" # 줄번호, 특정 파일만
grep -A 3 -B 3 "panic" file      # 매칭 줄 전후 3줄
grep -v "DEBUG" file             # invert (DEBUG 제외)
grep -E "error|fatal" file       # extended regex

# find
find . -name "*.log"
find . -type f -mtime -1         # 1일 안 수정
find . -type f -size +100M
find . -type f -exec chmod 644 {} \;
find . -type d -empty -delete    # 빈 디렉터리 삭제

# awk·sort·uniq·sed 조합
ps aux | awk '{print $1}' | sort | uniq -c | sort -rn   # user별 프로세스 수
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head   # IP top
```

---

## 7. *재현성 있는 환경 만들기*

옛 패턴: *서버에 SSH로 들어가서 install*. *눈으로 본 것만 진실*. → *재현 불가능*.

### Immutable Infrastructure
- *서버를 변경하지 않음*. 변경 = *새 이미지 빌드 + 새 인스턴스*.
- AMI / VM 이미지 / 컨테이너 이미지
- *Cattle, not Pets*

### Configuration Management (Ansible)
- *선언적 상태* — "이 서버는 *이런 상태여야 한다*"
- *멱등성* — 여러 번 적용해도 *같은 결과*
- 자세히 → `ansible-basics/` (예정)

### 컨테이너
- *imageに 모든 의존성*. *외부 install 안 필요*.
- 자세히 → [`docker-basics/`](../docker-basics/), [`kubernetes-basics/`](../kubernetes-basics/)

---

## 8. *RHEL 진영의 특수성*

### SELinux 기본 Enforcing
- 권한 충분한데 *접근 거부* → SELinux 의심 ([03 참조](03-filesystem-and-permissions.md))

### firewalld 기본 활성
- 새 포트 *기본 차단*. 명시적 allow 필요.

### dnf module
- 같은 패키지의 *여러 stream* (예: nginx 1.20 vs 1.22)
- `dnf module list nginx`, `dnf module enable nginx:1.22`

### subscription-manager (RHEL 진짜)
- Rocky/Alma는 없음 (RHEL 호환 무료)

---

## 9. *Ubuntu 진영의 특수성*

### snap
- *컨테이너화된 패키지 형식*. *시스템 라이브러리와 격리*.
- 시작 느림, 디스크 더 씀
- `snap list`, `snap remove ...`

### Universe / Multiverse
- *공식 외 저장소*. *enable 필요* (보통 기본 enable)

### unattended-upgrades
- *자동 보안 업데이트* 기본 설정 (Ubuntu Server)

---

## 10. 자주 헷갈리는 것

### 10-1. *디스크 100% 차서 못 SSH*
- *디스크 다 차면 ssh 세션도 안 됨* — *로그 못 씀*
- 예방: *log rotation*, *디스크 75% 알람*

### 10-2. *대용량 파일 삭제 후 free 명령에 반영 안 됨*
- 위 [02 + 03](02-process-and-signals.md) 참조 — *프로세스가 fd로 잡고 있음*
- `lsof | grep deleted` 후 프로세스 재시작

### 10-3. *rm -rf 사고*
- *상대 경로·환경변수* 사고 — `rm -rf $UNDEFINED/*` → `rm -rf /*`
- 예방: *항상 ls로 먼저 확인*, *quote 정확*, *Trash 도구 (trash-cli)*

### 10-4. *yum / apt가 *중간에 멈춤* — lock*
- `/var/run/yum.pid`, `/var/lib/dpkg/lock` 등
- 다른 프로세스 실행 중 (`ps aux | grep -E "yum|apt"`)
- 정말 stuck이면 *조심스럽게* lock 제거

### 10-5. *시간 안 맞아서 *TLS·인증 실패**
- *NTP 동기화 안 됨* — `timedatectl status`
- *컨테이너 시간*은 호스트와 동기 (시간 namespace는 별도 없음)

### 10-6. *cron이 *환경변수 없어서 안 됨**
- cron의 환경은 *최소* — PATH 짧음
- 해결: 스크립트 안에서 *전체 경로*, 또는 *환경 source*

---

## 🎤 면접 빈출 Q&A

### Q1. *서버가 갑자기 느려졌을 때* 디버깅 순서는?
> (위 Section 3 12단계). 핵심: *시스템 상위 (uptime/top) → 자원 별 (메모리/디스크/네트워크) → 문제 프로세스 좁히기 → strace/lsof로 깊이 → kernel 메시지 (dmesg)*. 컨테이너 환경이면 `kubectl describe`/`logs`/`events`. *외부 의존 (DB·외부 API) 상태*도 잊지 말기. Brendan Gregg의 *USE method (Util/Sat/Err) per 자원*도 유용.

### Q2. *디스크 사용량이 갑자기 차오름* — 어떻게 좁혀가나요?
> `df -h`로 *어느 마운트* 확인. 그 마운트의 *큰 디렉터리*: `du -sh /var/* | sort -h | tail`. *로그가 흔함* — `/var/log/` 안. *deleted but held* 파일도 의심: `lsof +L1`. 큰 파일 찾기: `find / -size +1G -type f 2>/dev/null`. 임시 완화 후 *log rotation·디스크 알람* 정비.

### Q3. *OOM Killer 발동 후 진단*?
> `dmesg | grep -i oom`, `journalctl -k | grep oom`. 출력에 *killed process 이름·PID·메모리 사용량·cgroup path*. *시스템 OOM*이면 시스템 전체 메모리 부족. *cgroup OOM*이면 *그 cgroup memory.max 초과* (시스템 메모리 남아도 발생). K8s면 *Pod OOMKilled* — limit 부족 또는 *cgroup-aware 안 한 런타임* (JVM 옛 버전 등).

### Q4. *Load Average 폭증인데 CPU는 한가함* — 가능 원인?
> *D-state 프로세스가 많음* — LA는 *runnable + uninterruptible (I/O 대기)*. *디스크 I/O 병목* 의심. `iostat -x 1`의 `await`·`%util`. `ps aux | awk '$8=="D"'` 로 D-state 프로세스 식별. NFS 끊김, 디스크 hang, DB의 *느린 쿼리* 등.

### Q5. RHEL/Rocky vs Ubuntu 패키지 매니저 차이?
> RHEL 진영 = *dnf* (RPM). Ubuntu/Debian = *apt* (DEB). 의존성 해석·repo 관리·signed package 다 비슷. 특수 차이: RHEL은 *SELinux 기본 Enforcing·firewalld 기본 활성·dnf module*, Ubuntu는 *snap·unattended-upgrades·universe repo*. 운영에선 *어느 진영인지* 알면 *특수성 대비* 가능.

### Q6. *재현성 있는 서버* 만들려면?
> *Immutable infrastructure* — 서버 변경 안 함, 변경 = *새 이미지 + 새 인스턴스*. AMI/VM 이미지 or 컨테이너 이미지. **+ Configuration management (Ansible)** — 선언적 상태, 멱등성. 둘 다 *서버를 cattle, not pets로* 다루기. 옛 SSH 직접 install 패턴은 *재현 불가*.

### Q7. *deleted but fd-held* 파일 — 어떻게 회복?
> `lsof | grep deleted` 또는 `lsof +L1` — *링크 카운트 0인 fd*. 출력에 *프로세스·fd 번호·파일 경로*. **해당 프로세스 재시작**하면 fd 해제 → 디스크 회복. 재시작 불가능하면 `/proc/{pid}/fd/{N}` 의 *그 fd로 직접 truncate* 가능 (위험).

---

## 🔗 Cross-reference

- **systemd / journalctl** → [01-systemd.md](01-systemd.md)
- **프로세스·시그널·OOM·LA** → [02-process-and-signals.md](02-process-and-signals.md)
- **파일시스템·권한** → [03-filesystem-and-permissions.md](03-filesystem-and-permissions.md)
- **cgroup·namespace (cgroup OOM 등)** → [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md)
- **컨테이너 환경 디버깅** → [`docker-basics/`](../docker-basics/), [`kubernetes-basics/`](../kubernetes-basics/)
- **Ansible로 패키지·서비스 선언** → `ansible-basics/` (예정)
- **관측성 (Prometheus·Grafana로 *상시 모니터링*)** → `observability-basics/` (예정)

---

## 📝 3줄 요약

1. *패키지 관리*는 dnf (RHEL) / apt (Ubuntu). 운영에선 *immutable infrastructure + IaC*가 정석 — 매번 install 패턴 회피.
2. *장애 디버깅 12단계 흐름* — 시스템 상위 → 자원별 (메모리/디스크/네트워크) → 프로세스 → syscall (strace/lsof) → kernel (dmesg). *어디서 시작할지*가 핵심.
3. *deleted but fd-held*, *cgroup OOM*, *D-state·LA 폭증*, *재현성 없는 서버 변경* 같은 *고전적 함정*을 알면 디버깅이 *마법이 아닌 작업*이 됨.

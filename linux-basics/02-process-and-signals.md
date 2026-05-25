# 02 — Process & Signals: 디버깅의 베이스

> **공식 근거:** `man 7 signal`, `man ps`, `man strace`, [Linux kernel docs — Process Management](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)
>
> **선행:** 없음. *모든 디버깅의 출발점*.

---

## 🎯 한 문장

> **Linux의 모든 실행은 *프로세스*. *fork로 복제 + exec로 교체*가 기본. PID·UID·시그널·메모리·자원 한도가 *진단의 5축*. 좀비·OOM Killer·load average 등 *고전적 함정*을 모르면 디버깅이 *마법*이 된다.**

---

## 1. 프로세스의 생애 — fork / exec / wait / exit

### fork(2)
- *현재 프로세스를 복제*. 부모와 자식이 *거의 동일* (PID만 다름)
- 자식: *부모의 메모리·fd·env 모두 copy* (실제론 Copy-on-Write)
- 호출 결과: 부모는 *자식 PID*, 자식은 *0* 받음 (분기점)

### exec(2)
- *현재 프로세스의 코드·메모리를 다른 프로그램으로 교체*
- *PID는 그대로*. *코드만 바뀜*.
- 흔한 패턴: fork → 자식에서 exec (= 새 프로그램 실행)

### wait(2)
- 부모가 *자식 종료 기다림*
- 자식의 *exit code 회수* + *프로세스 테이블 정리*

### exit(2)
- 프로세스 *종료*. exit code (0~255) 부모에 전달.
- 좀비 상태로 전환 (부모가 wait할 때까지)

### 흐름
```
shell (PID 1234)
  ├── fork()
  └── 자식 (PID 1235, 부모 복제본)
        └── exec("ls", ["ls", "-l"])  ← 메모리·코드 교체, PID 1235 유지
        └── (ls 실행)
        └── exit(0)
        └── (zombie 상태)
  └── wait()  ← 자식 종료 회수
```

---

## 2. 좀비 (Zombie) — *종료됐는데 부모가 안 회수*

### 정의
- 자식이 *exit*했으나 *부모가 wait 안 함*
- 프로세스 테이블에 *exit code만 남음*. 메모리·fd는 회수됨.
- `ps`의 STAT 컬럼에 *Z* 표시

### 왜 문제?
- 좀비 자체는 자원 거의 안 먹음 (PID slot 1개)
- *PID slot 고갈* → 새 프로세스 못 만듦
- *부모 코드 버그*의 신호 — 자식 관리 실패

### 해결
- 부모가 *wait* 호출하거나 *SIGCHLD signal handler*에서 처리
- 부모 죽으면 *PID 1 (init)*이 *자식 인계* → init이 wait → 좀비 해소
- → 컨테이너 안 *PID 1이 부실하면* 좀비 누적 (shell script로 PID 1 띄울 때 흔함)

→ [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md) 의 *PID 1 함정*과 같은 문제.

---

## 3. Process State — `ps`의 STAT 컬럼

```bash
ps aux
# USER  PID  %CPU %MEM  STAT  ...
```

| STAT | 의미 |
|---|---|
| **R** | Running 또는 runnable (CPU 잡고 있거나 잡을 준비) |
| **S** | Interruptible Sleep (대기 중 — 정상) |
| **D** | *Uninterruptible Sleep* (보통 I/O 대기). *kill 안 됨*. |
| **Z** | Zombie |
| **T** | Stopped (SIGSTOP 받음) 또는 traced |
| **X** | Dead (아주 잠깐 보임) |

### D-state의 *진짜* 의미
- 보통 *디스크 I/O 대기*
- *kill -9 도 안 듣음* — 커널이 *I/O 끝날 때까지 기다림*
- D-state가 *오래 지속* → 보통 *NFS 끊김, 디스크 hang, NVMe 펌웨어 버그* 등
- *Load Average에 포함됨* → CPU 사용률 낮은데 LA 폭증의 흔한 원인

---

## 4. Load Average — *CPU 사용률이 아니다*

```bash
uptime
# 14:32  up 5 days, load average: 1.05, 0.85, 0.45
```

3 숫자 = *1분, 5분, 15분 평균*.

### 정의
- *runnable + uninterruptible (D-state)* 프로세스 *평균 수*
- *CPU 코어 수와 비교*해서 해석

### 해석
- 4 core machine에서 LA 4.0 → *CPU 100% 사용 (포화)*
- LA 8.0 → *2배 over* (큐 누적, 응답 지연)
- LA 0.5 → *25% 사용*

### CPU 사용률과 다른 이유
- LA는 *D-state 포함* — *디스크 I/O 대기 프로세스*도 포함
- → CPU `idle`인데 LA 높음 → *I/O 병목 의심*
- 확인: `iostat -x 1`, `iotop`로 *디스크 I/O*, `vmstat 1`의 *wa 컬럼*

---

## 5. 시그널 — 프로세스 간 통신의 *가장 단순한 형태*

`man 7 signal`. 자주 쓰는 것:

| Signal | # | 의미 | 무시 가능? |
|---|---|---|---|
| **SIGTERM** | 15 | *정중한 종료 요청* — 정리·flush 후 자발 종료 | ✅ |
| **SIGKILL** | 9 | *즉시 종료* — handler 무시 (커널이 강제) | ❌ |
| **SIGHUP** | 1 | *config 다시 읽기* (관습, daemon이 인터프리트) | ✅ |
| **SIGINT** | 2 | Ctrl+C — interactive interrupt | ✅ |
| **SIGSTOP** | 19 | *정지* (Ctrl+Z의 일부). 무시 불가. | ❌ |
| **SIGCONT** | 18 | STOP에서 재개 | ✅ |
| **SIGCHLD** | 17 | *자식 상태 변경* 알림 | ✅ (보통 handler 등록) |
| **SIGUSR1/2** | 10/12 | *앱 정의* (보통 reload·dump 등) | ✅ |
| **SIGSEGV** | 11 | Segmentation fault (memory access 위반) | ✅ (보통 core dump 후 종료) |
| **SIGABRT** | 6 | abort() — 자살. assert 실패 등. | ✅ |
| **SIGPIPE** | 13 | *읽는 쪽 끊긴 pipe에 쓰려고 함* | ✅ |
| **SIGALRM** | 14 | alarm() timer | ✅ |

### kill 명령
```bash
kill 1234              # SIGTERM (기본)
kill -9 1234           # SIGKILL
kill -TERM 1234        # 이름으로
kill -HUP nginx        # pidof 자동
killall nginx          # 이름으로 모든 매칭
pkill -f "java.*MyApp" # 패턴 매칭
```

### Graceful Shutdown의 *진짜* 의미
- SIGTERM 받음 → *현재 요청 완료까지 기다림* + *새 요청 안 받음* + *리소스 정리* → exit
- *시간 제한*: 보통 *30초* (`systemctl stop` 또는 K8s `terminationGracePeriodSeconds`)
- 시간 초과 → *SIGKILL 강제*

### 컨테이너에서의 함정
- *Shell script로 띄운 PID 1*은 *SIGTERM 자식에 전파 안 함*
- → exec form 또는 tini 사용 ([`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md))

---

## 6. OOM Killer — *메모리 부족 시 누구를 죽일지*

### 발동 조건
- *시스템 전체 메모리 부족* (swap 포함 고갈)
- → kernel이 *OOM Killer 실행*
- *cgroup 메모리 한도 초과* → *그 cgroup 안에서* OOM Killer

### 점수 시스템 — `oom_score`
- 각 프로세스에 *0~1000 점수*
- 메모리 *많이 쓰는 + 자식 많은 + 우선순위 낮은* 프로세스 점수 ↑
- 가장 *점수 높은* 프로세스 *kill*

확인:
```bash
cat /proc/{pid}/oom_score        # 현재 점수
cat /proc/{pid}/oom_score_adj    # 점수 조정 (-1000 ~ 1000)
```

### 보호하고 싶은 프로세스
- *oom_score_adj = -1000* → *OOM Killer가 절대 안 죽임*
- systemd unit: `OOMScoreAdjust=-1000`
- *주의*: 이거 잘못 쓰면 *시스템 hang*. 진짜 critical만.

### OOM 발생 로그
```bash
dmesg | grep -i oom
journalctl -k | grep -i oom
# 출력에 *killed process*, *total-vm*, *anon-rss* 정보
```

→ 컨테이너에서 OOM 자주 → *cgroup memory limit 너무 작거나* / *메모리 누수* / *JVM이 호스트 메모리 기준으로 힙 잡음*

---

## 7. 메모리 — *RSS·VSS·Cache의 차이*

`top` / `ps`의 메모리 컬럼:

| | 의미 |
|---|---|
| **VIRT (VSS)** | *가상 메모리 크기* — 실제 안 쓴 페이지도 포함 |
| **RES (RSS)** | *실제 물리 메모리 사용량* — *진짜로 쓰는 양* |
| **SHR** | *공유 메모리* (다른 프로세스와 공유 가능, 라이브러리 등) |
| **MEM%** | RSS / 전체 메모리 |

→ *VIRT가 크다고 진짜 메모리 많이 쓰는 게 아님*. RSS가 진짜 지표.

### Free / Available / Buffers / Cache
```bash
free -h
#               total   used   free   shared  buff/cache  available
# Mem:          15Gi    8.0Gi  500Mi  100Mi   6.5Gi       7.0Gi
```

- *free*만 보면 *500Mi*. 하지만 *available은 7.0Gi*.
- 차이: *buff/cache 6.5Gi*는 *언제든 메모리 필요 시 회수 가능*
- *available* = *진짜 가용 메모리*. **여기를 보라.**
- "메모리 다 찼다" 라는 흔한 오해의 원인.

---

## 8. CPU — top·htop·vmstat

### top / htop
```bash
top
# load average: ...
# %Cpu(s):  20.0 us,  5.0 sy,  0.0 ni,  75.0 id,  0.0 wa,  0.0 hi,  0.0 si
```

| 컬럼 | 의미 |
|---|---|
| **us** | user space (앱 코드) |
| **sy** | system (커널) |
| **ni** | nice (낮은 우선순위) |
| **id** | idle |
| **wa** | I/O wait (디스크) — *높으면 디스크 병목* |
| **hi** | hardware interrupt |
| **si** | software interrupt |
| **st** | steal (가상화: hypervisor가 빼앗아간 시간) |

### vmstat
```bash
vmstat 1 5     # 1초 간격, 5번
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```

- *r*: runnable 프로세스 — *CPU 부족 의심* (코어 수 초과 시)
- *b*: blocked (D-state) — *I/O 부족 의심*
- *si/so*: swap in/out — *swap 발생 = 메모리 부족 신호*
- *bi/bo*: block I/O — *디스크 사용량*

---

## 9. strace / lsof — *프로세스 내부 들여다보기*

### strace — *system call 추적*
```bash
strace -p 1234                    # 실행 중 프로세스 attach
strace -e open,read,write ls      # 특정 syscall만
strace -c ls                      # 통계 (어떤 syscall이 얼마나)
strace -f myapp                   # fork 따라가기
```

→ *"왜 응답 안 함"*, *"왜 파일 못 읽음"* 디버깅의 무기.

### lsof — *열린 file descriptor*
```bash
lsof -p 1234                      # 특정 프로세스
lsof /var/log/myapp.log           # 그 파일 누가 열고 있나
lsof -i :8080                     # 8080 포트 누가 listen
lsof -i tcp                       # 모든 TCP 연결
```

→ *"파일 삭제했는데 디스크 안 줄어듦"* → *프로세스가 fd로 잡고 있음* (lsof로 확인 후 그 프로세스 재시작)

### ss — netstat 후계자
```bash
ss -tunlp                         # TCP/UDP listening + 프로세스
ss -tan                           # 모든 TCP 연결 상태
ss -s                             # 통계
```

---

## 10. 자주 헷갈리는 것

### 10-1. *kill -9를 너무 자주*
- SIGKILL은 *graceful 안 됨* — 리소스 leak, lock 안 풀림, 일부 commit 누락
- *항상 SIGTERM 먼저, 안 되면 SIGKILL*
- systemd는 자동으로 *SIGTERM 후 timeout → SIGKILL*

### 10-2. *D-state 프로세스를 kill*
- *D-state는 kill 안 됨*. SIGKILL도 마찬가지.
- *I/O 끝날 때까지 기다려야*
- 영원히 안 끝나면 (NFS 끊김 등) *리부팅 외 해결 없음*

### 10-3. *Process list가 자기 자신을 안 보여줌*
- `ps` 자체가 *fork → exec*. ps 실행 시점엔 *이전 ps는 종료됨*.

### 10-4. *PID 1을 kill 시도*
- 일반 user는 권한 없음
- root여도 *SIGTERM/SIGKILL이 보호된 init에 의미 X* (Linux는 PID 1에 *기본 신호 무시*)
- 진짜 종료하려면 `systemctl poweroff`/`reboot`

### 10-5. *child가 *parent 죽으면 잘 죽지 않음**
- parent 죽으면 child는 *고아*. *init이 부모 인계*.
- child를 같이 죽이려면 *process group* 활용 (`setpgid`, `killpg`) 또는 *cgroup으로 일괄*

### 10-6. *htop의 *MEM% 합이 100% 넘음**
- *공유 메모리* 때문 — 라이브러리 등은 *여러 프로세스에 중복 계산*
- RSS만으로 *실제 사용량 합산* 부정확. *기본적으론 추정치*.

---

## 🎤 면접 빈출 Q&A

### Q1. `ps aux`에서 STAT 컬럼 R/S/D/Z 의미?
> **R** Running/Runnable (CPU 잡고 있거나 잡을 준비). **S** Interruptible Sleep (대기, 정상). **D** Uninterruptible Sleep (I/O 대기, *kill 안 됨*). **Z** Zombie (exit 후 부모 wait 대기). D-state 길게 가면 *NFS/디스크 hang* 의심, Z 누적은 *부모 wait 안 함* 신호.

### Q2. 좀비 프로세스는 *왜* 생기나? 어떻게 처리?
> 자식이 exit했는데 부모가 wait 안 함 → 프로세스 테이블에 *exit code만 남은 상태*. *PID slot 차지*. 부모 코드가 SIGCHLD handler 또는 wait 호출해야. 부모 죽으면 *init이 인계 → wait → 해소*. 컨테이너의 *PID 1 부실* (shell script로 띄움)이 흔한 원인 — `tini`/`dumb-init` 또는 `exec form`으로 해결.

### Q3. OOM Killer는 어떻게 *어떤 프로세스를 죽일지* 결정?
> 각 프로세스에 `oom_score` (0~1000) — *메모리 많이 쓰는 + 자식 많은 + 우선순위 낮은* 프로세스가 높음. 가장 높은 거 kill. `oom_score_adj` (-1000 ~ 1000)로 조정 가능. *시스템 critical 프로세스*는 `OOMScoreAdjust=-1000`으로 보호. `dmesg | grep oom`으로 사건 로그 확인.

### Q4. Load Average가 의미하는 것? CPU 사용률과 다른 이유?
> *runnable + uninterruptible (D-state) 프로세스 평균 수* (1/5/15분). *CPU 사용률은 *실제 CPU 시간 비율*. LA는 *D-state도 포함* — *디스크 I/O 대기 프로세스도 LA에 포함*. → CPU idle인데 LA 높음 = *I/O 병목*. `iostat -x 1`, `vmstat 1`의 *wa 컬럼*로 확인.

### Q5. Graceful Shutdown 흐름은? *컨테이너에서 안 됨* 원인?
> SIGTERM → 앱이 *현재 요청 처리 완료 + 새 요청 안 받음 + 리소스 정리* → exit. 시간 초과 → SIGKILL 강제. **컨테이너 안 안 되는 흔한 원인**: PID 1이 *shell script* — bash는 SIGTERM을 *자식에 전파 안 함*. 해결: *exec form ENTRYPOINT* (`ENTRYPOINT ["nginx", "-g", "daemon off;"]`), 또는 `tini`/`dumb-init` 사용.

### Q6. *서버가 갑자기 느려졌을 때* 디버깅 순서?
> (1) `uptime` — LA. (2) `top` / `htop` — CPU·메모리·D-state 프로세스. (3) `vmstat 1` — wa·si/so 컬럼. (4) `iostat -x 1` — 디스크 I/O. (5) `free -h` — 메모리 (available 컬럼). (6) `ss -tnp` — 네트워크 연결. (7) `dmesg` / `journalctl -k` — kernel 메시지 (OOM·hardware). (8) `strace -p {pid}` — 특정 프로세스 syscall.

### Q7. RSS vs VIRT 차이?
> *VIRT (VSS)*는 *가상 메모리 크기* — 매핑된 모든 페이지, 실제 안 쓴 것도 포함. *RES (RSS)*는 *실제 물리 메모리 사용량*. VIRT 큰 게 *진짜 메모리 많이 쓰는 게 아님*. **RSS만 봐라.** 단, RSS도 *공유 메모리 중복 계산* 가능 — 정확한 총합은 *PSS (Proportional Set Size)* 또는 *cgroup의 memory.current*.

---

## 🔗 Cross-reference

- **systemd가 process tree·cgroup 관리** → [01-systemd.md](01-systemd.md)
- **cgroup memory limit과 OOM Killer** → [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md)
- **PID namespace + PID 1** → [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md), [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)
- **K8s Pod의 OOMKilled** → [`kubernetes-basics/02`](../kubernetes-basics/02-workloads.md)
- **장애 디버깅 통합 체크리스트** → [05-package-and-troubleshooting.md](05-package-and-troubleshooting.md)

---

## 📝 3줄 요약

1. *fork·exec·wait·exit*가 프로세스 생애. *zombie·D-state·SIGKILL 안 됨* 같은 함정을 모르면 디버깅이 마법.
2. *Load Average는 CPU 사용률 X* — runnable + D-state 평균. *CPU idle인데 LA 높음 = I/O 병목*. *RSS만 진짜 메모리*.
3. *Graceful shutdown은 SIGTERM 후 timeout → SIGKILL*. 컨테이너 안 PID 1 shell script가 *시그널 전파 안 함* 가장 흔한 함정.

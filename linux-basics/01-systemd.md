# 01 — systemd: unit·service·timer·journal·target

> **공식 근거:** `man systemd`, `man systemd.unit`, `man systemd.service`, [systemd docs](https://www.freedesktop.org/wiki/Software/systemd/)
>
> **선행:** [02-process-and-signals.md](02-process-and-signals.md) (PID/signal 기본)

---

## 🎯 한 문장

> **systemd = *PID 1 init + 서비스 관리 + 의존성 + 소켓 활성화 + cgroup 관리 + journal + timer*. RHEL/Ubuntu 표준. *unit*이 모든 것의 단위.**

---

## 1. systemd가 *왜* 표준이 됐는가

옛 SysV init의 한계:
- *순차 실행* — 부팅 느림
- *의존성 명시 부족* — 순서 hardcode
- *프로세스 추적 약함* — daemonize 후 *부모 잃은 자식 추적 X*
- *시그널·재시작·로깅 다 따로*

systemd 해결:
- **병렬 부팅** (의존성 그래프 기반)
- **명시적 dependency** (Wants/Requires/After/Before)
- **cgroup으로 *서비스 전체 프로세스 트리* 추적·관리**
- **journald + 통합 로깅**
- **socket activation** (서비스 시작 *없이* 포트 listen, 첫 연결 시 시작)
- **timer** (cron 대체)

---

## 2. Unit — 모든 것의 단위

systemd가 관리하는 *모든 것이 unit*. 12종.

| Unit 종류 | 확장자 | 무엇 |
|---|---|---|
| **service** | `.service` | *서비스 (daemon)*. 가장 흔함. |
| **socket** | `.socket` | socket activation 단위 |
| **target** | `.target` | *grouping*. runlevel 대체 (multi-user.target 등) |
| **timer** | `.timer` | cron 대체 |
| **mount** | `.mount` | 마운트 포인트 |
| **automount** | `.automount` | 첫 접근 시 마운트 |
| **device** | `.device` | 디바이스 (udev로 자동) |
| **path** | `.path` | 파일 변경 감지 → 서비스 trigger |
| **swap** | `.swap` | swap 영역 |
| **slice** | `.slice` | *cgroup slice* (자원 그룹) |
| **scope** | `.scope` | systemd가 *외부에서 생성된 프로세스* 관리 |
| **snapshot** | `.snapshot` | (deprecated) |

### Unit 파일 위치
| 위치 | 우선순위 |
|---|---|
| `/etc/systemd/system/` | 가장 높음 (운영자 정의) |
| `/run/systemd/system/` | 런타임 생성 |
| `/lib/systemd/system/` (또는 `/usr/lib/...`) | 패키지 기본 |

→ 패키지의 기본 unit을 *override* 하려면 `/etc/systemd/system/{name}.service.d/override.conf` 작성:
```bash
systemctl edit nginx        # ← override 파일 자동 생성·편집
```

---

## 3. Service Unit — 가장 흔한 단위

```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --port 8080
Restart=on-failure
RestartSec=5s
Environment="LOG_LEVEL=info"
EnvironmentFile=-/etc/myapp/env

# 자원 제한 (cgroup)
MemoryMax=512M
CPUQuota=200%
TasksMax=100

# 보안 강화
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

### `Type=` 옵션
| Type | 의미 |
|---|---|
| `simple` (기본) | ExecStart의 프로세스가 *주 프로세스*. fork 안 함. |
| `forking` | 옛 daemon 패턴. 부모는 즉시 종료, 자식이 주 프로세스. |
| `oneshot` | 한 번 실행 후 종료 (스크립트 같은) |
| `notify` | 프로세스가 *준비됐다고 systemd에 알림* (`sd_notify`) — 정확한 ready 시점 |
| `exec` | simple과 비슷하나 *exec까지 기다림* |
| `dbus` | D-Bus 이름 등록 시 ready 간주 |

→ 현대 앱은 *simple* 또는 *notify*. *forking은 옛 daemon 호환용*.

### Restart 정책
| 값 | 의미 |
|---|---|
| `no` (기본) | 안 함 |
| `on-failure` | 비정상 종료 시 |
| `on-abnormal` | 신호·OOM 등으로 죽었을 때 |
| `on-watchdog` | watchdog timeout 시 |
| `always` | *어떻게 종료되든* 재시작 |
| `unless-stopped` | systemctl stop 외엔 항상 |

→ 운영 서비스는 보통 `on-failure` 또는 `always`. *Restart loop 함정* — `StartLimitIntervalSec`·`StartLimitBurst` 로 *재시작 폭주 차단*.

### Dependency
| Directive | 의미 |
|---|---|
| `Requires=X` | X 필수. X 실패 시 *이 unit도 실패*. |
| `Wants=X` | X 시작 시도. X 실패해도 *이 unit은 계속*. |
| `After=X` | X *시작 완료 후*에 시작 |
| `Before=X` | X *시작 전*에 시작 |
| `Conflicts=X` | X와 *동시 실행 불가* |
| `BindsTo=X` | Requires + X 죽으면 *이 unit도 죽음* |

→ *Requires vs Wants*가 가장 헷갈림. *강 의존성은 Requires*, *약 의존성은 Wants*.

→ *After/Before는 *순서만*. dependency 자체는 Requires/Wants가 만듦*.

---

## 4. systemctl — 기본 명령

```bash
# 상태
systemctl status nginx
systemctl is-active nginx
systemctl is-enabled nginx

# 시작·정지·재시작
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx          # SIGHUP (설정 reload, 재시작 X)

# 부팅 시 자동 시작
systemctl enable nginx
systemctl disable nginx
systemctl enable --now nginx    # enable + start 한 번에

# 의존성 보기
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx  # 누가 nginx에 의존

# 모든 unit 보기
systemctl list-units --type=service
systemctl list-units --failed   # 실패한 unit만

# 부팅 분석
systemd-analyze                  # 부팅 시간
systemd-analyze blame            # 각 service 시작 시간
systemd-analyze critical-chain   # 부팅 critical path
```

---

## 5. Target — runlevel의 후계자

```bash
systemctl get-default            # 현재 기본 target
systemctl set-default multi-user.target

systemctl isolate rescue.target  # 즉시 전환
```

| Target | 의미 |
|---|---|
| `multi-user.target` | 멀티유저 (CLI). 서버 표준. |
| `graphical.target` | 멀티유저 + GUI |
| `rescue.target` | 단일유저 모드 (복구) |
| `emergency.target` | 최소 (rootfs만 마운트) |
| `reboot.target` / `poweroff.target` / `halt.target` | 종료 동작 |
| `network-online.target` | 네트워크 *사용 가능 상태* (DHCP 완료 등) |

`WantedBy=multi-user.target` 의 의미: *서비스가 부팅 시 자동 시작*되려면 이 target이 *원함*이라고 등록.

---

## 6. Journal — 통합 로깅

```bash
# 전체
journalctl

# 특정 service
journalctl -u nginx
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "yesterday" --until "today 12:00"

# 실시간 (tail -f)
journalctl -u nginx -f

# 특정 PID
journalctl _PID=1234

# kernel 메시지
journalctl -k
journalctl -k --since today

# 우선순위
journalctl -u nginx -p err       # err 이상 (err·crit·alert·emerg)

# 부팅별
journalctl --list-boots
journalctl -b -1                 # 직전 부팅
journalctl -b 0                  # 현재 부팅
```

### Journal storage
- 기본 `/var/log/journal/` (영구). 없으면 `/run/log/journal/` (휘발).
- `journald.conf` 의 `Storage=persistent` 로 영구화.
- 크기 제한 — `SystemMaxUse=500M` 등

### Journal vs rsyslog
- journald = systemd 표준, *binary log*, 메타데이터 풍부 (`_PID`, `_UID`, `_UNIT`...)
- rsyslog = 전통 syslog. 텍스트. 외부 server 전송에 강함.
- 보통 *journald 기본 + rsyslog로 forwarding* 패턴

---

## 7. Timer — cron 대체

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=*-*-* 02:00:00      # 매일 02:00
Persistent=true                # 시스템 꺼졌던 시간 보상 (놓친 trigger 따라잡기)
RandomizedDelaySec=300         # 0~300초 random delay (분산)

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup task

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```bash
systemctl enable --now backup.timer
systemctl list-timers
```

### Timer vs cron
| | cron | systemd timer |
|---|---|---|
| 로그 | 잘 안 됨 (직접 redirect) | journald 자동 |
| 부팅 후 놓친 trigger | 무시 | `Persistent=true`로 보상 |
| 의존성 | 없음 | unit dependency 사용 |
| 분산 (jitter) | 별도 도구 | `RandomizedDelaySec` |
| 운영 가시성 | 약 | `systemctl list-timers`로 한 눈에 |

→ 현대 운영은 *systemd timer* 권장.

---

## 8. Socket Activation — *지연 시작*

```ini
# myapp.socket
[Socket]
ListenStream=8080

[Install]
WantedBy=sockets.target
```

```ini
# myapp.service
[Service]
ExecStart=/opt/myapp/server
```

- systemd가 *8080 포트 listen*만 함
- *첫 연결* 도착 시 systemd가 *myapp.service 시작* + *fd를 그 프로세스에 전달*
- 이전 *xinetd*의 아이디어
- 부팅 시간 단축, 사용 안 하는 서비스는 *실행 안 함*

---

## 9. systemd와 cgroup — 자원 관리

systemd가 *모든 service를 자동으로 cgroup에 배치*. 그래서:
```bash
systemd-cgls                     # cgroup 트리 보기
systemd-cgtop                    # cgroup별 자원 사용량 실시간
```

→ docker 없이도 *service별 자원 한도* 가능:
```ini
[Service]
MemoryMax=512M
CPUQuota=200%                    # 2 CPU 분량
TasksMax=100
IOWeight=500
```

→ [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md) 와 직결.

---

## 10. 자주 헷갈리는 것

### 10-1. `restart` vs `reload`
- `restart` — 프로세스 *완전 재시작* (downtime 있음)
- `reload` — *SIGHUP*. 설정만 다시 읽음 (대부분 앱이 지원).
- nginx는 `reload`로 *graceful config reload*

### 10-2. `enable` vs `start`
- `enable` — *부팅 시 자동 시작* (영구)
- `start` — *지금 시작* (현재 세션만)
- `enable --now` — 둘 다

### 10-3. `Requires` 망상 — *자식이 죽어도 부모는 그대로*
- `Requires=X` 는 *시작 시점*에만 강제. 시작 후 X 죽어도 *이 unit은 계속*.
- *X 죽으면 이 unit도 죽이려면* `BindsTo=X` 추가.

### 10-4. journal이 *디스크 다 먹음*
- 기본 `SystemMaxUse=` 미설정 → *디스크 10%까지* 사용
- 명시: `SystemMaxUse=1G`, `SystemMaxFileSize=100M`, `MaxRetentionSec=30day`

### 10-5. *override 했는데 안 먹힘*
- `systemctl daemon-reload` 안 하면 *systemd 안 다시 읽음*
- override 후 `systemctl daemon-reload && systemctl restart nginx`

### 10-6. *unit 파일 문법 error*
- `systemctl status` 안 알려줌
- `journalctl -xe`로 *전체 에러*
- `systemd-analyze verify {file}` 로 *문법 검증*

---

## 🎤 면접 빈출 Q&A

### Q1. systemd는 왜 SysV init을 대체했나요?
> SysV는 *순차 부팅 (느림), 의존성 명시 부족, 프로세스 추적 약함, 로깅·재시작·시그널 도구 분산*. systemd는 *병렬 부팅 (의존성 그래프), 명시적 dependency (Requires/Wants/After), cgroup으로 service 프로세스 트리 추적, journald 통합 로깅, socket activation, timer*. 운영 가시성·효율 대폭 향상.

### Q2. Requires와 Wants 차이?
> *Requires*는 *강 의존성* — 의존 unit 실패 시 *이 unit도 실패*. *Wants*는 *약 의존성* — 의존 unit 시작 시도하나 *실패해도 이 unit은 계속*. 둘 다 *시작 순서는 정의 안 함* — 순서는 *After/Before*로 별도. `BindsTo`는 *Requires + 동기 종료*.

### Q3. `restart` vs `reload` 차이?
> *restart*는 프로세스 *완전 재시작* (downtime). *reload*는 *SIGHUP* — 설정만 다시 읽음 (앱이 지원해야). nginx는 `reload`로 *graceful config reload* (기존 연결 유지). 운영에서 *설정 변경 후 reload*가 표준.

### Q4. journalctl로 *어제 4시 ~ 5시 nginx 에러* 보기?
> `journalctl -u nginx --since "yesterday 04:00" --until "yesterday 05:00" -p err`. `-p err`은 err 이상 우선순위. `-u`로 특정 unit, `--since/--until`로 시간 범위. `-b -1`로 직전 부팅, `-f`로 실시간.

### Q5. systemd timer가 cron보다 *좋은 점*은?
> (1) *journald 자동 로깅*. (2) `Persistent=true`로 *시스템 꺼졌던 시간 보상* — 부팅 후 놓친 trigger 따라잡음. (3) *unit dependency 사용 가능* — 다른 service 시작 후에만 실행 등. (4) `RandomizedDelaySec`로 *부하 분산*. (5) `systemctl list-timers`로 *한 눈에 가시성*.

### Q6. service unit에서 *cgroup 자원 한도*를 어떻게?
> `[Service]` 섹션의 `MemoryMax=512M`, `CPUQuota=200%` (2 CPU 분량), `TasksMax=100`, `IOWeight=500`. systemd가 *자동으로 service별 cgroup 생성·적용*. docker 없이도 *프로덕션 service에 자원 제한 강제* 가능. `systemd-cgls` / `systemd-cgtop`으로 확인.

### Q7. *서버 부팅이 느림* — 어디서 시작?
> `systemd-analyze` (전체 부팅 시간), `systemd-analyze blame` (각 service 소요 시간 sorted), `systemd-analyze critical-chain` (부팅 critical path — 한 unit 짧아도 *직전 의존 unit이 느리면* 영향). 가장 느린 unit의 *After dependency* 의심.

---

## 🔗 Cross-reference

- **프로세스·시그널·PID 1** → [02-process-and-signals.md](02-process-and-signals.md)
- **cgroup 깊이 (systemd가 활용)** → [04-cgroup-and-namespace.md](04-cgroup-and-namespace.md)
- **컨테이너의 PID 1 패턴** → [`docker-basics/02`](../docker-basics/02-namespace-and-cgroup.md)
- **K8s에선 systemd 안 씀** — 컨테이너 안 *직접 실행*. 노드의 *kubelet·containerd는 systemd로 운영*.
- **Ansible로 systemd unit 배포** → `ansible-basics/` (예정)

---

## 📝 3줄 요약

1. systemd = *PID 1 + 서비스 매니저 + cgroup + journal + timer + socket activation*. 모든 게 *unit*. Requires(강 의존성) / Wants(약 의존성) / After(순서)를 구분.
2. service unit에 *cgroup 자원 한도 + 보안 강화* 가능 — docker 없이도 production-grade.
3. journal·timer는 *전통 syslog·cron 대체*. 가시성·재현성·운영 효율 우위.

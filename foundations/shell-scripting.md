# Shell Scripting — fork/exec부터 멱등성까지

> bash는 *도구* 가 아니라 *Linux의 인터페이스*. 그래서 [[declarative-and-reconciliation]]의 멱등성 가정이 가장 자주 깨지는 자리이고,
> container entrypoint·k8s lifecycle hook·ansible task·CI workflow가 모두 만나는 공용어다.

---

## 1. Fork/Exec와 Subshell — Shell 작동의 근본

### Process model

shell이 명령어를 실행하는 두 가지 방법:

| 방식 | 어떻게 | 예시 |
|---|---|---|
| **External command** | `fork()` 로 자식 프로세스 생성 → 자식이 `execve()` 로 바이너리 교체 | `ls`, `grep`, `awk` |
| **Builtin** | shell 자체가 실행 (fork 없음) | `cd`, `export`, `read`, `[[ ]]` |

`cd /tmp` 가 *별도 프로세스에서* 실행되면 부모 shell의 working directory가 안 바뀐다. 그래서 `cd`는 builtin이어야 한다. 같은 이유로 `export VAR=x | cat` 의 `export`는 subshell에서 실행되어 *외부로 빠져나가지 못한다*.

### Subshell의 형태

```bash
(cmd)        # subshell — fork됨, env·cwd 변경이 부모에 영향 없음
{ cmd; }     # group command — 같은 shell, env·cwd 변경이 부모에 남음
cmd | cmd2   # 파이프 양쪽 모두 subshell (bash 기본)
$(cmd)       # command substitution — subshell
```

### 함정: 파이프 안의 변수 변경

```bash
count=0
seq 1 10 | while read n; do count=$((count+1)); done
echo $count   # 0 (10이 아니다)
```

`while`이 파이프 오른쪽에 있어 *subshell* 에서 돈다. `count` 변경은 subshell 안에서만 일어나 부모에 안 남는다. 해결:

```bash
count=0
while read n; do count=$((count+1)); done < <(seq 1 10)  # process substitution
echo $count   # 10
```

이걸 모르면 ansible playbook의 `shell:` 모듈에서 *디버깅 불가능한 침묵 버그* 가 된다.

---

## 2. `set -euo pipefail` — 정확한 의미론

방어적 스크립트의 표준 prefix:

```bash
set -euo pipefail
```

각 옵션이 *정확히* 무엇을 하는지 모르면 *함정이 더 커진다*.

### `-e` (errexit) — exit 1+ 인 명령이 나오면 즉시 종료

**하지만 다음 컨텍스트에선 안 죽는다**:

```bash
if cmd; then ...    # if/while/until 조건의 명령
cmd && other        # && / || 좌측
cmd | other         # 파이프 중간 (마지막만 봄, pipefail 없으면)
! cmd               # 부정
```

→ "왜 errexit인데 안 죽지?" 의 대부분 원인. errexit는 **확인되지 않은 실패** 에만 반응.

### `-u` (nounset) — 미정의 변수 참조 시 종료

```bash
echo "$UNDEFINED"   # error
echo "${UNDEFINED:-default}"  # OK — default 사용
echo "${UNDEFINED-}"          # OK — 빈 문자열
```

`"$@"` 가 비어 있어도 `-u` 가 죽지 않는 게 일반적이지만, bash 4.4 미만에선 *죽었다*. CI 이미지 bash 버전 차이로 *프로덕션만 깨지는* 버그가 종종.

### `-o pipefail` — 파이프 중 *어느* 명령이 실패해도 전체 exit code가 0이 아님

```bash
set -o pipefail
false | true        # exit 1 (pipefail 없으면 exit 0)
```

기본 동작은 *마지막 명령의 exit code* 만 보기 때문에, `curl ... | jq ...` 에서 curl 실패가 묻힌다. pipefail은 그걸 막는다.

### `-x` (xtrace) — 실행되는 명령을 stderr에 출력 (디버깅용)

운영 스크립트는 끄고, 디버깅 시 `bash -x script.sh` 또는 `set -x` ... `set +x` 로 부분 활성.

---

## 3. Signal과 Trap — 청소·PID 1 문제

### Trap의 기본

```bash
trap 'echo cleanup; rm -f /tmp/lock' EXIT
trap 'echo got SIGTERM; exit' TERM
```

`EXIT` 는 *정상 종료·시그널·set -e 종료* 모두에서 실행 → **청소 코드의 표준 자리**.

### PID 1 문제 (container의 영원한 함정)

container의 PID 1 프로세스는 두 가지 *특수 책임* 을 진다:

1. **시그널 forwarding** — `docker stop` 은 PID 1에 SIGTERM을 보낸다. PID 1이 자식들에게 전달 안 하면 자식은 영원히 종료 안 됨.
2. **Zombie reaping** — 자식 프로세스가 종료하면 PID 1이 `wait()` 해야 함. 안 하면 좀비 누적.

bash script를 PID 1로 쓰면:
- 기본적으로 *시그널을 자식에게 forwarding 안 함*
- `wait` 호출 안 하면 좀비

해결:
- `tini`·`dumb-init` 같은 *작은 init* 을 PID 1로
- 또는 script에서 명시적으로 `trap 'kill -TERM $child' TERM; wait $child`
- k8s에서 `terminationGracePeriodSeconds` 가 *PID 1의 시그널 처리에 의존* 한다는 사실 기억

→ [[declarative-and-reconciliation]] §3의 control loop이 graceful shutdown을 가정하는데, PID 1이 시그널 못 받으면 *전제가 깨진다*.

### Trap의 함정

```bash
trap 'echo $LINENO' ERR    # ERR trap은 errexit에 안 잡힌 곳에서만 발동
trap '' INT                # SIGINT 무시 (Ctrl+C 무효화)
trap - INT                 # 기본 동작 복원
```

`EXIT` trap은 *한 번만* 실행. 함수 안에서 `trap`을 다시 설정하면 *전역 trap을 덮어쓴다* (local 아님).

---

## 4. FD·Redirect·Here-Doc — `2>&1` 순서 함정

### File descriptor 기본

- `0` = stdin, `1` = stdout, `2` = stderr
- `&` = "fd로 해석해라" 의미

### `2>&1` 순서 함정

```bash
cmd > file 2>&1     # OK: stdout→file, 그다음 stderr→stdout(=file). 둘 다 file로.
cmd 2>&1 > file     # 함정: stderr→stdout(=terminal), 그다음 stdout→file. stderr는 terminal에 남음.
```

redirect는 *왼쪽에서 오른쪽으로* 평가되고, `2>&1` 시점의 stdout이 무엇이냐가 결정.

`&>file` 또는 `>file 2>&1` 둘 다 OK. `&>` 는 bash 확장 — POSIX sh에선 안 됨.

### Here-doc과 Here-string

```bash
cat <<EOF        # here-doc, 변수 확장 O
$USER
EOF

cat <<'EOF'      # 따옴표 → 변수 확장 X (스크립트·sed 패턴에 유용)
$USER
EOF

cat <<-EOF       # 탭 들여쓰기 제거 (탭만, 스페이스 X)
	$USER
EOF

cmd <<<"input"   # here-string, fd 0으로 문자열 주입
```

`<<-` 가 *탭만* 인식하는 것이 미묘한 버그 원인 — 에디터가 스페이스로 자동 변환하면 동작 안 함.

### `/dev/null` 과 `/dev/stdin`

```bash
cmd > /dev/null              # stdout 버림
cmd > /dev/null 2>&1         # 둘 다 버림
cmd < /dev/null              # stdin을 EOF로 (interactive 방지)
```

ansible의 `shell:` 모듈이 *interactive 입력 대기로 hang* 되는 흔한 패턴 → `< /dev/null` 추가.

---

## 5. Exit Code 규약

| Code | 의미 |
|---|---|
| `0` | 성공 |
| `1` | 일반 실패 |
| `2` | 사용법 오류 (잘못된 인자) |
| `126` | 실행 권한 없음 |
| `127` | command not found |
| `128+N` | 시그널 N에 의해 종료 (예: `128+15=143` = SIGTERM) |
| `130` | SIGINT (Ctrl+C, 128+2) |

운영 스크립트는 *명시적으로* exit code를 정해야 한다. ansible의 `failed_when`·k8s의 `livenessProbe`·CI의 `if: failure()` 가 모두 exit code에 의존.

### `$?` 의 함정

```bash
cmd1
cmd2
echo $?    # cmd2의 exit code, cmd1 아님
```

`$?` 는 *직전 명령* 의 exit code. 보존하려면:

```bash
cmd1
rc=$?
cmd2
if [[ $rc -ne 0 ]]; then ...
```

---

## 6. Idempotency — Shell에서 *수동으로* 만들기

[[declarative-and-reconciliation]] §4에서 본 멱등성은 ansible·terraform·k8s가 *모듈/provider/controller 레벨* 에서 보장한다. shell은 *기본적으로 안 한다*. 운영 스크립트라면 작성자가 패턴으로 만든다.

### 표준 패턴

```bash
# 1. 디렉토리: -p로 멱등
mkdir -p /var/log/app

# 2. 파일에 라인 추가: 이미 있으면 skip
grep -qxF 'export FOO=bar' ~/.bashrc || echo 'export FOO=bar' >> ~/.bashrc

# 3. 심볼릭 링크: -f로 덮어쓰기 또는 -n으로 dir 아닌 경우만
ln -sfn /opt/app/current /opt/app/active

# 4. 패키지: apt/yum 자체가 멱등이지만 *version pin* 필수
apt install -y nginx=1.24.0-1ubuntu1

# 5. 서비스: systemctl은 이미 *원하는 상태가* 면 no-op
systemctl enable --now nginx
```

### Lock으로 *동시 실행* 멱등성

같은 스크립트를 두 번 동시에 돌릴 때:

```bash
exec 200>/var/lock/myscript.lock
flock -n 200 || { echo "already running"; exit 1; }
# 이후 코드는 동시 실행 안 됨
```

cron job·k8s CronJob이 *이전 실행이 안 끝났는데 새로 시작* 하는 race를 막는다.

### 작업의 *상태 확인 → 분기*

```bash
if ! kubectl get ns my-app &>/dev/null; then
    kubectl create ns my-app
fi
```

ansible 모듈이 내부적으로 하는 일을 *손으로* 하는 것. 그래서 ansible 공식 문서가 "거의 항상 *다른 모듈을 먼저 찾아라*"라고 한다.

---

## 7. Cross-reference 지도

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| Container PID 1, entrypoint, signal forwarding | `docker-basics/` 이미지·실행 토픽 |
| systemd unit이 시그널을 어떻게 전달하나 | `linux-basics/01-systemd.md` |
| ansible `shell:` vs 다른 모듈 선택 기준 | `ansible-basics/` idempotency |
| Container `livenessProbe` exec command | `kubernetes-basics/` workload |
| CI workflow step의 exit code | `cicd-gitops-basics/` GitHub Actions |
| terraform provisioner의 shell 호출 | `terraform-basics/` workflow |
| 멱등성의 *원리* 가 왜 필요한지 | [[declarative-and-reconciliation]] §4 |

---

## 한 줄 요약

> **shell은 Linux의 인터페이스다.** fork/exec, 시그널, fd가 보이지 않는 곳까지 영향을 미친다.
> *멱등성은 자동으로 오지 않는다* — 작성자가 만든다.

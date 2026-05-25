# 04 — Volumes와 Storage: 데이터는 어디 저장되고, 컨테이너가 죽으면 어떻게 되는가

> **공식 근거:** [Docker storage drivers](https://docs.docker.com/engine/storage/drivers/), [OverlayFS kernel docs](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html), [Docker volumes docs](https://docs.docker.com/storage/volumes/)
>
> **선행:** [01-image-and-layers.md](01-image-and-layers.md) (layer 개념), [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md) (MNT namespace)

---

## 🎯 한 문장

> **컨테이너의 파일시스템 = *image의 read-only layer들* 위에 *writable layer 한 장*을 OverlayFS로 합친 것. 컨테이너가 죽으면 *writable layer만 사라짐*. 데이터를 *살리려면* image 안이 아니라 *volume*(외부 스토리지)에 둬야 함.**

---

## 1. OverlayFS — *여러 디렉터리를 합쳐 하나처럼 보이는 마법*

OverlayFS는 Linux 커널의 *union filesystem*. 여러 디렉터리(layer)를 *하나로 합쳐* 마운트.

### 구조
```
   ┌────────────────┐  ← 위에서 보면 합쳐진 모습 (merged)
   │ /etc/nginx.conf│    여기 가서 읽고 쓰면 됨
   │ /usr/bin/node  │
   │ /app/index.js  │
   │ ... 모든 파일  │
   └────────────────┘
        ▲    (마운트해서 이렇게 보여줌)
        │
   ┌────────────────┐  upperdir (writable)  ← 컨테이너의 writable layer
   │ (변경된 파일만)│  
   └────────────────┘
   ┌────────────────┐  lowerdir3 (read-only) ← image의 최상위 layer
   │ COPY ./app ... │
   └────────────────┘
   ┌────────────────┐  lowerdir2 (read-only) ← image의 중간 layer
   │ apt install ...│
   └────────────────┘
   ┌────────────────┐  lowerdir1 (read-only) ← image의 베이스 layer
   │ ubuntu rootfs  │
   └────────────────┘
   ┌────────────────┐  workdir
   │ (내부 작업용)  │
   └────────────────┘
```

### 동작 규칙
- **읽기**: *위에서 아래로* 찾음. upperdir에 있으면 그거, 없으면 lowerdir들 차례로.
- **쓰기**: *항상 upperdir에만*.
- **수정 (Copy-on-Write)**:
  - lowerdir의 `/etc/nginx.conf` 를 수정하려고 함
  - → 그 파일을 *먼저 upperdir로 복사* → upperdir의 copy를 수정
  - lowerdir의 원본은 *그대로* (read-only)
- **삭제**:
  - lowerdir의 파일 삭제 시 → upperdir에 *whiteout* 파일 (`.wh.filename`) 생성
  - merged 뷰에서 *보이지 않음*

### 실제 위치 (Linux)
```bash
ls /var/lib/docker/overlay2/
# 각 컨테이너·image layer마다 디렉터리

cat /proc/mounts | grep overlay
# overlay on /var/lib/docker/overlay2/.../merged type overlay 
#   (lowerdir=...:...:..., upperdir=..., workdir=...)
```

→ docker 내부 폴더 *직접 들어가 보면* OverlayFS의 layer 구조가 *그대로 보임*.

---

## 2. Copy-on-Write의 *함의*

### 함의 1. *컨테이너 안 쓰기 = upperdir로 복사 발생*
- 작은 파일은 무시할 만하나, *큰 파일 수정* (예: DB 데이터, 로그)은 *복사 비용 + I/O 성능 손해*
- → DB·로그처럼 *지속적으로 쓰기 발생하는 데이터*는 OverlayFS 위에 두면 *느림*
- → Volume으로 빼서 *직접 디스크에 쓰기*

### 함의 2. *컨테이너가 죽으면 upperdir 삭제 → 데이터 손실*
- `docker rm {container}` → writable layer 함께 삭제
- 안에 만들었던 파일·DB 데이터 *전부 사라짐*
- *Image는 안 죽음* (그대로 보존)

### 함의 3. *같은 image로 컨테이너 여러 개 띄우면 → 각자 독립 upperdir*
- A 컨테이너의 변경은 B 컨테이너에 안 보임
- → 컨테이너끼리 *데이터 공유 X* (by default)

→ 이 3가지가 *왜 volume이 필요한가*의 답.

---

## 3. 3가지 데이터 보존 방식 — Bind Mount / Volume / tmpfs

| 방식 | 호스트 위치 | 라이프사이클 | 언제 |
|---|---|---|---|
| **Bind mount** | 호스트의 *임의 경로* (`/home/user/data`) | docker가 관리 X (호스트 파일시스템) | 개발 시 코드 즉시 반영 (`./src:/app`), 호스트 설정 파일 마운트 |
| **Volume** | docker가 관리 (`/var/lib/docker/volumes/{name}/`) | docker volume 명령으로 관리 | *프로덕션*에서 데이터 영속화 (DB 등) |
| **tmpfs** | RAM (디스크에 안 씀) | 컨테이너 죽으면 사라짐 | 민감 데이터 임시 처리, *디스크 I/O 회피* |

### Bind mount 예시
```bash
docker run -v /home/user/data:/app/data myapp
# 호스트의 /home/user/data 가 컨테이너 안 /app/data 로 보임
# 양쪽에서 수정 시 즉시 반영
```

### Volume 예시
```bash
docker volume create mydata
docker run -v mydata:/var/lib/mysql mysql
# docker가 관리하는 storage에 데이터 저장
# 컨테이너 죽어도 mydata 그대로

docker volume ls
docker volume inspect mydata
# Mountpoint: /var/lib/docker/volumes/mydata/_data
```

### tmpfs 예시
```bash
docker run --tmpfs /tmp myapp
# /tmp 는 RAM 위 (디스크 X). 컨테이너 죽으면 사라짐.
# 민감 데이터 (토큰 등) 임시 처리에 적합
```

### Bind mount vs Volume — 언제 무엇?

| 상황 | 권장 |
|---|---|
| 로컬 개발 (코드 즉시 반영) | Bind mount |
| 호스트 설정 파일 (nginx.conf) | Bind mount (read-only) |
| 프로덕션 DB 데이터 | Volume |
| 컨테이너 간 공유 데이터 | Volume |
| 호스트 OS 비의존이 중요 | Volume (Windows·Mac에서도 동일 동작) |
| 백업·복구 자동화 | Volume (docker 명령으로 쉬움) |
| 마운트 위치 호스트 GUI에서 보고 싶음 | Bind mount |

---

## 4. Storage Driver — *OverlayFS만 있는 게 아니다*

docker는 storage driver를 *플러그인 방식*으로. 현재 사실상 표준은 **overlay2**.

| Driver | 특징 | 사용 |
|---|---|---|
| **overlay2** | 위 OverlayFS 기반. *현재 표준*. | 거의 모든 환경 |
| **aufs** | overlay 이전의 union FS. 커널 mainline 아님. | 매우 구버전 (deprecated) |
| **btrfs** | Btrfs 파일시스템의 snapshot 활용 | Btrfs 사용 환경 |
| **zfs** | ZFS snapshot 활용 | ZFS 사용 환경 |
| **devicemapper** | block-level (각 컨테이너가 가상 block device) | RHEL 구버전 (deprecated) |
| **vfs** | 모든 layer를 *복사*. 매우 비효율. | 테스트·CI 환경에서 storage driver 선택지 없을 때 |

확인:
```bash
docker info | grep -i 'storage driver'
# Storage Driver: overlay2
```

→ **99% 환경은 overlay2**. 다른 거 쓰는 경우는 *왜 그런지* 의심 (보통 호스트 FS 제약).

---

## 5. Volume의 *드라이버 플러그인* — 클라우드 통합

볼륨은 기본적으로 *로컬 디스크*에 저장되지만, *드라이버 플러그인*으로 *외부 스토리지*에 연결 가능:

| Driver | 백엔드 |
|---|---|
| `local` (기본) | 호스트 디스크 |
| `nfs` (custom) | NFS 서버 |
| AWS EBS / EFS plugin | AWS 블록·파일 스토리지 |
| GCE PD plugin | GCP persistent disk |
| Ceph RBD plugin | Ceph 분산 스토리지 |

```bash
docker volume create --driver local --opt type=nfs \
  --opt o=addr=nfs.example.com,rw --opt device=:/path/to/dir mynfsvol
```

→ K8s에서는 *PersistentVolume (PV) + StorageClass + CSI* 가 *같은 추상화*. CSI(Container Storage Interface)는 K8s판 *볼륨 드라이버 플러그인 표준*.

→ docker volume driver와 K8s CSI는 *같은 아이디어, 다른 추상화*. `kubernetes-basics/` (예정) 에서 더.

---

## 6. *데이터 영속성 패턴* — 운영 관점

### 패턴 1. DB는 항상 *외부 storage*에
- `mysql` 컨테이너 → `-v mysql_data:/var/lib/mysql`
- 컨테이너 재시작·재배포·이미지 업그레이드 시에도 *데이터 유지*
- 운영에선 *RDS / Aurora 같은 매니지드* 가 더 흔함 — *컨테이너 안에 DB 안 두는 게 일반적*

### 패턴 2. *로그는 stdout/stderr 으로*
- 12-Factor App: "logs as event streams"
- 컨테이너 안 *파일에 쓰지 말고* stdout으로
- docker daemon이 *log driver*로 수집 (json-file / syslog / fluentd / awslogs)
- K8s에선 *Pod stdout → 노드 로그 수집기 → 중앙 집계* (Loki/Fluent Bit 등)

### 패턴 3. *설정 파일은 환경 변수 또는 ConfigMap*
- 컨테이너 *이미지 안에 환경별 설정* 박지 말기
- 환경 변수 / 외부 마운트 / K8s ConfigMap·Secret

### 패턴 4. *읽기 전용 컨테이너* (보안)
```bash
docker run --read-only --tmpfs /tmp myapp
# rootfs 전체 read-only. /tmp만 tmpfs로 쓰기 허용.
```
- 침해 시 *변조 어려움*
- K8s `securityContext.readOnlyRootFilesystem: true`

---

## 7. 흔한 함정

### 7-1. Bind mount로 *호스트 디렉터리 통째로* 마운트하면
- 컨테이너 안의 동일 경로 *내용이 사라진 것처럼 보임* (호스트 디렉터리가 *덮어쓰기*)
- 예: 컨테이너 image의 `/app`에 코드 있는데 `-v ./empty:/app` 마운트 → 빈 디렉터리로 보임

### 7-2. Volume에 *기존 데이터 있는 디렉터리* 마운트하면
- 첫 마운트 시: *컨테이너 안 그 경로의 데이터*를 volume에 *복사* (initialize)
- 두 번째부터: volume의 데이터가 *우선*
- → 같은 volume을 다른 image와 함께 쓸 때 *예상치 못한 데이터*가 보일 수 있음

### 7-3. *호스트 → 컨테이너* 권한 이슈
- Bind mount의 호스트 파일 소유자 (uid)와 컨테이너 안 프로세스 (uid) 가 다르면 *권한 거부*
- 해결: 컨테이너 USER를 호스트 파일 uid와 맞추기, 또는 마운트 옵션, 또는 chmod

### 7-4. *Mac/Windows에서 bind mount 느림*
- Linux는 native. Mac/Win은 *VM 안에서 docker 실행 + 파일 공유* → 느림
- 해결: cached/delegated 옵션, 또는 *VM 안 디스크에 코드 두기*, 또는 *원격 dev container*

### 7-5. *컨테이너 안에서 디스크 사용량 폭증*
- 로그 파일을 stdout이 아니라 *컨테이너 파일시스템*에 쓰면 → upperdir 폭증
- 호스트 디스크 다 차면 *모든 컨테이너 영향*
- log driver 또는 stdout 강제, log rotation

---

## 🎤 면접 빈출 Q&A

### Q1. 컨테이너 데이터는 어디 저장되나요? 컨테이너가 죽으면?
> Image의 read-only layer들 위에 *writable layer 한 장*이 OverlayFS로 합쳐져 컨테이너의 `/`가 된다. 컨테이너 안에서 *모든 쓰기는 writable layer에만*. 컨테이너 삭제 시 *writable layer가 함께 삭제* → 안의 데이터 모두 사라짐. 데이터 영속화하려면 *volume*이나 *bind mount*로 외부 스토리지에.

### Q2. OverlayFS의 동작 원리는?
> 여러 디렉터리(layer)를 *위에서 아래로 쌓아* 하나로 보이게 하는 union filesystem. *읽기*는 위에서부터 찾음. *쓰기*는 항상 *최상위 upperdir*에만. lowerdir의 파일을 *수정*하려고 하면 *먼저 upperdir로 복사* 후 수정 (Copy-on-Write). 삭제는 *whiteout* 파일로 표현.

### Q3. Bind mount와 Volume의 차이, 언제 무엇?
> *Bind mount*는 호스트의 *임의 경로*를 마운트. docker가 관리 X. 개발 시 코드 즉시 반영·호스트 설정 파일에 적합. *Volume*은 docker가 관리하는 storage. 라이프사이클·백업·드라이버 플러그인이 docker로 통합. *프로덕션 DB 데이터·이식성*에 적합. 보통 운영은 *volume*.

### Q4. 컨테이너 안에 DB 두면 안 되나요?
> *데이터를 컨테이너 fs에 두면* writable layer라 컨테이너 죽을 때 사라짐. 그래서 *반드시 volume*에 mount. 그리고 *프로덕션*에서는 *컨테이너 안에 DB 자체를 두지 않는 게 일반적* — 매니지드 DB(RDS/Aurora) 또는 *DB 전용 호스트*에. 이유: 백업·HA·관리 부담·OverlayFS 성능 비용.

### Q5. K8s의 PV/PVC와 docker volume의 관계는?
> 추상화 수준 차이. docker volume = *단일 호스트* 안에서 volume. K8s PV = *클러스터 전체* volume 추상화, *Pod와 lifecycle 분리*. CSI(Container Storage Interface)는 *K8s 표준 드라이버 인터페이스* — docker volume driver와 같은 아이디어를 K8s에서 표준화한 것.

### Q6. *읽기 전용 컨테이너*는 어떻게? 왜?
> `docker run --read-only` 또는 K8s `securityContext.readOnlyRootFilesystem: true`. 의도: *침해 시 변조 어렵게* 만들어 supply chain 보안 강화. 단, 앱이 *어딘가에 쓰긴 해야 함* → `/tmp`만 tmpfs로 열거나, *명시적 volume*만 쓰기 허용.

---

## 🔗 Cross-reference

- **Layer 개념** → [01-image-and-layers.md](01-image-and-layers.md)
- **MNT namespace + pivot_root** → [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md)
- **K8s PV/PVC/StorageClass/CSI** → `kubernetes-basics/` (예정)
- **읽기 전용·보안 컨텍스트** → [06-security.md](06-security.md), `k8s-security-basics/` (예정)
- **12-Factor App** "logs as event streams" → 외부 자료, `cicd-gitops-basics/` 일부 (예정)

---

## 📝 3줄 요약

1. 컨테이너 fs = *image의 read-only layer들 + writable layer 한 장* (OverlayFS Copy-on-Write).
2. 컨테이너 죽으면 *writable layer만 사라짐*. 영속화하려면 *volume* (또는 bind mount, tmpfs).
3. 운영에선 *DB는 컨테이너 안에 안 두고 volume/매니지드 DB*, *로그는 stdout으로* — 12-Factor의 자연스러운 결과.

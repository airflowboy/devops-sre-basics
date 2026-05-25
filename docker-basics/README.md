# Docker / 컨테이너 — 동작 원리 중심

> *"Docker는 VM과 뭐가 달라요?"* 에 대한 *진짜 답*은 한 문장 — **"Docker는 가상화가 아니라 *Linux 커널의 격리 기능을 잘 포장한 도구*다."**

이 폴더는 *Docker를 쓰는 법*이 아니라 *Docker가 어떻게 작동하는지*를 정리한 reference.

---

## 🎯 한 문장 큰 그림

> **컨테이너 = 호스트 커널을 공유하지만, 자신만의 *네임스페이스(격리된 시야)* 와 *cgroup(자원 한도)* 을 가진 *그냥 Linux 프로세스*.**

VM처럼 보이지만 VM이 아니다. `ps aux | grep nginx` 를 호스트에서 치면 컨테이너 안 nginx가 그대로 보인다 (격리는 *컨테이너 안에서 봤을 때만* 작동). 이걸 이해하면 docker의 90%가 풀린다.

---

## 🧩 Docker의 4가지 구성 요소 (한 페이지 요약)

| 요소 | 한 줄 설명 | 깊이 보기 |
|---|---|---|
| **Image** | 실행 환경 + 코드 + 메타데이터 의 *불변 스냅샷*. content-addressable (sha256). | [`01-image-and-layers.md`](01-image-and-layers.md) |
| **격리 (namespace + cgroup)** | 컨테이너가 *VM처럼 보이게* 만드는 핵심. Linux 커널 기능. | [`02-namespace-and-cgroup.md`](02-namespace-and-cgroup.md) |
| **네트워크 / 스토리지** | 컨테이너가 *외부와 통신하고 데이터를 보존*하는 방식. | [`03-networking.md`](03-networking.md), [`04-volumes-and-storage.md`](04-volumes-and-storage.md) |
| **빌드 / 보안** | Dockerfile → 이미지 흐름. 그리고 *컨테이너 ≠ 보안 경계* 라는 사실. | [`05-dockerfile-and-build.md`](05-dockerfile-and-build.md), [`06-security.md`](06-security.md) |

---

## 🌪 자주 헷갈리는 7가지

### 1. Docker는 VM이 *아니다*
- VM = 하이퍼바이저가 *가상 하드웨어* 만들고 그 위에 *전체 OS 부팅*
- 컨테이너 = 호스트 커널 *그대로 공유*, 격리된 *프로세스 트리·파일시스템 뷰·네트워크 인터페이스*만 새로 부여
- → 컨테이너 부팅 = 프로세스 시작 (밀리초). VM 부팅 = OS 부팅 (수십 초).
- → 컨테이너 안에서 `uname -r` 하면 **호스트 커널 버전**이 나옴.

### 2. Image와 Container의 관계
- Image = *클래스*, Container = *인스턴스*
- 한 이미지로 컨테이너 수십 개 띄울 수 있음 (각자 독립된 writable layer)
- *컨테이너가 죽어도 이미지는 그대로*. 같은 이미지로 새 컨테이너 → 동일 시작점.

### 3. Layer는 *읽기 전용*, 컨테이너만 *쓰기 가능*
- Image = 여러 *read-only* layer의 스택 (OverlayFS로 합쳐 보임)
- 컨테이너 = 그 위에 *writable layer 한 장* 추가
- 컨테이너 안에서 파일 수정 → writable layer에만 기록 (Copy-on-Write)
- → 컨테이너 죽으면 writable layer만 사라짐. **그래서 영속화하려면 volume.** ([04 참조](04-volumes-and-storage.md))

### 4. `docker run`은 사실 *명령어 모음의 단축어*
실제로는 내부에서 이런 일이 일어남:
```
1. 이미지 없으면 → docker pull (manifest + layers fetch)
2. 새 namespace 7개 생성 (PID, NET, MNT, UTS, IPC, USER, CGROUP)
3. 새 cgroup 생성, 자원 한도 적용
4. writable layer를 image 위에 마운트 (OverlayFS)
5. 네트워크 namespace에 veth 만들어 docker0 bridge에 연결
6. 그 namespace 안에서 `cmd` 프로세스 fork+exec
7. 표준 입출력을 daemon에 연결
```
→ 이걸 *수동으로* 한 번 해보면 ([02 참조](02-namespace-and-cgroup.md)) docker가 마법이 아니게 됨.

### 5. `localhost`가 *컨테이너 안에선 다른 의미*
- 호스트의 `localhost` ≠ 컨테이너의 `localhost`
- 각 컨테이너는 *자신의 네트워크 namespace*, 그 안의 `lo` 인터페이스가 있음
- 한 컨테이너의 nginx에 다른 컨테이너에서 `curl localhost` → 자기 자신만 봄
- 통신하려면 *bridge 네트워크의 컨테이너 이름* 으로 (DNS 해결됨, `curl nginx`)
- ([03 참조](03-networking.md))

### 6. `-p 80:8080` 의 *진짜 의미*
- *호스트의 80번 포트로 들어온 트래픽을 컨테이너의 8080번으로 DNAT*
- 내부적으로 iptables `PREROUTING` 체인에 룰 추가
- `iptables -t nat -L DOCKER` 로 직접 확인 가능
- → 즉 *호스트 방화벽이 막으면 publish해도 못 들어옴*
- ([03 참조](03-networking.md))

### 7. Docker ≠ 컨테이너. 컨테이너 ≠ 보안 경계
- **컨테이너 ≠ Docker.** containerd / CRI-O / Podman 등 다양한 *런타임*이 같은 OCI 표준 따름. Docker는 그중 하나.
- **컨테이너 ≠ VM 수준 보안 경계.** 커널 공유 → 커널 exploit 하나로 호스트 탈출 가능. *진짜 격리*가 필요하면 gVisor / Kata Containers (둘 다 *마이크로 VM* 방식)
- ([06 참조](06-security.md))

---

## 📂 토픽 파일 인덱스

| 파일 | 핵심 질문 | 언제 보기 |
|---|---|---|
| [01-image-and-layers.md](01-image-and-layers.md) | "이미지는 어떻게 저장되고 전송되는가" | 면접: *image vs container 차이*, *layer 캐시 동작* |
| [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md) | "컨테이너는 어떻게 격리되는가" — *가장 중요* | 면접: *Docker 작동 원리*, *VM과 차이* |
| [03-networking.md](03-networking.md) | "컨테이너 간/외부와 어떻게 통신하는가" | 면접: *bridge vs host*, *publish 동작* |
| [04-volumes-and-storage.md](04-volumes-and-storage.md) | "데이터는 어디 저장되고 컨테이너 죽으면 어떻게 되는가" | 면접: *데이터 영속성*, *OverlayFS* |
| [05-dockerfile-and-build.md](05-dockerfile-and-build.md) | "Dockerfile → 이미지 변환은 어떻게" | 면접: *이미지 크기 최적화*, *multi-stage*, *BuildKit* |
| [06-security.md](06-security.md) | "컨테이너 보안의 layer는 어디부터 어디까지인가" | 면접: *container escape*, *root 컨테이너*, *image 신뢰* |

---

## 🔗 Cross-reference

- **02 namespace + cgroup** 의 *Linux 커널 기능들* — 이게 [`kubernetes-basics`]의 Pod 격리에서도 동일하게 등장. Pod = *namespace를 공유하는 컨테이너들의 묶음*.
- **03 networking** 의 veth/bridge/iptables — [`network-basics/06-kubernetes.md`](../network-basics/06-kubernetes.md) 의 CNI 플러그인이 *같은 원리를 K8s에 적용*한 것.
- **06 security** 의 capabilities·seccomp — [`k8s-security-basics`]의 PSS(Pod Security Standards) 가 *이걸 K8s 차원에서 강제하는 것*.
- **OCI image spec** ([01 참조](01-image-and-layers.md)) — Helm chart도, ArgoCD가 배포하는 것도, 결국 *OCI 이미지*를 K8s에 띄우는 것. 모든 것의 베이스.

---

## 🎤 면접 빈출 질문 (이 폴더로 답변 가능)

| 질문 | 답이 있는 파일 |
|---|---|
| Docker는 VM과 어떻게 다른가요? | [02](02-namespace-and-cgroup.md) |
| 이미지 layer caching은 어떻게 작동하나요? | [01](01-image-and-layers.md), [05](05-dockerfile-and-build.md) |
| `docker run` 했을 때 내부에서 일어나는 일을 단계별로 설명해주세요. | [02](02-namespace-and-cgroup.md) + [03](03-networking.md) |
| 컨테이너 안에서 호스트의 파일을 보려면? 반대로 호스트에서 컨테이너 파일은? | [04](04-volumes-and-storage.md) |
| `-p 8080:80` 옵션 없으면 외부에서 접근 안 되는 이유는? | [03](03-networking.md) |
| 이미지 크기를 줄이는 방법 3가지 이상 설명. | [05](05-dockerfile-and-build.md) |
| 컨테이너에서 root로 실행하면 안 되는 이유는? | [06](06-security.md) |
| Container escape이 가능한 시나리오를 설명해주세요. | [06](06-security.md) |
| Docker가 없는 환경에서 containerd만 써본 적 있나요? 차이는? | [01](01-image-and-layers.md) 끝 부분 |

---

## 📐 학습 순서 권장

1. **02 namespace + cgroup** — 가장 중요. 모든 게 여기서 파생됨.
2. **01 image + layers** — 데이터 측면. 02와 함께 보면 시너지.
3. **03 networking** — 02의 NET namespace 응용편.
4. **04 volumes** — 02의 MNT namespace 응용편.
5. **05 Dockerfile** — 위 4개를 *실용적으로 묶는* 방법.
6. **06 security** — 위 5개의 *위협 모델 + 완화* 통합.

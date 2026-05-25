# 03 — Networking: 컨테이너는 어떻게 통신하는가

> **공식 근거:** [Docker Networking docs](https://docs.docker.com/network/), Linux `man 8 ip`, `man 8 iptables`, [CNI spec](https://github.com/containernetworking/cni/blob/main/SPEC.md) (cross-ref용)
>
> **선행 학습:** [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md) 의 *NET namespace* 섹션, [network-basics/](../network-basics/) 전반.

---

## 🎯 한 문장

> **컨테이너 네트워크 = 각 컨테이너의 *별도 NET namespace* + *veth pair*(가상 케이블) + *bridge*(가상 스위치) + *iptables*(가상 라우터/방화벽).**

→ 마법이 아니라 *Linux가 원래 가진 네트워크 기능*들의 조합. `ip` / `iptables` / `bridge` 명령으로 *직접 보고 만질 수 있음*.

---

## 1. 컨테이너 네트워킹의 4가지 부품

```
                    ┌──────────────────────────────────┐
                    │           Host                    │
                    │                                   │
                    │   eth0 (192.168.1.10)             │
                    │      │                            │
                    │   ┌──┴───────────────────────┐    │
                    │   │  iptables (NAT, MASQ)    │    │
                    │   └──┬───────────────────────┘    │
                    │      │                            │
                    │   ┌──┴──────┐                     │
                    │   │ docker0 │  bridge             │
                    │   │172.17.0.1│                    │
                    │   └─┬─────┬─┘                     │
                    │     │     │  veth pair (가상 케이블)│
                    │  ┌──┴┐ ┌──┴┐                      │
                    │  │vethA│ │vethB│ 호스트 쪽            │
                    │  └─┬─┘ └─┬─┘                       │
                    └────┼─────┼─────────────────────────┘
                         │     │  (NET namespace 경계)
                       ┌─┴─┐ ┌─┴─┐
                       │eth0│ │eth0│ 컨테이너 쪽 (각자 자기 namespace 안)
                       │172.17.0.2│ │172.17.0.3│
                       └────┘ └────┘
                       Container A  Container B
```

| 부품 | 역할 | 비유 |
|---|---|---|
| **NET namespace** | 컨테이너의 *별도 네트워크 스택* (인터페이스·라우팅·iptables 다 자기 것) | 자기만의 *방* |
| **veth pair** | 양 끝이 *서로 연결된* 가상 인터페이스 한 쌍 | 두 방 사이의 *전화선* |
| **bridge (`docker0`)** | 여러 veth를 *L2에서 연결*하는 가상 스위치 | 단지 안의 *공용 복도* |
| **iptables** | NAT, 필터링, 포트 publish의 *룰 엔진* | 단지 정문의 *경비실* |

---

## 2. `docker network ls` — 기본 3가지 네트워크

```
NAME      DRIVER    SCOPE
bridge    bridge    local    ← 컨테이너 띄울 때 기본
host      host      local    ← 호스트 네트워크 공유
none      null      local    ← 네트워크 없음
```

| 드라이버 | 동작 | 언제 쓰나 |
|---|---|---|
| **bridge** | 위 그림 그대로 — docker0 + veth + iptables | *기본*, 외부와 통신 + 컨테이너 간 격리 |
| **host** | 호스트의 NET namespace 그대로 *공유* (별도 namespace 안 만듦) | 성능 극단 (NAT 비용 0), 보안 격리 X. 보통 *피함*. |
| **none** | NET namespace 만들지만 인터페이스 없음 (`lo`만) | 네트워크 자체 차단 (batch job, 보안) |
| **container** | 다른 컨테이너의 NET namespace 공유 (`--network=container:other`) | sidecar 패턴, K8s Pod의 베이스 |
| **overlay** | *여러 호스트 간* 컨테이너 네트워크 (VXLAN 활용) | Swarm 모드, *K8s에선 CNI가 대신 함* |
| **macvlan** | 컨테이너에 *물리 NIC의 MAC 주소* 직접 부여 | 레거시 시스템 통합 (드물게) |

---

## 3. Bridge 네트워크 — *기본*의 내부 동작

`docker run nginx` 만 했을 때 일어나는 일:

### Step 1. veth pair 생성
```bash
# 호스트에서 수동으로 만들어볼 때
ip link add vethA type veth peer name vethB
# vethA와 vethB는 가상 케이블의 양 끝
```

### Step 2. 한쪽을 컨테이너 namespace로 이동
```bash
ip link set vethB netns {컨테이너_PID}
# vethB는 이제 컨테이너 안에서만 보임
```

### Step 3. 호스트 쪽 (vethA)을 docker0 bridge에 연결
```bash
ip link set vethA master docker0
ip link set vethA up
```

### Step 4. 컨테이너 안에서 IP 할당
```bash
nsenter -t {PID} -n ip addr add 172.17.0.2/16 dev vethB
nsenter -t {PID} -n ip link set vethB up
nsenter -t {PID} -n ip route add default via 172.17.0.1
```

### Step 5. 결과
- 컨테이너 안에서 `ping 172.17.0.1` (docker0) → 응답
- 컨테이너 안에서 `ping 8.8.8.8` (외부) → docker0 → iptables MASQ → 호스트 eth0 → 외부

`docker network inspect bridge` 로 위 구조 *수치로* 확인 가능.

---

## 4. *어떻게* 외부 인터넷에 나가나 — iptables MASQUERADE

컨테이너는 *사설 IP* (172.17.0.2). 외부로 그대로 나가면 *응답 받을 수 없음* (사설 IP는 인터넷에서 의미 없음).

→ docker는 *호스트의 iptables*에 **MASQUERADE** 룰을 박는다:

```bash
sudo iptables -t nat -L POSTROUTING -n -v
# 출력 예시:
# Chain POSTROUTING (policy ACCEPT)
# target     prot opt source         destination
# MASQUERADE all  --  172.17.0.0/16  !172.17.0.0/16
```

**의미:** *docker0 네트워크(172.17.0.0/16)에서 나가는데 docker0 밖으로 가는 패킷*은 *출발지 IP를 호스트 IP로 바꿔라* (=NAT).

→ 외부에선 *호스트가 요청한 것처럼 보임*. 응답이 호스트로 옴 → 호스트가 *원래 어느 컨테이너였는지* 기억해두고 (conntrack) 그 컨테이너로 전달.

이건 [`network-basics/02-routing-and-nat.md`](../network-basics/02-routing-and-nat.md) 의 *NAT Gateway* 와 *같은 원리*. AWS NAT GW가 하는 일을 *docker가 호스트 안에서 iptables로* 하는 것.

---

## 5. `-p 8080:80` — 외부에서 *들어오는* 트래픽

기본적으로 컨테이너는 *나가는 건 자유, 들어오는 건 차단*. (NAT의 자연스러운 한계)

`-p 8080:80` 옵션 → docker가 iptables에 **DNAT** 룰 추가:

```bash
sudo iptables -t nat -L DOCKER -n -v
# Chain DOCKER (2 references)
# target     prot opt source     destination       
# DNAT       tcp  --  0.0.0.0/0  0.0.0.0/0   tcp dpt:8080 to:172.17.0.2:80
```

**의미:** *호스트의 8080번 포트로 들어온 TCP는 172.17.0.2의 80번으로 redirect.*

### 흔한 함정
- *호스트 방화벽 (firewalld/ufw)이 8080을 막고 있으면* publish해도 외부에서 못 들어옴
- `-p 80:80` (1024 이하 포트)은 *호스트 root 권한 필요*
- `-p 127.0.0.1:8080:80` 하면 *호스트 외부에서 안 보임* (디버깅·보안용)
- `-P` (대문자) 는 *모든 EXPOSE된 포트를 호스트 랜덤 포트로* 자동 publish

→ 면접 빈출: *"publish는 어떻게 동작하나"* — DNAT + iptables 답.

---

## 6. 컨테이너 간 통신 — *DNS와 user-defined network*

### 기본 bridge의 한계
- 기본 `bridge` 네트워크에선 *컨테이너 이름으로 통신 불가* (IP로만)
- → 컨테이너 IP는 *재시작 시 바뀜* → 어떻게 안정적으로 통신?

### 해결: User-defined bridge
```bash
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet --name app busybox sh
# app 컨테이너 안에서:
ping web    # ← 동작! (docker가 embedded DNS 제공)
```

- docker daemon이 *embedded DNS resolver* 띄움 (127.0.0.11)
- 컨테이너의 `/etc/resolv.conf` 가 자동으로 그쪽으로 설정됨
- *컨테이너 이름·alias·서비스명*으로 다른 컨테이너 lookup 가능

### K8s에서는 이게 어떻게?
K8s도 *같은 원리*. Pod 안 컨테이너의 `/etc/resolv.conf`가 *CoreDNS* 가리킴. *Service 이름*으로 lookup → 가상 IP(ClusterIP) → kube-proxy가 iptables로 실제 Pod로 전달.

→ docker network DNS와 K8s Service DNS는 *같은 아이디어, 다른 구현*. [`kubernetes-basics/`](../) (예정) 에서 더 깊이.

---

## 7. Overlay 네트워크 — *여러 호스트 간*

`docker network create -d overlay mynet` (Swarm 모드)

- 여러 호스트 위 컨테이너들이 *같은 L2 네트워크처럼* 통신
- 내부적으로 **VXLAN** — UDP 패킷에 *원본 L2 frame을 캡슐화*해서 다른 호스트로 전송
- 받는 호스트가 *풀어서* 로컬 컨테이너에 전달

→ **K8s에서는 docker overlay 안 씀**. 대신 *CNI 플러그인* (Calico, Flannel, Cilium 등)이 같은 역할. 일부 CNI는 *VXLAN과 동일한 원리* 사용 (Flannel default), 일부는 *BGP 라우팅* 사용 (Calico).

→ 1차 SRE로드맵에서 만진 *Calico VXLAN* 이 바로 이것. 자세히는 [`network-basics/06-kubernetes.md`](../network-basics/06-kubernetes.md) 와 `kubernetes-basics/` (예정).

---

## 8. `--net=host` 의 *진짜* 의미와 *위험*

```bash
docker run --net=host nginx
```

- 컨테이너가 *호스트의 NET namespace 그대로 사용*. 별도 네트워크 격리 X.
- nginx가 *호스트의 80번 포트를 직접* 점유
- 장점: NAT 비용 0, 성능 최고
- 단점:
  - 포트 충돌 (호스트의 80번 이미 사용 중이면 fail)
  - 컨테이너 안에서 *호스트의 모든 인터페이스* 보임 (보안)
  - 다른 호스트의 같은 컨테이너와 *연결 패턴 다름*

→ **K8s에선 `hostNetwork: true`** 와 같은 의미. *DaemonSet으로 노드 자체 모니터링 에이전트* 같은 경우만 사용.

---

## 9. 디버깅 — *왜 안 통하지?*

흔한 문제와 디버깅 순서:

### A. 컨테이너 → 컨테이너 통신 안 됨
1. 같은 네트워크인지? `docker network inspect mynet`
2. *기본 bridge*면 이름으로 통신 안 됨 → user-defined network로
3. 컨테이너 안에서 `nslookup other-container` → DNS 작동?
4. `ping {대상_컨테이너_IP}` 로 IP 직접 → 네트워크 자체 문제?

### B. 컨테이너 → 외부 안 됨
1. 컨테이너 안에서 `ip route show` → default 게이트웨이 있나? (보통 172.17.0.1)
2. `ping 172.17.0.1` (게이트웨이=docker0) → 성공?
3. `ping 8.8.8.8` (외부 IP) → 호스트 iptables MASQ 작동?
4. `nslookup google.com` → DNS 작동? (`/etc/resolv.conf` 확인)
5. 호스트에서 `iptables -t nat -L POSTROUTING -n -v` → MASQ 룰 존재?
6. 호스트 자체가 외부 통신 되는가?

### C. 외부 → 컨테이너 안 됨 (publish 했는데)
1. `docker port {container}` → publish 룰 등록됐나
2. 호스트에서 `curl localhost:8080` → 호스트→컨테이너 OK?
3. 호스트 외부에서 `curl {호스트IP}:8080` → 외부→호스트 OK?
4. 호스트 방화벽 (firewalld/ufw)이 8080 막고 있나?
5. `iptables -t nat -L DOCKER -n -v` → DNAT 룰 존재?
6. 클라우드면 보안 그룹(SG)·NACL이 막나?

→ **방향성을 좁히는 게 핵심**. 한 번에 끝까지 가지 말고 *단계별 확인*.

---

## 10. 자주 헷갈리는 것

### 10-1. `127.0.0.1` 은 *컨테이너 안에서 다른 의미*
- 컨테이너의 `127.0.0.1` = *그 컨테이너의 lo* (자기 자신만)
- 한 컨테이너의 *nginx*에 다른 컨테이너에서 `curl 127.0.0.1` → 자기 자신 보고 *실패*
- → 컨테이너 간은 *bridge 네트워크의 컨테이너 이름* 으로

### 10-2. 컨테이너 IP는 *재시작 시 바뀐다*
- bridge 네트워크는 동적 IP 할당
- → 운영에서 IP 직접 의존하면 *부서짐*
- → 항상 *이름·DNS·서비스 추상화* 통해 통신

### 10-3. `EXPOSE` 는 *publish 가 아니다*
- Dockerfile의 `EXPOSE 80` = *문서화*. 실제 포트는 안 열림.
- 실제로 외부에 노출하려면 *run 할 때* `-p` 또는 `-P`
- → `EXPOSE 80` + `-P` 조합이면 자동으로 모든 EXPOSE 포트가 호스트 랜덤 포트로 publish

### 10-4. *DNS resolution이 안 될 때* 의심 순서
- 컨테이너 `/etc/resolv.conf` 확인 — embedded DNS (127.0.0.11) 또는 호스트 DNS 가리키는지?
- *기본 bridge*인지 *user-defined*인지 — 기본은 embedded DNS 없음
- 호스트 자체의 DNS가 작동하나? (`docker run alpine nslookup google.com`)

### 10-5. `iptables -F` 로 *컨테이너 네트워크가 깨지는 이유*
- docker는 iptables에 *수십 개 룰*을 박는다 (DOCKER 체인, MASQ 룰 등)
- `iptables -F` (모든 룰 삭제) → docker가 박은 룰도 삭제
- → 컨테이너 외부 통신 망함
- 복구: `systemctl restart docker` (재실행 시 룰 재등록)

---

## 🎤 면접 빈출 Q&A

### Q1. 컨테이너 간 통신은 어떻게 이루어지나요?
> 각 컨테이너는 자기만의 *NET namespace*. veth pair로 호스트의 docker0 bridge에 연결. *같은 bridge*의 컨테이너끼리는 L2에서 직접 통신. user-defined network면 *embedded DNS*로 컨테이너 이름 lookup도 가능.

### Q2. `-p 8080:80` 의 내부 동작은?
> docker daemon이 *호스트의 iptables*에 *PREROUTING + DNAT 룰*을 추가. *호스트의 8080번 TCP*가 들어오면 *컨테이너의 IP:80*으로 redirect. `iptables -t nat -L DOCKER -n -v` 로 직접 확인 가능. 호스트 방화벽이 8080 막으면 publish 했어도 외부에서 못 들어옴.

### Q3. host network와 bridge의 차이, 언제 무엇을 쓰나요?
> bridge = 컨테이너가 *별도 NET namespace*, NAT/iptables 거쳐 통신. 격리·이식성 좋으나 NAT 비용 약간. host = 컨테이너가 *호스트 NET namespace 공유*, 격리 X, 성능 최고. → 일반적으로 bridge가 기본. host는 *극단 성능* 필요할 때 (모니터링 에이전트 등) 또는 *DaemonSet으로 노드 자체 작업*.

### Q4. 컨테이너가 외부 인터넷에 나갈 수 있는 *이유*는?
> 컨테이너 IP는 사설 (172.17.0.x). 그대로 나가면 응답 못 받음. docker가 호스트 iptables에 *MASQUERADE* 룰 → 나가는 패킷의 *출발지 IP를 호스트 IP로* 치환. 외부에선 호스트가 요청한 것처럼 보임. 응답이 호스트로 옴 → conntrack이 *원래 컨테이너 기억*해두고 전달. AWS NAT GW와 같은 원리.

### Q5. K8s의 Service와 docker network DNS는 어떻게 다른가요?
> 아이디어는 *같음* — *이름으로 lookup해서 안정적 endpoint*. 구현은 다름. docker는 embedded DNS (127.0.0.11)가 단일 호스트 안에서. K8s는 *CoreDNS*가 클러스터 전체에서 *Service 이름 → ClusterIP* lookup, *kube-proxy* (또는 IPVS·eBPF)가 ClusterIP → 실제 Pod로 routing. K8s가 *훨씬 더 큰 스케일*에서 같은 추상화 제공.

### Q6. CNI는 docker network와 어떤 관계인가요?
> docker network는 *단일 호스트* 네트워킹의 docker 자체 구현. CNI(Container Network Interface)는 *K8s의 네트워크 추상화 표준* — *플러그인 인터페이스만 정의*하고, 실제 구현은 플러그인(Calico/Flannel/Cilium)이 함. K8s는 docker network를 *안 씀*, *CNI에 위임*. CNI 플러그인 내부적으로는 *같은 Linux 기능들* (veth/bridge/iptables/BGP/VXLAN/eBPF) 활용.

---

## 🔗 Cross-reference

- **NET namespace 자체** → [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md)
- **IP·라우팅·NAT 기본** → [network-basics/01-basics.md](../network-basics/01-basics.md), [02-routing-and-nat.md](../network-basics/02-routing-and-nat.md)
- **방화벽 (iptables 응용)** → [network-basics/03-firewalls.md](../network-basics/03-firewalls.md)
- **AWS VPC + Security Group** (라우팅·iptables의 클라우드 버전) → [network-basics/05-aws-vpc.md](../network-basics/05-aws-vpc.md)
- **K8s 네트워킹 (Pod IP, Service, Ingress, NetworkPolicy, CNI)** → [network-basics/06-kubernetes.md](../network-basics/06-kubernetes.md), `kubernetes-basics/` (예정)
- **Service Mesh (Istio 등)** — 컨테이너 네트워킹의 *L7 추상화* → 예정

---

## 📝 3줄 요약

1. 컨테이너 네트워크 = *NET namespace + veth + bridge + iptables*. 마법 아니라 Linux가 원래 가진 기능들의 조합.
2. 외부 통신 = *MASQ NAT*, publish = *DNAT*. 둘 다 호스트 iptables에 룰을 추가하는 일.
3. K8s CNI는 docker network의 *같은 아이디어를 다른 추상화로* 구현. 베이스는 같은 Linux 네트워크 기능들.

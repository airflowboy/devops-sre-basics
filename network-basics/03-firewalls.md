# 03. 방화벽 — 누가 들어올 수 있나 (SG vs NACL vs 호스트 방화벽)

> **라우팅이 됐어도 방화벽이 막으면 안 통함.** "분명 SG 열었는데 왜 안 되지?"의 90%는 *다른 층 방화벽*을 못 본 것.

## 목차
- [방화벽의 3층](#방화벽의-3층)
- [Security Group (SG) — 호실 문지기](#security-group-sg--호실-문지기)
- [NACL — 단지 정문 보안](#nacl--단지-정문-보안)
- [호스트 방화벽 — 집 안 도어록](#호스트-방화벽--집-안-도어록)
- [K8s NetworkPolicy](#k8s-networkpolicy)
- [디버깅 순서](#디버깅-순서)
- [치트시트](#치트시트)

---

## 방화벽의 3층

패킷이 EC2 안 프로세스에 도달하려면 **여러 방화벽**을 통과해야 함:

```
   인터넷
     │
     ▼
┌─────────────────────────┐
│ ① NACL (서브넷 단위)     │  ← 단지 정문 보안 (대규모 거름)
└─────────────────────────┘
     │
     ▼
┌─────────────────────────┐
│ ② Security Group (EC2)  │  ← 호실 문지기 (인스턴스 단위)
└─────────────────────────┘
     │
     ▼
┌─────────────────────────┐
│ ③ 호스트 방화벽 (iptables/firewalld) │  ← 집 안 도어록 (OS 안)
└─────────────────────────┘
     │
     ▼
   프로세스 (예: nginx)
```

→ **하나라도 막으면 안 통함**. 디버깅 시 *세 층 다* 확인.

---

## Security Group (SG) — 호실 문지기

> **인스턴스(또는 ENI) 단위** 방화벽. AWS의 가장 자주 쓰는 방화벽.

### 핵심 특징
- **stateful** — 들어온 트래픽의 *응답*은 자동 허용 (별도 outbound 룰 불필요).
- **allow only** — 명시적 허용만. 차단 룰 없음 (있는 게 없으면 차단).
- 인스턴스에 여러 SG 부여 가능 (모든 SG의 룰이 *OR*로 적용).
- SG ID로 다른 SG 참조 가능 (예: ALB SG에서 오는 것만 허용).

### 룰 형식
```
Inbound:
  Protocol  Port    Source           Description
  ─────────────────────────────────────────────
  TCP       22      211.195.63.108/32   SSH from my IP
  TCP       80      0.0.0.0/0           HTTP from anywhere
  TCP       443     0.0.0.0/0           HTTPS from anywhere

Outbound:
  TCP       0-65535  0.0.0.0/0          (모든 outbound — 기본)
```

### Stateful의 의미
- inbound TCP 80 허용 → 그 요청의 응답 outbound는 *자동 허용*. 별도 outbound 룰 X.
- → 그래서 outbound는 보통 "전부 허용" 둠 (응답이 알아서 통하니까). 엄격하려면 outbound도 제한 가능.

### 흔한 패턴
```
# ALB SG
inbound:  TCP 443 from 0.0.0.0/0          # HTTPS 받음
outbound: TCP 8080 to app-sg               # 백엔드로

# App (EC2) SG
inbound:  TCP 8080 from alb-sg             # ALB에서만 받음 (IP 아니라 SG 참조!)
outbound: TCP 5432 to db-sg                # DB로

# DB (RDS) SG
inbound:  TCP 5432 from app-sg             # App에서만
outbound: (필요 X)
```

→ **SG 참조 패턴**: IP가 아니라 SG ID로 참조. 인스턴스 늘어나도 룰 안 바뀜.

### 함정
- "왜 80은 되는데 8080은 안 되지?" — SG에 8080 inbound 룰 없음.
- "Default SG에 다 들어가 있는데 왜 못 통함?" — Default SG의 *기본 inbound가 비어있을 수 있음*. 또는 *같은 SG 인스턴스끼리만* 허용 룰 (self-reference).
- "프라이빗 EC2에 SG 다 열었는데 외부 못 함" — SG는 inbound만 다룸. outbound 인터넷은 *라우팅(NAT GW)* 문제. [02-routing-and-nat.md](02-routing-and-nat.md) 참조.

### 우리 프로젝트
- Ch 08 Phase B의 `web` SG: SSH from 내 IP / HTTP from anywhere (data "http"로 자동 탐지)
- Ch 10 캡스톤에서: ALB SG / EKS node SG / RDS SG / Redis SG 매트릭스

---

## NACL — 단지 정문 보안

> **서브넷 단위** 방화벽. SG보다 위. 잘 안 건드림.

### SG와 다른 점
| | **SG** | **NACL** |
|---|---|---|
| 범위 | 인스턴스 | 서브넷 |
| stateful? | ✅ 자동 응답 허용 | ❌ inbound·outbound 양쪽 명시 |
| 룰 종류 | allow만 | allow + deny |
| 룰 우선순위 | 없음 (OR) | 번호순 (낮은 번호 먼저) |
| 기본값 | 새 SG = 다 막힘 | 기본 NACL = 다 열림 |

### 언제 NACL 쓰나
- "특정 IP를 전 서브넷에서 차단" (악성 IP 블록 등)
- SG가 *너무 많아져서* 한 군데서 거르고 싶을 때
- 컴플라이언스 요구로 *서브넷 단위 격리* 필요할 때

### 일반 학습·소규모 운영
- **거의 안 건드림.** SG로 충분.
- AWS Console에서 기본값(다 허용) 유지.

### Stateless의 함정
NACL은 *outbound 응답을 자동 허용 안 함*. 그래서:
- inbound TCP 80 허용했다면 → outbound도 *ephemeral 포트 범위*(32768-65535)를 허용해야 응답 나감.
- 이거 잊고 NACL 만지면 모든 게 깨짐. **SG로 충분하면 NACL 안 건드리는 게 답**.

---

## 호스트 방화벽 — 집 안 도어록

> EC2 안의 OS 자체 방화벽 (`iptables`, `firewalld`, `ufw`).

### 왜 또 있나
- 다층 방어 (defense in depth).
- OS 단위로 더 세밀한 룰 가능.
- 클라우드 외(온프레미스)에서도 작동.

### Ch 02·05에서 만진 firewalld
- Rocky Linux의 기본 방화벽. zone 기반.
- `--add-port=80/tcp --permanent` 식으로 룰 추가.
- 우리 홈랩에선 *cluster CIDR을 trusted zone에 추가*하지 않으면 K8s 통신 깨짐 (Ch 03·04-C 학습).

### iptables / nftables
- 더 저수준. 컨테이너·K8s가 직접 조작.
- Calico NetworkPolicy도 결국 iptables 룰로 떨어짐.
- `iptables -L -n -v`로 룰 확인.

### 컨테이너 / EKS에선 보통 안 만짐
- 호스트 방화벽은 K8s가 알아서 (kube-proxy + CNI가 iptables 채움).
- *사람이 직접 추가하면 K8s 룰과 충돌* 가능 — 보통 SG·NetworkPolicy로 처리.

### 함정
- "SG·NACL 다 열었는데 안 됨" → 호스트 방화벽 의심. `sudo iptables -L -n` 확인.
- AMI에 따라 기본 firewalld·iptables 다름. Amazon Linux 2023은 거의 비활성, Ubuntu는 ufw 비활성.

---

## K8s NetworkPolicy

> Pod간 트래픽 제어. **CNI 단의 방화벽** (Calico/Cilium이 처리).

### SG와 비슷한 듯 다름
- SG는 *VM 단위*. NetworkPolicy는 *Pod 단위* (라벨 기반).
- 둘 다 stateful + allow only + default deny (정책 적용 시).
- NetworkPolicy는 *namespace 범위* (cluster-wide는 Calico GlobalNetworkPolicy).

### Ch 09 Phase A에서 학습
- `default-deny-ingress` + `allow-client-to-web` 패턴.
- *기본 거부, 명시 허용* — SG와 같은 철학.

→ 자세한 건 [06-kubernetes.md](06-kubernetes.md).

---

## 디버깅 순서

"안 통함"이라는 증상에서 *어디가 막혔나* 좁히는 순서:

```
1. 라우팅 (L3): ping, IP 도달?
   $ ping <대상IP>
   → 안 되면: 라우트 테이블·IGW·NAT 의심 (02-routing-and-nat.md)

2. 포트 (L4): nc로 TCP handshake?
   $ nc -zv <IP> <port>
   $ telnet <IP> <port>      # 대안
   → 안 되면: SG → NACL → 호스트 방화벽 → 프로세스가 그 포트 듣나 순으로

3. 응용 (L7): curl로 실제 응답?
   $ curl -v <URL>
   → 안 되면: 서버 측 앱 문제 (nginx 설정 등). 또는 TLS 인증서.

4. DNS: 이름 해석?
   $ dig <도메인>
   $ nslookup <도메인>
   → 안 되면: CoreDNS / resolver / Route53 / NetworkPolicy(53/udp)
```

### "이 포트 누가 듣고 있나?" 한 줄
```bash
sudo ss -tlnp                  # TCP listen 포트 + 프로세스
sudo netstat -tlnp             # (구식 명령, 동일)
sudo lsof -i :8080             # 8080 듣는 프로세스
```

### 패킷 캡처로 실체 확인 (강력함)
```bash
sudo tcpdump -i any port 80 -nn        # 80번 포트 트래픽 보기
sudo tcpdump -i any host 1.2.3.4       # 특정 IP 트래픽 보기
sudo tcpdump -i eth0 -w /tmp/cap.pcap  # 파일로 저장 후 Wireshark
```
→ "분명 보냈는데 안 도착함" 같은 케이스에서 *어디서 사라졌나* 정확히 보임.

---

## 치트시트

### 안 통하면 점검 순서
```
1. ping <IP>            → L3 (라우팅·IGW·NAT)
2. nc -zv <IP> <port>   → L4 (SG → NACL → 호스트방화벽 → 프로세스)
3. curl <URL>           → L7 (앱 + TLS)
4. dig <도메인>         → DNS (resolver, CoreDNS, NetworkPolicy)
```

### "통한다면 *어느 IP*에서?" 항상 확인
- SG 룰의 source가 IP인지 SG 참조인지
- 내 IP가 NAT 뒤라면 검색되는 공인 IP가 진짜 발신지
- `curl ifconfig.me` or `curl checkip.amazonaws.com` 으로 내 공인 IP

### SG 운영 패턴
```
ALB SG    ←(443)── 인터넷
   │
   ▼ (8080, SG 참조)
App SG
   │
   ▼ (5432, SG 참조)
DB SG
```
→ IP 하드코딩 X. SG 참조로 인스턴스 늘어나도 룰 안 바뀜.

### NACL은 거의 안 건드림
- SG로 충분
- 건드릴 거면 *outbound ephemeral 포트* 잊지 말기

### 한 줄 정리
> **SG는 호실 문지기, NACL은 단지 정문 보안, 호스트 방화벽은 집 안 도어록.**
> **셋 다 통과해야 패킷이 프로세스에 닿는다.**

---

## 다음 읽을 거
- [04-load-balancers.md](04-load-balancers.md) — 여러 서버에 트래픽 분배. L4 vs L7.

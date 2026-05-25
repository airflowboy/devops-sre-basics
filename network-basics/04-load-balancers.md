# 04. 로드밸런서 — 여러 서버에 트래픽 분배

> "음식점 큐 매니저" — "1번 테이블 비었어요, 이쪽으로!" 손님들을 빈 자리로 분배.

## 목차
- [왜 로드밸런서가 필요한가](#왜-로드밸런서가-필요한가)
- [L4 vs L7 — 가장 중요한 구분](#l4-vs-l7--가장-중요한-구분)
- [AWS의 3종](#aws의-3종)
- [헬스 체크](#헬스-체크)
- [세션 어피니티](#세션-어피니티)
- [K8s Service vs Ingress vs LB](#k8s-service-vs-ingress-vs-lb)
- [치트시트](#치트시트)

---

## 왜 로드밸런서가 필요한가

### 문제 — 서버 1대로는 안 됨
- 트래픽이 한 대 용량 초과 → 느려지거나 죽음.
- 그 한 대가 죽으면 *서비스 전체 다운*.
- 배포 시 그 한 대 재시작 = 다운타임.

### 해법 — 같은 서버 여러 대 + 앞에 분배기
```
       [로드밸런서]
         /  |  \
        /   |   \
     서버A  서버B  서버C   ← 같은 앱, 3개 복제본
```

- 트래픽 분산 → 한 서버당 부담 ↓
- 한 서버 죽어도 LB가 알아서 살아있는 서버로
- 배포 시 한 대씩 빼고 업데이트 (rolling update)

→ **확장성 + 가용성 + 무중단 배포** 한 번에.

### 비유 — 인기 음식점
- 손님(트래픽) 100명이 한 번에 옴
- 입구의 매니저(LB)가 "이쪽 자리(서버A), 저쪽 자리(B)" 분배
- 서버 한 명이 점심 먹으러 가도 매니저가 다른 자리로 보내줌
- 그 사이 새 서버를 한 명 더 고용해서 자리 추가 가능 (HPA)

---

## L4 vs L7 — 가장 중요한 구분

> 로드밸런서가 *어느 OSI 계층에서 동작하느냐* — 능력이 크게 다름.

### L4 (Layer 4) — Transport
- **TCP/UDP 수준**에서 분배. 패킷의 IP·포트만 보고 결정.
- 빠르고 단순. 어떤 프로토콜이든 (HTTP/gRPC/PostgreSQL/Redis 다 가능).
- **연결 안의 내용은 안 봄** — 그래서 빠르지만 똑똑한 라우팅 불가.

### L7 (Layer 7) — Application
- **HTTP 헤더·경로 보고** 분배.
- 똑똑한 라우팅 가능:
  - `/api/users` → user-service
  - `/api/orders` → order-service
  - `Host: api.example.com` → API 백엔드
  - `Host: web.example.com` → 웹 백엔드
- TLS 종단(termination), 헤더 추가/수정, 인증, rate limit 등.
- 단점: L4보다 살짝 느림 (HTTP 파싱).

### 비교표
| | **L4 LB** | **L7 LB** |
|---|---|---|
| 본 거 | IP + 포트 | + HTTP 헤더 + 경로 |
| 프로토콜 | TCP/UDP 전부 | HTTP/HTTPS/gRPC |
| 라우팅 룰 | 백엔드 풀 1개 | 경로/호스트별 분기 |
| TLS 종단 | 통과만 (passthrough) | 종단 + 백엔드 평문 |
| 속도 | 빠름 | 살짝 느림 |
| 비용 (AWS) | NLB: ~$0.0225/hr | ALB: ~$0.025/hr |
| 흔한 용도 | DB, Redis, gRPC, 게임 | 웹 트래픽 90% |

### "둘 다 필요할 때"
- 인터넷 → ALB (L7, TLS 종단, 경로 라우팅) → 백엔드 Pod들
- 인터넷 → NLB (L4) → Redis 클러스터 (HTTP가 아닌 TCP)

### Ch 10에서 우리 선택
- 발급 API 같은 HTTP 트래픽 → **ALB** (L7)
- Kafka·Redis 같은 TCP → **클러스터 내부에서만** (외부 노출 안 함, NLB도 안 씀)

---

## AWS의 3종

| 이름 | 계층 | 특징 | 언제 |
|------|:---:|------|------|
| **ALB** (Application LB) | L7 | HTTP/HTTPS/gRPC 라우팅, WebSocket | 웹 트래픽 (대부분) |
| **NLB** (Network LB) | L4 | TCP/UDP, 정적 IP, 매우 빠름 | DB·게임·gRPC 트래픽 |
| **CLB** (Classic LB) | L4/L7 (옛 버전) | deprecated | 신규엔 안 씀 |

### ALB 특화 기능
- **Path-based routing** (`/api/*` → service-A, `/web/*` → service-B)
- **Host-based routing** (도메인별 분기)
- **HTTP 헤더 기반 라우팅**
- **WAF 통합** (Phase I에서 활용)
- **WebSocket 지원** (Phase E 가상 대기열 WebSocket)
- **TLS 인증서 — AWS Certificate Manager 통합**

### NLB 특화
- **정적 IP** (Elastic IP 부여 가능 — ALB는 DNS 이름만)
- **수십만 RPS 처리** (단일 LB)
- **TLS passthrough** (ALB는 종단)

### K8s에서
- **AWS Load Balancer Controller**가 K8s `Ingress` → ALB 자동 생성, `Service type=LoadBalancer` → NLB 자동 생성.
- 우리 EKS 캡스톤에서 이거 적극 활용 (manifest만 쓰면 ALB가 알아서).

---

## 헬스 체크

> "이 서버 살아있나?" 확인. 죽은 서버한테 트래픽 안 보내려고.

### 기본 동작
- LB가 주기적으로 백엔드에 요청 보냄 (예: `GET /healthz`)
- 2xx 응답 = 살아있음 (Healthy)
- 5xx 또는 응답 없음 N번 연속 = 죽음 (Unhealthy) → 트래픽 제외
- 다시 N번 연속 성공 = 복귀 (Healthy)

### 설정
```
Health check path: /healthz
Healthy threshold: 2 회 연속 성공
Unhealthy threshold: 3 회 연속 실패
Interval: 30초
Timeout: 5초
```

### 우리 프로젝트
- Ch 07의 Go 앱: `GET /healthz → 200 ok`
- K8s readiness probe와 비슷한 개념 (사실 LB가 K8s Service를 통해 본다면 Pod readiness가 더 빠름)

### 함정
- "Health check가 너무 자주 → 백엔드 부하" — 30초 정도면 충분
- "Health check 경로에 인증 걸려있음 → LB가 401 받고 unhealthy" — 헬스체크는 무인증 endpoint로
- 헬스체크 path가 *실제 트래픽 경로*와 다른 의존성 가지면 false negative

---

## 세션 어피니티 (Sticky Session)

> "같은 사용자는 같은 서버로 계속 보냄"

### 언제 필요?
- 세션을 **서버 메모리**에 저장하는 옛 앱 (서버 A에 로그인 했는데 B로 가면 다시 로그인)
- WebSocket 등 *연결 유지*가 중요한 경우

### 어떻게?
- 쿠키 기반 (ALB가 `AWSALB` 쿠키 발급)
- 소스 IP 기반 (NLB)

### 운영 권장
- **세션을 서버 메모리에 저장하지 말 것.** Redis 등 *공유 스토어*에.
- → 그러면 어느 서버로 가도 OK → sticky session 불필요 → LB가 더 자유롭게 분배.

### 우리 캡스톤
- 가상 대기열 WebSocket은 sticky 필요할 수도 (한 WebSocket 연결 유지)
- 발급 API는 stateless → sticky 불필요

---

## K8s Service vs Ingress vs LB

K8s에 LB 관련 *3개 추상화*가 있어 헷갈림. 정리:

| | **Service (ClusterIP)** | **Service (NodePort)** | **Service (LoadBalancer)** | **Ingress** |
|---|---|---|---|---|
| 노출 | 클러스터 내부만 | 노드의 30000~32767 포트 | 외부 LB(NLB/ALB)로 노출 | L7 라우팅 |
| 외부 접근 | ❌ | ✅ (노드 IP:포트) | ✅ (LB 주소) | ✅ (LB + 경로 라우팅) |
| 비용 | 무료 | 무료 | + LB 비용 | + LB 비용 (보통 ALB) |
| 흔한 용도 | 클러스터 내 통신 | 개발/테스트 | TCP 노출 (gRPC 등) | 웹 트래픽 (대부분) |

### 일반 패턴 (EKS + ALB)
```
인터넷
  │
  ▼
ALB (AWS Load Balancer Controller가 Ingress 보고 만듦)
  │
  ▼
Service (ClusterIP) — Pod IP 풀 추상화
  │
  ▼
Pod A, Pod B, Pod C
```

### 헷갈리는 거 — "Service가 LB 아닌가?"
- Service는 **클러스터 내부의 추상화**. 클러스터 안의 Pod IP들을 *하나의 가상 IP(ClusterIP)*로 묶음.
- kube-proxy가 *iptables*로 ClusterIP → 실제 Pod IP 분배 (단순 L4 LB).
- → "*외부에서* 들어오는 LB"는 ALB/NLB (AWS 자원), "*내부* 분배"는 Service.
- Ingress는 그 둘을 *이어주는 정책 객체* — Ingress Controller가 보고 실제 ALB 만듦.

→ 자세한 건 [06-kubernetes.md](06-kubernetes.md).

---

## 치트시트

### 빠른 선택
- HTTP/HTTPS 트래픽? → **ALB**
- TCP·UDP·gRPC? → **NLB**
- 정적 IP 필요? → **NLB** (또는 ALB + Global Accelerator)
- 클러스터 내부만? → **Service ClusterIP**

### L4 vs L7 — 시그널
- "경로별로 다른 서비스로?" → L7
- "DB·Redis 같은 비-HTTP?" → L4
- "TLS 종단?" → L7 (NLB도 가능하나 ALB가 표준)
- "초저지연·고처리량?" → L4

### 운영 패턴
```
Internet
  │ (HTTPS:443, ACM 인증서)
  ▼
ALB (TLS 종단 + 경로 라우팅 + WAF 통합)
  │ (HTTP:8080, 평문 — 내부망이라 OK)
  ▼
Pod (Go 앱)
```

→ **TLS는 ALB에서 종단**, 백엔드와는 평문. 단순 + 빠름. (zero-trust면 mTLS로 백엔드까지 암호화)

### 헬스체크 잘 만들기
- 가벼운 endpoint (`/healthz`)
- 의존성 *최소* (DB·외부 API 안 호출)
- 무인증
- 200 OK 만으로 충분 — 본문 무관

### 한 줄 정리
> **L4 = 빠르고 단순한 분배. L7 = 똑똑한 라우팅 (HTTP 한정).**
> **EKS + ALB Controller로 manifest만 쓰면 LB 자동.**

---

## 다음 읽을 거
- [05-aws-vpc.md](05-aws-vpc.md) — VPC·Subnet·IGW·NAT GW를 *Ch 08·10에서 우리가 만든 것*과 맞춰 보기.

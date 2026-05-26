# TLS & PKI — 전송 보안의 토대

> k8s apiserver·kubelet·etcd 통신, ingress, cert-manager, ACM, IRSA OIDC, registry 인증,
> prometheus scrape, argocd repo 접근, GitHub Actions OIDC federation —
> 모두 같은 X.509·TLS·OIDC 위에 쌓인다.

---

## 1. X.509 인증서 — 정체를 증명하는 구조

X.509는 *공개키 + 메타데이터 + CA의 서명* 묶음. 인증서 한 장이 다음을 증명:

1. 이 공개키는 이 *주체(subject)* 의 것이다 (CN/SAN으로 식별)
2. 그 사실을 이 *CA가 서명* 했다
3. 이 기간(NotBefore/NotAfter) 동안만 유효

### 필드 구조

```
Certificate:
  Version: v3
  Serial Number: ...
  Issuer: CN=My CA, O=Example
  Validity:
    Not Before: 2026-01-01
    Not After:  2027-01-01
  Subject: CN=api.example.com
  Subject Public Key Info: RSA 2048 / ECDSA P-256 / Ed25519
  Extensions:
    Subject Alternative Name (SAN): DNS:api.example.com, DNS:*.example.com
    Key Usage: digitalSignature, keyEncipherment
    Extended Key Usage (EKU): serverAuth, clientAuth
    Basic Constraints: CA:FALSE
    Authority Key Identifier: ...
  Signature Algorithm: sha256WithRSAEncryption
  Signature: <CA의 개인키로 위 내용을 서명>
```

### CN deprecated — SAN을 써라

과거에는 `CN=api.example.com` 으로 호스트명 매칭했다. 2017년 Chrome 58부터 **SAN만** 본다 (RFC 6125 권고). CN에만 있고 SAN에 없으면 *브라우저가 거절*.

자체 CA로 cert를 만들 때 SAN을 빼먹는 게 *입문자의 90% 함정*. openssl로 생성 시 `-addext "subjectAltName=DNS:..."` 필수.

### EKU — *어떤 용도* 의 인증서인가

같은 cert를 server·client 양쪽에 쓰면 위험. EKU로 용도 분리:

| EKU | 의미 |
|---|---|
| `serverAuth` | TLS server (웹사이트) |
| `clientAuth` | TLS client (mTLS의 클라이언트 쪽) |
| `codeSigning` | 코드 서명 |
| `emailProtection` | S/MIME |

k8s의 mTLS는 *같은 CA* 가 발급하지만 EKU로 server cert와 client cert를 구분한다.

---

## 2. CA 체인과 신뢰 검증

### 체인의 모양

```
Root CA (self-signed, 신뢰의 anchor)
   └─ Intermediate CA (Root가 서명)
        └─ Leaf cert (Intermediate가 서명, 실제 서비스가 사용)
```

서버는 보통 *Leaf + Intermediate* 를 보낸다 (Root는 클라이언트가 *이미 갖고 있어야* 함). 그래서 *intermediate 빼먹기* 가 두 번째로 흔한 함정. `openssl s_client -showcerts -connect host:443` 으로 체인 확인.

### Trust store — *Root CA* 를 어디서 가져오는가

| 위치 | 무엇 |
|---|---|
| `/etc/ssl/certs/ca-certificates.crt` | Debian/Ubuntu |
| `/etc/pki/tls/certs/ca-bundle.crt` | RHEL/CentOS |
| `/etc/ssl/cert.pem` | Alpine |
| `$JAVA_HOME/lib/security/cacerts` | JVM (별도!) |
| OS keychain | macOS |
| `mkcert -CAROOT` | 로컬 dev |

container 이미지가 `scratch` 또는 `distroless` 면 ca-certificates 패키지를 *명시적으로 복사* 해야 한다. 안 그러면 outbound HTTPS가 전부 깨진다 — Go 바이너리의 *흔한 first-deploy 장애*.

### 자체 CA의 신뢰 주입

- k8s: `kube-apiserver --client-ca-file=...`, kubeconfig의 `certificate-authority-data`
- container: 이미지 빌드 시 ca cert 파일을 `/usr/local/share/ca-certificates/` 에 넣고 `update-ca-certificates`
- Go: `crypto/tls.Config.RootCAs` 또는 `SSL_CERT_FILE` 환경변수
- JVM: `keytool -import` 로 cacerts에 추가

---

## 3. TLS Handshake — 무엇이 오가는가

(TLS 1.3 기준, 1.2는 RTT가 한 번 더)

```
Client                                            Server
  │  ClientHello (지원 cipher, key share, SNI)   │
  ├─────────────────────────────────────────────>│
  │                                              │  ServerHello + cert + cert verify
  │<─────────────────────────────────────────────┤
  │  Finished                                    │
  ├─────────────────────────────────────────────>│
  │  (이후 application data — 암호화됨)            │
  │<════════════════════════════════════════════>│
```

핵심:
- **SNI (Server Name Indication)** — 클라이언트가 *어느 호스트* 와 통신하려는지 평문으로 알림. 같은 IP에 여러 호스트가 있을 때 서버가 어느 cert를 보낼지 결정. ESNI/ECH로 암호화 시도 중.
- **Cert verify** — 서버가 자기 *개인키* 로 핸드쉐이크를 서명. 클라이언트가 cert의 공개키로 검증 → "이 cert를 제시한 게 진짜 그 cert의 소유자"
- **Cipher negotiation** — TLS 1.3은 AEAD 한정 (AES-GCM, ChaCha20-Poly1305). 1.2는 CBC도 가능 (취약).

### mTLS

서버뿐 아니라 클라이언트도 cert로 자신을 증명. 핸드쉐이크 중 서버가 `CertificateRequest` 보내고, 클라이언트가 자기 cert 제시.

| 일반 TLS | mTLS |
|---|---|
| 클라이언트가 서버 신원 검증 | 양쪽이 서로 검증 |
| 서버 cert만 필요 | 양쪽 모두 cert 필요 |
| 클라이언트 인증은 *별도 레이어* (token·session) | 클라이언트 인증이 *TLS 자체* |

k8s 내부 통신 (apiserver ↔ kubelet ↔ etcd)이 전부 mTLS. cert가 곧 정체.

---

## 4. k8s의 PKI — 같은 패턴이 어디에나

k8s 클러스터는 *기본적으로* 다음 CA들을 운영:

| CA | 무엇을 서명 |
|---|---|
| `kubernetes-ca` (cluster CA) | apiserver, kubelet client, controller-manager, scheduler |
| `etcd-ca` | etcd peer 간, etcd server↔client |
| `front-proxy-ca` | aggregator (API extension) |
| ServiceAccount signing key (CA 아님, key pair) | SA token (JWT) 서명 |

`/etc/kubernetes/pki/` 에 다 있다. kubeadm이 이걸 자동 생성.

### Cert 만료 1년 함정

kubeadm은 cert를 *1년 유효* 로 발급. 클러스터를 1년 이상 운영하면 *어느 날 갑자기 apiserver가 안 뜸*. 해결:
- `kubeadm certs renew all`
- `kubeadm upgrade` 가 자동 갱신 → 정기 upgrade가 사실상 cert rotation
- v1.19+ 에서 kubelet client cert는 *자동 rotation* 지원 (`--rotate-certificates`)

### Pod이 mount 받는 cert

- ServiceAccount token: `/var/run/secrets/kubernetes.io/serviceaccount/`
  - `ca.crt` — apiserver를 검증할 때 쓸 CA
  - `token` — apiserver에 자신을 증명할 때 쓸 JWT (→ [[identity-and-auth]] §3)
- cert-manager가 발급한 cert: Secret으로 마운트, 또는 CSI driver로 직접 주입

---

## 5. cert-manager와 자동 발급

### ACME (Let's Encrypt) 흐름

```
1. cert-manager가 CSR 생성 + ACME 서버에 요청
2. ACME가 challenge 발급 — "이 도메인 통제하는 거 증명해라"
   - HTTP-01: /.well-known/acme-challenge/... 에 파일 호스팅
   - DNS-01: _acme-challenge.example.com TXT 레코드 생성
3. cert-manager가 challenge 응답 (ingress·DNS provider 사용)
4. ACME가 확인 → cert 발급
5. cert-manager가 cert를 Secret으로 저장 → ingress가 사용
```

DNS-01이 wildcard cert 발급 가능, internal 도메인에도 작동 (HTTP-01은 외부 접근 필요).

### 자체 CA + cert-manager

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: my-ca }
spec:
  ca:
    secretName: my-ca-keypair   # Root CA의 cert + key
```

→ 클러스터 내부용 cert를 *자동 갱신* 가능. 사람이 cert 만들고 mount하던 시대의 끝.

---

## 6. OIDC와 JWT — *cert 없는* 신원 증명

cert는 *발급/갱신/회수* 가 비싸다. 단기 작업 (CI 1회 실행, Pod 1회 lifetime)에는 과하다. → **OIDC + JWT** 모델.

### JWT 구조

```
header.payload.signature
  │     │        └─ IdP의 개인키로 위 둘을 서명
  │     └─ 클레임 (sub, iss, aud, exp, iat, ...)
  └─ 알고리즘 (RS256, ES256, ...)
```

검증:
1. IdP의 **공개키 (JWKS)** 를 신뢰된 endpoint에서 fetch (`/.well-known/jwks.json`)
2. signature를 그 공개키로 검증
3. `exp` (만료), `nbf` (유효 시작), `aud` (의도된 수신자), `iss` (발급자) 확인

### OIDC = "JWT를 표준화한 신원 protocol"

OIDC discovery: `/.well-known/openid-configuration` 에 IdP의 모든 endpoint·JWKS URL·지원 알고리즘이 적혀 있음.

**핵심 통찰**: cert는 *장기 신원*, JWT는 *단기 토큰*. 둘 다 비대칭 암호로 서명되지만 운영 cost가 다름.

### 인프라에서 OIDC가 등장하는 곳

| 시나리오 | IdP | Audience |
|---|---|---|
| k8s SA token | apiserver | `https://kubernetes.default.svc` (기본) |
| IRSA (k8s SA → AWS) | k8s apiserver의 OIDC issuer URL (S3에 호스팅) | `sts.amazonaws.com` |
| GitHub Actions → AWS | `token.actions.githubusercontent.com` | `sts.amazonaws.com` |
| GitHub Actions → GCP | (같은 IdP) | GCP workload identity pool |
| ArgoCD SSO | corporate IdP (Okta·Keycloak·Google) | `argocd` |

→ [[identity-and-auth]] §5~7에서 이 *교환 메커니즘* 을 자세히.

---

## 7. Cross-reference 지도

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| k8s 내부 mTLS 구조, apiserver auth | `kubernetes-basics/` security, `k8s-security-basics/` |
| ingress + cert-manager 실제 구성 | `kubernetes-basics/` ingress |
| Container 이미지에 ca-certificates 넣기 | `docker-basics/` 이미지 |
| AWS ACM, ALB TLS termination | `aws-basics/` 네트워킹 |
| Prometheus scrape TLS | `observability-basics/` Prometheus |
| Registry 인증 (Docker Hub, ECR, ghcr.io) | `docker-basics/` 이미지, `cicd-gitops-basics/` |
| OIDC token이 *왜 신원인가* | [[identity-and-auth]] |
| Cert 만료가 reconciliation을 깨는 사례 | [[declarative-and-reconciliation]] §5 drift |

---

## 한 줄 요약

> **TLS는 두 가지를 보장한다 — 통신 상대의 정체, 통신 내용의 비밀.**
> X.509는 *장기 신원*, JWT는 *단기 토큰*. 둘 다 비대칭 암호 위에 있다.
> 인프라 보안 사고의 절반은 *cert 만료*, 나머지 절반은 *SAN/EKU 빠뜨림* 이다.

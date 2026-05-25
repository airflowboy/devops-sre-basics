# DevOps / SRE Basics — 동작 원리 중심 reference

> *"docker는 어떻게 격리를 만드나요?"*
> *"Kubernetes Pod이 죽으면 어떻게 다시 뜨나요?"*
> *"ArgoCD는 어떻게 git을 단일 진실 공급원으로 만드나요?"*
>
> 위 질문에 *외운 답*이 아니라 *작동 원리부터 설명*할 수 있는 상태가 목표.

---

## 🎯 이 repo의 목적

DevOps/SRE 도구들을 **튜토리얼 사용법** 이 아니라 **작동 원리·내부 메커니즘** 관점에서 정리한 reference.

- ✅ *왜 그렇게 작동하는가* 를 공식 문서·소스 코드·표준(OCI, K8s API spec, RFC 등) 기반으로 설명
- ✅ 도구 간 *연결고리* — "이건 그때 저거랑 같은 원리야" 식의 cross-reference
- ✅ 면접에서 *깊이 질문* 들어왔을 때 한 단계씩 좁혀가는 사고 흐름
- ❌ 단순 사용법·튜토리얼 (그건 공식 문서·블로그가 더 잘 함)
- ❌ 특정 버전 의존 명령어 모음 (도구는 바뀜, 원리는 안 바뀜)

---

## 📚 폴더 구성

| 폴더 | 주제 | 파일 |
|---|---|:---:|
| [`network-basics/`](network-basics/) | IP·CIDR·라우팅·NAT·LB·VPC·K8s 네트워킹 | 7 |
| [`linux-basics/`](linux-basics/) | systemd·프로세스·시그널·FS·권한·cgroup·namespace·troubleshooting | 6 |
| [`docker-basics/`](docker-basics/) | 이미지·레이어·격리·네트워크·볼륨·빌드·보안 | 7 |
| [`kubernetes-basics/`](kubernetes-basics/) | Control plane·kubelet·etcd·workload·스케줄링·CRD·RBAC | 8 |
| [`helm-basics/`](helm-basics/) | chart·template·release·hooks·values 설계·OCI registry | 6 |
| [`ansible-basics/`](ansible-basics/) | playbook·inventory·idempotency·role·collection·AWX | 6 |
| [`observability-basics/`](observability-basics/) | 3 pillars·Prometheus·PromQL·Alertmanager·SLO·error budget·OpenTelemetry | 7 |
| [`cicd-gitops-basics/`](cicd-gitops-basics/) | CI vs CD·GitHub Actions·ArgoCD·deployment strategies·supply chain | 6 |
| [`terraform-basics/`](terraform-basics/) | provider·state·module·workflow·Terragrunt·Atlantis | 6 |
| [`k8s-security-basics/`](k8s-security-basics/) | PSS·NetworkPolicy·IRSA·supply chain·secret 관리 | 6 |
| [`aws-basics/`](aws-basics/) | IAM·VPC·EKS·S3·RDS·KMS·FinOps | 7 |

**총 11 폴더 / 72 파일.**

각 폴더 README는 *큰 그림 + 자주 헷갈리는 것 + 토픽 인덱스*. 그 다음 토픽 파일들이 *깊이*.

---

## 🔗 Cross-reference 정책

도구들은 *서로의 기반 위에 쌓여* 있다. 같은 개념이 여러 폴더에서 다른 이름·다른 추상화 수준으로 등장:

| 같은 원리 | 등장 위치 |
|---|---|
| **Linux namespace** | docker-basics/02 (격리 구현) → kubernetes-basics/01 (Pod = namespace 공유) → k8s-security-basics/01 (PSS) |
| **Network namespace + veth + bridge** | network-basics/06 (CNI 입문) → docker-basics/03 (docker0 bridge) → kubernetes-basics/03 (Pod 네트워킹) |
| **iptables / DNAT** | network-basics/02·03 → docker-basics/03 (`-p` publish) → kubernetes-basics/03 (kube-proxy iptables 모드) |
| **선언적 상태 reconciliation** | kubernetes-basics/06 (controller 패턴) → cicd-gitops-basics/03 (ArgoCD sync loop) → terraform-basics (plan-apply) |
| **OCI 표준** | docker-basics/01 (image spec) → kubernetes-basics (containerd·CRI-O가 OCI 런타임) → helm-basics/05 (OCI chart) |
| **OIDC trust** | cicd-gitops-basics/02 (GitHub Actions ↔ AWS) → kubernetes-basics/07 (IRSA) → k8s-security-basics/03 (Workload Identity) → aws-basics/01 |
| **RBAC** | kubernetes-basics/07 → k8s-security-basics/03 → aws-basics/01 (IAM 비교) |
| **공급망 보안 (cosign·SBOM·admission)** | docker-basics/06 → cicd-gitops-basics/05 → k8s-security-basics/04 |
| **External Secrets Operator** | kubernetes-basics/04 → k8s-security-basics/05 → aws-basics/05 |

→ 토픽 파일 안에 *"이건 X와 같은 원리, 거기선 Y라고 부른다"* 같은 노트가 적극 박혀 있음.

---

## 🎓 활용 시나리오

1. **챕터 학습 전** — 해당 폴더 README 훑어 큰 그림 잡기
2. **막힐 때** — 해당 토픽 파일에서 *작동 원리*로 거슬러 올라가기
3. **면접 준비** — 각 폴더 README의 *"자주 헷갈리는 N가지"* 와 *"면접 빈출 질문"* 섹션 위주로 복습
4. **다른 도구 학습 시작 시** — cross-reference 따라가서 *내가 이미 아는 원리의 다른 이름* 확인

---

## 📐 작성 원칙 (모든 문서 공통)

- **공식 문서·표준 우선 인용** (각 토픽 시작부에 *어느 공식 문서/spec 기반*인지 표기)
- **작동 원리 → 사용법 → 흔한 함정 → 면접 빈출 Q&A** 순서
- **그림 위주가 아닌 *흐름 위주*** (ASCII flow OK, 필요 시 Mermaid)
- **숫자·예시 구체적으로** (예: "큰 이미지" 가 아니라 "node:20-alpine 약 130MB")
- **각 토픽 끝에 3줄 요약** — *지하철에서 면접 직전 훑기용*

---

## 🎤 면접 빈출 — 폴더별 핵심 질문

| 폴더 | 핵심 질문 |
|---|---|
| network-basics | NAT 동작, 라우팅 테이블 해석, Load Balancer 종류 |
| linux-basics | 좀비/OOM/Load Average 의미, systemd dependency, 장애 디버깅 12단계 |
| docker-basics | Docker는 어떻게 격리? K8s가 docker 빼는 의미? layer caching? |
| kubernetes-basics | kubectl apply 8단계, Controller pattern, IRSA |
| helm-basics | chart 구조, values 우선순위, checksum/config 패턴 |
| ansible-basics | 멱등성 보장, 변수 22단계, Ansible vs Terraform |
| observability-basics | RED vs USE, multi-burn-rate alert, Distributed tracing |
| cicd-gitops-basics | GitOps push vs pull, OIDC trust boundary, Progressive Delivery |
| terraform-basics | state 필요한 이유, plan vs apply, Drift detection |
| k8s-security-basics | PSS profile, NetworkPolicy default-deny, Cosign keyless 원리 |
| aws-basics | User vs Role, IRSA 동작, KMS envelope encryption, Spot 안전 사용 |

---

## 📦 관련 학습 repo

- **[airflowboy/devops-sre-redo](https://github.com/airflowboy/devops-sre-redo)** — 이 reference 문서를 *손으로 적용*하는 학습 프로젝트 (멘토 모드)
- **[airflowboy/sre-project](https://github.com/airflowboy/sre-project)** (구 `SRE로드맵`) — 1차 사이클 (AI 주도로 빠르게 흐름 본 reference 코드베이스)

# foundations/ — 11개 챕터를 가로지르는 토대

> 한 문장: 도구별 챕터가 *수직* 이라면, 여기는 *수평*.
> 같은 원리가 여러 챕터에서 다른 이름·다른 추상화 수준으로 등장할 때, 그 원리 자체를 한 번 깊이 정리한 문서들.

Spring AOP의 *aspect* 와 같은 위치. 각 챕터(클래스/메서드)가 수직이라면, foundation은 *그 위를 가로지르는 횡단 관심사*.

---

## 1. 이 폴더가 존재하는 이유

repo README의 "🔗 Cross-reference 정책" 표를 *지탱하는 뿌리* 문서들. 다음 조건을 모두 만족할 때만 여기 들어온다:

- ✅ **3개 이상 챕터를 가로지른다**
- ✅ **단일 챕터의 자식 토픽이 아니다**
- ✅ **도구 사용법·튜토리얼이 아니다** — 원리·메커니즘 중심
- ❌ "있으면 좋은 잡지식" 은 들어오지 않는다

이 가드레일이 *foundations 폴더의 비대화를 막는다*. 폴더가 11개 챕터 중 하나처럼 커지면 *분류 모델 자체가 깨진다*.

---

## 2. 문서 인덱스 (읽는 순서)

1번이 cross-ref **허브** — 나머지가 여기서 가지를 친다. 1번부터 순서대로 읽기를 권장.

| # | 문서 | 한 줄 |
|---|---|---|
| 1 | [declarative-and-reconciliation.md](declarative-and-reconciliation.md) | k8s·terraform·ansible·argocd가 공유하는 메타 패턴 |
| 2 | [shell-scripting.md](shell-scripting.md) | bash/POSIX 원리 — entrypoint·hook·playbook의 공용어 |
| 3 | [tls-and-pki.md](tls-and-pki.md) | 전송 보안 — 모든 컴포넌트 통신의 토대 |
| 4 | [identity-and-auth.md](identity-and-auth.md) | 신원 보안 — uid부터 IRSA·OIDC federation까지 |
| 5 | [yaml-and-templating.md](yaml-and-templating.md) | 설정의 공용 언어 + Go template / Jinja2 |
| 6 | [go-for-infra.md](go-for-infra.md) | 인프라 도구 *읽을 수 있는* 수준의 Go 이해 |

---

## 3. 챕터 ↔ Foundation 매트릭스

각 셀의 ●는 *그 챕터에서 이 foundation이 작동 원리의 일부로 등장* 한다는 의미.
단순히 "한번 언급됨"은 제외 — 그래야 신호가 의미를 가진다.

|  | linux | docker | k8s | helm | ansible | obs | cicd | tf | k8s-sec | aws | net |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 1. declarative-reconciliation |   |   | ● | ● | ● |   | ● | ● |   |   |   |
| 2. shell-scripting | ● | ● | ● |   | ● |   | ● | ● |   |   |   |
| 3. tls-and-pki |   | ● | ● | ● |   | ● | ● |   | ● | ● | ● |
| 4. identity-and-auth | ● | ● | ● |   | ● |   | ● | ● | ● | ● |   |
| 5. yaml-and-templating |   | ● | ● | ● | ● | ● | ● | ● | ● |   |   |
| 6. go-for-infra |   | ● | ● | ● |   | ● | ● | ● |   |   |   |

### 읽는 법

- **횡축** = 챕터를 읽다가 "이 원리 어디서 자세히 보지?" → foundation으로
- **종축** = foundation을 읽다가 "이걸 실제로 어디서 만나지?" → 챕터로

각 foundation 문서 마지막에 *"Cross-reference 지도"* 표가 있고, 챕터 문서에서는 `[[name]]` 으로 역링크 권장.

> 📌 매트릭스는 *예고* 다. 실제 챕터 문서에 cross-ref 링크를 *심는* 작업이 따로 필요.
> 매 챕터 README를 한 번 훑으며 ● 표시된 foundation으로 `[[link]]` 추가하는 게 다음 작업.

---

## 4. 폴더 비대화 방지 약속

현재 6개. 후보 추가 시 §1의 *세 조건* 을 재검토.

**추가 후보 (보류)**:
- `time-and-clocks.md` — NTP skew, cert/JWT 만료, 분산 트레이스 시간 정렬. 작지만 운영 사고 빈도 높음. 6개 cross-ref 채워본 뒤 *진짜 필요한지* 재평가.

**추가 거부된 후보** (왜 빠졌는지):
- regex-and-text → 도구 묶음, 원리 깊이 얕음
- logging conventions → `observability-basics` 와 정면 충돌
- process & init → §2 shell-scripting + linux-basics에 흡수됨
- networking primitives → `network-basics` 가 본진
- caching/TTL → 너무 narrow

새 문서를 추가하고 싶을 때는 *기존 6개에 흡수 가능한지 먼저 확인*. 추가는 *비용* 이지 *공짜* 가 아니다.

---

## 5. 작성·유지보수 원칙

- **튜토리얼 아님**. "이 도구를 어떻게 설치하나" 는 챕터에. foundation은 *왜 그렇게 작동하나*.
- **공식 문서·표준·소스 코드** 인용 우선. 블로그 X.
- **Cross-ref 밀도** 가 자산. 각 문서가 다른 문서·챕터로 *최소 5곳* 링크.
- 매 문서 끝에 **"한 줄 요약"** — 한 문장으로 압축 안 되면 글이 흩어져 있는 것.

---

## 한 줄 요약

> **Foundation = 11개 챕터의 *공통 토대*.**
> 이걸 한 번 정리하면 *각 챕터에서 같은 설명이 반복되지 않고, 깊이가 올라간다*.
> 단, 욕심내면 *제2의 chapter 폴더*가 되어 분류 모델이 깨진다. 6개에서 멈추는 게 핵심.

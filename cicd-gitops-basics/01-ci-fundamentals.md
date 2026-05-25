# 01 — CI Fundamentals: 파이프라인이 *진짜* 보장해야 하는 것

> **공식 근거:** [Continuous Integration (Martin Fowler)](https://martinfowler.com/articles/continuousIntegration.html), [OCI / SLSA spec](https://slsa.dev/)
>
> **다음:** [02-github-actions.md](02-github-actions.md) (예정), [05-build-and-registry.md](05-build-and-registry.md) (예정)

---

## 🎯 한 문장

> **CI = *코드 변경마다 자동으로 *통합·검증해 *재현 가능한 산출물*을 만드는 프로세스*. 진짜 가치는 *빠른 피드백 + 통합 비용 분산 + 산출물 재현성*. 단순히 "테스트 자동 실행"이 아님.**

---

## 1. CI의 *진짜* 가치 — 3가지

### 1-1. *빠른 피드백*
- 코드 push → *수 분 안에* 통과/실패 결과
- *늦은 피드백*은 *늦은 디버깅 비용* (코드 컨텍스트 사라짐, 다른 작업으로 넘어감)

### 1-2. *통합 비용 분산*
- "*마지막에 한 번에 합치자*" → *기말 시험 = 며칠간의 머지 헬*
- "*매일 main에 머지*" → *작은 비용 자주*. 충돌·통합 문제 *즉시 발견·해결*

### 1-3. *재현 가능한 산출물*
- "*내 노트북에선 됐는데...*" 방지
- *CI는 깨끗한 환경에서 빌드* → *그게 production이 받는 산출물*
- *binary 해시*, *image digest*가 *"검증된 것"의 증거*

---

## 2. CI 파이프라인의 표준 단계

```
1. checkout       (코드 받기)
2. lint           (코드 스타일·문법 검사)
3. unit test      (격리된 단위 테스트)
4. SAST/SCA/secret-scan  (보안·취약점·시크릿 누출)
5. build artifact (image / package / binary)
6. integration test (실 의존성과 통합)
7. push artifact  (registry / artifact store)
8. (선택) cosign sign + SBOM
9. trigger CD      (ArgoCD가 자동 감지하는 경우 trigger 불필요)
```

각 단계의 *목적*과 *실패 시 의미*가 분명해야:

| 단계 | 실패가 의미하는 것 |
|---|---|
| lint | 스타일 위반 → reviewer 시간 낭비. *coding standard 강제* |
| unit test | 로직 회귀. *bug fix를 보장하는 안전망* |
| SAST | 코드 패턴의 보안 결함 (SQL injection, XSS 등) |
| SCA | 의존성의 *알려진 CVE* (log4shell 같은) |
| secret-scan | git에 *AccessKey·token 누출* (gitleaks 등) |
| build | 컴파일·이미지 생성 자체 fail |
| integration test | 외부 의존성과의 *통합 회귀* |

---

## 3. *Trunk-based vs GitFlow* — 분기 전략의 차이

| 항목 | Trunk-based | GitFlow |
|---|---|---|
| 메인 브랜치 | `main` 만 | `main`, `develop`, `release/*`, `hotfix/*` |
| feature 브랜치 | *짧게* (수 시간 ~ 1일), 자주 main으로 머지 | *길게* (수 일 ~ 수 주), develop으로 머지 |
| 머지 빈도 | 매일 main 머지 | 릴리즈 시점에 main |
| CI 부하 | main에 자주 → *항상 main이 deployable* | 릴리즈 시점에 부하 집중 |
| 추천 | 현대 DevOps · 빠른 배포 | 릴리즈 사이클이 긴 제품 (모바일·임베디드) |

→ 현대 DevOps는 *대부분 trunk-based*. *feature flag*로 *완성 안 된 기능 숨김*. CI/CD가 *매번 deployable*.

---

## 4. *Test Pyramid* — *비용·속도·신뢰의 trade-off*

```
            /\
           /  \   E2E (수십 개)
          /----\
         /      \  Integration (수백 개)
        /--------\
       /          \  Unit (수천 개)
      /____________\
```

- **Unit**: *빠르고 싸다*. 격리된 함수·모듈 테스트. CI에서 *매번* 실행.
- **Integration**: *중간 비용*. DB·메시지 큐 등 *실 의존성*과. *PR마다* 실행.
- **E2E**: *느리고 비싸다*. 실제 UI·전체 시스템. *밤마다 한 번* 또는 *PR 라벨 시*.

흔한 안티패턴: *E2E만 많이* 작성 → *느린 피드백 + flaky test 폭증*.

---

## 5. 파이프라인 *속도 최적화* — 자주 빠지는 함정

### 5-1. *캐시 적극 활용*
- *의존성 다운로드*는 매번 안 함 (`npm install`, `pip install`, `go mod download`)
- *image layer 캐시*는 *registry cache mount* 활용 (BuildKit)
- *test 결과 캐시*는 *변경된 모듈만* 재실행 (Bazel·Turbo·Nx)

### 5-2. *병렬화*
- 독립된 job은 *동시 실행* (matrix·parallel keys)
- *전체 시간 ≠ 모든 단계 합* — *DAG의 critical path*

### 5-3. *unit과 integration 분리*
- unit은 항상, integration은 *변경 영역에 해당하는 것만*
- monorepo면 *changed files 기반 selective run*

### 5-4. *flaky test 즉시 격리*
- 가끔 fail하는 test는 *팀 신뢰를 갉아먹음* — *retry로 가리지 말고 fix 또는 quarantine*
- *flaky rate 모니터링*

### 5-5. *fast-fail*
- *lint·secret-scan 같은 빠른 검사 먼저* → 일찍 차단
- *느린 integration test* 늦게

---

## 6. *Artifact의 *불변성*과 *재현성**

### 불변성
- 한 번 만든 artifact는 *수정 안 함*. 새 변경 = 새 artifact.
- *digest로 식별* (`@sha256:abc`)
- *tag는 alias* — 바뀔 수 있음. *digest pin이 정석*.

### 재현성
- 같은 코드 + 같은 의존성 → *비트 단위로 같은 binary* 가 *이상*
- *완전한 재현성*은 어려움 (timestamp, build ID 등). *근사적 재현성*은 일반적.
- *SLSA Level 2~4*가 *재현성·증명 가능성*의 기준 ([공급망 보안 — `docker-basics/06` 참조](../docker-basics/06-security.md))

---

## 7. *Secrets 관리* — CI에서 자주 빠지는 것

### 옛 방식 (위험)
- `.env` 파일을 *CI에 committed*
- 환경변수 평문 출력 → *CI 로그에 남음*
- 정적 cloud AccessKey를 *CI secret*에 박음 → *유출 시 영구 위험*

### 모범
- *secret은 CI provider의 secret store에* (GitHub Encrypted Secrets, GitLab CI Variables, Vault 등)
- *로그 마스킹* — secret이 stdout에 노출되면 자동 ****
- **OIDC 기반 임시 자격증명** — *정적 키 0개*
  - GitHub Actions ↔ AWS: `aws-actions/configure-aws-credentials@v4`
  - GitLab CI ↔ AWS: id_tokens
  - 둘 다 *단기 토큰*. 유출돼도 짧은 유효기간
- *secret 자체를 git에 안 박음* (.gitignore + pre-commit gitleaks)

### Secret이 *image에 박히는* 사고
- `ARG SECRET=xxx` 빌드 → image history에 남음 → `docker history` 로 누구나 볼 수 있음
- → BuildKit `--mount=type=secret` 사용 ([`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md))

---

## 8. *CI 환경의 격리* — Runner 선택

| Runner 유형 | 격리 | 보안 | 비용 |
|---|---|---|---|
| **공유 호스트** (직접 운영) | 약 | 약 | 저 |
| **VM per job** (GitHub-hosted, GitLab shared) | 강 | 강 | 중 |
| **Container per job** | 중 | 중 | 저 |
| **Self-hosted runner** | 직접 결정 | 직접 결정 | 저 (운영비) |

### Self-hosted runner의 *위험*
- 한 job의 *artifact·tmpfile* 이 다음 job에 *남을 수 있음*
- *PR fork의 코드*가 runner에서 *임의 코드 실행* → 호스트 침해 가능성
- → *ephemeral runner* (매 job마다 destroy), *fork PR은 manual approve* 필수

---

## 9. *Branch protection + CI 게이트*

main 브랜치 보호 룰 (GitHub 예):
- *Require pull request before merging*
- *Require status checks to pass* — CI 통과 못 하면 머지 X
- *Require approvals* (n명)
- *Require linear history* (rebase·squash 강제)
- *Restrict who can push* (관리자도 강제)

→ *CI가 단순 알림이 아니라 *머지 게이트*가 되는 것*이 핵심. *통과 안 하면 main에 못 들어옴*.

---

## 10. 자주 헷갈리는 것

### 10-1. *CI green = 안전*이 아님
- CI는 *알려진 종류의 검증*만. *예상 못 한 회귀*는 못 잡음
- *post-deploy verification* (canary, smoke test)도 *CD 단계*에 필요

### 10-2. *CI를 너무 큰 monorepo로*
- 작은 변경에도 *전체 빌드 30분* → *팀이 CI 안 봄*
- *changed files 기반 selective build* (Bazel·Nx·Turbo·`paths` 필터)

### 10-3. *image tag로 배포*
- *mutable tag* → reproducibility 깨짐
- *digest pin* 필수 (`@sha256:...`)

### 10-4. *secret이 PR fork에 노출*
- GitHub Actions의 `pull_request` 이벤트는 *fork의 PR에 secret 비공개* (보안 default)
- `pull_request_target` 사용 시 *secret 공개 + fork 코드 실행* → *매우 위험*
- → 일반 PR은 `pull_request`만 사용

### 10-5. *flaky test를 retry로 숨김*
- *근본 원인 안 잡으면* 영원히 *팀 신뢰 갉아먹음*
- *flaky rate 측정* + *주기적 quarantine* + *fix backlog*

---

## 🎤 면접 빈출 Q&A

### Q1. CI와 CD의 차이는?
> **CI**: *코드 → 검증된 산출물 (image, package, binary)*. lint·test·SAST·SCA·build·push. **CD**: *산출물 → 운영 환경*. Canary·rollback·환경별 promotion. 한 도구가 둘 다 할 수 있어도 *책임·권한 분리*가 좋음 (CI는 만들기, CD는 운영 적용). GitOps에선 *CI가 manifest repo에 commit*하고 *CD는 ArgoCD가 watch*.

### Q2. CI 파이프라인 *속도 최적화* 어떻게?
> (1) *캐시 적극* — 의존성·image layer·test 결과. (2) *병렬화* — 독립된 job 동시. critical path 단축. (3) *fast-fail* — lint·secret-scan 같은 *빠른 검사 먼저*. (4) *selective build* — monorepo에선 *changed files 기반*. (5) *flaky test 격리* — retry로 숨기지 말고 *quarantine + fix*.

### Q3. CI에서 secret을 안전하게 다루려면?
> (1) *secret store* 사용 (GitHub Encrypted Secrets, GitLab CI Variables, Vault 통합). (2) *로그 마스킹* — secret이 stdout 노출 시 자동 ****. (3) **OIDC 기반 임시 자격증명** — 정적 키 0개, AWS·GCP 모두 지원. (4) *secret을 git에 안 박음* — pre-commit `gitleaks`. (5) *image에 안 박힘* — BuildKit `--mount=type=secret`. (6) *fork PR에 secret 비공개* (`pull_request` vs `pull_request_target` 주의).

### Q4. Trunk-based development와 GitFlow 차이? 언제 무엇?
> *Trunk-based*: *main에 자주 머지*, 짧은 feature 브랜치, 항상 deployable, *feature flag*로 미완성 숨김. *GitFlow*: develop/release/hotfix 분기 많음, 길게 가는 feature 브랜치. → *현대 DevOps·웹 서비스*는 거의 trunk-based. *모바일·임베디드처럼 릴리즈 사이클 긴 제품*은 GitFlow.

### Q5. Test pyramid란? E2E만 많이 짜면 안 되는 이유?
> *Unit (많고 빠름) → Integration (중간) → E2E (적고 느림)* 피라미드 구조. E2E는 *비싸고 flaky*. 너무 많으면 *CI 시간 폭증 + 신뢰 하락*. 진짜 bug는 *unit이 잘 잡음* (작은 단위에서). E2E는 *주요 user journey 몇 개*만, *야간 batch* 또는 *PR 라벨 시*.

### Q6. flaky test를 어떻게 다뤄야 하나요?
> *retry로 숨기지 말 것* — 근본 원인을 안 잡으면 *영원히 팀 신뢰 갉아먹음*. (1) *flaky rate 측정*, (2) 발견 즉시 *quarantine* (CI에서 skip 또는 별도 group), (3) *fix backlog* 우선순위, (4) 근본 원인 (race condition, time-sensitive, external dependency 변동 등) 추적.

### Q7. *CI green인데 prod에서 깨졌다*. 왜 그럴 수 있나?
> CI는 *알려진 종류의 검증*만. *production env 차이* (data scale, traffic pattern, external service version, network 조건), *integration test에서 mock 사용*, *E2E 빠진 경로* 등에서 누락 가능. 그래서 *CD 단계의 verification* (canary, smoke test, observability)이 *CI 끝 = 안전*이 아님을 *전제*로 설계됨.

---

## 🔗 Cross-reference

- **GitHub Actions 실제 동작 + OIDC** → [02-github-actions.md](02-github-actions.md) (예정)
- **ArgoCD GitOps reconciliation** → [03-argocd.md](03-argocd.md) (예정)
- **Image build 깊이 (BuildKit cache·multi-stage)** → [`docker-basics/05`](../docker-basics/05-dockerfile-and-build.md)
- **Image scan + cosign + SBOM** → [`docker-basics/06`](../docker-basics/06-security.md), [05-build-and-registry.md](05-build-and-registry.md) (예정)
- **K8s admission에서 서명 검증** → [`kubernetes-basics/07`](../kubernetes-basics/07-rbac-and-security.md), `k8s-security-basics/` (예정)

---

## 📝 3줄 요약

1. CI = *빠른 피드백 + 통합 비용 분산 + 재현 가능한 산출물*. lint → test → SAST/SCA → build → push 표준 순서. 산출물은 *digest로 식별, mutable tag 금지*.
2. *secret 관리*가 가장 자주 빠짐. *OIDC 임시 자격증명*, *image에 안 박히기 (BuildKit secret)*, *fork PR 보호*.
3. *Trunk-based + branch protection + CI 게이트*가 현대 표준. *flaky test는 retry로 숨기지 말고 quarantine*. *CI green = 안전 아님* — CD verification 필수.

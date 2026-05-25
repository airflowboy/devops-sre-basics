# 05 — Dockerfile과 Build: 어떻게 이미지가 만들어지는가

> **공식 근거:** [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [BuildKit docs](https://docs.docker.com/build/buildkit/), [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
>
> **선행:** [01-image-and-layers.md](01-image-and-layers.md) (image/layer 개념)

---

## 🎯 한 문장

> **Dockerfile = *image를 만드는 명령의 순서적 레시피*. 각 *상태 변경 instruction이 layer 한 장*을 만들고, *같은 입력 → 같은 sha256 → cache hit*. 빌드 효율·이미지 크기·보안은 *모두 instruction 순서와 선택*에서 결정.**

---

## 1. Dockerfile 기본 instruction (사실상 다 외워야 함)

| Instruction | 역할 | Layer 생성? |
|---|---|:---:|
| `FROM` | 베이스 이미지 지정 (반드시 첫 줄, 또는 ARG 이후 첫 줄) | ✓ (베이스 layer들 그대로) |
| `RUN` | 빌드 시점에 *명령 실행* (`apt-get install` 등) | ✓ |
| `COPY` | 빌드 컨텍스트의 파일/디렉터리를 image에 복사 | ✓ |
| `ADD` | COPY + URL 다운로드 + tar 자동 압축 해제 | ✓ |
| `WORKDIR` | 이후 명령의 *작업 디렉터리* 설정 (없으면 `mkdir -p` + `cd`) | ✓ |
| `ENV` | 빌드·실행 시 환경 변수 설정 | ✓ |
| `ARG` | *빌드 시점*만 사용되는 변수 (이미지에 남지 않음) | (메타) |
| `USER` | 이후 명령 + 컨테이너 실행 시 *사용자* (uid/gid) | (메타) |
| `EXPOSE` | *문서화* — 어떤 포트를 쓸 거라고 표시. 실제 publish X | (메타) |
| `VOLUME` | 해당 경로를 *volume으로 선언* (anonymous volume 자동 생성) | (메타) |
| `ENTRYPOINT` | 컨테이너 시작 시 *항상 실행*되는 명령 | (메타) |
| `CMD` | ENTRYPOINT의 *기본 인자* (또는 ENTRYPOINT 없으면 그 자체가 실행 명령) | (메타) |
| `HEALTHCHECK` | 컨테이너 health check 명령 정의 | (메타) |
| `LABEL` | 메타데이터 (key=value) | (메타) |

→ **(메타)** 표시는 *layer는 안 만들지만 image config에 기록*. 거의 비용 없음.

---

## 2. *각 instruction이 layer를 어떻게 만드는가*

### RUN의 동작
```dockerfile
RUN apt-get update && apt-get install -y nginx
```
실제로 일어나는 일:
1. 이전 layer를 *임시 컨테이너로 띄움*
2. 그 안에서 `apt-get update && apt-get install -y nginx` 실행
3. 컨테이너의 *변경분*을 *새 layer*로 commit
4. 컨테이너 제거

### COPY/ADD
1. 빌드 컨텍스트의 파일을 *새 layer*로 *그대로* 추가
2. layer의 sha256은 *파일 내용의 해시*에 의존

### ENV/USER/WORKDIR/ENTRYPOINT/CMD/...
1. *layer는 안 만듦*. 그냥 image config JSON에 기록.

---

## 3. Layer 캐시 — *Dockerfile 순서가 빌드 시간을 결정*

`docker build` 동작:
1. 각 instruction마다 *입력의 해시*를 계산
2. 이전 빌드 캐시에 *같은 해시가 있으면 → 그 layer 재사용*
3. *없으면 → 실행 후 새 layer*. *그 이후 모든 instruction*도 *cache miss* (캐시 무효화).

### 핵심 원칙: *덜 자주 바뀌는 걸 위에, 자주 바뀌는 걸 아래*

❌ 나쁜 예:
```dockerfile
FROM node:20-alpine
COPY . /app/                # 코드 1줄만 바꿔도 캐시 깨짐
WORKDIR /app
RUN npm install             # ↑ 깨졌으니 매번 재실행 (느림)
CMD ["node", "index.js"]
```

✅ 좋은 예:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json /app/   # 의존성 정의만
RUN npm ci --omit=dev                        # 의존성 안 바뀌면 캐시 hit
COPY . /app/                                  # ← 코드는 마지막에
CMD ["node", "index.js"]
```

→ 빌드 시간 *수십 배* 차이. 운영 CI/CD에서 *가장 자주 보는 최적화*.

### `RUN`의 *같은 줄* 원칙
```dockerfile
# ❌ 각 RUN이 별도 layer → apt 캐시·임시파일이 image에 남음
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ 한 RUN으로 묶어 *최종 layer엔 정리된 결과만*
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

이유: layer는 *변경분*만 저장 — 한 layer에서 만든 파일을 *다음 layer에서 지워도* 원본 layer의 파일은 남음 ([01 참조](01-image-and-layers.md)).

---

## 4. Multi-stage Build — *빌드 의존성을 최종 image에서 분리*

**문제:** Go·Java·TypeScript는 *빌드 도구*가 *런타임보다 훨씬 큼*. 빌드 도구를 최종 image에 안 남기고 싶음.

**해결:** 빌드용 stage 와 런타임 stage 를 *Dockerfile 한 파일 안에 두 개*. 마지막 stage가 *결과 image*.

### Go 예시
```dockerfile
# === Stage 1: 빌드 ===
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

# === Stage 2: 런타임 ===
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

결과:
- 빌드 stage: ~1GB (Go 컴파일러 + 의존성)
- 최종 image: ~10MB (distroless + 정적 바이너리)
- → *100배 절감*

### Java 예시 (Maven + Jib 없이 수동)
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B package -DskipTests

FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/*.jar /app.jar
USER 1000
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Multi-stage의 더 강력한 활용
- 여러 stage에서 *각자 다른 베이스*
- `--target` 옵션으로 *중간 stage만* 빌드 가능 (CI에서 *테스트 stage만* 별도로)
- 같은 Dockerfile로 *dev 이미지*와 *prod 이미지* 동시 관리

---

## 5. 이미지 크기 최적화 — 4가지 베이스 비교

| Base image | 크기 | 특징 | 언제 |
|---|---|---|---|
| `ubuntu:22.04` | ~80MB | 완전한 OS, apt 사용 가능, 디버깅 편함 | 익숙함이 우선일 때, prod 가급적 회피 |
| `alpine:3.19` | ~5MB | musl libc 기반 *극소* OS, apk 사용 | size 우선, *단 glibc 의존 앱 주의* (호환성 문제 발생 가능) |
| `gcr.io/distroless/static` | ~2MB | OS 패키지 매니저·셸 없음. *정적 바이너리만* 실행. | Go·Rust 정적 바이너리. 보안·size 둘 다 최강 |
| `scratch` | 0MB | *완전히 빈* image. 자기가 다 채워야 함. | Go static binary 같은 *완전 self-contained* 앱 |

### Alpine의 함정 — *musl libc*
- Alpine은 glibc 대신 *musl libc* 사용. 작지만 *호환성 문제* 종종.
- Node.js의 *prebuilt native module*이 alpine에서 안 돌 수 있음 → 컴파일 필요 → 느려짐
- Java도 *Alpine 전용 JDK*가 별도 필요 (`-alpine` 태그)
- Python의 일부 wheel (numpy 등)도 alpine에서 *소스 컴파일* 필요
- → *alpine은 면접에선 안전한 답*이지만 *실전에선 검증 필요*

### Distroless의 함정 — *디버깅 어려움*
- 셸 없음 → `kubectl exec` 로 들어가도 *명령 실행 불가*
- → debug용으로 `:debug` 태그 별도 (`gcr.io/distroless/static:debug` — busybox 포함)
- 또는 *ephemeral container* (K8s 1.25+) 로 디버깅용 image 임시 attach

---

## 6. `.dockerignore` — *빌드 컨텍스트* 최소화

빌드 시 docker는 *Dockerfile이 있는 디렉터리 전체*를 daemon에 전송 (= "빌드 컨텍스트"). 큰 디렉터리면 *전송 시간 + 빌드 캐시 깨짐* 발생.

```
# .dockerignore 예시
.git
.gitignore
node_modules
dist/
*.log
.env
.env.*
.vscode/
.idea/
docker-compose.override.yml
README.md
```

효과:
- 전송 시간 단축
- COPY 시 *불필요 파일이 image에 안 들어감*
- *비밀 파일이 image에 박히는 사고 방지* (`.env`, `*.key` 등)

---

## 7. BuildKit — *현대 빌드 엔진*

도커는 2018년부터 *BuildKit* 이라는 새 빌드 엔진 도입. 현재는 *기본*.

### Legacy builder vs BuildKit

| 항목 | Legacy | BuildKit |
|---|---|---|
| 빌드 순서 | 순차 | *DAG 기반 병렬* (의존성 없는 단계 동시 실행) |
| 캐시 | layer 단위 | layer + *파일 단위 캐시* (RUN 안에서도) |
| 출력 | 항상 image | image / tar / 로컬 디렉터리 / registry 직접 push |
| Secret | env로 noop | *mount 시점만 노출* (image에 안 박힘) |
| SSH | 안 됨 | private repo clone 시 호스트 SSH agent 활용 |
| 진단 | 부족 | 컬러풀한 진행 상황 + spans |

활성화:
```bash
export DOCKER_BUILDKIT=1
docker build .
# 또는
docker buildx build .  # 최신 명령 (BuildKit 명시)
```

### BuildKit Secret — *비밀 안 박히게 빌드*
```dockerfile
# syntax=docker/dockerfile:1
FROM alpine
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret  # 빌드 시점에만 보임, layer에 안 박힘
```
```bash
docker buildx build --secret id=mysecret,src=./secret.txt .
```

→ 옛날엔 `ARG SECRET=xxx` 로 비밀 넘기다가 *image layer에 그대로 남는 사고* 빈발. BuildKit secret로 해결.

### BuildKit SSH — *private repo clone*
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git
```
```bash
docker buildx build --ssh default .
```

---

## 8. Build Args vs Environment Variables

```dockerfile
ARG VERSION=1.0       # 빌드 시점 변수
ENV APP_VERSION=$VERSION   # 런타임 환경 변수
```

| | ARG | ENV |
|---|---|---|
| 언제 | 빌드 시점 | 빌드 + 런타임 |
| Image에 남나? | *값은 안 남음* (메타데이터에 *키만* 기록) | 남음 |
| 컨테이너 안에서 보이나? | ❌ | ✅ |
| 빌드 명령으로 override | `--build-arg VERSION=2.0` | `-e APP_VERSION=2.0` (run 시) |

⚠️ *비밀번호·토큰을 ARG로 넘기지 말 것* — *image history에 남음* (`docker history` 로 보임). BuildKit secret 사용.

---

## 9. ENTRYPOINT vs CMD — 헷갈리는 2가지

| | ENTRYPOINT | CMD |
|---|---|---|
| 역할 | *항상 실행*되는 명령 | ENTRYPOINT의 *기본 인자* (또는 ENTRYPOINT 없으면 그 자체) |
| Override | `--entrypoint` (`docker run` 옵션) | `docker run image arg1 arg2` (마지막 인자들) |
| 권장 형식 | *exec form* (JSON 배열) | exec form |

### Exec form vs Shell form
```dockerfile
# Exec form (권장)
ENTRYPOINT ["nginx", "-g", "daemon off;"]

# Shell form
ENTRYPOINT nginx -g 'daemon off;'
```

- *Shell form* → 내부적으로 `/bin/sh -c "nginx ..."` 로 실행됨 → *bash가 PID 1*, 시그널 전파 X ([02 참조](02-namespace-and-cgroup.md))
- *Exec form* → nginx가 *직접 PID 1*, 시그널 정상

→ 면접 빈출: *"왜 exec form 권장?"* — PID 1 + 시그널 답.

### ENTRYPOINT + CMD 조합 패턴
```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```
- 기본 실행: `python app.py --port 8080`
- override: `docker run myapp --port 9090` → `python app.py --port 9090`
- 즉 *기본 인자만 바꿀 수 있게* 함

---

## 10. 흔한 함정

### 10-1. `WORKDIR` 없이 상대 경로
```dockerfile
RUN ./script.sh   # 어디서 실행? 보통 / 또는 직전 디렉터리. 의도 명확히 X
```
- *항상 WORKDIR* 먼저 명시. 의존성 줄임.

### 10-2. `COPY . /app/` 의 함정
- *모든 파일* 복사. `.dockerignore` 없으면 `.git`·`node_modules`·민감 파일 다 들어감
- → 항상 `.dockerignore` 먼저 작성

### 10-3. 임시 컨테이너의 결과를 영구화하려 함
- `RUN cd /app && python -m venv venv` 다음 `RUN ./venv/bin/python ...`
- *RUN마다 별도 컨테이너* → `cd`로 바꾼 디렉터리는 다음 RUN에 영향 X
- → `WORKDIR /app` 사용

### 10-4. `latest` 베이스 이미지
- `FROM node` (== `FROM node:latest`) → 빌드 재현성 깨짐. 어제와 오늘 다를 수 있음.
- → *항상 명시 버전*. `FROM node:20-alpine`, 더 엄격하면 digest pin.

### 10-5. `RUN curl ... | bash` (script 다운+실행)
- 보안: *다운로드된 스크립트의 sha 검증 안 함*
- → 최소한 sha 확인: `curl -O ... && echo "{sha} -" | sha256sum -c && bash ...`
- 또는 *공식 패키지 매니저* 사용

---

## 🎤 면접 빈출 Q&A

### Q1. Dockerfile의 layer caching 원리?
> 각 instruction의 *입력*에서 *해시* 계산. 캐시에 같은 해시가 있으면 *해당 layer 재사용*. *한 instruction이 cache miss면 그 이후 모든 instruction*도 cache miss. 그래서 *덜 자주 바뀌는 instruction을 위에, 자주 바뀌는 걸 아래에*. 의존성 설치 → 코드 복사 순서가 정석.

### Q2. 이미지 크기를 줄이는 방법 3가지 이상?
> (1) *작은 베이스* — alpine / distroless / scratch. (2) *multi-stage build* — 빌드 의존성을 최종 image에 안 포함. (3) *RUN 한 줄에서 설치+청소* — 같은 layer에서 임시파일 제거 (`apt-get install ... && rm -rf /var/lib/apt/lists/*`). (4) `.dockerignore` 로 불필요 파일 제외. (5) `--no-install-recommends` (apt), `--virtual` (apk).

### Q3. Multi-stage build를 왜 쓰나요?
> *빌드 의존성*(컴파일러·SDK 등)을 *최종 image에 안 남기기* 위해. 결과: 이미지 *수십~수백 배 작음*, *공격 표면 축소*, *런타임에 불필요한 도구 X*. Go·Rust 정적 바이너리 + distroless 조합이 사실상 표준.

### Q4. ENTRYPOINT와 CMD의 차이? Exec form을 왜 권장?
> ENTRYPOINT는 *항상 실행*되는 명령, CMD는 그 *기본 인자*. `docker run image arg1` 하면 *arg1이 CMD를 override*. *Exec form* (JSON 배열) 은 명령을 *직접 PID 1로 실행* → 시그널 정상 전파. *Shell form*은 `/bin/sh -c` 거쳐서 bash가 PID 1 → SIGTERM 전파 안 됨 → graceful shutdown 깨짐.

### Q5. BuildKit이 legacy builder 대비 좋은 점?
> (1) *DAG 기반 병렬 빌드* — 의존성 없는 단계 동시 실행. (2) *파일 단위 캐시*. (3) *secret mount* — 비밀이 image layer에 안 박힘. (4) *SSH agent forwarding* — private repo clone 안전하게. (5) *다양한 output* — image 외에 tar/디렉터리/registry 직접 push.

### Q6. ARG로 비밀번호 넘기면 안 되는 이유?
> *Image history*에 키와 값이 *그대로 남음*. `docker history --no-trunc {image}` 로 누구나 볼 수 있음. → BuildKit `--mount=type=secret` 사용. 빌드 시점에만 마운트되고 *layer에 박히지 않음*.

---

## 🔗 Cross-reference

- **Layer / image** → [01-image-and-layers.md](01-image-and-layers.md)
- **PID 1 / exec form 의 graceful shutdown** → [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md)
- **CI에서 빌드 캐시 활용** (GitHub Actions cache, registry cache) → `cicd-gitops-basics/` (예정)
- **Image signing / SBOM** (cosign, syft) → [06-security.md](06-security.md)
- **Helm/K8s에서 image 명세** → `helm-basics/`, `kubernetes-basics/` (예정)

---

## 📝 3줄 요약

1. *상태 변경 instruction = layer 한 장*. layer 캐시는 *같은 입력 → 같은 sha256*. instruction 순서가 빌드 시간을 결정.
2. *Multi-stage build + 작은 베이스 image (distroless·alpine·scratch)* 가 사이즈·보안 최적화의 정석.
3. ENTRYPOINT는 *exec form*으로 PID 1 시그널 보장. ARG로 비밀 넘기지 말고 BuildKit secret 사용.

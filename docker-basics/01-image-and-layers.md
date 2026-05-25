# 01 — Image와 Layer: 어떻게 저장되고 전송되는가

> **공식 spec 근거:** [OCI Image Format Specification v1.1](https://github.com/opencontainers/image-spec/blob/main/spec.md), [Docker Engine docs — About storage drivers](https://docs.docker.com/engine/storage/drivers/)

---

## 🎯 한 문장

> **이미지는 *코드+환경*의 *불변 스냅샷*이며, *manifest + config + 여러 read-only layer*의 묶음이다. 각 layer는 content-addressable(sha256 해시로 식별)이고, 변경되지 않은 layer는 *재사용·재전송 안 함*.**

---

## 1. OCI Image Spec — 이미지는 *3종 파일*의 묶음

Docker image는 사실 *OCI Image Spec*을 따르는 *표준 형식*이다. Docker만의 것이 아니라 containerd·CRI-O·Podman 모두 같은 spec을 따른다.

이미지 한 개는 *논리적으로* 다음 3종 파일의 묶음:

```
┌─────────────────────────────────────────────────┐
│  Manifest (JSON)                                 │  ← "이 이미지의 목차"
│  - config digest (sha256:...)                   │
│  - layer digests (여러 개, sha256:... 각각)      │
├─────────────────────────────────────────────────┤
│  Config (JSON)                                   │  ← "이 이미지의 메타데이터"
│  - 환경변수, CMD, ENTRYPOINT, USER, ...         │
│  - rootfs.diff_ids (layer 순서)                  │
│  - history (각 layer가 어떤 명령으로 만들어졌나) │
├─────────────────────────────────────────────────┤
│  Layers (tar.gz, 여러 개)                        │  ← "실제 파일들의 변경분"
│  - Layer 1: base OS의 /usr, /lib, ...            │
│  - Layer 2: apt install nginx 의 변경분          │
│  - Layer 3: COPY ./app /app 의 변경분            │
│  ...                                             │
└─────────────────────────────────────────────────┘
```

**눈으로 확인:** registry에서 manifest를 직접 받아볼 수 있음.
```bash
# nginx:latest 이미지의 manifest
curl -s -H 'Accept: application/vnd.oci.image.manifest.v1+json' \
  https://registry-1.docker.io/v2/library/nginx/manifests/latest
```

응답 예시 (요약):
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "digest": "sha256:abc123...",
    "size": 7634
  },
  "layers": [
    { "digest": "sha256:def456...", "size": 28567920 },
    { "digest": "sha256:ghi789...", "size": 24561 },
    ...
  ]
}
```

→ **결국 이미지 = 작은 JSON 두 개 + 여러 tarball**.

---

## 2. Layer — *변경분*을 따로 저장하는 이유

### 핵심 발상
- 컨테이너 이미지는 *대부분 OS 부분이 같다* (예: 모두 ubuntu:22.04 베이스)
- → *변경된 부분만* 따로 저장하면 *압도적으로 효율적*

### 어떻게 동작하나
- 각 layer = *이전 상태에서 추가/변경/삭제된 파일들의 tar*
- *삭제*는 특수한 *whiteout* 파일로 표현 (`.wh.filename`)
- 실행 시점에 **OverlayFS** 가 *모든 layer를 위에서 아래로* 합쳐 보여줌 → ([04 참조](04-volumes-and-storage.md))

### Dockerfile → Layer 매핑
대부분의 *상태 변경* instruction이 layer 하나를 만든다:
```dockerfile
FROM ubuntu:22.04         # 베이스 이미지의 모든 layer를 그대로 받음
RUN apt-get update        # ← 새 layer 1개
RUN apt-get install nginx # ← 새 layer 1개
COPY . /app               # ← 새 layer 1개
CMD ["nginx", "-g", "daemon off;"]  # ← 메타데이터 (config에만 반영, layer 없음)
```

`docker history nginx:latest` 로 layer 목록 확인 가능.

### Layer 캐시
- *같은 instruction*을 *같은 입력*으로 실행 → *기존 layer 재사용* (sha256 같음)
- 한 instruction이 변경되면 → *그 이후 모든 instruction* 재실행 (cache invalidation)
- 그래서 **자주 안 바뀌는 명령을 위에, 자주 바뀌는 걸 아래**:
```dockerfile
COPY package.json /app/  # ← 의존성 정의만, 안 자주 바뀜
RUN npm install           # ← 위 layer가 같으면 캐시 hit
COPY . /app/              # ← 소스 코드, 자주 바뀜
```
잘못된 순서:
```dockerfile
COPY . /app/      # ← 코드 한 줄 바꿔도 캐시 miss
RUN npm install   # ← 매번 재실행
```

[`05-dockerfile-and-build.md`](05-dockerfile-and-build.md) 에서 더 깊이.

---

## 3. Content-addressable Storage — *해시가 곧 식별자*

- 모든 layer·config는 *그 내용의 sha256 해시*로 식별됨
- 같은 내용 → 같은 해시 → 한 번만 저장·전송
- 내용이 1바이트라도 바뀌면 → 완전히 다른 해시
- 위변조 *불가능* (다른 내용에 같은 해시 만들 수 없음)

### 실용적 함의
- **Image tag는 사람용 alias**, *진짜 식별자*는 digest (`@sha256:abc...`)
- Tag는 바뀔 수 있음 (`nginx:latest` 가 어제와 오늘 다를 수 있음)
- 운영에서 *재현성*을 원하면 **digest로 pin**:
  ```yaml
  image: nginx@sha256:abc123...   # 절대 안 바뀜
  # vs
  image: nginx:latest             # 언제 바뀔지 모름
  ```

---

## 4. `docker pull` 의 실제 동작

```
client                              registry
  │                                    │
  │ ── GET /v2/library/nginx/manifests/latest ─────────►│
  │ ◄── manifest (JSON) ───────────────────────────────│
  │                                    │
  │ (manifest 파싱 → 필요한 layer digest 목록 추출)    │
  │                                    │
  │ ── (이미 있는 layer는 skip)        │
  │                                    │
  │ ── GET /v2/library/nginx/blobs/sha256:def456 ─────►│
  │ ◄── layer1 (tar.gz) ───────────────────────────────│
  │ ── GET /v2/library/nginx/blobs/sha256:ghi789 ─────►│
  │ ◄── layer2 (tar.gz) ───────────────────────────────│
  │ ... (병렬로)                       │
  │                                    │
  │ (로컬에 저장 + OverlayFS 마운트 준비) │
```

**핵심 관찰:**
- 모든 통신은 *HTTPS + 표준 REST*. 특별한 프로토콜 X.
- *각 layer는 독립적으로 cacheable*. 같은 베이스 이미지 쓰는 다른 이미지 pull 시 *재다운로드 안 함*.
- registry는 *blob store + manifest store* 의 두 가지만 안다 (단순함).

---

## 5. Registry — 결국 *HTTP API 서버*

OCI Distribution Spec 이 registry API의 표준. ECR·GHCR·Docker Hub·Harbor 모두 같은 API.

### 주요 endpoint
| HTTP | Path | 용도 |
|---|---|---|
| `GET` | `/v2/` | API 버전 체크 |
| `GET` | `/v2/{name}/manifests/{tag-or-digest}` | manifest 다운로드 |
| `PUT` | `/v2/{name}/manifests/{tag}` | manifest 업로드 (push) |
| `GET` | `/v2/{name}/blobs/{digest}` | layer 다운로드 |
| `HEAD` | `/v2/{name}/blobs/{digest}` | layer 존재 확인 (있으면 push skip) |
| `POST` + `PUT` | `/v2/{name}/blobs/uploads/` | layer 업로드 (multipart) |

### 인증
- 보통 *Bearer token* (registry가 직접 발급 or 외부 IDP 위임)
- `docker login` = 자격증명 받아 `~/.docker/config.json` 에 저장
- 클라우드 registry는 *단기 토큰* 권장 (AWS ECR `get-login-password`, GHCR with PAT, ...)

→ 면접 빈출: *"registry와 어떻게 인증하나요"* — 위 흐름 답.

---

## 6. Docker만의 것이 아니다 — Containerd / CRI-O

OCI Image Spec 덕분에 **여러 런타임이 같은 이미지를 호환**:

| 런타임 | 위치 | 비고 |
|---|---|---|
| **dockerd** | 데스크탑·구식 K8s 노드 | containerd 위에 build/CLI 얹은 형태 (실제로 dockerd → containerd 내부 위임) |
| **containerd** | 현대 K8s 노드 표준 | CNCF graduated, OCI 런타임 (`runc` 사용) |
| **CRI-O** | 일부 K8s 배포판 | K8s CRI 전용으로 더 lightweight |
| **Podman** | rootless 강조, RHEL 진영 | daemon-less |

**현대 K8s (1.20+) 에서 `dockerd`는 *deprecated*.** 이유: K8s는 *CRI(Container Runtime Interface)* 만 사용하면 됨. dockerd 거치는 중간 단계가 불필요. → containerd 직접 사용으로 전환.

→ 면접 빈출: *"K8s가 docker를 빼겠다는 게 무슨 의미인가"* — dockerd shim 제거지, *이미지 호환성*은 유지 (OCI 표준이라).

---

## 7. 자주 헷갈리는 것

### 7-1. *이미지 크기*는 *압축된 layer 합*이지 *해제 후 크기*가 아님
- `docker images` 의 SIZE = *해제 후 디스크 사용량*
- registry에 올라간 크기 = *압축된 합* (보통 절반 이하)
- → 이미지 *다운로드 속도* 와 *실행 디스크 사용량* 이 다름

### 7-2. `latest` 태그는 *최신*이라는 뜻이 *아님*
- 단순히 *태그를 안 명시했을 때의 기본값* (도커 컨벤션)
- 누군가 `latest`를 *3년 전 빌드*로 push하면 그게 `latest`
- 운영에선 *명시적 버전 태그* 또는 *digest* 사용 권장

### 7-3. `docker pull` 후에도 *디스크에 layer 중복* 가능?
- 일반적으론 X — content-addressable이라 같은 layer는 한 번만 저장
- 단, *storage driver*에 따라 차이 (overlay2가 표준)
- `docker system df -v` 로 layer 별 사용량·중복 확인

### 7-4. *layer를 너무 많이 만들면* 안 좋은가?
- 옛날엔 *layer 수 제한* (128) 있었음. 지금은 사실상 무제한.
- 단, *layer 많을수록 metadata 오버헤드* 약간 증가
- 더 중요한 건 *각 layer의 효율성* (불필요한 파일 안 남기기, [05 참조](05-dockerfile-and-build.md))

### 7-5. `RUN rm -rf /tmp/*` 로 *이미지가 작아지지 않는다*
- 각 RUN은 *별도 layer*. 삭제는 *whiteout* 으로 표시되지만 *원본 layer는 남음*
- 진짜로 줄이려면 *같은 RUN 안에서* 설치+사용+삭제:
  ```dockerfile
  RUN curl -O https://... && \
      tar xzf ...        && \
      make install        && \
      rm -rf /tmp/source
  ```
- 또는 multi-stage build로 *최종 stage에 필요한 것만 복사* (가장 깔끔)

---

## 8. Image와 Container의 관계 (다시 정리)

```
                    ┌────────────────────┐
                    │  Image (불변)       │
                    │  Layer1 (read-only) │
                    │  Layer2 (read-only) │
                    │  Layer3 (read-only) │
                    └──────────┬─────────┘
                               │ (한 이미지에서 여러 컨테이너 띄울 수 있음)
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌─────────┐      ┌─────────┐      ┌─────────┐
        │Cont. A  │      │Cont. B  │      │Cont. C  │
        │ writable│      │ writable│      │ writable│
        │ layer   │      │ layer   │      │ layer   │
        └─────────┘      └─────────┘      └─────────┘
        (각자 독립된 writable layer가 image 위에 마운트됨)
```

- 컨테이너 *생성* = OverlayFS 마운트 + writable layer 생성 + namespace/cgroup 셋업 → ([02 참조](02-namespace-and-cgroup.md))
- 컨테이너 *삭제* = writable layer 제거 (image는 그대로)
- → **데이터 보존하려면 volume** ([04 참조](04-volumes-and-storage.md))

---

## 🎤 면접 빈출 Q&A

### Q1. Image와 Container의 차이는?
> Image는 *불변의 스냅샷*이고, Container는 *그 image를 실행한 인스턴스*다. Image = 클래스, Container = 인스턴스 관계. 한 image로 컨테이너 여러 개 띄울 수 있고, 각각 독립된 writable layer를 가진다. 컨테이너가 죽어도 image는 유지된다.

### Q2. 이미지 layer caching은 어떻게 동작하나요?
> 각 layer는 *내용의 sha256 해시*로 식별된다(content-addressable). Dockerfile의 각 instruction이 layer 하나를 만들고, *입력이 같으면 같은 해시* → 기존 layer 재사용. 단, *한 instruction이 바뀌면 그 이후 모든 instruction*은 cache miss. 그래서 자주 안 바뀌는 명령(예: `COPY package.json`)을 위에, 자주 바뀌는 명령(`COPY . .`)을 아래에 두는 게 핵심.

### Q3. Tag vs Digest의 차이는?
> Tag(`nginx:latest`)는 *사람용 alias*. 누가 어느 이미지를 *그 태그로 push*했는지에 따라 가리키는 대상이 바뀔 수 있다. Digest(`nginx@sha256:abc...`)는 *내용의 해시*. 절대 안 바뀐다. 운영에선 *재현성과 보안* 위해 digest pin 권장.

### Q4. Image 크기를 줄이는 방법 3가지?
> (1) **베이스 이미지를 작은 걸로** — `alpine`, `distroless`, `scratch`. (2) **multi-stage build** — 빌드 의존성을 최종 이미지에서 분리. (3) **layer 효율** — 같은 RUN에서 설치+청소, `--no-install-recommends`, `apt-get clean`, `.dockerignore` 활용.

### Q5. `docker pull` 시 layer는 어떻게 받아오나요?
> registry에 *manifest* 먼저 요청 → 거기서 layer digest 목록 추출 → 각 layer를 *digest로 GET* (이미 있는 건 skip). 모두 *HTTPS + REST*. content-addressable이라 *같은 layer는 한 번만* 받음.

### Q6. K8s가 "docker 빼겠다"는 게 무슨 뜻인가요?
> dockerd라는 *중간 daemon*을 K8s가 더 이상 사용하지 않겠다는 뜻. K8s는 *CRI*만 필요하고 containerd가 그걸 직접 제공함. **이미지 호환성은 그대로** — OCI 표준이라 docker로 빌드한 이미지가 containerd에서 그대로 실행됨. 개발자 입장에선 *거의 변화 없음*.

---

## 🔗 Cross-reference

- 이미지는 *어떻게 실행되는가* → [02-namespace-and-cgroup.md](02-namespace-and-cgroup.md)
- Layer가 *실제로 어떻게 합쳐지는가* (OverlayFS) → [04-volumes-and-storage.md](04-volumes-and-storage.md)
- Dockerfile 작성 best practice → [05-dockerfile-and-build.md](05-dockerfile-and-build.md)
- 이미지 *서명·SBOM·공급망 보안* → [06-security.md](06-security.md), `k8s-security-basics/` (예정)
- registry HTTP API → [네트워크 기초의 HTTP 부분](../network-basics/01-basics.md)

---

## 📝 3줄 요약

1. 이미지 = manifest + config + layer(들)의 묶음, 각 layer는 content-addressable (sha256).
2. Layer는 *변경분*만 저장 → 같은 베이스 공유 시 *재사용·재전송 안 함*. 캐시 동작이 Dockerfile 순서에 민감.
3. Image는 OCI 표준. Docker만의 것이 아님 — containerd·CRI-O·Podman 다 호환. K8s가 docker 빼는 건 *중간 daemon 제거*지 *이미지 호환성*은 그대로.

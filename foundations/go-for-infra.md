# Go for Infra — *언어 문서가 아닌, 생태계 분석*

> docker, k8s, helm, terraform, prometheus, argocd, etcd, istio, vault, cilium, fluxcd —
> 인프라 도구의 *압도적 다수* 가 Go로 쓰여 있다.
> 우연이 아니다. 이 문서는 "왜 Go인가" + "Go의 어떤 특성이 어떤 패턴을 만드는가" 의 분석.

> ⚠️ **이 문서는 Go 언어 tutorial이 아니다.** 문법은 [공식 tour](https://go.dev/tour) 로.
> 목표: 인프라 도구의 소스를 *읽고 이해할 수 있는* 수준 + Go가 *왜* 인프라에 맞는지 이해.

---

## 1. 왜 Go인가 — 다섯 가지 이유

### 1. Static binary

Go는 *모든 의존성을 한 바이너리에 정적 링크* (CGo 안 쓰면). 결과:

- 배포는 *바이너리 하나 복사*. runtime 설치 없음 (Java JRE, Python venv 따위).
- container 이미지가 `FROM scratch` 또는 `FROM gcr.io/distroless/static` 으로 작아진다.
- cross-compile 쉬움: `GOOS=linux GOARCH=arm64 go build` 한 줄.

### 2. Concurrency가 *언어 차원*

goroutine + channel이 *문법*. 별도 라이브러리·OS thread API 없이 *수만 개의 concurrent task* 가능.

k8s controller가 *동시에 수백 개의 리소스를 reconcile* 하는 패턴이 자연스럽게 표현됨.

### 3. GC + 합리적인 latency

Go GC는 *concurrent + low-pause* (1ms 이하 목표). 인프라 도구는 *대량 처리보다 응답성* 중요 → 잘 맞음.

비교: Java는 throughput 우수하지만 stop-the-world pause가 길 수 있음. Rust는 GC 없어서 더 빠르지만 *복잡성이 훨씬 높음*.

### 4. 표준 라이브러리가 *네트워크·HTTP·암호화에 강함*

`net/http`, `crypto/tls`, `encoding/json` 가 모두 표준. 외부 의존성 거의 없이 HTTP server·client·TLS·JSON API 가능.

→ k8s API client, prometheus exporter, terraform provider 모두 *외부 의존성 작은* 바이너리로 떨어진다.

### 5. 단순함 — *팀이 읽을 수 있는* 코드

generics 거의 없음 (1.18+ 추가됐지만 제한적), 매크로 없음, 상속 없음, 예외 없음. 코드가 *지루하지만 명확*. 인프라 도구처럼 *오래 유지보수* 해야 하는 코드베이스에 유리.

Rob Pike의 말: "Go is for big teams of average programmers."

---

## 2. Goroutine + Channel — Reconciler 패턴의 자연 언어

### Goroutine

```go
go doSomething()    // 새 goroutine에서 실행, fork 없이.
```

OS thread보다 훨씬 가벼움 (수 KB stack, growable). 한 프로세스에서 *수만 개* 가능. Go runtime이 OS thread에 multiplexing.

### Channel

```go
ch := make(chan int, 10)   // buffered channel
ch <- 42                    // 보내기
v := <-ch                   // 받기
```

goroutine 간 통신. Go의 명언: "**Don't communicate by sharing memory; share memory by communicating.**" 락 대신 채널.

### 이것이 k8s controller에 *왜* 맞는가

k8s controller-runtime의 작동:

```
1. Informer가 apiserver를 watch (goroutine 1)
   → event를 workqueue에 push
2. Worker goroutine N개가 workqueue에서 event를 pop (goroutine 2~N+1)
   → 각 worker가 Reconcile(key) 호출
3. Reconcile 안에서 또 client-go가 API 호출 (필요시 더 많은 goroutine)
```

매 단계가 *독립적 goroutine* + *channel/queue로 연결*. lock이 거의 없음. [[declarative-and-reconciliation]] §3의 control loop이 *자연스럽게* Go의 동시성 모델에 맞는다.

### `select` — 여러 channel 동시 대기

```go
for {
    select {
    case event := <-eventCh:
        handle(event)
    case <-ctx.Done():
        return       // 취소 시그널
    case <-time.After(30 * time.Second):
        // periodic resync
    }
}
```

이 패턴이 controller·proxy·load balancer 어디에나 나옴.

---

## 3. `context.Context` — *취소와 타임아웃의 공용어*

### 문제

Goroutine을 *어떻게 멈추는가*? Go는 thread interrupt 같은 게 없다 (의도적). 답: *명시적으로 신호를 주고받음*.

### Context

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

result, err := client.Get(ctx, "key")    // 5초 안에 안 끝나면 ctx.Done() 트리거
```

`context.Context` 는 다음을 전달:
- **취소 신호** (`ctx.Done()` 채널이 닫힘)
- **데드라인** (`ctx.Deadline()`)
- **값** (request-scoped data, 남용 주의)

### 인프라 도구 전체의 *관통하는 첫 번째 인자*

k8s client-go, gRPC, net/http, database/sql, prometheus client — 모두 *첫 인자가 `ctx context.Context`*.

```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

→ HTTP 요청이 들어오면 그 요청의 ctx가 *모든 하위 호출에 전파*. 클라이언트가 연결을 끊으면 ctx 취소 → 하위 호출 전부 자동 취소 → goroutine leak 방지.

이 *컨벤션이 일관* 한 게 Go 생태계의 큰 강점.

---

## 4. Interface + Struct Embedding — *Duck typing + 구성*

### Interface = *암묵적 구현*

```go
type Stringer interface {
    String() string
}

type User struct { Name string }
func (u User) String() string { return u.Name }
// User는 자동으로 Stringer를 만족. "implements" 키워드 없음.
```

→ *외부 라이브러리의 타입* 도 새 interface를 자동으로 만족 가능. 의존 방향이 *유연*.

### k8s에서의 활용

```go
type Reconciler interface {
    Reconcile(ctx context.Context, req Request) (Result, error)
}
```

내가 작성한 controller가 이 메서드만 가지면 *자동으로* k8s controller-runtime에 plug-in.

### Struct embedding — *상속이 아닌 구성*

```go
type Base struct { Name string }
func (b Base) Hello() { fmt.Println("hi", b.Name) }

type App struct {
    Base       // embedding
    Version string
}

a := App{Base: Base{Name: "x"}, Version: "1.0"}
a.Hello()    // Base의 메서드를 그대로 호출 가능
```

상속과 다른 점: *명시적 위임*. terraform provider·k8s API type 정의에서 자주 보임.

### 인터페이스 발견 패턴

코드 읽다가:

```go
func DoThing(w io.Writer, r io.Reader) error
```

→ `io.Writer`·`io.Reader` 가 standard interface. file, bytes.Buffer, net.Conn 모두 만족. *함수가 무엇을 받을 수 있는지가 의미적으로 명확*.

---

## 5. 에러 핸들링 — `if err != nil` 의 바다

```go
data, err := os.ReadFile("foo")
if err != nil {
    return fmt.Errorf("read foo: %w", err)
}
```

매 호출마다 반환값과 에러 분리. 예외 없음. 코드가 *시끄럽지만 명확*.

### `%w` — 에러 wrapping

`fmt.Errorf("... %w", err)` 가 에러를 *감싼다*. `errors.Is(err, fs.ErrNotExist)` 로 *원인 확인 가능*.

### sentinel error vs typed error

```go
// sentinel
var ErrNotFound = errors.New("not found")
if errors.Is(err, ErrNotFound) { ... }

// typed
type APIError struct { Code int; Msg string }
func (e *APIError) Error() string { return e.Msg }

var apiErr *APIError
if errors.As(err, &apiErr) { use(apiErr.Code) }
```

k8s client-go는 typed error 적극 사용 — `apierrors.IsNotFound(err)`, `IsAlreadyExists(err)`, `IsConflict(err)`. controller code 읽을 때 *이 함수들* 자주 본다.

---

## 6. 도구 — `pprof`, race detector, `go test`

### `pprof` — 운영 중인 인프라 도구의 *내부 들여다보기*

대부분의 Go 인프라 도구가 `/debug/pprof/` endpoint를 노출. 거기서:
- CPU profile
- memory profile (heap, allocs)
- goroutine stack dump
- block/mutex contention

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof -http=:8080 cpu.prof
```

prometheus·k8s controller·etcd가 OOM·hang 됐을 때 *현장 부검* 도구.

### Race detector

```bash
go test -race ./...
go run -race main.go
```

동시성 버그를 *실제 발생 시* 잡아준다. CI에 거의 필수.

### `go test`

- *별도 framework 없음*. 표준.
- `testing.T`·`testing.B`·`testing.F` (fuzz).
- table-driven test가 표준 패턴.

→ 인프라 도구 contributing guide의 "테스트는 어떻게" 가 *언어 표준에 맞춰져 있음* — 진입 장벽 낮춤.

---

## 7. 인프라 도구의 *공통 코드 패턴*

오픈소스 인프라 도구 코드를 처음 열 때 자주 보는 패턴들:

### `cobra` + `viper` — CLI

거의 모든 Go CLI (kubectl, helm, terraform, argocd, prometheus, vault…)가 cobra 사용. 명령 구조가 *비슷하게 생긴다*.

### `client-go` informer 패턴

```
SharedInformerFactory → Informer → Lister + EventHandler
                                            ↓
                                      workqueue → Reconcile
```

k8s controller 작성자가 *반복적으로 보는* 구조.

### `prometheus/client_golang` — metric 노출

```go
var requestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total"},
    []string{"method", "code"},
)
```

`/metrics` endpoint 한 줄로 추가. 그래서 *모든 Go 인프라 도구가 prometheus와 잘 맞는다*.

### `zap`·`logrus` — 구조화 로깅

stdlib `log` 는 빈약 → 대부분 zap/logrus 사용. k8s는 `klog`. structured logging이 *기본 가정*.

---

## 8. Cross-reference 지도

| 보고 싶은 것 | 가야 할 곳 |
|---|---|
| Controller pattern, informer, workqueue | `kubernetes-basics/` controller·CRD |
| Container의 static binary 활용 (scratch, distroless) | `docker-basics/` 이미지 |
| Helm template이 *Go template*인 이유 | [[yaml-and-templating]] §3 |
| Terraform provider SDK의 interface 구조 | `terraform-basics/` provider |
| Prometheus client·exporter | `observability-basics/` Prometheus |
| ArgoCD application controller | `cicd-gitops-basics/` ArgoCD |
| Goroutine·channel이 control loop에 맞는 이유 | [[declarative-and-reconciliation]] §3 |
| Context.Context와 cert·token 만료 처리 | [[tls-and-pki]], [[identity-and-auth]] |

---

## 한 줄 요약

> **Go는 인프라에 *우연히* 맞은 게 아니라 *설계상* 맞다.**
> Static binary·goroutine·context·interface — 네 가지가 control loop·분산 시스템·CLI 도구의 *공통 요구* 와 정확히 겹친다.
> Go를 *읽을 수 있게* 되는 게 인프라 도구를 *원리적으로* 이해하는 첫 단계.

---
name: golang-dependency-injection
description: Dependency injection in Go — why DI matters (testability, decoupling, lifecycle), manual constructor injection (the recommended default), functional options for optional dependencies, and a comparison of DI libraries (google/wire compile-time, uber-go/fx runtime + lifecycle, samber/do, uber-go/dig). Trigger this skill when designing service architecture, wiring dependencies, refactoring tightly-coupled code, choosing a DI library, or when the user mentions inversion of control, service containers, or wiring.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🔌"
allowed-tools: Read Edit Write Glob Grep Agent
---

# Dependency Injection in Go

DI is not a framework — it's a discipline. The discipline is: a component declares what it needs, and someone *outside* the component supplies it. In Go, that "someone" is usually a constructor at the composition root.

## Why DI matters

- **Testability.** A component that takes a `Clock` interface can be tested with a fake clock; one that calls `time.Now()` directly cannot.
- **Decoupling.** Business logic depends on `OrderRepository` (an interface), not on `*pgx.Conn`. Swapping Postgres for SQLite, or adding a caching layer, is a wiring change, not a rewrite.
- **Lifecycle management.** A constructed graph has a clear start order (built first, used later) and a clear teardown order (closed last, opened first). Globals make this implicit and bug-prone.
- **Single composition root.** A reader can find every concrete dependency in one place — usually `cmd/<app>/main.go`.

## Manual constructor injection — the recommended default

For most Go services, you do not need a DI library. You need:

1. Interfaces declared by the *consumer*.
2. Constructors that take those interfaces as parameters.
3. A composition root (`main.go`) that wires concrete adapters into constructors.

```go
// 1. Consumer declares the dependency it needs (small, purpose-fit).
package billing

type ClockPort interface {
    Now() time.Time
}

type Service struct {
    clock ClockPort
    repo  ChargeRepository
}

// 2. Constructor takes the dependency in.
func NewService(clock ClockPort, repo ChargeRepository) *Service {
    return &Service{clock: clock, repo: repo}
}
```

```go
// 3. Composition root wires concretes.
package main

func main() {
    db := openDB()
    defer db.Close()

    clock := realClock{}
    repo  := postgres.NewChargeRepository(db)
    svc   := billing.NewService(clock, repo)

    server := http.NewServer(svc)
    log.Fatal(server.ListenAndServe())
}
```

This is **the** Go-idiomatic pattern. It's fully type-checked, easy to read, easy to test (substitute fakes for `clock` and `repo`), and adds zero dependencies. Prefer it until the wiring graph genuinely outgrows it (typically 30+ services, multiple deployment shapes, complex startup/shutdown).

## Functional options for optional dependencies

When a constructor has many *optional* parameters (timeouts, loggers, metrics, custom transports), use functional options instead of a 12-parameter signature:

```go
type Option func(*Service)

func WithLogger(l *slog.Logger) Option { return func(s *Service) { s.log = l } }
func WithRetries(n int) Option         { return func(s *Service) { s.retries = n } }

func NewService(repo Repo, opts ...Option) *Service {
    s := &Service{repo: repo, log: slog.Default(), retries: 3}
    for _, opt := range opts { opt(s) }

    return s
}

// usage
svc := NewService(repo, WithRetries(5), WithLogger(myLog))
```

Mandatory dependencies stay positional; optional ones become `WithX` options.

## Pointer or value? Pointer.

Constructors return pointers (`*Service`) when the type has methods, holds state, or is large. The receiver matches: `func (s *Service) Charge(...)`. Mixing pointer and value receivers on the same type is a code smell.

## Anti-patterns to avoid

### 1. Package-level globals as dependencies

```go
// bad — implicit, untestable
var DB *sql.DB

func GetUser(id string) (User, error) { return DB.Query(...) }
```

The function depends on a global; tests cannot substitute a fake DB without coordinated mutation. Inject `db *sql.DB` (or a narrower interface) explicitly.

### 2. `init()` side effects

`init()` runs at import time, in nondeterministic order between packages, and cannot return errors. Use it only for *registering* values into compile-time tables (e.g. `image.RegisterFormat`); never for opening connections, reading files, or building services.

### 3. Service locator

```go
// bad — ServiceLocator hides dependencies behind a string lookup
locator.Get("UserRepo").(UserRepo).Find(id)
```

Hidden dependencies are the problem DI solves. Don't reintroduce them with a typed-but-untyped lookup.

### 4. Constructing dependencies inside constructors

```go
// bad — constructor reaches into infra
func NewService() *Service {
    db, _ := sql.Open("postgres", os.Getenv("DSN"))

    return &Service{db: db}
}
```

Now the constructor is impossible to test without a real Postgres. Take the `db` (or a `Repo` interface) as a parameter.

## DI libraries — when manual wiring runs out

Reach for a library when:

- The graph has 30+ nodes and `main.go` is a wall of `NewX(a, b, c)`.
- You need lifecycle hooks (`Start`/`Stop`) coordinated across many services.
- You need multiple compositions (HTTP server, CLI tool, worker, all sharing 80% of the graph).

### Comparison

| Library | Style | Strengths | Trade-offs |
|---|---|---|---|
| **`google/wire`** | Compile-time codegen | Zero runtime cost; errors at `go generate`; output is plain Go you can read; no reflection. | Codegen step in the build; small DSL to learn. |
| **`uber-go/fx`** | Runtime reflection + lifecycle | First-class `OnStart`/`OnStop` hooks; signal-aware `Run()`; modular `fx.Module`. | Reflection at startup; heavier than necessary for small graphs. |
| **`samber/do`** | Runtime container, generics-based | Type-safe via generics; small surface; supports scopes, health checks, graceful shutdown. | Younger ecosystem; runtime resolution. |
| **`uber-go/dig`** | Runtime reflection (the engine under fx) | Flexible primitive when you don't want fx's lifecycle. | Lower-level; you re-build what fx already provides. |

### When to pick which

- **Default: manual constructor injection** in `main.go`. Don't reach for a library unless you've felt actual pain.
- **`google/wire`** when you want compile-time guarantees and a codegen step is acceptable. Best for services where the graph is large but stable.
- **`uber-go/fx`** when you need lifecycle coordination (start/stop ordering across dozens of components) or are already in the Uber/Jaeger ecosystem.
- **`samber/do`** when you want a generics-based, lightweight runtime container without fx's extras.
- **`uber-go/dig`** rarely on its own; usually you want fx instead.

### `wire` — micro-example

```go
//go:build wireinject
// +build wireinject

package main

import "github.com/google/wire"

func InitializeServer() (*Server, error) {
    wire.Build(
        openDB,
        postgres.NewChargeRepository,
        wire.Bind(new(billing.ChargeRepository), new(*postgres.ChargeRepository)),
        billing.NewService,
        http.NewServer,
    )

    return nil, nil
}
```

Run `go generate` and wire emits a `wire_gen.go` file that contains the explicit constructor calls. You commit and ship that file.

### `fx` — micro-example

```go
app := fx.New(
    fx.Provide(
        openDB,
        postgres.NewChargeRepository,
        billing.NewService,
        http.NewServer,
    ),
    fx.Invoke(func(*http.Server) {}), // force Server construction
)
app.Run()
```

`fx.Lifecycle` lets components register `OnStart`/`OnStop` hooks; `fx.Run()` blocks on signals and tears down in reverse order.

## Lifecycle hygiene (any approach)

- **Construction order** is determined by the dependency graph (a service is built after its dependencies).
- **Teardown order** is the reverse — close the HTTP listener first, then drain in-flight work, then close the DB.
- **Defer in `main`** is fine for small programs; for many resources, prefer an explicit shutdown sequence keyed off `signal.NotifyContext` or fx's lifecycle.

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

srv := http.NewServer(svc)

go func() { _ = srv.ListenAndServe() }()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil { log.Error(err) }

_ = db.Close()
```

## How to apply (checklist)

1. Every dependency a component needs → declared as an interface on the *consumer* side.
2. Every constructor takes those interfaces as parameters; mandatory positional, optional via `WithX`.
3. Concrete types are wired only at the composition root (`main.go` or a dedicated wiring package).
4. No package-level globals for I/O (`db`, `httpClient`, `cache`).
5. No I/O in `init()`.
6. No constructor opens connections internally — accept the dep, don't fetch it.
7. Reach for `wire`/`fx` only when manual wiring genuinely hurts.

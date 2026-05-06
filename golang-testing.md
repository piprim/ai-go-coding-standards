---
name: golang-testing
description: Production-ready testing in Go — table-driven tests, subtests with t.Run, parallel tests with t.Parallel, testify (assert/require/suite), mocks (counterfeiter / mockery / hand-rolled), unit vs integration vs e2e, benchmarks with testing.B and benchstat, code coverage with go test -cover, fuzzing (Go 1.18+), test fixtures in testdata/, goroutine-leak detection with go.uber.org/goleak, snapshot/golden-file tests, CI with GitHub Actions, and idiomatic test naming. Trigger this skill on every test file edit, when writing new features (TDD), and when the user mentions tests, mocks, fixtures, coverage, fuzzing, or flaky tests.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🧪"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Agent
---

# Golang Testing

Tests are not optional and they're not slow. With table-driven tests, parallelism, and the standard tooling, a 10k-line Go service should run its full unit-test suite in under a few seconds.

## 1. Table-driven tests — the default shape

```go
func TestNormalizeEmail(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name string
        in   string
        want string
    }{
        {"already-lower",   "alice@example.com", "alice@example.com"},
        {"mixed-case",      "Alice@Example.COM", "alice@example.com"},
        {"surrounding-ws",  "  bob@x.io  ",      "bob@x.io"},
        {"empty",           "",                  ""},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()

            got := NormalizeEmail(tt.in)
            if got != tt.want {
                t.Errorf("NormalizeEmail(%q) = %q, want %q", tt.in, got, tt.want)
            }
        })
    }
}
```

Pattern:

- One slice of `tt` cases at the top, one `t.Run` loop at the bottom.
- Case names are *descriptive scenarios*, not `case_1`.
- Outer `t.Parallel()` lets this test run alongside others; inner `t.Parallel()` lets cases run alongside each other.

Since Go 1.22, the `tt` loop variable is per-iteration, so the old `tt := tt` shadow is no longer needed.

## 2. Subtests with `t.Run`

`t.Run` gives you:

- Targeted runs: `go test -run TestNormalizeEmail/mixed-case`
- Independent failure reports
- Nested setup/teardown via subtests

Use `t.Run` even for non-table tests when grouping related assertions improves output.

## 3. Parallel tests

`t.Parallel()` says "this test can run alongside other parallel tests in this package". Default it to **on** for unit tests; turn it **off** when the test mutates package-level state, environment variables, or working directory.

```go
func TestThing(t *testing.T) {
    t.Parallel()
    // ...
}
```

`go test -p N` controls package-level parallelism; `t.Parallel` controls test-level. Combine them and a 50-test package can finish in the time of its slowest test.

## 4. testify — `assert`, `require`, `suite`

The stdlib `t.Errorf` works but is verbose for non-trivial assertions. Many projects (this one included) use `github.com/stretchr/testify`:

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestExample(t *testing.T) {
    cfg, err := LoadConfig("testdata/valid.yaml")
    require.NoError(t, err)             // stops the test if err is non-nil
    require.NotNil(t, cfg)

    assert.Equal(t, "prod", cfg.Env)    // continues on failure, reports both
    assert.Len(t, cfg.Hosts, 3)
}
```

Rule of thumb:

- **`require`** for preconditions whose failure makes the rest of the test meaningless (`require.NoError`, `require.NotNil`).
- **`assert`** for independent checks where you'd rather see all failures at once.

`testify/suite` is fine when you genuinely need shared setup/teardown across many tests, but plain functions plus `t.Run` cover most cases more cleanly.

## 5. Mocks and fakes

### Hand-rolled fakes — the simplest path

If the dependency is an interface with a few methods, write a struct that records calls:

```go
type fakeRepo struct {
    saved []Order
    err   error
}

func (f *fakeRepo) Save(_ context.Context, o Order) error {
    if f.err != nil {
        return f.err
    }

    f.saved = append(f.saved, o)

    return nil
}
```

Cheap, no codegen, no magic. **Default to this.**

### Generated mocks — for larger interfaces

When an interface has 10+ methods or you need flexible expectation chains:

- **`counterfeiter`** — generates a `FakeXxx` struct with `XxxCallCount`, `XxxArgsForCall(i)`, `XxxReturns(...)`. Stable, simple API.
- **`mockery`** — generates testify-style mocks (`On(...).Return(...)`).
- **`gomock`** — Google's framework with strict expectation-order checks; verbose for casual use.

Pick one team-wide; don't mix.

## 6. Unit vs integration vs e2e

| Layer | What it tests | Speed | Dependencies |
|---|---|---|---|
| **Unit** | One function/struct in isolation | <50 ms each | None (fakes for ports) |
| **Integration** | An adapter against a real external system | seconds | One real dependency (DB, queue, HTTP server) |
| **End-to-end** | A full user flow through the binary | seconds-minutes | Multiple real systems |

Tag integration tests with a build tag so unit runs stay fast:

```go
//go:build integration

package billing_test
```

Run units in normal CI: `go test ./...`
Run integration on demand: `go test -tags=integration ./...`

## 7. Benchmarks with `testing.B`

```go
func BenchmarkParse(b *testing.B) {
    raw := []byte(`{"id":"abc","amount":1234}`)

    b.ResetTimer()                 // exclude setup
    b.ReportAllocs()               // also report allocations

    for range b.N {                // Go 1.22+ range-over-int form
        _, _ = Parse(raw)
    }
}
```

- **`b.ResetTimer()`** after expensive setup; otherwise setup is counted.
- **`b.ReportAllocs()`** for the alloc count — usually the more useful metric than ns/op.
- **Compare with `benchstat`**:

```bash
go test -bench=. -benchmem -count=10 ./... | tee old.txt
# make changes
go test -bench=. -benchmem -count=10 ./... | tee new.txt
benchstat old.txt new.txt
```

## 8. Coverage

```bash
go test -coverprofile=cover.out ./...
go tool cover -func=cover.out         # text summary
go tool cover -html=cover.out         # interactive HTML
```

Aim for 80%+ on application code. Don't aim for 100% — uncovered lines (defensive panics, init that won't fail) are sometimes correct. Use coverage as a *map of what isn't tested*, not as a target.

## 9. Fuzzing (Go 1.18+)

```go
func FuzzParse(f *testing.F) {
    f.Add([]byte(`{"id":"abc"}`))   // seed corpus
    f.Add([]byte(`{}`))

    f.Fuzz(func(t *testing.T, raw []byte) {
        _, err := Parse(raw)
        // assert: must not panic, must return either a value or an error
        if err == nil {
            // round-trip property
        }
    })
}
```

Run: `go test -fuzz=FuzzParse -fuzztime=30s`. Fuzz-found inputs land in `testdata/fuzz/FuzzParse/` and are run as regular cases on every subsequent `go test`.

Use fuzzing on parsers, decoders, validators, and anywhere user input crosses a trust boundary.

## 10. Test fixtures in `testdata/`

Go ignores any directory named `testdata` for build purposes. Put example inputs and expected outputs there:

```
billing/
  charge.go
  charge_test.go
  testdata/
    valid_request.json
    invalid_amount.json
```

```go
raw, err := os.ReadFile("testdata/valid_request.json")
require.NoError(t, err)
```

## 11. Golden-file (snapshot) tests

For functions whose output is large but deterministic (HTML render, generated code, marshaled JSON), compare to a checked-in golden file:

```go
func TestRender(t *testing.T) {
    got := Render(input)

    golden := filepath.Join("testdata", "render.golden")
    if *update {                                  // -update flag
        require.NoError(t, os.WriteFile(golden, got, 0o600))
        return
    }

    want, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.Equal(t, string(want), string(got))
}

var update = flag.Bool("update", false, "update golden files")
```

Update the goldens deliberately: `go test -update ./...`, review the diff in code review, commit.

## 12. Goroutine-leak detection — `go.uber.org/goleak`

A test that starts goroutines and leaves them running is a slow-growing leak in production. Verify with:

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

This fails the test run if any goroutines are still alive at the end. For a single test, use `defer goleak.VerifyNone(t)`.

## 13. Memory and race detection

- **Always run `-race` in CI.** It catches data races that can't be found by reading the code.
  ```bash
  go test -race ./...
  ```
- The race detector adds CPU and memory overhead — fine for CI, not for production.

## 14. CI with GitHub Actions

Minimal config that catches most issues:

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.x' }
      - run: go vet ./...
      - run: go test -race -coverprofile=cover.out ./...
      - run: go tool cover -func=cover.out
      - uses: golangci/golangci-lint-action@v6
      - run: go run golang.org/x/vuln/cmd/govulncheck@latest ./...
```

A matrix on `os: [ubuntu-latest, macos-latest]` is cheap insurance for cross-platform code.

## 15. Idiomatic naming

- **Test functions:** `TestXxx(t *testing.T)`. The `Xxx` matches the function or behavior under test.
- **Subtests:** describe the scenario, not the data — `"missing_required_field"`, `"empty_input"`, `"two_users_same_email"`.
- **Helper functions:** `setupX(t *testing.T)` or methods on a `harness` struct; call `t.Helper()` inside so failures point at the caller.
- **Test files:** `<file>_test.go` next to the file under test. **External test packages** (`package foo_test` in `foo_test.go`) for testing only the public API — recommended for library packages.

## How to apply (checklist)

1. New unit test → table-driven, `t.Parallel()` on outer and inner runs unless mutating shared state.
2. Test depends on a service → take an interface; pass a hand-rolled fake first, generated mock only if the interface is large.
3. Touching an external system → tag with `//go:build integration`; never run in the unit suite.
4. Performance-sensitive code → write a `Benchmark` and a benchstat baseline; track regressions.
5. Parser / decoder / validator → write a `Fuzz` with seeds.
6. Test data → in `testdata/` next to the test.
7. Large deterministic output → golden file + `-update` flag.
8. Code that spawns goroutines → `goleak` in `TestMain`.
9. CI → `go test -race`, `golangci-lint`, `govulncheck`, all gating.
10. Coverage → use as a map, not a target; review uncovered branches in PRs.

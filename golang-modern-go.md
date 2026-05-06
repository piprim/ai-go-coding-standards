---
name: golang-modern-go
description: Modern Go idioms and language features Claude must use when writing or refactoring Go code in this project — covers context-aware function variants, errors.AsType (Go 1.26+) over errors.As, errors.Join, the slices/maps/cmp standard packages, range over int (1.22+) and range over function (1.23+), min/max/clear builtins, any over interface{}, and other post-1.18 patterns. Trigger this skill on every Go edit and explicitly when the user mentions "modernize", "upgrade", "Go 1.x", or when older idioms are spotted in the file under edit.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🆕"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Agent
---

# Modern Go

Go evolves fast and pre-1.18 idioms now read as legacy code. This skill captures the post-1.18 patterns the project relies on. When you read code that predates these features, opportunistically modernize it — but don't sneak unrelated rewrites into a focused change.

## 1. Context-aware function variants

Always prefer the context-aware variant of a function whenever the standard library or a third-party package offers one — `http.NewRequestWithContext` over `http.NewRequest`, `sql.QueryContext` over `sql.Query`, `exec.CommandContext` over `exec.Command`, `(*sql.DB).PingContext` over `Ping`, etc.

**Why:** context propagation for cancellation and timeouts is the only way to cut off long-running operations and prevent goroutine/connection leaks when a request is aborted. A function without a `ctx` is a function the caller cannot interrupt.

**How to apply:** any time an API offers a `*Context` or `*WithContext` alternative, use it and pass the caller's `ctx` straight through. Never call `context.Background()` deep inside a request path — that severs the cancellation chain. The first parameter of any function that does I/O, blocks, or calls another `ctx`-taking function should itself be `ctx context.Context`.

```go
// wrong — no cancellation, command keeps running after caller times out
out, err := exec.Command("git", "status").Output()

// correct — caller's deadline cancels the subprocess
out, err := exec.CommandContext(ctx, "git", "status").Output()
```

## 2. errors.AsType (Go 1.26+) over errors.As

To extract an error of a specific type from an error chain, use the new generic `errors.AsType[T]` instead of the older `errors.As(err, &target)` pattern.

**Why:** `errors.AsType` returns the typed error directly (no out-pointer ceremony) and the type is part of the call, not implied by an unrelated variable declaration above. It reads top-to-bottom and never declares a variable that ends up unused.

```go
// modern — Go 1.26+
if netErr, ok := errors.AsType[*net.OpError](err); ok {
    // use netErr
}

// old — pre-1.26
var netErr *net.OpError
if errors.As(err, &netErr) {
    // use netErr
}
```

Keep using `errors.Is` for sentinel comparisons (`errors.Is(err, io.EOF)`) — `AsType` is for *typed* errors with fields, `Is` is for *value* identity.

## 3. errors.Join for multiple errors (Go 1.20+)

When a function can produce multiple independent errors (e.g. closing several resources, validating several fields), combine them with `errors.Join` rather than returning only the first or building ad-hoc strings.

```go
func close(rs ...io.Closer) error {
    var errs []error
    for _, r := range rs {
        if err := r.Close(); err != nil {
            errs = append(errs, err)
        }
    }

    return errors.Join(errs...)
}
```

`errors.Join` returns `nil` when the slice is empty or all-nil, and an `errors.Is` / `errors.AsType` lookup walks every joined error.

## 4. `any` over `interface{}` (Go 1.18+)

`any` is the predeclared alias for `interface{}`. Use `any` everywhere — function signatures, type parameters, struct fields, map values. `interface{}` is reserved for code that must compile against pre-1.18 Go (which this project does not).

```go
// modern
func Log(fields map[string]any) { ... }

// legacy
func Log(fields map[string]interface{}) { ... }
```

The `revive` rule `use-any` is enabled in this project; CI fails on `interface{}`.

## 5. The `slices`, `maps`, and `cmp` standard packages (Go 1.21+)

Replace hand-rolled loops with the standard generic helpers:

| Hand-rolled                              | Modern                                        |
|------------------------------------------|-----------------------------------------------|
| linear `for` to find an element          | `slices.Contains`, `slices.Index`             |
| sort with custom less function           | `slices.SortFunc(s, cmp.Compare)`             |
| `sort.Slice(...)` for primitives         | `slices.Sort(s)`                              |
| reverse via swap loop                    | `slices.Reverse(s)`                           |
| copy slice                               | `slices.Clone(s)`                             |
| collect map keys                         | `slices.Collect(maps.Keys(m))`                |
| collect map values                       | `slices.Collect(maps.Values(m))`              |
| zero a map                               | `clear(m)`                                    |
| pick first non-zero of two/three values  | `cmp.Or(a, b, c)`                             |

```go
// wrong — manual sort
sort.Slice(users, func(i, j int) bool { return users[i].Age < users[j].Age })

// correct — generic, type-safe, no closure
slices.SortFunc(users, func(a, b User) int { return cmp.Compare(a.Age, b.Age) })
```

The `revive` rule `use-slices-sort` is enabled.

## 6. Range over an integer (Go 1.22+)

Use `for i := range N` instead of the C-style `for i := 0; i < N; i++` when you just want a counted loop.

```go
// modern
for i := range 10 {
    fmt.Println(i)
}

// legacy
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

## 7. Range over a function — iterators (Go 1.23+)

When you author a sequence (lazy, infinite, paginated), expose it as an `iter.Seq[T]` or `iter.Seq2[K, V]` so callers can use `for v := range seq { ... }` directly.

```go
import "iter"

// Producer
func words(s string) iter.Seq[string] {
    return func(yield func(string) bool) {
        for _, w := range strings.Fields(s) {
            if !yield(w) {
                return
            }
        }
    }
}

// Consumer
for w := range words("the quick brown fox") {
    fmt.Println(w)
}
```

This replaces channel-based iterators (which leaked goroutines on early break) and visitor-callback APIs.

## 8. Loop variable scoping — Go 1.22+

Since Go 1.22, the loop variable in `for i, v := range s` is **per-iteration**, so capturing it in a goroutine or closure is safe. Drop the old `v := v` shadowing trick.

```go
// modern — safe under 1.22+
for _, v := range items {
    go process(v)
}

// pre-1.22 workaround — no longer needed, delete on sight
for _, v := range items {
    v := v
    go process(v)
}
```

## 9. `min`, `max`, `clear` builtins (Go 1.21+)

```go
hi := max(a, b, c)
lo := min(a, b)
clear(cache)            // zero a map or slice in place
```

No more `if a > b { ... } else { ... }` clamp code, no more `for k := range m { delete(m, k) }`.

## 10. Modernize on sight — but stay scoped

When you touch a function, modernize the legacy idioms inside *that function*. Don't open a global rewrite PR. The `golangci-lint` `modernize` analyzer (or `gopls`) will surface the rest; sweep them in a dedicated change.

## How to apply (checklist)

1. Every function that does I/O or calls a context-aware API: takes `ctx context.Context` as its first parameter.
2. Every type assertion on errors: `errors.AsType[*T](err)` first; fall back to `errors.Is` for sentinels.
3. Every `interface{}`: rewrite to `any`.
4. Every `for i := 0; i < N; i++` with no other clauses: rewrite to `for i := range N`.
5. Every manual sort/contains/index loop: check `slices` / `maps` / `cmp` first.
6. Every `v := v` inside a `for range`: delete (Go 1.22+ makes it unnecessary).

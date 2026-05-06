---
name: golang-naming
description: Idiomatic Go naming conventions Claude must apply when writing or reviewing Go code — packages, constructors, structs, interfaces, constants, enums, errors, booleans, receivers, getters, functional options, acronyms, test functions, and subtests. Trigger this skill on every Go file edit and explicitly when the user mentions package naming, "MixedCaps vs snake_case", "Get prefix", "ALL_CAPS constants", or naming alternatives.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🔤"
allowed-tools: Read Edit Write Glob Grep Agent
---

# Golang Naming Conventions

Idiomatic names are the cheapest documentation in Go. Apply these rules automatically; deviations need a reason.

## Packages

- **Lowercase, single word, no underscores or camelCase.** `httputil`, not `HttpUtil` or `http_util`.
- **Singular**, not plural — `model`, not `models`. The package's *plurality* lives in the types it exports.
- **Avoid generic dumping-ground names** — `utils`, `helpers`, `common`, `base`, `misc`, `lib`, `shared`. They invite incoherent contents and circular imports. If you can't name the package after what's *in* it, it shouldn't exist.
- **Avoid stutter** — a `user` package should export `User`, not `UserModel`; callers write `user.User`, not `user.UserModel`.
- **Match the directory name.** Revive's `package-directory-mismatch` enforces this with a few exceptions (`testcases`, `testinfo`).
- **Internal-only code** lives under `internal/`.

## Constructors

- **`New`** — default factory for the package's primary type: `bytes.NewBuffer`, `time.NewTicker`. Returns `*T` (or `T, error`).
- **`NewT`** — when a package exposes more than one constructible type: `errors.New`, `bufio.NewReader`, `bufio.NewWriter`.
- **No constructor** when the zero value is usable (`sync.Mutex{}`, `bytes.Buffer{}`, `strings.Builder{}`). Document the zero-value contract instead.
- Validation belongs in the constructor: `NewEmail(raw string) (Email, error)` — never let an invalid value escape.

## Structs

- **PascalCase exported, camelCase unexported.**
- **Avoid stutter:** `http.Request`, not `http.HTTPRequest`.
- **Don't repeat the type in fields:** `User.Name`, not `User.UserName`.
- **Type embedding promotes fields** — pick names so the promoted access reads naturally.

## Interfaces

- **Behavior-named.** `Reader`, `Writer`, `Closer`, `Stringer`. The interface is *what it does*, not *what it is*.
- **`-er` suffix for single-method interfaces.** `Reader { Read(...) }`, `Stringer { String() string }`.
- **Composed interfaces use the verb-pair convention.** `ReadWriter`, `ReadCloser`, `ReadWriteCloser`.
- **"Accept interfaces, return structs."** Functions take the smallest interface that captures the dependency; they return concrete types so callers see all the available behavior.
- **Define interfaces on the consumer side**, not the provider side — keeps producers from declaring dozens of speculative interfaces and forces small, purpose-fit ones.

```go
// good — small consumer-side interface
package report

type userFetcher interface {
    FetchUser(ctx context.Context, id string) (User, error)
}

func Build(ctx context.Context, f userFetcher, ids []string) ([]Row, error) { ... }
```

## Constants

- **MixedCaps**, never `ALL_CAPS` or `SCREAMING_SNAKE_CASE`. `MaxRetries`, not `MAX_RETRIES`.
- **Group with `iota`** for related sequences:

```go
const (
    StatusPending  Status = iota // 0
    StatusActive                  // 1
    StatusArchived                // 2
)
```

- **Sentinel errors** are `var ErrXxx = errors.New("...")`, not `const`.

## Enums

- **Type-safe with a named type and `iota`.** Never use bare `int` for enums.
- **Prefer the zero value to mean "unknown / unset"** so an uninitialized enum is detectably wrong:

```go
type Status int

const (
    StatusUnknown Status = iota
    StatusReady
    StatusRunning
    StatusDone
)
```

- **Implement `fmt.Stringer`** so enum values print as readable names.

## Errors

- **Sentinel errors:** `ErrNotFound`, `ErrTimeout` — package-level vars, named with `Err` prefix.
- **Error types:** `XxxError` — types that carry structured data (`*url.Error`, `*os.PathError`).
- **Distinguish:** sentinels are for `errors.Is`; types are for `errors.AsType`.
- **Error message format** (also enforced by revive's `error-strings`):
  - Lowercase first letter
  - No trailing punctuation
  - No newlines

```go
var ErrInvalidInput = errors.New("invalid input")  // good
var ErrInvalidInput = errors.New("Invalid input.") // bad — lint fail
```

## Booleans

- **Phrase positively.** `isReady`, `isOpen`, `hasItems`, `canRetry` — never `isNotReady` or `isNotOpen`. Negated names produce double negatives at call sites: `if !isNotReady` is unreadable.
- **`is`/`has`/`can` prefixes** are conventional but optional when the field name is already a clear predicate (`open`, `closed`).

## Receivers

- **1-2 characters, lowercase.** `u` for `User`, `c` for `Client`, `db` for `*DB`. Revive's `receiver-naming` caps the length at 2.
- **Consistent across all methods** of the same type — never mix `(u *User)` and `(user *User)` in the same file/package.
- **Don't use `self` or `this`.** Go isn't OO-pretending; the receiver is just a parameter.

## Getters and setters

- **Drop the `Get` prefix.** A field accessor on `User` named `Name` is a getter — `user.Name()`. Reserve `GetX` for cases that *do work* (network call, computation with side effects).
- **`SetX` is fine** for setters that *do work* or guard invariants. For plain field assignment, expose the field instead.

```go
// good
func (u *User) Name() string { return u.name }

// bad — Java-style getter
func (u *User) GetName() string { return u.name }

// fine — actually fetches over the network
func (c *Client) GetUser(ctx context.Context, id string) (User, error) { ... }
```

## Functional options

- **`WithX` prefix** for option functions: `WithTimeout(d)`, `WithLogger(l)`, `WithRetries(n)`.
- **Each option is a function** that closes over its argument and mutates the config struct, exposed as a single type:

```go
type Option func(*config)

func WithTimeout(d time.Duration) Option {
    return func(c *config) { c.timeout = d }
}

func New(opts ...Option) *Client {
    c := &config{ /* defaults */ }
    for _, opt := range opts {
        opt(c)
    }
    return &Client{cfg: *c}
}

// usage
cli := New(WithTimeout(5*time.Second), WithRetries(3))
```

## Acronyms and initialisms

- **All-caps when exported, all-lowercase when unexported.** `URL`, `userURL`. Never `Url` or `userUrl`.
- Common ones: `URL`, `HTTP`, `HTTPS`, `JSON`, `XML`, `YAML`, `SQL`, `ID`, `IP`, `UUID`, `API`, `OS`, `RPC`, `TLS`.
- Revive's `var-naming` allows `ID` and rejects `Id`. The project also denies `VM`.

```go
// good
type APIClient struct { ... }
var customerID string

// bad — lint fail
type ApiClient struct { ... }
var customerId string
```

## Test functions

- **`TestXxx(t *testing.T)`** — unit tests. The `Xxx` matches the function/type under test.
- **`BenchmarkXxx(b *testing.B)`** — benchmarks.
- **`ExampleXxx()`** — runnable examples that double as documentation.
- **`FuzzXxx(f *testing.F)`** — fuzz tests (Go 1.18+).

## Subtest names

- **Descriptive, hyphen- or underscore-separated.** Avoid spaces (they become `_` in test output and break easy `-run` filters).

```go
// good
t.Run("returns_error_on_empty_input", func(t *testing.T) { ... })
t.Run("normalizes-email-case",        func(t *testing.T) { ... })

// bad — runs as "valid input but no auth" but matches awkwardly with -run
t.Run("valid input but no auth", func(t *testing.T) { ... })
```

For table-driven tests, name the case after the *scenario*, not the input shape: `"empty_input"`, `"two_users_same_email"`, not `"case_1"`.

## How to apply (checklist)

1. New package → can you name it after what's *in* it? If you reach for `utils`/`helpers`, redesign.
2. New type → does the package name + type name stutter?
3. New interface → is it behavior-named with `-er` (single method) or composed (`ReadCloser`-style)?
4. New constant → MixedCaps; if it's an enum, named type + `iota` + `String()`.
5. New error → sentinel (`ErrX`) or type (`XError`)? Lowercase message, no punctuation.
6. New boolean → phrased positively, optional `is`/`has`/`can` prefix.
7. New method → receiver is 1-2 chars, consistent across the type.
8. New getter → drop the `Get` prefix unless work is being done.
9. New options → `WithX` functions returning `Option`.
10. New acronym → exported = ALL CAPS, unexported = all lowercase, never `Url`/`Id`.

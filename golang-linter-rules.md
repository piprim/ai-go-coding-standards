---
name: golang-linter-rules
description: Project-specific Go linter rules that Claude must respect proactively when writing or modifying Go code — covers nlreturn (blank line before return), wrapcheck (always wrap external errors), add-constant (no repeated string literals or magic numbers), error-strings formatting. Trigger this skill on every Go edit, even if the user never mentions linting; the project's CI fails on any violation.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "✅"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Agent
---

# Golang Linter Rules

## 1. nlreturn — blank line before `return`

Add a blank line before every `return` statement that is preceded by other statements in the same `{}` block. The exception is a `return` that is the *only* statement in its block (e.g. `if err != nil { return err }`).

**Why:** the `nlreturn` linter treats a missing blank line before `return` as an error. Beyond the linter, the blank line visually separates the *result* from the *work that produced it*, which makes scanning long functions easier.

**How to apply:** any time you write a `return` that is not the only line in its enclosing `{}` block, put a blank line before it.

```go
// wrong — no blank line before return
func f() string {
    s := compute()
    return s
}

// correct — blank line before return
func f() string {
    s := compute()

    return s
}

// also correct — sole statement, no blank line needed
if err != nil {
    return err
}
```

## 2. wrapcheck — wrap errors from external packages

Never return an error from an external package (any import outside the current module) directly. Always wrap it with `fmt.Errorf("context: %w", err)` so the caller can see *what operation failed*, not just *which library complained*.

**Why:** the `wrapcheck` linter flags bare returns of external errors as errors. More importantly, an unwrapped `os.Open` error in a 50-frame stack tells you nothing about which file the program was trying to open.

**How to apply:** any time you call a function from an external package and return its error, add a wrapping message that names the operation and, where relevant, the inputs (path, URL, ID). Use `%w` (not `%v`) so `errors.Is` / `errors.AsType` keep working.

```go
// wrong — bare return of external error
f, err := os.Open(path)
if err != nil {
    return err
}

// correct — wrapped with context
f, err := os.Open(path)
if err != nil {
    return fmt.Errorf("open %q: %w", path, err)
}
```

Errors from the **same module** may also need wrapping depending on the call site — when in doubt, wrap. The marginal cost is one line; the marginal value is a debuggable production stack trace.

## 3. add-constant — no repeated string literals or large magic numbers

Define a named constant when:

- the **same string literal** appears 3 or more times, OR
- a **numeric literal greater than 10** is used (always replace with a constant, even on first use).

Exceptions baked into revive's defaults: empty string `""`, integers `0`, `1`, `2`, and floats `0.0`, `1.0`, `2.0` may stay literal.

**Why:** repeated literals drift apart silently — one updates, the others don't, and you ship a bug. Magic numbers carry no meaning at the call site.

**How to apply:** when you find yourself typing the same string a third time, stop and extract a `const`. When you write `if retries > 25`, extract `const maxRetries = 25` first and reference it.

```go
// wrong — magic number, repeated literal
if attempts > 25 { ... }
log.Printf("module=%s op=connect", "billing")
log.Printf("module=%s op=charge",  "billing")
log.Printf("module=%s op=refund",  "billing")

// correct
const (
    maxRetries  = 25
    moduleLabel = "billing"
)

if attempts > maxRetries { ... }
log.Printf("module=%s op=connect", moduleLabel)
log.Printf("module=%s op=charge",  moduleLabel)
log.Printf("module=%s op=refund",  moduleLabel)
```

## 4. error-strings — formatting

Error strings passed to `errors.New`, `fmt.Errorf`, `xerrors.New`, `panic`, and `core.WriteError.Message` must:

- start with a **lowercase** letter
- **not end** in punctuation (no trailing `.`, `!`, `?`)
- **not contain** line breaks

**Why:** errors are typically wrapped (`fmt.Errorf("ctx: %w", err)`), and the result reads as a single sentence. A capitalized or punctuated inner message produces ugly nested output like `read config: Failed to open file.`.

```go
// wrong
return errors.New("Failed to open file.")

// correct
return errors.New("failed to open file")
```

## 5. unhandled-error — every error must be checked

Every returned `error` must be handled (assigned, returned, or explicitly ignored with `_ = …`). The only allowlisted unhandled calls in this project are `fmt.Printf`, `fmt.Println`, `fmt.Fprint`, `fmt.Fprintln`, `fmt.Fprintf`.

**Why:** silent error drops are how production goes mute. Even when "it can't fail here," the explicit `_` documents the decision and survives refactors.

```go
// wrong — error silently dropped
file.Close()

// correct — explicit choice
_ = file.Close()        // ok if you've decided to ignore it
if err := file.Close(); err != nil {
    log.Warn("close file", "err", err)
}
```

## 6. Structural Complexity & Sizing Limits
These thresholds are strictly enforced to keep entities small and readable:
* Maximum **5** parameters per function.
* Maximum **3** return values per function.
* Maximum **50** statements and **150** lines per function.
* Maximum **500** lines per file (skips comments and blank lines).
* Maximum **124** characters per line.
* Maximum cyclomatic complexity of **15** per function.
* Maximum cognitive complexity of **20** per function.
* Maximum **3** levels of nesting for control structures (`if`, `for`, `switch`).
* Maximum **10** public structs per package.

## 7. Strict Naming Conventions
* Files must match regex `^[_a-z][_a-z0-9]*\.go$`. Excludes `**/migrations/**`.
* Package name must match the directory name. Exceptions: `testcases`, `testinfo`, and excluded `migrations`.
* Do not use user-defined bad names like `foo` or `bar`.
* Initialisms are strictly checked. `ID` is allowed, but `VM` is denied. Upper-case constants are allowed.
* Receiver names have a maximum length of **2** characters.
* Must match `^[a-z][a-z0-9]{0,}$`.

## 8. Styling & Code Format Enforcement
* Maximum **3** instances of the same string literal before requiring a constant. Exceptions: `""`, integers `0,1,2`, and floats `0.0`, `1.0`, `2.0`.
* Always use `make` for map initialization.
* Function arguments and return values must use must use the `short` style.
* Allowed to use `any` style.
* Allows switches with no default clause.
* Strongly prefer early returns to avoid nesting.
* Strict formatting for `core.WriteError.Message`, `fmt.Errorf`, and `panic` (e.g., must not start with a capital letter, must not end in punctuation, must not contain line breaks).

## 9. Security, Scope & Blocklists
* Never import `crypto/md5` or `crypto/sha1`.
* `context.Context` must be the first parameter, though `*testing.T` and `*github.com/user/repo/testing.Harness` are allowed before it.
* All returned errors must be handled, EXCEPT for calls to `fmt.Printf`, `fmt.Println`, `fmt.Fprint`, `fmt.Fprintln`, and `fmt.Fprintf`.
* Parameters/Receivers matching `^_` are allowed to be unused.



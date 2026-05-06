---
name: golang-linter-rules
description: Project-specific Go linter rules that Claude must respect proactively when writing or modifying Go code — covers nlreturn (blank line before return), wrapcheck (always wrap external errors), add-constant (no repeated string literals or magic numbers), error-strings formatting, and the mandatory `golangci-lint run` step. Trigger this skill on every Go edit, even if the user never mentions linting; the project's CI fails on any violation.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "✅"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Agent
---

# Golang Linter Rules

These rules are enforced by `golangci-lint` in this project. Violations fail CI. Apply them automatically while you write code, not as an afterthought — fixing later costs more than getting it right the first time.

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

## 6. Run the linter — every time

After generating or modifying any Go file, run:

```bash
golangci-lint run ./...
```

Fix every violation before reporting the task as done. If the linter complains about something the user has explicitly asked you to keep, prefer a narrowly-scoped `//nolint:rulename // reason` comment over disabling the rule globally.

## How to apply (checklist)

1. Before writing a `return`, ask: is there at least one statement before it in this block? If yes → blank line.
2. Before returning an `err` from an external call, ask: is there context the caller needs? If yes (almost always) → wrap with `fmt.Errorf("op: %w", err)`.
3. Before typing a literal, ask: have I typed this string twice already, or is this number > 10? If yes → extract a `const`.
4. Before writing an error message, ask: does it start lowercase and avoid trailing punctuation?
5. Before declaring done, run `golangci-lint run ./...` and fix every finding.

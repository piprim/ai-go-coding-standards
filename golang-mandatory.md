---
name: golang-mandatory
description: Index of mandatory Go rules for this project. Loading this skill must immediately pull in the ten focused skills it points to. Trigger this skill on any Go-related task as a shortcut for "apply every project Go rule"; do not stop after reading the index — follow the pointers.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "3.0.0"
  openclaw:
    emoji: "📚"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Agent
---

# Mandatory Go Rules — Index

This file is an index. The rules themselves live in focused skills so each can be triggered, read, and updated independently. **Always load every skill in the table below before writing or modifying Go code in this project.**

## Locally-restated skills (load all)

| Skill | Scope |
|-------|-------|
| [`focused-functions`](./focused-functions.md) | Single-responsibility decomposition, factorization of duplicated logic, and avoiding `else` via early returns / polymorphism. **Language-agnostic baseline.** |
| [`golang-linter-rules`](./golang-linter-rules.md) | Project-specific `golangci-lint` rules — `nlreturn`, `wrapcheck`, `add-constant`, `error-strings`, `unhandled-error`, plus the mandatory `golangci-lint run` step. |
| [`golang-modern-go`](./golang-modern-go.md) | Modern Go idioms — context-aware variants, `errors.AsType` (1.26+), `errors.Join`, `slices`/`maps`/`cmp`, range-over-int (1.22+), range-over-func (1.23+), `min`/`max`/`clear`, `any` over `interface{}`. |
| [`golang-calisthenics`](./golang-calisthenics.md) | Flexible design heuristics — wrap meaningful primitives in value-object types, Law of Demeter (one dot per line, fluent-interface exception), keep entities small (`<500` lines / file, `<10` files / package), narrow structs over fat aggregates. |
| [`golang-naming`](./golang-naming.md) | Idiomatic naming — packages, constructors, structs, interfaces, constants, enums, errors, booleans, receivers, getters, functional options, acronyms, test functions, subtests. |
| [`golang-data-structures`](./golang-data-structures.md) | Slices (header / capacity growth / preallocation / `slices` pkg), maps (`maps` pkg, `sync.Map`), arrays, `container/list/heap/ring`, `strings.Builder` vs `bytes.Buffer`, generic collections, `unsafe.Pointer`, `weak.Pointer` (1.24+), copy semantics. |
| [`golang-dependency-injection`](./golang-dependency-injection.md) | Why DI matters; manual constructor injection (the default); functional options; library comparison (`google/wire` compile-time, `uber-go/fx` runtime + lifecycle, `samber/do`, `uber-go/dig`). |
| [`golang-security`](./golang-security.md) | Injection (SQL, command, XSS, path traversal), cryptography (`crypto/rand`, banned MD5/SHA1, password hashing), filesystem, network/TLS, cookies, secrets, memory & concurrency safety, safe logging, tooling (`gosec`, `govulncheck`). |
| [`golang-testing`](./golang-testing.md) | Table-driven tests, subtests, parallelism, testify, mocks, unit vs integration vs e2e, benchmarks + benchstat, coverage, fuzzing (1.18+), `testdata/`, golden files, goleak, race detector, CI. |
| [`golang-revive-rules`](./golang-revive-rules.md) | Comprehensive list of every `revive` rule and parameter enforced in this project — argument-limit 5, function-length 50/150, file-length 500, line-length 124, cyclomatic 15, cognitive 20, max-control-nesting 3, naming, blocklists, and dozens more. |

## Externally-delegated skills (consult on demand)

These topics are not restated locally. When a task touches one of them, invoke the upstream skill rather than guessing:

| Topic | Upstream skill |
|-------|----------------|
| Code style and formatting beyond the linter rules above | `samber/cc-skills-golang/golang-code-style` |
| Dependency management (`go.mod`, MVS, vulnerability scanning, Dependabot/Renovate) | `samber/cc-skills-golang/golang-dependency-management` |
| Idiomatic design patterns (functional options, resource lifecycle, resilience) | `samber/cc-skills-golang/golang-design-patterns` |
| Documentation (godoc comments, README, llms.txt, examples) | `samber/cc-skills-golang/golang-documentation` |
| Project layout (monorepo, `cmd/`, `internal/`, workspaces) | `samber/cc-skills-golang/golang-project-layout` |
| Struct & interface design (composition, embedding, type assertions, struct tags) | `samber/cc-skills-golang/golang-structs-interfaces` |
| Context plumbing in depth (cancellation, timeouts, context values) | `samber/cc-skills-golang/golang-context` |

If any upstream skill is unavailable in the current environment, ask the user to install it before proceeding on a task that depends on it.

## How to apply

1. On any Go task, load all ten **locally-restated** skills before writing or modifying code.
2. When a task touches a topic in the **externally-delegated** table, invoke that upstream skill in addition.
3. After edits, run `golangci-lint run ./...` and fix every finding before reporting the task done.
4. If any rule conflicts with an explicit user instruction in `CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, or in the conversation itself, the user wins — note the deviation and continue.

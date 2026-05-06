# AI Go Coding Standards

**A curated set of 12 flat-file coding-standard skills for Claude Code and compatible AI coding agents.**

Covers a language-agnostic single-responsibility baseline, ports-and-adapters architecture, and a full Go stack:
- revive linter rules
- modern Go (1.22+ / 1.23+ / 1.26+) idioms
- naming conventions
- data-structure choices
- dependency injection
- security defaults
- production-ready testing

Drop the `.md` files into your project and the agent applies them automatically.

## Foundation (language-agnostic)

- **[`focused-functions`](./focused-functions.md)** — Each function does exactly one nameable thing; duplicated logic is extracted on sight; `else` gives way to early returns or polymorphism.

## Architecture

- **[`hexagonal-architecture`](./hexagonal-architecture.md)** — Ports & Adapters: domain-centric design with inbound/outbound port interfaces, adapter implementations at the edges, and a single composition root. Cross-language (TypeScript / Java / Kotlin / Go).

## Go — linter & rules enforcement

- **[`golang-revive-rules`](./golang-revive-rules.md)** — Comprehensive list of every `revive` rule and parameter the project enforces (argument-limit 5, function-length 50/150, file-length 500, line-length 124, cyclomatic 15, cognitive 20, naming, blocklists).
- **[`golang-linter-rules`](./golang-linter-rules.md)** — Project-specific `golangci-lint` rules: `nlreturn` (blank line before return), `wrapcheck` (always wrap external errors), `add-constant` (no repeated literals or magic numbers), `error-strings` formatting, mandatory `golangci-lint run` step.

## Go — language usage

- **[`golang-modern-go`](./golang-modern-go.md)** — Post-1.18 idioms: context-aware variants, `errors.AsType` (1.26+), `errors.Join`, `slices`/`maps`/`cmp`, range-over-int (1.22+), range-over-func (1.23+), `min`/`max`/`clear`, `any` over `interface{}`.
- **[`golang-naming`](./golang-naming.md)** — Idiomatic naming: packages, constructors, structs, interfaces, constants, enums, errors, booleans, receivers, getters, functional options, acronyms, test functions.
- **[`golang-data-structures`](./golang-data-structures.md)** — Slices (header / cap / preallocation / `slices` pkg), maps (`maps` pkg, `sync.Map`), arrays, `container/*`, `strings.Builder` vs `bytes.Buffer`, generics, `unsafe`/`weak.Pointer`, copy semantics.

## Go — design heuristics

- **[`golang-calisthenics`](./golang-calisthenics.md)** — Flexible design heuristics: wrap meaningful primitives in value-object types, Law of Demeter (one dot per line, fluent-interface exception), keep entities small, narrow structs over fat aggregates.
- **[`golang-dependency-injection`](./golang-dependency-injection.md)** — Why DI matters; manual constructor injection (the default); functional options; library comparison (`google/wire` compile-time, `uber-go/fx` runtime + lifecycle, `samber/do`, `uber-go/dig`).

## Go — production concerns

- **[`golang-security`](./golang-security.md)** — Injection (SQL / command / XSS / path traversal), crypto (`crypto/rand`, banned MD5/SHA1, password hashing), filesystem, TLS, cookies, secrets, memory & concurrency safety, safe logging, `gosec` / `govulncheck`.
- **[`golang-testing`](./golang-testing.md)** — Table-driven tests, subtests, parallelism, testify, mocks, unit vs integration vs e2e, benchmarks + benchstat, coverage, fuzzing, `testdata/`, golden files, `goleak`, race detector, CI.

## Index

- **[`golang-mandatory`](./golang-mandatory.md)** — Index file. Loading it pulls in all 10 Go skills above; also lists 7 externally-delegated samber skills to consult on demand.

## Trigger validation

The [`evals/trigger-tests.json`](./evals/trigger-tests.json) file contains realistic user prompts for each skill (3 should-trigger + 1 near-miss should-not-trigger) — used to sanity-check the `description` field in each skill's frontmatter.

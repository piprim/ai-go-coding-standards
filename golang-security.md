---
name: golang-security
description: Security and vulnerability prevention for Go — injection (SQL, command, XSS, path traversal), cryptography (crypto/rand vs math/rand, banned MD5/SHA1, password hashing), filesystem safety, network/TLS, cookies, secrets management, memory & concurrency safety, and safe logging. Trigger this skill on every Go change touching user input, authentication, authorization, crypto, file I/O, network calls, or secrets — even when the user does not explicitly say "security", "vulnerability", or "audit". Also covers tooling — gosec, govulncheck, codeql.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🛡️"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(govulncheck:*) Bash(gosec:*) Agent
---

# Golang Security

Most Go security failures come from a small set of repeat offenders. Apply these defaults; treat any deviation as a code-review blocker.

## 1. Injection

### SQL injection — always parameterize

```go
// wrong — string interpolation, classic injection
db.QueryContext(ctx, "SELECT * FROM users WHERE name = '"+name+"'")

// correct — parameterized; driver does the escaping
db.QueryContext(ctx, "SELECT * FROM users WHERE name = $1", name)
```

Never concatenate user input into SQL — not even "trusted" input from internal services. The cost of a `$N` placeholder is zero and it survives every refactor where "trusted" turns out not to be.

For `IN (...)` clauses with a variable number of values, build the placeholder list (`$1, $2, $3`), don't interpolate the values themselves.

### Command injection — never use a shell

```go
// wrong — invokes a shell, vulnerable to spaces/quotes/`;`
exec.CommandContext(ctx, "sh", "-c", "convert "+userPath+" /tmp/out.png").Run()

// correct — args are exec'd directly, no shell parsing
exec.CommandContext(ctx, "convert", userPath, "/tmp/out.png").Run()
```

Always pass arguments as separate strings, never as a single shell command. If you genuinely must run shell, validate the input against a strict allowlist *before* it touches the command line.

### XSS — `html/template`, not `text/template`

For HTML output, use `html/template`. It auto-escapes per context (HTML body, attributes, JS, URLs).

```go
// wrong — no escaping, attacker controls HTML
import "text/template"
tmpl, _ := template.New("p").Parse("<p>{{.}}</p>")

// correct — context-aware escaping
import "html/template"
tmpl, _ := template.New("p").Parse("<p>{{.}}</p>")
```

`text/template` is for non-HTML strings (config files, code generation). Pick deliberately.

### Path traversal

```go
// wrong — user controls the path; ../../../etc/passwd works
data, _ := os.ReadFile(filepath.Join(uploadDir, userFilename))

// correct — clean, resolve, validate the result is still under uploadDir
clean := filepath.Clean(userFilename)
full := filepath.Join(uploadDir, clean)
abs, err := filepath.Abs(full)
if err != nil {
    return fmt.Errorf("resolve path: %w", err)
}

uploadAbs, _ := filepath.Abs(uploadDir)
if !strings.HasPrefix(abs, uploadAbs+string(os.PathSeparator)) {
    return errors.New("path escapes upload directory")
}
```

Better still: use `os.Root` (Go 1.24+) which scopes all `os.*` calls to a directory and refuses to escape it.

## 2. Cryptography

### Random — `crypto/rand`, not `math/rand`

```go
// wrong — math/rand is predictable
import "math/rand"
token := fmt.Sprintf("%x", rand.Int63())

// correct — crypto/rand is non-deterministic
import "crypto/rand"
buf := make([]byte, 32)
if _, err := rand.Read(buf); err != nil {
    return fmt.Errorf("generate token: %w", err)
}

token := hex.EncodeToString(buf)
```

`math/rand` is for simulations and shuffling test data. **Anything used for security** (tokens, IDs, salts, IVs, session identifiers, CSRF cookies) uses `crypto/rand`.

### Banned hashes — no MD5, no SHA-1

The project's revive `imports-blocklist` rejects `crypto/md5` and `crypto/sha1`. Both are broken for security purposes:

- **Hashing for security** (passwords, signatures, integrity) → `crypto/sha256` or `crypto/sha512`.
- **Non-cryptographic hashing** (cache keys, sharding, bloom filters) → `hash/maphash`, `hash/fnv`, or `xxhash`.

### Password hashing

Never store passwords with a general-purpose hash (even SHA-256). Use a slow, salted, memory-hard KDF:

```go
import "golang.org/x/crypto/bcrypt"

hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
// store hash; never the password

// verify
err := bcrypt.CompareHashAndPassword(hash, []byte(submitted))
```

`bcrypt` is the conservative default. `argon2id` (`golang.org/x/crypto/argon2`) is the modern choice when you control parameters; pick one team-wide and document the cost factor.

### Constant-time comparisons

Comparing secrets (HMAC tags, tokens, password hashes already computed) with `==` leaks timing information. Use `crypto/subtle.ConstantTimeCompare`:

```go
if subtle.ConstantTimeCompare(expectedMAC, gotMAC) != 1 {
    return errors.New("invalid signature")
}
```

## 3. Filesystem safety

- **Validate file extensions and content types** server-side; never trust the client-supplied `Content-Type`.
- **Limit upload size** with `http.MaxBytesReader`.
- **Set explicit permissions** when creating files: `os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o600)` — the `0o600` denies group/other access; `O_EXCL` refuses to overwrite.
- **Use `os.Root` (Go 1.24+)** to scope file operations to a directory.

## 4. Network security

### TLS

- **TLS 1.2 minimum, TLS 1.3 preferred.** `tls.Config{ MinVersion: tls.VersionTLS12 }` (or 13).
- **Don't disable verification** outside of throwaway scripts. `InsecureSkipVerify: true` in production is a vulnerability.
- **For server certs**, use the standard chain — Let's Encrypt via `golang.org/x/crypto/acme/autocert`, or a real cert from your PKI. Self-signed certs in production force every client to weaken its verification.

### URLs

- **Reject `http://` schemes** for any endpoint that handles credentials, tokens, or PII.
- **Validate URLs from untrusted input** before fetching — block `file://`, `gopher://`, internal IP ranges (`127.0.0.0/8`, `169.254.169.254` for cloud metadata, RFC1918 ranges) to prevent SSRF.
- Revive's `unsecure-url-scheme` flags hard-coded `http://` URLs in source.

### HTTP client defaults

The default `http.Client` has no timeout. Always set one:

```go
client := &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{MinVersion: tls.VersionTLS12},
        // sensible defaults for keep-alives, idle conns, etc.
    },
}
```

## 5. Cookies

```go
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    sessionID,
    Path:     "/",
    HttpOnly: true,                    // not readable from JS
    Secure:   true,                    // HTTPS only
    SameSite: http.SameSiteLaxMode,    // or Strict, depending on flow
    MaxAge:   3600,
})
```

Three flags that should always be on for auth cookies: **HttpOnly**, **Secure**, **SameSite**.

## 6. Secrets management

- **Never hardcode** API keys, passwords, or tokens in source. Even in test files.
- **Read from env vars or a secret store** (Vault, AWS/GCP secret manager). Never from a committed config file.
- **Never log secrets.** Even at DEBUG level — logs end up in third-party aggregators.
- **Redact in error messages.** When wrapping an error that contains a request, scrub the `Authorization` header before formatting.
- **Rotate.** Anything that can leak (a long-lived API key) should be rotatable without code changes.

## 7. Memory and concurrency safety

- **Run with `-race` in CI.** Data races are silent corruption; `go test -race ./...` catches most.
- **Goroutine leaks** are a security issue (resource exhaustion DoS). Use `golang.org/x/sync/errgroup` or `context.WithCancel` so child goroutines exit when the parent does. Verify with `go.uber.org/goleak` in tests.
- **Bounded concurrency.** Don't `go func() { ... }()` in a loop over user input — that's a DoS-for-free pattern. Use a worker pool or a semaphore.
- **Avoid `unsafe.Pointer`** in code paths that touch user data. Type punning is how memory corruption escapes the type system.

## 8. Safe logging

- Use **structured logging** (`log/slog`) so fields are typed and easy to redact.
- **Allowlist** what you log — `slog.Info("auth", "user_id", uid)`, not `slog.Info("auth", "request", req)`.
- **Don't log request bodies** containing PII or credentials. If you need request shape for debugging, log lengths and field names, not values.
- **Set a log retention policy** and follow it; long retention is a leak waiting to happen.

## 9. Tooling — run these in CI

- **`govulncheck ./...`** — checks dependencies and the standard library against the Go vulnerability database. Should be a CI gate.
- **`gosec ./...`** — finds common insecure patterns (hardcoded creds, weak crypto, unsafe `exec`, etc.).
- **`golangci-lint`** with the `gosec`, `bodyclose`, `noctx`, and `errcheck` linters enabled.
- **CodeQL / Semgrep** for cross-cutting policy rules in larger orgs.

## How to apply (checklist)

1. Building a SQL query → parameterized, never `+` or `Sprintf`.
2. Running an external command → `exec.CommandContext` with separate args, no shell.
3. Rendering HTML → `html/template`.
4. Joining a path with user input → `filepath.Clean` + boundary check, or `os.Root`.
5. Generating a token / salt / ID → `crypto/rand`, never `math/rand`.
6. Hashing a password → `bcrypt` or `argon2id`, never raw SHA.
7. Importing `crypto/md5` or `crypto/sha1` → don't (revive blocks it).
8. Comparing secret values → `subtle.ConstantTimeCompare`.
9. Setting an auth cookie → `HttpOnly`, `Secure`, `SameSite` all on.
10. New `http.Client` → `Timeout` and `MinVersion: TLS12`.
11. Reading a secret → from env or secret manager, never hardcoded.
12. Logging → structured fields, allowlisted, never raw bodies.
13. Before merging → `govulncheck ./...` and `gosec ./...` clean.

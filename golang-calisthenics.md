---
name: golang-calisthenics
description: Object-calisthenics-style design heuristics adapted for Go — wrap meaningful primitives in value-object types, respect the Law of Demeter (one dot per line, with fluent-interface exception), keep entities small (no file >500 lines, no package >10 files), and prefer narrowly-scoped structs over fat aggregates. Trigger this skill when designing new types, reviewing struct or method-call shape, or when the user mentions DDD, value objects, Law of Demeter, fluent interfaces, or "the file is getting long".
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🤸"
allowed-tools: Read Edit Write Glob Grep Agent
---

# Go Calisthenics — Flexible Design Heuristics

These are *flexible* rules — heuristics, not absolutes. They steer designs toward smaller, more cohesive types and clearer dependency shapes. Apply them by default; deviate consciously when the alternative is awkward, and document the reason in a code comment when you do.

## 1. Wrap meaningful primitives in value-object types

A `string` named `email`, `userID`, `orderID`, or `currencyCode` carries a *meaning* and usually *behavior* (validation, normalization, formatting). When that's true, wrap it in a named type instead of letting the primitive flow through every signature.

**Why:** named types catch parameter mix-ups at compile time (`SendEmail(userID, email)` versus `SendEmail(email, userID)`), give you one canonical place for validation, and turn DDD value-object semantics from a comment into a type.

**Apply when:** the value has rules (an email must contain `@`), invariants (a `Cents` is non-negative), or a domain-specific behavior (`OrderID.Short()` returns the last 8 chars). Don't wrap when the primitive is genuinely just a number — a loop index, an arbitrary count.

```go
// thin — passing strings around, easy to swap
func Notify(userID, email, message string) error { ... }

// strong — types prevent the swap, encapsulate the rules
type (
    UserID  string
    Email   string
)

func (e Email) Validate() error { ... }
func (e Email) Normalize() Email { ... }

func Notify(uid UserID, e Email, message string) error { ... }
```

Constructors that return `(T, error)` are the right place to enforce invariants: `NewEmail(raw string) (Email, error)`.

## 2. One dot per line — Law of Demeter

Avoid chained method calls that walk through several object boundaries: `a.B().C().D()`. Prefer assigning intermediate steps to named variables, or asking the immediate collaborator to do the work for you.

**Why:** chains couple the caller to the *internal structure* of every type in the path. A change to `B.C()` ripples to every chain that walked through it. Named intermediates also give the next reader (and the debugger) a place to land.

```go
// brittle — caller knows about User, Profile, AND Address
city := user.Profile().Address().City()

// better — ask the immediate collaborator
city := user.City()

// or — name the step
profile := user.Profile()
city := profile.AddressCity()
```

**Exception — fluent interfaces and method chaining patterns.** Builders, query builders, and DSL-style APIs are *designed* to chain. The chain returns the same type (or a closely related builder type) and accumulates configuration, not navigation. These are fine and expected:

```go
q := db.Select("id", "name").
    From("users").
    Where("active = ?", true).
    OrderBy("name").
    Limit(50)

cmd := cobra.Command{...}
cmd.Flags().StringP("config", "c", "", "config file")
```

The rule applies to **navigation chains**, not **construction chains**.

## 3. Keep all entities small

- **No file longer than ~500 lines** (revive's `file-length-limit` enforces this; comments and blank lines don't count).
- **No package larger than ~10 source files.**
- **No function longer than 50 statements / 150 lines** (revive's `function-length`).

**Why:** long files hide cohesion problems — they're usually two or three responsibilities glued together. Once a file passes 500 lines or a package passes 10 files, the next change is harder than the previous one. Splitting early is cheaper than splitting late.

**How to apply:** when a file approaches 500 lines, identify the natural seam (commands, sub-domain, layer) and extract before adding more. When a package approaches 10 files, ask whether you have *one* package doing two jobs.

## 4. Prefer narrow structs over fat aggregates

Try to keep structs to **two instance fields** as a *target*, not a hard cap. The rule's purpose is to push you toward decomposition: if you find yourself writing a `User` with 14 fields, the design is asking for sub-types — `User { ID UserID; Profile Profile; Settings Settings }` — where `Profile` and `Settings` are themselves small structs.

**Why:** small structs compose; large structs accumulate concerns and become a magnet for unrelated methods. High cohesion comes from small, focused types collaborating, not from one struct that "has everything you need".

**Apply with judgment.** A DTO mapped from JSON or a database row legitimately has many fields — that's its job. A *behavior-bearing* domain type is the one to keep narrow.

```go
// fat — many concerns in one type
type User struct {
    ID, Name, Email           string
    Street, City, Zip, Country string
    Theme, Language, Timezone  string
    LastLogin, CreatedAt       time.Time
    // ... and methods touching all of these
}

// decomposed — each sub-type has cohesive fields and methods
type User struct {
    ID       UserID
    Profile  Profile
    Address  Address
    Prefs    Preferences
    Activity ActivityLog
}
```

## How to apply (checklist)

1. New type — does the primitive carry meaning or behavior? If yes, wrap it.
2. New method call — am I navigating through 2+ boundaries? If yes, ask the immediate collaborator or name the intermediate step (unless this is a fluent-interface chain).
3. File is approaching 400 lines — find the seam now, not at 500.
4. Struct has more than ~5 behavior-bearing fields — look for a sub-type aching to be extracted.

For control-flow shape (avoiding `else`, early returns, polymorphism over branching), see the **focused-functions** skill.

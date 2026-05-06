---
name: focused-functions
description: Non-negotiable single-responsibility and code-factorization rules — every function does exactly one nameable thing, duplicated logic is extracted on sight, and `else` branches give way to early returns or polymorphism. Use this skill on every function or method Claude writes, modifies, or reviews, in any language; trigger proactively even when the user does not mention "refactor", "decompose", or "DRY", because these are baseline coding standards in this project.
---

# Focused Functions & Factorization

These are non-negotiable base coding rules. Apply them automatically whenever writing or modifying code, in any language.

## Rule 1 — Each function/method does exactly one thing

Decompose until each unit has a single, nameable responsibility:

- If you need "and" to describe what a function does, split it.
- If a function has setup + work + teardown at the same abstraction level, each section is a candidate for extraction.
- Orchestrators call helpers; helpers do not call orchestrators.
- A function that is hard to name is a function that does too much.

**Bad:**
```go
func processAndSaveUser(db *sql.DB, raw []byte) error {
    var u User
    json.Unmarshal(raw, &u)         // parse
    u.Email = strings.ToLower(u.Email) // normalize
    db.Exec("INSERT ...", u.Name, u.Email) // persist
    sendWelcomeEmail(u.Email)       // side-effect
    return nil
}
```

**Good:**
```go
func parseUser(raw []byte) (User, error)     { ... }
func normalizeUser(u *User)                  { ... }
func saveUser(ctx context.Context, db *sql.DB, u User) error { ... }
func onboardUser(ctx context.Context, db *sql.DB, raw []byte) error {
    u, err := parseUser(raw)
    // ...
    normalizeUser(&u)

    return saveUser(ctx, db, u)
}
```

## Rule 2 — Factorize reusable code

When two or more callsites share the same logic, extract it:

- Duplicate code is a bug waiting to diverge — extract immediately.
- If the shared logic lives in the same package, extract a private helper.
- If it spans packages, promote to a shared package or pass behavior as a parameter.
- Prefer extracting concrete helpers over generic abstractions (YAGNI).

**Signals to factorize:**
- Same error-wrapping pattern repeated (`fmt.Errorf("context: %w", err)`)
- Same guard clause repeated across functions
- Same form/TUI construction repeated across commands
- Same store-open / defer-close pattern repeated

## Rule 3 — Avoid `else` statements

Where the code wants to write `if … else …`, rework it into one of:

- **Early return on the negative branch**, dedenting the positive — the happy path then stays at the lowest indentation level.
- **Defensive guard clauses** at the top of the function that bail out if preconditions aren't met.
- **Polymorphism** (Strategy / State patterns) when the dispatch is open-ended and the branches are likely to grow.

**Why:** `else` after a `return`, `panic`, `break`, or `continue` is structurally redundant — revive's `superfluous-else` rule flags it. More importantly, deeply nested if/else trees hide the happy path behind layers of indentation; early returns expose it.

```go
// bad — happy path indented under negative branch
func handle(req Request) error {
    if req.IsValid() {
        result, err := process(req)
        if err != nil {
            return err
        } else {
            return save(result)
        }
    } else {
        return ErrInvalidRequest
    }
}

// good — guards first, happy path flat
func handle(req Request) error {
    if !req.IsValid() {
        return ErrInvalidRequest
    }

    result, err := process(req)
    if err != nil {
        return err
    }

    return save(result)
}
```

## How to apply

1. Before writing a new function: write its name and one-sentence description. If you cannot write one sentence without "and", decompose the design first.
2. After writing a block of code: scan for duplication with existing code in the same file and package. Extract if found.
3. While writing control flow: when you reach for `else`, ask whether early return or guard clause works first.
4. During review: flag any function longer than ~20 lines as a decomposition candidate unless every line is at the same abstraction level.

---
name: golang-revive-rules
description: Comprehensive enforcement of ALL revive linter rules with the project's exact parameters. Use this skill whenever writing, reviewing, or refactoring Go code — Claude MUST respect every threshold (argument-limit 5, function-length 50/150, file-length 500, line-length 124, cyclomatic 15, cognitive 20, max-control-nesting 3, etc.), every naming rule, and every blocklisted import listed below. Trigger proactively even if the user does not mention "revive" or "lint".
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "2.0.0"
  openclaw:
    emoji: "🚨"
---

# Respect Revive Linter Rules
The `revive` linter is strictly configured in this project. You must adhere to the following rules and their exact parameters when generating or refactoring Go code.

## 1. Structural Complexity & Sizing Limits
These thresholds are strictly enforced to keep entities small and readable:
*   **`argument-limit`:** Maximum **5** parameters per function.
*   **`function-result-limit`:** Maximum **3** return values per function.
*   **`function-length`:** Maximum **50** statements and **150** lines per function.
*   **`file-length-limit`:** Maximum **500** lines per file (skips comments and blank lines).
*   **`line-length-limit`:** Maximum **124** characters per line.
*   **`cyclomatic`:** Maximum cyclomatic complexity of **15** per function.
*   **`cognitive-complexity`:** Maximum cognitive complexity of **20** per function.
*   **`max-control-nesting`:** Maximum **3** levels of nesting for control structures (`if`, `for`, `switch`).
*   **`max-public-structs`:** Maximum **10** public structs per package.

## 2. Strict Naming Conventions
*   **`filename-format`:** Files must match regex `^[_a-z][_a-z0-9]*\.go$`. Excludes `**/migrations/**`.
*   **`package-directory-mismatch`:** Package name must match the directory name. Exceptions: `testcases`, `testinfo`, and excluded `migrations`.
*   **`package-naming`:** Do not use user-defined bad names like `foo` or `bar`.
*   **`var-naming`:** Initialisms are strictly checked. `ID` is allowed, but `VM` is denied. Upper-case constants are allowed.
*   **`receiver-naming`:** Receiver names have a maximum length of **2** characters.
*   **`import-alias-naming`:** Must match `^[a-z][a-z0-9]{0,}$`.
*   **`banned-characters`:** The character `7` is banned in identifiers.

## 3. Styling & Code Format Enforcement
*   **`add-constant`:** Maximum **3** instances of the same string literal before requiring a constant. Exceptions: `""`, integers `0,1,2`, and floats `0.0`, `1.0`, `2.0`.
*   **`enforce-map-style`:** Always use `make` for map initialization.
*   **`enforce-repeated-arg-type-style`:** Function arguments must use the `full` style (explicit types), while return values must use the `short` style.
*   **`enforce-slice-style`:** Allowed to use `any` style.
*   **`enforce-switch-style`:** Allows switches with no default clause (`allowNoDefault`).
*   **`early-return`:** Strongly prefer early returns to avoid nesting (`preserve-scope`, `allow-jump`).
*   **`struct-tag`:** Tags validate format. Ignores `validate`, but enforces format for `json,inline` and `bson,outline,gnu`.
*   **`string-format`:** Strict formatting for `core.WriteError.Message`, `fmt.Errorf`, and `panic` (e.g., must not start with a capital letter, must not end in punctuation, must not contain line breaks).

## 4. Security, Scope & Blocklists
*   **`imports-blocklist`:** Never import `crypto/md5` or `crypto/sha1`.
*   **`dot-imports`:** Banned globally, EXCEPT for `github.com/onsi/ginkgo/v2` and `github.com/onsi/gomega`.
*   **`context-as-argument`:** `context.Context` must be the first parameter, though `*testing.T` and `*github.com/user/repo/testing.Harness` are allowed before it.
*   **`defer`:** Beware of gotchas. Warns on `call-chain`, `loop`, `method-call`, `recover`, `immediate-recover`, and `return`.
*   **`error-strings`:** Checks formatting for `xerrors.New` alongside standard errors.
*   **`unhandled-error`:** All returned errors must be handled, EXCEPT for calls to `fmt.Printf`, `fmt.Println`, `fmt.Fprint`, `fmt.Fprintln`, and `fmt.Fprintf`.
*   **`unused-parameter` / `unused-receiver`:** Parameters/Receivers matching `^_` are allowed to be unused.

## 5. Additional Enabled Default Rules
The following structural, bug-prevention, and logical rules are also fully enabled and must not be violated:
*   **Logic & Flow:** `bare-return`, `confusing-results`, `constant-logical-expr`, `deep-exit`, `empty-block`, `identical-branches`, `identical-ifelseif-branches`, `identical-ifelseif-conditions`, `identical-switch-branches`, `identical-switch-conditions`, `if-return`, `indent-error-flow`, `optimize-operands-order`, `superfluous-else`, `unconditional-recursion`, `unnecessary-if`, `unnecessary-stmt`, `unreachable-code`, `useless-break`, `useless-fallthrough`.
*   **Variables & State:** `atomic`, `bool-literal-in-expr`, `datarace`, `epoch-naming`, `increment-decrement`, `modifies-parameter`, `modifies-value-receiver`, `range`, `range-val-address`, `range-val-in-closure`, `redefines-builtin-id`, `string-of-int`, `time-date`, `time-equal`, `time-naming`, `unchecked-type-assertion`, `var-declaration`.
*   **Types & Packaging:** `blank-imports`, `confusing-naming`, `context-keys-type`, `duplicated-imports`, `empty-lines`, `error-naming`, `error-return`, `errorf`, `exported`, `get-return`, `import-shadowing`, `inefficient-map-lookup`, `nested-structs`, `package-comments`, `redundant-build-tag`, `redundant-import-alias`, `redundant-test-main-exit`, `unexported-naming`, `unexported-return`, `unsecure-url-scheme`.
*   **Modern Go Overrides:** `use-any`, `use-errors-new`, `use-fmt-print`, `use-slices-sort`, `use-waitgroup-go`, `waitgroup-by-value`, `forbidden-call-in-wg-go`.

## 6. Proactive Checking

Always run `golangci-lint run` after generating or refactoring Go code to guarantee none of these rules have been violated. Fix every violation before declaring the task done.

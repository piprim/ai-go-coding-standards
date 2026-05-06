---
name: golang-data-structures
description: Choosing and using Go data structures correctly — slices (header/cap/growth/preallocation/`slices` package), maps (hash buckets, `maps` package, `sync.Map`), arrays, container/list/heap/ring, strings.Builder vs bytes.Buffer, generic collections, unsafe.Pointer and weak.Pointer (Go 1.24+), and copy semantics. Trigger this skill when designing or optimizing collections, choosing between slice/map/array, building generic containers, using container/* packages, working with unsafe or weak pointers, or questioning slice/map internals.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: pivaldi
  version: "1.0.0"
  openclaw:
    emoji: "🧱"
allowed-tools: Read Edit Write Glob Grep Agent
---

# Golang Data Structures

Pick the right structure first; optimize the wrong one and you're sanding a tire. This skill is a decision guide and a reminder of the parts everyone forgets.

## 1. Slices

### Header & layout

A slice is a 24-byte (on 64-bit) struct: `{ptr *T, len int, cap int}`. The slice value is *not* the data — slicing or assigning a slice copies the header but shares the backing array.

```go
a := []int{1, 2, 3, 4}
b := a[1:3]   // b shares backing array with a
b[0] = 99
fmt.Println(a) // [1 99 3 4]  ← a was mutated
```

### Capacity growth

`append` grows capacity when `len == cap`. The growth factor is roughly:

- **~2× when small** (cap < ~256)
- **~1.25× when large** (and rounded for memory allocator class sizes)

The exact numbers shift between Go releases. Don't rely on them for correctness; do rely on them for sizing intuition.

### Preallocation — the cheapest performance win

When you know (or can estimate) the final length, preallocate:

```go
// wasteful — reallocates O(log n) times
out := []Row{}
for _, r := range rows { out = append(out, transform(r)) }

// efficient — zero growth allocations
out := make([]Row, 0, len(rows))
for _, r := range rows { out = append(out, transform(r)) }
```

If you'll write by index, preallocate length:

```go
out := make([]Row, len(rows))
for i, r := range rows { out[i] = transform(r) }
```

### The `slices` package (Go 1.21+)

Don't hand-roll the basics. Use `slices.*`:

| Need | Use |
|---|---|
| equality | `slices.Equal(a, b)` |
| contains element | `slices.Contains(s, v)` |
| index of element | `slices.Index(s, v)` |
| sort | `slices.Sort(s)` / `slices.SortFunc(s, cmp)` |
| sort stable | `slices.SortStableFunc(s, cmp)` |
| reverse | `slices.Reverse(s)` |
| copy | `slices.Clone(s)` |
| concat | `slices.Concat(a, b, c)` |
| delete by range | `slices.Delete(s, i, j)` |
| insert | `slices.Insert(s, i, v...)` |
| binary search | `slices.BinarySearch(s, v)` |
| min / max | `slices.Min(s)` / `slices.Max(s)` |

### Subtle pitfalls

- **A slice keeps the entire backing array alive.** Slicing a tiny window of a huge byte slice prevents the rest from being collected. Use `slices.Clone` to detach.
- **`append` may or may not allocate.** Functions that `append` to a parameter slice can mutate the caller's data — pass slices conservatively, or `slices.Clone` first.
- **Range-loop value is a copy.** `for _, v := range users { go save(&v) }` is dangerous pre-1.22; from 1.22 the variable is per-iteration but it's still a copy of the element.

## 2. Maps

### Internals

A Go map is a hash table with bucket arrays. Important consequences:

- **Iteration order is randomized** — never depend on it.
- **Maps are not safe for concurrent reads + writes.** A concurrent write panics ("fatal error: concurrent map writes").
- **Lookup never panics** — missing keys return the zero value with `ok=false`.

### Initialization

```go
// good — explicit capacity hint when known
m := make(map[string]int, expectedSize)

// good — empty map literal
m := map[string]int{}

// bad — nil map cannot be assigned to
var m map[string]int
m["a"] = 1 // panic: assignment to entry in nil map
```

Revive's `enforce-map-style` requires `make(...)` over `map[K]V{}` for empty maps in this project.

### The `maps` package (Go 1.21+)

| Need | Use |
|---|---|
| keys (iter) | `maps.Keys(m)` (returns `iter.Seq[K]`) |
| values (iter) | `maps.Values(m)` |
| collect to slice | `slices.Collect(maps.Keys(m))` |
| copy | `maps.Clone(m)` |
| equality | `maps.Equal(a, b)` |
| zero in place | `clear(m)` (builtin) |

### Concurrent maps

- **Mutex-guarded `map[K]V`** is the simplest, fastest answer for most workloads.
- **`sync.Map`** is for *very specific* patterns: high read-to-write ratio, or write-once-read-many keys. It's slower than a mutex map for general use. Prefer `RWMutex + map` until benchmarks say otherwise.

## 3. Arrays

Arrays in Go are fixed-size value types: `[N]T`. They're rarely the right choice in application code; slices are. Reach for arrays when:

- The size is part of the *type* (cryptographic digests: `[32]byte` for SHA-256).
- You need a value-typed buffer (no heap alloc, copies on assignment).
- You're interfacing with C or a fixed-layout binary format.

```go
var key [32]byte           // fixed-size cryptographic key
copy(key[:], decoded)      // [:] converts array to slice
```

## 4. `container/list`, `container/heap`, `container/ring`

The standard library ships three classic structures that are surprisingly underused:

- **`container/list`** — doubly-linked list. Useful for LRU caches, when you need O(1) splice.
- **`container/heap`** — priority queue (min-heap). Implement the `heap.Interface` (`Len/Less/Swap/Push/Pop`) and call `heap.Init/Push/Pop/Fix`.
- **`container/ring`** — circular list. Useful for fixed-size sliding windows.

These predate generics; you'll use `any` and type-assert. For new code, a generic heap built on `[]T` is often cleaner.

## 5. `strings.Builder` vs `bytes.Buffer`

| Need | Use |
|---|---|
| build a `string` | `strings.Builder` |
| build a `[]byte` | `bytes.Buffer` |
| read what you wrote | `bytes.Buffer` (Builder is write-only) |
| satisfy `io.Writer` | both |

```go
var sb strings.Builder
sb.Grow(estimatedLen)               // preallocate
for _, p := range parts {
    sb.WriteString(p)
    sb.WriteByte('\n')
}
out := sb.String()                  // no copy
```

**Never use `+`-concatenation in a loop.** Each `+` allocates a new string and copies both sides — O(n²) work for n parts.

## 6. Generic collections (Go 1.18+)

Define your own type-safe containers when stdlib doesn't fit:

```go
type Stack[T any] struct{ items []T }

func (s *Stack[T]) Push(v T)        { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool)  {
    if len(s.items) == 0 {
        var zero T

        return zero, false
    }

    v := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]

    return v, true
}
```

Constraints (`comparable`, `cmp.Ordered`, custom interface unions) keep these honest.

## 7. Pointers — `unsafe.Pointer` and `weak.Pointer`

### `unsafe.Pointer`

Last-resort tool for FFI, type punning, and certain serializers. **Avoid in application code.** Every use:

- Bypasses the type system → no compile-time guarantees.
- Is ABI-coupled → may break across Go versions.
- Demands the rules in `unsafe.Pointer`'s package doc be followed *exactly* (six allowed conversion patterns).

If you reach for `unsafe`, explore `reflect`, generics, or stdlib helpers first.

### `weak.Pointer` (Go 1.24+)

A reference that does **not** keep its target alive. Useful for caches that should not extend lifetime:

```go
type Cache struct {
    items map[string]weak.Pointer[Item]
}
```

If the only references to an `Item` are weak pointers, the GC may collect it. Read with `wp.Value()` — returns `*T` or `nil`.

## 8. Copy semantics — what shares, what copies

| Type | Assignment | Function arg | Range loop |
|---|---|---|---|
| Slice | header copied, **data shared** | header copied, **data shared** | element is **copy** |
| Map | reference copied, **data shared** | reference copied, **data shared** | element is **copy** |
| Channel | reference copied, **data shared** | reference copied, **data shared** | n/a |
| Array | **full copy** | **full copy** | element is copy |
| Struct | **full copy** | **full copy** (unless pointer) | element is copy |
| Pointer | reference copied, **data shared** | reference copied, **data shared** | n/a |

Mental model: slices, maps, channels, and pointers are *handles* — copying the handle doesn't copy the data. Arrays and structs are *values* — copying copies everything.

**Common bug:** mutating a struct returned from a slice element doesn't persist:

```go
type Item struct{ Count int }

items := []Item{{Count: 1}}
for _, it := range items {
    it.Count++           // mutates the copy
}
fmt.Println(items[0])    // {Count: 1}  ← unchanged

// fix — index, or use slice of pointers
for i := range items { items[i].Count++ }
```

## How to apply (checklist)

1. Building a slice in a loop → `make([]T, 0, n)` if you can estimate `n`.
2. Looking for an element → check the `slices` package before writing a loop.
3. Empty map → `make(map[K]V)` (or with capacity hint), never bare `map[K]V{}` (revive enforces).
4. Concurrent map → `RWMutex + map[K]V` first; `sync.Map` only for proven read-heavy / write-once patterns.
5. Building a string → `strings.Builder` with `Grow`; never `+=` in a loop.
6. Reaching for `unsafe.Pointer` → stop, see if generics or `reflect` solve it.
7. Need a non-extending reference → `weak.Pointer[T]` (Go 1.24+).
8. Returning a struct from a slice and mutating it → index, don't iterate by value.

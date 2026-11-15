---
name: go-modern
description: "Go 1.23-1.27 features and deprecations that pre-2026-cutoff models don't know or get wrong. Use when writing Go code, go.mod files, tests, or tooling setup. Covers range-over-func iterators, the go.mod tool directive, generic type aliases, os.Root, testing/synctest, new(expr), errors.AsType, sync.WaitGroup.Go, and traps like the no-op rand.Seed and the tools.go hack."
---

# Go 1.23 → 1.27

Latest stable: **1.27.4** (patched 2026-11-10). 1.26 (2026-02) and 1.25 (2025-08) shipped the largest post-cutoff batches.

## Stale → current

| Stale (old models still emit this) | Current |
| --- | --- |
| `tools.go` blank-import hack for CLI deps | `tool` directive in go.mod (1.24): `go get -tool <pkg>`, run with `go tool <name>` |
| `rand.Seed(time.Now().UnixNano())` | **No-op since 1.24.** Use `math/rand/v2` (1.22) or `crypto/rand` |
| `runtime.SetFinalizer` | `runtime.AddCleanup` (1.24) |
| `for i := 0; i < b.N; i++` | `for b.Loop() { ... }` (1.24) |
| `wg.Add(1); go func(){ defer wg.Done(); ... }` | `wg.Go(func() { ... })` (1.25) |
| Sleep/timeouts in async tests | `testing/synctest` (GA 1.25) — virtualized time bubbles |
| `x := &T{...}` / `new(T)` dance for computed init | `new(expr)` (1.26): `p := new(yearsSince(born))` |
| `errors.As` + type-switch ceremony | `errors.AsType[MyErr](err)` (1.26) |
| Manual `GOMAXPROCS` tuning in containers | Container-aware, auto-updating since 1.25 |

## Range-over-func iterators (1.23)

```go
import "iter"

func Backward[E any](s []E) iter.Seq2[int, E] {
    return func(yield func(int, E) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(i, s[i]) {
                return
            }
        }
    }
}
for i, v := range Backward(s) { ... }
// valid signatures: func(func() bool) | func(func(K) bool) | func(func(K, V) bool)
```
Iterator stdlib since 1.23/1.24: `slices.All/Values/Backward/Collect/Sorted/Chunk`, `maps.Keys/All/Insert/Collect`, `strings.Lines`, `strings.SplitSeq`.

## tool directive (1.24)

```sh
go get -tool golang.org/x/tools/cmd/stringer
# go.mod gains: tool golang.org/x/tools/cmd/stringer
go tool stringer
```

## Generic type aliases (1.24)

```go
type Map[K comparable, V any] = map[K]V
```

## os.Root (1.24, expanded 1.25) — path-confined FS access

```go
root, err := os.OpenRoot("/srv/data")
f, err := root.Open("users/alice.txt")  // symlink-escape-proof
fs := root.FS()                         // implements fs.FS
// 1.25 added: ReadFile, WriteFile, MkdirAll, RemoveAll, Rename, Stat, ...
```

## testing/synctest (GA 1.25)

```go
func TestCache(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        go expireAfter(c, time.Hour)
        synctest.Wait()   // returns once ALL bubble goroutines are blocked; virtual clock jumps
        // assert expired state — instant, no sleeps, no flakiness
    })
}
```

## 1.26 additions

```go
p := new(42)   // *int — new(expr) accepts expressions
if e, ok := errors.AsType[MyErr](err); ok { ... }
type Adder[A Adder[A]] interface { Add(A) A }   // self-referential generic types
```
Also: Green Tea GC default (less GC overhead), rewritten `go fix` with "modernizers" that mechanically update old idioms, `slog.NewMultiHandler`, `crypto/hpke`.

## More post-cutoff stdlib, quickly

- 1.23: `unique` (value interning); timer channels unbuffered when go.mod ≥ 1.23 — `Stop`/`Reset` guarantee no stale values.
- 1.24: `weak`; `crypto/mlkem`, `crypto/hkdf`, `crypto/pbkdf2`, `crypto/sha3`; `omitzero` JSON tag; Swiss-table maps; `crypto/rand.Read` never fails (error check unnecessary); generic type aliases.
- 1.25: `encoding/json/v2` behind `GOEXPERIMENT=jsonv2`; go.mod `ignore` directive; `net/http.CrossOriginProtection` (CSRF); `slog.GroupAttrs`; `runtime/trace.FlightRecorder`.

## Deprecations/removals to stop emitting

- `crypto/cipher`: `NewOFB`, `NewCFB*` deprecated (1.24) — use AEAD or CTR.
- `crypto/x509`: SHA-1 signatures unsupported; RSA keys < 1024 bits rejected.
- `http.ReverseProxy.Director` deprecated (1.26) → `Rewrite`.
- TLS: 3DES dropped from defaults; SHA-1 handshakes disallowed (1.25); TLS < 1.2 defaults tightening (1.27).
- `crypto/rsa`: PKCS#1 v1.5 **encryption** deprecated (1.26).
- 1.26: `random` args of `GenerateKey`/`Sign*` in ecdsa/ed25519/rsa/ecdh are ignored (global random is injectable for tests only).

## Sources

- https://go.dev/doc/devel/release (1.27.4 current, fetched 2026-11-15)
- https://go.dev/doc/go1.23 (2024-08) · https://go.dev/doc/go1.24 (2025-02) · https://go.dev/doc/go1.25 (2025-08) · https://go.dev/doc/go1.26 (2026-02) · https://go.dev/blog/go1.26

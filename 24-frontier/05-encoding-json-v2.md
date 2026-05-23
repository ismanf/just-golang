# encoding/json/v2 — Experimental → Stable in 1.26

## TL;DR

**`encoding/json/v2`** is a long-planned redesign of Go's JSON encoder/decoder that lands as **stable in Go 1.26** after years as `GOEXPERIMENT=jsonv2`. Authored primarily by **Joe Tsai** ([proposal #71497](https://github.com/golang/go/issues/71497)), v2 fixes nearly every long-standing complaint about the stdlib's `encoding/json`: streaming I/O, configurable behavior via options, much faster default performance, stricter handling of duplicates and overflow, better support for `time.Time` and friends. Crucially, **`encoding/json` v1 stays** — v2 is a *new* package, not a replacement. The single biggest gotcha: **v2 changes default semantics**. JSON that v1 accepted may be rejected by v2 (duplicate keys, malformed UTF-8). Migration is per-package, not automatic.

## Mental Model

```
   encoding/json (v1, unchanged):
       Marshal/Unmarshal whole values.
       Reflection-driven; slow on hot paths.
       Lenient by default (allows duplicates, partial reads, etc.).
       
   encoding/json/v2 (new in 1.26):
       MarshalEncode/UnmarshalDecode for streaming.
       Decoder/Encoder for low-level access.
       Options as first-class arguments: `jsonv2.RejectUnknownMembers`,
       `jsonv2.WithMarshalers`, ...
       Faster by ~2× on representative benchmarks.
       Stricter by default.

   v1 and v2 coexist:
       import "encoding/json"           // legacy, unchanged
       import "encoding/json/v2"        // new
       import "encoding/json/jsontext"  // low-level tokenizer for streaming
```

The split: `json/v2` is the public API; `json/jsontext` is the underlying token-level processor that both can build on.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"os"

	"encoding/json/v2"
)

type User struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Age   int    `json:"age,omitempty"`
}

func main() {
	// Marshal
	u := User{Name: "Alice", Email: "a@example.com", Age: 30}
	b, err := json.Marshal(u)
	if err != nil { panic(err) }
	fmt.Println(string(b))
	// {"name":"Alice","email":"a@example.com","age":30}

	// Unmarshal
	var u2 User
	if err := json.Unmarshal(b, &u2); err != nil { panic(err) }
	fmt.Printf("%+v\n", u2)

	// Streaming encode
	enc := json.NewEncoder(os.Stdout)
	if err := enc.Encode(u); err != nil { panic(err) }
}
```

API is intentionally close to v1 for ease of migration; behavior differs in stricter defaults and per-call options.

## Deep Dive

### History

- **2020**: Joe Tsai begins design notes for v2.
- **2021**: First public proposal, [#63397](https://github.com/golang/go/issues/63397) (later superseded).
- **2022-2024**: Iterations under `github.com/go-json-experiment/json` (out-of-tree).
- **Go 1.21+**: included as `GOEXPERIMENT=jsonv2`.
- **Go 1.26**: stabilized as `encoding/json/v2` and `encoding/json/jsontext`.

The experimental package logged ~3 years of production use by early adopters before stabilization.

### Why v1 wasn't enough

`encoding/json` has known issues:

1. **Reflection-heavy hot path**: slow vs hand-written or generated codecs.
2. **Lenient parsing**: duplicate keys silently accepted; invalid UTF-8 sometimes coerced.
3. **No streaming for primitives**: `Encoder.Encode` works on whole values; no token-level read/write.
4. **Limited configuration**: `Encoder.SetIndent`, `Encoder.SetEscapeHTML`, and a few struct tags. Most behavior is hardcoded.
5. **`time.Time` quirks**: forces RFC 3339, no easy override.
6. **Inconsistent `json:"-"`**: hides field; but `json:""` is also valid and means the field uses its Go name.
7. **`omitempty` is broad-brush**: applies to zero values; can't customize per type.

v2 addresses each.

### Per-call options

v2's distinguishing feature: behavior is configurable per `Marshal`/`Unmarshal` call.

```go
import "encoding/json/v2"

opts := json.JoinOptions(
    json.Deterministic(true),                   // sort map keys
    json.WithMarshalers(json.MarshalToFunc[time.Time](func(t time.Time) ([]byte, error) {
        return []byte(t.Format("2006-01-02")), nil
    })),
)

b, _ := json.Marshal(value, opts)
```

Composable; no global state.

### `jsontext` — low-level tokenizer

```go
import "encoding/json/jsontext"

dec := jsontext.NewDecoder(reader)
for {
    tok, err := dec.ReadToken()
    if err == io.EOF { break }
    if err != nil { return err }
    switch tok.Kind() {
    case '{', '[': /* start */
    case '}', ']': /* end */
    case '"': /* string */ s := tok.String()
    case 't', 'f': /* bool */
    case 'n': /* null */
    case '0': /* number */
    }
}
```

Pull-based, allocation-aware. Lets you build custom decoders that bypass the reflection layer.

### Defaults that changed

| Behavior | v1 | v2 |
|---|---|---|
| Duplicate keys | accepted | error |
| Invalid UTF-8 in strings | coerced/replaced | error |
| Missing required fields | only with `omitempty` reverse | configurable |
| Map key ordering | unspecified | deterministic in some modes |
| Large numbers (int64-overflow JSON) | lossy | error |
| `\u` escapes for HTML | always for `<`, `>`, `&` | configurable |
| Number parsing on string fields | sometimes lenient | strict |

Migration: run v2 on your existing JSON corpus; some inputs may now error. Most fixes are simple (clean data, configure leniency where needed).

### Performance

Joe Tsai's benchmarks (vary by workload):

- v2 marshal: ~2× faster than v1.
- v2 unmarshal: ~2-3× faster than v1.
- v2 with `jsontext` direct: 5-10× faster than v1 for hot paths.
- Allocations per call: roughly half.

The improvements come from:
- Better struct-field caching.
- Less reflection per call.
- Lazy decoding.
- SIMD-friendly UTF-8 validation.

### Custom marshalers

```go
type Date time.Time

func (d Date) MarshalJSONv2(enc *jsontext.Encoder, opts json.Options) error {
    return enc.WriteToken(jsontext.String(time.Time(d).Format("2006-01-02")))
}

func (d *Date) UnmarshalJSONv2(dec *jsontext.Decoder, opts json.Options) error {
    tok, err := dec.ReadToken()
    if err != nil { return err }
    t, err := time.Parse("2006-01-02", tok.String())
    if err != nil { return err }
    *d = Date(t)
    return nil
}
```

The v2 interface methods take `jsontext.Encoder/Decoder` instead of `[]byte`. Lets you read/write tokens incrementally; saves allocations.

Backward compatibility: types implementing v1's `MarshalJSON` still work in v2 (v2 falls back). New code should implement `MarshalJSONv2`.

### Streaming a large array

```go
import (
    "encoding/json/v2"
    "encoding/json/jsontext"
    "io"
)

func writeUsers(w io.Writer, users <-chan User) error {
    enc := jsontext.NewEncoder(w)
    if err := enc.WriteToken(jsontext.BeginArray); err != nil { return err }
    for u := range users {
        if err := json.MarshalEncode(enc, u); err != nil { return err }
    }
    return enc.WriteToken(jsontext.EndArray)
}
```

Write items one at a time; total memory is per-item not per-array.

### Backward-compat aliases

`encoding/json` (v1) is unchanged. To migrate gradually:

```go
import (
    json1 "encoding/json"
    json2 "encoding/json/v2"
)

// Use both during migration
```

Or alias at package level:

```go
// In your code, swap one package import at a time
import "encoding/json/v2" // formerly: import "encoding/json"
```

### v2 struct tags

v2 introduces new tag values:

- `json:",string"` — encode/decode as string (v1 had this too).
- `json:",omitzero"` — omit when value is type's zero (v2 only; clearer than `omitempty`).
- `json:",inline"` — embed without nesting.
- `json:",unknown"` — catch-all field for unrecognized keys.

```go
type Config struct {
    Name    string         `json:"name"`
    Extra   map[string]any `json:",unknown"`
    Inline  shared         `json:",inline"`
}
```

### `omitempty` vs `omitzero`

```go
type T struct {
    Tags []string `json:",omitempty"`   // omit if nil OR empty
    Name string   `json:",omitzero"`    // omit if exactly the zero value
}
```

`omitempty` (v1 and v2) skips zero-length collections.
`omitzero` (v2) skips exactly Go's zero value (for `time.Time`, this is `IsZero()`).

For `time.Time`, `omitzero` is what most people want.

### Marshaler / Unmarshaler interface

```go
// v1:
type Marshaler interface { MarshalJSON() ([]byte, error) }
type Unmarshaler interface { UnmarshalJSON([]byte) error }

// v2:
type Marshaler interface { MarshalJSON([]byte) ([]byte, error) }   // appends
type Unmarshaler interface { UnmarshalJSON([]byte) error }
// plus
type MarshalerV2 interface {
    MarshalJSONv2(*jsontext.Encoder, Options) error
}
type UnmarshalerV2 interface {
    UnmarshalJSONv2(*jsontext.Decoder, Options) error
}
```

v2 supports both. Old code keeps working.

### `WithMarshalers` / `WithUnmarshalers`

Configure handling per type without changing the type:

```go
opts := json.WithMarshalers(json.MarshalToFunc[time.Duration](func(d time.Duration) ([]byte, error) {
    return []byte(`"` + d.String() + `"`), nil
}))

json.Marshal(value, opts)
```

`time.Duration` is rendered as `"5s"` instead of nanoseconds. No struct tag needed.

### Removed lenient behaviors

v2 rejects what v1 accepted:

- `null` into a numeric field → error (v1 zeroed).
- `"123"` into an int field without `,string` tag → error.
- Top-level number 0x7FFFFFFFFFFFFFFF + 1 → error (overflow).
- Duplicate JSON object keys → error.

For migration, audit your data sources; add `json.AllowDuplicateNames(true)` or similar opt-in lenience where required.

### Custom error reporting

v2 errors carry richer info:

```go
err := json.Unmarshal([]byte(`{"a":1,"a":2}`), &v)
// err: "duplicate name `a` at offset 11"
```

Errors include byte offsets, field paths, and types.

### Migrating a service

1. Bump `go` directive in `go.mod` to 1.26.
2. Change one import: `"encoding/json"` → `"encoding/json/v2"`.
3. Run tests. Fix failures (often around dup keys, time formats, missing fields).
4. Profile; observe ~2× throughput improvement on JSON-heavy paths.
5. Repeat per package.

Most migrations are one-day jobs for medium services.

## Standard Library Hooks

- `encoding/json/v2` — main package.
- `encoding/json/jsontext` — token-level streaming.
- `encoding/json` — v1 (still works, unchanged).
- Common patterns: `json.Marshal`, `json.Unmarshal`, `json.NewEncoder`, `json.NewDecoder`.
- v2 additions: `json.MarshalToFunc`, `json.UnmarshalFromFunc`, `json.JoinOptions`, etc.
- v2 options: `json.Deterministic`, `json.AllowDuplicateNames`, `json.AllowInvalidUTF8`, `json.RejectUnknownMembers`.

## Real-World Patterns

### 1. Switch all imports

```go
// Before
import "encoding/json"

// After
import "encoding/json/v2"
```

Most call sites are source-compatible.

### 2. Custom time format

```go
opts := json.WithMarshalers(json.MarshalToFunc[time.Time](func(t time.Time) ([]byte, error) {
    return []byte(`"` + t.UTC().Format("2006-01-02T15:04:05Z") + `"`), nil
}))

b, _ := json.Marshal(record, opts)
```

No struct tag needed; opt-in formatter for the call.

### 3. Streaming decode

```go
dec := json.NewDecoder(resp.Body)
for {
    var item Item
    if err := dec.Decode(&item); err == io.EOF { break } else if err != nil {
        return err
    }
    process(item)
}
```

Same API as v1; per-Item allocation. Use `jsontext.Decoder` for tokens if you need lower overhead.

### 4. Strict decoding for APIs

```go
opts := json.JoinOptions(
    json.RejectUnknownMembers(true),
    json.RejectDuplicateNames(true),
)

if err := json.Unmarshal(body, &req, opts); err != nil {
    http.Error(w, err.Error(), 400)
    return
}
```

API rejects misformed payloads outright.

### 5. Catch-all unknown fields

```go
type Config struct {
    Name  string         `json:"name"`
    Other map[string]any `json:",unknown"`
}
```

Unknown JSON keys land in `Other`; useful for forward-compat config.

### 6. v2 + iter for huge arrays

```go
dec := jsontext.NewDecoder(r)
dec.ReadToken()  // '['

for dec.PeekKind() != ']' {
    var item Item
    if err := json.UnmarshalDecode(dec, &item); err != nil { return err }
    process(item)
}
```

Token-level array streaming. No intermediate slice.

## Anti-Patterns & Gotchas

**Treating v2 as drop-in for v1.** Some payloads will error in v2. Test.

**Using `json.Unmarshal` for huge documents.** Use `Decoder` (streaming).

**Setting `AllowDuplicateNames(true)` globally.** Defeats v2's strictness. Use only where needed.

**Migrating without checking benchmarks.** Performance gains are real but workload-dependent.

**Ignoring v1 → v2 behavior diffs in error messages.** Some err strings change; tests that match err text break.

**Mixing v1 `MarshalJSON` with v2 options.** v2 calls the v1 method as fallback; if the v1 method ignores options, you get inconsistent output.

**Trying to share Marshaler implementations across packages with both v1 and v2 callers.** Define both `MarshalJSON` and `MarshalJSONv2`.

**Forgetting `time.Time` defaults.** v2 honors `IsZero()` for omitzero; if your code relied on `omitempty` for `time.Time`, behavior may differ.

**Importing `json/v2` without bumping `go` directive.** `go vet` complains.

**Calling `jsontext.Decoder.ReadToken` after EOF**. Returns `io.EOF` repeatedly; loop with `err == io.EOF { break }`.

## Performance Notes

Joe Tsai's published benchmarks (v2 vs v1, vary by workload):

- Marshal small struct: ~2× faster.
- Unmarshal small struct: ~2-3× faster.
- Marshal large array: ~1.5× faster.
- Unmarshal stream: 3-5× faster.
- Token-level (jsontext): 5-10× faster for hot paths.
- Allocations: roughly half.

Compared to third-party libs:
- `json-iterator/go`: v2 catches up or matches.
- `goccy/go-json`: still slightly faster on some benchmarks.
- `sonic` (Bytedance): SIMD-heavy; faster than v2 for x86_64 specifically.

For most apps, v2 is fast enough.

## How Big Companies Use It

v2 ships in 1.26 (early 2026); production adoption began with the experimental package:

- **Google internal**: experimental jsonv2 in production for over a year before GA.
- **Joe Tsai's employer (Google Cloud)**: heavy user of the experimental version.
- **Several startups** running on `GOEXPERIMENT=jsonv2`: documented at Gophercon talks.
- **The Go module proxy**: uses jsonv2 patterns for tooling output.
- **Tailscale**: documented evaluation.

Post-1.26, broad adoption is expected.

## Source Code References

Pinned to `go1.26`.

- `encoding/json/v2`: [`src/encoding/json/v2/`](https://github.com/golang/go/tree/release-branch.go1.26/src/encoding/json/v2).
- `encoding/json/jsontext`: [`src/encoding/json/jsontext/`](https://github.com/golang/go/tree/release-branch.go1.26/src/encoding/json/jsontext).
- Experimental out-of-tree fork: [`go-json-experiment/json`](https://github.com/go-json-experiment/json).
- Proposal #71497: https://github.com/golang/go/issues/71497.
- Original design doc: https://go.googlesource.com/proposal/.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "json/v2: A successor to encoding/json" — Joe Tsai talks at GopherCon.
- Proposal #71497 (final stable): https://github.com/golang/go/issues/71497.
- Experimental package docs: https://github.com/go-json-experiment/json.
- Russ Cox commentary on v2: research.swtch.com posts.
- Joe Tsai's design notes (in the experimental repo's docs).
- "Benchmarking JSON in Go" — multiple community posts.
- Bytedance Sonic comparison: https://github.com/bytedance/sonic.
- "Why we wrote a JSON parser from scratch" — Sonic team blog.

## Exercises / Self-Check

1. Migrate a small Go service from `encoding/json` to `encoding/json/v2`. Document the behavior diffs you encountered.
2. Implement `MarshalJSONv2`/`UnmarshalJSONv2` for a custom date type that uses YYYY-MM-DD format.
3. Use `jsontext.Decoder` to stream-parse a 1 GiB JSON array without ever holding more than one item in memory.
4. Benchmark `json.Marshal` (v1 vs v2) on a representative struct. Quantify the speedup.
5. With `json.RejectDuplicateNames(true)`, construct a JSON document that v1 silently accepts but v2 rejects. Why is the new default safer?

# `encoding/json` (and `encoding/json/v2` in 1.26)

## TL;DR

`encoding/json` marshals/unmarshals Go values to/from JSON via reflection plus struct tags. `Marshal`/`Unmarshal` operate on bytes; `NewEncoder`/`NewDecoder` stream over `io.Writer`/`io.Reader`. The 1.26 `encoding/json/v2` (experimental for several releases, stabilized in 1.26) fixes long-standing wart areas: faster, deterministic field ordering, configurable behavior, no surprise reflection of unexported fields. New code on 1.26+ should consider `/v2`; existing code keeps `/v1` working.

## Mental Model

```
Marshal(v):
  reflect on v → walk fields → emit JSON tokens → []byte

Unmarshal(data, &v):
  parse JSON tokens → reflect on *v → set fields

Struct tags drive name/omit/option:
  type T struct {
      Name string `json:"name"`
      Age  int    `json:"age,omitempty"`
      Hidden string `json:"-"`
  }
```

## Syntax & Basic Usage

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
	Pwd  string `json:"-"`
}

func main() {
	u := User{ID: 1, Name: "Ada", Pwd: "secret"}
	b, _ := json.Marshal(u)
	fmt.Println(string(b))

	var v User
	json.Unmarshal([]byte(`{"id":2,"name":"Linus"}`), &v)
	fmt.Println(v)
	// Output:
	// {"id":1,"name":"Ada"}
	// {2 Linus }
}
```

## Deep Dive

### Struct tags

```go
type Item struct {
	Name   string  `json:"name"`              // rename
	Price  float64 `json:"price,omitempty"`   // omit if zero
	Hidden string  `json:"-"`                  // never serialize
	Raw    string  `json:",string"`            // wrap value in JSON string
}
```

`omitempty` skips empty values (zero, nil, empty slice/map/string). `json:"-"` excludes entirely. Comma options are `omitempty`, `string`.

In v2, `omitzero` (more precise than `omitempty`) and `format:"RFC3339"` for time become available.

### Streaming

```go
dec := json.NewDecoder(r)
for dec.More() {
	var item Item
	if err := dec.Decode(&item); err != nil { return err }
	process(item)
}
```

Use for large arrays where loading everything into memory is wasteful.

```go
enc := json.NewEncoder(w)
enc.SetIndent("", "  ")
enc.Encode(value) // adds a trailing newline
```

### `Marshal*` variants

- `Marshal(v)` — bytes.
- `MarshalIndent(v, prefix, indent)` — pretty-printed.
- `RawMessage` — `[]byte` that's already valid JSON; useful for delayed decoding or passthrough.
- `Number` — preserves precision for JSON numbers (use `UseNumber()` on decoder).

### Custom marshaling

```go
type Money int64

func (m Money) MarshalJSON() ([]byte, error) {
	return []byte(fmt.Sprintf(`"%d.%02d"`, m/100, m%100)), nil
}
func (m *Money) UnmarshalJSON(data []byte) error {
	// parse "12.34"
}
```

Or implement `encoding.TextMarshaler` / `TextUnmarshaler` — used when the value is a map key or appears in places JSON marshalers don't apply.

### Decoding into `any`

```go
var v any
json.Unmarshal(data, &v)
// v is map[string]any for objects, []any for arrays, float64 for numbers,
// string, bool, nil.
```

Numbers default to `float64`. For precise integer handling:

```go
dec := json.NewDecoder(r)
dec.UseNumber()
// numbers become json.Number; call .Int64() / .Float64() / .String()
```

### Strict decoding

```go
dec := json.NewDecoder(r)
dec.DisallowUnknownFields() // error if JSON has fields not in struct
err := dec.Decode(&v)
```

### Reflection cost and the tag cache

`encoding/json` caches reflection info per struct type. First marshal of a new type allocates the metadata; subsequent calls reuse. Hot paths still pay for reflection traversal — for ultimate speed use code generators (`easyjson`, `sonic`, `goccy/go-json`).

### Field ordering

`/v1` orders by struct field declaration order. `/v2` (1.26) keeps this default but also supports stable ordering of map keys (`/v1` already sorts map keys alphabetically).

### Embedded structs

```go
type Base struct{ ID int `json:"id"` }
type Doc struct {
	Base
	Title string `json:"title"`
}
// Marshal(Doc{Base: Base{1}, Title: "hi"}) → {"id":1,"title":"hi"}
```

Embedded fields are promoted at the top level. Use `json:"-"` to exclude or `json:"base"` to nest.

### Time

`time.Time.MarshalJSON` outputs RFC3339Nano. Custom layouts require a wrapper type or the v2 `format` option.

### `RawMessage` passthrough

```go
type Envelope struct {
	Type string          `json:"type"`
	Body json.RawMessage `json:"body"` // not parsed yet
}

var e Envelope
json.Unmarshal(data, &e)
switch e.Type {
case "user":
	var u User; json.Unmarshal(e.Body, &u)
}
```

Use case: tagged unions over JSON.

### `encoding/json/v2` highlights (1.26)

- Major speedup (often 2-3×).
- Fewer allocations.
- Configurable options (`json.Options` interface).
- Better handling of unknown fields.
- `omitzero` for non-empty-but-zero cases (e.g., `time.Time` is "zero" but not "empty").
- Bytes-by-default API; reflection avoided where possible.

Migration: v1 code compiles unchanged; v2 lives at `encoding/json/v2` and `encoding/json/jsontext` (low-level tokens).

## Standard Library Hooks

- `encoding.TextMarshaler` / `TextUnmarshaler` for types appearing in map keys.
- `io.Reader`/`Writer` for streaming.
- `time.Time` — built-in RFC3339Nano support.

## Real-World Patterns

### 1. HTTP JSON API handler

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	var u User
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	if err := dec.Decode(&u); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest); return
	}
	saved, err := svc.Create(r.Context(), u)
	if err != nil { http.Error(w, err.Error(), http.StatusInternalServerError); return }
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(saved)
}
```

### 2. Streaming large arrays

```go
dec := json.NewDecoder(file)
// Read opening '['
if _, err := dec.Token(); err != nil { return err }
for dec.More() {
	var item Item
	if err := dec.Decode(&item); err != nil { return err }
	process(item)
}
dec.Token() // consume closing ']'
```

Use case: ingesting multi-gigabyte JSON dumps.

### 3. Tagged-union with `RawMessage`

```go
type Event struct {
	Type string          `json:"type"`
	Data json.RawMessage `json:"data"`
}

func dispatch(e Event) error {
	switch e.Type {
	case "click":
		var c Click; if err := json.Unmarshal(e.Data, &c); err != nil { return err }
		return handleClick(c)
	case "purchase":
		var p Purchase; if err := json.Unmarshal(e.Data, &p); err != nil { return err }
		return handlePurchase(p)
	}
	return fmt.Errorf("unknown event %q", e.Type)
}
```

### 4. Custom marshalers for domain types

```go
type Duration time.Duration
func (d Duration) MarshalJSON() ([]byte, error) {
	return []byte(`"` + time.Duration(d).String() + `"`), nil
}
func (d *Duration) UnmarshalJSON(data []byte) error {
	s, err := strconv.Unquote(string(data))
	if err != nil { return err }
	v, err := time.ParseDuration(s)
	*d = Duration(v); return err
}
```

Use case: config files with human-readable durations.

### 5. Map with stable encoding

```go
// json.Marshal sorts map[string]any keys alphabetically — already deterministic.
type Config struct {
	Flags map[string]bool `json:"flags"`
}
```

Use case: golden-file tests where output must be byte-stable.

## Anti-Patterns & Gotchas

**Using `Marshal` on a struct with unexported fields and being surprised they're omitted.** Only exported fields are marshaled. (`/v2` makes this stricter.)

**Decoding numbers as `float64` for IDs.** Loses precision above 2^53. Use `json.Number` or define an int field.

**`json.Unmarshal` into a nil destination.** Panics; pass `&v`, not `v`.

**Using `Marshal` then `Unmarshal` to "deep copy".** Slow and loses information (channels, funcs, unexported fields).

**Forgetting `omitempty` on optional fields.** Zero values serialize as `0`, `""`, `null`.

**`Marshal` failing on a type with a `Marshaler` that returns invalid JSON.** Test your custom marshalers.

**Decoding into `interface{}` and pattern-matching with type switches everywhere.** Define structs.

**`DisallowUnknownFields()` in a backward-compatible API.** Breaks clients on the next field addition.

**Streaming decoder + remaining bytes not consumed.** `dec.More()` checks for more elements; you must consume the closing token if you opened.

**Using `json.RawMessage` value type for a long-lived value.** It's a `[]byte` aliasing the input; if the input is reused, your raw message changes underneath.

## Performance Notes

- `Marshal` allocates the output `[]byte`. `Encoder.Encode` writes directly to the `Writer`.
- Reflection cache: first-time-per-type cost; reuse pays back over many encodes.
- `easyjson`/`sonic`/`go-json` are 3-10× faster than stdlib for tight code paths; trade dependency + codegen for speed.
- `/v2` closes much of that gap.
- For maximum perf: pre-allocate destination structs in a `sync.Pool`, use `Encoder` with a buffered writer.

## How Big Companies Use It

- **Kubernetes** uses `encoding/json` for API server marshaling; performance pain led to investment in stdlib improvements.
- **Cockroach** uses stdlib JSON for HTTP admin APIs; internal serialization uses Protobuf.
- **Discord** moved to `goccy/go-json` for hot serialization paths.
- **Cloudflare** uses streaming `Decoder` for analytics ingestion (avoiding multi-GB buffers).
- **Tailscale** uses `encoding/json` for control-plane responses and ships a custom decoder for hot paths.

## Source Code References

Pinned to `go1.26`.

- `encoding/json`: [`src/encoding/json/`](https://github.com/golang/go/tree/master/src/encoding/json).
- `encoding/json/v2` (1.26): [`src/encoding/json/v2/`](https://github.com/golang/go/tree/master/src/encoding/json/v2).
- `encoding/json/jsontext` (1.26): [`src/encoding/json/jsontext/`](https://github.com/golang/go/tree/master/src/encoding/json/jsontext).
- Reflection cache in `encode.go`, `decode.go`.

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/encoding/json.
- Go blog, "JSON and Go": https://go.dev/blog/json.
- v2 proposal: https://github.com/golang/go/issues/63397.
- Vitess engineering, "Faster JSON in Go": https://vitess.io/blog/.

## Exercises / Self-Check

1. Decode a JSON document into `any`. Recurse through and print key paths.
2. Implement a `Money` type with custom `MarshalJSON`/`UnmarshalJSON`.
3. Stream a 1 GB JSON array of objects; show that memory stays bounded.
4. Build a tagged-union decoder using `json.RawMessage`.
5. Benchmark `encoding/json` v1 vs v2 on a 10 KB struct. Where does v2 win biggest?

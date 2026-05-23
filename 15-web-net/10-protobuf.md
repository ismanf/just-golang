# Protocol Buffers — `google.golang.org/protobuf`

## TL;DR

Protocol Buffers ("protobuf") is Google's open-source IDL and wire format: define messages in `.proto` files, generate code in 10+ languages, serialize compactly to binary. Two Go module families exist: the **old** `github.com/golang/protobuf` (v1, deprecated since 2020) and the **modern** `google.golang.org/protobuf` (v2, the only one you should use). The mental model: **messages are pure data**, the generated Go types have field tags that drive serialization, and three numeric IDs ("field numbers") are the schema's anchor — change a field's number and you've broken every old client and every stored record. Go 1.26's protobuf-go v1.36+ ships the **opaque API** (struct fields are private; getters/setters only), which finally fixes presence/default-value confusion and trades a bit of ergonomic awkwardness for correctness. Five disciplines: **schema versioning** (add fields, never repurpose field numbers; mark removed with `reserved`); **buf** for tooling (linting, breaking-change detection, code-gen orchestration); **field presence** (`optional` keyword in proto3, restored after a 5-year absence; lets you distinguish "zero" from "not set"); **well-known types** (`Timestamp`, `Duration`, `Any`, `Empty`, `Struct`); and **wire-compatibility** as a contract (never break it without a major-version bump). The single biggest gotcha that bites teams who came from JSON: **proto3 default values are indistinguishable from "not set"** unless you use `optional` — a `bool` field that's `false` may mean "explicitly false" or "no value sent."

## Mental Model

```
   .proto file (IDL)
   ─────────────────
   message User {
     string  id    = 1;
     string  email = 2;
     int32   age   = 3;
   }
        │
        │ protoc + protoc-gen-go
        ▼
   Generated Go file
   ──────────────────
   type User struct {
       Id    string  // field number 1
       Email string  // field number 2
       Age   int32   // field number 3
       // + protobuf-internal fields
   }
   func (m *User) Reset() { ... }
   func (m *User) String() string { ... }
   func (m *User) ProtoReflect() protoreflect.Message { ... }
        │
        │ proto.Marshal(u) / proto.Unmarshal(b, u)
        ▼
   Binary on the wire
   ──────────────────
   tag(1, string) "abc"  tag(2, string) "a@b"  tag(3, varint) 42
```

Three invariants worth tattooing:

1. **Field numbers are the contract.** Once `email = 2` is in production, that field number means email forever.
2. **Adding a field is a non-breaking change.** Old code ignores unknown fields; new code reads the new field if present, falls back to zero value otherwise.
3. **Removing a field is allowed; reusing its number is not.** Use `reserved 2;` to prevent accidental reuse.

## Setup

```bash
# Toolchain
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# Module
go get google.golang.org/protobuf
```

You need `protoc` (the protobuf compiler) installed system-wide, or use **Buf** which wraps it:

```bash
# Install buf
brew install bufbuild/buf/buf       # or via Go install

# Project layout
proto/
  user/v1/user.proto
  buf.yaml
  buf.gen.yaml
```

```yaml
# buf.yaml
version: v2
modules: [{path: proto}]
lint:
  use: [DEFAULT]
breaking:
  use: [FILE]
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
```

```bash
buf generate    # → ./gen/go/user/v1/user.pb.go
buf lint
buf breaking --against '.git#branch=main'
```

Buf is the modern toolchain. Skip raw `protoc` for new projects.

## Writing a `.proto` File

```proto
syntax = "proto3";
package user.v1;
option go_package = "example.com/api/user/v1;userv1";

import "google/protobuf/timestamp.proto";

message User {
  string id    = 1;
  string email = 2;
  string name  = 3;
  Role   role  = 4;
  google.protobuf.Timestamp created_at = 5;
  optional string display_name = 6;   // presence-aware (proto3 1.15+)
}

enum Role {
  ROLE_UNSPECIFIED = 0;        // proto3 requires a 0-value
  ROLE_ADMIN       = 1;
  ROLE_USER        = 2;
}

message ListUsersRequest {
  int32  limit  = 1;
  string cursor = 2;
}

message ListUsersResponse {
  repeated User users       = 1;
  string       next_cursor = 2;
}
```

Conventions:

- **One package, one major version** in the path: `user.v1`. When you make a breaking change, bump to `user.v2` (parallel package).
- **`go_package`** with the Go import path and short package alias.
- **`reserved`** to mark deprecated field numbers:
  ```proto
  message User {
    reserved 7, 8, 10 to 15;
    reserved "old_field_name";
  }
  ```
- **Enums** must include `_UNSPECIFIED = 0` as the default.
- **Field numbers 1-15** use 1 byte on the wire; 16-2047 use 2 bytes. Reserve low numbers for hot fields.

## Generated Go API — Legacy "Open" Struct API

```go
import userv1 "example.com/api/user/v1"

u := &userv1.User{
    Id:    "42",
    Email: "a@b.com",
    Name:  "Alice",
    Role:  userv1.Role_ROLE_ADMIN,
}

b, err := proto.Marshal(u)
// b is a binary protobuf blob

var u2 userv1.User
err = proto.Unmarshal(b, &u2)
```

Marshal/Unmarshal are the entire serialization API. The generated types have a `ProtoReflect()` method (for reflection-based tools) and a `String()` method (for printf-style debug; uses protobuf text format).

## The Opaque API (protobuf-go v1.36+, Go 1.26 era)

Long-standing gripe: in the open API, you can't tell whether a `string` is "" because the sender set it to "" or because they didn't send it. Field presence (`optional`) lets you, but only if used.

The **opaque API** makes all generated struct fields private and exposes getters/setters:

```proto
syntax = "proto3";
package user.v1;
option features.field_presence = EXPLICIT;
```

Generated:

```go
// Old (open):
//   u.Email = "a@b"
//   _ = u.Email

// New (opaque):
u := userv1.User_builder{Email: proto.String("a@b")}.Build()

// Getters
email := u.GetEmail()                // returns "" if unset
hasEmail := u.HasEmail()             // true presence check
u.SetEmail("new@b")
u.ClearEmail()
```

Benefits:

- Presence is unambiguous for every field, not just `optional`.
- Lazy decoding becomes possible (fields decoded on first access).
- Future memory-layout optimisations.

Trade-off: more verbose. Most existing codebases haven't migrated yet; new ones should consider it.

Choose per-file via `option features.api_level = API_OPAQUE;` (Edition 2024).

## Well-Known Types

```proto
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/any.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/field_mask.proto";

message Event {
  google.protobuf.Timestamp at = 1;
  google.protobuf.Duration  ttl = 2;
  google.protobuf.StringValue maybe_name = 3;   // wrapper for nullable string
  google.protobuf.Any         payload = 4;       // typed dynamic message
}
```

Go side:

```go
import "google.golang.org/protobuf/types/known/timestamppb"

e := &eventv1.Event{
    At:  timestamppb.New(time.Now()),
    Ttl: durationpb.New(time.Hour),
}

t := e.GetAt().AsTime()    // back to time.Time
d := e.GetTtl().AsDuration()
```

`timestamppb.Timestamp` is internally `seconds + nanos`. Always use the helpers to construct/extract.

## Wire Compatibility — The Inviolable Rules

You **CAN**:

- Add a new field (with a new field number).
- Remove a field (mark `reserved`).
- Rename a field (only the Go name changes; field number is the contract).
- Change a field's *name* — wire-compat preserved.
- Add a new enum value (older clients see it as the underlying int).

You **CANNOT** (without major-version bump):

- Reuse a field number for a different type or meaning.
- Change a field's type in a wire-incompatible way (e.g., `int32` → `string`).
- Reorder field numbers (wire format depends on them, but you can't change them without breaking).
- Remove the `reserved` mark and add a different field with the same number.

You **CAN with caveats**:

- Change between compatible numeric types (`int32 ↔ int64 ↔ uint32 ↔ bool` — same varint wire format). Mostly safe but think about value ranges.
- Convert `optional` ↔ `singular` (compatible but presence semantics change).
- Add a `oneof` to existing fields (subtle — read the spec carefully).

## `oneof`

```proto
message Payment {
  oneof method {
    string card_token = 1;
    string bank_ach   = 2;
    string crypto_addr = 3;
  }
}
```

Exactly one of the variants is set at any time. Go:

```go
p := &paymentv1.Payment{
    Method: &paymentv1.Payment_CardToken{CardToken: "tok_abc"},
}

switch m := p.GetMethod().(type) {
case *paymentv1.Payment_CardToken:
    use(m.CardToken)
case *paymentv1.Payment_BankAch:
    use(m.BankAch)
}
```

`oneof` is how you model sum types in protobuf. Note: cannot be `repeated`; cannot include `map`; cannot include another `oneof`.

## `map`

```proto
message Tags {
  map<string, string> labels = 1;
}
```

Generated as `map[string]string`. Wire format is repeated entries `{key, value}` — internally an unrolled list. Order is not preserved.

## Marshal Options

```go
b, err := proto.MarshalOptions{
    Deterministic: true,    // stable byte output for same input
    UseCachedSize: true,
}.Marshal(m)

err = proto.UnmarshalOptions{
    DiscardUnknown: false,  // default false — preserve unknown fields for round-trip
    AllowPartial:   false,  // reject if required fields (proto2) missing
}.Unmarshal(b, m)
```

`Deterministic: true` ensures the same message always serialises to the same bytes — important for content-addressed storage, signatures, hash-based dedup.

## JSON Interop

```go
import "google.golang.org/protobuf/encoding/protojson"

// To JSON
b, err := protojson.Marshal(u)

// From JSON
err = protojson.Unmarshal(b, u)
```

Protobuf JSON has its own conventions:

- Field names are camelCase by default (`createdAt`, not `created_at`).
- `Timestamp` serialises as RFC 3339 string.
- `Duration` as `"3.5s"`.
- Enums as their name string (`ROLE_ADMIN`).
- Missing fields ≠ zero values (if you use `optional`).

Useful for HTTP/JSON gateways (`grpc-gateway`) and Connect-Go.

## `protoreflect` — Reflection

```go
import "google.golang.org/protobuf/reflect/protoreflect"

msg := u.ProtoReflect()
desc := msg.Descriptor()
for i := 0; i < desc.Fields().Len(); i++ {
    field := desc.Fields().Get(i)
    fmt.Println(field.Name(), field.Kind(), msg.Get(field).String())
}
```

Used by generic tools: JSON converters, validators, transformers (e.g., redaction by tag), debug formatters.

## Buf Breaking-Change Detection

```bash
buf breaking --against 'https://github.com/example/api.git#branch=main'
```

Catches:

- Removed fields without `reserved`.
- Changed field types/numbers.
- Changed enum values.
- Removed services / methods.

Run in CI on every PR touching `.proto`. Block merges with breaking changes.

## Anti-Patterns & Gotchas

**Reusing a field number.** Catastrophic — old binary reads gibberish.

**Numbering everything from 1 sequentially.** Reserve "hot" fields for 1-15 (1-byte tag). Less-hot or future fields at 16+.

**Forgetting `_UNSPECIFIED = 0`.** Proto3 requires it. Without, the zero enum value will look identical to the lowest defined one.

**Using `required` in proto2.** Don't. Required is forever; you can never remove it without breaking. Proto3 dropped `required` for this reason.

**Treating `false` and "" as "not set."** They're zero values; without `optional`, you can't distinguish. Either use `optional` or a wrapper type.

**Sharing a `.proto` between v1 and v2 services.** v2 must be a different package (`user.v2`) in a different path.

**Hand-editing generated `.pb.go` files.** Lost on next generate. Customize via separate `.go` files in the same package.

**Storing protobuf in databases without versioning.** Schema changes mean old data may have unknown fields. Either always preserve unknown fields (`DiscardUnknown: false`) or version the serialisation explicitly.

**Using `Any` everywhere.** It works, but loses type safety and forces marshal-unmarshal at every layer. Prefer `oneof` for known sum types.

**`encoding/json` on a proto type instead of `protojson`.** The `json:` tags on generated structs don't match proto JSON conventions; the output is unparseable by other languages.

**Skipping `buf lint`.** Catches package layout issues, missing field numbers, naming inconsistencies.

**Mutating a proto message concurrently.** Not safe; race detector will flag.

**Stack-allocating big protos.** Generated structs can be large; pass `*User`, not `User`, everywhere.

**Forgetting that maps are unordered.** `proto.Marshal` may serialise in any order (unless `Deterministic: true`).

**Long-running services that don't handle unknown fields.** A new field added by a v+1 producer should pass through unchanged in the v service. Default behavior is correct; `DiscardUnknown: true` breaks it.

**Mixing v1 (`github.com/golang/protobuf`) and v2 (`google.golang.org/protobuf`) in one module.** Compatibility shims exist but cause subtle bugs.

## Performance Notes

(Approx. on modern x86.)

| Operation | Cost |
|-----------|------|
| `proto.Marshal` (small message) | 0.5-2 µs |
| `proto.Marshal` (1 KB message) | 5-20 µs |
| `proto.Unmarshal` (small) | 0.5-3 µs |
| `proto.Unmarshal` (1 KB) | 10-40 µs |
| `protojson.Marshal` (small) | 5-20 µs |
| `protojson.Unmarshal` (small) | 10-50 µs |
| `proto.Equal` | O(message size) |
| `proto.Clone` | O(message size) |

Protobuf is ~3-5x faster than `encoding/json` and ~2-3x smaller on the wire. The gap narrows for trivial messages; widens for nested.

`vtprotobuf` (https://github.com/planetscale/vtprotobuf) generates additional non-reflection-based serialization paths — 2-5x faster than vanilla protobuf-go for hot paths.

## Generated File Size & Build Time

```bash
# A typical .proto with 20 messages generates ~5k lines of Go.
# Build time impact: ~1-2s for protoc-gen-go on a medium proto.
```

Don't check `gen/` into the same module as your `.proto`s; check it in OR generate at build time, but pick one.

## How Big Companies Use It

- **Google** (originator) — protobuf is everywhere, internally and externally. Open-sourced 2008.
- **Cloudflare** uses protobuf for internal RPC + storage formats.
- **Discord** uses protobuf for their voice infrastructure (Go) and other backend.
- **Uber** uses protobuf with Thrift in some legacy stacks.
- **Square** uses protobuf + Buf for schema management.
- **Twitch** uses Twirp (protobuf + HTTP) extensively.
- **Stripe** uses protobuf internally for service-to-service.
- **PlanetScale** maintains `vtprotobuf` for high-performance generation.
- **CockroachDB** uses protobuf for inter-node RPC and KV layer.
- **etcd**, **Kubernetes** API objects, **Bazel** — all protobuf.

## Source Code References

- `google.golang.org/protobuf`: https://github.com/protocolbuffers/protobuf-go.
- Original protobuf: https://github.com/protocolbuffers/protobuf.
- Buf: https://github.com/bufbuild/buf.
- Buf docs: https://buf.build/docs/.
- `protoc-gen-go`: https://pkg.go.dev/google.golang.org/protobuf/cmd/protoc-gen-go.
- Well-known types: https://github.com/protocolbuffers/protobuf-go/tree/master/types/known.
- `vtprotobuf` (perf-focused codegen): https://github.com/planetscale/vtprotobuf.
- `twirp` (HTTP/RPC over protobuf): https://github.com/twitchtv/twirp.

## Further Reading

- Protobuf docs: https://protobuf.dev/.
- "Protobuf Best Practices" (Buf): https://buf.build/docs/lint/rules/.
- Edition 2024 spec: https://protobuf.dev/editions/.
- "Wire format" page: https://protobuf.dev/programming-guides/encoding/.
- "Versioning APIs at scale" (various Google AIP docs).
- "The Opaque API: why and how" (protobuf-go blog/changelog).
- "Avoid these protobuf footguns" — Cloudflare blog.

## Exercises / Self-Check

1. Define a `User` and `Order` message. Generate Go via Buf. Marshal, transmit (e.g., over TCP), unmarshal on the other side.
2. Add a new field. Verify old binaries (built before the change) can still unmarshal new messages (ignoring the new field) and old binaries' messages parse into the new struct (with zero value for the new field).
3. Try to repurpose a field number. Confirm via `buf breaking` that this is flagged.
4. Use `optional` to add presence semantics to a `bool` field. Verify `HasField()` works as expected.
5. Use a `oneof` to model "one of three payment methods." Serialize, deserialize, and use a switch to handle each variant.
6. Convert a proto message to JSON via `protojson`; compare output to `encoding/json` on the same struct. Observe the differences (field names, timestamp format).
7. Use `vtprotobuf` for a hot path; benchmark vs vanilla. Quantify the win.
8. Add `buf lint` + `buf breaking` to CI. Make a breaking change; confirm CI rejects.

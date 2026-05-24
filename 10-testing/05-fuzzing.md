# Fuzzing — `testing.F` (since Go 1.18)

## TL;DR

A Go fuzz target is `func FuzzXxx(f *testing.F)` in a `*_test.go` file. You seed it with example inputs via **`f.Add(...)`**, then call **`f.Fuzz(func(t *testing.T, args ...) { ... })`** with a closure whose parameters become the fuzz inputs. The framework mutates the seeds, runs the closure with mutated arguments, and reports any input that causes a `t.Error`, panic, or timeout. Supported parameter types: `[]byte`, `string`, `bool`, `int*`, `uint*`, `float*`, `rune`. The fuzz engine uses **coverage-guided mutation** (libFuzzer-style instrumentation) — it favors inputs that exercise new code paths. Crashes are persisted to `testdata/fuzz/FuzzXxx/<hash>` and become regression cases on subsequent `go test` runs. Invoke fuzzing with `go test -fuzz=FuzzXxx -fuzztime=30s`. Coverage-guided fuzzing was added in **Go 1.18**.

## Mental Model

```
   func FuzzParse(f *testing.F) {
       // 1. Seed corpus.
       f.Add("hello")
       f.Add("{}")
       f.Add(string(loadFile("testdata/fuzz/seed1")))

       // 2. Fuzz target (called many times with mutations).
       f.Fuzz(func(t *testing.T, in string) {
           v, err := Parse(in)
           if err == nil && v == nil {
               t.Errorf("Parse(%q) returned nil without error", in)
           }
       })
   }

   go test -fuzz=FuzzParse -fuzztime=60s ./pkg
        │
        ▼
   1. run all seeds as deterministic test cases
   2. for fuzztime duration:
        ├─ pick a corpus entry, mutate
        ├─ run closure with mutated input
        ├─ if new coverage observed, add to corpus
        ├─ if crash/error, persist to testdata/fuzz/FuzzParse/<hash>
   3. on exit, report new findings and corpus size
```

Seed inputs are *required*; the framework needs starting points to mutate. Crashes saved to `testdata/` make `go test ./pkg` (without `-fuzz`) replay them as regression checks.

## Syntax & Basic Usage

```go
func FuzzReverse(f *testing.F) {
    testcases := []string{"Hello, world", " ", "!12345"}
    for _, tc := range testcases {
        f.Add(tc)
    }
    f.Fuzz(func(t *testing.T, orig string) {
        rev := Reverse(orig)
        doubleRev := Reverse(rev)
        if orig != doubleRev {
            t.Errorf("Before: %q, after: %q", orig, doubleRev)
        }
        if utf8.ValidString(orig) && !utf8.ValidString(rev) {
            t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
        }
    })
}
```

Run:

```bash
$ go test -fuzz=FuzzReverse                         # fuzz indefinitely (Ctrl-C to stop)
$ go test -fuzz=FuzzReverse -fuzztime=30s ./pkg
$ go test -fuzz=FuzzReverse -fuzztime=1000x ./pkg   # fixed iterations
$ go test -fuzz=FuzzReverse -fuzzminimizetime=1m ./pkg
$ go test ./pkg                                      # run seeds and persisted failures as tests
```

## Deep Dive

### Seeding (`f.Add`)

```go
f.Add(string("hello"))                  // single-arg fuzz
f.Add([]byte("hi"), int(42))            // multi-arg fuzz
f.Add(int8(0), float64(0.1), "x")
```

`f.Add` records a seed for the fuzz engine to start from. Seed inputs are also run as regular tests when `go test` runs without `-fuzz` — so they double as table-driven test cases.

**Argument types** allowed: `[]byte`, `string`, `bool`, `byte`, `rune`, `int`, `int8/16/32/64`, `uint`, `uint8/16/32/64`, `float32/64`. Other types (structs, slices of structs) aren't directly fuzzable; serialize to `[]byte` and parse inside the fuzz target.

The arguments to `f.Add` must match (in number and types) the parameters of the `f.Fuzz` closure (after the leading `*testing.T`).

### `f.Fuzz` target

```go
f.Fuzz(func(t *testing.T, data []byte, n int) {
    out, err := Process(data, n)
    if err == nil && out == nil {
        t.Errorf("Process returned nil without error")
    }
})
```

The closure runs once per input. Inside:

- Use `t.Errorf`/`t.Fatalf` to flag failures.
- Use `t.Skip` to discard inputs that don't satisfy preconditions (e.g., "skip empty inputs"; the engine will move on).
- Never call `t.Parallel` — fuzz iterations run concurrently across worker processes anyway.

### Properties to assert (the heart of fuzzing)

Fuzz targets are **property checkers**, not example tests. Common properties:

| Property                    | Example                                                        |
|-----------------------------|----------------------------------------------------------------|
| **Round-trip**              | `Marshal` then `Unmarshal` returns original.                   |
| **Idempotence**             | `f(f(x)) == f(x)`.                                              |
| **Invariant preservation**  | After op, some invariant still holds.                          |
| **Equivalence**              | New implementation matches old (oracle).                       |
| **Never crash**             | `Parse(x)` returns error, never panics.                        |
| **Bounded output**          | `len(Compress(x)) <= len(x) * 2` (sanity).                     |

```go
// Round-trip
f.Fuzz(func(t *testing.T, in []byte) {
    encoded, err := Encode(in)
    if err != nil { return }   // not a crash; expected error
    decoded, err := Decode(encoded)
    if err != nil { t.Fatalf("decode failed: %v", err) }
    if !bytes.Equal(in, decoded) {
        t.Errorf("round-trip mismatch")
    }
})
```

### Persisted failures

When the engine finds a crashing input, it writes:

```
testdata/fuzz/FuzzReverse/771e938e4458e983
```

The file contains the input (serialized for re-creation). Subsequent `go test ./pkg` (no `-fuzz`) replays this file as a regular test case — once you fix the bug, the test must still pass on this exact input. **Commit the file to source control**.

### Minimization

When a failure occurs, the engine tries to minimize the input — finding the smallest sub-input that still triggers the failure. Controlled by `-fuzzminimizetime` (default 1 min). Minimization makes debugging easier; the saved file holds the minimized version.

### `-fuzz` and `-fuzztime`

```bash
$ go test -fuzz=Pattern             # fuzz indefinitely
$ go test -fuzz=Pattern -fuzztime=60s
$ go test -fuzz=Pattern -fuzztime=1000x
```

`-fuzz` filters by regex like `-run`. Only one fuzz target per invocation actively fuzzes; others run only their seeds.

### Worker processes

The engine spawns N worker processes (default `GOMAXPROCS`) that fuzz independently and share corpus. Override:

```bash
$ go test -fuzz=. -parallel=4 ./pkg     # cap to 4 workers
```

Each worker re-runs `TestMain` (if defined); be careful with setup that conflicts (e.g., binding the same port).

### Corpus location

```
testdata/fuzz/FuzzXxx/        ← persisted crashes (commit)
$GOCACHE/fuzz/FuzzXxx/        ← discovered corpus (local; not committed)
```

The discovered corpus is per-machine; the persisted crashes are tracked in source.

### `go test -run TestX` and seeds

```bash
$ go test ./pkg                # runs all tests; FuzzX seeds + persisted crashes
$ go test -run FuzzReverse ./pkg   # runs only FuzzReverse's seeds + persisted
$ go test -fuzz FuzzReverse ./pkg  # actively fuzzes
```

The distinction: `-run` matches all test types (including fuzz seeds as deterministic tests); `-fuzz` activates the mutation engine.

### Fuzz with multiple args

```go
func FuzzPair(f *testing.F) {
    f.Add(int(0), "")
    f.Add(int(10), "abc")
    f.Fuzz(func(t *testing.T, n int, s string) {
        if n < 0 { return }
        result := Repeat(s, n)
        if utf8.RuneCountInString(result) != utf8.RuneCountInString(s)*n {
            t.Errorf("Repeat(%q, %d) wrong length", s, n)
        }
    })
}
```

The engine mutates each argument independently.

### Building a corpus

For complex inputs (real-world files, protocol messages), seed from a directory:

```go
func FuzzParse(f *testing.F) {
    matches, _ := filepath.Glob("testdata/seeds/*.json")
    for _, m := range matches {
        data, _ := os.ReadFile(m)
        f.Add(data)
    }
    f.Fuzz(func(t *testing.T, data []byte) { /* ... */ })
}
```

Or put one file per case under `testdata/fuzz/FuzzParse/`:

```
testdata/fuzz/FuzzParse/
    seed1
    seed2
    seed3
```

The engine reads them as seeds automatically. File format:

```
go test fuzz v1
[]byte("...")
```

Files are auto-generated by failures; manual ones must follow the format.

### Code coverage and corpus growth

The engine uses libFuzzer-style coverage instrumentation: each branch the program takes contributes to a coverage bitmap. Inputs that exercise *new* branches join the corpus and seed future mutations. Without instrumentation, fuzzing is dumb random search.

Go's coverage isn't as fine-grained as C++ libFuzzer (no per-comparison guided), but covers basic-block reachability well.

### Skip vs. return

```go
f.Fuzz(func(t *testing.T, in []byte) {
    if len(in) == 0 {
        return                  // skip silently
        // OR
        t.Skip()                // skip with reason
    }
    // ...
})
```

Functionally similar. `return` is simpler; use `t.Skip` only when you want the skipped count tracked.

### CI integration

```yaml
- run: go test -fuzz=FuzzCritical -fuzztime=2m ./pkg
```

Fuzz for a fixed duration per CI run. Persist any new failures (commit them; they become regression tests for next time).

For continuous fuzzing across days/weeks, use [OSS-Fuzz](https://google.github.io/oss-fuzz/) or self-host (e.g., ClusterFuzzLite). The Go vuln team uses OSS-Fuzz extensively for stdlib.

### Interplay with `-race`

```bash
$ go test -fuzz=FuzzX -race ./pkg
```

Race detector + fuzzing finds concurrency bugs reachable only via fuzz-generated input. Slower but valuable for concurrent code.

### Limitations

- No struct fuzzing (use `[]byte` and parse inside).
- No `time.Time` fuzzing.
- No reproducibility across Go versions (mutation algorithm may change).
- Inputs are not symbolic — fuzzing won't find paths gated by complex constraints (e.g., "input starts with magic bytes XYZ"); seed with such inputs.
- No support for stateful protocol fuzzing out of the box; you'd hand-roll a state machine inside the target.

## Standard Library Hooks

- `testing.F` — fuzz target type.
- `testing.F.Add` — seed corpus.
- `testing.F.Fuzz` — register target.
- `testing.F.Skip` — skip during seed phase.
- `testing.F.Fail`, `Errorf`, `Fatalf` — error reporting (used at seed time and during fuzz).
- `testing/quick` — basic property-based testing (predates fuzzing; less powerful but stdlib).

## Real-World Patterns

### 1. Round-trip serializer

```go
func FuzzJSONRoundTrip(f *testing.F) {
    f.Add(`{"a":1}`)
    f.Add(`null`)
    f.Add(`[1, "two", true]`)
    f.Fuzz(func(t *testing.T, data string) {
        var v any
        if err := json.Unmarshal([]byte(data), &v); err != nil {
            return    // not a valid JSON input
        }
        out, err := json.Marshal(v)
        if err != nil {
            t.Fatalf("marshal: %v", err)
        }
        var v2 any
        if err := json.Unmarshal(out, &v2); err != nil {
            t.Fatalf("re-unmarshal: %v", err)
        }
        if !reflect.DeepEqual(v, v2) {
            t.Errorf("round-trip mismatch")
        }
    })
}
```

### 2. Never-panic parser

```go
func FuzzParseURL(f *testing.F) {
    f.Add("https://example.com")
    f.Add("foo://bar")
    f.Add("")
    f.Fuzz(func(t *testing.T, raw string) {
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("Parse panicked on %q: %v", raw, r)
            }
        }()
        _, _ = url.Parse(raw)
    })
}
```

### 3. Equivalence to oracle

```go
func FuzzFastVsSlow(f *testing.F) {
    f.Add(uint32(42))
    f.Fuzz(func(t *testing.T, n uint32) {
        if FastSqrt(n) != SlowSqrt(n) {
            t.Errorf("FastSqrt(%d) != SlowSqrt(%d)", n, n)
        }
    })
}
```

### 4. Bounded output

```go
func FuzzCompress(f *testing.F) {
    f.Add([]byte("hello"))
    f.Fuzz(func(t *testing.T, data []byte) {
        compressed := Compress(data)
        if len(compressed) > len(data)*2 + 100 {
            t.Errorf("Compress bloated: %d → %d", len(data), len(compressed))
        }
    })
}
```

### 5. Stateful protocol (hand-rolled)

```go
func FuzzProtocol(f *testing.F) {
    f.Add([]byte{0x01, 0x02, 0x03})
    f.Fuzz(func(t *testing.T, script []byte) {
        s := NewState()
        for _, b := range script {
            switch b & 0x3 {
            case 0: s.Push(int(b >> 2))
            case 1: s.Pop()
            case 2: s.Top()
            case 3: s.Reset()
            }
        }
    })
}
```

Each fuzz byte triggers a state operation. Useful for finding bad state sequences.

### 6. Crash regression replay

After `testdata/fuzz/FuzzX/abc123` is written, a plain `go test ./pkg` runs it. Once the bug is fixed, the file becomes a permanent regression test.

### 7. CI fuzz budget

```yaml
jobs:
  fuzz-critical:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: go test -fuzz=FuzzCriticalParser -fuzztime=5m ./pkg
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: fuzz-corpus
          path: testdata/fuzz/
```

Upload corpus on failure for inspection.

## Anti-Patterns & Gotchas

**Fuzzing without seeds.** `f.Fuzz` without prior `f.Add` calls is an error. Always seed.

**Catching errors and silencing them in the target.** If `Process(x)` returns an error and you `return`, the fuzz target ignores that input. Make sure errors are *expected* errors, not panics or wrong results.

**Asserting equality with a brittle reference (timestamp, random).** Fuzz inputs that include time may differ across runs. Hash or normalize before comparing.

**Mutating shared state from the fuzz target.** The engine runs the target many times concurrently. Don't mutate globals.

**Forgetting to commit `testdata/fuzz/*`.** Persisted crashes are committed; without them, the regression case is lost.

**Fuzz target with `t.Parallel`.** Not supported; the engine parallelizes already.

**Long-running fuzz target per iteration.** If your target takes 10 ms, throughput is 100 iter/sec — slow learning. Aim for sub-ms targets if possible.

**Mixing fuzz target args with package-level state.** Reset state per call inside the target.

**Using `testing.Short()` inside a fuzz target.** Fuzzing doesn't honor `-short`; the engine drives iteration count.

**No timeout for the fuzzed function.** A pathological input may hang. Wrap in `context.WithTimeout` if the target can loop:

```go
f.Fuzz(func(t *testing.T, data []byte) {
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    Process(ctx, data)
})
```

**Comparing `[]byte` with `==`.** Compile error; use `bytes.Equal`.

**Treating fuzzing as a replacement for unit tests.** Fuzz finds bugs you didn't think of; unit tests pin behavior you do think about. Need both.

## Performance Notes

- Fuzz iteration cost: dominated by the target function; overhead per iter ~µs.
- Coverage instrumentation: ~10–30% slowdown on the target's runtime.
- Corpus size: grows with coverage discovery; 100s – 10k inputs typical for medium projects.
- Worker process count: default `GOMAXPROCS`; lower with `-parallel`.
- Time to first crash: minutes for shallow bugs; hours/days for deep ones.

For high-throughput fuzzing, keep the target small (extract just the function under test, avoid I/O, avoid setup).

## How Big Companies Use It

- **Google** runs Go stdlib fuzzers on OSS-Fuzz continuously: https://github.com/google/oss-fuzz/tree/master/projects/golang.
- **The Go team** uses native fuzzing extensively for `encoding/*`, `crypto/*`, `net/http`: https://github.com/golang/go.
- **Cloudflare** fuzzes their TLS and QUIC implementations: https://blog.cloudflare.com/tag/fuzzing.
- **Tailscale** fuzzes `wireguard-go` and `derp` protocol parsing: https://tailscale.com/blog.
- **CockroachDB** fuzzes their SQL parser via go-fuzz (pre-1.18) and native fuzzing (post): https://github.com/cockroachdb/cockroach.
- **HashiCorp** fuzzes Vault crypto paths via OSS-Fuzz: https://github.com/hashicorp/vault.
- **Discord** fuzzes their custom protocol parsers in CI.

## Source Code References

Pinned to `go1.26`.

- `testing.F`: [`src/testing/fuzz.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/fuzz.go).
- Fuzzing engine: [`src/internal/fuzz`](https://github.com/golang/go/tree/release-branch.go1.26/src/internal/fuzz).
- Mutator: [`src/internal/fuzz/mutator.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/internal/fuzz/mutator.go).
- Coverage instrumentation: [`src/cmd/compile/internal/coverage`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/coverage).
- `go test -fuzz` driver: [`src/cmd/go/internal/test/test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/test/test.go).
- Worker process model: [`src/internal/fuzz/worker.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/internal/fuzz/worker.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go fuzzing tutorial" (Go docs): https://go.dev/doc/tutorial/fuzz.
- "Fuzzing is beta-ready" (Katie Hockman, Jay Conrod): https://go.dev/blog/fuzz-beta.
- "Native fuzzing in Go 1.18" (Go blog): https://go.dev/blog/fuzz-go-118.
- "Go fuzzing reference": https://pkg.go.dev/testing#hdr-Fuzzing.
- "Internals of Go fuzzing" (Katie Hockman talk): https://www.youtube.com/watch?v=t-vV-DiQjFw.
- OSS-Fuzz Go integration: https://google.github.io/oss-fuzz/getting-started/new-project-guide/go-lang.
- `go-fuzz` (legacy pre-1.18 fuzzer by Dmitry Vyukov): https://github.com/dvyukov/go-fuzz.

## Exercises / Self-Check

1. Write a fuzz target for `json.Unmarshal`. Add three seeds. Run for 30 s. Did any crashes surface?
2. Find an existing unit test in your codebase that you could re-express as a fuzz target (e.g., a parser or serializer).
3. Trigger a panic in a fuzz target intentionally. Confirm a file appears under `testdata/fuzz/FuzzX/`.
4. Run `-fuzz=FuzzX -race -fuzztime=2m`. Are there race-detected failures that weren't visible serially?
5. Modify a fuzz target to use multiple arguments (e.g., `[]byte, int`). Add corresponding `f.Add` seeds.

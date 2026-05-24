# Everything Golang

A comprehensive, opinionated reference covering Go (Golang) from first principles to the deepest, weirdest corners of the runtime. Targets **Go 1.26+** but flags every modern feature back to 1.18 (generics) where relevant.

Each entry below is a placeholder for a dedicated page. When asked to "write the page for [topic]", follow the **Author Instructions** section verbatim and produce the file at the listed path.

---

## Author Instructions (Read This Before Writing Any Page)

These rules apply to **every** page in this repository. Do not deviate without explicit user approval.

1. **File location & naming**
   - Each topic lives at the path listed in the index below (e.g. `01-getting-started/03-go-toolchain.md`).
   - Use kebab-case filenames, two-digit numeric prefixes for ordering inside each section.
   - Create parent folders as needed.

2. **Target version**
   - Default to **Go 1.26+** syntax, behavior, and standard library.
   - When a feature was introduced in an earlier version (1.18 generics, 1.21 `slices`/`maps`/`cmp`, 1.22 loop var scoping, 1.23 range-over-func, 1.24 `weak`/`testing/synctest`, 1.25 `structs`, etc.), mark it with a `(since 1.X)` tag.
   - Flag deprecated/removed APIs explicitly with `(deprecated in 1.X)` or `(removed in 1.X)`.

3. **Page structure (mandatory)**
   Every page must contain the following sections, in this order:
   1. **TL;DR** — 2–4 sentences. What this is, when to use it, the single biggest gotcha.
   2. **Mental Model** — the underlying concept. Diagram in ASCII / mermaid if helpful.
   3. **Syntax & Basic Usage** — minimum viable example, runnable.
   4. **Deep Dive** — semantics, edge cases, internals. Cite the spec section where applicable.
   5. **Standard Library Hooks** — relevant packages and functions in `std`.
   6. **Real-World Patterns** — at least 2 idiomatic patterns with full code.
   7. **Anti-Patterns & Gotchas** — what trips people up. Include the bug, why it bites, the fix.
   8. **Performance Notes** — allocation behavior, escape, GC implications, benchmark numbers where possible.
   9. **How Big Companies Use It** — link to public engineering posts / open-source repos. Examples: Kubernetes, Docker, Uber, Cloudflare, Dropbox, Discord, Twitch, Tailscale, HashiCorp.
   10. **Source Code References** — direct links into [github.com/golang/go](https://github.com/golang/go) (prefer `src/runtime/`, `src/cmd/compile/`, `src/sync/`, etc.) with file path + line range. Pin to a tagged release tag (e.g. `go1.26`).
   11. **Further Reading** — primary sources first (spec, design docs, proposals on `golang/go/issues`), then high-signal third-party material (Russ Cox blog, Dave Cheney, go.dev/blog, Bryan C. Mills, Damian Gryski, Anton Sergeev, `research.google` posts).
   12. **Exercises / Self-Check** — 3–5 short prompts the reader can attempt.

4. **Code samples**
   - All examples must be **compilable as written** against Go 1.26+.
   - Include the `package` declaration. Use `package main` for runnable snippets and a realistic package name otherwise.
   - Prefer the standard library. If a third-party module is necessary, list its `go get` line.
   - Show output as a `// Output:` comment so the snippet doubles as an `Example_` test.
   - No `panic("TODO")`. Either show the real thing or omit the section.

5. **Sources & citations**
   - Cite **primary sources** (Go spec, proposal docs, runtime source) before secondary.
   - Every external link must be a stable URL — prefer permalinks (commit-pinned GitHub URLs, archive.org for ephemeral blog posts).
   - When pulling from the runtime, copy a small snippet (≤15 lines) and link to the full file. Respect Go's BSD-3 license — include a one-line attribution.

6. **Tone & format**
   - Direct, technical, no fluff. Assume the reader can program — explain *Go-specific* nuance, not what a variable is.
   - Prose over bullet lists where possible; reserve lists for genuinely enumerable things.
   - Code over prose where possible.
   - No marketing language. No "powerful", "robust", "elegant" filler.

7. **When done writing a page**
   - Reply with exactly: `done`
   - No summary, no recap of what you wrote, no preview of what's next.

8. **When the user says "write pages for [topic]"**
   - If `[topic]` matches one index entry → write that file only.
   - If `[topic]` matches a whole section → write every file in the section, one after another, in order. Reply `done` only after the last file is saved.
   - If ambiguous, list the candidate paths and ask which.

---

## Index

### Part 0 — Orientation
- [00-overview/01-why-go.md](00-overview/01-why-go.md) — Design goals, history, what Go optimizes for (and against)
- [00-overview/02-go-versions-timeline.md](00-overview/02-go-versions-timeline.md) — Every release from 1.0 → 1.26, headline features per version
- [00-overview/03-the-go-team-and-proposal-process.md](00-overview/03-the-go-team-and-proposal-process.md) — How language changes happen, `golang/proposal`, the freeze cycle
- [00-overview/04-go-1-26-whats-new.md](00-overview/04-go-1-26-whats-new.md) — Headline changes in 1.26+, compatibility promise

### Part 1 — Getting Started
- [01-getting-started/01-installation.md](01-getting-started/01-installation.md) — Official installer, gvm, asdf, homebrew, building from source
- [01-getting-started/02-hello-world.md](01-getting-started/02-hello-world.md) — Anatomy of the smallest Go program
- [01-getting-started/03-go-toolchain.md](01-getting-started/03-go-toolchain.md) — `go build`, `go run`, `go install`, `go env`, `go version`
- [01-getting-started/04-project-layout.md](01-getting-started/04-project-layout.md) — `cmd/`, `internal/`, `pkg/`, the `golang-standards/project-layout` debate
- [01-getting-started/05-editor-setup.md](01-getting-started/05-editor-setup.md) — `gopls`, VS Code, Neovim, GoLand, Zed

### Part 2 — Language Basics
- [02-language-basics/01-variables-and-constants.md](02-language-basics/01-variables-and-constants.md) — `var`, `:=`, `const`, untyped constants, iota
- [02-language-basics/02-primitive-types.md](02-language-basics/02-primitive-types.md) — int sizes, float, complex, bool, byte, rune
- [02-language-basics/03-zero-values.md](02-language-basics/03-zero-values.md) — Why Go has no `null`-style undefined state
- [02-language-basics/04-type-conversion-and-assertions.md](02-language-basics/04-type-conversion-and-assertions.md) — Conversion vs assertion, comma-ok idiom
- [02-language-basics/05-operators-and-precedence.md](02-language-basics/05-operators-and-precedence.md) — Including `&^` (AND NOT)
- [02-language-basics/06-control-flow.md](02-language-basics/06-control-flow.md) — `if`, `for` (all forms), `switch`, `goto`, labels
- [02-language-basics/07-defer.md](02-language-basics/07-defer.md) — Stack semantics, open-coded defers, arguments captured at defer-time
- [02-language-basics/08-functions.md](02-language-basics/08-functions.md) — Multiple returns, named returns, variadic, function values
- [02-language-basics/09-closures.md](02-language-basics/09-closures.md) — Capture semantics, the loop variable trap (and 1.22 fix)
- [02-language-basics/10-iota-deep-dive.md](02-language-basics/10-iota-deep-dive.md) — Bit flags, skip values, typed enums via iota

### Part 3 — Composite Types
- [03-composite-types/01-arrays.md](03-composite-types/01-arrays.md) — Fixed-size, value semantics, when you actually want one
- [03-composite-types/02-slices.md](03-composite-types/02-slices.md) — Header layout, growth strategy, `append` aliasing, `slices.Clip`
- [03-composite-types/03-maps.md](03-composite-types/03-maps.md) — Swiss-table implementation (1.24+), iteration randomization, no addressable values
- [03-composite-types/04-strings-bytes-runes.md](03-composite-types/04-strings-bytes-runes.md) — Immutability, UTF-8, `strings.Builder`, `[]byte` conversion costs
- [03-composite-types/05-structs.md](03-composite-types/05-structs.md) — Field alignment, padding, tags, anonymous fields, struct comparison
- [03-composite-types/06-pointers.md](03-composite-types/06-pointers.md) — No pointer arithmetic, `new` vs `&T{}`, nil pointer methods

### Part 4 — Methods, Interfaces, Generics
- [04-methods-interfaces-generics/01-methods.md](04-methods-interfaces-generics/01-methods.md) — Value vs pointer receivers, method sets
- [04-methods-interfaces-generics/02-interfaces.md](04-methods-interfaces-generics/02-interfaces.md) — Structural typing, `iface` and `eface` runtime layout
- [04-methods-interfaces-generics/03-empty-interface-and-any.md](04-methods-interfaces-generics/03-empty-interface-and-any.md) — `any` alias, boxing costs
- [04-methods-interfaces-generics/04-type-assertions-and-switches.md](04-methods-interfaces-generics/04-type-assertions-and-switches.md)
- [04-methods-interfaces-generics/05-embedding.md](04-methods-interfaces-generics/05-embedding.md) — Composition over inheritance, method promotion, ambiguity rules
- [04-methods-interfaces-generics/06-generics-fundamentals.md](04-methods-interfaces-generics/06-generics-fundamentals.md) — Type parameters, since 1.18
- [04-methods-interfaces-generics/07-constraints-and-type-sets.md](04-methods-interfaces-generics/07-constraints-and-type-sets.md) — `comparable`, `~`, the `constraints` proposal
- [04-methods-interfaces-generics/08-generic-data-structures.md](04-methods-interfaces-generics/08-generic-data-structures.md) — Stack, ring buffer, LRU cache
- [04-methods-interfaces-generics/09-when-not-to-use-generics.md](04-methods-interfaces-generics/09-when-not-to-use-generics.md) — Russ Cox's "when in doubt, don't"

### Part 5 — Error Handling
- [05-errors/01-error-interface.md](05-errors/01-error-interface.md) — Why errors are values
- [05-errors/02-errors-package.md](05-errors/02-errors-package.md) — `errors.Is`, `errors.As`, `errors.Join` (1.20+)
- [05-errors/03-error-wrapping.md](05-errors/03-error-wrapping.md) — `%w` verb, custom `Unwrap`
- [05-errors/04-sentinel-vs-typed-vs-opaque.md](05-errors/04-sentinel-vs-typed-vs-opaque.md) — Dave Cheney's taxonomy
- [05-errors/05-panic-and-recover.md](05-errors/05-panic-and-recover.md) — When (rarely) to use them
- [05-errors/06-error-handling-at-scale.md](05-errors/06-error-handling-at-scale.md) — How Kubernetes, Docker, and CockroachDB structure errors

### Part 6 — Packages & Modules
- [06-packages-modules/01-package-fundamentals.md](06-packages-modules/01-package-fundamentals.md) — Naming, exported identifiers, `init()`
- [06-packages-modules/02-imports.md](06-packages-modules/02-imports.md) — Import paths, aliases, blank/dot imports
- [06-packages-modules/03-go-modules.md](06-packages-modules/03-go-modules.md) — `go.mod`, `go.sum`, MVS algorithm
- [06-packages-modules/04-mod-directives.md](06-packages-modules/04-mod-directives.md) — `replace`, `exclude`, `retract`, `toolchain`
- [06-packages-modules/05-workspaces.md](06-packages-modules/05-workspaces.md) — `go.work` (since 1.18)
- [06-packages-modules/06-versioning-and-semver.md](06-packages-modules/06-versioning-and-semver.md) — `/v2+` import paths, pre-release tags
- [06-packages-modules/07-vendoring.md](06-packages-modules/07-vendoring.md) — When it still matters
- [06-packages-modules/08-private-modules-and-goproxy.md](06-packages-modules/08-private-modules-and-goproxy.md) — `GOPRIVATE`, `GONOSUMCHECK`, Athens, JFrog

### Part 7 — Concurrency
- [07-concurrency/01-goroutines.md](07-concurrency/01-goroutines.md) — `go` keyword, scheduler basics, stack growth
- [07-concurrency/02-channels.md](07-concurrency/02-channels.md) — Buffered vs unbuffered, send/receive semantics, `hchan` runtime struct
- [07-concurrency/03-select.md](07-concurrency/03-select.md) — Pseudo-random choice, default branch, nil-channel trick
- [07-concurrency/04-sync-mutex-rwmutex.md](07-concurrency/04-sync-mutex-rwmutex.md) — Starvation mode, when RWMutex is slower than Mutex
- [07-concurrency/05-sync-waitgroup-once-cond.md](07-concurrency/05-sync-waitgroup-once-cond.md) — `sync.WaitGroup.Go` (1.25+)
- [07-concurrency/06-sync-map-pool.md](07-concurrency/06-sync-map-pool.md) — When sync.Map beats `map+Mutex`, sync.Pool's GC interaction
- [07-concurrency/07-sync-atomic.md](07-concurrency/07-sync-atomic.md) — Typed atomics (1.19+), memory ordering
- [07-concurrency/08-context.md](07-concurrency/08-context.md) — Cancellation, deadlines, values, `context.AfterFunc` (1.21+)
- [07-concurrency/09-go-memory-model.md](07-concurrency/09-go-memory-model.md) — Happens-before, the official memory model doc, sequential consistency for atomics (1.19+)
- [07-concurrency/10-race-detector.md](07-concurrency/10-race-detector.md) — `-race`, false negatives, ThreadSanitizer underneath
- [07-concurrency/11-scheduler-internals.md](07-concurrency/11-scheduler-internals.md) — G-M-P model, work stealing, preemption (since 1.14)
- [07-concurrency/12-concurrency-patterns.md](07-concurrency/12-concurrency-patterns.md) — Fan-out/in, pipelines, worker pools, semaphore, bounded parallelism
- [07-concurrency/13-errgroup-and-singleflight.md](07-concurrency/13-errgroup-and-singleflight.md) — `golang.org/x/sync`
- [07-concurrency/14-testing-synctest.md](07-concurrency/14-testing-synctest.md) — Deterministic concurrency tests (stable 1.25)
- [07-concurrency/15-channels-vs-mutexes.md](07-concurrency/15-channels-vs-mutexes.md) — The classic debate, with benchmarks
- [07-concurrency/16-goroutine-leaks.md](07-concurrency/16-goroutine-leaks.md) — Detection, `goleak`, the dangling-receiver pattern

### Part 8 — Standard Library Deep Dive
- [08-stdlib/01-fmt.md](08-stdlib/01-fmt.md) — Verbs, `Stringer`, `Formatter`, custom verbs
- [08-stdlib/02-io-and-io-fs.md](08-stdlib/02-io-and-io-fs.md) — `io.Reader`/`Writer` design, `io/fs` abstraction
- [08-stdlib/03-bufio.md](08-stdlib/03-bufio.md) — Scanner pitfalls (`bufio.MaxScanTokenSize`)
- [08-stdlib/04-os-and-exec.md](08-stdlib/04-os-and-exec.md) — `os.Root` (1.24+), `exec.Cmd` lifecycle
- [08-stdlib/05-os-signal.md](08-stdlib/05-os-signal.md) — `signal.NotifyContext`
- [08-stdlib/06-path-and-filepath.md](08-stdlib/06-path-and-filepath.md) — Slash vs OS separators
- [08-stdlib/07-strings-and-bytes.md](08-stdlib/07-strings-and-bytes.md) — Builder, Reader, common allocation traps
- [08-stdlib/08-strconv.md](08-stdlib/08-strconv.md)
- [08-stdlib/09-unicode-and-utf8.md](08-stdlib/09-unicode-and-utf8.md)
- [08-stdlib/10-time.md](08-stdlib/10-time.md) — Monotonic clock, locations, layout reference string mystery
- [08-stdlib/11-math-rand-bigint.md](08-stdlib/11-math-rand-bigint.md) — `math/rand/v2` (1.22+)
- [08-stdlib/12-encoding-json.md](08-stdlib/12-encoding-json.md) — `encoding/json/v2` (experimental → 1.26), streaming, ordering
- [08-stdlib/13-encoding-xml-gob-csv.md](08-stdlib/13-encoding-xml-gob-csv.md)
- [08-stdlib/14-encoding-base64-hex.md](08-stdlib/14-encoding-base64-hex.md)
- [08-stdlib/15-net.md](08-stdlib/15-net.md) — `netip.Addr`, dialers, resolvers
- [08-stdlib/16-net-http-server.md](08-stdlib/16-net-http-server.md) — `ServeMux` 1.22 routing, `http.Server` graceful shutdown
- [08-stdlib/17-net-http-client.md](08-stdlib/17-net-http-client.md) — Transport reuse, connection pooling, the `defer Body.Close` mistake
- [08-stdlib/18-html-text-template.md](08-stdlib/18-html-text-template.md) — Auto-escaping, custom funcs
- [08-stdlib/19-log-slog.md](08-stdlib/19-log-slog.md) — Structured logging (since 1.21)
- [08-stdlib/20-context-revisited.md](08-stdlib/20-context-revisited.md)
- [08-stdlib/21-sort.md](08-stdlib/21-sort.md) — Now mostly superseded by `slices.Sort`
- [08-stdlib/22-container-heap-list-ring.md](08-stdlib/22-container-heap-list-ring.md)
- [08-stdlib/23-crypto.md](08-stdlib/23-crypto.md) — `crypto/rand`, `crypto/tls`, FIPS-140 in 1.24+
- [08-stdlib/24-database-sql.md](08-stdlib/24-database-sql.md) — Connection pool internals, `context.Context` integration
- [08-stdlib/25-compress-archive.md](08-stdlib/25-compress-archive.md)
- [08-stdlib/26-regexp.md](08-stdlib/26-regexp.md) — RE2 guarantees, no backreferences, why
- [08-stdlib/27-runtime.md](08-stdlib/27-runtime.md) — `GOMAXPROCS`, `Gosched`, `KeepAlive`, `SetFinalizer`
- [08-stdlib/28-runtime-debug.md](08-stdlib/28-runtime-debug.md) — `BuildInfo`, `GOMEMLIMIT`, `SetGCPercent`
- [08-stdlib/29-flag.md](08-stdlib/29-flag.md) — The minimalist case for `flag` over Cobra
- [08-stdlib/30-embed.md](08-stdlib/30-embed.md) — `//go:embed`, FS variant
- [08-stdlib/31-iter.md](08-stdlib/31-iter.md) — Range-over-func (since 1.23), `iter.Seq` / `iter.Seq2`
- [08-stdlib/32-slices-maps-cmp.md](08-stdlib/32-slices-maps-cmp.md) — Generic helpers (since 1.21)
- [08-stdlib/33-unique.md](08-stdlib/33-unique.md) — Value interning (since 1.23)
- [08-stdlib/34-weak.md](08-stdlib/34-weak.md) — Weak pointers (since 1.24)
- [08-stdlib/35-structs.md](08-stdlib/35-structs.md) — `structs.HostLayout` (since 1.25)
- [08-stdlib/36-maphash-netip-expvar.md](08-stdlib/36-maphash-netip-expvar.md)

### Part 9 — Tooling (every binary the toolchain ships)
- [09-tooling/01-go-build.md](09-tooling/01-go-build.md) — Build cache, `-trimpath`, `-buildvcs`
- [09-tooling/02-go-run-install.md](09-tooling/02-go-run-install.md)
- [09-tooling/03-go-mod.md](09-tooling/03-go-mod.md) — Every subcommand
- [09-tooling/04-go-work.md](09-tooling/04-go-work.md)
- [09-tooling/05-go-test.md](09-tooling/05-go-test.md) — `-run`, `-count`, `-race`, `-cover`, `-coverprofile`, `-bench`, `-cpu`
- [09-tooling/06-go-fmt-and-gofmt.md](09-tooling/06-go-fmt-and-gofmt.md)
- [09-tooling/07-go-vet.md](09-tooling/07-go-vet.md) — Built-in analyzers
- [09-tooling/08-go-doc-and-godoc.md](09-tooling/08-go-doc-and-godoc.md) — `pkg.go.dev`
- [09-tooling/09-go-generate.md](09-tooling/09-go-generate.md) — Directive syntax, `stringer`, `mockgen`
- [09-tooling/10-go-fix-and-go-tool.md](09-tooling/10-go-fix-and-go-tool.md)
- [09-tooling/11-go-tool-compile-link.md](09-tooling/11-go-tool-compile-link.md) — Inspecting SSA, `-gcflags`, `-ldflags`
- [09-tooling/12-go-tool-pprof.md](09-tooling/12-go-tool-pprof.md)
- [09-tooling/13-go-tool-trace.md](09-tooling/13-go-tool-trace.md) — The new (1.21+) tracer
- [09-tooling/14-go-tool-objdump-nm.md](09-tooling/14-go-tool-objdump-nm.md)
- [09-tooling/15-go-env.md](09-tooling/15-go-env.md)
- [09-tooling/16-gopls.md](09-tooling/16-gopls.md) — The official LSP
- [09-tooling/17-golangci-lint.md](09-tooling/17-golangci-lint.md) — Configuration, picking linters
- [09-tooling/18-staticcheck.md](09-tooling/18-staticcheck.md) — `SA*` check catalog
- [09-tooling/19-delve.md](09-tooling/19-delve.md) — `dlv debug`, headless mode, attach
- [09-tooling/20-govulncheck.md](09-tooling/20-govulncheck.md)
- [09-tooling/21-benchstat.md](09-tooling/21-benchstat.md)
- [09-tooling/22-goimports-gofumpt.md](09-tooling/22-goimports-gofumpt.md)
- [09-tooling/23-third-party-tools.md](09-tooling/23-third-party-tools.md) — `air`, `goreleaser`, `mage`, `task`

### Part 10 — Testing
- [10-testing/01-unit-tests.md](10-testing/01-unit-tests.md)
- [10-testing/02-table-driven.md](10-testing/02-table-driven.md)
- [10-testing/03-subtests-and-tparallel.md](10-testing/03-subtests-and-tparallel.md)
- [10-testing/04-benchmarks.md](10-testing/04-benchmarks.md) — `b.Loop` (since 1.24)
- [10-testing/05-fuzzing.md](10-testing/05-fuzzing.md) — Since 1.18
- [10-testing/06-examples-as-tests.md](10-testing/06-examples-as-tests.md)
- [10-testing/07-testdata-and-golden-files.md](10-testing/07-testdata-and-golden-files.md)
- [10-testing/08-httptest.md](10-testing/08-httptest.md)
- [10-testing/09-mocking-strategies.md](10-testing/09-mocking-strategies.md) — Interfaces, gomock, mockery
- [10-testing/10-integration-tests.md](10-testing/10-integration-tests.md) — `testcontainers-go`
- [10-testing/11-coverage.md](10-testing/11-coverage.md) — `go test -cover`, coverage of integration tests (1.20+)
- [10-testing/12-testify-controversy.md](10-testing/12-testify-controversy.md) — Why some teams ban it
- [10-testing/13-testing-synctest.md](10-testing/13-testing-synctest.md)

### Part 11 — Reflection, Unsafe & Low-Level
- [11-low-level/01-reflect-fundamentals.md](11-low-level/01-reflect-fundamentals.md) — Three laws of reflection (Pike)
- [11-low-level/02-reflect-performance.md](11-low-level/02-reflect-performance.md) — Why it's slow, when it's fine
- [11-low-level/03-unsafe-pointer.md](11-low-level/03-unsafe-pointer.md) — The 6 valid patterns from the `unsafe` doc
- [11-low-level/04-unsafe-slice-string.md](11-low-level/04-unsafe-slice-string.md) — `unsafe.Slice`, `unsafe.String` (since 1.20)
- [11-low-level/05-cgo.md](11-low-level/05-cgo.md) — `import "C"`, calling overhead, GC interaction
- [11-low-level/06-cgo-pitfalls.md](11-low-level/06-cgo-pitfalls.md) — Pointer passing rules, the cgo checker
- [11-low-level/07-go-assembly.md](11-low-level/07-go-assembly.md) — Plan 9 dialect, `TEXT` directive, NOSPLIT
- [11-low-level/08-linkname.md](11-low-level/08-linkname.md) — `//go:linkname`, runtime hacks, 1.23 restrictions
- [11-low-level/09-build-tags.md](11-low-level/09-build-tags.md) — `//go:build`, `_GOOS_GOARCH.go`
- [11-low-level/10-go-directives.md](11-low-level/10-go-directives.md) — `//go:noinline`, `//go:nosplit`, `//go:noescape`, `//go:nocheckptr`

### Part 12 — Runtime Internals
- [12-runtime/01-memory-layout.md](12-runtime/01-memory-layout.md) — mheap, mcentral, mcache, mspan
- [12-runtime/02-stack-vs-heap.md](12-runtime/02-stack-vs-heap.md) — Escape analysis, `-gcflags=-m`
- [12-runtime/03-garbage-collector.md](12-runtime/03-garbage-collector.md) — Tricolor mark-sweep, write barriers, soft memory limit (1.19+)
- [12-runtime/04-gc-tuning.md](12-runtime/04-gc-tuning.md) — `GOGC`, `GOMEMLIMIT`, `debug.SetGCPercent`
- [12-runtime/05-goroutine-internals.md](12-runtime/05-goroutine-internals.md) — `g` struct, stack growth, preemption
- [12-runtime/06-scheduler-deep-dive.md](12-runtime/06-scheduler-deep-dive.md) — G-M-P, netpoller, sysmon, asynchronous preemption
- [12-runtime/07-finalizers-cleanups.md](12-runtime/07-finalizers-cleanups.md) — `SetFinalizer`, `runtime.AddCleanup` (since 1.24)
- [12-runtime/08-traceback-and-panic.md](12-runtime/08-traceback-and-panic.md)
- [12-runtime/09-runtime-pprof-internals.md](12-runtime/09-runtime-pprof-internals.md)
- [12-runtime/10-compiler-pipeline.md](12-runtime/10-compiler-pipeline.md) — Parse → typecheck → SSA → assembly
- [12-runtime/11-ssa-and-inlining.md](12-runtime/11-ssa-and-inlining.md) — Inlining budget, mid-stack inlining
- [12-runtime/12-bounds-check-elimination.md](12-runtime/12-bounds-check-elimination.md)
- [12-runtime/13-pgo.md](12-runtime/13-pgo.md) — Profile-guided optimization (since 1.20, GA 1.21)

### Part 13 — Performance & Profiling
- [13-performance/01-profiling-cpu.md](13-performance/01-profiling-cpu.md)
- [13-performance/02-profiling-memory.md](13-performance/02-profiling-memory.md) — Heap, allocs, in-use
- [13-performance/03-profiling-block-mutex.md](13-performance/03-profiling-block-mutex.md)
- [13-performance/04-profiling-goroutine.md](13-performance/04-profiling-goroutine.md)
- [13-performance/05-execution-tracer.md](13-performance/05-execution-tracer.md)
- [13-performance/06-benchmarking-methodology.md](13-performance/06-benchmarking-methodology.md) — `benchstat`, statistical significance
- [13-performance/07-allocation-reduction.md](13-performance/07-allocation-reduction.md) — sync.Pool, pre-sizing slices/maps
- [13-performance/08-string-byte-conversion.md](13-performance/08-string-byte-conversion.md) — Compiler optimizations for `m[string(b)]`
- [13-performance/09-cache-friendly-code.md](13-performance/09-cache-friendly-code.md) — Field ordering, false sharing, padding
- [13-performance/10-simd-via-assembly.md](13-performance/10-simd-via-assembly.md) — `avo`, when it's worth it
- [13-performance/11-pgo-in-practice.md](13-performance/11-pgo-in-practice.md)

### Part 14 — Build, Deploy, Distribute
- [14-build-deploy/01-cross-compilation.md](14-build-deploy/01-cross-compilation.md) — `GOOS`, `GOARCH`, `GOARM`
- [14-build-deploy/02-static-vs-dynamic-binaries.md](14-build-deploy/02-static-vs-dynamic-binaries.md) — `CGO_ENABLED=0`, musl
- [14-build-deploy/03-ldflags-and-versioning.md](14-build-deploy/03-ldflags-and-versioning.md) — Injecting build info
- [14-build-deploy/04-reproducible-builds.md](14-build-deploy/04-reproducible-builds.md) — `-trimpath`, `SOURCE_DATE_EPOCH`
- [14-build-deploy/05-docker-for-go.md](14-build-deploy/05-docker-for-go.md) — Distroless, scratch, multi-stage
- [14-build-deploy/06-wasm.md](14-build-deploy/06-wasm.md) — `js/wasm`, `wasip1` (since 1.21)
- [14-build-deploy/07-plugins.md](14-build-deploy/07-plugins.md) — `plugin` package — why nobody uses it
- [14-build-deploy/08-goreleaser.md](14-build-deploy/08-goreleaser.md)

### Part 15 — Web, Networking, RPC
- [15-web-net/01-http-server-patterns.md](15-web-net/01-http-server-patterns.md) — Routing, middleware, graceful shutdown
- [15-web-net/02-http-client-patterns.md](15-web-net/02-http-client-patterns.md) — Retries, circuit breakers, connection pool tuning
- [15-web-net/03-rest-apis.md](15-web-net/03-rest-apis.md)
- [15-web-net/04-grpc.md](15-web-net/04-grpc.md) — `grpc-go`, interceptors, streaming
- [15-web-net/05-graphql.md](15-web-net/05-graphql.md) — gqlgen
- [15-web-net/06-websockets.md](15-web-net/06-websockets.md) — `nhooyr/websocket`, `gorilla/websocket`
- [15-web-net/07-frameworks.md](15-web-net/07-frameworks.md) — chi, gin, echo, fiber — comparison
- [15-web-net/08-tls.md](15-web-net/08-tls.md) — `crypto/tls`, ALPN, mTLS
- [15-web-net/09-http3-quic.md](15-web-net/09-http3-quic.md) — `quic-go`
- [15-web-net/10-protobuf.md](15-web-net/10-protobuf.md) — `google.golang.org/protobuf`

### Part 16 — Data, Storage, Persistence
- [16-data/01-database-sql.md](16-data/01-database-sql.md) — The official driver-agnostic API
- [16-data/02-pgx.md](16-data/02-pgx.md) — High-perf PostgreSQL
- [16-data/03-mysql-driver.md](16-data/03-mysql-driver.md)
- [16-data/04-sqlx-and-sqlc.md](16-data/04-sqlx-and-sqlc.md) — Pick a side
- [16-data/05-orms-gorm-ent.md](16-data/05-orms-gorm-ent.md) — When ORMs help and when they don't
- [16-data/06-redis.md](16-data/06-redis.md) — `go-redis`, `rueidis`
- [16-data/07-mongodb.md](16-data/07-mongodb.md)
- [16-data/08-embedded-stores.md](16-data/08-embedded-stores.md) — BoltDB, BadgerDB, Pebble
- [16-data/09-cache-patterns.md](16-data/09-cache-patterns.md) — `ristretto`, `bigcache`, `freecache`

### Part 17 — Observability
- [17-observability/01-structured-logging-slog.md](17-observability/01-structured-logging-slog.md)
- [17-observability/02-metrics-prometheus.md](17-observability/02-metrics-prometheus.md) — `client_golang`
- [17-observability/03-distributed-tracing-otel.md](17-observability/03-distributed-tracing-otel.md) — OpenTelemetry Go
- [17-observability/04-runtime-metrics.md](17-observability/04-runtime-metrics.md) — `runtime/metrics`
- [17-observability/05-continuous-profiling.md](17-observability/05-continuous-profiling.md) — Parca, Pyroscope, Polar Signals
- [17-observability/06-error-tracking.md](17-observability/06-error-tracking.md) — Sentry, Bugsnag patterns

### Part 18 — Security
- [18-security/01-govulncheck-in-ci.md](18-security/01-govulncheck-in-ci.md)
- [18-security/02-supply-chain.md](18-security/02-supply-chain.md) — Sigstore, SLSA, module checksum DB
- [18-security/03-secure-coding.md](18-security/03-secure-coding.md) — Input validation, SSRF, path traversal, `os.Root` (1.24+)
- [18-security/04-crypto-best-practices.md](18-security/04-crypto-best-practices.md)
- [18-security/05-fips-140.md](18-security/05-fips-140.md) — Native FIPS mode (1.24+)
- [18-security/06-secrets-management.md](18-security/06-secrets-management.md)

### Part 19 — Design Patterns & Architecture
- [19-patterns/01-functional-options.md](19-patterns/01-functional-options.md) — Rob Pike's pattern
- [19-patterns/02-builder.md](19-patterns/02-builder.md)
- [19-patterns/03-dependency-injection.md](19-patterns/03-dependency-injection.md) — Manual, `wire`, `fx`
- [19-patterns/04-middleware-chain.md](19-patterns/04-middleware-chain.md)
- [19-patterns/05-repository-pattern.md](19-patterns/05-repository-pattern.md)
- [19-patterns/06-clean-hexagonal.md](19-patterns/06-clean-hexagonal.md) — Ports and adapters in Go
- [19-patterns/07-cqrs-event-sourcing.md](19-patterns/07-cqrs-event-sourcing.md)
- [19-patterns/08-state-machines.md](19-patterns/08-state-machines.md)
- [19-patterns/09-functional-go.md](19-patterns/09-functional-go.md) — How far you can push it, and when to stop

### Part 20 — Go at Scale (Big Tech Case Studies)
- [20-big-tech/01-google-kubernetes.md](20-big-tech/01-google-kubernetes.md) — Architecture, why Go, what hurts at scale
- [20-big-tech/02-google-gvisor.md](20-big-tech/02-google-gvisor.md) — User-space kernel in Go
- [20-big-tech/03-uber-microservices.md](20-big-tech/03-uber-microservices.md) — 3500+ Go services, Cadence
- [20-big-tech/04-cloudflare.md](20-big-tech/04-cloudflare.md) — Edge in Go, BoringCrypto, RRDNS
- [20-big-tech/05-dropbox-magic-pocket.md](20-big-tech/05-dropbox-magic-pocket.md)
- [20-big-tech/06-discord-state-service.md](20-big-tech/06-discord-state-service.md) — The infamous GC blog
- [20-big-tech/07-twitch.md](20-big-tech/07-twitch.md) — Chat at scale
- [20-big-tech/08-netflix.md](20-big-tech/08-netflix.md)
- [20-big-tech/09-tailscale.md](20-big-tech/09-tailscale.md) — Wireguard, control plane
- [20-big-tech/10-hashicorp-stack.md](20-big-tech/10-hashicorp-stack.md) — Terraform, Vault, Consul, Nomad
- [20-big-tech/11-cncf-projects.md](20-big-tech/11-cncf-projects.md) — Prometheus, etcd, containerd, Helm
- [20-big-tech/12-cockroachdb.md](20-big-tech/12-cockroachdb.md) — Distributed SQL in Go
- [20-big-tech/13-influxdb.md](20-big-tech/13-influxdb.md)
- [20-big-tech/14-caddy-traefik.md](20-big-tech/14-caddy-traefik.md) — Modern web servers
- [20-big-tech/15-minio.md](20-big-tech/15-minio.md) — S3-compatible object storage
- [20-big-tech/16-anti-case-studies.md](20-big-tech/16-anti-case-studies.md) — Teams that moved off Go (and why)

### Part 21 — Go 1.18 → 1.26 Feature Reference
- [21-version-features/01-go-1.18-generics-fuzzing-workspaces.md](21-version-features/01-go-1.18-generics-fuzzing-workspaces.md)
- [21-version-features/02-go-1.19-doc-comments-soft-memlimit.md](21-version-features/02-go-1.19-doc-comments-soft-memlimit.md)
- [21-version-features/03-go-1.20-errors-join-pgo-preview.md](21-version-features/03-go-1.20-errors-join-pgo-preview.md)
- [21-version-features/04-go-1.21-slices-maps-cmp-slog-pgo-ga.md](21-version-features/04-go-1.21-slices-maps-cmp-slog-pgo-ga.md)
- [21-version-features/05-go-1.22-loop-var-rangeint-http-routing.md](21-version-features/05-go-1.22-loop-var-rangeint-http-routing.md)
- [21-version-features/06-go-1.23-rangefunc-unique-iterators.md](21-version-features/06-go-1.23-rangefunc-unique-iterators.md)
- [21-version-features/07-go-1.24-weak-cleanup-synctest-osroot.md](21-version-features/07-go-1.24-weak-cleanup-synctest-osroot.md)
- [21-version-features/08-go-1.25-structs-waitgroup-go-greentea-gc.md](21-version-features/08-go-1.25-structs-waitgroup-go-greentea-gc.md)
- [21-version-features/09-go-1.26-and-beyond.md](21-version-features/09-go-1.26-and-beyond.md)

### Part 22 — Idioms & Style
- [22-idioms/01-effective-go-revisited.md](22-idioms/01-effective-go-revisited.md)
- [22-idioms/02-google-style-guide.md](22-idioms/02-google-style-guide.md) — `google.github.io/styleguide/go/`
- [22-idioms/03-uber-style-guide.md](22-idioms/03-uber-style-guide.md)
- [22-idioms/04-code-review-comments.md](22-idioms/04-code-review-comments.md) — The canonical wiki
- [22-idioms/05-naming-conventions.md](22-idioms/05-naming-conventions.md)
- [22-idioms/06-doc-comments.md](22-idioms/06-doc-comments.md) — Since 1.19 they're Markdown-ish
- [22-idioms/07-package-design.md](22-idioms/07-package-design.md) — Cohesion, naming, cyclic dependencies

### Part 23 — Interop & Foreign Code
- [23-interop/01-cgo-deep-dive.md](23-interop/01-cgo-deep-dive.md)
- [23-interop/02-calling-go-from-c.md](23-interop/02-calling-go-from-c.md) — `-buildmode=c-shared`, `c-archive`
- [23-interop/03-calling-go-from-python.md](23-interop/03-calling-go-from-python.md)
- [23-interop/04-calling-go-from-rust-and-vice-versa.md](23-interop/04-calling-go-from-rust-and-vice-versa.md)
- [23-interop/05-ffi-without-cgo.md](23-interop/05-ffi-without-cgo.md) — `purego`, syscall

### Part 24 — Misc / Frontier
- [24-frontier/01-experimental-arenas.md](24-frontier/01-experimental-arenas.md) — Why they were pulled
- [24-frontier/02-coroutines-and-iter-pull.md](24-frontier/02-coroutines-and-iter-pull.md)
- [24-frontier/03-generic-type-aliases.md](24-frontier/03-generic-type-aliases.md) — Since 1.24
- [24-frontier/04-range-over-func-advanced.md](24-frontier/04-range-over-func-advanced.md)
- [24-frontier/05-encoding-json-v2.md](24-frontier/05-encoding-json-v2.md) — Experimental → 1.26
- [24-frontier/06-greenteagc.md](24-frontier/06-greenteagc.md) — The 1.25+ GC redesign
- [24-frontier/07-future-proposals.md](24-frontier/07-future-proposals.md) — Active language proposals worth watching

### Appendices
- [appendix/A-glossary.md](appendix/A-glossary.md)
- [appendix/B-cheatsheets.md](appendix/B-cheatsheets.md) — Common stdlib idioms, one-liners
- [appendix/C-bibliography.md](appendix/C-bibliography.md) — Canonical sources, books, talks
- [appendix/D-talks-and-videos.md](appendix/D-talks-and-videos.md) — GopherCon, dotGo, GoLab
- [appendix/E-people-to-follow.md](appendix/E-people-to-follow.md) — Russ Cox, Bryan C. Mills, Filippo Valsorda, Damian Gryski, Dave Cheney, etc.

---

## Canonical Resources

Reference these before writing any page:

- **Language spec**: https://go.dev/ref/spec
- **Memory model**: https://go.dev/ref/mem
- **Modules reference**: https://go.dev/ref/mod
- **Standard library**: https://pkg.go.dev/std
- **Source code**: https://github.com/golang/go (pin to `release-branch.go1.26`)
- **Proposals**: https://github.com/golang/proposal
- **Release notes**: https://go.dev/doc/devel/release
- **Effective Go**: https://go.dev/doc/effective_go
- **Go blog**: https://go.dev/blog
- **Google Go style**: https://google.github.io/styleguide/go/
- **Uber Go style**: https://github.com/uber-go/guide
- **Go wiki — code review comments**: https://go.dev/wiki/CodeReviewComments
- **research!rsc**: https://research.swtch.com (Russ Cox)
- **Dave Cheney**: https://dave.cheney.net
- **High Performance Go**: https://github.com/dgryski/go-perfbook

# `flag` — Minimalist CLI Flag Parsing

## TL;DR

`flag` is the stdlib CLI flag parser: declarative `flag.String`/`Int`/`Bool`/`Duration` registration, then `flag.Parse()`. Supports `-flag value` and `-flag=value`; single-dash and double-dash both work. No subcommands, no shell completion, no env-var binding. For complex CLIs reach for Cobra, urfave/cli, or kong — but for simple tools, stdlib `flag` is enough and ships zero dependencies.

## Mental Model

```
Register flags before Parse:
    name  := flag.String("name", "default", "usage")
    age   := flag.Int("age", 0, "usage")
    debug := flag.Bool("debug", false, "usage")

flag.Parse() walks os.Args[1:]; positional remain in flag.Args().

Custom types: implement flag.Value (String() + Set(s)).
```

## Syntax & Basic Usage

```go
package main

import (
	"flag"
	"fmt"
)

func main() {
	name := flag.String("name", "world", "greeting target")
	count := flag.Int("count", 1, "how many times")
	flag.Parse()

	for i := 0; i < *count; i++ {
		fmt.Printf("hello, %s\n", *name)
	}
	// Run: go run . -name=Go -count=3
	// Output:
	// hello, Go
	// hello, Go
	// hello, Go
}
```

## Deep Dive

### Built-in types

```go
name := flag.String("name", "default", "usage")
n    := flag.Int("n", 0, "usage")
i64  := flag.Int64("i64", 0, "usage")
u    := flag.Uint("u", 0, "usage")
b    := flag.Bool("b", false, "usage")
f    := flag.Float64("f", 0.0, "usage")
d    := flag.Duration("d", time.Second, "usage")
```

Each returns a pointer.

### `*Var` variants

```go
var name string
flag.StringVar(&name, "name", "default", "usage")
```

Useful when you have an existing variable (e.g., a config struct field).

### Custom types — `flag.Value` interface

```go
type StringSlice []string

func (s *StringSlice) String() string     { return strings.Join(*s, ",") }
func (s *StringSlice) Set(v string) error { *s = append(*s, v); return nil }

var hosts StringSlice
flag.Var(&hosts, "host", "host (repeatable)")
flag.Parse()
// invocation: -host a -host b -host c
// hosts == ["a","b","c"]
```

Any type implementing `flag.Value` can be registered with `flag.Var`.

### Positional arguments

After `Parse`, `flag.Args()` returns positional args. `flag.NArg()` is the count.

```go
flag.Parse()
files := flag.Args() // e.g., the trailing paths after flags
```

### `FlagSet` for sub-commands

```go
serveCmd := flag.NewFlagSet("serve", flag.ExitOnError)
addr := serveCmd.String("addr", ":8080", "listen address")

if len(os.Args) > 1 && os.Args[1] == "serve" {
	serveCmd.Parse(os.Args[2:])
	run(*addr)
}
```

This is the foundation for subcommands. For more than 2-3 subcommands, use a third-party library.

### Usage and help

```go
flag.Usage = func() {
	fmt.Fprintln(os.Stderr, "myapp [flags] file...")
	flag.PrintDefaults()
}
```

`-h` and `-help` call Usage automatically. `-flag` followed by unknown flag also calls it.

### Error handling

`flag.NewFlagSet(name, flag.ErrorHandling)`:

- `ContinueOnError` — `Parse` returns an error; you decide.
- `ExitOnError` — calls `os.Exit(2)` on error.
- `PanicOnError` — panics.

The default `flag.CommandLine` is `ExitOnError`.

### Environment variables

Stdlib `flag` does NOT bind env vars. Pattern:

```go
addr := flag.String("addr", lookupEnv("ADDR", ":8080"), "address")

func lookupEnv(key, def string) string {
	if v, ok := os.LookupEnv(key); ok { return v }
	return def
}
```

## Standard Library Hooks

- `os.Args` — what `flag` parses.
- `flag.Value` interface — for custom types.

## Real-World Patterns

### 1. Service main with flags + env override

```go
func main() {
	var (
		addr   = flag.String("addr", lookupEnv("ADDR", ":8080"), "listen addr")
		dbURL  = flag.String("db",   lookupEnv("DB_URL", ""),     "database url")
		debug  = flag.Bool("debug",  lookupBool("DEBUG", false),  "debug logging")
	)
	flag.Parse()

	level := slog.LevelInfo
	if *debug { level = slog.LevelDebug }
	slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: level})))

	// ... run
}
```

### 2. Subcommands

```go
func main() {
	if len(os.Args) < 2 { usage(); os.Exit(2) }
	switch os.Args[1] {
	case "serve": serve(os.Args[2:])
	case "migrate": migrate(os.Args[2:])
	default: usage(); os.Exit(2)
	}
}

func serve(args []string) {
	fs := flag.NewFlagSet("serve", flag.ExitOnError)
	addr := fs.String("addr", ":8080", "address")
	fs.Parse(args)
	runServe(*addr)
}
```

### 3. Repeatable flag

```go
type repeatable []string
func (r *repeatable) String() string     { return strings.Join(*r, ",") }
func (r *repeatable) Set(v string) error { *r = append(*r, v); return nil }

var tags repeatable
flag.Var(&tags, "tag", "tag (may be specified multiple times)")
```

### 4. Boolean with default-true

Bool flags work as `-flag` (set true) or `-flag=false`. For a default-true flag:

```go
verbose := flag.Bool("verbose", true, "verbose output")
// -verbose=false to silence
```

### 5. CLI with help and version

```go
showVer := flag.Bool("version", false, "print version and exit")
flag.Parse()
if *showVer {
	info, _ := debug.ReadBuildInfo()
	fmt.Println(info.Main.Version)
	os.Exit(0)
}
```

## Anti-Patterns & Gotchas

**Using `flag` for complex CLIs.** Reach for Cobra/urfave/kong above ~5 commands.

**Calling `flag.Parse` from inside a goroutine.** Race with other accesses.

**Defining flags after `Parse`.** They won't be registered.

**Mixing `os.Args[1:]` parsing with `flag.Args()` semantics.** Use one or the other.

**Modifying flag values after `Parse`.** Confusing for users; the flag won't reset on next Parse call.

**Forgetting `*` to deref pointer.** `if name == "x"` (pointer vs string) is a compile error, but `fmt.Println(name)` silently prints a pointer.

**Custom `flag.Value` whose `Set` returns nil even on bad input.** Confusing failure mode.

**`-help` showing nothing helpful.** Customize `flag.Usage`.

## Performance Notes

Negligible. `Parse` walks args once. Don't worry.

## How Big Companies Use It

- **The Go toolchain (`go` command)** uses `flag` extensively for subcommands.
- **Caddy** uses Cobra externally but stdlib `flag` for some internal tools.
- **Tailscale CLI** uses a custom wrapper over stdlib `flag` plus subcommand routing.
- **Lots of internal Google tools** use `flag` directly.

## Source Code References

Pinned to `go1.26`.

- `flag`: [`src/flag/flag.go`](https://github.com/golang/go/blob/master/src/flag/flag.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/flag.
- Effective Go on the flag package: https://go.dev/doc/effective_go.
- Cobra: https://github.com/spf13/cobra (when stdlib isn't enough).

## Exercises / Self-Check

1. Build a CLI with `-name`, `-count`, and positional file args. Print each file `count` times prefixed by `name:`.
2. Implement a repeatable `-host` flag via `flag.Value`.
3. Add an env-var fallback for every flag.
4. Implement two subcommands (`serve`, `migrate`) using `flag.NewFlagSet`.
5. Why doesn't stdlib `flag` support `--long=value` with two dashes? It does (one or two dashes both work). Verify and explain in docs.

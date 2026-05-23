# `regexp` — Regular Expressions

## TL;DR

`regexp` implements RE2 syntax — guaranteed linear time, no catastrophic backtracking, no backreferences, no lookbehind/lookahead. Compile once with `regexp.MustCompile` (typically at package level), reuse the `*Regexp` across goroutines (safe). For static patterns, fixed string ops (`strings.Index`, `strings.Contains`) are 10-100× faster than regex — only reach for regex when actual regex features are needed.

## Mental Model

```
MustCompile(pattern) → *Regexp  (thread-safe, immutable)
   ├─ Match(b)           → bool
   ├─ FindString(s)      → first match string
   ├─ FindAllString(s,n) → up to n matches
   ├─ FindStringSubmatch(s) → []string with capture groups
   ├─ ReplaceAllString(s, repl)
   └─ Split(s, n)

RE2 = linear-time finite automaton; some PCRE features missing intentionally.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"regexp"
)

var re = regexp.MustCompile(`(\w+)\s+(\d+)`)

func main() {
	m := re.FindStringSubmatch("hello 42 world 7")
	fmt.Println(m) // [hello 42 hello 42]

	all := re.FindAllStringSubmatch("hello 42 world 7", -1)
	fmt.Println(all)
	// Output:
	// [hello 42 hello 42]
	// [[hello 42 hello 42] [world 7 world 7]]
}
```

## Deep Dive

### Compilation

- `regexp.Compile(p)` — returns `(*Regexp, error)`.
- `regexp.MustCompile(p)` — panics on error; use for package-level constants.
- `regexp.CompilePOSIX(p)` — POSIX-leftmost-longest semantics (rarely needed).

Compile is expensive — do it once. The compiled `*Regexp` is safe for concurrent use.

### Pattern syntax (RE2)

- `.` — any byte (not rune by default; see `(?s)` and `(?u)`).
- `\d`, `\w`, `\s` — digit, word, space (ASCII unless `(?u)`).
- `\p{L}`, `\p{Han}` — Unicode classes.
- `^`, `$` — line anchors (per-line if `(?m)`).
- `\A`, `\z` — text anchors.
- `(...)`, `(?P<name>...)` — capture groups.
- `(?:...)` — non-capturing.
- `*`, `+`, `?` — greedy quantifiers.
- `*?`, `+?`, `??` — lazy.
- `{n}`, `{n,m}` — counted.
- `[abc]`, `[^abc]`, `[a-z]` — classes.

**Not supported (intentional):**

- Backreferences (`\1`).
- Lookahead/lookbehind (`(?=...)`, `(?!...)`).

These features would break the linear-time guarantee.

### Methods

`Find`/`FindAll`/`FindString`/`FindAllString`/`FindIndex`/`FindAllIndex` × `Submatch` variants × `Index` variants. Combinatorial; pick what you need:

- `FindString` — first match as string.
- `FindAllString(s, n)` — all matches (n = -1 for all).
- `FindStringSubmatch(s)` — first match + captures.
- `FindAllStringSubmatch(s, n)` — every match with captures.
- `FindAllStringIndex` — byte positions.
- `ReplaceAllString(s, repl)` — `$1`, `$2`, `${name}` in repl.
- `ReplaceAllStringFunc(s, fn)` — fn receives each match.

For byte versions: replace `String` with nothing (`FindAll`, `Find`).

### Named captures

```go
re := regexp.MustCompile(`(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})`)
m := re.FindStringSubmatch("2026-05-21")
for i, name := range re.SubexpNames() {
	if name != "" { fmt.Println(name, m[i]) }
}
// year 2026
// month 05
// day 21
```

### Replacement syntax

```go
re := regexp.MustCompile(`(\w+):(\d+)`)
re.ReplaceAllString("host:80 host2:443", "$1 on $2")
// "host on 80 host2 on 443"
```

`$$` for a literal `$`. `${name}` for named captures.

### Flags

```go
(?i)foo   // case-insensitive
(?s).     // . matches newline
(?m)^     // ^/$ match per line
(?u)\w    // Unicode \w
```

Combine: `(?ims)foo`.

### Performance characteristics

- Linear in input length, no matter the pattern complexity.
- Multiple matches via `FindAll` allocate per match; for hot paths use `FindAllIndex` + slice the input.
- `regexp` is slow for simple cases — `strings.Contains` is much faster for literal substrings.

## Standard Library Hooks

- `strings.Index`, `strings.Contains`, `strings.HasPrefix` — fast paths for literal patterns.
- `bytes` mirrors — `Find` etc. on `[]byte`.
- `regexp/syntax` — parse and inspect regex ASTs.

## Real-World Patterns

### 1. Validate email-ish input

```go
var emailRe = regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$`)

func validEmail(s string) bool {
	return emailRe.MatchString(strings.ToLower(s))
}
```

(For real email validation, use a parser; regex is heuristic.)

### 2. Extract structured fields from a log line

```go
var logRe = regexp.MustCompile(`^(?P<ts>\S+)\s+(?P<level>\w+)\s+(?P<msg>.+)$`)

m := logRe.FindStringSubmatch(line)
if m != nil {
	ts := m[logRe.SubexpIndex("ts")]
	lvl := m[logRe.SubexpIndex("level")]
	// ...
}
```

Use case: ad-hoc log parsing.

### 3. Mass replace with function

```go
var urlRe = regexp.MustCompile(`https?://\S+`)
out := urlRe.ReplaceAllStringFunc(text, func(match string) string {
	return "<" + match + ">"
})
```

### 4. Split CSV-like with embedded quotes (don't)

Regex is the wrong tool for proper CSV. Use `encoding/csv`. Reach for regex only on truly ad-hoc fields.

### 5. Find all numbers in a doc

```go
var numRe = regexp.MustCompile(`-?\d+(\.\d+)?`)
nums := numRe.FindAllString(text, -1)
```

## Anti-Patterns & Gotchas

**Compiling regex in a hot loop.** Always compile once.

**Using regex when `strings.Contains` would do.** Much slower.

**Expecting backreferences or lookahead.** RE2 doesn't support them; restructure.

**`regexp.MatchString` with `MustCompile` together** — `MatchString(pattern, s)` compiles each call.

**Greedy matching unintended.** `<.*>` matches as much as possible. Use `<.*?>` or `<[^>]+>`.

**`.` not matching newlines.** Add `(?s)` if you want it to.

**Case sensitivity surprises.** Add `(?i)`.

**Capture group index vs name confusion.** Use `SubexpIndex(name)`.

**Replacement string escapes.** `$1` in repl; use `$$` for literal `$`.

**Building regex from user input without escaping.** Use `regexp.QuoteMeta(s)`.

## Performance Notes

- Linear time guaranteed (no PCRE-style exponential blowup).
- Constant factor: regex is 5-50× slower than `strings.Index` for literal substrings.
- `Find*Index` returns positions; cheaper than `Find*Submatch` which copies into `[]string`.
- For 100k+ patterns, build an Aho-Corasick automaton (third-party) or use `regexp.MustCompile` with `|` of all patterns.

## How Big Companies Use It

- **Caddy** uses regex in path matcher syntax.
- **Hugo** uses regex in shortcode and pattern matching.
- **gopls** uses regex for some token classification.
- **Prometheus** uses regex in label matchers (with strict syntax).

## Source Code References

Pinned to `go1.26`.

- `regexp`: [`src/regexp/`](https://github.com/golang/go/tree/master/src/regexp).
- RE2 syntax parser: [`src/regexp/syntax/`](https://github.com/golang/go/tree/master/src/regexp/syntax).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/regexp, /regexp/syntax.
- Russ Cox, "Regular Expression Matching Can Be Simple And Fast": https://swtch.com/~rsc/regexp/regexp1.html — origin of RE2.
- RE2 syntax reference: https://github.com/google/re2/wiki/Syntax.

## Exercises / Self-Check

1. Write a regex that matches IPv4 addresses; test it on edge cases (`999.0.0.1` should NOT match).
2. Extract all hex colors `#abc` and `#aabbcc` from a CSS string. Use a single regex.
3. Why is `(a+)+` safe in Go but disastrous in PCRE? Read the RE2 paper.
4. Benchmark `strings.Contains(s, "x")` vs `regexp.MustCompile("x").MatchString(s)`.
5. Use named groups to parse `key=value, key=value` lines into a map.

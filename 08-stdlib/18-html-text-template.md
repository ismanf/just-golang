# `text/template` and `html/template`

## TL;DR

`text/template` is Go's general string-templating engine with pipelines, control flow, and user-defined functions. `html/template` is the same engine plus contextual auto-escaping (HTML, JS, CSS, URL contexts) — use it for HTML, never `text/template`, to avoid XSS. Both pre-parse and validate templates at load time; the runtime `Execute` is fast and write-to-`io.Writer`.

## Mental Model

```
text/template:
    "{{ .Name }} has {{ len .Items }} items"
                  ↑ pipeline: function or method on .

Auto-escape (html/template only):
    href="{{ .URL }}"          → URL-context escape
    <script>{{ .Data }}</script> → JS-context escape
    {{ .Body }}                  → HTML-text escape
    
Parse once at startup, Execute many times.
```

## Syntax & Basic Usage

```go
package main

import (
	"html/template"
	"os"
)

type Page struct {
	Title string
	Items []string
}

func main() {
	tmpl := template.Must(template.New("page").Parse(`
<h1>{{ .Title }}</h1>
<ul>{{ range .Items }}<li>{{ . }}</li>{{ end }}</ul>
`))
	tmpl.Execute(os.Stdout, Page{Title: "Demo", Items: []string{"a", "b", "<script>"}})
	// Output:
	//
	// <h1>Demo</h1>
	// <ul><li>a</li><li>b</li><li>&lt;script&gt;</li></ul>
}
```

Note `<script>` is escaped to `&lt;script&gt;` — XSS prevention.

## Deep Dive

### Actions and pipelines

```
{{ .Field }}            field access on current dot
{{ .Method }}           method call (no args)
{{ funcName .Arg }}     function call
{{ pipeline | funcName }} pipe-chain
{{ if .Cond }}A{{ else }}B{{ end }}
{{ range .Items }}{{ . }}{{ else }}empty{{ end }}
{{ with .Sub }}{{ .X }}{{ end }} sets dot to .Sub
{{ define "name" }}...{{ end }} named sub-template
{{ template "name" .Data }}      execute sub-template
{{ block "name" .Data }}default{{ end }}  combine define + execute
```

### Functions

Built-in: `len`, `index`, `slice`, `print`, `printf`, `js`, `urlquery`, `html`, `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `and`, `or`, `not`.

Custom funcs:

```go
tmpl := template.New("x").Funcs(template.FuncMap{
	"upper": strings.ToUpper,
	"date":  func(t time.Time) string { return t.Format("2006-01-02") },
})
tmpl.Parse(`{{ upper .Name }} joined {{ date .When }}`)
```

Funcs must be registered **before** `Parse` (they're resolved at parse-time).

### Whitespace control

```
{{- .Name }}  // trim preceding whitespace
{{ .Name -}}  // trim following whitespace
```

### Loading templates from disk or `embed.FS`

```go
//go:embed templates/*.html
var files embed.FS

tmpl := template.Must(template.ParseFS(files, "templates/*.html"))
tmpl.ExecuteTemplate(w, "page.html", data)
```

Use `ExecuteTemplate(w, name, data)` when multiple templates share one set.

### `html/template` contextual escaping

```html
<a href="{{ .URL }}">{{ .Label }}</a>
<script>var x = {{ .Data }};</script>
<div style="color:{{ .Color }}">...</div>
```

Each context (URL, JS, CSS, attribute, comment) gets a different escaper. Don't manually escape — the engine knows the context.

### Safe types — opt out of escaping

```go
type SafeHTML template.HTML
type SafeURL  template.URL

tmpl.Execute(w, struct{ Body template.HTML }{Body: template.HTML("<b>bold</b>")})
```

Use sparingly; cast only data you trust.

### Caching parsed templates

```go
var page = template.Must(template.ParseFiles("page.html"))
// Reuse `page` across many requests.
```

Parsing is expensive; executing is cheap.

### Error handling

`Parse` errors at startup; `Execute` errors at runtime if a field/method doesn't exist (with `MissingKey` option) or a func panics. Use `template.Option("missingkey=error")` for strict mode.

### `text/template` vs `html/template`

| | text/template | html/template |
|---|---|---|
| Auto-escape | No | Yes (context-aware) |
| Output type | any | HTML |
| Use for | code generation, plain text, config files | HTML responses |

For HTML, **always** `html/template`. Never use `text/template` for HTML even if you "escape manually" — context-aware escaping is hard.

## Standard Library Hooks

- `io.Writer` — Execute writes anywhere.
- `embed.FS` — embed templates in the binary.
- `template.FuncMap` — register custom funcs.
- `html/template.HTML/JS/URL/CSS/HTMLAttr` — opt-out types.

## Real-World Patterns

### 1. Web page render with embedded templates

```go
//go:embed templates/*
var tmplFS embed.FS

var pages = template.Must(template.ParseFS(tmplFS, "templates/*.html"))

func handler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	pages.ExecuteTemplate(w, "index.html", data)
}
```

### 2. Code generator

```go
const tmplStr = `// generated; do not edit
package {{ .Package }}

{{ range .Types }}
type {{ .Name }} struct {
	{{ range .Fields }}{{ .Name }} {{ .Type }} ` + "`json:\"{{ .JSON }}\"`" + `
	{{ end }}
}
{{ end }}
`

tmpl := template.Must(template.New("gen").Parse(tmplStr))
tmpl.Execute(out, def)
```

Use case: `stringer`-style codegen, OpenAPI client generators.

### 3. Email template with shared layout

```go
{{ define "layout" }}<html><body>{{ block "content" . }}{{ end }}</body></html>{{ end }}

{{ define "welcome" }}
{{ template "layout" . }}
{{ define "content" }}Hello {{ .Name }}!{{ end }}
{{ end }}
```

```go
tmpl.ExecuteTemplate(buf, "welcome", User{Name: "Ada"})
```

### 4. Conditional and loop

```go
{{ if .User.IsAdmin }}<a href="/admin">Admin</a>{{ end }}
{{ range $i, $item := .Items }}<li>{{ $i }}: {{ $item }}</li>{{ end }}
```

### 5. Markdown email rendering with safe HTML

```go
import "github.com/yuin/goldmark"

var buf bytes.Buffer
goldmark.Convert(markdown, &buf)
tmpl.Execute(w, struct{ Body template.HTML }{Body: template.HTML(buf.String())})
```

Use case: rendering CMS content.

## Anti-Patterns & Gotchas

**Using `text/template` for HTML.** XSS vulnerability waiting to happen.

**Casting user input to `template.HTML`.** Bypasses escaping → XSS.

**Parsing templates per request.** Expensive. Parse once at startup.

**Forgetting `template.Must`.** Errors from Parse get ignored.

**Adding funcs after Parse.** Parse needs to know the funcs to resolve them.

**Calling methods with arguments in templates.** Templates support no-arg methods only. Wrap in a func.

**Accessing unexported fields.** Templates can't (no reflection access).

**Logic-heavy templates.** Push logic into Go code; keep templates close to "just rendering."

**Sub-template scoping confusion.** `template "x"` runs `x` with the *current* dot unless you pass `.Data`.

**Multi-template file conflict.** Two templates with the same name overwrite silently.

## Performance Notes

- Parse: hundreds of microseconds per template.
- Execute: tens of microseconds for typical pages; allocates depending on writers.
- `html/template` does extra escaping work — typically 10-20% slower than `text/template`.
- Reuse parsed templates; never re-parse in the request path.
- For very high RPS, consider precompiling to Go code with `quicktemplate` or `templ`.

## How Big Companies Use It

- **Hugo** uses `html/template` and `text/template` for theme rendering.
- **Caddy** uses `text/template` for config interpolation.
- **Grafana** (Go backend) uses `html/template` for alert templates.
- **Kubernetes** `kubectl` uses `text/template` for output formatting (`-o template=...`).

## Source Code References

Pinned to `go1.26`.

- `text/template`: [`src/text/template/`](https://github.com/golang/go/tree/master/src/text/template).
- `html/template`: [`src/html/template/`](https://github.com/golang/go/tree/master/src/html/template).
- Context-aware escaping: [`src/html/template/escape.go`](https://github.com/golang/go/blob/master/src/html/template/escape.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/text/template, /html/template.
- Go blog, "Generating code": https://go.dev/blog/generate (uses `text/template`).
- "Go's html/template contextual auto-escape" design doc: https://research.swtch.com/htmltemplate.

## Exercises / Self-Check

1. Build a page template that renders a list and conditionally a "no items" message. Use `range` with `{{ else }}`.
2. Try injecting `<script>alert(1)</script>` into a `text/template` page — see XSS happen — then switch to `html/template`.
3. Define a layout template with a `block` and override it in a page template.
4. Add a custom `fmtDate` func; render times in `2006-01-02`.
5. Why must funcs be registered before `Parse`? Trace through `text/template/parse`.

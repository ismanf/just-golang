# `encoding/xml`, `encoding/gob`, `encoding/csv`

## TL;DR

`encoding/xml` parses and emits XML via struct tags, similar to `encoding/json` but with namespace/attribute support and quirks around mixed content. `encoding/gob` is Go's native binary encoding — fast, self-describing, but **Go-only** (no other language consumes it). `encoding/csv` reads/writes RFC 4180 CSV; trivially the simplest of the three but the most-used in data pipelines. Use XML for legacy/enterprise interop, gob for Go-to-Go RPC where you don't want Protobuf, CSV for tabular data exchange.

## Mental Model

```
XML  : verbose, namespaces + attributes + elements + character data
       struct tags: `xml:"name,attr"`, `xml:"name,chardata"`, namespaces

gob  : self-describing binary; encoder transmits type info once per type-stream
       only Go reads/writes; great for caches, file snapshots, internal RPC

CSV  : RFC 4180 lines; quoting rules; configurable delimiter (Comma) and comment char
```

## Syntax & Basic Usage

```go
package main

import (
	"encoding/csv"
	"encoding/xml"
	"fmt"
	"os"
	"strings"
)

type Book struct {
	XMLName xml.Name `xml:"book"`
	ID      string   `xml:"id,attr"`
	Title   string   `xml:"title"`
}

func main() {
	b, _ := xml.MarshalIndent(Book{ID: "B1", Title: "Go"}, "", "  ")
	fmt.Println(string(b))

	r := csv.NewReader(strings.NewReader("a,b,c\n1,2,3\n4,5,6\n"))
	rows, _ := r.ReadAll()
	fmt.Println(rows)

	w := csv.NewWriter(os.Stdout)
	w.Write([]string{"x", "y"})
	w.Write([]string{"1", "2"})
	w.Flush()
	// Output:
	// <book id="B1">
	//   <title>Go</title>
	// </book>
	// [[a b c] [1 2 3] [4 5 6]]
	// x,y
	// 1,2
}
```

## Deep Dive

### `encoding/xml`

Tag forms:

- `xml:"name"` — element name.
- `xml:"name,attr"` — XML attribute.
- `xml:",chardata"` — text content.
- `xml:",cdata"` — CDATA section.
- `xml:",comment"` — XML comment.
- `xml:"name>sub"` — wrap in `<name><sub>...</sub></name>`.
- `xml:"namespace name"` — with namespace.

Streaming:

```go
dec := xml.NewDecoder(r)
for {
	tok, err := dec.Token()
	if err == io.EOF { break }
	if se, ok := tok.(xml.StartElement); ok && se.Name.Local == "book" {
		var b Book
		if err := dec.DecodeElement(&b, &se); err != nil { return err }
		process(b)
	}
}
```

Use case: large XML documents (RSS, OPML, SOAP).

Quirks:

- Mixed content (text + child elements) is poorly handled.
- Namespaces require careful tagging.
- No XML schema validation; use third-party libraries.
- No XPath; use `golang.org/x/net/html` or `etree` for navigation.

### `encoding/gob`

```go
import "encoding/gob"

var buf bytes.Buffer
enc := gob.NewEncoder(&buf)
enc.Encode(value)

dec := gob.NewDecoder(&buf)
var v MyType
dec.Decode(&v)
```

Properties:

- Self-describing: types are transmitted once per stream.
- Handles cycles, interfaces (with registration), and unexported fields if you re-export.
- Backward compatible across versions if you add fields.
- Fast (faster than JSON), but only Go reads it.

Register interface types:

```go
gob.Register(&MyConcrete{})
```

### `encoding/csv`

```go
r := csv.NewReader(f)
r.Comma = ';'
r.Comment = '#'
r.LazyQuotes = true       // accept malformed quoting
r.FieldsPerRecord = -1    // variable-length rows
r.ReuseRecord = true      // reuse the returned slice across reads
for {
	row, err := r.Read()
	if err == io.EOF { break }
	if err != nil { return err }
	process(row)
}
```

`ReuseRecord = true` reuses the underlying slice — saves allocations but requires you to copy if you retain.

Writer:

```go
w := csv.NewWriter(f)
w.Comma = '\t'
w.Write([]string{"a", "b"})
w.WriteAll(rows)
w.Flush()
if err := w.Error(); err != nil { return err }
```

Always `Flush` before close. Check `Error()` after `WriteAll`/`Flush`.

## Standard Library Hooks

- `io.Reader`/`io.Writer` for all three.
- `encoding.TextMarshaler` / `TextUnmarshaler` for XML attributes.
- `bufio` for buffering before encoding/decoding.

## Real-World Patterns

### 1. Parse a large RSS feed with `xml.Decoder`

```go
func parseFeed(r io.Reader) ([]Item, error) {
	dec := xml.NewDecoder(r)
	var items []Item
	for {
		tok, err := dec.Token()
		if err == io.EOF { break }
		if err != nil { return nil, err }
		se, ok := tok.(xml.StartElement)
		if !ok || se.Name.Local != "item" { continue }
		var it Item
		if err := dec.DecodeElement(&it, &se); err != nil { return nil, err }
		items = append(items, it)
	}
	return items, nil
}
```

Use case: feed readers, podcast importers.

### 2. Gob-encoded cache snapshot

```go
func saveCache(path string, m map[string]Entry) error {
	f, err := os.Create(path); if err != nil { return err }
	defer f.Close()
	return gob.NewEncoder(f).Encode(m)
}
func loadCache(path string) (map[string]Entry, error) {
	f, err := os.Open(path); if err != nil { return nil, err }
	defer f.Close()
	var m map[string]Entry
	return m, gob.NewDecoder(f).Decode(&m)
}
```

Use case: warm caches across restarts; persisting in-memory state.

### 3. Streaming CSV transform

```go
func transformCSV(in io.Reader, out io.Writer) error {
	r := csv.NewReader(in); r.ReuseRecord = true
	w := csv.NewWriter(out); defer w.Flush()
	for {
		row, err := r.Read()
		if err == io.EOF { return w.Error() }
		if err != nil { return err }
		row[0] = strings.ToUpper(row[0])
		if err := w.Write(row); err != nil { return err }
	}
}
```

Use case: ETL pipelines, log normalization.

### 4. Tab-separated output for shell tools

```go
w := csv.NewWriter(os.Stdout)
w.Comma = '\t'
for _, r := range results {
	w.Write([]string{r.ID, r.Name, fmt.Sprint(r.Count)})
}
w.Flush()
```

Use case: CLIs producing output for `cut`, `awk` consumers.

### 5. XML with namespaces

```go
type Envelope struct {
	XMLName xml.Name `xml:"http://schemas.xmlsoap.org/soap/envelope/ Envelope"`
	Body    Body     `xml:"Body"`
}
```

Use case: SOAP / WS-* clients in enterprise integrations.

## Anti-Patterns & Gotchas

**Gob over the wire to non-Go clients.** Gob is Go-only.

**Gob without `gob.Register` for interface fields.** Decode fails with "unregistered type."

**Forgetting `csv.Writer.Flush`.** Data sits in buffer.

**Using XML where JSON would suffice.** Verbose, slow, edge cases. Pick XML only for interop.

**`csv.Reader` on CRLF files** with `LazyQuotes` disabled — usually fine but inspect; some CSV exporters embed CR inside fields.

**Reusing CSV records without copying.** `ReuseRecord = true` reuses the slice; downstream code holding the row sees later rows.

**XML decoder swallowing unknown elements silently.** Default behavior; if strict parsing is needed, use `dec.Strict = true` (limited) or hand-roll.

**Embedding `xml.Name` in every struct.** Unnecessary unless the element name needs explicit setting.

**Decoding gob from untrusted input.** Gob doesn't validate types; can panic. Use only with trusted senders.

## Performance Notes

- Gob is ~2× faster than JSON for marshaling, but does ~equal allocation work.
- `csv.Reader` with `ReuseRecord` is zero-alloc per row; without, allocates a `[]string` each read.
- `xml.Decoder.Token` is the streaming primitive; pull only the tokens you need.
- `xml.Unmarshal` reflects per field; for high-throughput XML, generate code with `chidley` or `xsdgen`.

## How Big Companies Use It

- **Hugo** parses many feed formats; XML decoder with streaming for large sitemaps.
- **HashiCorp** uses gob for some internal RPC; mostly moved to protobuf for cross-language services.
- **Cockroach** uses CSV for bulk import via `IMPORT INTO ... CSV`.
- **AWS SDK for Go (v1)** used XML extensively for legacy S3 / EC2 APIs; v2 uses smithy-generated XML.

## Source Code References

Pinned to `go1.26`.

- `encoding/xml`: [`src/encoding/xml/`](https://github.com/golang/go/tree/master/src/encoding/xml).
- `encoding/gob`: [`src/encoding/gob/`](https://github.com/golang/go/tree/master/src/encoding/gob).
- `encoding/csv`: [`src/encoding/csv/`](https://github.com/golang/go/tree/master/src/encoding/csv).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/encoding/xml, /gob, /csv.
- Go blog, "Gobs of data": https://go.dev/blog/gob.
- RFC 4180 (CSV): https://datatracker.ietf.org/doc/html/rfc4180.

## Exercises / Self-Check

1. Stream-parse an RSS feed via `xml.Decoder.Token`. Count items without loading all into memory.
2. Encode a `map[string]any` with `gob`; show that decoding into the same type recovers the value bit-perfectly.
3. Why does `gob.Register(&MyConcrete{})` matter when sending an interface value? Trigger the missing-registration error.
4. Read a tab-separated file with `csv.Reader` (set Comma). Handle quoted fields with embedded tabs.
5. Marshal a struct to XML with attributes and namespaces. Verify the output round-trips.

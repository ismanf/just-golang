# REST APIs — Design and Implementation in Go

## TL;DR

REST is less a strict standard than a *style*: resources addressed by URL, uniform verbs (GET/POST/PUT/PATCH/DELETE), stateless requests, hypermedia (often skipped — most "REST" APIs are really RPC-over-HTTP). Go's stdlib gives you everything you need for production REST without a framework — `net/http` for the server, `encoding/json` for serialisation, `1.22+`-style `ServeMux` for routing. Five disciplines distinguish a survivable REST API from one that becomes legacy code in 18 months: **resource modelling** (nouns not verbs; collections vs singletons; nesting depth ≤ 2), **versioning** (in the URL, in a header, or by media type — pick one, document it), **pagination** (cursor-based for infinite scroll, page-based for UIs that show "page 3 of 47"), **error responses** (RFC 9457 `application/problem+json`), and **OpenAPI** as the schema source of truth (drives clients, validation, mocks, docs). The single biggest gotcha: **return types that change unbidirectionally**. Once a field is in your response, removing it is a breaking change. Add fields freely; remove them only via a versioned endpoint.

## Mental Model

```
   Resource:  /users/{id}                /users/{id}/orders/{oid}
              ─────────                  ──────────────────────────
              singleton                  scoped sub-collection

   Verb       Effect
   ────       ──────
   GET        read; safe; idempotent; cacheable
   POST       create or invoke; not idempotent
   PUT        replace; idempotent
   PATCH      partial update; usually not idempotent
   DELETE     remove; idempotent

   Status     Meaning
   ──────     ───────
   200        OK with body
   201        Created (Location: /users/42)
   202        Accepted, processing async
   204        No body
   301/308    Moved (perm)
   304        Not modified (caching)
   400        Bad request (malformed)
   401        Unauthorized (missing/invalid credentials)
   403        Forbidden (authenticated but not allowed)
   404        Not found
   409        Conflict (e.g., version mismatch)
   422        Unprocessable entity (validation)
   429        Too many requests
   500        Server bug
   503        Unavailable (transient)
```

Two rules of thumb worth tattooing:

1. **Be a noun, not a verb.** `/users/{id}/cancel` is a verb-resource — RPC dressed as REST. Either accept it (it's pragmatic) or model state (`PATCH /users/{id}` with `{"status":"cancelled"}`).
2. **Pick consistency over purity.** A semi-RESTy API used consistently beats a pure REST API with one-off exceptions everywhere.

## Resource Modelling

```
GET    /users                  list users (with pagination)
POST   /users                  create a user
GET    /users/{id}             read a single user
PUT    /users/{id}             replace a user (full)
PATCH  /users/{id}             update a user (partial)
DELETE /users/{id}             delete a user

GET    /users/{id}/orders      list orders for user
POST   /users/{id}/orders      create order for user
GET    /orders/{oid}           read order directly (if globally addressable)
```

Nesting depth: **prefer ≤ 2 levels**. `/orgs/{o}/teams/{t}/members/{m}/permissions` is a smell. Often the deeper resource can be addressed flat (`/permissions/{pid}`) with the parent IDs as filter params.

Singleton resources (no `id`):

```
GET    /me                     the authenticated user
PUT    /me/preferences         singleton sub-resource
```

## A Minimal Production Handler

```go
package api

import (
    "encoding/json"
    "errors"
    "net/http"

    "log/slog"
)

type UserService interface {
    Get(ctx context.Context, id string) (User, error)
    Create(ctx context.Context, u CreateUserReq) (User, error)
}

type Server struct {
    users UserService
    log   *slog.Logger
}

func (s *Server) Routes() http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", s.getUser)
    mux.HandleFunc("POST /users",      s.createUser)
    return mux
}

func (s *Server) getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    u, err := s.users.Get(r.Context(), id)
    if errors.Is(err, ErrNotFound) {
        writeProblem(w, 404, "not_found", "user not found", r.URL.Path)
        return
    }
    if err != nil {
        s.log.ErrorContext(r.Context(), "get user", "err", err)
        writeProblem(w, 500, "internal", "internal error", r.URL.Path)
        return
    }
    writeJSON(w, 200, u)
}

func (s *Server) createUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserReq
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(&req); err != nil {
        writeProblem(w, 400, "bad_json", err.Error(), r.URL.Path)
        return
    }
    if err := req.Validate(); err != nil {
        writeProblem(w, 422, "validation", err.Error(), r.URL.Path)
        return
    }
    u, err := s.users.Create(r.Context(), req)
    if err != nil {
        s.log.ErrorContext(r.Context(), "create user", "err", err)
        writeProblem(w, 500, "internal", "internal error", r.URL.Path)
        return
    }
    w.Header().Set("Location", "/users/"+u.ID)
    writeJSON(w, 201, u)
}
```

What this gives you:

- **No framework dependency** — pure stdlib.
- **Decoder strict mode** (`DisallowUnknownFields`) — fail loud on typos.
- **Body cap** — no 100 MB POSTs.
- **RFC-9457 problem responses** (below).
- **`Location` header** on 201 — clients can follow.

## JSON Helpers

```go
func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(v); err != nil {
        // Already wrote headers; just log
        slog.Default().Error("encode failed", "err", err)
    }
}
```

For Go 1.25+, see `24-frontier/05-encoding-json-v2.md` — the v2 API is the future.

## Error Responses — RFC 9457 (`application/problem+json`)

```go
type Problem struct {
    Type     string `json:"type"`     // URI reference identifying the problem
    Title    string `json:"title"`    // short human-readable
    Status   int    `json:"status"`
    Detail   string `json:"detail,omitempty"`
    Instance string `json:"instance,omitempty"`
}

func writeProblem(w http.ResponseWriter, status int, code, detail, instance string) {
    w.Header().Set("Content-Type", "application/problem+json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(Problem{
        Type:     "https://errors.example.com/" + code,
        Title:    http.StatusText(status),
        Status:   status,
        Detail:   detail,
        Instance: instance,
    })
}
```

RFC 9457 (formerly RFC 7807) is the modern standard for HTTP error bodies. Adopt it once, document the `type` URI vocabulary, and clients can generate typed exceptions automatically.

For validation errors, extend with field-level info:

```json
{
  "type": "https://errors.example.com/validation",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "request failed validation",
  "errors": [
    {"field": "email", "code": "format", "msg": "must be a valid email"},
    {"field": "age",   "code": "min",    "msg": "must be >= 18"}
  ]
}
```

## Content Negotiation

```go
accept := r.Header.Get("Accept")
switch {
case strings.Contains(accept, "application/json"):
    writeJSON(w, 200, v)
case strings.Contains(accept, "application/xml"):
    writeXML(w, 200, v)
default:
    writeJSON(w, 200, v)  // sensible default
}
```

Almost no modern API supports multiple formats. JSON is the default. If you must, also support `application/vnd.api+json` (JSON:API) or `application/hal+json` (HAL).

## Versioning

Three approaches; pick one:

### URL prefix (most common)

```
/v1/users/{id}
/v2/users/{id}
```

Pros: discoverable; easy to route; easy to log/analyze. Cons: every URL is "the new one + the version" — ugly with deep nesting.

### Custom header

```
GET /users/{id}
API-Version: 2024-11-01
```

Pros: URL stable. Cons: invisible in logs; harder to debug; harder to cache. Stripe-style — date-based versions, server pins on first call.

### Media type

```
GET /users/{id}
Accept: application/vnd.example.user.v2+json
```

Pros: HATEOAS-pure. Cons: poor tooling support; clients hate it.

Production answer: **URL prefix, with date-based versions if you have many** (`/2024-11/users` Stripe-style works for very large APIs).

## Pagination

### Page-based (UI with "page 3 of 47")

```
GET /users?page=3&page_size=50

→ {
  "data": [...],
  "page": 3,
  "page_size": 50,
  "total": 2317
}
```

Pros: jumpable UI; familiar. Cons: drifts under writes (rows shift across pages); `OFFSET` is O(n) in most DBs.

### Cursor-based (infinite scroll, large datasets)

```
GET /users?cursor=eyJpZCI6ICIxMjMifQ&page_size=50

→ {
  "data": [...],
  "next_cursor": "eyJpZCI6ICIxNzMifQ"
}
```

Pros: stable under writes; O(1) per page; works at any scale. Cons: no jumping; needs an opaque cursor (encode last seen ID + sort tiebreaker).

```go
type Cursor struct {
    ID        string    `json:"id"`
    CreatedAt time.Time `json:"created_at"`
}

func encodeCursor(c Cursor) string {
    b, _ := json.Marshal(c)
    return base64.RawURLEncoding.EncodeToString(b)
}

func decodeCursor(s string) (Cursor, error) {
    b, err := base64.RawURLEncoding.DecodeString(s)
    if err != nil { return Cursor{}, err }
    var c Cursor
    return c, json.Unmarshal(b, &c)
}
```

Always **opaque** to clients (base64 of internal JSON). They must not parse it.

### Both, for huge APIs

GitHub-style: `Link` header with `rel="next"`, `rel="prev"`, `rel="first"`, `rel="last"` for page-based, plus a separate cursor for time-stable.

## Filtering, Sorting, Field Selection

```
GET /users?role=admin&status=active                # filter
GET /users?sort=-created_at,email                  # sort (- for desc)
GET /users?fields=id,name,email                    # sparse fieldsets
```

Document the query-parameter grammar. Reject unknown filters with 400. Build a small parser:

```go
type Filter struct {
    Role   string
    Status string
}

func parseFilter(q url.Values) (Filter, error) {
    f := Filter{Role: q.Get("role"), Status: q.Get("status")}
    if f.Role != "" && !validRole(f.Role) {
        return f, fmt.Errorf("invalid role: %q", f.Role)
    }
    return f, nil
}
```

Avoid generic filter DSLs (`?filter=age>30 AND name~^J`) — they evolve into mini-SQL with all SQL's footguns. If you need true expressiveness, use GraphQL.

## Concurrency Control — ETag / If-Match

```go
// GET response includes:
w.Header().Set("ETag", `"`+u.Version+`"`)

// PUT/PATCH request must echo:
ifMatch := strings.Trim(r.Header.Get("If-Match"), `"`)
if ifMatch != u.Version {
    writeProblem(w, 409, "version_conflict", "resource modified", r.URL.Path)
    return
}
```

Prevents lost updates from concurrent edits. Pair with optimistic locking in the DB.

For GETs, ETag + `If-None-Match` enables 304 caching:

```go
if r.Header.Get("If-None-Match") == `"`+u.Version+`"` {
    w.WriteHeader(304)
    return
}
```

## Bulk Operations

REST is awkward at bulk. Three patterns:

### Synchronous batch

```
POST /users/batch
[
  {"name": "alice", "email": "a@b"},
  {"name": "bob",   "email": "b@b"}
]

→ 200 OK
[
  {"status": "created", "id": "1"},
  {"status": "error", "code": "duplicate_email"}
]
```

Each sub-result has its own status. Top-level is 200 unless the entire batch failed.

### Async (long-running operation)

```
POST /imports
{"source": "s3://..."}

→ 202 Accepted
Location: /operations/op_abc123

GET /operations/op_abc123
{"state": "running", "progress": 0.5}
```

Pattern from Google's API Improvement Proposals (AIP-151). For batches that may take minutes.

### Streaming

NDJSON or `application/json-seq` (RFC 7464) — covered briefly in `05-graphql.md` and `06-websockets.md` for alternatives.

## Idempotency

Make `POST` retry-safe by accepting an `Idempotency-Key`:

```go
key := r.Header.Get("Idempotency-Key")
if key != "" {
    if cached := s.idempotency.Get(key); cached != nil {
        w.WriteHeader(cached.Status); w.Write(cached.Body)
        return
    }
}
// ... process ...
s.idempotency.Put(key, status, body)
```

TTL the cache (24h is common). Bind key + endpoint together so the same key on a different URL is a different operation.

## OpenAPI

Treat OpenAPI 3.1 as the source of truth:

```yaml
openapi: 3.1.0
paths:
  /users/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
```

Tools:

- **`oapi-codegen`** (https://github.com/oapi-codegen/oapi-codegen) — generate Go types and server stubs from OpenAPI.
- **`kin-openapi`** — runtime validation against the spec.
- **`swaggo/swag`** — generate spec from annotations (the reverse direction; less reliable).
- **Stoplight Studio**, **Redocly** — editing.
- **Spectral** — linting (`@stoplight/spectral`).

Generation flow:

```bash
oapi-codegen -package api -generate types,std-http openapi.yaml > api.gen.go
```

Now route handlers implement a generated interface; type-safe at compile time.

## CORS

```go
func cors(next http.Handler) http.Handler {
    allowed := map[string]bool{"https://app.example.com": true}
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")
        if allowed[origin] {
            w.Header().Set("Access-Control-Allow-Origin", origin)
            w.Header().Set("Vary", "Origin")
            w.Header().Set("Access-Control-Allow-Credentials", "true")
            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE")
            w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, Idempotency-Key")
        }
        if r.Method == http.MethodOptions {
            w.WriteHeader(204)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

Allow-list explicitly. `*` for `Access-Control-Allow-Origin` *plus* `Allow-Credentials: true` is rejected by browsers — pick one.

## Anti-Patterns & Gotchas

**Verb URLs.** `/getUser`, `/cancelOrder` — that's RPC; admit it or refactor.

**500 for client mistakes.** Validation errors are 422 (or 400). 401 is "I don't know you," 403 is "I know you and no." Don't mix them.

**Returning different shapes for success vs error from the same endpoint.** Clients can't deserialize without checking status first. Use Problem+JSON for errors so success vs error are always distinguishable by Content-Type.

**Stuffing IDs into the response body but not the URL.** REST: every created resource gets a URL. Echo it in `Location`.

**Mutating GETs.** Side effects on a read break caching and retries.

**Soft-deleting via a custom field but still returning soft-deleted records in `GET /users`.** Either hide them or document.

**Returning timestamps in local time / no time zone.** Always UTC, RFC 3339.

**Snake_case vs camelCase inconsistency.** Pick one, lint for it. JavaScript prefers camelCase; Go's json tags can do either.

**Returning unbounded list endpoints.** Always paginate.

**Hiding pagination metadata in headers AND duplicating in body.** Pick one. If headers, document `X-Total-Count`; if body, document the wrapper.

**Designing breaking changes you didn't realise.** Removing a field, renaming, changing nullability, narrowing a type's range — all breaking. Use versioning.

**No `Idempotency-Key` for write retries.** First retry storm hits and you've billed twice.

**Custom auth schemes.** Bearer tokens (OAuth 2 / OIDC) for users, mTLS for service-to-service, signed requests for server-to-server. Don't invent.

**Returning HTTP 200 with an error in the body.** Use the status. 200 = success. Frameworks/clients act on the status code.

**Including stack traces or SQL fragments in error responses.** Information disclosure.

**Treating PATCH as PUT-with-partial-fields.** Use `application/json-patch+json` (RFC 6902) for true patch ops; otherwise be clear that PATCH means "merge the provided fields with the existing resource."

## Performance Notes

- **JSON encode/decode**: ~1-5 µs per small struct; 100-500 µs for large.
- **DB query** typically dominates request latency — REST overhead is rounding error.
- **Compression** (`gzip`) for JSON responses >1 KB — 5x size reduction, ~50 µs encode cost.
- **HTTP keep-alive** saves the 1-3 ms TCP + TLS per request — make sure clients use it.
- **ETag/304 caching** for read-heavy endpoints — turn 10ms into 1ms.

## How Big Companies Use It

- **Stripe** uses URL-versioned APIs (`/v1`) with date-based sub-versions per API key. Idempotency keys are first-class. Problem-JSON-style errors with rich `error.code` taxonomy.
- **GitHub** uses URL versioning + `Accept: application/vnd.github+json` for legacy versions. Cursor-based pagination via `Link` header.
- **Atlassian** uses URL versioning + extensive OpenAPI publication. JSON:API in some products.
- **Google Cloud** APIs follow Google AIPs (https://google.aip.dev) — extremely consistent across hundreds of services.
- **Twilio** uses URL versioning with date-stamped releases.
- **Shopify** uses URL versioning and a documented deprecation policy (12 months).
- **HashiCorp** uses URL versioning for HCP and Vault.
- **Cloudflare** API v4 uses URL versioning, RFC-style errors, and OpenAPI.

## Source Code References

- `net/http`: https://github.com/golang/go/tree/master/src/net/http.
- `encoding/json`: https://github.com/golang/go/tree/master/src/encoding/json.
- `oapi-codegen`: https://github.com/oapi-codegen/oapi-codegen.
- `kin-openapi`: https://github.com/getkin/kin-openapi.
- Google AIPs: https://github.com/aip-dev/google.aip.dev.
- RFC 9457 (Problem Details): https://www.rfc-editor.org/rfc/rfc9457.
- RFC 7232 (Conditional Requests): https://www.rfc-editor.org/rfc/rfc7232.
- RFC 6902 (JSON Patch): https://www.rfc-editor.org/rfc/rfc6902.

## Further Reading

- Roy Fielding's dissertation (the original REST definition): https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm.
- "REST API Design Rulebook" (Mark Massé, O'Reilly).
- Google API Design Guide: https://cloud.google.com/apis/design.
- Microsoft Azure REST API Guidelines: https://github.com/microsoft/api-guidelines.
- "Hypermedia APIs with HTML5" (Mike Amundsen).
- "Web API Design: Crafting Interfaces that Developers Love" (Brian Mulloy, Apigee).
- Stripe's API changelog as a public artefact: https://stripe.com/docs/upgrades.
- "Standards.REST": https://standards.rest/.

## Exercises / Self-Check

1. Design a resource model for a blog: posts, comments, tags. Decide nesting depth and verbs for each. Justify.
2. Implement RFC 9457 problem responses for 404, 422, 500 in a Go handler. Verify Content-Type and shape with `curl`.
3. Build a cursor-based paginated `/users` endpoint. Encode cursors opaquely; ensure stability across concurrent writes.
4. Add `ETag` + `If-None-Match` caching to a `GET /users/{id}` endpoint. Verify 304 response.
5. Add `Idempotency-Key` support to a `POST /payments` endpoint. Retry the same key; confirm exactly one charge.
6. Generate server stubs from an OpenAPI 3.1 spec via `oapi-codegen`. Implement the interface; run `oapitest` (or kin-openapi runtime validation) against your live server.
7. Implement URL-prefix versioning (`/v1`, `/v2`) with shared business logic but different request/response shapes per version.
8. Lint your OpenAPI spec with Spectral. Fix all errors; document why each rule matters.

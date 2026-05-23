# GraphQL in Go — `gqlgen`

## TL;DR

GraphQL is a query language with a typed schema where **clients specify exactly the fields they want**, the server resolves them, and the response shape matches the query shape. Three Go libraries dominate: **`gqlgen`** (https://gqlgen.com — schema-first, code-generated, idiomatic Go; the de-facto choice), **`graphql-go/graphql`** (older, runtime-typed, much less common now), and **`graph-gophers/graphql-go`** (lighter, struct-tag-based). gqlgen's killer feature is **schema-first + code generation**: write `.graphql` SDL, generate Go types and resolver interfaces, implement the resolvers. The runtime is small; everything's compile-time-checked. Five operational disciplines that distinguish working GraphQL from breaking GraphQL: **dataloaders** (the N+1 problem is GraphQL's defining performance pitfall; batch your DB calls), **query complexity analysis** (so an attacker doesn't ask for `users { posts { comments { author { posts { ... } } } } }` 20 levels deep), **persisted queries** (only allow pre-registered queries in production; eliminates query injection and slashes payload size), **subscriptions over WebSocket or SSE** (real-time updates; covered partly in `06-websockets.md`), and **federation** if you have multiple services (Apollo Federation; `gqlgen` supports it). The single biggest gotcha: **GraphQL hides query cost from the URL.** Every query is a POST to `/graphql`; caching, rate limiting, monitoring become per-resolver problems, not per-route problems.

## Mental Model

```
   Schema (SDL)
   ────────────
   type User {
     id: ID!
     name: String!
     posts: [Post!]!
   }

         │
         │ gqlgen generates: Go types, ResolverRoot interface
         ▼

   Resolvers (you implement)
   ─────────────────────────
   func (r *queryResolver) User(ctx, id) (*User, error)
   func (r *userResolver) Posts(ctx, user) ([]*Post, error)

         │
         │ gqlgen runtime walks the query, calls resolvers
         ▼

   Response (JSON, mirroring the query shape)
   ──────────────────────────────────────────
   { "data": { "user": { "name": "...", "posts": [...] } } }
```

Two invariants:

1. **The shape of the response = the shape of the query.** No over-fetching, no under-fetching.
2. **Each field has its own resolver.** Resolvers can be cheap (return a struct field) or expensive (DB call). GraphQL doesn't know which — that's why dataloaders exist.

## Setup with gqlgen

```bash
mkdir myapi && cd myapi
go mod init example.com/myapi

# Bootstrap
go run github.com/99designs/gqlgen init
```

Generates:

- `gqlgen.yml` — config.
- `graph/schema.graphqls` — schema (you edit).
- `graph/schema.resolvers.go` — resolver stubs (you implement).
- `server.go` — HTTP server entry.

Workflow:

```bash
# Edit schema
$EDITOR graph/schema.graphqls

# Regenerate types and resolver stubs
go run github.com/99designs/gqlgen generate

# Implement new resolver stubs
$EDITOR graph/schema.resolvers.go
```

## A Real Schema

```graphql
# graph/schema.graphqls

type Query {
  user(id: ID!): User
  users(first: Int = 20, after: String): UserConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type Subscription {
  userUpdated(id: ID!): User!
}

type User {
  id: ID!
  email: String!
  name: String!
  posts(first: Int = 10): [Post!]!
  createdAt: Time!
}

type Post {
  id: ID!
  title: String!
  author: User!
  comments: [Comment!]!
}

type Comment {
  id: ID!
  body: String!
  author: User!
}

# Relay-style connection (cursor pagination)
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}
type UserEdge {
  cursor: String!
  node: User!
}
type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

input CreateUserInput {
  email: String!
  name: String!
}

input UpdateUserInput {
  name: String
}

type CreateUserPayload {
  user: User!
}

scalar Time
```

Key conventions:

- **Connections + edges** for paginated lists (Relay spec).
- **Input objects** for mutations (so adding optional fields is non-breaking).
- **Payload wrapper types** for mutations (so you can add side-effect data later — e.g., `clientMutationId`, `errors[]`).
- **Custom scalars** like `Time` you register in `gqlgen.yml`.

## Implementing Resolvers

```go
// graph/schema.resolvers.go (generated stubs you implement)

func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    return r.UserRepo.GetByID(ctx, id)
}

func (r *userResolver) Posts(ctx context.Context, obj *model.User, first *int) ([]*model.Post, error) {
    // Per-field resolver; called for every User in the response.
    // N+1 risk!
    n := 10
    if first != nil { n = *first }
    return r.PostRepo.ListByAuthor(ctx, obj.ID, n)
}
```

If `Query.users` returns 100 users and the query asks for `posts`, the `Posts` resolver runs 100 times — 100 separate DB queries. This is the N+1 problem.

## Dataloaders — Solving N+1

```go
import "github.com/graph-gophers/dataloader/v7"

type Loaders struct {
    PostsByUser *dataloader.Loader[string, []*model.Post]
}

func NewLoaders(postRepo *PostRepo) *Loaders {
    return &Loaders{
        PostsByUser: dataloader.NewBatchedLoader(
            func(ctx context.Context, userIDs []string) []*dataloader.Result[[]*model.Post] {
                posts, _ := postRepo.ListByAuthors(ctx, userIDs)
                byUser := map[string][]*model.Post{}
                for _, p := range posts { byUser[p.AuthorID] = append(byUser[p.AuthorID], p) }
                out := make([]*dataloader.Result[[]*model.Post], len(userIDs))
                for i, id := range userIDs {
                    out[i] = &dataloader.Result[[]*model.Post]{Data: byUser[id]}
                }
                return out
            },
            dataloader.WithWait[string, []*model.Post](2 * time.Millisecond),
        ),
    }
}

// Middleware to attach loaders per request
func LoadersMiddleware(repo *PostRepo) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := context.WithValue(r.Context(), loadersKey, NewLoaders(repo))
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Resolver uses the loader
func (r *userResolver) Posts(ctx context.Context, obj *model.User, first *int) ([]*model.Post, error) {
    loaders := ctx.Value(loadersKey).(*Loaders)
    return loaders.PostsByUser.Load(ctx, obj.ID)()
}
```

The dataloader **batches** loads within a small time window (2ms) and **dedups** identical keys. 100 individual `Posts` resolver calls become one `ListByAuthors([100 IDs])` DB query.

Lifecycle: **one loader instance per request**, never global (otherwise cross-request leakage).

## Server Wiring

```go
package main

import (
    "log"
    "net/http"

    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/handler/extension"
    "github.com/99designs/gqlgen/graphql/handler/transport"
    "github.com/99designs/gqlgen/graphql/playground"

    "example.com/myapi/graph"
)

func main() {
    resolver := &graph.Resolver{
        UserRepo: NewUserRepo(),
        PostRepo: NewPostRepo(),
    }

    srv := handler.New(graph.NewExecutableSchema(graph.Config{Resolvers: resolver}))
    srv.AddTransport(transport.POST{})
    srv.AddTransport(transport.GET{})
    srv.AddTransport(transport.MultipartForm{})
    srv.AddTransport(transport.Websocket{KeepAlivePingInterval: 10*time.Second})
    srv.Use(extension.Introspection{})
    srv.Use(extension.AutomaticPersistedQuery{Cache: lru.New(1000)})
    srv.Use(extension.FixedComplexityLimit(300))

    mux := http.NewServeMux()
    mux.Handle("/graphql", LoadersMiddleware(resolver.PostRepo)(srv))
    mux.Handle("/playground", playground.Handler("GraphQL", "/graphql"))

    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

## Complexity Limits

```go
srv.Use(extension.FixedComplexityLimit(300))
```

Each field has a default complexity of 1. Override per field with `gqlgen.yml` complexity directives:

```yaml
complexity:
  User.posts: 5
  Query.users: { ratio: 2, default: 1 }   # cost = first * 2
```

Reject queries exceeding the limit *before* execution. Critical for any public-facing API — without it, an attacker can `users(first: 1000) { posts(first: 1000) { comments(first: 1000) { ... } } }` and OOM you.

For more sophisticated limits, gqlgen supports custom complexity calculators.

## Persisted Queries

```go
srv.Use(extension.AutomaticPersistedQuery{Cache: lru.New(1000)})
```

Apollo's APQ protocol: client sends a query hash; server looks up the registered query. First call (cache miss) sends the full query + hash; subsequent calls send only the hash.

Benefits:
- **Bandwidth**: clients send 32-byte hash, not 2 KB query.
- **Cacheability**: hash → response, easier to cache.
- **Allow-listing**: in strict mode, **only registered queries** can execute. Eliminates query injection.

In production, lock to allow-list mode:

```go
srv.Use(extension.PersistedQuery{Cache: ...})
// Reject unknown queries with a 400
```

Build pipeline registers queries at deploy time.

## Subscriptions

```graphql
type Subscription {
  userUpdated(id: ID!): User!
}
```

```go
func (r *subscriptionResolver) UserUpdated(ctx context.Context, id string) (<-chan *model.User, error) {
    ch := make(chan *model.User, 1)
    sub := r.PubSub.Subscribe("user:" + id)
    go func() {
        defer close(ch)
        for {
            select {
            case <-ctx.Done():
                sub.Cancel()
                return
            case u := <-sub.C:
                ch <- u
            }
        }
    }()
    return ch, nil
}
```

Transport: WebSocket (graphql-ws / graphql-transport-ws) or SSE. gqlgen's `transport.Websocket` handles the protocol negotiation.

For multi-instance subscriptions (sharded backend), use Redis pub/sub or NATS to fan messages across all server replicas.

## Federation (Multi-Service)

If you have multiple services that each own a slice of the schema:

```graphql
# user-service
extend type Query {
  user(id: ID!): User
}
type User @key(fields: "id") {
  id: ID!
  name: String!
}

# post-service
extend type Query {
  post(id: ID!): Post
}
type Post {
  id: ID!
  title: String!
  author: User!    # User comes from user-service
}
extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!  # added to User by post-service
}
```

An **Apollo Router** (or `mercury`, `nautilus`) sits in front, parses queries, splits them across services, and assembles results. gqlgen has `federation` plugin support.

Federation adds complexity; only adopt with multiple teams owning distinct domains.

## Error Handling

```go
return nil, fmt.Errorf("user %q not found", id)
// → { "data": null, "errors": [{"message":"user \"42\" not found", "path":["user"]}] }
```

For typed errors with custom fields:

```go
func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    u, err := r.UserRepo.GetByID(ctx, id)
    if err != nil {
        return nil, &gqlerror.Error{
            Message: "user not found",
            Path:    graphql.GetPath(ctx),
            Extensions: map[string]interface{}{
                "code": "NOT_FOUND",
                "id":   id,
            },
        }
    }
    return u, nil
}
```

`extensions.code` is the canonical machine-readable error code field; clients dispatch on it. Common codes: `UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `BAD_USER_INPUT`, `INTERNAL_SERVER_ERROR`.

Partial errors: GraphQL can return both `data` and `errors`. If `user.posts` fails but `user.email` succeeds, the response includes the email and an error pointed at `path: ["user", "posts"]`. Use this carefully — clients must handle partial data.

## Caching

Three layers:

1. **HTTP-level caching is mostly useless** — every query is a POST. Use GET with persisted queries to enable CDN caching for read-only queries.
2. **Per-resolver caching** (in-process LRU, Redis) — wrap resolvers in a cache aside pattern.
3. **Apollo client-side cache** — normalised, per-entity cache in the browser. The most impactful caching layer.

## Anti-Patterns & Gotchas

**No dataloaders.** N+1 queries; every page load fans out hundreds of DB queries.

**No complexity limit.** Public API; attacker queries deeply nested fields and OOMs you.

**No persisted queries on public APIs.** Clients can craft arbitrary queries; you can't precisely cap cost.

**Introspection in production.** Useful for tools; also lets attackers map your schema. Disable or allow-list.

**Subscriptions backed by polling.** Defeats the point. Use real pub/sub.

**Schema design as "expose the database."** GraphQL types should be **product-shaped**, not table-shaped. Hide the join tables.

**Mutations that take more than one input.** GraphQL convention: one `input` object. Easier to evolve.

**Returning Node IDs from your DB directly without Relay-style globally-unique IDs.** Apollo and Relay clients assume `User:42` style; otherwise cache normalisation breaks.

**`null` everywhere.** GraphQL non-null (`!`) is your friend. If a field is always present, mark it; if it's optional, don't lie about it.

**Coupling resolver code to HTTP request.** Resolvers should take `ctx` and inputs only; the HTTP plumbing is in the transport layer.

**Per-query timeouts left at infinity.** A pathological query without a deadline never returns. Set `ctx` deadline in middleware.

**File uploads via base64 in a query.** Use the multipart spec (`transport.MultipartForm{}`) or a separate REST endpoint with a signed URL.

**Generated code committed but not regenerated on schema change.** PR review fails to catch the drift. Make `gqlgen generate` part of CI.

**Mixing public and admin schemas.** Two services or two endpoints; otherwise auth becomes per-field nightmare.

**Subscriptions without backpressure.** Slow client consumes events slowly; in-memory queue grows. Drop events with explicit policy.

## Performance Notes

- **gqlgen runtime per field**: ~1-5 µs overhead.
- **Query parsing**: ~10-50 µs for typical queries; cache parsed queries (persisted queries do this implicitly).
- **JSON encoding of response**: similar to REST.
- **Dataloader batch wait**: 2-5 ms is typical; trade off latency vs batch size.
- **Subscriptions over WebSocket**: ~100 µs per event published; scale to ~10k concurrent subscribers per instance with care.

## How Big Companies Use It

- **GitHub** uses GraphQL for their public API v4 (alongside REST v3); served by Ruby on Rails originally, with Go for internal services.
- **Shopify** uses GraphQL extensively (both Storefront and Admin APIs); per-query complexity limits enforced.
- **Netflix** uses GraphQL with their own federation framework (Domain Graph Service).
- **Airbnb** uses GraphQL across many services with strong schema review processes.
- **PayPal** migrated to GraphQL via Apollo Federation.
- **Atlassian** Confluence/Jira Cloud uses GraphQL.
- **Twitch** uses GraphQL (Twirp on the side for gRPC-style RPC).
- **Khan Academy** famously migrated entirely to GraphQL on Go.

In the Go ecosystem specifically, gqlgen is the dominant choice; most internal-tools shops, fintechs, and SaaS startups using GraphQL pick it.

## Source Code References

- gqlgen: https://github.com/99designs/gqlgen.
- graphql-go/graphql: https://github.com/graphql-go/graphql.
- graph-gophers/graphql-go: https://github.com/graph-gophers/graphql-go.
- graph-gophers/dataloader: https://github.com/graph-gophers/dataloader.
- Apollo Federation spec: https://www.apollographql.com/docs/federation/.
- graphql-ws protocol: https://github.com/enisdenjo/graphql-ws.

## Further Reading

- GraphQL spec: https://spec.graphql.org/.
- "Production-Ready GraphQL" (Marc-André Giroux, book).
- Apollo's "Principled GraphQL" essays.
- "GraphQL at GitHub" (GitHub Engineering blog).
- "Demystifying gqlgen" (99designs blog series).
- "Securing GraphQL APIs" (Apollo blog).
- "Federation v2 best practices" (Apollo docs).
- "How Khan Academy uses GraphQL" (engineering posts).

## Exercises / Self-Check

1. Bootstrap a gqlgen project. Define a `User` + `Post` schema. Implement basic CRUD resolvers backed by an in-memory store.
2. Add a per-request dataloader for `Post` → `Author`. Verify that querying `users { posts { author { name } } }` triggers one batched author lookup, not N.
3. Enable `extension.FixedComplexityLimit(300)`. Send a deeply nested query that exceeds; confirm rejection.
4. Implement persisted queries via APQ. Register a query at deploy time; verify clients can call it with the hash only.
5. Add a `userUpdated` subscription. Wire it to a publish/subscribe channel. Verify multiple WebSocket subscribers each receive events.
6. Define typed errors with `extensions.code`. Differentiate `UNAUTHENTICATED`, `NOT_FOUND`, `FORBIDDEN`.
7. Compare query latency for a "deeply nested" query before and after introducing dataloaders. Measure DB query count via a query counter.
8. Build a small Apollo Federation setup: a user service and a post service. Connect via Apollo Router. Confirm cross-service queries resolve.

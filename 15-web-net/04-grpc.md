# gRPC — `grpc-go`, Interceptors, Streaming

## TL;DR

gRPC is Google's open-source RPC framework: **Protobuf as the IDL**, **HTTP/2 as the transport**, **code generation for clients and servers in every major language**. The Go implementation (`google.golang.org/grpc`) is the reference. Four call types you must know: **unary** (one request → one response, the default), **server-streaming** (one request → many responses), **client-streaming** (many requests → one response), **bidirectional streaming** (interleaved many-many). Five operational disciplines that separate working gRPC from production gRPC: **deadlines on every call** (no default; missing them = clients hang), **status codes via `codes.Code`** (not `error.New`), **interceptors for cross-cutting concerns** (auth, logging, metrics — the gRPC equivalent of HTTP middleware), **keepalive tuning** (otherwise idle connections die behind NATs and load balancers), and **service config + retries** (declarative retry policy via JSON config, not ad-hoc client logic). Go 1.26 doesn't change `grpc-go` itself, but the **Connect-Go** library (Buf Build) and **gRPC-web** open up browser interop, and gRPC's transition to HTTP/3 transport is now beta in `grpc-go` v1.65+. The single biggest gotcha: **gRPC connection load balancing is not done at the TCP level** (one HTTP/2 conn, all multiplexed) — your L4 load balancer will not balance gRPC. Use **client-side LB** (resolver + balancer) or an L7 proxy (Envoy, Linkerd).

## Mental Model

```
   .proto file
   ─────────
        │
        │  protoc-gen-go + protoc-gen-go-grpc
        ▼
   Generated:
     - Message types (Go structs with serialisation)
     - Server interface  (UserServiceServer)
     - Client struct     (UserServiceClient)
        │
        ▼
   ┌────────────┐                          ┌────────────┐
   │  Client     │── HTTP/2 stream ───────►│  Server     │
   │  - dial      │   per-call deadline    │  - register │
   │  - call      │   metadata (headers)   │  - serve    │
   │  - close     │   payload (protobuf)   │  - status   │
   └────────────┘                          └────────────┘
```

Two invariants:

1. **One HTTP/2 connection multiplexes many RPCs.** Concurrent calls share streams.
2. **gRPC errors are `status.Status`**, not Go `error`s. A handler returning `errors.New("nope")` becomes `code: Unknown, message: "nope"` — useless to clients.

## Setup

```bash
# Toolchain
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Module
go get google.golang.org/grpc google.golang.org/protobuf
```

`proto/user/v1/user.proto`:

```proto
syntax = "proto3";
package user.v1;
option go_package = "example.com/api/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);            // server stream
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersReport);  // client stream
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);        // bidirectional
}

message User {
  string id = 1;
  string email = 2;
  google.protobuf.Timestamp created_at = 3;
}

message GetUserRequest { string id = 1; }
message ListUsersRequest { int32 limit = 1; string cursor = 2; }
message CreateUserRequest { string email = 1; }
message CreateUsersReport { int32 created = 1; }
message ChatMessage { string text = 1; }
```

Compile:

```bash
buf generate    # using Buf — recommended
# OR raw protoc:
protoc --go_out=. --go-grpc_out=. proto/user/v1/user.proto
```

## Minimal Server

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"

    userv1 "example.com/api/user/v1"
)

type server struct {
    userv1.UnimplementedUserServiceServer
    db map[string]*userv1.User
}

func (s *server) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.User, error) {
    u, ok := s.db[req.GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
    }
    return u, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil { log.Fatal(err) }
    srv := grpc.NewServer()
    userv1.RegisterUserServiceServer(srv, &server{db: map[string]*userv1.User{}})
    log.Fatal(srv.Serve(lis))
}
```

The `UnimplementedUserServiceServer` embed is required — it provides `mustEmbedUnimplemented*` methods that future-proof against new proto methods. If you don't embed it, adding a method to the proto breaks compilation.

## Minimal Client

```go
conn, err := grpc.NewClient("localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil { log.Fatal(err) }
defer conn.Close()

client := userv1.NewUserServiceClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

u, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: "42"})
if err != nil {
    if status.Code(err) == codes.NotFound {
        // handle 404
    }
    return err
}
```

`grpc.NewClient` (formerly `grpc.Dial`) is the 1.64+ API. It doesn't actually establish the connection until the first RPC — lazy. Don't recreate per call.

## Status Codes

```go
// On the server, ALWAYS return a status, not a bare error
return nil, status.Errorf(codes.InvalidArgument, "email is required")
return nil, status.Error(codes.PermissionDenied, "not allowed")

// With error details (typed):
st := status.New(codes.InvalidArgument, "validation failed")
st, _ = st.WithDetails(&errdetails.BadRequest{
    FieldViolations: []*errdetails.BadRequest_FieldViolation{
        {Field: "email", Description: "invalid format"},
    },
})
return nil, st.Err()
```

Common codes:

| Code | Meaning |
|------|---------|
| `OK` | Success |
| `Canceled` | Client canceled |
| `Unknown` | Unknown (the default for bare errors — avoid) |
| `InvalidArgument` | 400 in HTTP terms |
| `DeadlineExceeded` | Server timed out the client's deadline |
| `NotFound` | 404 |
| `AlreadyExists` | 409 |
| `PermissionDenied` | 403 |
| `Unauthenticated` | 401 |
| `ResourceExhausted` | 429 / quota |
| `FailedPrecondition` | 412 |
| `Aborted` | concurrency abort (e.g., optimistic locking) |
| `OutOfRange` | iteration past end |
| `Unimplemented` | 501 |
| `Internal` | 500 |
| `Unavailable` | 503 — retryable |
| `DataLoss` | data was lost |

Reading code on the client:

```go
if s, ok := status.FromError(err); ok {
    switch s.Code() {
    case codes.NotFound:    // ...
    case codes.Unavailable: // retry
    }
}
```

## Deadlines

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

The deadline propagates **across hops**. If service A calls service B with a 2s deadline, B sees the deadline in its incoming `ctx`, can subtract its own processing time, and pass a shorter deadline to service C. Without deadlines, an outage cascades into infinite hangs.

Server-side enforcement:

```go
func (s *server) Expensive(ctx context.Context, req *Req) (*Resp, error) {
    if d, ok := ctx.Deadline(); ok && time.Until(d) < 100*time.Millisecond {
        return nil, status.Error(codes.DeadlineExceeded, "deadline too short")
    }
    // ...
}
```

## Interceptors (Middleware)

### Unary server interceptor

```go
func loggingUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    slog.InfoContext(ctx, "rpc",
        "method", info.FullMethod,
        "dur", time.Since(start),
        "code", status.Code(err),
    )
    return resp, err
}

srv := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryUnary,
        loggingUnary,
        authUnary,
        otelgrpc.UnaryServerInterceptor(),   // tracing
    ),
)
```

### Unary client interceptor

```go
func authClient(ctx context.Context, method string, req, reply any,
    cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
    return invoker(ctx, method, req, reply, cc, opts...)
}

conn, _ := grpc.NewClient(addr,
    grpc.WithChainUnaryInterceptor(authClient, otelgrpc.UnaryClientInterceptor()),
)
```

`Chain*Interceptor` runs them outside-in, like HTTP middleware: leftmost wraps everything; rightmost is innermost.

### Stream interceptors

```go
func loggingStream(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
    handler grpc.StreamHandler) error {
    start := time.Now()
    err := handler(srv, ss)
    slog.InfoContext(ss.Context(), "stream",
        "method", info.FullMethod,
        "dur", time.Since(start),
    )
    return err
}
```

Wrap the `ServerStream` if you need to inspect or modify per-message data.

## Streaming

### Server-streaming

```proto
rpc ListUsers(ListUsersRequest) returns (stream User);
```

Server:

```go
func (s *server) ListUsers(req *userv1.ListUsersRequest, stream userv1.UserService_ListUsersServer) error {
    for _, u := range s.findAll() {
        if err := stream.Send(u); err != nil { return err }
        if err := stream.Context().Err(); err != nil { return err }   // honour cancel
    }
    return nil
}
```

Client:

```go
stream, _ := client.ListUsers(ctx, &userv1.ListUsersRequest{Limit: 100})
for {
    u, err := stream.Recv()
    if err == io.EOF { break }
    if err != nil { return err }
    handle(u)
}
```

### Client-streaming

```proto
rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersReport);
```

Server:

```go
func (s *server) CreateUsers(stream userv1.UserService_CreateUsersServer) error {
    var n int32
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&userv1.CreateUsersReport{Created: n})
        }
        if err != nil { return err }
        s.create(req)
        n++
    }
}
```

### Bidirectional streaming

```proto
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

Server:

```go
func (s *server) Chat(stream userv1.UserService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF { return nil }
        if err != nil { return err }
        // Possibly send 0..N responses
        if err := stream.Send(&userv1.ChatMessage{Text: "ack: " + msg.Text}); err != nil {
            return err
        }
    }
}
```

Concurrency: `Send` and `Recv` can be called concurrently from different goroutines, but `Send` itself is not concurrent-safe — serialise it.

## Keepalive

Behind LBs and NATs, idle connections die silently. Tune:

```go
import "google.golang.org/grpc/keepalive"

// Server
grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        Time:    10 * time.Second,  // ping if idle
        Timeout: 5 * time.Second,   // ping ack must come within
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             5 * time.Second,
        PermitWithoutStream: true,
    }),
)

// Client
grpc.NewClient(addr,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second,
        Timeout:             5 * time.Second,
        PermitWithoutStream: true,
    }),
)
```

Without keepalive, an idle gRPC conn behind AWS NLB dies after ~350s. First call after that hangs until TCP timeout (~minutes).

## Load Balancing

gRPC over a single L4 LB is broken — one TCP conn, all multiplexed, no fan-out. Three solutions:

### Client-side LB via resolver

```go
conn, _ := grpc.NewClient("dns:///myservice.namespace.svc:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
)
```

DNS resolver returns multiple A records; client opens one conn per backend; round-robins RPCs.

### Headless services in Kubernetes

```yaml
apiVersion: v1
kind: Service
metadata: { name: myservice }
spec:
  clusterIP: None      # headless — DNS returns pod IPs
  selector: { app: myservice }
  ports:
    - port: 50051
```

Combined with client-side LB, this is the canonical k8s gRPC pattern.

### Service mesh (Envoy, Linkerd)

Sidecar proxy does the L7 LB. Client talks to localhost; mesh distributes. Most robust for production.

## Retries & Service Config

gRPC supports declarative retry policy via JSON:

```go
const cfg = `{
  "methodConfig": [{
    "name": [{"service": "user.v1.UserService"}],
    "retryPolicy": {
      "MaxAttempts": 4,
      "InitialBackoff": "0.1s",
      "MaxBackoff": "1s",
      "BackoffMultiplier": 2.0,
      "RetryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    },
    "timeout": "5s"
  }]
}`

conn, _ := grpc.NewClient(addr, grpc.WithDefaultServiceConfig(cfg), ...)
```

Critical: **retries only happen for idempotent RPCs** that you mark as such. By default, gRPC retries on `UNAVAILABLE` because the server is presumed not to have received the call. For non-idempotent RPCs, an idempotency key in the metadata is still your friend.

## Reflection (Useful for `grpcurl`)

```go
import "google.golang.org/grpc/reflection"

srv := grpc.NewServer()
userv1.RegisterUserServiceServer(srv, &server{})
reflection.Register(srv)
```

Now `grpcurl localhost:50051 list` shows services. Enable in dev/staging; consider disabling in prod for security.

## gRPC-Web and Connect

Browsers can't speak HTTP/2 trailers and binary protobuf directly. Two bridges:

- **gRPC-Web** — protocol wrapping gRPC for browsers. Server-side, run an Envoy filter or `improbable-eng/grpc-web` proxy.
- **Connect** (https://connectrpc.com) — Buf's protocol that works for both browsers and gRPC clients. The Go server speaks both protocols on the same handler. Increasingly the default for new public-facing APIs.

```go
// connect-go server
import "connectrpc.com/connect"

mux := http.NewServeMux()
path, handler := userv1connect.NewUserServiceHandler(&server{})
mux.Handle(path, handler)
http.ListenAndServe(":8080", h2c.NewHandler(mux, &http2.Server{}))
```

Same proto, same generated code (different code generator), browser-friendly, supports gRPC clients too.

## TLS / mTLS

```go
// Server
creds, _ := credentials.NewServerTLSFromFile("server.crt", "server.key")
srv := grpc.NewServer(grpc.Creds(creds))

// Client
creds, _ := credentials.NewClientTLSFromFile("ca.crt", "")
conn, _ := grpc.NewClient(addr, grpc.WithTransportCredentials(creds))

// mTLS — both sides authenticate
serverTLS := &tls.Config{
    Certificates: []tls.Certificate{serverCert},
    ClientCAs:    clientCAs,
    ClientAuth:   tls.RequireAndVerifyClientCert,
}
srv := grpc.NewServer(grpc.Creds(credentials.NewTLS(serverTLS)))
```

See `08-tls.md` for full TLS configuration.

## Anti-Patterns & Gotchas

**`grpc.Dial`/`grpc.NewClient` per request.** Defeats connection multiplexing. Build once; reuse.

**`return nil, errors.New("nope")`.** Becomes `code: Unknown`. Always `status.Error(codes.X, ...)`.

**Calling unary RPCs without a deadline.** Outages turn into infinite hangs.

**L4 load balancing.** Single connection per client; one backend gets all the load.

**Keepalive defaults left untuned.** Idle connections die behind NATs/LBs; first call after idle stalls.

**Mixing `Send` calls from multiple goroutines on the same stream.** Not safe; corrupted frames.

**Forgetting `defer conn.Close()`** in clients. Connection (and its background goroutine) leaks.

**Ignoring `stream.Context().Err()` in long-running streams.** You'll continue producing work after client disconnect.

**Returning huge responses without chunking.** gRPC's default max message size is 4 MB. Either raise (`grpc.MaxRecvMsgSize`) or paginate / stream.

**`UnimplementedXxx` embed skipped.** Adding a new RPC method to the proto breaks compilation of all existing servers.

**Treating `Unknown` as a normal error.** It's almost always a misuse on the server side.

**No retries on `UNAVAILABLE`.** Transient connection drops become user-visible errors. Use service config.

**Retries on non-idempotent calls.** Double-writes.

**Letting protobuf's `optional` semantics surprise you.** In proto3, default values are indistinguishable from "not set" unless you use `optional` (Go 1.21+ field presence).

**Reflection enabled in production without auth.** Anyone can introspect your API.

**Logging full request/response bodies.** They may contain secrets or PII.

**Treating gRPC error messages as user-facing.** They're for developers; map to user-friendly UI in your front-end.

**Cross-language assumption: empty repeated field vs nil.** Different languages treat these slightly differently; document.

## Performance Notes

- **Per-RPC overhead**: ~50-200 µs server + ~50 µs client (excluding network and your handler).
- **Throughput on a single stream**: gated by HTTP/2 flow control; typically 100s of MB/s within a DC.
- **Concurrent RPCs per conn**: thousands (HTTP/2 multiplexing).
- **Serialisation cost**: ~ns/byte for protobuf; small messages serialise in <1 µs.
- **TLS overhead**: <5% after handshake (modern AES-GCM hardware accel).
- **Memory per server-side connection**: ~50-100 KB.
- **Max throughput**: typically 50k-200k unary RPCs/s/core, gated by GC and goroutine creation.

## How Big Companies Use It

- **Google** uses gRPC (internally Stubby, externally gRPC) for essentially all RPC. Originated there.
- **Netflix** moved many internal services from REST to gRPC; published case studies.
- **Square** uses gRPC + Buf for schema management.
- **Uber** uses gRPC with a custom routing/LB layer.
- **Spotify** uses gRPC with Connect for browser compatibility.
- **Kubernetes** uses gRPC for internal control plane (apiserver, kubelet, CRI, CSI, CNI).
- **etcd** is entirely gRPC.
- **CockroachDB** uses gRPC for inter-node RPC.
- **Cilium** (eBPF networking) uses gRPC for the Hubble observability layer.
- **Tendermint / CosmosSDK** uses gRPC for application interfaces.

## Source Code References

- `grpc-go` repo: https://github.com/grpc/grpc-go.
- Protobuf Go: https://github.com/protocolbuffers/protobuf-go.
- Buf (proto tooling): https://github.com/bufbuild/buf.
- Connect-Go: https://github.com/connectrpc/connect-go.
- OpenTelemetry gRPC: https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/google.golang.org/grpc/otelgrpc.
- grpc-ecosystem extensions (middleware, prometheus): https://github.com/grpc-ecosystem.

## Further Reading

- gRPC docs: https://grpc.io/docs/languages/go/.
- "gRPC Best Practices" (Google Cloud blog).
- "Production gRPC at Square" (Square Engineering).
- "gRPC: Up and Running" (Kasun Indrasiri, O'Reilly).
- "Why Connect?" (Buf blog): https://buf.build/blog/connect-a-better-grpc.
- "Load balancing gRPC" (Kuberenetes blog).
- "gRPC keepalive: deep dive" (various engineering blogs).
- "HTTP/2 spec relevant to gRPC": https://httpwg.org/specs/rfc7540.html.

## Exercises / Self-Check

1. Generate Go stubs from a `.proto` using Buf. Implement a unary `GetUser` server and client. Verify with `grpcurl`.
2. Add a server-streaming `ListUsers` RPC. Have the client honour cancellation: cancel mid-stream and confirm the server's `stream.Context().Err()` returns `context.Canceled`.
3. Wire chained unary interceptors for recovery, logging, and tracing. Trigger a panic; confirm the recovery interceptor handles it gracefully.
4. Configure keepalive on both client and server. Idle the connection for 5 minutes behind a simulated NAT; confirm the next call works.
5. Replace L4 LB with client-side round-robin via DNS resolver. Confirm RPCs distribute across backends.
6. Add a declarative retry policy via service config for `UNAVAILABLE`. Simulate a backend restart; confirm the client succeeds via retry.
7. Implement bidirectional streaming for a chat use case. Concurrently call `Recv` and `Send` from separate goroutines.
8. Set up Connect-Go alongside gRPC on the same proto. Make calls from both a Go gRPC client and a browser `fetch` over Connect. Verify both work.

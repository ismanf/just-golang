# CQRS and Event Sourcing

## TL;DR

**CQRS** (Command Query Responsibility Segregation) splits an application into a **write side** (commands that change state) and a **read side** (queries that return state) — often with different data models for each. **Event Sourcing** stores the application's state as an **immutable log of events** rather than the current snapshot; current state is derived by replaying events. The two are independent but frequently combined: ES is the canonical CQRS write side. In Go, both are pattern-shapes — there's no canonical framework. The single biggest gotcha: **CQRS and ES are heavy commitments**. They solve real problems (audit, replay, scale of reads) but introduce complexity (eventual consistency, event-version migrations, snapshot management) that most CRUD apps never need. Reach for them only when the problem genuinely demands it.

## Mental Model

```
   CRUD (the baseline):
   
   Client → Write to DB (UPDATE) → Read from DB (SELECT)
   The DB is the only state. Last write wins. No history.

   CQRS:
   
   Commands ─── write model ──── DB ─── replicate ──── read model ── Queries
                                                         (denormalized)
   
   Same underlying data, different shapes. Reads are precomputed views.
   
   Event Sourcing:
   
   Command → Validate → Append events to log → ...
   
                   ┌─── Projector A → read model A (e.g., user view)
   Event Log →────┼─── Projector B → read model B (e.g., audit)
                   └─── Projector C → search index
   
   Current state = fold(events, initial). Past states = fold(prefix(events, t)).
```

ES alone: one event log, multiple projections.
CQRS alone: two models, possibly two databases.
Combined: events drive projections; projections serve queries.

## Syntax & Basic Usage

A tiny in-memory event-sourced account:

```go
package account

import (
	"errors"
	"sync"
)

// Events (immutable, named, past-tense).
type Event interface{ isEvent() }

type AccountOpened struct{ ID, Owner string }
type Deposited struct{ ID string; Amount int }
type Withdrew struct{ ID string; Amount int }

func (AccountOpened) isEvent() {}
func (Deposited) isEvent()    {}
func (Withdrew) isEvent()      {}

// Aggregate (state derived from events).
type Account struct {
	ID, Owner string
	Balance   int
	uncommitted []Event
}

func (a *Account) Apply(e Event) {
	switch ev := e.(type) {
	case AccountOpened:
		a.ID = ev.ID
		a.Owner = ev.Owner
	case Deposited:
		a.Balance += ev.Amount
	case Withdrew:
		a.Balance -= ev.Amount
	}
}

// Commands (validated; return new events or error).
func (a *Account) Open(id, owner string) error {
	if a.ID != "" { return errors.New("already opened") }
	e := AccountOpened{ID: id, Owner: owner}
	a.Apply(e)
	a.uncommitted = append(a.uncommitted, e)
	return nil
}

func (a *Account) Deposit(amount int) error {
	if amount <= 0 { return errors.New("amount must be positive") }
	e := Deposited{ID: a.ID, Amount: amount}
	a.Apply(e)
	a.uncommitted = append(a.uncommitted, e)
	return nil
}

func (a *Account) Withdraw(amount int) error {
	if amount <= 0 { return errors.New("amount must be positive") }
	if a.Balance < amount { return errors.New("insufficient funds") }
	e := Withdrew{ID: a.ID, Amount: amount}
	a.Apply(e)
	a.uncommitted = append(a.uncommitted, e)
	return nil
}

// Store and rehydrate.
type Store struct {
	mu     sync.Mutex
	events map[string][]Event
}

func NewStore() *Store { return &Store{events: map[string][]Event{}} }

func (s *Store) Save(id string, events []Event) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.events[id] = append(s.events[id], events...)
	return nil
}

func (s *Store) Load(id string) ([]Event, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	return append([]Event(nil), s.events[id]...), nil
}

func Rehydrate(id string, events []Event) *Account {
	a := &Account{}
	for _, e := range events {
		a.Apply(e)
	}
	a.uncommitted = nil
	return a
}
```

Usage:

```go
s := NewStore()
a := &Account{}
a.Open("acc1", "Alice")
a.Deposit(100)
a.Withdraw(30)
s.Save("acc1", a.uncommitted)

// Later, in another process:
events, _ := s.Load("acc1")
a2 := Rehydrate("acc1", events)
fmt.Println(a2.Balance)  // 70
```

The events are the source of truth; the `Account` struct is a computed view.

## Deep Dive

### When to use CQRS

Use CQRS when:
- Read workload >> write workload (scale reads horizontally with cached projections).
- Read models need different shapes than write model (denormalized search, dashboards).
- Different teams own read vs write (decouples deployments).
- Compliance demands precomputed audit views.

Don't use CQRS when:
- One database table serves both reads and writes adequately.
- Eventual consistency is a problem (users expect immediate read-your-writes).
- The added complexity of two models exceeds the gain.

### When to use Event Sourcing

Use ES when:
- You need full audit trail of *why* state is what it is.
- Replay is required (test fixtures from prod events, rebuild projections, "time-travel debugging").
- Different consumers care about different *aspects* of state changes (search, fraud detection, billing).
- Domain naturally evolves through discrete events (banking, e-commerce, multi-step workflows).

Don't use ES when:
- State is a mutable graph (configuration, hot caches).
- Queries dominate; events are an unnecessary indirection.
- Audit isn't a requirement.
- Schema evolution is too painful for your team to manage (versioning events forever).

### Event shape

Events are:
- **Immutable**: once written, never modified.
- **Named in past tense**: `OrderPlaced`, not `PlaceOrder`.
- **Self-contained**: include all data needed to apply; don't reference external state.
- **Versioned**: as the schema evolves, events get version tags.

```go
type OrderPlaced struct {
    Version   int      `json:"version"`
    OrderID   string   `json:"order_id"`
    CustomerID string  `json:"customer_id"`
    Items     []Item   `json:"items"`
    Total     int      `json:"total_cents"`
    Currency  string   `json:"currency"`
    OccurredAt time.Time `json:"occurred_at"`
}
```

### Commands vs events

| Aspect | Command | Event |
|---|---|---|
| Naming | imperative (`PlaceOrder`) | past tense (`OrderPlaced`) |
| Origin | external (UI, API) | internal (result of command) |
| Can fail? | yes | no (already happened) |
| Mutation | yes | no, append-only |
| Identity | usually has command ID | has event ID + sequence |

A command produces zero or more events. Events are facts.

### Aggregates

An **aggregate** is a cluster of entities edited together with consistency. For an `Account`, the aggregate is just the account. For an `Order`, it might be the order + its line items.

Aggregates enforce business rules; they consume commands and emit events. They are *loaded* by replaying events:

```go
func LoadAggregate[T any](store EventStore, id string, apply func(*T, Event)) *T {
    events, _ := store.Load(id)
    var a T
    for _, e := range events {
        apply(&a, e)
    }
    return &a
}
```

### Optimistic concurrency

When two commands hit the same aggregate concurrently, only one's events should land. Pattern:

```go
type EventStore interface {
    Append(streamID string, expectedVersion int, events []Event) error
    Load(streamID string) (events []Event, version int, err error)
}
```

`expectedVersion` is the version we *think* the stream is at. If the actual version is higher, another command got there first; we error and retry.

```go
events, version, _ := store.Load("acc1")
a := Rehydrate(events)
if err := a.Withdraw(50); err != nil { /* ... */ }
if err := store.Append("acc1", version, a.uncommitted); err == ErrVersionConflict {
    // retry from Load
}
```

Same as `WHERE version=N` in SQL — optimistic concurrency.

### Snapshots

Replaying millions of events on every load is slow. **Snapshots**: periodically save the current state. On load, start from latest snapshot and apply remaining events.

```go
type Snapshot struct {
    AggregateID string
    Version     int
    State       []byte  // serialized aggregate
}
```

Snapshot every 100 events; load 1 snapshot + ≤100 events.

Snapshots are an optimization; events remain the source of truth.

### Projections

A **projection** consumes events and builds a read model:

```go
type UserView struct {
    ID    string
    Email string
    Total int
}

func ProjectUserView(events []Event, view *UserView) {
    for _, e := range events {
        switch ev := e.(type) {
        case UserRegistered:
            view.ID = ev.ID
            view.Email = ev.Email
        case OrderPlaced:
            if ev.CustomerID == view.ID {
                view.Total += ev.Total
            }
        }
    }
}
```

Projections can be:
- **Synchronous** (built into the command path).
- **Asynchronous** (a separate process consumes events, updates a separate database).
- **Eventual** (rebuilt periodically from scratch).

### Event store implementations

Common storage choices:

- **PostgreSQL** with an `events` table (`id, stream_id, version, type, payload, occurred_at`). Append-only by convention.
- **EventStoreDB** (https://www.eventstore.com): purpose-built event store. .NET-native but has Go clients.
- **Kafka**: events as Kafka topic. Replays are easy; storage is unlimited.
- **AWS DynamoDB**: stream + projections via DynamoDB Streams.
- **Custom**: many homegrown stores in Go.

PostgreSQL works for most teams; Kafka shines when you need wide fan-out to many consumers.

### Schema evolution

Events live forever. Schema changes need care:

1. **Add fields**: new field, default value for old events.
2. **Rename fields**: keep the old name; map at deserialize.
3. **Split events**: old `OrderPlaced` becomes new `OrderPlaced` + `OrderItemAdded`. Project past events into the new shape during replay.
4. **Delete events**: usually not; tombstone instead.

The discipline of event versioning is the hidden cost of ES.

### Eventual consistency

Read models lag behind writes by milliseconds to seconds. Users may see stale data immediately after a write.

Mitigations:
- "Read your writes" caching: hold the write's projection result in session for a few seconds.
- Optimistic UI updates.
- Long-polling for state confirmation.
- Document the lag in the API.

### Sagas / process managers

A **saga** coordinates a workflow across aggregates:

```go
type OrderSaga struct{}

func (s *OrderSaga) On(event Event) []Command {
    switch e := event.(type) {
    case OrderPlaced:
        return []Command{ChargeCard{OrderID: e.OrderID, Amount: e.Total}}
    case CardCharged:
        return []Command{ShipOrder{OrderID: e.OrderID}}
    case CardChargeFailed:
        return []Command{CancelOrder{OrderID: e.OrderID}}
    }
    return nil
}
```

A saga is itself an event-driven state machine; see `19-patterns/08-state-machines.md`.

### Frameworks in Go

There's no de-facto standard. Some libraries:

- [`go-cqrs/cqrs`](https://github.com/looplab/eventhorizon) — event sourcing framework.
- [`ThreeDotsLabs/watermill`](https://github.com/ThreeDotsLabs/watermill) — event-driven library; good docs.
- [`asyncapi/go`](https://github.com/asyncapi/go-parser) — AsyncAPI for event schemas.
- Many teams roll their own.

For sagas specifically: Cadence/Temporal (see `20-big-tech/03-uber-microservices.md`) is the popular choice.

### CQRS without ES

You can have CQRS without event sourcing. Pattern:

```
Write side: write to primary DB (last-write-wins; no events).
Read side: separate denormalized DB; updated via CDC (change data capture).
```

CDC tools (Debezium, Postgres logical replication, MySQL binlog) emit changes; consumers update read models.

CQRS without ES is easier to operate; you lose the audit/replay benefits.

### Read your writes

If the user just placed an order, they expect the order page to show it. Strategies:

- Wait briefly (eventual consistency window).
- Return the just-written entity from the command handler.
- Use a "session-stale-read" pattern: cache the user's recent writes in their session.

### Implementing in Go: practical advice

1. Start with one aggregate; don't ES the whole app.
2. Use Postgres; don't reach for Kafka unless replays demand it.
3. Treat events as part of your API: version them.
4. Projections should be idempotent: re-running them rebuilds the same state.
5. Test by replaying recorded events.

## Standard Library Hooks

- `context.Context`: pervasive.
- `database/sql`: event store backend.
- `encoding/json` or protobuf: event serialization.
- `time.Time`: `OccurredAt` on every event.
- `sync.RWMutex` / `sync.Map`: in-memory projections.
- `errors.Is`/`errors.As`: domain errors.

## Real-World Patterns

### 1. Write side: command → events → store

```go
func PlaceOrder(ctx context.Context, store EventStore, cmd PlaceOrderCmd) error {
    events, version, _ := store.Load(cmd.OrderID)
    o := RehydrateOrder(events)
    if err := o.Place(cmd); err != nil { return err }
    return store.Append(cmd.OrderID, version, o.uncommitted)
}
```

### 2. Read side: projection updates view

```go
func ProjectOrders(events <-chan Event, db *sql.DB) {
    for e := range events {
        switch ev := e.(type) {
        case OrderPlaced:
            db.Exec(`INSERT INTO orders_view (id, customer, total, status) VALUES ($1,$2,$3,'placed')`,
                ev.OrderID, ev.CustomerID, ev.Total)
        case OrderShipped:
            db.Exec(`UPDATE orders_view SET status='shipped' WHERE id=$1`, ev.OrderID)
        }
    }
}
```

### 3. Event store on Postgres

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    stream_id TEXT NOT NULL,
    version INTEGER NOT NULL,
    type TEXT NOT NULL,
    payload JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
);
```

The `UNIQUE (stream_id, version)` enforces optimistic concurrency at the DB level.

### 4. Snapshot every N events

```go
func MaybeSnapshot(store SnapshotStore, agg *Account) error {
    if len(agg.uncommitted) == 0 { return nil }
    if agg.Version % 100 != 0 { return nil }
    return store.Save(Snapshot{AggregateID: agg.ID, Version: agg.Version, State: serialize(agg)})
}
```

### 5. Replay from prod for tests

```go
func TestProjector_FromProductionEvents(t *testing.T) {
    events := loadEventsFromFile("testdata/prod_events.json")
    var view UserView
    for _, e := range events { ProjectUserView([]Event{e}, &view) }
    require.Equal(t, expected, view)
}
```

Real prod events; deterministic test.

## Anti-Patterns & Gotchas

**"Event sourcing for everything"**. ES has real costs. Choose per-domain.

**Events that reference mutable external state**. `OrderPlaced{Items: pointerToInventoryItems}` — pointers go stale. Embed the data.

**Skipping versioning until needed**. By then it's chaos. Version events from day one.

**Mutating events to "fix" them**. Never. Append a corrective event instead.

**Coupling projections too tightly to events**. If your read model breaks every time the event schema changes, you're projecting too directly. Use intermediate domain language.

**Snapshots that drift from events**. Snapshots must be re-derivable from events. If your snapshot has more info than events would replay to, you've broken the system.

**Synchronous projections holding up writes**. Projections should be async unless they're cheap.

**Sagas as monoliths**. Use a workflow engine (Temporal/Cadence) for complex sagas; ad-hoc Go code becomes a maintenance nightmare.

**ES without team buy-in**. The pattern needs everyone to understand and maintain it. Cargo-culting ES into a team that doesn't see the benefit is painful.

**Treating Kafka as your only persistent store**. Compaction settings matter; you don't want events deleted accidentally.

## Performance Notes

- Append to event log (Postgres): ~ms.
- Replay 1000 events to rebuild aggregate: ~10 ms.
- Snapshot deserialize: ~ms.
- Projection update (async): seconds of lag typical.
- Kafka throughput as event store: ~100k events/sec/partition.
- Event store size growth: linear; needs retention or archival.

## How Big Companies Use It

- **Banking and fintech** (often regulated): ES for audit trail.
- **E-commerce**: shipping/payment workflows as sagas.
- **Cadence/Temporal users** (Uber, Coinbase, Snap, DoorDash): workflows are sometimes ES-shaped.
- **Mercari**: CQRS + ES public talks.
- **Walmart Labs**: documented internal use.
- **Insurance companies**: regulatory replay requirements.
- **GitHub**: events are a public API (their HTTP `events` API) but their internal storage isn't pure ES.

## Source Code References

- looplab/eventhorizon: [`looplab/eventhorizon`](https://github.com/looplab/eventhorizon).
- Watermill: [`ThreeDotsLabs/watermill`](https://github.com/ThreeDotsLabs/watermill).
- EventStoreDB Go client: [`EventStore/EventStore-Client-Go`](https://github.com/EventStore/EventStore-Client-Go).
- Cadence Go SDK: [`uber/cadence-client`](https://github.com/uber/cadence-client).
- Temporal Go SDK: [`temporalio/sdk-go`](https://github.com/temporalio/sdk-go).
- A canonical Go ES example: [`ThreeDotsLabs/wild-workouts-go-ddd-example`](https://github.com/ThreeDotsLabs/wild-workouts-go-ddd-example).

## Further Reading

- Greg Young, "CQRS Documents": https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf.
- Martin Fowler, "Event Sourcing": https://martinfowler.com/eaaDev/EventSourcing.html.
- Martin Fowler, "CQRS": https://martinfowler.com/bliki/CQRS.html.
- Vaughn Vernon, *Implementing Domain-Driven Design* (book).
- Three Dots Labs, "Combining CQRS, DDD, and Clean Architecture": https://threedots.tech/.
- "Versioning in an Event Sourced System" — Greg Young.
- Eric Evans, *Domain-Driven Design* (book).
- "Watermill: Event-driven Go applications": https://watermill.io.
- Cadence vs Temporal docs.

## Exercises / Self-Check

1. Build a tiny Account aggregate (Open/Deposit/Withdraw). Persist events in Postgres. Rebuild on demand.
2. Add a projection that maintains a read model `account_balances(id, balance)`. Verify it stays consistent under concurrent writes.
3. Add snapshots every 100 events. Compare load time with and without.
4. Implement schema evolution: rename `Amount` to `AmountCents` in `Deposited`. How do you read old events?
5. Identify a feature in your work app that would benefit from ES. What about the rest of the app argues *against* ES?

# State Machines

## TL;DR

A **state machine** has a finite set of **states**, a finite set of **events**, and **transitions** (state × event → new state, possibly with side effects). In Go, three implementations dominate: **typed enum + switch statement** (simplest, ubiquitous), **transition table + map lookup** (declarative, easy to visualize), and **state pattern with interfaces** (verbose, useful when each state owns rich behavior). The single biggest gotcha: **don't reach for a state machine library prematurely**. Most Go state machines are 30 lines of `switch state, event`; libraries add ceremony, hide validation, and rarely help. Reach for explicit FSMs when invalid-transition bugs hurt you (orders, workflows, connection lifecycles); skip them for trivial 2-3-state toggles.

## Mental Model

```
   States and transitions:
   
   ┌─────────┐    place    ┌──────────┐    charge   ┌─────────┐
   │  Cart   │ ──────────▶ │ Pending  │ ──────────▶ │  Paid   │
   └─────────┘             └────┬─────┘             └────┬────┘
                                │                        │
                                │ cancel                 │ ship
                                ▼                        ▼
                          ┌──────────┐             ┌─────────┐
                          │ Canceled │             │ Shipped │
                          └──────────┘             └─────────┘

   Implementation choice depends on:
     - Number of states & events
     - Complexity of side effects per transition
     - Need for runtime introspection (visualization, validation)
```

## Syntax & Basic Usage

The simplest Go FSM — typed enum + switch:

```go
package order

import "errors"

type State int

const (
	Cart State = iota
	Pending
	Paid
	Shipped
	Canceled
)

func (s State) String() string {
	return [...]string{"Cart", "Pending", "Paid", "Shipped", "Canceled"}[s]
}

type Event int

const (
	Place Event = iota
	Charge
	Ship
	Cancel
)

func (e Event) String() string {
	return [...]string{"Place", "Charge", "Ship", "Cancel"}[e]
}

type Order struct {
	ID    string
	State State
}

func (o *Order) Apply(e Event) error {
	next, ok := transition(o.State, e)
	if !ok {
		return errors.New("invalid transition: " + o.State.String() + " --" + e.String() + "-->")
	}
	o.State = next
	return nil
}

func transition(s State, e Event) (State, bool) {
	switch s {
	case Cart:
		if e == Place { return Pending, true }
	case Pending:
		switch e {
		case Charge: return Paid, true
		case Cancel: return Canceled, true
		}
	case Paid:
		if e == Ship { return Shipped, true }
	}
	return s, false
}
```

Usage:

```go
o := &Order{ID: "o1", State: Cart}
o.Apply(Place)  // Cart → Pending
o.Apply(Charge) // Pending → Paid
o.Apply(Ship)   // Paid → Shipped
o.Apply(Cancel) // invalid: returns error
```

Easy to read, easy to test, no dependencies.

## Deep Dive

### The three implementations

#### 1. `switch` on state + event

(Shown above.) Pros: simplest. Cons: scales poorly to many transitions (each new event adds a `case`).

#### 2. Transition table

```go
type trans struct{ from State; on Event; to State }

var transitions = []trans{
    {Cart, Place, Pending},
    {Pending, Charge, Paid},
    {Pending, Cancel, Canceled},
    {Paid, Ship, Shipped},
}

func transition(s State, e Event) (State, bool) {
    for _, t := range transitions {
        if t.from == s && t.on == e { return t.to, true }
    }
    return s, false
}
```

Or via map:

```go
type key struct{ State State; Event Event }
var table = map[key]State{
    {Cart, Place}:    Pending,
    {Pending, Charge}: Paid,
    {Pending, Cancel}: Canceled,
    {Paid, Ship}:     Shipped,
}

func transition(s State, e Event) (State, bool) {
    next, ok := table[key{s, e}]
    return next, ok
}
```

Pros: declarative, easy to dump as DOT graphviz, easy to visualize. Cons: side effects per transition need to live elsewhere.

#### 3. State pattern (interfaces)

```go
type State interface {
    Place(*Order) State
    Charge(*Order) State
    Ship(*Order) State
    Cancel(*Order) State
}

type cartState struct{}
func (cartState) Place(o *Order) State   { return pendingState{} }
func (cartState) Charge(o *Order) State  { return cartState{} } // no-op
func (cartState) Ship(o *Order) State    { return cartState{} }
func (cartState) Cancel(o *Order) State  { return cartState{} }

type pendingState struct{}
func (pendingState) Place(o *Order) State  { return pendingState{} }
func (pendingState) Charge(o *Order) State { return paidState{} }
// etc.

type Order struct {
    ID    string
    state State
}

func (o *Order) Place() { o.state = o.state.Place(o) }
```

Pros: each state encapsulates its own logic. Cons: lots of boilerplate; every state implements every method; in Go this means many empty implementations.

The state pattern shines when each state has *rich* behavior (different validation, different I/O, different downstream effects). For simple transitions it's overkill.

### Side effects per transition

Real state machines do work on transition: send emails, write to DB, fire metrics. Where does that go?

#### Inline in the transition function

```go
func transition(o *Order, e Event) error {
    switch {
    case o.State == Pending && e == Charge:
        if err := chargeCard(o.PaymentID); err != nil { return err }
        o.State = Paid
        notifyCustomer(o.ID, "Payment received")
        return nil
    // ...
    }
    return errors.New("invalid")
}
```

Side effects coupled with transition. Hard to test.

#### Hooks before/after

```go
type Order struct {
    State State
    onEnter map[State]func(*Order) error
    onExit  map[State]func(*Order) error
}

func (o *Order) Apply(e Event) error {
    next, ok := transition(o.State, e)
    if !ok { return errors.New("invalid") }
    if exit, ok := o.onExit[o.State]; ok {
        if err := exit(o); err != nil { return err }
    }
    o.State = next
    if enter, ok := o.onEnter[next]; ok {
        if err := enter(o); err != nil { return err }
    }
    return nil
}
```

Hooks attached separately. Cleaner separation.

#### Return events to apply

```go
func transition(o *Order, e Event) ([]Effect, error) {
    switch {
    case o.State == Pending && e == Charge:
        return []Effect{ChargeCard{ID: o.PaymentID}, MarkPaid{ID: o.ID}}, nil
    }
    return nil, errors.New("invalid")
}

// Caller dispatches effects.
```

Pure transition: no I/O. Caller (or interpreter) executes effects. Easy to test.

### Guards

A guard is a predicate that must hold for a transition to fire:

```go
func transition(o *Order, e Event) (State, bool) {
    switch {
    case o.State == Pending && e == Charge:
        if o.Total <= 0 { return o.State, false } // guard
        return Paid, true
    }
    return o.State, false
}
```

Guards prevent transitions that would violate invariants.

### Hierarchical state machines

State patterns with nested states (Harel statecharts):

```
Order
├── Active
│   ├── Cart
│   ├── Pending
│   └── Paid
└── Final
    ├── Shipped
    └── Canceled
```

Transitions can fire on the parent state, applying to all children. Useful for "in any sub-state, allow cancel".

Implementations: explicit nested state machines, or flat representations with conditional logic. Hierarchical machines are rare in Go; usually overkill.

### Async state machines

Some events arrive from outside (timer expiry, network response). Pattern:

```go
func (o *Order) Run(ctx context.Context, events <-chan Event) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case e := <-events:
            if err := o.Apply(e); err != nil { /* log */ }
        }
    }
}
```

Goroutine per state machine, channel of events, single-threaded apply.

### State machines for connections

`net.Conn` lifecycle: `closed` → `accepted` → `established` → `closing` → `closed`. TCP itself is a state machine.

```go
type conn struct {
    state atomic.Int32 // current state
}

const (
    stateAccepted int32 = iota
    stateEstablished
    stateClosing
    stateClosed
)
```

Atomic state for lock-free reads; CAS for transitions.

### State machines for workflows

For multi-step workflows (order processing, deployment pipelines), explicit FSMs help:

```go
type Step struct {
    State State
    Fn    func(ctx context.Context) error
    Next  State
    OnErr State
}

var pipeline = []Step{
    {State: Start, Fn: validate, Next: Validated, OnErr: Failed},
    {State: Validated, Fn: process, Next: Processed, OnErr: Failed},
    {State: Processed, Fn: notify, Next: Completed, OnErr: Failed},
}
```

For complex workflows, use **Temporal/Cadence** (see `20-big-tech/03-uber-microservices.md`) which gives you durable, replayable workflows out of the box.

### Visualizing

A transition table can be exported to graphviz DOT:

```go
func toDot(transitions []trans) string {
    var sb strings.Builder
    sb.WriteString("digraph fsm {\n")
    for _, t := range transitions {
        fmt.Fprintf(&sb, "  %s -> %s [label=%q]\n", t.from, t.to, t.on.String())
    }
    sb.WriteString("}\n")
    return sb.String()
}
```

```bash
$ go run main.go > fsm.dot
$ dot -Tpng fsm.dot -o fsm.png
```

Excellent for design docs.

### Validation: prove all transitions reachable

In tests:

```go
func TestAllStatesReachable(t *testing.T) {
    visited := map[State]bool{Cart: true}
    queue := []State{Cart}
    for len(queue) > 0 {
        s := queue[0]
        queue = queue[1:]
        for _, e := range AllEvents {
            if next, ok := transition(s, e); ok && !visited[next] {
                visited[next] = true
                queue = append(queue, next)
            }
        }
    }
    for _, s := range AllStates {
        if !visited[s] { t.Errorf("unreachable state: %s", s) }
    }
}
```

Catch design bugs early.

### Persistence

Persist the state name (not the state object):

```sql
CREATE TABLE orders (id TEXT PRIMARY KEY, state TEXT);
```

On load: `o.State = parseState(row.state)`. Don't persist `o.state` (the interface in the state pattern); the type-vs-string mismatch invites bugs.

### Concurrency

A single state machine instance is usually owned by one goroutine. Concurrent access requires:

- Mutex around `Apply`.
- Or channel-based actor: send events, the goroutine processes serially.

Sharing one FSM across goroutines without sync is a race condition factory.

### Go libraries

- [`looplab/fsm`](https://github.com/looplab/fsm): declarative FSM.
- [`qmuntal/stateless`](https://github.com/qmuntal/stateless): C#-Stateless port.
- [`smallnest/gen`](https://github.com/smallnest/fsm): minimalistic.

Most teams write their own. The pattern is small enough.

### Temporal/Cadence: managed FSMs

Cadence/Temporal workflows are durable, replayable state machines. Stateful, survive crashes, support sleeps of days. Use them when the workflow is long-lived and resumable; use plain Go FSMs when transitions are millisecond-scoped.

## Standard Library Hooks

- `iota` for state constants.
- `String()` method per state via `[...]string{}` table.
- `context.Context` for shutdown of long-running state machines.
- `sync.Mutex` / `atomic.Int32` for concurrent FSMs.
- `time.Timer`/`time.Ticker` for timeout transitions.

## Real-World Patterns

### 1. Document the FSM in code comments

```go
// Order state machine:
//
//   Cart --place--> Pending --charge--> Paid --ship--> Shipped
//                         \
//                          \--cancel--> Canceled
//
//   Invalid transitions return an error.
```

Future readers thank you.

### 2. Event-driven with select

```go
func (o *Order) Run(ctx context.Context, events <-chan Event) {
    for {
        select {
        case <-ctx.Done(): return
        case e := <-events:
            if err := o.Apply(e); err != nil { /* log + metric */ }
        case <-time.After(o.timeoutForState()):
            o.Apply(Timeout)
        }
    }
}
```

Timer per state gives auto-timeout transitions.

### 3. State machine with logger + metrics

```go
func (o *Order) Apply(e Event) error {
    prev := o.State
    next, ok := transition(o.State, e)
    if !ok {
        invalidTransition.Inc(prev.String(), e.String())
        return errInvalidTransition
    }
    o.State = next
    transitionCount.Inc(prev.String(), e.String(), next.String())
    log.Info("transition", "order", o.ID, "from", prev, "to", next, "event", e)
    return nil
}
```

Telemetry on every transition makes debugging easy.

### 4. State machine as a workflow

```go
type ShipmentWorkflow struct{ state, prevState State }

func (w *ShipmentWorkflow) Step(ctx context.Context) error {
    switch w.state {
    case Pending:
        if err := w.label(); err != nil { return err }
        w.state = Labeled
    case Labeled:
        if err := w.pickup(); err != nil { return err }
        w.state = PickedUp
    case PickedUp:
        if err := w.deliver(ctx); err != nil { return err }
        w.state = Delivered
    case Delivered:
        return nil
    }
    return nil
}
```

Each call advances one step. Idempotent; safe to retry.

### 5. State machine as an HTTP API

```go
func (h *Handler) Charge(w http.ResponseWriter, r *http.Request) {
    o, _ := h.repo.Get(r.Context(), r.URL.Query().Get("id"))
    if err := o.Apply(Charge); err != nil {
        http.Error(w, err.Error(), 400)
        return
    }
    h.repo.Save(r.Context(), o)
    json.NewEncoder(w).Encode(o)
}
```

REST endpoints map to events; the FSM enforces valid transitions.

## Anti-Patterns & Gotchas

**State machine without explicit invalid-transition handling.** Silently no-op transitions hide bugs. Always return an error or panic.

**Side effects in the transition function before validation.** `chargeCard()` succeeded but state didn't transition because of a guard fail — money gone.

**Persisting state objects (not state names).** Refactoring breaks loaded objects. Persist a string/enum.

**Sharing one FSM across goroutines without sync.** Races.

**State pattern with 30 empty methods.** The state pattern is overkill for simple transitions. Use a switch.

**Trying to retro-fit an FSM library onto existing if-else code.** Often the library API is more rigid than your domain needs.

**"All states equally reachable" without thinking.** Some states should be terminal (Canceled, Completed). Verify with reachability tests.

**FSM for everything.** A two-state on/off toggle is just `bool`.

**Mixing async event handling and synchronous transition calls.** Pick one model per FSM.

**Ignoring timer transitions.** A real-world FSM rarely has only "internal" events; timeouts happen.

**Forgetting that events can race with timeouts.** Order matters; document precedence.

## Performance Notes

- `switch` transition: ~ns.
- Map-based transition lookup: ~10-20 ns.
- State pattern (interface dispatch): ~ns + indirection.
- Mutex around transition: ~20-50 ns under no contention.
- Atomic CAS for state: ~10 ns.

FSM overhead is rarely the bottleneck. Side effects are.

## How Big Companies Use It

- **Kubernetes pods**: lifecycle is a state machine (Pending → Running → Succeeded/Failed).
- **Cadence/Temporal workflows**: explicit state machines at framework level.
- **Stripe payments**: charge lifecycle (created → succeeded/failed; refund states).
- **AWS Step Functions**: state machine as a service.
- **OpenSSL**: TLS handshake is a state machine.
- **gRPC connection lifecycle**: explicit state machine in `grpc-go`.
- **Most game engines**: AI / NPC behavior as state machines.

## Source Code References

- looplab/fsm: [`looplab/fsm`](https://github.com/looplab/fsm).
- qmuntal/stateless: [`qmuntal/stateless`](https://github.com/qmuntal/stateless).
- Kubernetes pod phase: [`api/core/v1/types.go`](https://github.com/kubernetes/kubernetes) — search `PodPhase`.
- gRPC connectivity state: [`grpc-go/connectivity/connectivity.go`](https://github.com/grpc/grpc-go/blob/master/connectivity/connectivity.go).
- TCP state machine in netstack: [`gvisor.dev/pkg/tcpip/transport/tcp`](https://github.com/google/gvisor/tree/master/pkg/tcpip/transport/tcp).
- Temporal Go SDK workflow state: [`temporalio/sdk-go`](https://github.com/temporalio/sdk-go).

## Further Reading

- David Harel, "Statecharts" (the canonical paper): https://www.wisdom.weizmann.ac.il/~harel/papers/Statecharts.pdf.
- Erlang's `gen_statem`: the reference implementation.
- "Finite State Machines in Go" — many community blog posts.
- Refactoring Guru's State Pattern: https://refactoring.guru/design-patterns/state.
- Temporal docs (state-machine-as-workflow): https://temporal.io.
- "Statecharts: A Visual Formalism" — David Harel.
- "Modeling state in Go with explicit FSMs" — Mat Ryer talks.

## Exercises / Self-Check

1. Implement the Order FSM (Cart → Pending → Paid → Shipped + Canceled) with the switch approach. Add tests for every transition and every invalid transition.
2. Convert to the transition-table approach. Export it as graphviz DOT.
3. Add side effects: charge a fake card on Charge, send an email on Ship. Decide where they go (inline, hooks, returned effects).
4. Implement a connection FSM (closed → connecting → established → closing → closed) using atomic state. Add tests for concurrent transitions.
5. When would you reach for Temporal/Cadence instead of a Go-native FSM? Sketch a workflow that needs durability.

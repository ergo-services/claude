# Pool, Router, and WebWorker

Three `act` behaviors that sit in front of worker actors and forward work to them. All three are ordinary processes you embed and spawn like any actor, but instead of doing the work themselves they distribute it:

- **`act.Pool`** - a worker farm. Round-robins normal-priority traffic across N identical workers, with backpressure and automatic restart. Use it for horizontal scaling of stateless, order-independent work.
- **`act.Router`** - a dispatcher. Asks your code which named route should handle each message, forwards there. Use it for content-based dispatch, key-affinity sharding, and CQRS. It composes with Pool.
- **`act.WebWorker`** - an HTTP worker. Detects `meta.MessageWebRequest`, dispatches to per-method callbacks, and calls `Done()` for you. Use it behind the Web meta process (see `meta.md`) instead of hand-rolling request handling.

All three preserve the original sender: a worker sees `from`/`ref` as if the sender had targeted it directly, so responses go straight back to the caller and never through Pool or Router.

---

# Pool

`act.Pool` spawns workers at init and forwards each normal-priority message to an available worker based on mailbox backpressure. Unlike `SimpleOneForOne`, the Pool itself receives messages and load-balances them; the sender addresses the Pool PID and never learns which worker ran.

## PoolBehavior

```go
type PoolBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) (PoolOptions, error)
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

Only `Init` is mandatory; the rest have defaults that log a warning and return nil.

The Pool's `HandleMessage`/`HandleCall` are the Pool's own handlers for management traffic, not the worker's. Normal-priority traffic is forwarded to a worker and reaches the worker's callbacks; only higher-priority traffic reaches the Pool's.

| Message path | Who handles |
|--------------|-------------|
| Sent with `MessagePriorityMax` (Urgent queue) | Pool's `HandleMessage` - scale commands, reconfiguration |
| Sent with `MessagePriorityHigh` (System queue) | Pool's `HandleMessage` |
| Sent with `MessagePriorityNormal` (default) | Forwarded to a worker; worker's `HandleMessage` runs |
| Exit / event / inspect messages | Pool's own callbacks |
| Normal-priority message when all worker mailboxes full | Dropped, `messages_unhandled` increments |

High-priority `HandleCall` that returns `(nil, nil)` is not forwarded to a worker - the request is simply ignored and the caller times out. Only normal priority reaches workers.

## PoolOptions

```go
type PoolOptions struct {
    WorkerMailboxSize int64              // per-worker mailbox depth
    PoolSize          int64              // number of workers
    WorkerFactory     gen.ProcessFactory
    WorkerArgs        []any              // passed to each worker's Init
}
```

If `PoolSize < 1` (including the zero value), the Pool falls back to **3** workers rather than erroring.

**Total capacity** = `PoolSize * WorkerMailboxSize`. There is no Pool-level buffer beyond the workers' mailboxes. Two distinct backpressure layers apply:

- **Pool's own mailbox full at send time.** A local sender always gets `gen.ErrProcessMailboxFull` back; a remote sender gets it only with important delivery, otherwise the send is fire-and-forget with no feedback. The message never reaches the Pool.
- **All worker mailboxes full after the Pool accepted the message.** Once a normal-priority message is in the Pool's Main queue, the sender has already received success. If the Pool then cannot forward it because every worker mailbox is full, the drop is silent regardless of important delivery - it is only logged and counted as `messages_unhandled`.

## Dynamic Scaling

```go
newSize, err := p.AddWorkers(n)    // spawn N more workers, returns new size
newSize, err := p.RemoveWorkers(n) // terminate N workers (SendExit, normal), returns new size
```

Both return `gen.ErrNotAllowed` unless the Pool is in `ProcessStateRunning`. `RemoveWorkers` returns `act.ErrPoolEmpty` if asked to remove more workers than remain. New workers use the same factory, args, and mailbox size as init.

## Worker Restarts

Workers are spawned with `LinkParent: true`. The Pool does not run a supervision strategy - instead, `forward` transparently respawns a worker the moment it discovers one is gone: if forwarding fails with `gen.ErrProcessUnknown` or `gen.ErrProcessTerminated`, the Pool spawns a replacement (same factory/args), forwards the message to it, and increments `worker_restarts`. A worker can therefore vanish and be silently recreated even without `RemoveWorkers`, which is why worker state must not be relied on across messages. For real restart policy (intensity limits, escalation), put the Pool under a Supervisor.

## Inspection

`node.Inspect(poolPID)` returns:

| Key | Meaning |
|-----|---------|
| `pool_size` | configured worker count |
| `worker_behavior` | type name of the worker behavior |
| `worker_mailbox_size` | per-worker mailbox limit |
| `worker_restarts` | workers respawned during forwarding |
| `messages_forwarded` | total messages forwarded to workers |
| `messages_unhandled` | messages dropped (all workers full) |

High `messages_unhandled` means the Pool is undersized or workers are too slow; high `worker_restarts` points at worker instability.

## Worker Implementation

A worker is an ordinary `act.Actor`. It receives forwarded messages in its own `HandleMessage`, with `from` set to the original sender.

```go
type Worker struct {
    act.Actor
}

func createWorker() gen.ProcessBehavior {
    return &Worker{}
}

func (w *Worker) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case WorkRequest:
        w.Send(from, WorkResult{ID: m.ID, Result: process(m.Data)}) // reply to original sender
    }
    return nil
}
```

## Minimal Pool Setup

```go
type ComputePool struct {
    act.Pool
}

func createPool() gen.ProcessBehavior {
    return &ComputePool{}
}

func (p *ComputePool) Init(args ...any) (act.PoolOptions, error) {
    return act.PoolOptions{
        PoolSize:          10,
        WorkerMailboxSize: 100,
        WorkerFactory:     createWorker,
    }, nil // total capacity = 1000 in flight
}

// Runtime scaling via a high-priority command.
type MessageScaleUp struct{ By int }

func (p *ComputePool) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case MessageScaleUp:
        newSize, err := p.AddWorkers(m.By)
        if err != nil {
            p.Log().Error("scale failed: %s", err)
            return nil
        }
        p.Log().Info("scaled to %d workers", newSize)
    }
    return nil
}
```

Route work by sending to the Pool PID; send management commands with high priority so they reach the Pool, not a worker:

```go
node.Send(poolPID, WorkRequest{ID: "1", Data: payload})                    // -> a worker
process.SendWithPriority(poolPID, MessageScaleUp{By: 5}, gen.MessagePriorityHigh) // -> Pool
```

## When to Use Pool

| Message rate | Work items | Worker state | Use Pool? |
|--------------|------------|--------------|-----------|
| < ~1000 msg/sec | any | any | No - a single actor suffices |
| > ~1000 msg/sec | independent | stateless | Yes |
| any | ordered per key | any | No - use `act.Router` for key affinity |
| any | share state | stateful | No - state cannot be split |

## Pool vs TCPServer.ProcessPool

Do not use `act.Pool` as a target in `TCPServerOptions.ProcessPool`. That field expects `[]gen.Atom` (registered names) and binds each connection to a fixed member by index, keeping per-connection protocol state ordered. A Pool round-robins per message, which breaks the connection-to-worker binding. Use static named processes there; use `act.Pool` only for internal, order-independent work distribution.

## Anti-Patterns

- **Low-volume work** - routing overhead is wasted below ~1000 msg/sec.
- **`act.Pool` in `TCPServerOptions.ProcessPool`** - see above.
- **Relying on worker state across messages** - `RemoveWorkers` kills arbitrary workers and `forward` silently respawns dead ones.
- **Ordered messages** - no ordering guarantee across workers; shard with `act.Router` instead.
- **Blind `Call` to a Pool** - a worker may be unavailable; use `CallWithTimeout` and handle `gen.ErrTimeout` / `gen.ErrProcessMailboxFull`.

---

# Router

`act.Router` dispatches each message to a route your code chooses by name. A Pool picks the next worker in order without looking at the message; a Router asks `RouteMessage` / `RouteCall` which named route should handle it, resolves that name to a PID, and forwards. Route names are stable across worker restarts, so hash affinity and content dispatch stay coherent through failures.

## RouterBehavior

```go
type RouterBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) (RouterOptions, error)
    Terminate(reason error)

    RouteMessage(from gen.PID, message any) gen.Atom
    RouteCall(from gen.PID, ref gen.Ref, request any) gen.Atom

    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)

    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`Init`, `RouteMessage`, and `RouteCall` are mandatory. `act.Router` deliberately provides **no** defaults for the two routing callbacks, so the compiler refuses to build a router that cannot route. If you only route one direction, implement the other in one line returning `act.RouteDiscard`. The remaining callbacks have defaults (log a warning / return routing stats / no-op).

## RouterOptions and Route

```go
type RouterOptions struct {
    Routes      []Route
    MailboxSize int64
}

type Route struct {
    Name    gen.Atom
    Factory gen.ProcessFactory
    Args    []any
}
```

Every route in `Routes` is spawned during init with `LinkParent` and `LinkChild` set. Init is atomic: if any route fails to spawn (or has an empty `Name` / nil `Factory` / duplicate name), the Router does not start. Return `RouterOptions{}` with no routes for a **free router** - it owns no workers and resolves every name through the node registry (the gateway pattern). Owned routes and registry fallback can mix.

A route's factory can return anything - an actor, a pool, a supervisor, another router. Composition is why Router stays small.

## Routing Decisions

Normal-priority traffic goes through `RouteMessage` (async) / `RouteCall` (sync). The returned `gen.Atom` is resolved:

1. **Owned route first.** If the name matches a route from `Init` or `AddRoute`, forward to that route's worker.
2. **Local registry fallback.** Otherwise look the name up in the node's process registry and forward to that process.
3. **Neither.** Async: a `MessageRouteFailed` is delivered to `HandleMessage` (sender is not notified). Sync: the failure reason is returned to the caller.

Returning `act.RouteDiscard` (the empty atom, `RouteDiscard gen.Atom = ""`) drops the message: the `discarded` counter increments and the async sender gets nothing; a sync caller receives `gen.ErrDiscarded`.

Higher-priority traffic skips routing entirely and reaches the Router's own `HandleMessage`/`HandleCall`:

| Message path | Who handles |
|--------------|-------------|
| Normal priority `Send` | `RouteMessage`, then forward |
| Normal priority `Call` | `RouteCall`, then forward |
| High / Max priority `Send` | Router's `HandleMessage` |
| High / Max priority `Call` | Router's `HandleCall` |
| `MessageRouteFailed` (async forward failed) | Router's `HandleMessage` |

The high/max priority queue is the admin channel: use it for runtime management (adding routes, reconfiguration, stats) the Router should answer rather than forward.

## Route-Management API

Methods on the embedded `*act.Router`, callable from inside callbacks or outside, only while the Router is running.

| Method | Purpose | Notable errors |
|--------|---------|----------------|
| `Routes() []RouterRouteInfo` | snapshot of all routes | none (works in any state) |
| `Route(name) (RouterRouteInfo, bool)` | snapshot of one route; `ok` false if unknown | none (works in any state) |
| `AddRoute(Route) error` | append + spawn an owned route | `act.ErrRouteDuplicate`; empty name / nil factory error |
| `RemoveRoute(name) error` | shut down worker, drop the route; idempotent for unknown | - |
| `DisableRoute(name) error` | kill worker, mark disabled, keep slot; idempotent | `gen.ErrNoRoute` |
| `EnableRoute(name) error` | clear disabled, spawn fresh worker; idempotent | `gen.ErrNoRoute` |
| `ReplaceRoute(name, Route) error` | swap spec + respawn; spec `Name` must match or be empty | `gen.ErrNoRoute`; name-mismatch / nil-factory error |
| `RespawnRoute(name) error` | spawn a route that is currently down | `act.ErrRouteRunning`, `gen.ErrDisabled`, `gen.ErrNoRoute` |

All mutating methods return `gen.ErrNotAllowed` when the Router is not `ProcessStateRunning`, and `gen.ErrBusy` when a pending operation is already in flight on that route. On error the route's state is unchanged and the call can be retried. `Routes()` / `Route()` return immutable snapshots and work in any state.

```go
type RouterRouteInfo struct {
    Name     gen.Atom
    PID      gen.PID       // empty when the route is not running
    Disabled bool
    Pending  RoutePending
}
```

## Pending Operations

`DisableRoute`, `ReplaceRoute`, and `RemoveRoute` terminate the current worker first. Termination is asynchronous, so the Router records the intent and waits for the worker's exit before finalizing. During that wait the route is **pending**:

```go
const (
    RoutePendingNone    RoutePending = 0
    RoutePendingDisable RoutePending = 1
    RoutePendingReplace RoutePending = 2
    RoutePendingRemove  RoutePending = 3
)
```

While pending, every management call on that route returns `gen.ErrBusy`, and forwarding to it fails with a reason matching the pending action (see below). If the worker is already dead when you call these methods, the change completes synchronously and never enters pending state. Pending is per route - concurrent operations on different routes are fine.

## Worker Lifecycle

Owned routes are kept alive automatically. When a route's worker dies, the Router eagerly spawns a replacement from the same spec into the same slot, keeping the name stable, and increments `restarts`. If the respawn itself fails, the Router logs it and leaves the slot empty rather than crashing; the next message routed there retries the spawn (and `forward` respawns-and-retries once on `gen.ErrProcessUnknown` / `gen.ErrProcessTerminated`). This is minimal keep-alive, not supervision - there is no intensity limit or escalation. For a strict restart contract or mailbox preservation, supervise the worker under its own Supervisor **outside** the Router and route to its registered name through the registry. Do not put a Supervisor in a route slot: the Router would forward messages to the Supervisor process, which cannot use them.

## MessageRouteFailed

When a routed **async** send cannot be forwarded, the Router delivers this to its own `HandleMessage`. It is the only feedback for async routing failures - ignore it and such messages vanish silently.

```go
type MessageRouteFailed struct {
    Name    gen.Atom  // target returned by RouteMessage
    From    gen.PID   // original sender
    Message any       // the undelivered message
    Reason  error     // why it failed
}
```

| Reason | Meaning |
|--------|---------|
| `gen.ErrProcessUnknown` | name resolved to nothing (no route, no registry entry, or respawn failed) |
| `gen.ErrProcessMailboxFull` | target mailbox is full |
| `gen.ErrDisabled` | target route is disabled or mid-disable |
| `gen.ErrNoRoute` | target route is being removed |
| `gen.ErrBusy` | target route is mid-replace with no live worker |

Return non-nil from `HandleMessage` on the failure to terminate the Router; return nil to keep running (log, or persist to a dead-letter queue for replay). For **sync** calls there is no `MessageRouteFailed` - the same reason is returned to the caller directly.

## Inspection

Default `HandleInspect` returns router-level counters plus per-route entries: `type`, `routes_total`, `routes_active`, `routes_disabled`, `routes_pending`, `mailbox_size`, `forwarded`, `discarded`, `failed`, `restarts`, and `route:NAME:pid` / `route:NAME:disabled` / `route:NAME:pending` per route. Override to add your own fields.

## Content-Based Dispatch

```go
type EventRouter struct {
    act.Router
}

func (r *EventRouter) Init(args ...any) (act.RouterOptions, error) {
    return act.RouterOptions{
        Routes: []act.Route{
            {Name: "payments", Factory: factoryPaymentsWorker},
            {Name: "shipments", Factory: factoryShipmentsWorker},
            {Name: "reports", Factory: factoryReportsWorker},
        },
    }, nil
}

func (r *EventRouter) RouteMessage(from gen.PID, msg any) gen.Atom {
    switch msg.(type) {
    case MessagePaymentEvent:
        return "payments"
    case MessageShipmentEvent:
        return "shipments"
    case MessageReportEvent:
        return "reports"
    }
    return act.RouteDiscard
}

func (r *EventRouter) RouteCall(from gen.PID, ref gen.Ref, req any) gen.Atom {
    return act.RouteDiscard // no sync routing here
}
```

## Sharded Workers with Capacity (Router + Pool)

For key affinity with capacity per shard, make each shard an `act.Pool` registered under the shard name, keep the Router free, and let a single supervisor own both as siblings. The Router hashes the key to a shard name, resolves it through the registry to the pool, and the pool round-robins to a worker. Router and Pool compose by name through the registry, never by nesting.

```go
const ShardCount = 16

type ShardRouter struct {
    act.Router
    shards []gen.Atom
}

type ShardedMessage interface {
    ShardKey() uint64
}

func (r *ShardRouter) Init(args ...any) (act.RouterOptions, error) {
    r.shards = make([]gen.Atom, ShardCount)
    for i := range r.shards {
        r.shards[i] = gen.Atom(fmt.Sprintf("shard:%d", i))
    }
    return act.RouterOptions{}, nil // free router; shards live in the registry
}

func (r *ShardRouter) RouteMessage(from gen.PID, msg any) gen.Atom {
    keyed, ok := msg.(ShardedMessage)
    if ok == false {
        return act.RouteDiscard
    }
    return r.shards[keyed.ShardKey()%uint64(len(r.shards))]
}

func (r *ShardRouter) RouteCall(from gen.PID, ref gen.Ref, req any) gen.Atom {
    return act.RouteDiscard
}
```

## When to Use Router

**Use a Router when** routing depends on message content, you need key affinity for stateful work, you want named routes that survive restarts, you want one entry point dispatching to processes owned elsewhere, or async commands and sync queries route differently (CQRS), or you want to add/remove/disable/replace routes at runtime.

**Do not use a Router when** you need plain round-robin across identical workers (`act.Pool` is simpler and faster), when you need supervision policy (supervise externally, route by registered name), or when senders can already address workers directly.

**Pitfalls.** Derive shard names from a fixed table - adding routes at runtime changes N and breaks affinity. `RouteMessage` cannot tell owned from registry routes (it just returns a name); use `Route(name)` to check ownership. Pending operations are observable via `Route(name).Pending` - a just-disabled route reads `Pending: RoutePendingDisable` before it reads `Disabled: true`. Routers do not preserve mailboxes across worker restarts. `gen.Atom` addresses local names only - do remote forwarding explicitly from the admin path.

---

# WebWorker

`act.WebWorker` is a specialized actor for HTTP requests delivered as `meta.MessageWebRequest` from the Web meta process (`meta.CreateWebHandler`, see `meta.md`). Instead of receiving the raw message, checking the method, and remembering to call `Done()`, you implement per-method callbacks; WebWorker detects the request, dispatches by method, and calls `Done()` for you - even if your callback returns an error or panics. Prefer this over hand-handling `meta.MessageWebRequest` in a plain actor.

## WebWorkerBehavior

```go
type WebWorkerBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) error

    // HTTP method callbacks (from, writer, request) error
    HandleGet(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandlePost(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandlePut(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandlePatch(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandleDelete(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandleHead(from gen.PID, writer http.ResponseWriter, request *http.Request) error
    HandleOptions(from gen.PID, writer http.ResponseWriter, request *http.Request) error

    // Standard actor callbacks
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
    Terminate(reason error)
}
```

Every callback has a default, so implement only what you need. Unimplemented HTTP methods log a warning and respond **501 Not Implemented**; an unknown method responds 501 as well. Non-HTTP messages (config updates, control signals) reach `HandleMessage` / `HandleCall` normally while HTTP requests are handled specially.

## Callback Contract

`from` is the PID of the meta process that delivered the request. Write your response to `writer`; WebWorker calls `Done()` after the callback returns. Return `nil` to keep serving. Return a non-nil error to **terminate the worker** - reserve that for fatal conditions (lost DB connection, unrecoverable state). For per-request problems (validation, not found, conflict), write the appropriate status to `writer` and return `nil`.

```go
type APIWorker struct {
    act.WebWorker
}

func (w *APIWorker) HandleGet(from gen.PID, writer http.ResponseWriter, request *http.Request) error {
    user := w.lookup(request.URL.Query().Get("id"))
    json.NewEncoder(writer).Encode(user)
    return nil
}

func (w *APIWorker) HandlePost(from gen.PID, writer http.ResponseWriter, request *http.Request) error {
    var data CreateRequest
    if err := json.NewDecoder(request.Body).Decode(&data); err != nil {
        http.Error(writer, "invalid JSON", http.StatusBadRequest) // per-request error
        return nil
    }
    writer.WriteHeader(http.StatusCreated)
    json.NewEncoder(writer).Encode(w.create(data))
    return nil
}
```

## Concurrency via Pool

A single WebWorker serves one request at a time. For concurrency, front it with an `act.Pool` of WebWorkers registered under the handler's target name:

```go
type APIWorkerPool struct {
    act.Pool
}

func (p *APIWorkerPool) Init(args ...any) (act.PoolOptions, error) {
    return act.PoolOptions{
        PoolSize:          10,
        WorkerMailboxSize: 20,
        WorkerFactory:     func() gen.ProcessBehavior { return &APIWorker{} },
    }, nil
}
```

The Web handler sends each `meta.MessageWebRequest` to the pool, which forwards it to a free worker - up to `PoolSize` concurrent requests, with `PoolSize * WorkerMailboxSize` in flight before backpressure. See the Pool section above and `meta.md` for the Web meta process setup.

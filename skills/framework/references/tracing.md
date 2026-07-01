# Distributed Tracing

Tracing follows a chain of causally related messages across processes and nodes. When a process sends a message and the framework decides to track it, a 128-bit trace identity is attached to that message. Every message the recipient sends while handling it inherits the same identity, and so on down the chain. The trace ends when the last handler finishes without sending more traced messages - there is no explicit end and no timeout.

You configure tracing only on entry-point processes. The trace propagates through the whole downstream chain automatically, so a process that never opted in still participates in any trace that reaches it.

## Two Layers

A trace has two layers, and keeping them distinct is the key to using tracing well.

- **Message flow** - which messages travelled between which processes and how long each hop took. The framework records this on its own, with no code in your handlers. This is the skeleton of the trace.
- **Business operations** - what each handler actually did while holding a message (validate an order, reserve stock). The framework cannot know these; you mark them by hand with business spans. They appear as named intervals nested inside the message flow.

The message flow tells you a message named `ProcessOrder` was handled in 50ms. Business spans tell you what those 50ms did.

## Observation Points

As a trace flows, the framework records observations at up to three points per message. These are `gen.TracingPoint` values.

| Point | Constant | Recorded when |
|-------|----------|---------------|
| Sent | `gen.TracingPointSent` | Message leaves the sender |
| Delivered | `gen.TracingPointDelivered` | Message enters the recipient's mailbox (not yet processed) |
| Processed | `gen.TracingPointProcessed` | Recipient's handler returns (captures error, if any) |
| Span | `gen.TracingPointSpan` | A business span opened with `StartTracingSpan` |

The timing gaps are the diagnostic value: Sent to Delivered is network latency (remote) or scheduling delay (local); Delivered to Processed is mailbox wait plus handler execution.

Each traced message kind (`gen.TracingKind`) produces a specific subset of points:

| Kind | Constant | Observations |
|------|----------|--------------|
| Send | `gen.TracingKindSend` | Sent, Delivered, Processed |
| Request (`Call`) | `gen.TracingKindRequest` | Sent, Delivered, Processed |
| Response | `gen.TracingKindResponse` | Sent, Delivered |
| Spawn | `gen.TracingKindSpawn` | Sent, Processed |
| Terminate | `gen.TracingKindTerminate` | Processed |

### What does not get traced

Exit signals (`SendExit`), events (`SendEvent`), and delayed messages (`SendAfter`) do not carry trace context. Events would create an observation storm on fan-out; a delayed message is a scheduled future action, not a continuation of the current chain, so decoupling it prevents periodic self-ticks from creating an endless trace. A regular `Send` to self does carry the trace.

## Samplers

By default no process starts a trace. A sampler decides, per outgoing message, whether to start a new trace. Crucially, **the sampler is only consulted when there is no active propagating trace** - a process already handling a traced message inherits it regardless of its sampler. Set a sampler on one entry-point process and the trace follows the entire chain.

The sampler type is `gen.TracingSampler`:

```go
type TracingSampler interface {
    Sample() bool     // true = start a new trace for this outgoing message
    String() string   // human-readable name shown in Observer and inspection
}
```

Four built-in samplers:

| Sampler | Behavior |
|---------|----------|
| `gen.TracingSamplerDisable` | Never starts a trace. The default for every process and node. |
| `gen.TracingSamplerAlways` | Starts a trace for every outgoing message. |
| `gen.TracingSamplerRatio(rate float64)` | Traces the given fraction. `rate >= 1` returns `TracingSamplerAlways`; `rate <= 0` returns `TracingSamplerDisable`. |
| `gen.TracingSamplerRateLimit(perSecond int)` | At most `perSecond` new traces per second. `perSecond <= 0` returns `TracingSamplerDisable`. |

`Disable` and `Always` are package-level variables; `Ratio` and `RateLimit` are constructors. Implement `gen.TracingSampler` yourself for custom logic.

### Setting the sampler

A process sets its own sampler in `Init`. A node has an independent sampler that governs messages sent via `node.Send()` / `node.Call()`.

```go
func (a *gateway) Init(args ...any) error {
    if err := a.SetTracingSampler(gen.TracingSamplerAlways); err != nil {
        return err
    }
    a.SetTracingAttribute("service", "gateway")
    return nil
}
```

The sampler becomes active from the first `HandleMessage` / `HandleCall`; messages sent during `Init` itself are not traced.

| Method | Where | Notes |
|--------|-------|-------|
| `Process.SetTracingSampler(sampler) error` | process | Available in Init and Running. |
| `Process.TracingSampler() TracingSampler` | process | Returns `TracingSamplerDisable` if unset. |
| `Node.SetTracingSampler(sampler) error` | node | Governs `node.Send`/`node.Call`. Running only. |
| `Node.TracingSampler() TracingSampler` | node | Current node sampler. |
| `Node.SetProcessTracingSampler(pid, sampler) error` | node | Change a running process's sampler without restarting it. Running only. |

`SetProcessTracingSampler` is how you enable full tracing on a misbehaving process at runtime (also exposed in the Observer UI), then set it back to `TracingSamplerDisable` when done.

## Attributes

Attributes add business context to observations, making traces searchable. They are part of the observation record, not the trace context: only the trace ID and span ID travel over the network, so attributes stay local to the node that emitted the observation.

An attribute is a `gen.TracingAttribute{Key, Value string}`. There are two lifetimes.

**Permanent** - set on a process (or node), attached to every observation where that process participates, for its whole lifetime.

```go
a.SetTracingAttribute("service", "payment")
a.SetTracingAttribute("version", "2.1")
a.RemoveTracingAttribute("version")   // remove one
```

**One-shot** - set inside a handler, scoped to that single handler invocation and cleared automatically when the handler returns. A one-shot value with the same key as a permanent one takes priority for that invocation without modifying the permanent attribute.

```go
func (a *processor) HandleMessage(from gen.PID, message any) error {
    order := message.(Order)
    a.SetTracingSpanAttribute("order_id", order.ID)   // one-shot
    a.Send(warehousePID, ReserveStock{OrderID: order.ID})
    return nil
}
```

| Method | Lifetime | Target |
|--------|----------|--------|
| `Process.SetTracingAttribute(key, value)` | permanent | this process, every span |
| `Process.RemoveTracingAttribute(key)` | permanent | this process |
| `Process.SetTracingSpanAttribute(key, value)` | one-shot | current handler only |
| `Process.TracingAttributes() []TracingAttribute` | - | merged permanent + one-shot |
| `Node.SetTracingAttribute(key, value)` | permanent | node, independent of process attrs |
| `Node.RemoveTracingAttribute(key)` | permanent | node |

Which observation carries which attributes:

| Observation | Attributes |
|-------------|-----------|
| Sent | sender's permanent + one-shot |
| Delivered | receiver's permanent |
| Processed | receiver's permanent + one-shot |

The `ergo.` prefix is reserved for framework-generated attributes (`ergo.node`, `ergo.from`, `ergo.behavior`). Attempts to set an attribute with this prefix are silently ignored - for both `SetTracing*Attribute` and span attributes.

## Business Spans

Business spans open the opaque handler block. A span is a named interval you start and end yourself around a unit of domain work. It appears in the trace as a child of the handler that opened it, with its own name and start/end time.

`Process.StartTracingSpan(name) gen.TracingSpanScope` opens one. The scope has three methods, and you must close it:

```go
type TracingSpanScope interface {
    SetAttribute(key, value string) // "ergo." prefix reserved (ignored)
    End()                           // close successfully
    EndError(err error)             // close with an error, marked in the waterfall
}
```

The idiomatic pattern closes on the work boundary:

```go
func (p *processor) HandleMessage(from gen.PID, message any) error {
    order := message.(ProcessOrder)

    span := p.StartTracingSpan("validate-order")
    err := p.validate(order)
    if err != nil {
        span.EndError(err)
        return err
    }
    span.End()
    // ... continue
    return nil
}
```

Use only within the current handler goroutine. Name spans for domain meaning (`reserve-inventory`, not `step-2`) - the name is what appears in the waterfall and what people search for.

### Spans contain their sends

Any message a handler sends while a span is open becomes a child of that span: the outgoing Sent observation and the entire downstream chain it triggers nest underneath. This is what makes spans more than timers.

```go
reserve := p.StartTracingSpan("reserve-inventory")
p.Send(warehousePID, ReserveStock{OrderID: order.ID})
reserve.End()
```

### Nesting

Spans nest. Open a child span while a parent is still open and it becomes a subtree. Close in reverse order (the `defer` idiom does this naturally):

```
ProcessOrder (Processed, 50ms)
└─ fulfill-order
   ├─ reserve-inventory
   │  └─ ReserveStock  (Sent -> Delivered -> Processed at warehouse)
   └─ create-invoice
      └─ CreateInvoice (Sent -> Delivered -> Processed at billing)
```

### Annotate or initiate

A span behaves differently depending on whether the handler is already in a trace, and the same sampler that governs message tracing governs this.

- **Already in a trace** (the handler is processing a message that carried one): the span attaches as a child - passive annotation. This works with or without a sampler. Instrument handlers everywhere and they light up inside whatever traces flow through. This is the common case.
- **No active trace**: the span consults the process's sampler. With a sampler that samples, the span starts a new trace and becomes its root (initiator). With no sampler, `StartTracingSpan` returns `gen.TracingSpanScopeNoop` - all three methods are no-ops. An instrumented process never forces tracing into existence on its own.

So the decision "do I start traces?" lives in one place, the sampler, and applies uniformly to outgoing messages and business spans.

Keep `SetTracingSpanAttribute` and `span.SetAttribute` straight: the first attaches to the handler's own observations, the second attaches to the business span you opened.

## Exporters

Observations go nowhere by themselves. Register one or more tracing exporters on the node. Like loggers, a node can have many exporters, each receiving observations according to its own flags.

### Flags

`gen.TracingFlags` is a bitmask selecting which observations an exporter receives. Combine with bitwise OR.

| Flag | Delivers |
|------|----------|
| `gen.TracingFlagSend` | Sent observations (send / call / response) |
| `gen.TracingFlagReceive` | Delivered and Processed observations, **and business spans** |
| `gen.TracingFlagProcs` | Spawn and Terminate lifecycle events |
| `gen.TracingFlagInherit` | Children inherit tracing from the parent |

Note that business spans arrive under `TracingFlagReceive` - an exporter that wants to see business operations must set that flag.

### Two kinds of exporter

**Behavior-based** - a plain implementation of `gen.TracingBehavior`. Use for lightweight exporters that need no actor capabilities.

```go
type TracingBehavior interface {
    HandleSpan(TracingSpan)
    Terminate()
}
```

`HandleSpan` is called **synchronously** during the exporter iteration, so it blocks delivery to the next exporter in the chain. Keep it fast and never block in it.

```go
type traceCounter struct{ sends int64 }

func (tc *traceCounter) HandleSpan(span gen.TracingSpan) {
    if span.Kind == gen.TracingKindSend {
        atomic.AddInt64(&tc.sends, 1)
    }
}
func (tc *traceCounter) Terminate() {}

node.TracingExporterAdd("counter", &traceCounter{}, gen.TracingFlagSend)
```

**Process-based** - an actor whose mailbox receives `gen.TracingSpan` messages, handled in its `HandleSpan(gen.TracingSpan) error` callback. Use when the exporter needs batching timers, network sends, or other actor capabilities (this is how Pulse and Observer work internally). The callback only fires if the process is registered as an exporter via `TracingExporterAddPID`; the default `act.Actor.HandleSpan` just logs a warning for an unhandled span. **If the actor's mailbox is full, spans are silently dropped** (no error, no log), so the exporter must keep up with the observation rate.

```go
node.TracingExporterAddPID(pid, "observer",
    gen.TracingFlagSend|gen.TracingFlagReceive|gen.TracingFlagProcs)
```

### Registration API (node)

| Method | Purpose |
|--------|---------|
| `TracingExporterAdd(name, exporter, flags) error` | Register a `TracingBehavior`. `ErrTaken` if name exists. Running only. |
| `TracingExporterAddPID(pid, name, flags) error` | Register a process as exporter. `ErrTaken` if name exists. Running only. |
| `TracingExporterDelete(name)` | Remove by name; calls the behavior's `Terminate()`. |
| `TracingExporterDeletePID(pid)` | Remove a process-based exporter. |
| `TracingExporters() []string` | List registered exporter names. |
| `TracingExporterFlags(name) TracingFlags` | Flags for one exporter. |

Exporters can be added and removed at any time while the node runs.

### Registering at startup

`gen.TracingOptions` in `NodeOptions` registers exporters before the node reaches the running state. It has one field, `Exporters []gen.TracingExporter`, each entry a `{Name string, Exporter TracingBehavior, Flags TracingFlags}`.

```go
node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Tracing: gen.TracingOptions{
        Exporters: []gen.TracingExporter{
            {Name: "counter", Exporter: &traceCounter{}, Flags: gen.TracingFlagSend},
        },
    },
})
```

An empty `TracingOptions{}` registers nothing - it does not "enable tracing" on its own. Whether traces *start* is decided by samplers (`SetTracingSampler` / `SetProcessTracingSampler`), never by `TracingOptions`.

## Cross-Node Propagation

Sent is recorded on the sending node; Delivered and Processed on the receiving node. The framework preserves the trace identity across the network so observations emitted on different nodes correlate into one trace. This requires `EnableTracing: true` in `gen.NetworkFlags` on the participating nodes - without it, trace context does not cross the connection.

```go
Network: gen.NetworkOptions{
    Flags: gen.NetworkFlags{Enable: true, EnableTracing: true},
},
```

## Ready-Made Exporters

You rarely write an exporter from scratch. Two are supplied:

- **pulse** (`ergo.services/application/pulse`) - exports observations to an OTLP/HTTP backend (Grafana Tempo, Jaeger). Each node runs its own pulse instance and the backend assembles full cross-cluster traces. Pulse registers itself via `TracingExporterAddPID` at startup, so it needs no `TracingOptions` entry. See `applications-lib.md`.
- **observer** (`ergo.services/application/observer`) - real-time trace visualization in the web UI for the node it is attached to, plus runtime sampler control.

The custom-exporter path in this document is for needs those two do not cover.

## Anti-Patterns

- **Do not expect tracing without a sampler** - registering an exporter and setting `EnableTracing` produces nothing until some process or node has a non-`Disable` sampler. Traces are born at the sampler.
- **Do not block inside a behavior-based `HandleSpan`** - it runs synchronously and stalls every later exporter. Keep it to a counter increment or a non-blocking enqueue.
- **Do not let a process-based exporter fall behind** - a full mailbox drops spans silently. Size the mailbox and drain fast, or use pulse.
- **Do not set a sampler in `Init` and expect setup-phase sends to be traced** - the sampler only activates from the first `HandleMessage` / `HandleCall`. `Init` sends (including `SendAfter` ticks) are untraced.
- **Do not rely on `SendAfter` to continue a trace** - delayed messages start fresh. Wrap the cycle's work in a business span (with a sampler set) to make each tick one coherent trace.
- **Do not use the `ergo.` attribute prefix** - it is reserved and silently ignored.
- **Do not forget `EnableTracing` on both nodes** - traces will not cross the connection without it.

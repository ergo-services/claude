# Actors

Actor is the fundamental unit of computation in Ergo: a process with private state, a mailbox, and sequential message-handling callbacks. Implemented as `act.Actor` (ProcessBehavior).

## Behavior Interface

`act.Actor` embeds `gen.Process` and provides default no-op implementations. Override callbacks as needed.

| Callback | Signature | Purpose |
|----------|-----------|---------|
| `Init` | `Init(args ...any) error` | Setup; return error to abort spawn |
| `HandleMessage` | `HandleMessage(from gen.PID, message any) error` | Async message handler |
| `HandleCall` | `HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)` | Sync request handler; return response |
| `Terminate` | `Terminate(reason error)` | Cleanup on shutdown |
| `HandleEvent` | `HandleEvent(message gen.MessageEvent) error` | Subscribed event |
| `HandleLog` | `HandleLog(message gen.MessageLog) error` | Log message (if process is a logger) |
| `HandleInspect` | `HandleInspect(from gen.PID, item ...string) map[string]string` | Custom introspection data |
| `HandleSpan` | `HandleSpan(message gen.TracingSpan) error` | Tracing span |

**Split-handle callbacks** (enabled via `SetSplitHandle(true)`): dispatch by the address the sender used. A process has at most one registered name but can own any number of aliases, so this is useful when you want different handling depending on whether a message arrived via the registered name, via one of the aliases (and which one), or directly by PID (falls through to `HandleMessage`).

| Callback | Purpose |
|----------|---------|
| `HandleMessageName(name gen.Atom, from gen.PID, message any) error` | Message addressed via the process's registered name |
| `HandleMessageAlias(alias gen.Alias, from gen.PID, message any) error` | Message addressed via one of the process's aliases |
| `HandleCallName(name gen.Atom, from gen.PID, ref gen.Ref, request any) (any, error)` | Call addressed via registered name |
| `HandleCallAlias(alias gen.Alias, from gen.PID, ref gen.Ref, request any) (any, error)` | Call addressed via alias |

## Process Lifecycle States

| State | Value | Meaning |
|-------|-------|---------|
| `ProcessStateInit` | 1 | Running `Init()`, not yet registered |
| `ProcessStateSleep` | 2 | Idle, mailbox empty |
| `ProcessStateRunning` | 4 | Dispatching a message |
| `ProcessStateWaitResponse` | 8 | Blocked in `Call()` awaiting reply |
| `ProcessStateTerminated` | 16 | Shutting down |
| `ProcessStateZombee` | 32 | Killed forcibly, goroutine orphaned |

## Mailbox Priority Queues

Messages are dispatched in priority order:

| Priority | Value | Queue | Default Use |
|----------|-------|-------|-------------|
| `MessagePriorityMax` | 2 | Urgent | System control, kill signals |
| `MessagePriorityHigh` | 1 | System | Framework-internal messages |
| `MessagePriorityNormal` | 0 | Main | Application messages (default) |
| - | - | Log | Log messages |

## Trap Exit

By default, an exit signal from a linked target terminates this actor. With `SetTrapExit(true)`, exit signals from **other** processes are converted into `gen.MessageExit*` messages delivered to the mailbox, so the actor can inspect the reason and decide how to react.

One exception: an exit signal from the **parent** process is never trapped. The actor always terminates when its parent exits, regardless of `SetTrapExit` and regardless of the reason (the reason is not consulted). Only exit signals from non-parent targets are converted into mailbox messages.

```go
func (a *Supervisor) Init(args ...any) error {
    a.SetTrapExit(true)
    return nil
}

func (a *Supervisor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case gen.MessageExitPID:
        a.Log().Warning("linked process %v died: %v", m.PID, m.Reason)
    }
    return nil
}
```

## Spawning Child Processes

```go
// Anonymous child
pid, err := a.Spawn(createWorker, gen.ProcessOptions{})

// Named child (registered)
pid, err := a.SpawnRegister("worker-1", createWorker, gen.ProcessOptions{})

// Meta process (blocking I/O, returns Alias not PID)
alias, err := a.SpawnMeta(metaBehavior, gen.MetaOptions{})
```

### ProcessOptions

Configure the new process at spawn time. All fields are optional.

| Field | Type | Meaning |
|-------|------|---------|
| `MailboxSize` | `int64` | Max queued messages. `0` (default) = unlimited. When full, senders get `gen.ErrProcessMailboxFull` unless a `Fallback` is set |
| `LinkParent` | `bool` | Link the child to its parent: if the **parent** terminates, the child receives an exit signal and terminates. Ignored when spawned by the node (no parent process) |
| `LinkChild` | `bool` | Link the parent to the child: if the **child** terminates, the parent receives an exit signal and terminates. Ignored when spawned by the node |
| `LogLevel` | `gen.LogLevel` | Initial log level for the process. Default `gen.LogLevelInfo`; change later via `Log().SetLevel(...)` |
| `InitTimeout` | `int` | Max seconds for `Init()` to complete. `0` uses the default 5s; exceeding it terminates the process with `gen.ErrTimeout`. Remote spawn and application processes allow up to 15s |
| `PreserveMailbox` | `bool` | On abnormal termination (panic, abnormal return, kill), capture the mailbox so a supervisor can hand it to the restarted incarnation. Normal and shutdown exits never capture |
| `Env` | `map[gen.Env]any` | Process-specific environment, readable via `Env()`. Overrides parent, leader, and node values |
| `Leader` | `gen.PID` | Group leader PID. Defaults to inherited from parent |
| `Fallback` | `gen.ProcessFallback` | Where overflow goes when `MailboxSize` is full. See `messages.md` |
| `SendPriority` | `gen.MessagePriority` | Default priority for this process's `Send`. See `messages.md` |
| `ImportantDelivery` | `bool` | Confirmed cross-node delivery for every message. See `messages.md` |
| `Compression` | `gen.Compression` | Per-process outgoing compression. See `messages.md` |

`LinkParent` and `LinkChild` are two independent one-directional links - set both for the bidirectional Erlang-style coupling. The mailbox and delivery knobs (`MailboxSize`, `Fallback`, `SendPriority`, `ImportantDelivery`, `Compression`) are covered in `messages.md`.

### Remote Spawn

Spawning on another node is not automatic - the **target node** must explicitly allow it (security default-deny):

1. Enable the network flag: `NetworkFlags.EnableRemoteSpawn = true` in the target node's `NetworkOptions`.
2. Register the factory on the target node: `node.Network().EnableSpawn(name, factory, allowedNodes...)`. The `name` is the permission token callers use. `allowedNodes` is an ACL - empty means any node.

Only then callers can use `RemoteSpawn`:

```go
// From inside a process - inherits application, log level, env from caller
pid, err := a.RemoteSpawn("other@host", "worker", gen.ProcessOptions{}, arg1, arg2)

// With registration on the remote node
pid, err := a.RemoteSpawnRegister("other@host", "worker", "worker-001", gen.ProcessOptions{})

// Node-level alternative - no application inheritance
remote, err := node.Network().GetNode("other@host")
pid, err := remote.Spawn("worker", gen.ProcessOptions{}, arg1, arg2)
pid, err := remote.SpawnRegister("worker-001", "worker", gen.ProcessOptions{})
```

See `references/node.md` for `EnableSpawn` / `DisableSpawn` API.

## Links and Monitors

Both mechanisms notify you when a target terminates. They differ in the consequence and in direction.

**Link** - a single **directed** relation, not symmetric like Erlang. `a.Link(target)` means: when `target` terminates, this process receives an exit signal (and terminates too, unless it traps exit). The reverse is not true - if this process terminates, `target` is unaffected. For bidirectional coupling, both sides link to each other, or use the `LinkParent` / `LinkChild` spawn options (each is one direction).

**Monitor** - a directed observation with no lifecycle coupling: this process receives a `gen.MessageDown*` message on target termination and keeps running.

```go
err := a.Link(targetPID)      // or target ProcessID, Alias, Event, node Atom
err := a.Unlink(targetPID)

err := a.Monitor(targetPID)
err := a.Demonitor(targetPID)
```

Notifications arriving in `HandleMessage`. `MessageDown*` (monitors) are always delivered; `MessageExit*` (links) are delivered only when the process has `SetTrapExit(true)` - otherwise the exit signal terminates the process silently.

| Message Type | When |
|--------------|------|
| `gen.MessageDownPID{PID, Reason}` | Monitored process terminated |
| `gen.MessageDownProcessID{ProcessID, Reason}` | Monitored named process terminated |
| `gen.MessageDownAlias{Alias, Reason}` | Monitored alias released |
| `gen.MessageDownEvent{Event, Reason}` | Monitored event unregistered |
| `gen.MessageDownNode{Name}` | Monitored node disconnected |
| `gen.MessageExitPID{PID, Reason}` | Linked process died (trap exit) |
| `gen.MessageExitProcessID{ProcessID, Reason}` | Linked named process died (trap exit) |
| `gen.MessageExitAlias{Alias, Reason}` | Linked alias released (trap exit) |
| `gen.MessageExitEvent{Event, Reason}` | Linked event unregistered (trap exit) |
| `gen.MessageExitNode{Name}` | Linked node disconnected (trap exit) |

## Aliases

An alias is an additional temporary identifier for a process. A process has at most one registered name, but can create any number of aliases. Aliases are addressable like PIDs or registered names (via `Send`, `Call`, `Link`, `Monitor`) and survive only until the owning process deletes them or terminates.

Typical uses:
- Exposing multiple logical endpoints of one actor (e.g., separate aliases for "admin" and "client" facing channels), then dispatching via `HandleMessageAlias` / `HandleCallAlias` with `SetSplitHandle(true)`.
- Handing out a revocable address - delete the alias to cut off further communication without terminating the process.
- Meta processes (TCP connections, WebSocket sessions, etc.): each meta process's identifier is a `gen.Alias`, not a PID. See `meta.md`.

```go
alias, err := a.CreateAlias()
defer a.DeleteAlias(alias)

a.Send("collaborator", RegisterEndpoint{Addr: alias})
// Anyone with 'alias' can Send/Call this process.

aliases := a.Aliases()  // all live aliases for this process
```

Request/reply does not require aliases. Use `Call` / `HandleCall` - the framework generates a `gen.Ref` to correlate request and response automatically; the handler receives it as the `ref` parameter of `HandleCall(from, ref, request)`. Two return conventions matter: returning `(nil, nil)` defers the reply so the handler can later call `SendResponse(from, ref, msg)` (for example once an async event arrives), and returning `(result, gen.TerminateReasonNormal)` with a non-nil `result` sends the response and then terminates the actor normally. See `messages.md`.

## Events (Pub/Sub)

Events are named pub/sub channels identified by `{Name, Node}`. The name must be unique per node - the same name can exist on different nodes as independent events.

### Ownership vs Publishing

`RegisterEvent` makes the caller the **owner** and returns a `gen.Ref` token. The owner controls the event's lifecycle (unregister, notifications). By default publishing requires the token, not ownership - the owner can hand the token to other processes and any token-holder may publish via `SendEvent(name, token, msg)`. When the token check is on, publishing with a wrong or unknown token fails.

Setting `EventOptions.Open = true` disables the token check on `SendEvent`: any local process may then publish to the event by name regardless of the token value. The token is still returned but is no longer required to publish. `UnregisterEvent` is unaffected - only the registering process (or the node, for node-level events) can unregister. Use `Open` for a shared internal bus where the token ceremony protects nothing.

This separation enables useful patterns:
- Coordinator registers, worker pool publishes (multiple producers on one logical stream).
- Primary/backup rotation - backup holds the token and takes over publishing without re-registering.
- Open node-wide bus - many unrelated producers publish by name without distributing a token.

### Registering and Publishing

```go
// Owner (usually in Init)
token, err := a.RegisterEvent("price_update", gen.EventOptions{
    Notify: true,   // owner receives MessageEventStart/Stop on first/last subscriber
    Buffer: 10,     // keep last N events; late subscribers receive them on join
    Open:   false,  // true = any local process may publish by name (no token check)
})
defer a.UnregisterEvent("price_update")

// Publish (by anyone holding the token)
a.SendEvent("price_update", token, PriceUpdate{Symbol: "BTC", Price: 42000})
```

### Owner Notifications (Notify: true)

The owner gets these messages in `HandleMessage`:

| Message | Sent when |
|---------|-----------|
| `gen.MessageEventStart{Name}` | First subscriber appears |
| `gen.MessageEventStop{Name}` | Last subscriber leaves |

Use these to start/stop expensive data production on demand.

### Subscribing

Subscribers use `LinkEvent` or `MonitorEvent`. Both return buffered events on successful subscription.

```go
// Link - subscriber terminates if event owner terminates (unless trap exit)
lastEvents, err := a.LinkEvent(gen.Event{Name: "price_update", Node: "feed@host"})

// Monitor - subscriber receives MessageDownEvent on owner termination, no cascade
lastEvents, err := a.MonitorEvent(gen.Event{Name: "price_update", Node: "feed@host"})

for _, e := range lastEvents {
    handle(e)  // catch-up from buffer
}
```

For a local event, omit `Node` - the framework fills in the local node name.

### Delivery

New events arrive in `HandleEvent`:

```go
func (a *MyActor) HandleEvent(m gen.MessageEvent) error {
    // m.Event is the event identifier {Name, Node}
    // m.Timestamp is publish time (ns since epoch)
    switch payload := m.Message.(type) {
    case PriceUpdate:
        a.process(payload)
    }
    return nil
}
```

### Lifecycle

When the owner terminates or calls `UnregisterEvent`, all subscribers receive termination (`MessageExitEvent` for links, `MessageDownEvent` for monitors). The name becomes available for re-registration.

If a subscriber's node loses connection, that subscriber receives a termination with reason `gen.ErrNoConnection`.

### Network Scaling

Event distribution scales with **subscribing nodes**, not subscribers. Publishing to 1M subscribers spread across 10 nodes sends 10 network messages; the remote nodes fan out to local subscribers.

### Built-in CoreEvent Bus

Every node auto-registers one event for you at startup, named `gen.CoreEvent` (the atom `"core"`), owned by the node core. You never register or publish to it. It is always available - no registrar and no networking required - and buffered (last 1000 events). The node publishes the facts of its own lifecycle here so any process can react without polling.

Subscribe the same way you subscribe to any event, then type-switch on `m.Message` in `HandleEvent`:

```go
func (a *Watcher) Init(args ...any) error {
    // omit Node for the local bus; name a peer to watch its lifecycle remotely
    _, err := a.MonitorEvent(gen.Event{Name: gen.CoreEvent})
    return err
}

func (a *Watcher) HandleEvent(m gen.MessageEvent) error {
    switch e := m.Message.(type) {
    case gen.MessageCoreApplicationStarted:
        a.Log().Info("app %s started (%s)", e.Name, e.Mode)
    case gen.MessageCoreApplicationStopped:
        a.Log().Warning("app %s stopped: %v", e.Name, e.Reason)
    case gen.MessageCoreNodeConnected:
        a.Log().Info("node %s connected", e.Name)
    case gen.MessageCoreNodeDisconnected:
        a.Log().Warning("node %s disconnected: %v", e.Name, e.Reason)
    }
    return nil
}
```

The four message types carried on this bus:

| Message Type | Published when |
|--------------|----------------|
| `gen.MessageCoreApplicationStarted{Name, Mode}` | An application on this node reached the running state |
| `gen.MessageCoreApplicationStopped{Name, Mode, Reason}` | A running application on this node stopped. `Reason` carries the termination that stopped it (for a permanent application, the reason of the member whose termination stopped it) |
| `gen.MessageCoreNodeConnected{Name}` | A connection with a remote node was established |
| `gen.MessageCoreNodeDisconnected{Name, Reason}` | A connection with a remote node was lost |

Because the bus is buffered (1000 entries), subscribing from `Init()` returns the most recent transitions in the `LinkEvent` / `MonitorEvent` result slice. A process can therefore learn which applications are already running and which peers are already connected without racing the producer, and a restarted observer does not start blind.

Naming a peer node (`gen.Event{Name: gen.CoreEvent, Node: "worker1@host"}`) lets one observer watch the lifecycle of every node in the cluster from one place. This is the supported way for an actor to react to peers connecting or disconnecting (see `cluster.md`); it complements the disconnect-only `MonitorNode` / `LinkNode` by also delivering the connect notification and covering any peer without a per-node monitor. If a watched node becomes unreachable, the monitor fires with reason `gen.ErrNoConnection`.

The bus reports one node only: its own applications and its own peers. Cluster-wide facts (a node joining or leaving the cluster, application availability across the cluster) are reported separately by the registrar - see `cluster.md`.

## Registration

A process can have **at most one** registered name. `RegisterName` returns `gen.ErrTaken` if the process already has a name. To rename, call `UnregisterName` first, then `RegisterName` with the new one. Names are allowed in `Init` and `Running` states; other states return `gen.ErrNotAllowed`.

```go
err := a.RegisterName("my-worker")
err := a.UnregisterName()
```

Registered names are node-local. For cluster-wide addressing use `gen.ProcessID{Node: "other@host", Name: "my-worker"}` or discover via registrar (see `cluster.md`). For additional addresses without replacing the name, use aliases (below).

## Minimal Actor Snippet

```go
package main

import (
    "fmt"
    "ergo.services/ergo/act"
    "ergo.services/ergo/gen"
)

type Counter struct {
    act.Actor
    value int
}

func createCounter() gen.ProcessBehavior {
    return &Counter{}
}

type MessageIncrement struct{ By int }
type GetValueRequest struct{}
type GetValueResponse struct{ Value int }

func (a *Counter) Init(args ...any) error {
    a.Log().Info("counter started")
    return nil
}

func (a *Counter) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case MessageIncrement:
        a.value += m.By
    }
    return nil
}

func (a *Counter) HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) {
    switch request.(type) {
    case GetValueRequest:
        return GetValueResponse{Value: a.value}, nil
    }
    return nil, fmt.Errorf("unknown request: %T", request)
}

func (a *Counter) Terminate(reason error) {
    a.Log().Info("counter terminated: %v", reason)
}
```

## Anti-Patterns

- **Never block a callback for an unbounded time** - it runs on the mailbox dispatcher, so a deadline-less wait freezes the actor. Time-bounded I/O (a deadline'd HTTP/DB/socket call) is fine. A mutex or channel is a last resort and must be non-blocking or timeout-bounded (`TryLock`, `select` with timeout), never a naked `Lock()`, bare `<-ch`, or `time.Sleep`. A bounded mutex/channel still stalls the actor for its whole timeout and risks deadlock if the releaser waits on this actor - prefer lock-free structures (`sync.Map`, `atomic`) and messages.
- **Never** share mutable state between actors. Pass data by copy in messages.
- **Never** hold a pointer to another actor's internal struct.
- Use `any`, not `interface{}`.

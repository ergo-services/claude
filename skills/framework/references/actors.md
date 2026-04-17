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
| — | — | Log | Log messages |

## Trap Exit

By default, a linked process dying delivers an exit signal that terminates this actor. With `SetTrapExit(true)`, the signal becomes a `gen.MessageExitPID` message in the mailbox.

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

### Remote Spawn

Spawning on another node is not automatic — the **target node** must explicitly allow it (security default-deny):

1. Enable the network flag: `NetworkFlags.EnableRemoteSpawn = true` in the target node's `NetworkOptions`.
2. Register the factory on the target node: `node.Network().EnableSpawn(name, factory, allowedNodes...)`. The `name` is the permission token callers use. `allowedNodes` is an ACL — empty means any node.

Only then callers can use `RemoteSpawn`:

```go
// From inside a process — inherits application, log level, env from caller
pid, err := a.RemoteSpawn("other@host", "worker", gen.ProcessOptions{}, arg1, arg2)

// With registration on the remote node
pid, err := a.RemoteSpawnRegister("other@host", "worker", "worker-001", gen.ProcessOptions{})

// Node-level alternative — no application inheritance
remote, err := node.Network().GetNode("other@host")
pid, err := remote.Spawn("worker", gen.ProcessOptions{}, arg1, arg2)
pid, err := remote.SpawnRegister("worker-001", "worker", gen.ProcessOptions{})
```

See `references/node.md` for `EnableSpawn` / `DisableSpawn` API.

## Links and Monitors

**Link** — bidirectional; both processes die together (unless either traps exit).
**Monitor** — unidirectional; sender receives `MessageDownPID` on target termination.

```go
err := a.Link(targetPID)      // or target ProcessID, Alias, Event, node Atom
err := a.Unlink(targetPID)

err := a.Monitor(targetPID)
err := a.Demonitor(targetPID)
```

Notifications arriving in `HandleMessage`:

| Message Type | When |
|--------------|------|
| `gen.MessageDownPID{PID, Reason}` | Monitored process terminated |
| `gen.MessageDownProcessID{ProcessID, Reason}` | Monitored named process terminated |
| `gen.MessageDownAlias{Alias, Reason}` | Monitored alias released |
| `gen.MessageDownEvent{Event, Reason}` | Monitored event unregistered |
| `gen.MessageDownNode{Name}` | Monitored node disconnected |
| `gen.MessageExitPID{PID, Reason}` | Linked process died (with `SetTrapExit(true)`) |
| `gen.MessageExitNode{Name}` | Linked node disconnected |

## Aliases

An alias is an additional temporary identifier for a process. A process has at most one registered name, but can create any number of aliases. Aliases are addressable like PIDs or registered names (via `Send`, `Call`, `Link`, `Monitor`) and survive only until the owning process deletes them or terminates.

Typical uses:
- Exposing multiple logical endpoints of one actor (e.g., separate aliases for "admin" and "client" facing channels), then dispatching via `HandleMessageAlias` / `HandleCallAlias` with `SetSplitHandle(true)`.
- Handing out a revocable address — delete the alias to cut off further communication without terminating the process.
- Meta processes (TCP connections, WebSocket sessions, etc.): each meta process's identifier is a `gen.Alias`, not a PID. See `meta.md`.

```go
alias, err := a.CreateAlias()
defer a.DeleteAlias(alias)

a.Send("collaborator", RegisterEndpoint{Addr: alias})
// Anyone with 'alias' can Send/Call this process.

aliases := a.Aliases()  // all live aliases for this process
```

Request/reply does not require aliases. Use `Call` / `HandleCall` — the framework generates a `gen.Ref` to correlate request and response automatically; the handler receives it as the `ref` parameter of `HandleCall(from, ref, request)` and can pass it to `SendResponse(from, ref, msg)` for deferred replies. See `messages.md`.

## Events (Pub/Sub)

Events are named pub/sub channels identified by `{Name, Node}`. The name must be unique per node — the same name can exist on different nodes as independent events.

### Ownership vs Publishing

`RegisterEvent` makes the caller the **owner** and returns a `gen.Ref` token. The owner controls the event's lifecycle (unregister, notifications). Publishing requires the token, not ownership — the owner can hand the token to other processes and any token-holder may publish via `SendEvent(name, token, msg)`. Publishing with a wrong or unknown token fails.

This separation enables useful patterns:
- Coordinator registers, worker pool publishes (multiple producers on one logical stream).
- Primary/backup rotation — backup holds the token and takes over publishing without re-registering.

### Registering and Publishing

```go
// Owner (usually in Init)
token, err := a.RegisterEvent("price_update", gen.EventOptions{
    Notify: true,   // owner receives MessageEventStart/Stop on first/last subscriber
    Buffer: 10,     // keep last N events; late subscribers receive them on join
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
// Link — subscriber terminates if event owner terminates (unless trap exit)
lastEvents, err := a.LinkEvent(gen.Event{Name: "price_update", Node: "feed@host"})

// Monitor — subscriber receives MessageDownEvent on owner termination, no cascade
lastEvents, err := a.MonitorEvent(gen.Event{Name: "price_update", Node: "feed@host"})

for _, e := range lastEvents {
    handle(e)  // catch-up from buffer
}
```

For a local event, omit `Node` — the framework fills in the local node name.

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

- **Never** use mutex, channel receive/send, `time.Sleep`, or spawned goroutines inside a callback — they block the mailbox dispatcher.
- **Never** share mutable state between actors. Pass data by copy in messages.
- **Never** hold a pointer to another actor's internal struct.
- Use `any`, not `interface{}`.

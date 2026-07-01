# Errors and Termination Reasons

Errors in Ergo are not just return values from setup calls - they are also the **reason** a process carries when it dies. The same sentinel value shows up in three places a developer matches on:

- the `error` returned by an operation (`Call`, `Spawn`, `RegisterName`, `StartChild`, ...);
- the reason inside a Down/Exit notification (`gen.MessageDownPID.Reason`, `gen.MessageExitPID.Reason`, and the monitor/link variants);
- the reason passed to `ProcessTerminate(reason)` and returned from `HandleMessage` / `HandleCall`.

The framework defines a fixed set of sentinel errors (created with `errors.New`, so identity is stable) in `gen` and `act`. This page inventories them, describes the four termination reasons, and explains how to construct and match errors correctly - including across the network.

## Match with `errors.Is`, not `==`

Always compare with `errors.Is`:

```go
resp, err := p.Call(target, req)
if errors.Is(err, gen.ErrTimeout) {
    // ...
}
```

A framework-produced reason may be **wrapped** (`gen.Error` can hold a chain, see below), and a reason that arrived from another node was reconstructed from the wire, not handed to you as the original variable. `errors.Is` traverses the wrap chain and matches by identity; a bare `err == gen.ErrTimeout` misses both cases.

## Termination Reasons

Every process dies with a reason. Four framework reasons live in `gen` (defined in `gen/process.go`):

| Reason | Set when | Error logged |
|--------|----------|--------------|
| `gen.TerminateReasonNormal` | Process stopped cleanly - returned from a callback, or graceful stop | No |
| `gen.TerminateReasonShutdown` | Node is stopping (`node.Stop()`) and processes are torn down | No |
| `gen.TerminateReasonKill` | Forced kill (`node.Kill(pid)` or a kill signal) - no graceful shutdown | Yes |
| `gen.TerminateReasonPanic` | A panic in `ProcessInit`, `ProcessRun`, or a message handler; the framework recovers and terminates | Yes (with stack trace) |

**Logging semantics matter for noise control.** `Normal` and `Shutdown` are expected terminations and produce no error log. `Kill` and `Panic` are logged as errors, with `Panic` including a stack trace. If your logs are noisy with terminations, distinguish "I stopped a process on purpose" (return `TerminateReasonNormal`) from an actual fault.

### Returning a reason from a callback

`HandleMessage` and `HandleCall` return an `error` that is interpreted as the termination reason:

```go
func (a *worker) HandleMessage(from gen.PID, message any) error {
    switch message.(type) {
    case MessageStop:
        return gen.TerminateReasonNormal // stop cleanly, no error log
    }
    return nil // keep running
}
```

- Return `nil` - the process keeps running.
- Return `gen.TerminateReasonNormal` - the process stops cleanly (no error log).
- Return any other error (including `gen.Errorf(...)`) - the process stops and the reason is that error; if it is not one of the reasons above it is logged like a fault.

All four reasons are registered with EDF, so they survive a network hop and remain matchable with `errors.Is` on the far node.

## Constructing Errors: `gen.Errorf` and `gen.Error`

`gen.Errorf` is the framework-sanctioned `fmt.Errorf`. Use it - not `fmt.Errorf` - whenever the error may become a termination reason or a `HandleCall` result that could cross a node boundary.

```go
err := gen.Errorf("processing order %d: %w", id, ErrInvalidOrder)

errors.Is(err, ErrInvalidOrder) // true - locally AND on a remote node
```

It mirrors `fmt.Errorf` exactly (same verbs, `%w` for wrapping, single or multiple), but stores the wrapped operands in a `*gen.Error` so the chain is preserved across EDF. Plain `fmt.Errorf` wrapping is flattened to a string over the network - the far node gets the text but `errors.Is` against your sentinel no longer holds. See `edf.md` for the wire-format detail and the `NetworkFlags.EnableWrappedErrors` requirement (on by default at both ends).

### The `gen.Error` type

`gen.Errorf` returns a `*gen.Error`. You rarely build it by hand, but you should know its shape:

| Field | Type | Purpose |
|-------|------|---------|
| `Msg` | `string` | Formatted message text, as `fmt.Errorf` would produce |
| `Wrapped` | `[]error` | Operands from `%w` (supports multiple), traversed by `errors.Is` / `errors.Unwrap` |
| `Mailbox` | `*gen.ProcessMailbox` | Captured mailbox of a panicked process, tagged `edf:"-"` |

`Error()` returns `Msg` (or the first wrapped error's text when `Msg` is empty). `Unwrap()` returns `Wrapped`, so standard-library traversal works.

The `Mailbox` field is how a panicked process's un-consumed messages survive a supervisor restart: the framework captures the mailbox into the reason and the supervisor replays it into the restarted process. It is tagged `edf:"-"` so it is never sent across the network - a replayed mailbox is a same-node concern only.

Do **not** register `*gen.Error` with `RegisterError`; register your `errors.New` sentinels and build chains at runtime with `gen.Errorf`. See `edf.md`.

## `gen` Sentinel Errors

All sentinels below are registered with EDF, so they cross the network with identity preserved (`errors.Is` keeps working on the far node). Grouped by area, from `gen/errors.go`.

### Process and naming

| Error | When you get this |
|-------|-------------------|
| `gen.ErrNameUnknown` | No process registered under the given name |
| `gen.ErrParentUnknown` | Parent / leader is not set for this process |
| `gen.ErrNodeTerminated` | The node has been stopped |
| `gen.ErrProcessUnknown` | Target process does not exist |
| `gen.ErrProcessMailboxFull` | Target mailbox is at capacity (see `ProcessOptions.MailboxSize`) |
| `gen.ErrProcessTerminated` | Target process has terminated |
| `gen.ErrProcessIncarnation` | PID belongs to a previous incarnation - the peer node restarted and this PID is stale (see below) |

### Meta processes

| Error | When you get this |
|-------|-------------------|
| `gen.ErrMetaUnknown` | No meta process registered under the given alias |
| `gen.ErrMetaMailboxFull` | Meta process mailbox is at capacity |

### Applications

| Error | When you get this |
|-------|-------------------|
| `gen.ErrApplicationUnknown` | No application registered under the given name |
| `gen.ErrApplicationDepends` | A declared dependency failed to start |
| `gen.ErrApplicationState` | Operation invalid because the application is running/stopping |
| `gen.ErrApplicationLoadPanic` | The application's `Load` callback panicked |
| `gen.ErrApplicationEmpty` | Application spec has no items |
| `gen.ErrApplicationName` | Application spec has no name |
| `gen.ErrApplicationStopping` | A stop is already in progress |
| `gen.ErrApplicationRunning` | Application is still running (e.g. cannot unload) |

### Target manager and registrar

The target manager is the pluggable event/publish backend; the registrar is the service-discovery client.

| Error | When you get this |
|-------|-------------------|
| `gen.ErrTargetUnknown` | Unknown target |
| `gen.ErrTargetExist` | Target already exists |
| `gen.ErrTargetManagerOverload` | Target manager queue is full |
| `gen.ErrRegistrarTerminated` | The registrar client has terminated |

### Aliases and events

| Error | When you get this |
|-------|-------------------|
| `gen.ErrAliasUnknown` | Unknown alias |
| `gen.ErrAliasOwner` | The calling process does not own this alias |
| `gen.ErrEventUnknown` | Unknown event |
| `gen.ErrEventOwner` | The calling process does not own this event |
| `gen.ErrTaken` | Resource (name/alias/event) is already taken |

### Network

| Error | When you get this |
|-------|-------------------|
| `gen.ErrNetworkStopped` | The network stack is stopped |
| `gen.ErrNoConnection` | Cannot reach the remote node - a link/monitor to a remote target delivers it as the reason in `gen.MessageExitPID` / `gen.MessageExitProcessID` (note: `gen.MessageExitNode` reports only the node `Name` and carries no reason) |
| `gen.ErrNoRoute` | No route to the target |

### Generic

These are reused across many APIs; read them together with the operation that returned them.

| Error | When you get this |
|-------|-------------------|
| `gen.ErrTimeout` | Operation did not complete in time (default `Call` timeout is 5s) |
| `gen.ErrUnsupported` | Operation not supported in this configuration |
| `gen.ErrUnknown` | Unclassified failure |
| `gen.ErrNotAllowed` | Operation not permitted in the current process state (e.g. a setter outside `Init` / `Running`) |
| `gen.ErrDiscarded` | Recipient discarded the message - a Router route chose to discard (see `pool.md` / Router) |
| `gen.ErrDisabled` | Target is disabled - a Router route is disabled or mid-disable |
| `gen.ErrBusy` | Target is busy - a Router route is mid-replace with no current worker |
| `gen.ErrExceeded` | A limit was exceeded - notably a supervisor's restart intensity, which terminates the supervisor with this reason |
| `gen.ErrIncorrect` | Incorrect value or argument |
| `gen.ErrMalformed` | Malformed value |
| `gen.ErrResponseIgnored` | A response was produced but the caller had already stopped waiting |
| `gen.ErrUnregistered` | A name / alias / event was unregistered while the process itself keeps running - a monitor/link fires with this reason without the target dying |
| `gen.ErrTaken` | (see Aliases and events) |
| `gen.ErrAtomTooLong` | Atom exceeds 255 bytes |
| `gen.ErrTooLarge` | Payload exceeds the connection's `MaxMessageSize` |
| `gen.ErrInternal` | Internal framework error |

`gen.ErrExceeded` is the supervisor restart-intensity-exceeded reason. There is no `gen.ErrSupervisorRestartsExceeded` - match `gen.ErrExceeded`.

## `act` Sentinel Errors

The `act` package defines errors for the dynamic supervisor, pool, and router management APIs (`act/errors.go`).

### Supervisor operations

Returned from `StartChild`, `AddChild`, `EnableChild`, `DisableChild`, and spec validation. Note that dynamic add/remove is only available on the SimpleOneForOne strategy - the static strategies (OneForOne, AllForOne, RestForOne) reject structural changes once started.

| Error | When you get this |
|-------|-------------------|
| `act.ErrSupervisorStrategyActive` | Tried to add/remove a child on a static strategy (OFO/AFO/RFO) after it started - its child set is fixed at start |
| `act.ErrSupervisorChildUnknown` | No child with the given name |
| `act.ErrSupervisorChildRunning` | Child is already running (e.g. `StartChild` on a live child) |
| `act.ErrSupervisorChildDisabled` | Child is disabled |
| `act.ErrSupervisorChildDuplicate` | A child spec with that name already exists |
| `act.ErrSupervisorInvalidSpec` | The child spec failed validation (wrapped with the specific cause via `%w`) |

`act.ErrSupervisorInvalidSpec` is wrapped, so match it with `errors.Is`.

### Pool

| Error | When you get this |
|-------|-------------------|
| `act.ErrPoolEmpty` | Routing to a pool that currently has no worker process |

### Router

| Error | When you get this |
|-------|-------------------|
| `act.ErrRouteRunning` | Route registration/respawn while that route's worker is still alive |
| `act.ErrRouteDuplicate` | A route with that name already exists |

A Router's per-message rejection reasons are the generic `gen.ErrDiscarded` / `gen.ErrDisabled` / `gen.ErrBusy` above, delivered to the caller as the failure reason.

## Network-Visible Reasons on Remote Failure

When a `Send` / `Call` fails to reach a remote target, the reason you get back is one of the network sentinels. `gen.ErrProcessIncarnation` is the one that surprises developers: it means the PID you held is from the peer node's **previous** incarnation. The peer restarted, and every PID minted before the restart is now stale. Re-resolve the target (by name, or via the registrar) and retry - the process may well be alive again under a fresh PID.

The complete Important-Delivery failure set is in `messages.md`; the reasons it draws from are all in the tables above.

## Registering Your Own Sentinels

Framework sentinels cross the wire automatically. Your application's own `errors.New` sentinels do not, unless you register them - see `edf.md` (Error Registration). An unregistered error still travels as a string, but registering it makes the payload compact and, combined with `gen.Errorf` wrapping, keeps `errors.Is` working across nodes.

## Anti-Patterns

- **Never compare reasons with `==`** - use `errors.Is`; framework reasons can be wrapped and are reconstructed after a network hop.
- **Never wrap a cross-boundary error with `fmt.Errorf`** - use `gen.Errorf` so the `%w` chain survives EDF; plain `fmt.Errorf` flattens to a string on the wire.
- **Never register `*gen.Error`** - register `errors.New` sentinels and wrap them at runtime with `gen.Errorf`.
- **Never treat a `Kill` / `Panic` reason as expected shutdown** - they are logged as faults for a reason; use `TerminateReasonNormal` to stop a process on purpose without log noise.
- **Never match `gen.ErrSupervisorRestartsExceeded`** - that symbol does not exist; the restart-intensity reason is `gen.ErrExceeded`.

# Supervision

Supervisors manage the lifecycle of child processes with configurable restart policies. A supervisor is a `ProcessBehavior` (`act.Supervisor`) whose `Init` returns a `SupervisorSpec`. Children are spawned automatically; a child failure triggers restart according to the supervisor's type and restart strategy.

## SupervisorBehavior

Embed `act.Supervisor` and implement `Init`. Every other method has a default implementation on the embedded struct, so you only override what you need.

```go
type SupervisorBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) (SupervisorSpec, error)   // mandatory

    HandleChildStart(name gen.Atom, pid gen.PID) error
    HandleChildTerminate(name gen.Atom, pid gen.PID, reason error) error
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`HandleChildStart` / `HandleChildTerminate` fire only when `SupervisorSpec.EnableHandleChild = true`. See godoc for the exhaustive method contract.

## SupervisorSpec

```go
type SupervisorSpec struct {
    Children            []SupervisorChildSpec
    Type                SupervisorType
    Restart             SupervisorRestart
    EnableHandleChild   bool // fire HandleChildStart/Terminate
    DisableAutoShutdown bool // keep supervisor alive even when no child is left
}
```

`Children` must not be empty (except that `SimpleOneForOne` starts them dynamically). Children are spawned sequentially in declaration order during `Init`; if any spawn fails the supervisor terminates with that error - no partial trees.

## Supervisor Types

| Type | Value | Behavior |
|------|-------|----------|
| `SupervisorTypeOneForOne` | 0 | Only the failing child restarts (default) |
| `SupervisorTypeAllForOne` | 1 | All children stop, then all restart on any failure |
| `SupervisorTypeRestForOne` | 2 | Failing child plus all siblings started after it restart |
| `SupervisorTypeSimpleOneForOne` | 3 | Dynamic pool; one shared spec (template); instances spawned via `StartChild()` |

`OneForOne`, `AllForOne`, `RestForOne` register each child by its spec `Name`, so there is one instance per spec. `SimpleOneForOne` spawns unregistered instances of a single template.

## Restart Strategy

`SupervisorRestart.Strategy` sets the default rule for every child. Each child may override it (see [Per-child restart override](#per-child-restart-override)).

| Strategy | Value | Behavior |
|----------|-------|----------|
| `SupervisorStrategyInherit` | 0 | Zero value. At child level: use the supervisor's `Strategy`. At supervisor level: normalized to `Transient` on init. |
| `SupervisorStrategyTransient` | 1 | Restart only on abnormal termination (effective default) |
| `SupervisorStrategyTemporary` | 2 | Never restart |
| `SupervisorStrategyPermanent` | 3 | Always restart, even on normal exit |

"Abnormal" means an exit reason other than `gen.TerminateReasonNormal` or `gen.TerminateReasonShutdown`. Because the supervisor-level zero value normalizes to `Transient`, leaving `Restart.Strategy` unset gives you Transient, and a child that leaves its own `Restart.Strategy` unset inherits the supervisor's.

## Restart Intensity (Sliding Window)

```go
type SupervisorRestart struct {
    Strategy  SupervisorStrategy
    Intensity uint16 // max restarts in window
    Period    uint16 // window in seconds
    KeepOrder bool   // ignored for OneForOne and SimpleOneForOne
}
```

On init, `Intensity == 0` is normalized to `5` and `Period == 0` to `5`. So the defaults are `Intensity: 5, Period: 5`, and omitting both fields gives that policy - there is no supervisor-level way to make every failure fatal via `Intensity: 0`.

The supervisor keeps a sliding window of restart timestamps (millisecond granularity). When a child needs restart it appends "now", drops timestamps older than `Period` seconds, and if the remaining count exceeds `Intensity` the window is exceeded.

When exceeded, the default reaction is: stop all running children, then terminate the supervisor. The failure cascades to the parent supervisor.

- Each running child is stopped with `gen.ErrExceeded` as its exit reason.
- The supervisor itself terminates with a wrapped reason built by `gen.Errorf("supervisor restart intensity exceeded (max %d in %ds): %w: %w", intensity, period, gen.ErrExceeded, originalChildReason)`.

There is no `ErrSupervisorRestartsExceeded` symbol. Match the cause structurally:

```go
if errors.Is(reason, gen.ErrExceeded) {
    // restart intensity was exceeded; original child reason is also
    // in the wrap chain - traverse with errors.Unwrap / errors.As
}
```

`KeepOrder` preserves child order during a mass restart (`AllForOne`, `RestForOne`): stop last-to-first sequentially, waiting for each to exit, then restart in declaration order. With `KeepOrder: false` the affected children stop in parallel.

## SupervisorChildSpec

```go
type SupervisorChildSpec struct {
    Name        gen.Atom               // registered name / template name (required)
    Significant bool                   // ignored for SimpleOneForOne or Permanent strategy
    Factory     gen.ProcessFactory     // required
    Options     gen.ProcessOptions
    Args        []any                  // passed to child's Init()
    Restart     SupervisorChildRestart // optional per-child override
}
```

**`Significant`** (AllForOne / RestForOne only): when a significant child exits in a way that would otherwise not restart the group (normal exit under Transient, any exit under Temporary), the supervisor stops all children and terminates instead. Use it for a child whose completion means "subtree done." Ignored for OneForOne / SimpleOneForOne and ignored under Permanent.

## Per-child restart override

`SupervisorChildRestart` on a child spec overrides the supervisor defaults for that one child. A zero-value struct inherits everything, so adding it never changes existing behavior.

```go
type SupervisorChildRestart struct {
    Strategy  SupervisorStrategy // Inherit (0) = use supervisor Strategy
    Intensity uint16             // 0 = use supervisor's shared counter
    Period    uint16             // seconds; requires Intensity > 0 (else defaults to 5)
    OnExceed  OnExceed
}
```

Three independent, opt-in axes:

- **Strategy.** Leave it `SupervisorStrategyInherit` (zero) to use the supervisor's, or set `Transient` / `Temporary` / `Permanent` to mix strategies within one supervisor. For AllForOne / RestForOne this controls whether *this child's* exit triggers the group restart (Permanent: always; Transient: on abnormal exit; Temporary: never).
- **Counter.** `Intensity > 0` gives the child a dedicated restart counter, separate from the supervisor's shared one. For OneForOne the counter is per-spec; for SimpleOneForOne it is per-instance (one logical instance = one `StartChild` call; its counter survives across restarts even as the PID changes). If `Period` is 0 with `Intensity > 0`, `Period` defaults to 5.
- **OnExceed.** What to do when *this child's* dedicated counter overflows.

```go
type OnExceed uint8

const (
    OnExceedTerminateSupervisor OnExceed = 0 // default: terminate supervisor
    OnExceedDisable             OnExceed = 1 // disable/drop the offending child, keep supervisor alive
)
```

| OnExceed | OneForOne | SimpleOneForOne |
|----------|-----------|-----------------|
| `OnExceedTerminateSupervisor` (default) | Supervisor terminates (same wrapped `gen.ErrExceeded` reason as the shared counter) | Supervisor terminates |
| `OnExceedDisable` | Child spec is marked disabled; re-enable with `EnableChild` (which clears the local counter) | Offending instance is dropped; spec stays available for new `StartChild` calls |

Validation at `Init` (all wrapped in `ErrSupervisorInvalidSpec`, so `errors.Is(err, act.ErrSupervisorInvalidSpec)` matches):

- `Intensity > 0` is rejected for `AllForOne` / `RestForOne` (group restart makes a per-child threshold meaningless).
- `OnExceedDisable` requires `Intensity > 0`.
- `Period > 0` requires `Intensity > 0`.
- Unknown `Strategy` value.

A per-child counter does not shield the child from a *shared*-counter overflow: if another child floods the supervisor's shared counter, the whole subtree (including `OnExceedDisable` children) is torn down.

```go
Children: []act.SupervisorChildSpec{
    {Name: "database", Factory: createDB}, // shares the supervisor counter
    {
        Name:    "telemetry",
        Factory: createTelemetry,
        Restart: act.SupervisorChildRestart{
            Intensity: 100,
            Period:    60,
            OnExceed:  act.OnExceedDisable, // flaps -> disabled, siblings keep running
        },
    },
},
```

## Preserving the mailbox across restarts

By default a restarted child starts with an empty mailbox and any queued messages are lost. Set `Options.PreserveMailbox: true` on the child spec so a `OneForOne` / `SimpleOneForOne` restart carries the dying incarnation's mailbox to the new one:

```go
act.SupervisorChildSpec{
    Name:    "worker",
    Factory: createWorker,
    Options: gen.ProcessOptions{PreserveMailbox: true},
}
```

On abnormal termination the runtime captures the child's mailbox into the exit reason (a `*gen.Error`); the supervisor extracts it and hands it to the next spawn, preserving priority order. Notes:

- Only abnormal exits capture the mailbox (not `Normal` / `Shutdown`).
- The message being processed when the child failed is treated as consumed and is not replayed (avoids poison-pill loops).
- In-process struct state is not preserved; the new incarnation runs `Init` from scratch.
- Rejected for `AllForOne` / `RestForOne` at init (validated with `ErrSupervisorInvalidSpec`). The live mailbox cannot cross the network - remote spawns receive a nil mailbox.

## SupervisorChild (returned by Children())

```go
type SupervisorChild struct {
    Spec        gen.Atom // spec / template name
    Name        gen.Atom // registered name; empty for SimpleOneForOne instances
    PID         gen.PID
    Significant bool
    Disabled    bool
}
```

`SimpleOneForOne` instances are not name-registered, so their `Name` is empty; identify them by `Spec` and `PID`.

## Dynamic Children

Retrieve and manipulate children at runtime. Every method returns `gen.ErrNotAllowed` if the supervisor is not in `ProcessStateRunning`.

```go
children := s.Children() // []SupervisorChild

err := s.StartChild("worker", args...) // returns error only, never the PID
err := s.AddChild(act.SupervisorChildSpec{Name: "cache", Factory: createCache})
err := s.EnableChild("name")  // re-start a disabled spec
err := s.DisableChild("name") // stop child (TerminateReasonShutdown) and mark disabled
```

`StartChild(name, args...)` behaves differently by type:

- **SimpleOneForOne** - launches a new instance from the template. `args` are stored per-instance and replayed on that instance's restarts (not the template's `Args`).
- **OneForOne / AllForOne / RestForOne** - re-starts a named spec that previously terminated (e.g. normally). `args` update the spec for future restarts.

Error returns:

| Error | When |
|-------|------|
| `gen.ErrNotAllowed` | supervisor not running |
| `act.ErrSupervisorStrategyActive` | a restart strategy is in flight (AllForOne / RestForOne reject all four methods; OneForOne rejects `StartChild`/`AddChild`) |
| `act.ErrSupervisorChildUnknown` | no spec with that name |
| `act.ErrSupervisorChildRunning` | `StartChild` on a spec whose instance is still alive |
| `act.ErrSupervisorChildDisabled` | `StartChild` on a disabled spec |
| `act.ErrSupervisorChildDuplicate` | `AddChild` name collides with an existing spec |
| `act.ErrSupervisorInvalidSpec` | invalid child spec / restart / options |

While a restart strategy runs, the supervisor processes only the Urgent queue (where exit signals arrive) and ignores regular and system messages, which is why management calls are rejected mid-restart.

## Minimal Supervisor

```go
package main

import (
    "ergo.services/ergo/act"
    "ergo.services/ergo/gen"
)

type WorkerSup struct {
    act.Supervisor
}

func createSupervisor() gen.ProcessBehavior {
    return &WorkerSup{}
}

func (s *WorkerSup) Init(args ...any) (act.SupervisorSpec, error) {
    return act.SupervisorSpec{
        Type: act.SupervisorTypeOneForOne,
        Restart: act.SupervisorRestart{
            Strategy:  act.SupervisorStrategyTransient,
            Intensity: 5,
            Period:    5,
        },
        Children: []act.SupervisorChildSpec{
            {
                Name:        "db-writer",
                Significant: true,
                Factory:     createDBWriter,
            },
            {
                Name:    "cache",
                Factory: createCache,
            },
        },
    }, nil
}
```

## Dynamic Pool (SimpleOneForOne)

For homogeneous worker farms spawned on demand:

```go
func (s *WorkerPool) Init(args ...any) (act.SupervisorSpec, error) {
    return act.SupervisorSpec{
        Type: act.SupervisorTypeSimpleOneForOne,
        Restart: act.SupervisorRestart{
            Strategy: act.SupervisorStrategyTransient,
        },
        Children: []act.SupervisorChildSpec{
            {
                Name:    "worker", // shared template name
                Factory: createWorker,
            },
        },
    }, nil
}

// Elsewhere: request a new worker instance (returns error only)
err := sup.StartChild("worker", workerArgs...)
```

To keep one bad instance from wiping the whole pool, give the template a per-instance counter with `OnExceedDisable`:

```go
Children: []act.SupervisorChildSpec{{
    Name:    "worker",
    Factory: createWorker,
    Restart: act.SupervisorChildRestart{
        Intensity: 5,
        Period:    10,
        OnExceed:  act.OnExceedDisable, // drop just that instance on overflow
    },
}},
```

For performance-critical farms with mailbox-based load distribution, prefer `act.Pool` (see `pool.md`) over `SimpleOneForOne`.

## Child Failure Callback

Enable via `EnableHandleChild: true`:

```go
func (s *WorkerSup) HandleChildTerminate(name gen.Atom, pid gen.PID, reason error) error {
    s.Log().Warning("child %s (%v) terminated: %v", name, pid, reason)
    return nil
}
```

These callbacks are delivered *after* the restart strategy has finished acting on all children (they are not interleaved with restart activity), and `HandleChildStart` also fires on each child's initial startup. Returning a non-nil error from either callback terminates the supervisor. Use them for telemetry / integration, not to alter restart behavior - that is the framework's job.

## Anti-Patterns

- **Do not mix unrelated failure domains under one `AllForOne` supervisor** - a cache restart must not kill the DB connection pool. Split into separate supervisors.
- **Do not use `Permanent` for children that shut down normally as part of a workflow** - they will loop.
- **Do not manually spawn a process with the same name** as a supervisor-managed child; the supervisor owns that name.
- **Do not call management methods (`StartChild`, `AddChild`, `EnableChild`, `DisableChild`) during a restart** - they return `act.ErrSupervisorStrategyActive`. Wait for the strategy to finish.
- **Do not rely on a per-child `OnExceedDisable` counter to survive a shared-counter overflow** - it does not. Give every child its own `Intensity`, or raise the supervisor-level `Intensity`.
- **Do not use `SimpleOneForOne` for high-throughput load balancing** - use `act.Pool` (pool.md) for routing messages across workers with backpressure.

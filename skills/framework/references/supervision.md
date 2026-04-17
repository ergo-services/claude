# Supervision

Supervisors manage the lifecycle of child processes with configurable restart policies. A supervisor is a `ProcessBehavior` (`act.Supervisor`) whose `Init` returns a `SupervisorSpec`. Children are spawned automatically; failures trigger restart per the supervisor's strategy.

## SupervisorBehavior

```go
type SupervisorBehavior interface {
    Init(args ...any) (SupervisorSpec, error)
    HandleChildStart(name gen.Atom, pid gen.PID) error
    HandleChildTerminate(name gen.Atom, pid gen.PID, reason error) error
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`HandleChildStart` and `HandleChildTerminate` fire only when `SupervisorSpec.EnableHandleChild = true`.

## SupervisorSpec

```go
type SupervisorSpec struct {
    Children          []SupervisorChildSpec
    Type              SupervisorType
    Restart           SupervisorRestart
    EnableHandleChild bool   // fire HandleChildStart/Terminate
    DisableAutoShutdown bool // keep supervisor alive even if significant child exits
}
```

## Supervisor Types

| Type | Value | Behavior |
|------|-------|----------|
| `SupervisorTypeOneForOne` | 0 | Only the failing child restarts (default) |
| `SupervisorTypeAllForOne` | 1 | All children restart on any failure |
| `SupervisorTypeRestForOne` | 2 | Failed child plus all siblings after it restart |
| `SupervisorTypeSimpleOneForOne` | 3 | Dynamic pool; all children same spec; spawned via `StartChild()` |

## Restart Strategy

Applied per child (via `SupervisorRestart.Strategy`).

| Strategy | Behavior |
|----------|----------|
| `SupervisorStrategyTransient` | Restart only on abnormal termination (default) |
| `SupervisorStrategyTemporary` | Never restart |
| `SupervisorStrategyPermanent` | Always restart (even on normal exit) |

## Restart Intensity (Sliding Window)

```go
type SupervisorRestart struct {
    Strategy  SupervisorStrategy
    Intensity uint16  // max restarts in window
    Period    uint16  // window in seconds
    KeepOrder bool    // ignored for OneForOne and SimpleOneForOne
}
```

`Intensity: 5, Period: 5` means max 5 restarts within any 5-second sliding window. When exceeded, supervisor dies with `ErrSupervisorRestartsExceeded` — the failure cascades to its parent supervisor.

`KeepOrder` preserves child startup order during mass restarts (`AllForOne`, `RestForOne`).

## SupervisorChildSpec

```go
type SupervisorChildSpec struct {
    Name        gen.Atom          // registered name (required)
    Significant bool              // ignored for SimpleOneForOne or Permanent strategy
    Factory     gen.ProcessFactory
    Options     gen.ProcessOptions
    Args        []any             // passed to child's Init()
}
```

**`Significant`**: when a significant child exits, the supervisor also exits (unless `DisableAutoShutdown`). Use for critical children whose absence invalidates the supervisor's purpose.

## Dynamic Children

Retrieve and manipulate children at runtime:

```go
children := s.Children()  // []SupervisorChild

// Start a new instance from a named child spec.
// For SimpleOneForOne the spec is the worker template; args are stored
// per-instance and replayed on restart.
err := s.StartChild("worker", workerArgs...)  // returns error only, not PID

// Add a new SupervisorChildSpec at runtime (non-SimpleOneForOne).
err := s.AddChild(act.SupervisorChildSpec{Name: "cache", Factory: createCache})

// Enable/disable existing specs by name.
err := s.EnableChild("name")    // re-enable after DisableChild
err := s.DisableChild("name")   // stop child, do not restart
```

`StartChild(name gen.Atom, args ...any) error` — takes the spec name as first argument; the spawned PID is tracked internally (`s.Children()` lists it). It does not return the PID.

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
            Period:    10,
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
                Name:    "worker",  // shared spec name
                Factory: createWorker,
            },
        },
    }, nil
}

// Elsewhere: request a new worker (returns error only)
err := sup.StartChild("worker", workerArgs...)
```

For performance-critical worker farms with mailbox-based load distribution, prefer `act.Pool` (see `pool.md`) over `SimpleOneForOne`.

## Child Failure Callback

Enable via `EnableHandleChild: true`:

```go
func (s *WorkerSup) HandleChildTerminate(name gen.Atom, pid gen.PID, reason error) error {
    s.Log().Warning("child %s (%v) terminated: %v", name, pid, reason)
    return nil
}
```

Useful for telemetry and metrics. Do not use to alter restart behavior — that is the framework's job.

## Anti-Patterns

- **Do not set `Intensity: 0`** — every failure is fatal, cascading immediately.
- **Do not mix unrelated failure domains under one supervisor** with `AllForOne`; cache restart must not kill the DB connection pool.
- **Do not use `Permanent` strategy for children that shut down normally as part of a workflow**; they will loop.
- **Do not manually spawn children with the same name** as a supervisor-managed child; the supervisor owns that name.
- **Do not use `SimpleOneForOne` for high-throughput load balancing**. Use `act.Pool` (pool.md) for routing messages across workers with backpressure.

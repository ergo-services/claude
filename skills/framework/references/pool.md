# Pool

`act.Pool` is a high-throughput worker farm: a supervisor-like process that spawns workers and routes messages across them based on mailbox backpressure. Unlike `SimpleOneForOne`, the Pool itself receives messages and load-balances them across workers.

## PoolBehavior

```go
type PoolBehavior interface {
    Init(args ...any) (PoolOptions, error)
    HandleMessage(from gen.PID, message any) error   // Urgent/System priority only
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

Pool's `HandleMessage` is **not** the worker's handler — it is the Pool's own handler for management-priority traffic. Normal-priority messages arriving at the Pool PID are forwarded to workers and reach the worker's `HandleMessage`, never the Pool's.

| Message path | Who handles |
|--------------|------------|
| Sent with `MessagePriorityMax` (Urgent) | Pool's `HandleMessage` — used for scale commands, reconfiguration |
| Sent with `MessagePriorityHigh` (System) | Pool's `HandleMessage` |
| Sent with `MessagePriorityNormal` (default) | Forwarded to a worker; worker's `HandleMessage` runs |
| Exit/system-level messages delivered to Pool | Pool's `HandleMessage` |
| Normal-priority message when all worker mailboxes full | Dropped, `messages_unhandled` counter increments |

## PoolOptions

```go
type PoolOptions struct {
    WorkerMailboxSize int64              // per-worker mailbox depth
    PoolSize          int64              // number of workers
    WorkerFactory     gen.ProcessFactory
    WorkerArgs        []any
}
```

**Total capacity** = `PoolSize * WorkerMailboxSize`. When all worker mailboxes are full, incoming messages are **dropped** (counted as `messages_unhandled`, not queued at Pool level).

## Dynamic Scaling

```go
newSize, err := p.AddWorkers(n)    // spawn N more workers
newSize, err := p.RemoveWorkers(n) // terminate N workers
```

Available when Pool is in `ProcessStateRunning`.

## When to Use Pool

| Message rate | Work items | Worker state | Use Pool? |
|--------------|-----------|--------------|-----------|
| < 1000 msg/sec | any | any | No — single actor suffices |
| > 1000 msg/sec | independent | stateless | Yes |
| > 1000 msg/sec | ordered (per key) | any | No — use sharded actor hashing |
| any | workers share state | stateful | No — state can't be split |

## Worker Implementation

A worker is an ordinary `act.Actor`. It receives forwarded messages in its own `HandleMessage`.

```go
type Worker struct {
    act.Actor
}

func createWorker() gen.ProcessBehavior {
    return &Worker{}
}

type WorkRequest struct {
    ID   string
    Data []byte
}

func (w *Worker) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case WorkRequest:
        result := process(m.Data)
        w.Send(from, WorkResult{ID: m.ID, Result: result})
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
    }, nil  // total capacity = 1000 concurrent
}

// Dynamic scale at runtime
type ScaleUp struct{ By int }

func (p *ComputePool) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case ScaleUp:
        newSize, err := p.AddWorkers(m.By)
        if err != nil {
            p.Log().Error("scale failed: %v", err)
            return nil
        }
        p.Log().Info("scaled to %d workers", newSize)
    }
    return nil
}
```

Route messages by sending to the Pool PID:

```go
// Message goes to any available worker
err := node.Send(poolPID, WorkRequest{ID: "1", Data: payload})
```

## Pool vs TCPServer.ProcessPool

**Do not** use `act.Pool` as a target in `TCPServerOptions.ProcessPool`. The `ProcessPool` field expects `[]gen.Atom` (registered names), and the TCPServer routes each connection to a specific member based on its alias. Putting a Pool there breaks the connection-to-worker binding and corrupts per-connection protocol state.

For TCP servers, use static named processes in `ProcessPool`. For internal work distribution, use `act.Pool` with a Pool PID as the target.

## Anti-Patterns

- **Do not use Pool for low-volume work** — the overhead of routing is wasted below ~1000 msg/sec.
- **Do not put `act.Pool` in `TCPServerOptions.ProcessPool`** — see above.
- **Do not rely on worker state persisting across messages if workers can be dropped** — `RemoveWorkers` kills arbitrary workers.
- **Do not route ordered messages through a Pool** — no ordering guarantee across workers.
- **Do not route synchronous `Call` to a Pool unless you accept that a worker may be unavailable** — use `CallWithTimeout` and handle `gen.ErrTimeout` / `gen.ErrProcessMailboxFull`.

# Operational Actors

Three ready-made actors for operational concerns: Kubernetes health probes, Raft-style leader election, Prometheus metrics. Each is a separate Go module.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/actor/health` | `health` | Health probes for Kubernetes |
| `ergo.services/actor/leader` | `leader` | Distributed leader election |
| `ergo.services/actor/metrics` | `metrics` | Prometheus metrics export |

## health — Kubernetes Probes

Serves `/health/live`, `/health/ready`, `/health/startup` HTTP endpoints. Other actors register named signals with probe bitmasks and optional heartbeat timeouts; the health actor aggregates status across signals.

**When to use:** containerized deployment where K8s needs liveness/readiness/startup probes. Default port 3000.

### Options

```go
type Options struct {
    Host          string        // default: "localhost"
    Port          uint16        // default: 3000
    Path          string        // default: "/health"
    CheckInterval time.Duration // default: 1s, heartbeat-timeout scan interval
    Mux           *http.ServeMux // optional: share mux instead of starting own server
}
```

### Hook-in

```go
import "ergo.services/actor/health"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{})
node.SpawnRegister("health", health.Factory, gen.ProcessOptions{},
    health.Options{Port: 8080})
```

### Helper API (call from any actor)

```go
import "ergo.services/actor/health"

// Register a signal with probe type and heartbeat timeout (0 = no timeout)
health.Register(a, "health", "db", health.ProbeLiveness|health.ProbeReadiness, 30*time.Second)

// Send a heartbeat
health.Heartbeat(a, "health", "db")

// Explicitly mark up/down
health.SignalUp(a, "health", "db")
health.SignalDown(a, "health", "db")

health.Unregister(a, "health", "db")
```

Probe bitmask constants: `ProbeLiveness`, `ProbeReadiness`, `ProbeStartup`. A signal can participate in multiple probes by OR-ing bits.

**Semantics:** a signal marked down makes every probe containing that bit fail. K8s polls the HTTP endpoints, which atomically read current state — never blocks.

## leader — Distributed Leader Election

Raft-inspired term-based election with random backoff. Exactly one node in a bootstrap set becomes leader; on failure, survivors elect a replacement.

**When to use:** you need a single coordinator per cluster (job scheduler, migration runner, primary writer). Avoid for pure replication — use a proper consensus library.

### Behavior

```go
type ActorBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) (leader.Options, error)
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)

    // Leadership callbacks (override these)
    HandleBecomeLeader() error
    HandleBecomeFollower(leader gen.PID) error
    HandlePeerJoined(peer gen.PID) error
    HandlePeerLeft(peer gen.PID) error
    HandleTermChanged(oldTerm, newTerm uint64) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

### Options

```go
type Options struct {
    ClusterID string            // required, non-empty
    Bootstrap []gen.ProcessID   // initial peer list
    ElectionTimeoutMin int      // ms, default 150
    ElectionTimeoutMax int      // ms, default 300 (must be > Min)
    HeartbeatInterval  int      // ms, default 50 (must be < ElectionTimeoutMin)
}
```

### Hook-in

```go
import "ergo.services/actor/leader"

type MyLeader struct {
    leader.Actor  // embeds the behavior
}

func factoryMyLeader() gen.ProcessBehavior { return &MyLeader{} }

func (l *MyLeader) Init(args ...any) (leader.Options, error) {
    return leader.Options{
        ClusterID: "scheduler",
        Bootstrap: []gen.ProcessID{
            {Node: "n1@host", Name: "scheduler-leader"},
            {Node: "n2@host", Name: "scheduler-leader"},
            {Node: "n3@host", Name: "scheduler-leader"},
        },
    }, nil
}

func (l *MyLeader) HandleBecomeLeader() error {
    l.Log().Info("became leader")
    // start leader-only work here
    return nil
}

func (l *MyLeader) HandleBecomeFollower(leader gen.PID) error {
    l.Log().Info("follower of %v", leader)
    return nil
}
```

### Split-Brain Prevention

Elections require quorum. With 3 peers, majority is 2. A node isolated with only itself cannot elect itself — by design.

## metrics — Prometheus Exporter

Collects Ergo node and network telemetry (uptime, processes, heap, GC, per-connection traffic, mailbox latency, top-N processes) and exposes them via HTTP in Prometheus format. Extendable: embed `metrics.Actor` and add application metrics to the same registry.

**When to use:** any production deployment. Pair with Grafana or similar. Build tags `-tags=pprof` and `-tags=latency` enable deeper metrics.

### Options

```go
type Options struct {
    Host            string        // default: "localhost"
    Port            uint16        // default: 9090
    Path            string        // default: "/metrics"
    CollectInterval time.Duration // how often to sample internal metrics
    TopN            int           // top-N processes to track
    Shared          *metrics.Shared  // aggregate across multiple actor instances
}
```

### Hook-in

```go
import "ergo.services/actor/metrics"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{})
node.SpawnRegister("metrics", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Port: 9090})
```

### Custom Metrics

Send counter/gauge/histogram updates via typed messages:

```go
node.Send("metrics", metrics.MessageCounterAdd{
    Name:   "orders_processed",
    Labels: map[string]string{"status": "success"},
    Value:  1,
})

node.Send("metrics", metrics.MessageGaugeSet{
    Name:  "queue_depth",
    Value: 42,
})

node.Send("metrics", metrics.MessageHistogramObserve{
    Name:  "request_duration_seconds",
    Value: 0.127,
})
```

### Embedding for Application Metrics

```go
type MyMetrics struct {
    metrics.Actor
}

func factoryMyMetrics() gen.ProcessBehavior { return &MyMetrics{} }

func (m *MyMetrics) CollectMetrics() {
    m.Actor.CollectMetrics()  // collect base Ergo metrics
    // add your own prometheus.Collector registrations
}
```

### Shared Mode

When multiple `metrics.Actor` instances run on one node (e.g., per-app), pass a shared registry:

```go
shared := metrics.NewShared()

nodeA.SpawnRegister("metrics", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Shared: shared, Port: 9090})

nodeA.SpawnRegister("metrics-app2", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Shared: shared})  // no Port, single HTTP server
```

## Combining Them

For most services, pair `health` + `metrics`. If you don't want to spawn both by hand, use the **radar** application (`applications-lib.md`) which bundles them on a single HTTP endpoint.

## Anti-Patterns

- **Do not rely on `health` probes for business logic** — they are HTTP endpoints for K8s, not status queries for peers.
- **Do not bootstrap `leader` with a single peer** — elections require quorum.
- **Do not use `leader` for replication** — it guarantees a single elected process, not replicated state.
- **Do not export `metrics` without authentication in production** — use a sidecar, internal network, or TLS.
- **Do not send `MessageHistogramObserve` on a hot path without pre-registering the histogram name** — first observation triggers registration, which is more expensive.

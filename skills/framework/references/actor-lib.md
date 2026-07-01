# Operational Actors

Three ready-made actors for operational concerns: Kubernetes health probes, Raft-style leader election, Prometheus metrics. Each is a separate Go module.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/actor/health` | `health` | Health probes for Kubernetes |
| `ergo.services/actor/leader` | `leader` | Distributed leader election |
| `ergo.services/actor/metrics` | `metrics` | Prometheus metrics export |

## health - Kubernetes Probes

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

These package functions are the helper-API pattern in practice: each takes the caller's `gen.Process` and wraps the Send/Call to the health actor by its registered name. See `application.md` (Exposing a Helper API) for the pattern and when to take `gen.Node` instead.

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

Probe bitmask constants (type `health.Probe`): `ProbeLiveness`, `ProbeReadiness`, `ProbeStartup`. A signal can participate in multiple probes by OR-ing bits. Register with `Probe` 0 defaults to `ProbeLiveness`.

`Register` and `Unregister` are synchronous Calls and return an error; `Heartbeat`, `SignalUp`, `SignalDown` are async Sends.

**Semantics:**

- A signal marked down makes every probe containing that bit fail. K8s polls the HTTP endpoints, which atomically read pre-built state - serving a probe never blocks the actor.
- `Register` monitors the calling process. If that process terminates, every signal it registered is automatically marked down. Register from a long-lived actor, or re-register on restart.
- With a non-zero heartbeat timeout, missing a `Heartbeat` for longer than the timeout marks the signal down. Timeouts are scanned every `CheckInterval` (default 1s). The next `Heartbeat` recovers the signal (marks it up again).
- Override `HandleSignalUp(signal gen.Atom) error` / `HandleSignalDown(signal gen.Atom) error` on an embedded `health.Actor` to react to transitions (the `ActorBehavior` interface also requires them; the base `Actor` provides no-op defaults).

## leader - Distributed Leader Election

Raft-inspired term-based election with random backoff. Exactly one node in a bootstrap set becomes leader; on failure, survivors elect a replacement.

**When to use:** you need a single coordinator per cluster (job scheduler, migration runner, primary writer). Avoid for pure replication - use a proper consensus library.

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

### Querying and Control

The embedded `leader.Actor` exposes query and control methods. Call them from your callbacks or message handlers - never spin or block on them.

| Method | Returns | Purpose |
|--------|---------|---------|
| `IsLeader()` | `bool` | true if this process currently holds leadership |
| `Leader()` | `gen.PID` | current leader (zero PID if none) |
| `Term()` | `uint64` | current election term |
| `Peers()` | `[]gen.PID` | snapshot copy of discovered peers |
| `PeerCount()` | `int` | number of known peers |
| `HasPeer(pid)` | `bool` | whether a PID is a known peer |
| `ClusterID()` | `string` | configured cluster id |
| `Bootstrap()` | `[]gen.ProcessID` | configured bootstrap list |
| `Broadcast(msg)` | | send `msg` to every discovered peer |
| `BroadcastBootstrap(msg)` | | send `msg` to every bootstrap peer except self |
| `Join(peer)` | | add a peer dynamically (sends it a vote request) |

Gate leader-only work on `IsLeader()` and fan work out with `Broadcast`:

```go
func (l *MyLeader) HandleMessage(from gen.PID, message any) error {
    if l.IsLeader() == false {
        return nil  // only the leader schedules
    }
    l.Broadcast(MessageAssignTask{ID: "42"})
    return nil
}
```

Peers are discovered automatically from any election message, so `Bootstrap` only accelerates initial formation. See godoc for the full method set.

### Split-Brain Prevention

A candidate counts its own vote plus the grants it receives; to win it needs `floor(N/2)+1` where `N` is the number of discovered peers (peers exclude self). With 3 total nodes each node discovers 2 peers, so quorum is 2 - the candidate needs its own vote plus one grant. A node that has discovered no peers computes quorum 1 and self-elects immediately: this is the intended single-node bootstrap, not a bug.

Split-brain is prevented by term progression combined with same-term heartbeat step-down, not by quorum blocking a lone node. A term is monotonic; a higher term always wins, and a candidate seeing a higher term reverts to follower. If a leader receives a heartbeat claiming leadership in its own current term, it steps down to follower. So when two partitions heal, the node with the lower term or the one that receives the other's heartbeat yields, leaving exactly one leader.

## metrics - Prometheus Exporter

Collects Ergo node and network telemetry (uptime, processes, heap, GC, per-connection traffic, mailbox latency, top-N processes) and exposes them via HTTP in Prometheus format. Extendable: embed `metrics.Actor` and add application metrics to the same registry.

**When to use:** any production deployment. Pair with Grafana or similar. Build tags `-tags=pprof` and `-tags=latency` enable deeper metrics.

### Options

```go
type Options struct {
    Host            string        // default: "localhost" (standalone only)
    Port            uint16        // default: 3000 (standalone only)
    Path            string        // default: "/metrics"
    CollectInterval time.Duration // default: 10s, internal sampling interval
    TopN            int           // default: 50, top-N processes to track
    Mux             *http.ServeMux   // optional: share a mux instead of starting own server
    Shared          *metrics.Shared  // aggregate across multiple actor instances
}
```

`Host` and `Port` defaults are applied only in standalone mode (`Shared == nil`). `health` and `metrics` share default port 3000, so if you run both standalone without radar you must give them distinct ports or a shared `Mux`.

### Hook-in

```go
import "ergo.services/actor/metrics"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{})
node.SpawnRegister("metrics", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Port: 3000})
```

### Custom Metrics

A custom metric must be **registered before it is updated**. Registration is a synchronous Call (helpers wrap `RegisterRequest` and return an error); updates are async Sends. An update message for a name that was never registered is dropped with a warning - it is not auto-registered. So register once, then send updates on the hot path.

The metric type constants are `metrics.MetricGauge`, `metrics.MetricCounter`, `metrics.MetricHistogram`, `metrics.MetricTopN`. Register with the typed helpers, passing label names for Vec metrics (empty slice for a plain metric):

```go
import "ergo.services/actor/metrics"

// Register once (sync Call, returns error). "metrics" is the actor's registered name.
metrics.RegisterCounter(a, "metrics", "orders_processed", "Orders processed", []string{"status"})
metrics.RegisterGauge(a, "metrics", "queue_depth", "Pending queue depth", nil)
metrics.RegisterHistogram(a, "metrics", "request_duration_seconds", "Request latency", nil, nil)
```

Then update via helpers (async Sends). For Vec metrics pass label **values** in the same order as the registered label names; for plain metrics pass `nil`:

```go
metrics.CounterAdd(a, "metrics", "orders_processed", 1, []string{"success"})
metrics.GaugeSet(a, "metrics", "queue_depth", 42, nil)
metrics.HistogramObserve(a, "metrics", "request_duration_seconds", 0.127, nil)
```

The same operations are also available as typed messages if you prefer sending directly: `metrics.MessageCounterAdd`, `metrics.MessageGaugeSet`, `metrics.MessageGaugeAdd`, `metrics.MessageHistogramObserve`, `metrics.MessageUnregister` (each carries `Name`, `Value`, `Labels`). Registration has no message form - it must go through the `RegisterRequest` Call (or a helper).

Custom metrics are tied to the registering process: the metrics actor monitors it, and when it terminates the metric is unregistered automatically and disappears from `/metrics`. Register from a long-lived process, or re-register on restart.

### Top-N Custom Metrics

A top-N metric tracks the highest (or lowest) N observed values across label sets, backed by a dedicated actor per metric spawned under a simple-one-for-one (SOFO) supervisor. The `metrics` package supplies the pieces - `metrics.RegisterTopN`, `metrics.TopNObserve`, `metrics.TopNOrder` (`TopNMax` = largest N, `TopNMin` = smallest N) - but the SOFO supervisor that services `RegisterTopNRequest` ships with the **radar** application, so top-N is used through radar:

```go
import "ergo.services/application/radar"

// Register (sync Call to radar's top-N supervisor). Spawns a dedicated actor.
radar.RegisterTopN(a, "slow_queries", "Slowest queries", 10, radar.TopNMax, []string{"query"})

// Observe (async Send addressed to the per-metric actor)
radar.TopNObserve(a, "slow_queries", 1.42, []string{"select_orders"})
```

`TopNObserve` takes no explicit target name at the low level - the helper addresses the per-metric actor `radar_topn_<name>`.

### Embedding for Application Metrics

Embed `metrics.Actor` and override `CollectMetrics() error`. The framework calls it once per `CollectInterval`; a non-nil return stops collection, so propagate the base error and return `nil` on success:

```go
type MyMetrics struct {
    metrics.Actor
}

func factoryMyMetrics() gen.ProcessBehavior { return &MyMetrics{} }

func (m *MyMetrics) CollectMetrics() error {
    if err := m.Actor.CollectMetrics(); err != nil {  // base returns nil by default
        return err
    }
    // refresh your own gauges/counters here
    return nil
}
```

`metrics.ActorBehavior` also declares `HandleEvent(gen.MessageEvent) error`; the embedded `metrics.Actor` provides a default, so override it only if you consume node events.

### Shared Mode

When multiple `metrics.Actor` instances run on one node (e.g., per-app), pass a shared registry:

```go
shared := metrics.NewShared()

nodeA.SpawnRegister("metrics", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Shared: shared, Port: 3000})

nodeA.SpawnRegister("metrics-app2", metrics.Factory, gen.ProcessOptions{},
    metrics.Options{Shared: shared})  // no Port, single HTTP server
```

## Combining Them

For most services, pair `health` + `metrics`. If you don't want to spawn both by hand, use the **radar** application (`applications-lib.md`) which bundles them on a single HTTP endpoint.

## Anti-Patterns

- **Do not rely on `health` probes for business logic** - they are HTTP endpoints for K8s, not status queries for peers.
- **Do not confuse single-node bootstrap with split-brain safety** - a node with no discovered peers self-elects by design; correctness across partitions comes from term progression and same-term heartbeat step-down, so ensure peers can actually reach each other rather than relying on quorum to block a lone node.
- **Do not use `leader` for replication** - it guarantees a single elected process, not replicated state.
- **Do not export `metrics` without authentication in production** - use a sidecar, internal network, or TLS.
- **Do not update a custom metric before registering it** - updates to an unregistered name are dropped with a warning. Register once via `RegisterCounter`/`RegisterGauge`/`RegisterHistogram`, then send updates.
- **Do not register custom metrics from a short-lived actor** - the metric is monitored and unregistered when its owner dies. Register from a long-lived process or re-register on restart.

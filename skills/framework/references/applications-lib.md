# Operational Applications

Four ready-to-run applications for observability and diagnostics. Each is a separate Go module.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/application/mcp` | `mcp` | MCP diagnostics sidecar (powers the `devops` agent) |
| `ergo.services/application/observer` | `observer` | Web dashboard for live node state |
| `ergo.services/application/pulse` | `pulse` | OTLP tracing exporter |
| `ergo.services/application/radar` | `radar` | Bundled health + metrics endpoint |

## mcp - Diagnostics Sidecar

Exposes 45+ inspection and action tools over HTTP via MCP (Model Context Protocol). Enables AI agents (Claude Code, specifically the `devops` agent) to diagnose bottlenecks, inspect processes, run pprof, monitor metrics, and perform actions (send messages, kill processes) on live nodes without restarts. Supports cluster-wide proxying: pass `node=X` on any tool and MCP forwards to that node via native Ergo networking.

**When to use:** always, in every environment. Read-only in production.

### Options

```go
type Options struct {
    Host         string         // default: "localhost"
    Port         uint16         // default 9922; 0 = agent mode (no HTTP, actor-only proxy)
    CertManager  gen.CertManager // TLS
    Token        string         // Bearer auth; empty = no auth
    ReadOnly     bool           // disable action tools (send_message, process_kill, etc.)
    AllowedTools []string       // whitelist; nil/empty = all allowed (respecting ReadOnly)
    PoolSize     int            // worker pool for tool execution, default 5
    LogLevel     gen.LogLevel
}
```

### Hook-in (Primary)

```go
import "ergo.services/application/mcp"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{
        mcp.CreateApp(mcp.Options{
            Port:     9922,
            Token:    os.Getenv("MCP_TOKEN"),
            ReadOnly: true,  // production
        }),
    },
})
```

### Hook-in (Agent Mode for Cluster Member)

```go
mcp.CreateApp(mcp.Options{Port: 0})  // actor-only, queried via cluster proxy
```

Agent-mode nodes don't run HTTP; they serve inspection requests forwarded from an entry-point MCP node.

### Claude Code Connection

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp
```

See the `devops` agent and `devops` skill for diagnostic playbooks using these tools.

## observer - Web Dashboard

Live web UI showing processes, applications, network topology, events, and metrics via Server-Sent Events. Part of the observability stack.

**When to use:** interactive inspection during development; optional in production.

### Options

```go
type Options struct {
    Host     string         // default: "localhost"
    Port     uint16         // default: 9911
    PoolSize int            // POST worker pool; default: 25
    LogLevel gen.LogLevel
}
```

No built-in TLS or auth fields - front with a reverse proxy if exposure beyond localhost is needed.

### Hook-in

```go
import "ergo.services/application/observer"

observer.CreateApp(observer.Options{Port: 9911})
// Visit http://localhost:9911
```

## pulse - OTLP Tracing Exporter

Collects tracing spans from the node and exports to an OTLP/HTTP collector (Grafana Tempo, Jaeger, etc.) via a pool of worker actors that batch and flush periodically.

**When to use:** distributed tracing is part of the observability stack. Pair with `EnableTracing: true` in `NetworkFlags` for cross-node trace context propagation.

### Options

```go
type Options struct {
    URL           string            // default: "http://localhost:4318/v1/traces"
    Headers       map[string]string // e.g. auth tokens
    BatchSize     int               // default: 512 spans per batch
    FlushInterval time.Duration     // default: 5s
    PoolSize      int               // export workers, default: 3
    ExportTimeout time.Duration     // default: 10s per HTTP request
    Flags         gen.TracingFlags  // default: Send | Receive | Procs
}
```

### Hook-in

```go
import "ergo.services/application/pulse"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{
        pulse.CreateApp(pulse.Options{
            URL:     "http://tempo:4318/v1/traces",
            Headers: map[string]string{"X-Scope-OrgID": "tenant-1"},
        }),
    },
    Network: gen.NetworkOptions{
        Flags: gen.NetworkFlags{
            Enable:        true,
            EnableTracing: true,  // required for cross-node propagation
        },
    },
})
```

Pulse registers itself as a tracing exporter automatically at startup (its pool calls `Node().TracingExporterAddPID` in `Init`), so no `gen.TracingOptions` entry in `NodeOptions` is required. `TracingOptions.Exporters` is only for exporters you want registered before the node reaches the running state; pulse does not need it.

Spans are produced only when a trace is active. Turn tracing on with a node sampler (`node.SetTracingSampler(...)`) or by spawning processes with `TracingFlagInherit` so children carry trace context. `pulse.Options.Flags` (default `TracingFlagSend | TracingFlagReceive | TracingFlagProcs`) selects which span kinds pulse exports. See `references/node.md` for the tracing subsystem and sampler API.

## radar - Health + Metrics Bundle

Runs `actor/health` and `actor/metrics` on **one shared HTTP server**, simplifying K8s deployments (single port to expose, single sidecar to configure).

**When to use:** you want both health probes and Prometheus metrics with minimal wiring.

### Options

```go
type Options struct {
    Host                   string        // default: "localhost"
    Port                   uint16        // default: 9090
    HealthPath             string        // default: "/health"
    MetricsPath            string        // default: "/metrics"
    HealthCheckInterval    time.Duration // 0 -> 1s  (heartbeat-timeout scan)
    MetricsCollectInterval time.Duration // 0 -> 10s (base metric sampling)
    MetricsTopN            int           // 0 -> 50  (top-N processes tracked)
    MetricsPoolSize        int64         // default: 3 (custom-metric workers)
}
```

`radar.CreateApp` fills only `Host`, `Port`, `HealthPath`, `MetricsPath`, and `MetricsPoolSize`. `HealthCheckInterval`, `MetricsCollectInterval`, and `MetricsTopN` pass through to the underlying health and metrics actors, which apply their own defaults (1s, 10s, 50) when left zero.

### Hook-in

```go
import "ergo.services/application/radar"

node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{
        radar.CreateApp(radar.Options{
            Host: "0.0.0.0",
            Port: 9090,
        }),
    },
})

// K8s:
//   livenessProbe:  GET /health/live
//   readinessProbe: GET /health/ready
//   prometheus:     GET /metrics
```

### Feeding Health and Metrics from Actors

Radar wraps the bundled health and metrics actors behind its own package-level helpers, so you never import `actor/health` / `actor/metrics` or need to know the internal actor names. Every helper takes the calling `gen.Process` as its first argument and routes to the right radar actor.

Health / service state:

```go
import "ergo.services/application/radar"

// Register a service signal, then keep it live.
radar.RegisterService(process, "db", radar.ProbeLiveness|radar.ProbeReadiness, 30*time.Second)
radar.Heartbeat(process, "db")          // refresh the heartbeat
radar.ServiceUp(process, "db")          // mark healthy
radar.ServiceDown(process, "db")        // mark unhealthy
radar.UnregisterService(process, "db")
```

Custom Prometheus metrics:

```go
radar.RegisterCounter(process, "orders_total", "orders processed", []string{"status"})
radar.CounterAdd(process, "orders_total", 1, []string{"success"})

radar.RegisterGauge(process, "queue_depth", "pending jobs", nil)
radar.GaugeSet(process, "queue_depth", 42, nil)
radar.GaugeAdd(process, "queue_depth", -1, nil)

radar.RegisterHistogram(process, "request_seconds", "latency", nil, []float64{0.1, 0.5, 1})
radar.HistogramObserve(process, "request_seconds", 0.127, nil)

radar.UnregisterMetric(process, "queue_depth")
```

Top-N metrics (a dedicated actor tracks the ranked observations):

```go
radar.RegisterTopN(process, "slow_queries", "slowest queries", 10, radar.TopNMax, []string{"table"})
radar.TopNObserve(process, "slow_queries", 512.0, []string{"orders"})
```

Radar re-exports the types and constants these helpers need, so no import of the underlying packages is required:

| Symbol | Alias of | Use |
|--------|----------|-----|
| `radar.Probe` | `health.Probe` | probe bitmask type |
| `radar.ProbeLiveness` / `radar.ProbeReadiness` / `radar.ProbeStartup` | `health.Probe*` | OR them for `RegisterService` |
| `radar.TopNOrder` | `metrics.TopNOrder` | top-N ordering type |
| `radar.TopNMax` / `radar.TopNMin` | `metrics.TopN*` | keep largest / smallest values |

For the exhaustive helper list, see the radar package godoc.

## Choosing Between Them

| Need | Application |
|------|-------------|
| AI-driven diagnostics, cluster inspection | `mcp` |
| Interactive web dashboard | `observer` |
| Distributed tracing to Tempo/Jaeger | `pulse` |
| K8s probes + Prometheus (minimal setup) | `radar` |
| K8s probes without metrics | `actor/health` directly |
| Prometheus metrics without probes | `actor/metrics` directly |

`mcp` and `observer` are not redundant: MCP is for agents and automation, observer is for humans. Both can run simultaneously.

## Anti-Patterns

- **Do not run `mcp` without `ReadOnly: true` and a `Token` in production** - action tools (`process_kill`, `send_exit`) can disrupt services.
- **Do not expose `observer` or `mcp` directly to the internet** - use VPN, SSH tunnel, or authenticated proxy.
- **Do not combine `radar` with separate `health` or `metrics` spawns on the same node** - duplicate HTTP listeners and duplicate metrics registrations cause conflicts.
- **Do not set `pulse.BatchSize` too small** on high-traffic nodes - HTTP export overhead dominates; 512 is a reasonable default.
- **Do not forget `EnableTracing` in `NetworkFlags`** when using `pulse` in a cluster - spans won't propagate across nodes without it.

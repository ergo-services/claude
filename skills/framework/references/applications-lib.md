# Operational Applications

Four ready-to-run applications for observability and diagnostics. Each is a separate Go module.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/application/mcp` | `mcp` | MCP diagnostics sidecar (powers the `devops` agent) |
| `ergo.services/application/observer` | `observer` | Web dashboard for live node state |
| `ergo.services/application/pulse` | `pulse` | OTLP tracing exporter |
| `ergo.services/application/radar` | `radar` | Bundled health + metrics endpoint |

## mcp — Diagnostics Sidecar

Exposes 46+ inspection tools over HTTP via MCP (Model Context Protocol). Enables AI agents (Claude Code, specifically the `devops` agent) to diagnose bottlenecks, inspect processes, run pprof, monitor metrics, and perform actions (send messages, kill processes) on live nodes without restarts. Supports cluster-wide proxying: pass `node=X` on any tool and MCP forwards to that node via native Ergo networking.

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

## observer — Web Dashboard

Live web UI showing processes, applications, network topology, events, and metrics via Server-Sent Events. Part of the observability stack.

**When to use:** interactive inspection during development; optional in production.

### Options

```go
type Options struct {
    Host     string         // default: "localhost"
    Port     uint16         // default: 9911
    PoolSize int            // POST worker pool; default: 10
    LogLevel gen.LogLevel
}
```

No built-in TLS or auth fields — front with a reverse proxy if exposure beyond localhost is needed.

### Hook-in

```go
import "ergo.services/application/observer"

observer.CreateApp(observer.Options{Port: 9911})
// Visit http://localhost:9911
```

## pulse — OTLP Tracing Exporter

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
    Tracing: gen.TracingOptions{
        // enable tracing subsystem; refer to gen.TracingOptions
    },
})
```

## radar — Health + Metrics Bundle

Runs `actor/health` and `actor/metrics` on **one shared HTTP server**, simplifying K8s deployments (single port to expose, single sidecar to configure).

**When to use:** you want both health probes and Prometheus metrics with minimal wiring.

### Options

```go
type Options struct {
    Host                string        // default: "localhost"
    Port                uint16        // default: 9090
    HealthPath          string        // default: "/health"
    MetricsPath         string        // default: "/metrics"
    HealthCheckInterval time.Duration
    MetricsCollectInterval time.Duration
    MetricsTopN         int
    MetricsPoolSize     int64         // default: 3
}
```

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

Internally uses `actor/health` and `actor/metrics`, so the package-level helpers (`health.Register`, `health.Heartbeat`, etc.) still work — point them at the health actor name used by radar.

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

- **Do not run `mcp` without `ReadOnly: true` and a `Token` in production** — action tools (`process_kill`, `send_exit`) can disrupt services.
- **Do not expose `observer` or `mcp` directly to the internet** — use VPN, SSH tunnel, or authenticated proxy.
- **Do not combine `radar` with separate `health` or `metrics` spawns on the same node** — duplicate HTTP listeners and duplicate metrics registrations cause conflicts.
- **Do not set `pulse.BatchSize` too small** on high-traffic nodes — HTTP export overhead dominates; 512 is a reasonable default.
- **Do not forget `EnableTracing` in `NetworkFlags`** when using `pulse` in a cluster — spans won't propagate across nodes without it.

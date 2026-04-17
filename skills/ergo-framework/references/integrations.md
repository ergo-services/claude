# Integrations: Registrars and Loggers

External service integrations shipped as separate modules.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/registrar/etcd` | `etcd` | etcd-backed `gen.Registrar` |
| `ergo.services/registrar/saturn` | `saturn` | Saturn registrar client (commercial) |
| `ergo.services/logger/colored` | `colored` | Terminal logger with color coding |
| `ergo.services/logger/rotate` | `rotate` | File logger with time-based rotation and compression |

## registrar/etcd

Dynamic service discovery and configuration via etcd. Suitable for clusters ≤ 70 nodes. Requires an existing etcd cluster.

### Options

```go
type Options struct {
    Cluster            string
    Endpoints          []string      // etcd addresses (default: localhost:2379)
    Username           string
    Password           string
    TLS                *tls.Config
    InsecureSkipVerify bool
    DialTimeout        time.Duration
    RequestTimeout     time.Duration
    KeepAlive          time.Duration
    LeaseTTL           int64         // seconds; default 10. Lower = faster failover, more etcd churn.
}
```

### Hook-in

```go
import "ergo.services/registrar/etcd"

reg, err := etcd.Create(etcd.Options{
    Cluster:   "prod",
    Endpoints: []string{"etcd1:2379", "etcd2:2379", "etcd3:2379"},
})
if err != nil {
    return err
}

node, err := ergo.StartNode("orders@host1", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: reg,
        Cookie:    os.Getenv("CLUSTER_COOKIE"),
    },
})
```

Nodes auto-register on start. Application routes are published whenever apps start/stop. Resolver queries etcd for discovery.

### Semantics

- Keys namespaced by `Cluster` + `Prefix`.
- Lease-based TTL: nodes maintain a lease; on crash, etcd expires keys automatically.
- Proxy routing: not supported (`Info().SupportRegisterProxy == false`).

## registrar/saturn

Client for the Saturn central registrar. Designed for large clusters (100+ nodes) with custom binary protocol over TLS and token-based authentication. Commercial license required for production.

### Options

```go
type Options struct {
    Cluster            string
    Port               uint16        // default 4499
    KeepAlive          time.Duration
    InsecureSkipVerify bool
}
```

### Hook-in

```go
import "ergo.services/registrar/saturn"

reg := saturn.Create(
    "saturn.prod.example",
    os.Getenv("SATURN_TOKEN"),
    saturn.Options{Cluster: "prod"},
)

node, err := ergo.StartNode("orders@host1", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: reg,
        Cookie:    os.Getenv("CLUSTER_COOKIE"),
    },
})
```

### Semantics

- Persistent TLS connection to Saturn.
- Config push: Saturn can broadcast configuration changes to nodes at runtime.
- SHA-256 digest handshake for auth.
- Supports proxy routing.

## logger/colored

Terminal logger with ANSI color coding for log levels and Ergo-native types (atoms, PIDs, references, aliases, events). Built on `fatih/color`.

**Use for:** local development, CLI tools. **Do not** use in high-throughput production (terminal formatting is expensive).

### Options

```go
type Options struct {
    TimeFormat      string  // e.g. time.RFC3339; empty = nanosecond timestamp
    IncludeBehavior bool
    IncludeName     bool
    IncludeFields   bool
    ShortLevelName  bool    // DBG/INF/WRN/ERR instead of DEBUG/INFO/...
    DisableBanner   bool
}
```

### Hook-in

```go
import "ergo.services/logger/colored"

logger, err := colored.CreateLogger(colored.Options{
    TimeFormat:     time.RFC3339,
    ShortLevelName: true,
})
if err != nil {
    return err
}

node, _ := ergo.StartNode("dev@localhost", gen.NodeOptions{
    Log: gen.LogOptions{
        Level:         gen.LogLevelDebug,
        DefaultLogger: gen.DefaultLoggerOptions{Disable: true},  // avoid duplicate output
        Loggers: []gen.Logger{
            {Name: "console", Logger: logger},
        },
    },
})
```

## logger/rotate

File logger writing to timestamped files with automatic rotation and optional gzip compression. Rotation happens at fixed `Period` boundaries (hourly, daily, etc.).

**Use for:** production deployments that persist logs to disk. Pair with a log shipper (Vector, Fluentd) for forwarding.

### Options

```go
type Options struct {
    TimeFormat      string        // time format for rendered entries (see time pkg)
    IncludeBehavior bool          // include process/meta behavior in messages
    IncludeName     bool          // include registered process name
    IncludeFields   bool          // include associated log fields
    ShortLevelName  bool          // DBG/INF/WRN/ERR/... short level names
    Path            string        // directory; "~" is expanded
    Prefix          string        // filename prefix
    Period          time.Duration // rotation interval; minimum 1 minute
    Compress        bool          // gzip rotated files
    Depth           int           // how many rotated files to keep
}
```

File naming: `{Prefix}.{YYYYMMDDHHMM}.log[.gz]` (the timestamp sits between Prefix and `.log`).

### Hook-in

```go
import "ergo.services/logger/rotate"

fileLogger, err := rotate.CreateLogger(rotate.Options{
    Path:     "/var/log/myapp",
    Prefix:   "myapp",
    Period:   24 * time.Hour,
    Compress: true,
})
if err != nil {
    return err
}

node, _ := ergo.StartNode("prod@host", gen.NodeOptions{
    Log: gen.LogOptions{
        Level: gen.LogLevelInfo,
        Loggers: []gen.Logger{
            {Name: "file", Logger: fileLogger},
        },
    },
})
```

## Combining Loggers

Multiple loggers can coexist. Common production setup: colored logger on stderr (for human operators watching journald) + rotating file logger (for persistence).

```go
Log: gen.LogOptions{
    Level: gen.LogLevelInfo,
    Loggers: []gen.Logger{
        {Name: "console", Logger: coloredLogger},
        {Name: "file",    Logger: fileLogger},
    },
},
```

Each logger can be filtered independently via its `LogLevel` filter parameter if exposed.

## Anti-Patterns

- **Do not use embedded registrar in production** — single-point-of-failure, no persistence. See `cluster.md`.
- **Do not share an etcd cluster across environments** unless you also separate by `Options.Cluster` — mixed registrations cause accidental cross-env calls.
- **Do not use `colored` logger in production** — terminal color codes pollute log aggregation; also slower.
- **Do not set `rotate.Period` below 1 minute** — rejected by the library.
- **Do not forget `TimeFormat`** if you rely on log timestamps for correlation — default is raw nanosecond integer.
- **Do not disable `DefaultLogger`** and forget to configure a replacement — the node will run without any logger output.
- **Do not expose `InsecureSkipVerify: true`** on Saturn or etcd in production — defeats TLS entirely.

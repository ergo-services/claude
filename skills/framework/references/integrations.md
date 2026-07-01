# Integrations: Registrars and Loggers

External service integrations shipped as separate modules.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/registrar/etcd` | `etcd` | etcd-backed `gen.Registrar` |
| `ergo.services/registrar/saturn` | `saturn` | Saturn registrar client (commercial) |
| `ergo.services/logger/colored` | `colored` | Terminal logger with color coding |
| `ergo.services/logger/rotate` | `rotate` | File logger with time-based rotation and compression |
| `ergo.services/logger/sentry` | `sentry` | Ships panics and errors to a Sentry project |

Each module has its own `go.mod` and version. This reference covers the logger and registrar backends only. The node-side logging API (`gen.LogOptions`, log levels, `Log()` on a process) lives in `node.md` - a logger backend here is what you plug into `LogOptions.Loggers`.

## registrar/etcd

Dynamic service discovery and configuration via etcd. Suitable for clusters ≤ 70 nodes. Requires an existing etcd cluster.

### Options

```go
type Options struct {
    Cluster            string        // key namespace; empty falls back to "default"
    Endpoints          []string      // etcd addresses; default localhost:2379
    Username           string
    Password           string
    TLS                *tls.Config
    InsecureSkipVerify bool
    DialTimeout        time.Duration // default 10s
    RequestTimeout     time.Duration // default 10s
    KeepAlive          time.Duration // default LeaseTTL/3 (min 1s)
    LeaseTTL           int64         // seconds; default 10. Lower = faster failover, more etcd churn.
}
```

`etcd.Create` returns `(gen.Registrar, error)`. An empty `Cluster` silently shares the `"default"` namespace, so always set it explicitly to keep environments apart.

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

- Keys live under the fixed prefix `services/ergo`, namespaced by `Cluster`: cluster routes and config under `services/ergo/cluster/{Cluster}/`, global cross-cluster config under `services/ergo/config/`. There is no configurable key-prefix option.
- Lease-based TTL: nodes maintain a lease; on crash, etcd expires the node's keys automatically.
- Weighted routing: when an application runs on several nodes, `Resolver().ResolveApplication` balances across them with smooth weighted round-robin (Nginx-style) and returns the routes pre-ordered - the chosen node at index `[0]`, the tail sorted by remaining preference. A caller that just takes `routes[0]` gets weighted distribution for free. Set a higher `ApplicationRoute.Weight` on nodes that should take proportionally more traffic; a weight `<= 0` is treated as `1`.
- Proxy routing: not supported. `RegisterProxy`/`UnregisterProxy` return `gen.ErrUnsupported`, `Info().SupportRegisterProxy == false`, and `ResolveProxy` returns `gen.ErrNoRoute`.

### Events

The registrar registers a node-local event on start. Obtain its name via `Registrar.Event()`, then `LinkEvent`/`MonitorEvent` it to observe cluster changes (`Info().SupportEvent == true`). etcd publishes these typed events:

| Event | Fires when |
|-------|-----------|
| `etcd.EventNodeJoined` | a node registers in the cluster |
| `etcd.EventNodeLeft` | a node deregisters or its lease expires |
| `etcd.EventApplicationLoaded` | an application is loaded on a node |
| `etcd.EventApplicationStarted` | an application starts running |
| `etcd.EventApplicationStopping` | an application begins stopping |
| `etcd.EventApplicationStopped` | an application has stopped |
| `etcd.EventConfigUpdate` | an effective config value changes |

`EventConfigUpdate` fires only when the hierarchy-resolved value a node actually sees changes, not on every write to etcd.

```go
func (w *Watcher) Init(args ...any) error {
    reg, err := w.Node().Network().Registrar()
    if err != nil {
        return err
    }
    ev, err := reg.Event()
    if err != nil {
        return err
    }
    _, err = w.MonitorEvent(ev)
    return err
}

func (w *Watcher) HandleEvent(event gen.MessageEvent) error {
    switch e := event.Message.(type) {
    case etcd.EventNodeJoined:
        w.Log().Info("node joined: %s", e.Name)
    case etcd.EventConfigUpdate:
        w.Log().Info("config %s = %v", e.Item, e.Value)
    }
    return nil
}
```

## registrar/saturn

Client for the Saturn central registrar. Designed for large clusters (100+ nodes) with custom binary protocol over TLS and token-based authentication. Commercial license required for production.

### Options

```go
type Options struct {
    Cluster            string
    Port               uint16        // default 4499
    KeepAlive          time.Duration // default 3s
    InsecureSkipVerify bool
}
```

`saturn.Create(host, token, options)` returns `gen.Registrar` (no error) - it dials lazily on `Register`.

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
- Config push: Saturn broadcasts configuration changes to nodes at runtime.
- SHA-256 digest handshake for auth (mutual: the server proves it holds the same token).
- Proxy routing: `ResolveProxy` works and `Info().SupportRegisterProxy == true`, but `RegisterProxy`/`UnregisterProxy` are not yet implemented and return `gen.ErrUnsupported`. Do not rely on registering a proxy route through Saturn today. (etcd supports no proxy operations at all.)

### Events

Same subscription pattern as etcd - obtain the event name via `Registrar.Event()`, then `LinkEvent`/`MonitorEvent`. Saturn publishes the same set as etcd (`saturn.EventNodeJoined`, `EventNodeLeft`, `EventApplicationLoaded`, `EventApplicationStarted`, `EventApplicationStopping`, `EventApplicationStopped`, `EventConfigUpdate`) and additionally emits `saturn.EventApplicationUnloaded` when an application route is removed.

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
    Path            string        // directory; "~" expanded; default <exe-dir>/logs
    Prefix          string        // filename prefix; default executable basename
    Period          time.Duration // rotation interval; minimum 1 minute
    Compress        bool          // gzip rotated files
    Depth           int           // how many rotated files to keep; 0 = keep all
}
```

The live log is `{Prefix}.log`. On each `Period` boundary it is copied to `{Prefix}.{YYYYMMDDHHMM}.log[.gz]` and truncated. When `Depth > 0` only the newest `Depth` rotated files are kept; older ones are deleted.

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

## logger/sentry

Ships panics and errors to a Sentry project, with a stack trace captured at the panic origin. Unlike `colored`, this backend is meant for production - it is how you get crash and error reporting out of a running cluster.

**Use for:** production error and crash reporting. Register it alongside a file or console logger; it does not replace them (it only forwards the two highest-severity levels).

### What it captures

The backend forwards a fixed slice of the log stream. Info, Warning, Debug, Trace, and System levels are never sent.

| Level | Node | Network | Application | Meta | Process |
|-------|:----:|:-------:|:-----------:|:----:|:-------:|
| Panic | yes | yes | yes | yes | yes |
| Error | yes | yes | yes | opt-in | opt-in |

All panics are captured regardless of source. Errors from meta processes and from user processes are off by default - enable them with `CaptureMetaErrors` / `CaptureProcessErrors`. Panic events carry a stack trace pointing at the line that actually panicked (captured while the framework's recover handler is still on the goroutine), with framework and runtime frames marked out-of-app so your code stands out in Sentry.

### Options

```go
type Options struct {
    DSN                  string        // Sentry project DSN; empty falls back to SENTRY_DSN env
    Environment          string        // sentry environment tag
    Release              string        // running binary version
    ServerName           string        // overrides auto-detected hostname
    CaptureMetaErrors    bool          // also send error-level events from meta processes
    CaptureProcessErrors bool          // also send error-level events from user processes
    QueueLimit           int           // internal queue cap; default 1024, drops when full
    FlushTimeout         time.Duration // bounds Terminate() drain + flush; default 5s
    SkipFrames           int           // stack frames trimmed off the top; default 5
    BeforeSend           func(*sentry.Event, *sentry.EventHint) *sentry.Event // return nil to drop
}
```

The `sentry` type in `BeforeSend` is `github.com/getsentry/sentry-go`. The logger uses its own Sentry hub and client, so a global `sentry.Init()` elsewhere in the app is left untouched. When the queue is full, further events are dropped silently to bound memory.

### Hook-in

```go
import "ergo.services/logger/sentry"

sentryLogger, err := sentry.CreateLogger(sentry.Options{
    DSN:         os.Getenv("SENTRY_DSN"),
    Environment: "production",
    Release:     "orders@1.4.2",
})
if err != nil {
    return err
}

node, _ := ergo.StartNode("orders@host", gen.NodeOptions{
    Log: gen.LogOptions{
        Level: gen.LogLevelInfo,
        Loggers: []gen.Logger{
            {Name: "file",   Logger: fileLogger},
            {Name: "sentry", Logger: sentryLogger},
        },
    },
})
```

Keep the node `Level` at your normal setting (`LogLevelInfo` etc). The Sentry backend selects panic and error itself; a coarse node `Level` only controls what is generated in the first place.

## Combining Loggers

Multiple loggers can coexist. Common production setup: rotating file logger for full persistence + Sentry backend for crashes and errors; add a colored console logger in development.

Each registration entry (`gen.Logger`) has a `Filter []LogLevel` field. An empty `Filter` (the default) means the logger receives every level the node generates; a non-empty `Filter` restricts that logger to exactly the listed levels. Use it to keep a file logger verbose while a second backend only sees the loud stuff:

```go
Log: gen.LogOptions{
    Level: gen.LogLevelInfo,
    Loggers: []gen.Logger{
        {Name: "file", Logger: fileLogger}, // Filter empty: all levels
        {
            Name:   "alerts",
            Logger: alertLogger,
            Filter: []gen.LogLevel{gen.LogLevelError, gen.LogLevelPanic},
        },
    },
},
```

`Filter` is a framework field on the registration, independent of the logger implementation. The Sentry backend applies its own capture matrix on top, so leaving its `Filter` empty is fine.

## Anti-Patterns

- **Do not use embedded registrar in production** - single-point-of-failure, no persistence. See `cluster.md`.
- **Do not share an etcd cluster across environments without setting `Options.Cluster`** - an empty `Cluster` falls back to the shared `"default"` namespace, so registrations from different environments collide and cause accidental cross-env calls.
- **Do not rely on `saturn.RegisterProxy`** - it returns `gen.ErrUnsupported` even though `Info()` advertises proxy support. `ResolveProxy` works.
- **Do not use `colored` logger in production** - terminal color codes pollute log aggregation; also slower.
- **Do not set `rotate.Period` below 1 minute** - it is clamped up to 1 minute by the library.
- **Do not forget `TimeFormat`** if you rely on log timestamps for correlation - default is raw nanosecond integer.
- **Do not treat Sentry as your only logger** - it forwards only panic and error; keep a file or console logger for the full stream.
- **Do not disable `DefaultLogger`** and forget to configure a replacement - the node will run without any logger output.
- **Do not expose `InsecureSkipVerify: true`** on Saturn or etcd in production - defeats TLS entirely.

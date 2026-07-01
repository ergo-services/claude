# Node

The node is the runtime host for actors, applications, and meta processes. Configure it via `gen.NodeOptions` at startup. Each node has a unique name `<atom>@<host>` and joins the cluster via registrar or static routes.

The `gen.Node` interface (unlike `gen.Process`) is safe to call from any goroutine. Most operations require the node to be in Running state and return `gen.ErrNodeTerminated` otherwise; read-only accessors and cleanup methods work in all states. See the `gen.Node` godoc for the exhaustive method list.

## Starting a Node

```go
import (
    "ergo.services/ergo"
    "ergo.services/ergo/gen"
)

node, err := ergo.StartNode("mynode@localhost", gen.NodeOptions{})
if err != nil {
    panic(err)
}
defer node.Stop()

node.Wait()  // block until node exits
```

`ergo.StartNode` does two things implicitly, so what you observe at runtime differs from what you configured:

- It always prepends the built-in `system` application to `NodeOptions.Applications`. `node.Applications()` therefore includes an application you did not list.
- If `NodeOptions.Version` is the zero value, it auto-fills `Name`, `Release`, and `Commit` from Go build info (`debug.ReadBuildInfo`). `node.Version()` reflects the module path and VCS revision rather than an empty struct.

## NodeOptions

```go
type NodeOptions struct {
    ShutdownTimeout time.Duration          // default 3 min (gen.DefaultShutdownTimeout)
    Applications    []ApplicationBehavior  // auto-load and start
    Env             map[Env]any            // node-level env (lowest priority)
    Network         NetworkOptions
    Cron            CronOptions
    CertManager     CertManager            // TLS (see TLS Setup)
    TargetManager   TargetManager          // custom link/monitor backend
    Security        SecurityOptions
    Log             LogOptions
    Version         Version
    Tracing         TracingOptions         // see tracing.md
    Events          []NodeEventSpec        // node-level events registered before apps
}
```

### Node-level events

`Events` declares event buses that are registered with the node as producer **before any application starts**. Each declared event has `Open` forced to true and lives until the node stops. Use this to establish node-wide pub/sub buses that application processes can subscribe to from `Init()` without racing a producer process to call `RegisterEvent` first.

```go
type NodeEventSpec struct {
    Name   Atom  // unique within the node event namespace
    Buffer int   // ring-buffer size for recent MessageEvent values; 0 = no buffer
}
```

```go
Events: []gen.NodeEventSpec{
    {Name: "cluster.topology", Buffer: 16},
},
```

See the "Events (Pub/Sub)" section in `actors.md` for subscribing (`LinkEvent` / `MonitorEvent`) and producing.

## NetworkOptions

```go
type NetworkOptions struct {
    Mode               NetworkMode       // Enabled, Hidden, Disabled
    Cookie             string            // shared auth secret
    Flags              NetworkFlags      // capability toggles (see below)
    Registrar          Registrar         // dynamic discovery (see cluster.md)
    Handshake          NetworkHandshake  // custom auth protocol
    Proto              NetworkProto      // EDF (default) or Erlang
    Acceptors          []AcceptorOptions // listeners
    InsecureSkipVerify bool              // skip TLS cert verification (dev only)
    MaxMessageSize     int               // 0 = unlimited
    FragmentSize       int               // 0 = 65000 bytes
    FragmentTimeout    int               // seconds; 0 = 30s
    MaxFragmentAssemblies int            // 0 = 1000
    ProxyAccept        ProxyAcceptOptions
    ProxyTransit       ProxyTransitOptions
}
```

| NetworkMode | Behavior |
|-------------|----------|
| `NetworkModeEnabled` | Full networking (default). Accept and dial. |
| `NetworkModeHidden` | Dial out only. No acceptors; invisible to incoming connections. |
| `NetworkModeDisabled` | Networking off. Local-only node. |

## NetworkFlags

`NetworkFlags` is negotiated during handshake and both sides must agree. The gotcha is how the "default" is chosen, which is the opposite of the Go struct zero value:

- **Leave `Flags` zero (`Enable: false`)** and the framework replaces the whole struct with `gen.DefaultNetworkFlags`. So an out-of-the-box node has remote spawn, remote application start, fragmentation, proxy-accept, important delivery, simultaneous-connect, clock skew, tracing, and wrapped errors all **on**, and software keepalive at 15 seconds.
- **Set `Enable: true`** to customize, and you are now responsible for every flag. Any flag you do not set stays `false`. Setting `Enable: true` alone turns everything else off.

The table below shows the value you get when you leave `Flags` zero (the `gen.DefaultNetworkFlags` column), not the Go zero value.

| Flag | Default (Flags zero) | Purpose |
|------|----------------------|---------|
| `Enable` | true | Set `true` yourself to customize; then unset flags become `false` |
| `EnableRemoteSpawn` | true | Accept `RemoteSpawn` requests from peers |
| `EnableRemoteApplicationStart` | true | Accept remote application start |
| `EnableFragmentation` | true | Split messages larger than the fragment size |
| `EnableProxyTransit` | false | Relay connections on behalf of other nodes |
| `EnableProxyAccept` | true | Accept proxied incoming connections |
| `EnableImportantDelivery` | true | Support `SendImportant` / `CallImportant` |
| `EnableSimultaneousConnect` | true | Resolve simultaneous-connect races |
| `EnableClockSkew` | true | Measure clock drift (both sides must enable) |
| `EnableTracing` | true | Propagate trace context (both sides must enable) |
| `EnableWrappedErrors` | true | Structured `*gen.Error` wire format preserving the wrap chain; if either side disables it, `*gen.Error` degrades to a flat `.Error()` string (both sides must enable) |
| `EnableSchemaEvolution` | false | Length-prefix encoded structs so peers with different struct field counts interoperate (extra trailing fields skipped, missing ones zero-valued); both sides must enable; caps an encoded struct at 4 GB |
| `EnableSoftwareKeepAlive` | 15 | Seconds between app-level heartbeats (0 = off, max 255) |

`EnableProxyTransit` and `EnableSchemaEvolution` are the two flags that stay off even with the framework defaults.

The flag layer is not the security gate for remote spawn. A default node accepts remote spawn requests at the flag level, but the actual gate is the per-factory allowlist installed via `Network().EnableSpawn(...)` (see below) - without that, no factory is spawnable regardless of the flag.

## AcceptorOptions

```go
type AcceptorOptions struct {
    Cookie             string       // per-acceptor cookie (empty = node default)
    Host               string       // interface, e.g. "0.0.0.0"
    Port               uint16       // default 11144
    PortRange          uint16       // 0 = try all ports up to 65535; 1 = only Port; N = N ports
    RouteHost          string       // advertised public host (for NAT)
    RoutePort          uint16       // advertised public port
    TCP                string       // "tcp4" (default), "tcp6", "tcp"
    BufferSize         int          // TCP buffer size; 0 = system default
    MaxMessageSize     int          // overrides NetworkOptions.MaxMessageSize
    Flags              NetworkFlags // per-acceptor capabilities
    CertManager        CertManager  // per-acceptor TLS (nil falls back to node CertManager)
    InsecureSkipVerify bool         // per-acceptor cert-verification skip
}
```

Multiple acceptors allow different interfaces, cookies, capabilities, or TLS per listener. A per-acceptor `CertManager` overrides the node-level `NodeOptions.CertManager`; when nil, the acceptor falls back to the node manager. Setting `CertManager` on one acceptor and leaving it nil on another lets a single node expose both plain and TLS listeners.

## SecurityOptions

```go
type SecurityOptions struct {
    ExposeEnvInfo                   bool  // include env in Info() responses
    ExposeEnvRemoteSpawn            bool  // remote spawns inherit env
    ExposeEnvRemoteApplicationStart bool  // remote app starts see env
}
```

Default: all `false` (most secure). Enable only when debugging or when a remote deployment pipeline needs env inheritance.

## LogOptions

```go
type LogOptions struct {
    Level         LogLevel             // default LogLevelInfo
    DefaultLogger DefaultLoggerOptions // built-in console logger
    Loggers       []Logger             // custom loggers (colored, rotate, etc.)
}
```

See `logging.md` for logger implementations and levels.

## Core Node Methods

### Spawning
```go
pid, err := node.Spawn(factory, gen.ProcessOptions{}, args...)
pid, err := node.SpawnRegister("name", factory, gen.ProcessOptions{}, args...)
```

### Messaging
```go
err := node.Send(target, msg)
err := node.SendWithPriority(target, msg, gen.MessagePriorityHigh)
resp, err := node.Call(target, req)
resp, err := node.CallWithTimeout(target, req, 10)  // seconds
resp, err := node.CallImportant(target, req)
```

`target` may be a `PID`, `ProcessID`, `Alias`, `Atom`, or `string`. The node's core PID (`node.PID()`) is the sender for all node-level sends and calls.

### Applications
```go
name, err := node.ApplicationLoad(app, args...)
err := node.ApplicationStart(name, gen.ApplicationOptions{})
err := node.ApplicationStop(name)
info, err := node.ApplicationInfo(name)
apps := node.Applications()          // []Atom (loaded + started)
running := node.ApplicationsRunning() // []Atom
```

### Network
```go
network := node.Network()
err := node.NetworkStart(gen.NetworkOptions{...})  // if started with network off
err := node.NetworkStop()
```

### Remote Spawn and Remote Application Start (Target-Side Permissions)

For a remote node to spawn a process on this node via `process.RemoteSpawn` / `RemoteNode.Spawn`, two things must be true on **this** node:

1. `NetworkFlags.EnableRemoteSpawn` must be on. It is on by default (see NetworkFlags above); if you set `Enable: true` you must set it explicitly.
2. The factory is registered via `Network().EnableSpawn(...)`. This is the real security gate - nothing is spawnable remotely until you allow it here.

```go
// Allow any node to spawn "worker"
err := node.Network().EnableSpawn("worker", createWorker)

// Restrict to specific nodes
err := node.Network().EnableSpawn("worker", createWorker,
    "scheduler@n1", "scheduler@n2")

// Remove a node from the allowlist
err := node.Network().DisableSpawn("worker", "scheduler@n2")

// Remove the factory entirely
err := node.Network().DisableSpawn("worker")
```

Calling `EnableSpawn` again updates the allowlist. The factory must be the same type - `EnableSpawn` returns `gen.ErrTaken` if the name is already registered with a different factory.

For remote application start, the symmetric API (flag `EnableRemoteApplicationStart`, on by default):

```go
err := node.Network().EnableApplicationStart("orders")               // any node
err := node.Network().EnableApplicationStart("orders", "admin@host")  // allowlist
err := node.Network().DisableApplicationStart("orders")
```

### Discovery
```go
registrar, err := node.Network().Registrar()
routes, err := registrar.Resolver().Resolve("other@host")
appRoutes, err := registrar.Resolver().ResolveApplication("orders")
```

See `cluster.md` for registrars, routes, tags, and proxy topologies.

### Introspection

All three of these require Running state and return an error (`gen.ErrNodeTerminated` when not Running):

```go
info, err := node.Info()             // NodeInfo: uptime, counters, memory, etc.
pinfo, err := node.ProcessInfo(pid)  // per-process details
plist, err := node.ProcessList()     // []PID of all processes
pname, err := node.ProcessName(pid)  // registered name or empty Atom
```

`Inspect` and `InspectMeta` synchronously query a **local** process or meta process for its self-reported inspection items (the same items Observer and MCP surface). They return `map[string]string` and are local-only - a remote target returns `gen.ErrNotAllowed`.

```go
items, err := node.Inspect(pid, "state", "mailbox")
metaItems, err := node.InspectMeta(alias)
```

### Node Events

The node is itself an event producer. Register a bus, then send to all subscribers. Note the node-level `SendEvent` takes an extra `MessageOptions` argument compared to the three-argument process-level `SendEvent`.

```go
ref, err := node.RegisterEvent("cluster.topology", gen.EventOptions{Buffer: 16})
err = node.SendEvent("cluster.topology", ref, gen.MessageOptions{}, MessageTopologyChanged{...})
err = node.UnregisterEvent("cluster.topology")

info, err := node.EventInfo(gen.Event{Name: "cluster.topology", Node: node.Name()})
events, err := node.EventListInfo(0, 10)  // 0 = oldest first, limit 10
```

To register buses before applications start (so subscribers cannot race the producer), declare them in `NodeOptions.Events` instead. Subscribing is covered in `actors.md`.

### Loggers and Tracing (runtime)

Loggers and tracing exporters can be registered at runtime, not just via `NodeOptions.Log` / `NodeOptions.Tracing`:

```go
err := node.LoggerAdd("audit", auditLogger, gen.LogLevelWarning, gen.LogLevelError)
err = node.LoggerAddPID(pid, "audit")   // route logs to a process
node.LoggerDelete("audit")

err = node.TracingExporterAdd("otlp", exporter, gen.TracingFlags{})
err = node.TracingExporterAddPID(pid, "otlp", gen.TracingFlags{})
node.TracingExporterDelete("otlp")

err = node.SetTracingSampler(sampler)              // node-level Send/Call sampling
err = node.SetProcessTracingSampler(pid, sampler)  // per-process
```

A logger name prefixed with `.` is a **hidden logger**: it is excluded from fan-out and receives logs only from processes that explicitly select it via `SetLogger(name)`. Use hidden loggers to peel one process's logs into a separate stream. See `logging.md` and `tracing.md` for the exporter and sampler details.

### Lifecycle
```go
node.Stop()                                      // graceful, honors ShutdownTimeout
node.StopWithTimeout(30 * time.Second)           // graceful, custom deadline
node.StopForce()                                 // immediate, no graceful shutdown
node.Wait()                                       // block until terminated
err := node.WaitWithTimeout(60 * time.Second)     // ErrTimeout if still running
err := node.Kill(pid)                             // hard-kill a local process
```

SIGTERM triggers graceful node shutdown automatically - the framework enables signal handling at startup, so the production template already honors `ShutdownTimeout` on SIGTERM with no extra code. Call `node.SetCTRLC(false)` to opt out when the application installs its own signal handler.

## Cron

```go
cron := node.Cron()
err := cron.AddJob(gen.CronJob{
    Name: "midnight-sweep",
    Spec: "0 0 * * *",
    Action: gen.CreateCronActionMessage("scheduler", gen.MessagePriorityNormal),
})
err = cron.RemoveJob("midnight-sweep")
err = cron.EnableJob("midnight-sweep")
err = cron.DisableJob("midnight-sweep")
info, err := cron.JobInfo("midnight-sweep")
```

A cron action is one of three: send a message (`CreateCronActionMessage`), spawn a local process (`CreateCronActionSpawn`), or spawn a remote process (`CreateCronActionRemoteSpawn`). There is no "call" action. Jobs registered at startup go in `NodeOptions.Cron.Jobs`.

## TLS Setup

The framework ships `gen.CreateCertManager`, so you do not implement `gen.CertManager` yourself. Load a `tls.Certificate` and hand it to the constructor:

```go
cert, err := tls.LoadX509KeyPair("node.pem", "node-key.pem")
if err != nil {
    panic(err)
}
cm := gen.CreateCertManager(cert)

node, err := ergo.StartNode("secure@host", gen.NodeOptions{
    CertManager: cm,
    Network: gen.NetworkOptions{
        Acceptors: []gen.AcceptorOptions{
            {Host: "0.0.0.0", Port: 11145},
        },
    },
})
```

When `CertManager` is non-nil the node enables TLS on its acceptors and presents this certificate on outgoing connections. The framework pins the minimum TLS version to 1.2 on both acceptor and dialer. `InsecureSkipVerify: true` bypasses certificate validation (dev only). For a self-signed dev certificate, `lib.GenerateSelfSignedCert("name")` returns a `tls.Certificate` you can pass to `CreateCertManager`.

### Certificate rotation

`node.CertManager()` is the runtime accessor. Because acceptors and dialers read the certificate through `GetCertificateFunc()` on every handshake, calling `Update` hot-swaps the certificate with no node restart - new connections pick it up immediately, existing connections keep the certificate they were established with:

```go
newCert, _ := tls.LoadX509KeyPair("new.pem", "new-key.pem")
node.CertManager().Update(newCert)
```

### Mutual TLS

For mutual TLS (both sides present and verify certificates), use `gen.CreateCertAuthManager`, which returns a `gen.CertAuthManager` extending `CertManager` with CA-pool and authentication settings. Pass it as `NodeOptions.CertManager`; the framework type-asserts to `CertAuthManager` and applies the mTLS settings to acceptors and dialers automatically.

```go
cert, _ := tls.LoadX509KeyPair("node.pem", "node-key.pem")
caPEM, _ := os.ReadFile("cluster-ca.pem")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(caPEM)

cam := gen.CreateCertAuthManager(cert)
cam.SetClientCAs(caPool)                          // server: verify client certs
cam.SetClientAuth(tls.RequireAndVerifyClientCert) // server: require + verify client cert
cam.SetRootCAs(caPool)                            // client: verify server certs
cam.SetServerName("node.cluster.local")           // client: SNI / verification name

node, err := ergo.StartNode("secure@host", gen.NodeOptions{CertManager: cam})
```

| Side | Method | Purpose |
|------|--------|---------|
| Server | `SetClientCAs` | CA pool used to verify client certificates |
| Server | `SetClientAuth` | Client-cert policy (see below) |
| Client | `SetRootCAs` | CA pool used to verify server certificates |
| Client | `SetServerName` | Server name for SNI and verification |

**Gotcha:** `CreateCertAuthManager` defaults `ClientAuth` to `tls.NoClientCert`, so simply constructing the manager does **not** enforce client certificates. You must call `SetClientAuth(tls.RequireAndVerifyClientCert)` (or another non-`NoClientCert` value) to enforce mTLS on the server side.

`ClientAuth` values, weakest to strongest: `tls.NoClientCert`, `tls.RequestClientCert`, `tls.RequireAnyClientCert`, `tls.VerifyClientCertIfGiven`, `tls.RequireAndVerifyClientCert`.

The certificate itself rotates via `Update` at runtime as above. The server-side `ClientAuth` and `ClientCAs` are captured when the acceptor is created, so changing those after start requires a node restart.

## Production Node Template

```go
node, err := ergo.StartNode("orders@node1.prod.example", gen.NodeOptions{
    ShutdownTimeout: 30 * time.Second,
    Applications: []gen.ApplicationBehavior{
        createOrdersApp(),
    },
    Network: gen.NetworkOptions{
        Mode:   gen.NetworkModeEnabled,
        Cookie: os.Getenv("CLUSTER_COOKIE"),
        Flags: gen.NetworkFlags{
            Enable:                  true,   // customizing: every flag now explicit
            EnableImportantDelivery: true,
            EnableTracing:           true,
            EnableWrappedErrors:     true,
            EnableSoftwareKeepAlive: 15,
        },
        Registrar: etcd.Create(etcd.Options{
            Endpoints: []string{"etcd1:2379", "etcd2:2379"},
        }),
        Acceptors: []gen.AcceptorOptions{
            {Host: "0.0.0.0", Port: 11144},
        },
    },
    Security: gen.SecurityOptions{},  // all false
    Log: gen.LogOptions{
        Level: gen.LogLevelInfo,
        Loggers: []gen.Logger{
            {Name: "rotate", Logger: rotate.CreateLogger(rotate.Options{...})},
        },
    },
})
```

Note the trade-off with `Flags`: setting `Enable: true` means you opt every capability in by hand. If you leave `Flags` zero instead, you get the full `gen.DefaultNetworkFlags` set (remote spawn, important delivery, tracing, wrapped errors, keepalive=15, and more) without listing them - but then the per-factory `EnableSpawn` allowlist is still what actually gates remote spawn.

## Anti-Patterns

- **Do not assume zero `Flags` means everything is off.** Zero `Flags` means `gen.DefaultNetworkFlags` (most capabilities on). If you set `Enable: true`, every capability you want must be set explicitly.
- **Do not rely on `NetworkFlags` as the remote-spawn security gate.** The gate is `Network().EnableSpawn` - without it nothing is spawnable, flags notwithstanding.
- **Do not construct a `CertAuthManager` and assume mTLS is enforced.** `ClientAuth` defaults to `tls.NoClientCert`; call `SetClientAuth` explicitly.
- **Do not copy introspection code that ignores errors.** `Info`, `ProcessList`, and `ProcessName` all return an error and require Running state.
- **Do not expose env variables in production** - leave `SecurityOptions` zero.
- **Do not share cookies across environments** - dev, staging, prod each get their own.
- **Do not use `InsecureSkipVerify` in production** - defeats the purpose of TLS.
- **Do not skip `ShutdownTimeout`** - default is 3 minutes; long-running apps need to cleanly shut down or force-exit.
- **Do not start the node without `Registrar` in a cluster** - the embedded registrar is single-process, no persistence or failover.
- **Do not mix `NetworkModeHidden` nodes with proxy transit** - hidden nodes by design do not accept; proxying them requires explicit dial-through config.

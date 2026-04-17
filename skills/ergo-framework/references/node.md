# Node

The node is the runtime host for actors, applications, and meta processes. Configure it via `gen.NodeOptions` at startup. Each node has a unique name `<atom>@<host>` and joins the cluster via registrar or static routes.

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

## NodeOptions

```go
type NodeOptions struct {
    ShutdownTimeout time.Duration          // default 3 min
    Applications    []ApplicationBehavior  // auto-load and start
    Env             map[Env]any            // node-level env (lowest priority)
    Network         NetworkOptions
    Cron            CronOptions
    CertManager     CertManager            // TLS
    TargetManager   TargetManager          // custom link/monitor backend
    Security        SecurityOptions
    Log             LogOptions
    Version         Version
    Tracing         TracingOptions
}
```

## NetworkOptions

```go
type NetworkOptions struct {
    Mode              NetworkMode          // Enabled, Hidden, Disabled
    Cookie            string               // shared auth secret
    Flags             NetworkFlags         // capability toggles
    Registrar         Registrar            // dynamic discovery
    Handshake         NetworkHandshake     // custom auth protocol
    Proto             NetworkProto         // EDF (default) or Erlang
    Acceptors         []AcceptorOptions    // listeners
    InsecureSkipVerify bool
    MaxMessageSize    int
    FragmentSize      int                  // split messages > 65000 bytes
    FragmentTimeout   int
    MaxFragmentAssemblies int
    SoftwareKeepAliveMisses int
    ProxyAccept       ProxyAcceptOptions
    ProxyTransit      ProxyTransitOptions
}
```

| NetworkMode | Behavior |
|-------------|----------|
| `NetworkModeEnabled` | Full networking (default). Accept and dial. |
| `NetworkModeHidden` | Dial out only. No acceptors; invisible to incoming connections. |
| `NetworkModeDisabled` | Networking off. Local-only node. |

## NetworkFlags

All optional; set `Enable: true` at the top of `NetworkFlags` to make the framework honour customized flag values (otherwise framework defaults are used). Both sides must agree (negotiated during handshake).

| Flag | Default | Purpose |
|------|---------|---------|
| `Enable` | false | Must be `true` to activate non-default flag values below |
| `EnableRemoteSpawn` | false | Accept `RemoteSpawn` requests from peers |
| `EnableRemoteApplicationStart` | false | Accept remote `ApplicationStart` |
| `EnableFragmentation` | true | Split messages > 65000 bytes |
| `EnableProxyTransit` | false | Relay connections for other nodes |
| `EnableProxyAccept` | false | Accept proxied incoming connections |
| `EnableImportantDelivery` | false | Support `SendImportant` / `CallImportant` |
| `EnableSimultaneousConnect` | false | Resolve simultaneous-connect races |
| `EnableClockSkew` | false | Measure clock drift (both sides) |
| `EnableTracing` | false | Propagate trace context (both sides) |
| `EnableSoftwareKeepAlive` | 0 | Seconds between app-level heartbeats (0=off, max 255) |

## AcceptorOptions

```go
type AcceptorOptions struct {
    Cookie    string  // per-acceptor cookie (empty = use node default)
    Host      string  // interface, e.g. "0.0.0.0"
    Port      uint16  // default 11144
    PortRange uint16  // try N ports starting from Port
    RouteHost string  // advertised public host (for NAT)
    RoutePort uint16  // advertised public port
}
```

Multiple acceptors allow different interfaces or different authentication per listener.

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

### Applications
```go
name, err := node.ApplicationLoad(app, args...)
err := node.ApplicationStart(name, gen.ApplicationOptions{})
err := node.ApplicationStop(name)
info, err := node.ApplicationInfo(name)
apps := node.Applications()
running := node.ApplicationsRunning()
```

### Network
```go
network := node.Network()
err := node.NetworkStart(gen.NetworkOptions{...})  // if started with network off
err := node.NetworkStop()
```

### Remote Spawn and Remote Application Start (Target-Side Permissions)

For a remote node to spawn a process on this node via `process.RemoteSpawn` / `RemoteNode.Spawn`, two things must be true on **this** node:

1. `NetworkFlags.EnableRemoteSpawn = true` in `NetworkOptions`.
2. The factory is explicitly enabled via `Network().EnableSpawn(...)`.

```go
// Allow any node to spawn "worker"
err := node.Network().EnableSpawn("worker", createWorker)

// Restrict to specific nodes
err := node.Network().EnableSpawn("worker", createWorker,
    "scheduler@n1", "scheduler@n2")

// Remove a node from the ACL
err := node.Network().DisableSpawn("worker", "scheduler@n2")

// Remove the factory entirely
err := node.Network().DisableSpawn("worker")
```

Calling `EnableSpawn` again updates the ACL. The factory must be the same type — you cannot swap factories under the same name.

For remote application start, the symmetric API:

```go
// Flag: NetworkFlags.EnableRemoteApplicationStart = true
err := node.Network().EnableApplicationStart("orders")         // any node
err := node.Network().EnableApplicationStart("orders", "admin@host")  // ACL
err := node.Network().DisableApplicationStart("orders")
```

`EnableSpawn` returns `gen.ErrTaken` if the name is already registered with a different factory.

### Discovery
```go
registrar, err := node.Network().Registrar()
routes, err := registrar.Resolver().Resolve("other@host")
appRoutes, err := registrar.Resolver().ResolveApplication("orders")
```

### Introspection
```go
info := node.Info()                              // node-level metrics
pinfo, err := node.ProcessInfo(pid)              // process details
plist := node.ProcessList()                      // all PIDs
pname := node.ProcessName(pid)                   // registered name or ""
```

### Lifecycle
```go
node.Stop()                                      // graceful, no return value
node.StopForce()                                 // immediate, no return value
err := node.WaitWithTimeout(60 * time.Second)
err := node.Kill(pid)                            // hard-kill a process
```

## Cron

```go
cron := node.Cron()
err := cron.AddJob(gen.CronJob{
    Name: "midnight-sweep",
    Spec: "0 0 * * *",
    Action: gen.CreateCronActionMessage("scheduler", gen.MessagePriorityNormal),
})
err := cron.RemoveJob("midnight-sweep")
err := cron.EnableJob("midnight-sweep")
err := cron.DisableJob("midnight-sweep")
info, err := cron.JobInfo("midnight-sweep")
```

Cron actions can send messages, calls, or spawn processes. See `CreateCronAction*` helpers in `gen`.

## TLS Setup

```go
certManager := <implementation>      // your gen.CertManager

opts := gen.NodeOptions{
    CertManager: certManager,
    Network: gen.NetworkOptions{
        Acceptors: []gen.AcceptorOptions{
            {Host: "0.0.0.0", Port: 11145},
        },
    },
}
```

When `CertManager` is non-nil, the node enables TLS on listener sockets. Both sides must present valid certificates; `InsecureSkipVerify: true` bypasses cert validation (dev only).

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
            Enable:                  true,
            EnableImportantDelivery: true,
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

## Anti-Patterns

- **Do not expose env variables in production** — leave `SecurityOptions` zero.
- **Do not share cookies across environments** — dev, staging, prod each get their own.
- **Do not use `InsecureSkipVerify` in production** — defeats the purpose of TLS.
- **Do not skip `ShutdownTimeout`** — default is 3 minutes; long-running apps need to cleanly shut down or force-exit.
- **Do not start the node without `Registrar` in a cluster** — static routes break on IP changes.
- **Do not mix `NetworkModeHidden` nodes with proxy transit** — hidden nodes by design don't accept; proxying them requires explicit dial-through config.

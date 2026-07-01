# Cluster

A cluster is a set of Ergo nodes that share a cookie and a registrar. Each node advertises its listeners to the registrar; peers resolve those advertisements to connect. Once connected, actors talk to remote peers with the same network-transparent `Send` / `Call` they use locally.

This reference covers registrar selection, the `Registrar` and `Resolver` interfaces, resolving nodes and applications, tag- and state-based selection, cluster-change events, static routes, and proxy topologies. For reacting to peers going up and down at the process level, see the CoreEvent section in `actors.md`.

## Registrar Options

| Registrar | Package | Scale | License | When |
|-----------|---------|-------|---------|------|
| Embedded (built-in) | none (default) | dev only | - | Local development, single-machine testing |
| etcd | `ergo.services/registrar/etcd` | up to ~70 nodes | open source | Small-to-medium clusters with an existing etcd |
| Saturn | `ergo.services/registrar/saturn` | 100+ nodes | commercial | Large clusters, dedicated registrar service |

When `NetworkOptions.Registrar` is nil the node uses the built-in embedded registrar. It tries to start a small registrar server on `localhost:4499`; if the port is taken it connects to the one already there as a client. Same-host nodes discover each other through that local server; cross-host resolution is a best-effort UDP query to `host:4499`. It resolves node routes only - it does not support application routes, proxy registration, centralized config, events, or `Nodes()`. Use it for development, never for production.

## Registrar Interface

Obtain the registrar from the node, then use it directly or reach for its `Resolver`:

```go
registrar, err := node.Network().Registrar()
if err != nil {
    return err  // gen.ErrNetworkStopped if the node's network is not running
}
resolver := registrar.Resolver()
```

The user-facing methods:

| Method | Returns | Purpose |
|--------|---------|---------|
| `Resolver()` | `Resolver` | Node/proxy/application route discovery (see below) |
| `Nodes()` | `([]gen.Atom, error)` | List every node currently registered in the cluster |
| `Config(items ...string)` | `(map[string]any, error)` | Read centralized config values (gated by `SupportConfig`) |
| `ConfigItem(item string)` | `(any, error)` | Read a single centralized config value (gated by `SupportConfig`) |
| `Event()` | `(gen.Event, error)` | Token to Link/Monitor for cluster-change notifications (gated by `SupportEvent`) |
| `RegisterProxy(to gen.Atom)` | `error` | Advertise this node as a proxy for `to` (gated by `SupportRegisterProxy`) |
| `UnregisterProxy(to gen.Atom)` | `error` | Remove that proxy advertisement |
| `Info()` | `RegistrarInfo` | Capability flags and connection details |
| `Version()` | `gen.Version` | Registrar implementation version |

`Register` / `RegisterApplicationRoute` / `UnregisterApplicationRoute` / `Terminate` exist on the interface but are driven by the node runtime, not application code. See the `gen.Registrar` godoc for the exhaustive list.

Every optional capability returns `gen.ErrUnsupported` when the registrar does not implement it. Check `Info()` before relying on one.

### Capability flags (RegistrarInfo)

`registrar.Info()` returns a `RegistrarInfo` describing what the backing registrar can do:

| Field | Type | Meaning |
|-------|------|---------|
| `Server` | `string` | Registrar server address (for example `localhost:2379` for etcd) |
| `EmbeddedServer` | `bool` | True when the registrar server runs inside this node (Saturn, embedded) |
| `SupportRegisterProxy` | `bool` | `RegisterProxy` / proxy resolution are available |
| `SupportRegisterApplication` | `bool` | Application routes can be advertised and resolved |
| `SupportConfig` | `bool` | `Config` / `ConfigItem` are available |
| `SupportEvent` | `bool` | `Event()` returns a live token |
| `Version` | `gen.Version` | Registrar implementation version |

Capabilities of the shipping registrars:

| Capability | Embedded | etcd | Saturn |
|------------|----------|------|--------|
| Resolve node routes | yes | yes | yes |
| `Nodes()` | no | yes | yes |
| Application routes | no | yes | yes |
| `Config` / `ConfigItem` | no | yes | yes |
| `Event()` | no | yes | yes |
| Proxy registration | no | no | flag reports yes, method not yet implemented |

## Resolver

```go
type Resolver interface {
    Resolve(node gen.Atom) ([]Route, error)
    ResolveProxy(node gen.Atom) ([]ProxyRoute, error)
    ResolveApplication(name gen.Atom) (ApplicationRoutes, error)
}
```

### Route

`Resolve("othernode@host")` returns every candidate route to a node - one per advertised listener:

```go
type Route struct {
    Host             string
    Port             uint16
    TLS              bool
    HandshakeVersion gen.Version
    ProtoVersion     gen.Version
}
```

You rarely call `Resolve` yourself. `node.Network().GetNode(name)` resolves and connects in one step, and `Send` / `Call` to a remote `gen.ProcessID` connects on demand.

### ApplicationRoute and ApplicationRoutes

`ResolveApplication` reports which nodes run an application, so a caller can pick a target:

```go
type ApplicationRoute struct {
    Node   gen.Atom
    Name   gen.Atom
    Weight int
    Mode   ApplicationMode
    Tags   []gen.Atom
    State  ApplicationState  // Loaded, Initializing, Running, Stopping
}
```

The return value is `ApplicationRoutes` - a named slice (`[]ApplicationRoute`) with three chainable filter methods. Use them instead of hand-rolling loops:

| Method | Keeps routes that... |
|--------|----------------------|
| `WithTags(tags ...gen.Atom)` | carry ALL of the given tags |
| `WithoutTags(tags ...gen.Atom)` | carry NONE of the given tags (excluded if they have ANY) |
| `WithState(states ...ApplicationState)` | are in one of the given states |

Each returns a new `ApplicationRoutes`, so calls chain. Calling any of them with no arguments is a no-op.

```go
routes, err := registrar.Resolver().ResolveApplication("orders")
if err != nil {
    return err
}
routes = routes.
    WithState(gen.ApplicationStateRunning).
    WithTags("production").
    WithoutTags("maintenance")
if len(routes) == 0 {
    return gen.ErrNoRoute
}
target := routes[0]  // caller picks; see weighting below
```

## Configuration

### etcd

`etcd.Create` returns `(gen.Registrar, error)` - build it first, check the error, then pass the value into `NetworkOptions`. Inlining it as a single expression does not compile.

```go
import "ergo.services/registrar/etcd"

reg, err := etcd.Create(etcd.Options{
    Cluster:   "prod",
    Endpoints: []string{"etcd1:2379", "etcd2:2379"},
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

`etcd.Options` defaults: `Endpoints` falls back to `localhost:2379`, `Cluster` to `default`, `DialTimeout` and `RequestTimeout` to 10s, `LeaseTTL` to 10s. See `integrations.md` for the full option list.

### Saturn

`saturn.Create` returns a single `gen.Registrar` (no error), so its inline use in `NetworkOptions` is correct:

```go
import "ergo.services/registrar/saturn"

node, err := ergo.StartNode("orders@host1", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: saturn.Create("saturn-host", "token", saturn.Options{
            Cluster:   "prod",
            Port:      4499,  // default 4499
            KeepAlive: 30 * time.Second,
        }),
        Cookie: os.Getenv("CLUSTER_COOKIE"),
    },
})
```

## Tag- and State-Based Service Discovery

Applications publish `Tags` at startup (see `application.md`) and the runtime keeps each route's `State` current. Callers filter routes to pick deployment variants without knowing node names in advance.

```go
func (g *Gateway) callProductionAPI(req any) (any, error) {
    registrar, err := g.Node().Network().Registrar()
    if err != nil {
        return nil, err
    }
    routes, err := registrar.Resolver().ResolveApplication("orders")
    if err != nil {
        return nil, err
    }
    routes = routes.WithState(gen.ApplicationStateRunning).WithTags("production")
    if len(routes) == 0 {
        return nil, gen.ErrNoRoute
    }
    r := routes[0]
    return g.Call(gen.ProcessID{Node: r.Node, Name: "orders-api"}, req)
}
```

`ApplicationRoute` carries the node, weight, mode, tags, and state - enough to choose a target - but not the application's process `Map`. There are two ways to address the concrete process on the chosen node:

- Address a process by a name known by convention (`orders-api` above). Use this when roles are stable and the coupling cost is low.
- Look the role up through the application's published `Map` when authors want freedom to rename processes without breaking callers:

```go
remote, err := g.Node().Network().GetNode(r.Node)
if err != nil {
    return nil, err
}
info, err := remote.ApplicationInfo("orders")
if err != nil {
    return nil, err
}
name, found := info.Map["api"]
if found == false {
    return nil, gen.ErrUnknown
}
return g.Call(gen.ProcessID{Node: r.Node, Name: name}, req)
```

### Blue/Green, canary, maintenance

| Pattern | Tags | Selection |
|---------|------|-----------|
| Blue/Green cutover | `"blue"`, `"green"` | `WithTags("green")` selects the active color; switch by changing the caller's tag |
| Canary rollout | `"stable"`, `"canary"` | route a share to `WithTags("canary")`, fall back to `WithTags("stable")` |
| Maintenance drain | `"production"`, `"maintenance"` | `WithoutTags("maintenance")` skips draining instances |

Combine with `WithState(gen.ApplicationStateRunning)` so you never target an instance that is still initializing or already stopping.

### Weighting

`ApplicationRoute.Weight` biases selection, but the framework does not auto-balance `Send` / `Call` - the caller decides which route to use, normally `routes[0]`. How the slice is ordered depends on the registrar:

- The **etcd** resolver reorders results per call with smooth weighted round-robin (Nginx-style): the winner is placed at index 0 and the remainder ordered by preference for fallback. Weights `<= 0` are treated as 1. So repeatedly calling `ResolveApplication` and taking `routes[0]` distributes load by weight.
- The **Saturn** client returns routes unordered; a caller that wants weighting must implement it.

Because filtering (`WithTags`/`WithState`) reslices in place, apply filters and then take `routes[0]` to keep the etcd rotation intact.

## Reacting to Cluster Changes

When `Info().SupportEvent` is true, `registrar.Event()` returns a `gen.Event` token. Link or monitor it from a process to receive a message whenever cluster membership, config, application deployment, or proxy state changes.

```go
func (a *Watcher) Init(args ...any) error {
    registrar, err := a.Node().Network().Registrar()
    if err != nil {
        return err
    }
    token, err := registrar.Event()
    if err != nil {
        if err == gen.ErrUnsupported {
            return nil  // this registrar does not emit events
        }
        return err
    }
    _, err = a.LinkEvent(token)  // or MonitorEvent
    return err
}
```

The message payloads are registrar-specific. Both shipping registrars deliver their own package-local event types, so type-switch on the concrete registrar's types in `HandleEvent`:

```go
func (a *Watcher) HandleEvent(message gen.MessageEvent) error {
    switch ev := message.Message.(type) {
    case etcd.EventNodeJoined:
        a.Log().Info("node joined: %s", ev.Name)
    case etcd.EventNodeLeft:
        a.Log().Info("node left: %s", ev.Name)
    case etcd.EventApplicationStarted:
        a.Log().Info("app %s started on %s", ev.Name, ev.Node)
    }
    return nil
}
```

etcd publishes `etcd.EventNodeJoined` / `EventNodeLeft`, `EventConfigUpdate`, and `EventApplicationLoaded` / `Started` / `Stopping` / `Stopped` / `Unloaded`. Saturn publishes the equivalently named types in the `saturn` package. Consult each package for exact field shapes.

The `gen` package also declares a canonical, portable set - `gen.MessageRegistrarNodeJoined`, `gen.MessageRegistrarApplicationStarted`, and so on (see `gen/registrar_events.go`). These are the types used by the testing harnesses and by any registrar that opts into the portable contract; the shipping etcd and Saturn clients do not deliver them today, so guard on the concrete package types when consuming a real cluster.

## Static Routes

A registrar is the primary discovery mechanism, but for a fixed topology you can pin routes directly on the node, with or without a registrar. This overrides discovery for matching nodes.

```go
err := node.Network().AddRoute("prod-*@example.com", gen.NetworkRoute{
    Route: gen.Route{Host: "10.0.0.5", Port: 11144},
}, 100)                                    // higher weight = preferred

routes, err := node.Network().Route("prod-orders@example.com")  // ordered by weight
err = node.Network().RemoveRoute("prod-*@example.com")
```

The match pattern supports wildcards (`prod-*@example.com`). Multiple routes for the same pattern are tried in descending weight order. A `NetworkRoute` can also carry a per-route `Cookie`, `Cert`, `Flags`, and its own `Resolver`.

Static routes trade discovery for control: they survive a registrar outage but break when a host's IP changes. Prefer a registrar for anything elastic; reach for static routes only for genuinely fixed endpoints, and treat that as the intentional exception to the "never hardcode addresses" rule below.

## Proxy Routing

Proxy nodes relay connections between peers that cannot connect directly (firewall, NAT, distinct security zones). Proxying requires two things together: the negotiated `NetworkFlags` on the participating nodes, AND a registrar whose `Info().SupportRegisterProxy` is true.

| Flag | Role |
|------|------|
| `EnableProxyTransit` | This node relays on behalf of others |
| `EnableProxyAccept` | This node accepts incoming proxied connections |

A `ProxyRoute` describes a single hop, not a chain: it is one `(Proxy, To)` pair.

```go
type ProxyRoute struct {
    To    gen.Atom  // final destination
    Proxy gen.Atom  // intermediate node to connect through
}

proxyRoutes, err := resolver.ResolveProxy("target@isolated-host")
err = registrar.RegisterProxy("target@isolated-host")
```

Support status in the shipping registrars: the embedded registrar returns `gen.ErrUnsupported` for both `RegisterProxy` and `ResolveProxy`. etcd returns `gen.ErrUnsupported` from `RegisterProxy` (it reports `SupportRegisterProxy == false`) and `gen.ErrNoRoute` from `ResolveProxy`. Saturn reports `SupportRegisterProxy == true`, but its `RegisterProxy` is not yet implemented and returns `gen.ErrUnsupported`. So proxy registration is not functional through any bundled registrar today - keep the API in view, but do not build on it until a registrar ships a working implementation.

## NetworkFlags Cheat Sheet

Flags are negotiated at handshake; both sides must agree. Set them in `NetworkOptions.Flags`:

```go
Flags: gen.NetworkFlags{
    Enable:                  true,  // required to customize any flag
    EnableRemoteSpawn:       true,
    EnableImportantDelivery: true,
    EnableTracing:           true,
    EnableSoftwareKeepAlive: 15,    // seconds; 0 disables
}
```

See `node.md` for the full flag table.

## Cluster Cookie

Every node in the cluster must share the same `NetworkOptions.Cookie`. The cookie authenticates peers during handshake. Load it from a secret, never commit it:

```go
Cookie: os.Getenv("CLUSTER_COOKIE"),
```

Mismatched cookies fail the handshake and increment the acceptor's `HandshakeErrors` counter (`AcceptorInfo.HandshakeErrors`).

## Connection Pool

Each inter-node connection uses a pool of TCP connections (default 3). Messages from the **same sender** always traverse the **same** TCP connection, so per-sender ordering holds. Different senders may spread across connections in parallel.

Pool size comes from the **acceptor** (receiving side) via its `handshake.Options.PoolSize` (default 3 when unset). If node A (pool 10) dials node B (pool 5), the connection carries 5 TCP links - B's setting.

Observe pool state at runtime through `RemoteNode.Info()` (a `RemoteNodeInfo`):

| Field | Meaning |
|-------|---------|
| `PoolSize` | Target number of TCP connections |
| `PoolLen` | Currently established connections (equals `PoolSize` once fully filled) |
| `PoolDSN` | `host:port` of each pooled connection |
| `Reconnections` | Cumulative pool-item reconnects; non-zero signals instability |

A `PoolLen` stuck below `PoolSize` with a rising `Reconnections` count points to a flaky link between the two nodes.

## Anti-Patterns

- **Never** use the embedded registrar in production. It is a minimal in-process/UDP scheme with no persistence, no application routes, and no failover.
- **Never** hardcode node addresses in business logic. Resolve via `Resolver()` so topology changes need no code changes; use static routes only for genuinely fixed endpoints.
- **Never** share a cookie between dev, staging, and prod.
- **Never** omit `EnableImportantDelivery` in production - silent drops on a busy queue are painful to diagnose.
- **Do not** hand-roll tag matching - `WithTags` / `WithoutTags` / `WithState` already do it, and pairing them keeps intent clear.
- **Do not** assume `ResolveApplication` balances traffic - only the etcd resolver rotates by weight, and the caller still picks `routes[0]`.
- **Do not** design without tags - even a single-environment deployment benefits from a `"production"` marker for later operational work.
- **Do not** confuse `Route` (how to reach a node) with `ApplicationRoute` (where an application runs). They resolve independently.
- **Do not** count on ordering across different senders. Within-sender ordering is guaranteed; between-sender ordering is not.
- **Do not** call `RegisterProxy` / `ResolveProxy` on etcd or the embedded registrar - they return `gen.ErrUnsupported` / `gen.ErrNoRoute`.

# Cluster

A cluster is a set of Ergo nodes connected via a shared registrar and cookie. Nodes discover each other dynamically; actors communicate with peers via network-transparent `Send`/`Call`. This document covers registrar selection, Route resolution, Tag-based selection, and proxy topologies.

## Registrar Options

| Option | Scale | License | When |
|--------|-------|---------|------|
| Embedded | dev only | — | Local development, single-machine testing |
| etcd | ≤ 70 nodes | open source | Small-to-medium clusters with existing etcd |
| Saturn | 100+ nodes | commercial | Large clusters, dedicated registrar service |

Embedded registrar is automatic when `NetworkOptions.Registrar` is nil. Production clusters must set one.

## Registrar Interface

```go
type Registrar interface {
    Register(node NodeRegistrar, routes RegisterRoutes) (StaticRoutes, error)
    Resolver() Resolver
    RegisterProxy(to gen.Atom) error
    UnregisterProxy(to gen.Atom) error
    RegisterApplicationRoute(route ApplicationRoute) error
    UnregisterApplicationRoute(name gen.Atom) error
    Info() RegistrarInfo
    Terminate()
}
```

## Resolver

```go
type Resolver interface {
    Resolve(node gen.Atom) ([]Route, error)
    ResolveProxy(node gen.Atom) ([]ProxyRoute, error)
    ResolveApplication(name gen.Atom) ([]ApplicationRoute, error)
}
```

Obtain via:

```go
registrar, err := node.Network().Registrar()
resolver := registrar.Resolver()
```

## Route

```go
type Route struct {
    Host             string
    Port             uint16
    TLS              bool
    HandshakeVersion gen.Version
    ProtoVersion     gen.Version
}
```

A node is advertised to the cluster via one or more Routes — each listener's host/port pair. `Resolve("othernode@host")` returns all candidate routes.

## ApplicationRoute

```go
type ApplicationRoute struct {
    Node   gen.Atom
    Name   gen.Atom
    Weight int
    Mode   ApplicationMode
    Tags   []gen.Atom
    State  ApplicationState
}
```

Resolve an application to discover which nodes run it, with weights and tags for selection.

## Configuration

### etcd

```go
import "ergo.services/registrar/etcd"

node, err := ergo.StartNode("orders@host1", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: etcd.Create(etcd.Options{
            Endpoints:      []string{"etcd1:2379", "etcd2:2379"},
            RequestTimeout: 10 * time.Second,
        }),
    },
})
```

### Saturn

```go
import "ergo.services/registrar/saturn"

node, err := ergo.StartNode("orders@host1", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: saturn.Create("saturn-host", "token", saturn.Options{
            Cluster:   "prod",
            Port:      4499,
            KeepAlive: 30 * time.Second,
        }),
    },
})
```

## Tag-Based Service Discovery

Applications publish Tags at startup (see `application.md`). Callers filter routes by tag to pick deployment variants.

`ApplicationRoute` (returned by `Resolver().ResolveApplication`) carries `Node`, `Name`, `Weight`, `Mode`, `Tags`, `State` — enough to pick a target node. It does **not** carry the application's `Map`. If the caller already knows the concrete process name by convention, address it directly:

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
    for _, r := range routes {
        if containsTag(r.Tags, "production") {
            return g.Call(gen.ProcessID{
                Node: r.Node,
                Name: "orders-api",  // known by convention
            }, req)
        }
    }
    return nil, gen.ErrNoRoute
}

func containsTag(tags []gen.Atom, want gen.Atom) bool {
    for _, t := range tags {
        if t == want {
            return true
        }
    }
    return false
}
```

To resolve a role through the published `Map` instead of hard-coding the process name, fetch `ApplicationInfo` from the chosen remote node:

```go
remote, err := g.Node().Network().GetNode(r.Node)
if err != nil {
    return nil, err
}
info, err := remote.ApplicationInfo("orders")
if err != nil {
    return nil, err
}
name, ok := info.Map["api"]
if ok == false {
    return nil, gen.ErrUnknown
}
return g.Call(gen.ProcessID{Node: r.Node, Name: name}, req)
```

Use direct naming when roles are stable and the cost of coupling is low. Use the `Map` lookup when application authors want freedom to rename processes without breaking callers.

## Blue/Green and Canary Patterns

| Pattern | Tags | Client behavior |
|---------|------|-----------------|
| Blue/Green cutover | `"blue"`, `"green"` | Single active tag; switch by updating client config |
| Canary rollout | `"stable"`, `"canary"` | Route N% of traffic to `canary`; fall back to `stable` |
| Maintenance | `"production"`, `"maintenance"` | Skip `maintenance` instances for writes |

Weight can refine selection within a tag group: two `production` instances with weights 70/30 split load accordingly.

## Proxy Routing

Proxy nodes relay connections between peers that cannot connect directly (firewall, NAT, different security zones).

Enable proxy capability via `NetworkFlags`:

| Flag | Role |
|------|------|
| `EnableProxyTransit` | This node relays on behalf of others |
| `EnableProxyAccept` | This node accepts incoming proxied connections |

Resolve a proxy path:

```go
proxyRoutes, err := resolver.ResolveProxy("target@isolated-host")
// Each ProxyRoute lists intermediate hops: source → proxy1 → ... → target
```

Register this node as a proxy for a target:

```go
err := registrar.RegisterProxy("target@isolated-host")
```

Not all registrars support proxying — check `registrar.Info().SupportRegisterProxy`.

## NetworkFlags Cheat Sheet

Both sides must agree (negotiated at handshake). Setup these in `NetworkOptions.Flags`:

```go
Flags: gen.NetworkFlags{
    Enable:                  true,  // required to customize any flag
    EnableRemoteSpawn:       true,
    EnableImportantDelivery: true,
    EnableTracing:           true,
    EnableSoftwareKeepAlive: 15,    // seconds
}
```

See `node.md` for the full flag table.

## Cluster Cookie

Every node in the cluster must share the same `NetworkOptions.Cookie`. The cookie authenticates peers during handshake. Use a strong secret via env var; never commit it:

```go
Cookie: os.Getenv("CLUSTER_COOKIE"),
```

Different cookies in the same cluster → handshake failures (`HandshakeErrors` counter increments).

## Connection Pool

Each inter-node connection uses a pool of TCP connections (default 3). Messages from the **same sender** always traverse the **same TCP connection** (ordered per sender). Different senders may use different TCP connections in parallel.

Pool size is determined by the **acceptor** (receiving side) during handshake. If node A (pool=10) dials node B (pool=5), the connection has 5 TCP links (B's setting).

## Anti-Patterns

- **Never** use embedded registrar in production. It's a single-process map; no persistence, no discovery, no failover.
- **Never** hardcode node addresses in business logic. Resolve via `Resolver()` so topology changes don't require code changes.
- **Never** share a cookie between dev/staging/prod.
- **Never** omit `EnableImportantDelivery` in production — silent message drops produce nightmares.
- **Do not** design without tags — even a single-environment deployment benefits from a `"production"` marker for future operational work.
- **Do not** confuse `Route` (how to reach a node) with `ApplicationRoute` (where an application runs). They resolve independently.
- **Do not** count on order across different senders. Within-sender ordering is guaranteed; between-sender ordering is not.

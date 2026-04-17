# Application

An application bundles a supervision tree into a deployment unit with a name, version, lifecycle, and cluster-level discovery metadata. Implemented via `gen.ApplicationBehavior`.

## ApplicationBehavior

```go
type ApplicationBehavior interface {
    Load(node gen.Node, args ...any) (ApplicationSpec, error)
    Start(mode ApplicationMode)    // no error return
    Terminate(reason error)
}
```

**Important:** `Load` takes `gen.Node` as its first parameter. Older docs may omit it.

`Start` runs after the spec has been instantiated and all group members are spawned. `Terminate` runs after all members have stopped.

## ApplicationSpec

```go
type ApplicationSpec struct {
    Name        gen.Atom                // unique identifier (namespace-separate from processes)
    Description string
    Version     gen.Version
    Depends     ApplicationDepends
    Env         map[gen.Env]any         // inherited by all member processes
    Group       []ApplicationMemberSpec // members to spawn on start
    Mode        ApplicationMode
    Weight      int                     // load-balance hint for clients
    Tags        []gen.Atom              // published to registrar for discovery
    Map         map[string]gen.Atom     // logical role → registered process name
    LogLevel    gen.LogLevel
}
```

## ApplicationMemberSpec

```go
type ApplicationMemberSpec struct {
    Factory gen.ProcessFactory  // creates the process behavior
    Options gen.ProcessOptions  // spawn options
    Name    gen.Atom            // registered name
    Args    []any               // passed to Init()
}
```

## ApplicationDepends

```go
type ApplicationDepends struct {
    Applications []gen.Atom  // other apps that must run first
    Network      bool        // require network to be up
}
```

Dependency failures during `ApplicationStart` return an error.

## ApplicationMode

| Mode | Value | Behavior |
|------|-------|----------|
| `ApplicationModeTemporary` | 1 | App continues even if members exit normally or abnormally |
| `ApplicationModeTransient` | 2 | App stops if any member exits abnormally |
| `ApplicationModePermanent` | 3 | App stops if any member exits (normal or abnormal) |

**Note:** these are different from Supervisor restart strategies, which share the names `Temporary`/`Transient`/`Permanent`. Supervisor strategies control **child restart**; application mode controls **app lifecycle**.

## ApplicationState

| State | Value |
|-------|-------|
| `ApplicationStateLoaded` | 1 |
| `ApplicationStateRunning` | 2 |
| `ApplicationStateStopping` | 3 |

## Node-Level Lifecycle Methods

```go
// Load: register the spec, do NOT start members
name, err := node.ApplicationLoad(app, args...)

// Start (variants)
err := node.ApplicationStart(name, gen.ApplicationOptions{})
err := node.ApplicationStartPermanent(name, gen.ApplicationOptions{})
err := node.ApplicationStartTransient(name, gen.ApplicationOptions{})
err := node.ApplicationStartTemporary(name, gen.ApplicationOptions{})

// Stop
err := node.ApplicationStop(name)
err := node.ApplicationStopWithTimeout(name, 30*time.Second)
err := node.ApplicationStopForce(name)

// Query
info, err := node.ApplicationInfo(name)
pids, err := node.ApplicationProcessList(name)
apps := node.Applications()           // all loaded
running := node.ApplicationsRunning() // only running

// Unload (must be stopped first)
err := node.ApplicationUnload(name)
```

Auto-start via `NodeOptions.Applications` — the node loads and starts these on boot.

## Tags and Map (Service Discovery)

**Tags** mark the instance's deployment variant. Clients filter during resolution:

```go
Tags: []gen.Atom{"production", "region-eu"}
```

**Map** exposes logical roles to callers without coupling to concrete process names:

```go
Map: map[string]gen.Atom{
    "api":    "api-handler",
    "worker": "background-worker",
}
```

`Map` is not carried in `ApplicationRoute`. Callers that want to resolve a role through it must fetch `ApplicationInfo` from the target node: `remote.ApplicationInfo("orders")` → `info.Map["api"]`. See the "Tag-Based Service Discovery" section below. If the role → name mapping is stable and agreed across services, callers can also skip the lookup and address the process by its known name.

## Tag-Based Service Discovery

`Resolver().ResolveApplication(name)` returns `[]ApplicationRoute` with `Node`, `Tags`, `Weight`, `Mode`, `State`. It does not carry `Map` — to resolve a role string to a process name, either address the process by a known name directly or fetch `ApplicationInfo` from the chosen node.

```go
func (g *Gateway) callPrimary(req any) (any, error) {
    registrar, _ := g.Node().Network().Registrar()
    routes, err := registrar.Resolver().ResolveApplication("orders")
    if err != nil {
        return nil, err
    }
    for _, r := range routes {
        if !hasTag(r.Tags, "production") {
            continue
        }
        // Option A: caller knows the concrete name by convention
        return g.Call(gen.ProcessID{Node: r.Node, Name: "orders-api"}, req)

        // Option B: resolve role via Map on the remote node
        // remote, _ := g.Node().Network().GetNode(r.Node)
        // info, _ := remote.ApplicationInfo("orders")
        // return g.Call(gen.ProcessID{Node: r.Node, Name: info.Map["api"]}, req)
    }
    return nil, gen.ErrNoRoute
}
```

## Minimal Application

An application is a plain struct implementing `gen.ApplicationBehavior`. There is no embeddable base type — just implement `Load`, `Start`, `Terminate`.

```go
package main

import (
    "ergo.services/ergo/gen"
)

type OrdersApp struct{}

func createOrdersApp() gen.ApplicationBehavior {
    return &OrdersApp{}
}

func (a *OrdersApp) Load(node gen.Node, args ...any) (gen.ApplicationSpec, error) {
    return gen.ApplicationSpec{
        Name:        "orders",
        Description: "Order processing service",
        Version:     gen.Version{Release: "1.0.0"},
        Mode:        gen.ApplicationModeTransient,
        Tags:        []gen.Atom{"production"},
        Weight:      100,
        Map: map[string]gen.Atom{
            "api":    "orders-api",
            "worker": "orders-worker",
        },
        Group: []gen.ApplicationMemberSpec{
            {Factory: createAPI,    Name: "orders-api"},
            {Factory: createWorker, Name: "orders-worker"},
        },
        Env: map[gen.Env]any{"db_dsn": "postgres://..."},
    }, nil
}

func (a *OrdersApp) Start(mode gen.ApplicationMode) {}
func (a *OrdersApp) Terminate(reason error)         {}
```

`Start` and `Terminate` do not receive `gen.Node` — there is no implicit `a.Log()`. If you need logging in these callbacks, stash the `gen.Node` passed into `Load` onto the struct:

```go
type OrdersApp struct {
    node gen.Node
}

func (a *OrdersApp) Load(node gen.Node, args ...any) (gen.ApplicationSpec, error) {
    a.node = node
    return gen.ApplicationSpec{ /* ... */ }, nil
}

func (a *OrdersApp) Start(mode gen.ApplicationMode) {
    a.node.Log().Info("orders app started in mode: %s", mode)
}

func (a *OrdersApp) Terminate(reason error) {
    a.node.Log().Info("orders app terminated: %v", reason)
}
```

## Starting the Node with the App

```go
node, err := ergo.StartNode("orders@host", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{createOrdersApp()},
})
if err != nil {
    panic(err)
}
node.Wait()
```

## Anti-Patterns

- **Do not put unrelated concerns into one application** — one app = one bounded context.
- **Do not put the supervision tree inline in `Load` without a root supervisor** — `Group` spawns processes in parallel; if they must start in order or share a failure domain, wrap them in a supervisor member.
- **Do not use `ApplicationModePermanent` with short-lived members** — normal exits trigger app shutdown.
- **Do not rely on application start order without `Depends.Applications`** — the node starts auto-loaded apps in list order, but explicit dependencies prevent subtle races.
- **Do not duplicate `Map` keys** across apps in a cluster — callers resolving a role expect one meaning.

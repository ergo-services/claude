# Application

An application groups related processes into one deployment unit with a name, version, lifecycle callbacks, and cluster-level discovery metadata. You define an application type, embed `app.Application`, and implement `Load` to return an `ApplicationSpec` describing the members to run and the failure policy. The node follows that spec when starting the application and supervises the group according to the declared mode.

## Defining an Application

Embed `app.Application` (from `ergo.services/ergo/app`) and implement `Load`. The embed supplies the framework plumbing plus no-op defaults for the other lifecycle callbacks, so a minimal application implements only `Load`.

```go
import (
    "ergo.services/ergo/app"
    "ergo.services/ergo/gen"
)

type OrdersApp struct {
    app.Application
}

func createOrdersApp() gen.ApplicationBehavior {
    return &OrdersApp{}
}

func (a *OrdersApp) Load(args ...any) (gen.ApplicationSpec, error) {
    a.Log().Info("loading orders application")
    return gen.ApplicationSpec{
        Name:        "orders",
        Description: "Order processing service",
        Version:     gen.Version{Name: "orders", Release: "1.0.0"},
        Mode:        gen.ApplicationModeTransient,
        Tags:        []gen.Atom{"production"},
        Weight:      100,
        Map: map[string]gen.Atom{
            "api":    "orders_api",
            "worker": "orders_worker",
        },
        Group: []gen.ApplicationMemberSpec{
            {Factory: createAPI, Name: "orders_api"},
            {Factory: createWorker, Name: "orders_worker"},
        },
        Env: map[gen.Env]any{"db_dsn": "postgres://..."},
    }, nil
}
```

The embedded `app.Application` promotes the methods of `gen.Application` onto your type - `a.Node()`, `a.Log()`, `a.Name()`, `a.Env(key)`, `a.EnvList()`, `a.Tags()`, `a.AddTag()`, `a.Weight()`, `a.SetWeight()`, and so on. They are bound to the running application before `Load` runs, so they work inside `Load` and every other callback. There is no need to stash `gen.Node` on the struct.

## ApplicationBehavior

```go
type ApplicationBehavior interface {
    PreLoad(app Application, args ...any) (ApplicationSpec, error)  // framework entry - do NOT override
    Load(args ...any) (ApplicationSpec, error)                     // you implement this
    Init(ref Ref, mode ApplicationMode) error                      // optional, pre-start
    Start(ref Ref, mode ApplicationMode)                           // optional, post-start
    Stop(ref Ref, reason error)                                    // optional, pre-stop
    Terminate(reason error)                                        // optional, post-stop
}
```

`app.Application` implements `PreLoad` (it binds the runtime application and dispatches to your `Load`) and supplies no-op defaults for `Init`, `Start`, `Stop`, `Terminate`. Override only the callbacks you need.

**Do not override `PreLoad`.** It is the framework entry point that binds the runtime application before dispatching `Load`. Overriding it breaks the binding and panics on the next default callback.

## Lifecycle Callbacks

The callbacks run in a fixed order around the member group. Each of `Init`, `Start`, `Stop` receives a `gen.Ref` carrying a deadline; call `ref.IsAlive()` to detect that your callback has run out of its timeout budget.

| Callback | When | Purpose | On timeout / error |
|----------|------|---------|--------------------|
| `Load` | at load | Validate config, return the spec. Declarative, avoid side effects. | error aborts the load |
| `Init` | pre-start, before any member spawns | Open resources the group needs (pools, caches, queues). | error aborts start; `Terminate` is NOT called; state reverts to `Loaded`. `ref.IsAlive() == false` means `InitTimeout` exceeded - unwind and return `gen.ErrTimeout`. |
| `Start` | post-start, group already running | Register health checks, export metrics, announce readiness. | timeout is non-fatal; app stays `Running`. |
| `Stop` | pre-stop, group still running | Drain in-flight work, deregister, mark unhealthy. Blocks member exit until it returns or `StopTimeout` expires. | on `StopTimeout` the framework proceeds to stop members. |
| `Terminate` | post-stop, group finished | Close resources opened in `Init`. | - |

Startup sequence: state `Loaded` -> `Initializing` -> run `Init` -> spawn each `Group` member sequentially in list order -> state `Running` -> run `Start`. If a member spawn fails after `Init` succeeded, the app moves to `Stopping`, already-spawned members are killed, and `Terminate` runs so resources opened in `Init` are released.

```go
func (a *OrdersApp) Init(ref gen.Ref, mode gen.ApplicationMode) error {
    dsn, _ := a.Env("db_dsn")
    pool, err := openDBPool(dsn)
    if err != nil {
        return err // start aborts; Terminate is not called
    }
    a.pool = pool
    return nil
}

func (a *OrdersApp) Start(ref gen.Ref, mode gen.ApplicationMode) {
    a.Log().Info("orders started in %s mode", mode)
}

func (a *OrdersApp) Stop(ref gen.Ref, reason error) {
    a.drain(ref) // return before ref.IsAlive() goes false
}

func (a *OrdersApp) Terminate(reason error) {
    a.pool.Close()
    a.Log().Info("orders terminated: %v", reason)
}
```

## ApplicationSpec

```go
type ApplicationSpec struct {
    Name         gen.Atom                // unique id (separate namespace from process names)
    Description  string
    Version      gen.Version
    Depends      gen.ApplicationDepends  // apps / network required before start
    Network      gen.ApplicationNetwork  // wire-format registration (see below)
    Env          map[gen.Env]any         // inherited by all member processes
    Group        []gen.ApplicationMemberSpec
    Mode         gen.ApplicationMode
    Weight       int                     // load-balance hint for clients
    Tags         []gen.Atom              // published to registrar for discovery
    Map          map[string]gen.Atom     // logical role -> registered process name
    LogLevel     gen.LogLevel
    InitTimeout  time.Duration           // 0 -> 15s
    StartTimeout time.Duration           // 0 -> 15s
    StopTimeout  time.Duration           // 0 -> 15s
}
```

Each timeout defaults to 15 seconds when left at 0. They can be overridden per start via `ApplicationOptions`.

## ApplicationMemberSpec

```go
type ApplicationMemberSpec struct {
    Factory gen.ProcessFactory  // creates the process behavior
    Options gen.ProcessOptions  // spawn options
    Name    gen.Atom            // registered name (empty -> anonymous)
    Args    []any               // passed to the process Init()
}
```

Members spawn strictly sequentially in list order - each member's spawn is awaited (its process `Init` completes) before the next begins. There is no inter-process readiness gating: "spawned" does not mean "fully ready to serve". If a member must be fully ready before the next starts, coordinate it explicitly - wrap the group in a supervisor member, or use `Depends` between separate applications, or open the shared resource in the application's own `Init`.

## ApplicationDepends

```go
type ApplicationDepends struct {
    Applications []gen.Atom  // other apps that must be running first
    Network      bool        // require network to be up
}
```

If B depends on A, the node ensures A is running before starting B. Dependency failures during `ApplicationStart` return an error.

## ApplicationNetwork

Declares wire-format values the application contributes to the node's network. Processed during `ApplicationLoad`, before any member spawns, and silently ignored when the node's network mode is `NetworkModeDisabled`. This is the standard way an application registers the EDF types it sends over the wire (the framework's own `system_app` uses it).

```go
Network: gen.ApplicationNetwork{
    RegisterTypes:  []any{Order{}, Customer{}},   // Go types for EDF
    RegisterErrors: []error{ErrInvalidOrder},      // sentinel errors for wire transport
    RegisterAtoms:  []gen.Atom{"my_atom"},         // atoms for the wire-format atom cache
}
```

## ApplicationMode

The mode is the failure policy - what happens when a member terminates.

| Mode | Value | Behavior |
|------|-------|----------|
| `ApplicationModeTemporary` | 1 | App keeps running through member exits; stops only when all members have stopped |
| `ApplicationModeTransient` | 2 | App stops if any member exits abnormally (normal/shutdown exits are tolerated) |
| `ApplicationModePermanent` | 3 | App stops if any member exits at all (normal or abnormal) |

These share names with supervisor restart strategies but are unrelated: supervisor strategies control **child restart**; application mode controls **app lifecycle**.

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

// Start (variants; the *Permanent/Transient/Temporary forms force the mode)
err := node.ApplicationStart(name, gen.ApplicationOptions{})
err := node.ApplicationStartPermanent(name, gen.ApplicationOptions{})
err := node.ApplicationStartTransient(name, gen.ApplicationOptions{})
err := node.ApplicationStartTemporary(name, gen.ApplicationOptions{})

// Stop
err := node.ApplicationStop(name)                              // graceful: runs Stop then Terminate
err := node.ApplicationStopWithTimeout(name, 30*time.Second)   // graceful with custom timeout
err := node.ApplicationStopForce(name)                         // skip Stop, kill members, still runs Terminate

// Query
info, err := node.ApplicationInfo(name)
pids, err := node.ApplicationProcessList(name, limit)                // limit is required
short, err := node.ApplicationProcessListShortInfo(name, limit)      // cheaper bulk info
apps := node.Applications()           // all loaded
running := node.ApplicationsRunning() // only running

// Unload (must be stopped first)
err := node.ApplicationUnload(name)
```

`ApplicationProcessList` and `ApplicationProcessListShortInfo` both require a `limit`. Prefer `ApplicationProcessListShortInfo` over listing PIDs and calling `ProcessInfo` on each.

Auto-start via `NodeOptions.Applications` - the node loads and starts these on boot, in list order.

### ApplicationOptions

```go
type ApplicationOptions struct {
    Env          map[gen.Env]any
    LogLevel     gen.LogLevel
    InitTimeout  time.Duration  // overrides ApplicationSpec.InitTimeout when > 0
    StartTimeout time.Duration  // overrides ApplicationSpec.StartTimeout when > 0
    StopTimeout  time.Duration  // overrides ApplicationSpec.StopTimeout when > 0
}
```

### ApplicationInfo

Returned by `node.ApplicationInfo(name)` or `remoteNode.ApplicationInfo(name)`.

| Field | Meaning |
|-------|---------|
| `Name` | application id |
| `Weight` | routing weight |
| `Tags` | discovery labels |
| `Map` | logical role -> process name |
| `Description`, `Version` | as declared in the spec |
| `Env` | populated only when `Security.ExposeEnvInfo` is enabled |
| `Depends` | declared dependencies |
| `Mode`, `State` | current mode and state |
| `Parent` | node that started this application (local node for local starts; requesting node for remote starts) |
| `Uptime` | seconds since the application started |
| `Group` | PIDs of all member processes |

## Starting the Node with the App

```go
node, err := ergo.StartNode("orders@localhost", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{createOrdersApp()},
})
if err != nil {
    panic(err)
}
node.Wait()
```

## Environment Layering

Application env variables are inherited by all member processes. They override node-level variables and are themselves overridden by process-specific variables: node provides defaults, application provides service values, processes override for their own needs.

## Runtime Tags and Weight

`Tags` and `Weight` are not just static spec fields - they can change at runtime to drive discovery. From inside any application callback the embedded `app.Application` exposes `AddTag`, `RemoveTag`, `SetTags`, `SetWeight`. A member process reaches the same runtime application through `Process.Application()`:

```go
// inside an application callback
a.AddTag("ready")

// inside a member process
func (w *Worker) HandleMessage(from gen.PID, msg any) error {
    if application := w.Application(); application != nil && w.degraded() {
        application.AddTag("maintenance")
    }
    return nil
}
```

`Process.Application()` returns `nil` for processes spawned outside any application (directly via `node.Spawn`). Mutations push an updated route to the registrar so other nodes see the change on the next route refresh - use it to flip an instance into maintenance, mark it ready after warmup, or adjust its weight under load.

## Tags and Map (Service Discovery)

**Tags** mark the instance's deployment variant so callers can filter during resolution (blue/green, canary, maintenance):

```go
Tags: []gen.Atom{"production", "region-eu"}
```

**Map** exposes logical roles to callers without coupling them to concrete process names:

```go
Map: map[string]gen.Atom{
    "api":    "orders_api",
    "worker": "orders_worker",
}
```

`Map` is not carried in `ApplicationRoute`. A caller resolving a role through it must fetch `ApplicationInfo` from the target node (`remote.ApplicationInfo("orders")` -> `info.Map["api"]`). If the role-to-name mapping is stable and agreed across services, callers can also address the process by its known name directly.

## Tag-Based Service Discovery

`Resolver().ResolveApplication(name)` returns `gen.ApplicationRoutes` - a slice of `ApplicationRoute` (`Node`, `Name`, `Weight`, `Mode`, `Tags`, `State`) with chainable, non-destructive filter methods:

- `WithTags(tags...)` keeps routes that have **all** the given tags.
- `WithoutTags(tags...)` drops routes that contain **any** of the given tags.
- `WithState(states...)` keeps routes in the given states.

It does not carry `Map`; resolve a role to a process name either by a known-name convention or by fetching `ApplicationInfo` from the chosen node.

```go
func (g *Gateway) callPrimary(req any) (any, error) {
    registrar, err := g.Node().Network().Registrar()
    if err != nil {
        return nil, err
    }
    routes, err := registrar.Resolver().ResolveApplication("orders")
    if err != nil {
        return nil, err
    }
    selected := routes.
        WithTags("production").
        WithoutTags("draining").
        WithState(gen.ApplicationStateRunning)
    for _, r := range selected {
        // Option A: caller knows the concrete name by convention
        return g.Call(gen.ProcessID{Node: r.Node, Name: "orders_api"}, req)

        // Option B: resolve the role via Map on the remote node
        // remote, _ := g.Node().Network().GetNode(r.Node)
        // info, _ := remote.ApplicationInfo("orders")
        // return g.Call(gen.ProcessID{Node: r.Node, Name: info.Map["api"]}, req)
    }
    return nil, gen.ErrNoRoute
}
```

The embedded in-memory registrar does not support application route registration, so in single-node or statically-routed deployments tags are reachable only via direct `ApplicationInfo()` calls, not through resolver queries.

## Announcing a Stop

A stopped application is not restarted by the node; recovery is left to you. Rather than hand-wiring notifications from `Terminate`, rely on the node's own event bus: the node publishes a `gen.MessageCoreApplicationStopped` (carrying the application name and stop reason) on `gen.CoreEvent`. Interested processes subscribe - locally, or cluster-wide since events cross nodes - and the application never tracks who is watching.

## Exposing a Helper API

An application (or any reusable actor library) is invoked by other code. If callers build message values by hand and hardcode your registered process name, your internals leak: renaming a process or changing a message breaks every caller. Ship a package-level helper API instead - exported functions that talk to the application's local instance by its registered name (a `gen.Atom`) and keep the message types private.

```go
package orders

const name gen.Atom = "orders" // registered process name, private to this package

// message types stay unexported - callers never construct them
type messagePlace struct{ Item string; Qty int }
type statusRequest struct{ ID string }
type statusResponse struct{ Status OrderStatus }

// Place is fire-and-forget: it wraps a Send to the local instance.
func Place(process gen.Process, item string, qty int) error {
	return process.Send(name, messagePlace{Item: item, Qty: qty})
}

// Status needs a reply: it wraps a Call.
func Status(process gen.Process, id string) (OrderStatus, error) {
	v, err := process.Call(name, statusRequest{ID: id})
	if err != nil {
		return OrderStatus{}, err
	}
	return v.(statusResponse).Status, nil
}
```

A caller writes `orders.Place(a, "sku-1", 3)` from its own callback - no message construction, no process name.

Rules:

- **The helper hides your messaging.** Callers go through the functions, so the message types stay unexported and callers depend only on the signatures and plain Go types - you can change the messaging freely. Only cross-node messages need exported fields and EDF registration.
- **Pass the handle in; never use a package-global node** (a global breaks multi-node addressing and tests). It is normally `gen.Process`. When there is no actor context - a web server calling a service on another node - take `gen.Node` and address it explicitly: `node.Call(gen.ProcessID{Name: name, Node: peer}, ...)`. That fuller form is the only place the node is named.
- Operations needing a result wrap `Call` (synchronous, returns an error); fire-and-forget operations wrap `Send`. Keep the `MessageXXX` / `XXXRequest` naming.

For a composite application, re-export the sub-components' helpers under one namespace so callers never import or name the parts. `application/radar` is the model: `radar.RegisterService` delegates to `health.Register`, `radar.CounterAdd` to `metrics.CounterAdd` - callers never learn that radar is built from the health and metrics actors. See `actor-lib.md` for the single-module helper APIs (`actor/health`, `actor/metrics`) and `applications-lib.md` for radar's full surface.

## Anti-Patterns

- **Do not put unrelated concerns into one application** - one app = one bounded context.
- **Do not override `PreLoad`** - it is the framework entry point; overriding it panics on the next callback.
- **Do not assume members are ready when the next one spawns** - members start sequentially without readiness gating; wrap the group in a supervisor member, or use `Depends`, or open shared resources in `Init` when ordering matters.
- **Do not use `ApplicationModePermanent` with short-lived members** - a normal exit triggers app shutdown.
- **Do not rely on application start order without `Depends.Applications`** - auto-loaded apps start in list order, but explicit dependencies prevent subtle races.
- **Do not duplicate `Map` keys** across apps in a cluster - callers resolving a role expect one meaning.

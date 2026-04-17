# Unit Testing

`ergo.services/ergo/testing/unit` provides in-process testing for actors without spinning up a real node. Spawn an actor with a mock runtime, drive it with messages, and assert on its observable behavior (sends, calls, spawns, logs, terminations, cron jobs, node connections) via a fluent API.

Import:
```go
import (
    "testing"
    . "ergo.services/ergo/testing/unit"  // optional dot-import for Equal, True, etc.
    "ergo.services/ergo/testing/unit"
)
```

## Creating a Test Actor

```go
actor, err := unit.Spawn(t, factoryFn, options...)
```

`factoryFn` is the same `gen.ProcessFactory` used in production (returns `gen.ProcessBehavior`). Returns `*TestActor`.

### Options

| Option | Purpose |
|--------|---------|
| `WithArgs(args ...any)` | Pass args to `Init(args ...any) error` |
| `WithLogLevel(gen.LogLevel)` | Set log level (Trace, Debug, Info, Warning, Error, Panic) |
| `WithEnv(map[gen.Env]any)` | Process environment variables |
| `WithParent(gen.PID)` | Parent PID |
| `WithRegister(gen.Atom)` | Register actor under this name |
| `WithNodeName(gen.Atom)` | Default: `"test@localhost"` |

```go
actor, err := unit.Spawn(t, factoryWorker,
    unit.WithArgs("config"),
    unit.WithLogLevel(gen.LogLevelDebug),
    unit.WithRegister("worker-1"),
)
if err != nil {
    t.Fatal(err)
}
```

## TestActor Core Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `PID()` | `gen.PID` | Actor's unique identifier |
| `Node()` | `gen.Node` | Mock node (type-assertable to `*TestNode`) |
| `Process()` | `*TestProcess` | Wrapped process for low-level calls |
| `Behavior()` | `gen.ProcessBehavior` | Raw behavior; type-assert for state inspection |

## Sending Messages

```go
// Async (chainable)
actor.SendMessage(senderPID, "ping")

// Sync (returns CallResult{Request, Response, Error, Ref, From})
result := actor.Call(senderPID, GetStatusRequest{})
if result.Error == nil {
    resp := result.Response.(GetStatusResponse)
}
```

## Fluent Assertions (Should*)

Pattern: `ShouldXxx().Matcher1().Matcher2().Assert()`. Each builder validates expectations against captured events after message dispatch.

### Send

```go
actor.ShouldSend().
    To(targetPID).                          // or gen.Atom("name"), gen.ProcessID{...}, gen.Alias
    Message(expectedValue).                 // exact match
    MessageMatching(func(any) bool { ... }).// predicate
    Times(n).                               // count
    Once().                                 // shortcut for Times(1)
    CaptureAs("slot").                      // stash matching msg for later Retrieved("slot")
    Assert()

actor.ShouldNotSend().To(targetPID).Assert()
```

### Call (sync)

```go
actor.ShouldCall().
    To(targetPID).
    Request(expectedRequest).
    Once().
    Assert()
```

### Response

```go
actor.ShouldSendResponse().
    To(senderPID).
    Response(expectedResponse).
    WithRef(result.Ref).
    Once().
    Assert()

actor.ShouldSendResponseError().
    To(senderPID).
    Error(gen.ErrProcessUnknown).
    ErrorMatching(func(err error) bool { ... }).
    WithRef(result.Ref).   // Ref(result.Ref) is an alias of WithRef
    Once().
    Assert()
```

### Spawn

```go
spawnResult := actor.ShouldSpawn().
    Factory(workerFactory).
    Once().
    Capture()  // returns SpawnResult{PID, Factory, Options, Args}

// Or assert only
actor.ShouldSpawn().Times(3).Assert()

// Meta processes (returns Alias, not PID)
metaResult := actor.ShouldSpawnMeta().Once().Capture()
// metaResult.ID is the Alias of the spawned meta process

actor.ShouldRemoteSpawn().
    ToNode("other@host").
    WithName("remote-worker").
    Once().Assert()

actor.ShouldRemoteApplicationStart().
    ToNode("app-node@host").
    Once().Assert()
```

### Log

```go
actor.ShouldLog().
    Level(gen.LogLevelError).
    Containing("timeout").
    Once().
    Assert()
```

### Termination and Exit

```go
actor.ShouldTerminate().
    WithReason(gen.TerminateReasonNormal).
    ReasonMatching(func(err error) bool { ... }).
    Once().
    Assert()

actor.ShouldNotTerminate().Assert()

actor.ShouldSendExit().
    To(targetPID).
    WithReason(someErr).
    Once().
    Assert()

actor.ShouldSendExitMeta().
    ToMeta(someAlias).
    Once().Assert()

// Direct checks
if actor.IsTerminated() {
    reason := actor.TerminationReason()
}
```

### Node Connection

```go
actor.ShouldConnect().
    ToNode("other@host").
    Once().Assert()

actor.ShouldDisconnect().
    ToNode("other@host").
    Once().Assert()
```

### Cron Jobs

```go
actor.ShouldAddCronJob().
    WithName("nightly").
    WithSpec("0 0 * * *").
    Once().Assert()

actor.ShouldRemoveCronJob().WithName("nightly").Once().Assert()
actor.ShouldEnableCronJob().WithName("nightly").Once().Assert()
actor.ShouldDisableCronJob().WithName("nightly").Once().Assert()

// Force a cron job to fire
err := actor.TriggerCronJob("nightly")

// Assert execution outcome
actor.ShouldExecuteCronJob().
    WithName("nightly").
    WithoutError().
    Once().Assert()

// Mock time for time-dependent tests
actor.SetCronMockTime(time.Date(2026, 4, 17, 0, 0, 0, 0, time.UTC))
```

## Matchers

Compose into `MessageMatching`, `ResponseMatching`, `ErrorMatching`, `ReasonMatching`, `JobMatching`, etc.

| Matcher | Purpose |
|---------|---------|
| `Equals(expected any)` | Deep equality |
| `IsValidPID()` | Value is a valid `gen.PID` |
| `IsValidAlias()` | Value is a valid `gen.Alias` |
| `IsTypeGeneric[T]()` | Value is of type `T` (generic) |
| `HasField(name, matcher)` | Struct has field matching `matcher` |
| `StructureMatching(template, map[string]Matcher)` | Struct matches template with per-field matchers |

```go
actor.ShouldSend().
    To("manager").
    MessageMatching(StructureMatching(WorkerCreated{}, map[string]Matcher{
        "ID": IsValidAlias(),
    })).
    Once().Assert()
```

## Failure Injection

Inject errors into the mock node, process, log, or network layer. Useful for exercising error handling paths.

Available on: `TestNode`, `TestProcess`, `TestLog`, `TestNetwork`.

| Method | Effect |
|--------|--------|
| `SetMethodFailure(method, err)` | Fail on every call |
| `SetMethodFailureAfter(method, nCalls, err)` | Fail after N successes |
| `SetMethodFailureOnce(method, err)` | Fail exactly once |
| `SetMethodFailurePattern(method, pattern, err)` | Fail when args stringify to match `pattern` |
| `ClearMethodFailure(method)` | Remove one failure |
| `ClearMethodFailures()` | Remove all |
| `GetMethodCallCount(method)` | How many times the method was invoked |

```go
node := actor.Node().(*unit.TestNode)
node.SetMethodFailure("Spawn", gen.ErrProcessUnknown)

actor.SendMessage(senderPID, "create_worker")
actor.ShouldLog().Level(gen.LogLevelError).Containing("spawn").Once().Assert()

count := node.GetMethodCallCount("Spawn")
unit.Equal(t, 1, count)
```

## Network Simulation

Simulate remote nodes, disconnects, and cluster-level assertions without real sockets.

```go
testNetwork := actor.RemoteNode()                                   // *TestNetwork helper
testRemote := actor.CreateRemoteNode("other@host", true)             // true = already connected
remote, err := actor.GetRemoteNode("other@host")                     // (gen.RemoteNode, error)
remote = actor.ConnectRemoteNode("other@host")                       // returns gen.RemoteNode, not error
err = actor.DisconnectRemoteNode("other@host")                       // returns error
nodes := actor.ListConnectedNodes()                                  // []gen.Atom
```

## Event Inspection

Every message/call/spawn/log/etc. emitted by the actor lands in an event buffer. Assertions consume it; you can also inspect directly.

```go
events := actor.Events()     // []Event
count := actor.EventCount()
last := actor.LastEvent()
actor.ClearEvents()          // reset before a new scenario
```

Event types you can encounter: `SendEvent`, `SendResponseEvent`, `SendResponseErrorEvent`, `SpawnEvent`, `SpawnMetaEvent`, `CallEvent`, `ExitEvent`, `ExitMetaEvent`, `MonitorEvent`, `DemonitorEvent`, `LinkEvent`, `UnlinkEvent`, `LogEvent`, `RegisterNameEvent`, `RegisterEvent`, `FailureEvent`, `CronJobAddEvent`, `CronJobRemoveEvent`, `CronJobEnableEvent`, `CronJobDisableEvent`, `CronJobExecutionEvent`, `TerminateEvent`, `RemoteSpawnEvent`, `NodeConnectionEvent`.

## Capture and Retrieve

For dynamic values (aliases, IDs) that aren't known ahead of time:

```go
actor.SendMessage(senderPID, "start")

// Capture during assertion
actor.ShouldSend().To("logger").CaptureAs("log-msg").Once().Assert()
captured := actor.Retrieved("log-msg")

// Manual capture
actor.Capture("session-id", "abc123")
id := actor.Retrieved("session-id")
```

## Built-in Assertions

Zero-dependency helpers for state checks:

```go
unit.Equal(t, expected, actual, "optional message")
unit.NotEqual(t, expected, actual)
unit.True(t, condition)
unit.False(t, condition)
unit.Nil(t, value)
unit.NotNil(t, value)
unit.Contains(t, haystack, needle)  // strings
unit.IsType(t, expectedType, value)
```

## Complete Test Example

```go
package orders

import (
    "testing"

    "ergo.services/ergo/act"
    "ergo.services/ergo/gen"
    "ergo.services/ergo/testing/unit"
)

type OrderActor struct {
    act.Actor
    received int
}

func factoryOrderActor() gen.ProcessBehavior {
    return &OrderActor{}
}

type PlaceOrder struct{ ID string }
type OrderPlaced struct{ ID string }

func (o *OrderActor) Init(args ...any) error {
    o.Log().Info("orders started")
    return nil
}

func (o *OrderActor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case PlaceOrder:
        o.received++
        o.Send("audit", OrderPlaced{ID: m.ID})
    }
    return nil
}

func TestOrderActor_PlaceOrder(t *testing.T) {
    actor, err := unit.Spawn(t, factoryOrderActor)
    if err != nil {
        t.Fatal(err)
    }

    // Initial log on start
    actor.ShouldLog().Level(gen.LogLevelInfo).Containing("orders started").Once().Assert()

    // Drive the message
    actor.SendMessage(gen.PID{}, PlaceOrder{ID: "order-42"})

    // Assert outbound event
    actor.ShouldSend().
        To("audit").
        Message(OrderPlaced{ID: "order-42"}).
        Once().
        Assert()

    // Inspect private state
    behavior := actor.Behavior().(*OrderActor)
    unit.Equal(t, 1, behavior.received)
}

func TestOrderActor_AuditFailure(t *testing.T) {
    actor, err := unit.Spawn(t, factoryOrderActor)
    if err != nil {
        t.Fatal(err)
    }
    actor.ClearEvents()

    node := actor.Node().(*unit.TestNode)
    node.SetMethodFailure("Send", gen.ErrProcessMailboxFull)

    actor.SendMessage(gen.PID{}, PlaceOrder{ID: "order-43"})

    // Actor tried and the send failed
    unit.Equal(t, 1, node.GetMethodCallCount("Send"))
}
```

## Idiomatic Patterns

### Pattern: Dynamic ID capture

```go
actor.SendMessage(senderPID, "create_worker")
spawnResult := actor.ShouldSpawnMeta().Once().Capture()

actor.ShouldSend().
    To("manager").
    MessageMatching(func(msg any) bool {
        wc, ok := msg.(WorkerCreated)
        return ok && wc.ID == spawnResult.ID
    }).
    Once().Assert()
```

### Pattern: Multi-step workflow

```go
actor.SendMessage(senderPID, "init")
actor.ShouldSpawn().Times(2).Assert()

actor.SendMessage(senderPID, "process")
actor.ShouldSend().To(workerPID).Once().Assert()

actor.SendMessage(senderPID, "shutdown")
actor.ShouldTerminate().WithReason(gen.TerminateReasonShutdown).Assert()
```

### Pattern: Async response via deferred Send

```go
result := actor.Call(senderPID, AsyncJobRequest{})
// result.Response is nil because the handler returned (nil, nil)
// The real response arrives later via SendResponse.

actor.ShouldSendResponse().
    To(senderPID).
    WithRef(result.Ref).
    Once().
    Assert()
```

### Pattern: Failure injection path

```go
node := actor.Node().(*unit.TestNode)
node.SetMethodFailurePattern("RegisterName", "forbidden-*", gen.ErrTaken)

actor.SendMessage(senderPID, TryRegister{Name: "forbidden-xyz"})
actor.ShouldLog().Level(gen.LogLevelWarning).Containing("taken").Once().Assert()
```

## Anti-Patterns

- **Do not depend on wall-clock time inside tests.** Use `SetCronMockTime` and `TriggerCronJob` for time-driven code.
- **Do not write tests that require a real network.** Use `CreateRemoteNode`, `ConnectRemoteNode`, `ShouldConnect`/`ShouldDisconnect`.
- **Do not verify internal counters by reading the behavior before `Init` finishes.** Assertions against startup behavior must come after `Spawn` returns successfully.
- **Do not forget `ClearEvents()` between scenarios in a table-driven test** — otherwise assertions match events from previous iterations.
- **Do not rely on goroutine scheduling** — `Spawn`, `SendMessage`, and `Call` are synchronous in the unit test harness. Sleeps are never needed.
- **Do not ignore `CallResult.Error`** — assert on it explicitly when the test expects a failure path.

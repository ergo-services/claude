# Testing

Ergo ships a testing toolkit under `ergo.services/ergo/testing/` built on one idea: an actor is not a function you call and inspect. It runs on its own goroutine, keeps private state, and speaks only in messages. So you test what it *does* - the messages it sends, the children it spawns, the way it terminates - not the fields it holds. Every outward action is captured as a typed record, and a test drives an input, then asserts on the records produced.

Four packages cover the span from a single actor to a live cluster, and they share one assertion grammar:

| Package | Layer | Use it for |
|---------|-------|-----------|
| `testing/check` | the grammar | the record types and the `Should...` assertion chain that `unit` and `stage` are written in |
| `testing/unit` | one actor, mock node | a single actor's decision logic, synchronously and deterministically. Most tests live here |
| `testing/stage` | live multi-node runtime | supervision, restarts, links/monitors across nodes, remote spawn, disconnects |
| `testing/mock` | standalone fakes | ordinary code (a helper, a resolver, a constructor) that consumes a `gen.*` interface but is not an actor |

`unit` and `stage` are not two ways to write the same test; they answer different questions. A typical suite tests the bulk of its actor logic with `unit` (microseconds, never flaky) and reserves `stage` for the cross-node and supervision scenarios only the real runtime exhibits.

## check: The Assertion Grammar

You rarely import `check` directly - `unit` and `stage` expose its assertions on their own handles - but it defines the vocabulary both reuse. Learn it once here.

### The Journal of Records

As the thing under test runs, the harness appends every action it observes to an ordered journal of typed *records*. You never build one; the harness does. Records fall into three groups:

- **Egress** - what the actor does: `Send`, `Call`, `Spawn`, `SpawnMeta`, `RemoteSpawn`, `RemoteApplicationStart`, `Link`, `Unlink`, `Monitor`, `Demonitor`, `Forward`, `SendResponse`, `SendExit`, `SendExitMeta`, `SendEvent`, `CreateAlias`, `DeleteAlias`, `RegisterEvent`, `UnregisterEvent`, `Log`, `AddCronJob`, `RemoveCronJob`, `SendAfter`, `Span`.
- **Lifecycle** - `Terminated`, the actor's own end.
- **Ingress** - what *reaches* the actor: `Delivered`, `Down`, `Exit`, `Event`, and the wire subscriptions (`WireLink`, `WireMonitor`, ...). These appear only in `stage`, where delivery is real; `unit` has no real delivery to observe.

Each record carries fields describing the action - a `Send` has `From`, `To`, `Message`, `Options`, `Error`; a `Spawn` has `Parent`, `Child`, `Register`, `Factory`, `Options`, `Error`. You match on those fields, not on text. The full field list is in the package godoc; in practice you meet each record through the assertion that selects it.

### The Assertion Chain

An assertion is a single chain with four parts: choose a record type, narrow it with filters, say how many should match, and run it.

```go
sub.ShouldSend().To("db").Message(SaveUser{ID: 7}).Once().Assert()
```

Read left to right: of the recorded `Send` actions (`ShouldSend`), the ones to `"db"` carrying that message (the filters), there should be exactly one (`Once`); evaluate now and report a failure if not (`Assert`). There is one `ShouldXxx` builder per record type.

### Cardinalities (how many)

```go
sub.ShouldSpawn().Once().Assert()                  // exactly one
sub.ShouldSpawn().Times(3).Assert()                // exactly three
sub.ShouldSend().To("metrics").AtLeast(1).Assert() // one or more
sub.ShouldSend().To("audit").None().Assert()       // never (assert a negative)
```

There is no separate "should not" builder; a negative is `None()`.

### Filters (narrowing)

Filters are named after the record's fields and reach well past `To` and `Message`:

```go
sub.ShouldSend().To("worker").Priority(gen.MessagePriorityHigh).Once().Assert()
sub.ShouldSpawn().Factory(factoryWorker).Times(3).Assert()
sub.ShouldLog().Level(gen.LogLevelError).Containing("timeout").Once().Assert()
```

Where an action can fail, `Error` matches an exact error and `ErrorIs` a wrapped one:

```go
sub.ShouldCall().To("db").ErrorIs(gen.ErrTimeout).Once().Assert()
```

When no named filter fits, `Where` takes a typed predicate over the record itself:

```go
sub.ShouldSend().Where(func(r check.Send) bool {
    n, ok := r.Message.(Notification)
    return ok && n.Urgent
}).AtLeast(1).Assert()
```

Filters compose: a record must satisfy all of them to match.

Common filters by builder:

| Builder | Filters |
|---------|---------|
| `ShouldSend` | `From`, `To`, `Message`, `Error`, `ErrorIs`, `Priority`, `Important`, `KeepNetworkOrder` |
| `ShouldCall` | `From`, `To`, `Request`, `Error`, `ErrorIs` |
| `ShouldSpawn` | `From`, `Child`, `Register`, `Factory`, `Error`, `ErrorIs` |
| `ShouldSpawnMeta` | `From`, `Alias`, `Error`, `ErrorIs` |
| `ShouldRemoteSpawn` | `From`, `To` (node), `Name`, `Register`, `Child`, `Error`, `ErrorIs` |
| `ShouldRemoteApplicationStart` | `From`, `To` (node), `Name`, `Mode`, `Error`, `ErrorIs` |
| `ShouldSendResponse` | `From`, `To`, `Message`, `Ref`, `Error`, `ErrorIs` |
| `ShouldLog` | `From`, `Level`, `Message`, `Containing` |
| `ShouldTerminate` | `Reason`, `ReasonIs`, `Normally`, `Abnormally` |
| `ShouldSendExit` / `ShouldSendExitMeta` | `From`, `To` / `Meta`, `Reason`, `ReasonIs`, `Error`, `ErrorIs` |
| `ShouldMonitor` / `ShouldLink` (and Demonitor/Unlink) | `From`, `Target`, `Error`, `ErrorIs` |
| `ShouldAddCronJob` | `From`, `Name`, `Spec`, `Error`, `ErrorIs` |
| `ShouldRemoveCronJob` | `From`, `Name`, `Error`, `ErrorIs` |
| `ShouldSendAfter` | `From`, `To`, `Message`, `After`, `Priority`, `Error`, `ErrorIs` |
| `ShouldSendEvent` | `From`, `Name`, `Token`, `Error`, `ErrorIs`, `Priority` |
| `ShouldDeliver` (ingress) | `From`, `To`, `ToProcessID`, `ToAlias`, `Message` |
| `ShouldReceiveDown` (ingress) | `To`, `About`, `AboutProcessID`, `AboutAlias`, `AboutNode`, `Reason`, `ReasonIs` |
| `ShouldReceiveExit` (ingress) | `To`, `About`, `AboutNode`, `Reason`, `ReasonIs` |
| `ShouldReceiveEvent` (ingress) | `To`, `Event`, `Message`, `Timestamp` |

### Termination

`ShouldTerminate` is a record like the others, with a small vocabulary for the reason:

```go
sub.ShouldTerminate().Reason(errBoom).Once().Assert()  // exact reason
sub.ShouldTerminate().Abnormally().Once().Assert()     // crashed, panicked, killed, or errored
sub.ShouldTerminate().Normally().Once().Assert()       // TerminateReasonNormal or Shutdown
sub.ShouldTerminate().None().Assert()                  // still running
```

### Reading a Snapshot, or Waiting: Within

Whether an assertion waits depends on the layer, and this is the one real difference between `check` in `unit` and in `stage`. In `unit` the actor has already run by the time you assert, so the journal is final - the assertion reads a snapshot and returns at once. In `stage` real actors run concurrently, so `Within` turns the terminal into a bounded wait that polls until the assertion holds or the deadline passes:

```go
node.ShouldDeliver().To(worker).Within(time.Second).Once().Assert()
```

A positive assertion is satisfied the moment its count is met. A negative watches for the whole window: `None().Within(...)` passes only if nothing matched. One sharp edge: with an exact count, `Within` is met at the first poll where the count equals n, so a still-growing count that overshoots n never re-reads as n - for a count that only grows, use `AtLeast` rather than `Times`.

### Scoping to a Phase: Mark and Since

A test often runs in stages, and an earlier stage may have produced the same record you now want to count. `Mark` records the current journal position; `Since` restricts the next assertion to what came after it:

```go
sub.SendMessage(client, "first")
mark := sub.Mark()

sub.SendMessage(client, "second")
sub.ShouldSend().To(client).Since(mark).Once().Assert() // only the reply to "second"
```

This is also how you express "no *second* occurrence after a legitimate first": mark past the first, then `None().Since(mark)`.

### Pulling Values Out: Capture and Collect

Actors produce values you cannot know in advance - a spawned child's PID, an allocated alias. `Capture` returns the first matching record so you can read those values; note it returns `(R, bool)`:

```go
spawn, ok := sub.ShouldSpawn().Once().Capture()
childPID := spawn.Child
```

`Collect` returns every matching record in observation order, which is what you want when the order itself is under test (a round-robin distribution, say):

```go
sends := sub.ShouldSend().To("worker").Collect() // []check.Send, in order
```

`Records()` returns the whole journal as a slice for debugging; assertions should use the grammar.

### Assert or Must

Both evaluate the same way; they differ on failure. `Assert` reports it and lets the test continue. `Must` stops the test immediately - use it when later steps cannot run meaningfully without this one.

### Plain Helpers

Not everything in a test is a record. `check` carries plain helpers so a test needs no second assertion library:

```go
check.NoError(t, err)
check.ErrorIs(t, err, gen.ErrProcessUnknown)
check.Equal(t, JobQueued{ID: "42"}, got) // note the order: want, got
check.True(t, behavior.started)
```

The set is `NoError`, `Error`, `ErrorIs`, `ErrorContains`, `Equal`, `NotEqual`, `True`, `False`, `Nil`, `NotNil`, `Contains`.

### Stub Matchers

One last piece of vocabulary shows up in the stubbing APIs of `unit` and `mock`, not in assertions. A small set of `check.Matcher` constructors narrows a stub to the calls it should handle: `Anything()`, `Equals(want)`, `MatchedBy(func(any) bool)`, and `IsType[V]()`.

```go
sub.OnCall("db").Where(check.IsType[Query]()).Respond(rows)
```

`IsType[V]` matches a value assignable to `V`: a concrete type matches its exact dynamic type, an interface matches any value implementing it.

## unit: One Actor on a Mock Node

`unit` spawns one behavior on a mock node, gives you a `Subject` to drive its callbacks, and records everything the actor does. It is synchronous: no real goroutines, no clock. You deliver a message, the handler runs to completion on the calling goroutine, and by the time the call returns the records are there to read.

### The Shape of a Test

```go
sub, err := unit.Spawn(t, factoryWorker, gen.ProcessOptions{})
if err != nil {
    t.Fatal(err)
}

sub.SendMessage(client, "ping")
sub.ShouldSend().To(client).Message("pong").Once().Assert()
```

`unit.Spawn` runs the behavior's `ProcessInit` and returns a `*Subject`. The `Subject` embeds `check`'s asserter, which is why `sub.ShouldSend(...)` is a method on it. The signature is:

```go
func Spawn(t testing.TB, factory gen.ProcessFactory, options gen.ProcessOptions, args ...any) (*Subject, error)
```

`options` is the real `gen.ProcessOptions` (there are no `WithX` functional options); trailing `args` are forwarded to `ProcessInit`, exactly as `gen.Node.Spawn` forwards them.

### Entry Points

| Entry point | Effect |
|-------------|--------|
| `unit.Spawn(t, factory, opts, args...)` | spawn on a default node, run `ProcessInit`; returns `(*Subject, error)` |
| `unit.SpawnRegister(t, name, factory, opts, args...)` | same, with a registered name |
| `unit.Prepare(t, factory, opts, args...)` | build the `Subject` WITHOUT running `ProcessInit`; returns `*Subject` |
| `unit.PrepareRegister(t, name, factory, opts, args...)` | same, with a registered name |
| `unit.StartNode(t, name, gen.NodeOptions{})` | build a configurable node first; returns `*MockNode` |

When you need more than a default node - a node name, a seeded environment, a log level - build the node first and spawn on it. Log level and env come from `gen.NodeOptions`; the default node name is `unit@localhost` and the default log level is `Info`.

```go
n := unit.StartNode(t, "worker@localhost", gen.NodeOptions{
    Env: map[gen.Env]any{"role": "primary"},
    Log: gen.LogOptions{Level: gen.LogLevelDebug},
})
sub, err := n.Spawn(factoryWorker, gen.ProcessOptions{})
```

`MockNode` also carries `SpawnRegister`, `Prepare`, and `PrepareRegister`.

### Driving Inputs

`SendMessage` is one of a family, one for every way a message reaches an actor, so you can exercise each callback in isolation:

| Driver | Drives |
|--------|--------|
| `SendMessage` / `SendMessageName` / `SendMessageAlias` | `HandleMessage` and its name/alias split-handlers |
| `SendMessageWithPriority` | a message at a given queue priority |
| `Call` / `CallName` / `CallAlias` / `CallWithPriority` | `HandleCall`, returning the actor's reply |
| `DeliverExit` / `DeliverExitMessage` | an exit signal on the urgent queue |
| `DeliverDown` / `DeliverDownMessage` | a monitor's down notification |
| `DeliverEvent` / `DeliverRegistrarEvent` | `HandleEvent` |
| `DeliverLog` | `HandleLog` (actor registered as a logger) |
| `DeliverSpan` | `HandleSpan` (actor registered as a tracing exporter) |
| `Inspect` | `HandleInspect`, returning the reported map |
| `FireTimers` | scheduled `SendAfter` messages whose target is the actor |
| `FireCron` | a registered cron job's `gen.MessageCron` |

`Call` is the driver that hands a value back - it returns `(any, error)`:

```go
resp, err := sub.Call(client, "status")
check.NoError(t, err)
check.Equal(t, "ready", resp)
```

Keep the direction straight: `Call` is you calling *into* the actor. Controlling what the actor's *own* outbound calls return is a different tool, `OnCall`, below. A message the actor sends to itself is recorded as an outgoing send, not looped into its mailbox - to drive its reaction, deliver it yourself with `SendMessage`.

### Stubbing What the Actor Does

An actor that calls a dependency cannot be tested alone unless you decide what the call returns. `OnCall` intercepts the actor's outbound `Call`:

```go
sub.OnCall("backend").Respond("OK")
sub.SendMessage(client, "ping")
sub.ShouldSend().To("client").Message("OK").Once().Assert()

sub.OnCall("backend").Fail(gen.ErrTimeout) // exercise the error branch
```

`OnCall` offers `Respond(v)`, `Fail(err)`, `RespondWith(func(request any) (any, error))`, and `Where(matcher)` to narrow by request. Other outward actions with a return value have their own typed stub, and the error-only ones take `Fail` / `FailFunc`:

| Stub | Configures | Outcomes |
|------|-----------|----------|
| `OnCall(to)` | outbound `Call` | `Respond`, `Fail`, `RespondWith`, `Where` |
| `OnSpawn(factory)` | `Spawn` / `SpawnRegister` | `Return(pid)`, `Fail`, `ReturnFunc` |
| `OnSpawnMeta(sample)` | `SpawnMeta` (matched by behavior type) | `Return(alias)`, `Fail` |
| `OnRemoteSpawn(node, name)` | `RemoteSpawn` | `Return(pid)`, `Fail` |
| `OnCreateAlias()` | `CreateAlias` | `Return(alias)`, `Fail` |
| `OnRegisterEvent(name)` | `RegisterEvent` | `Return(token)`, `Fail` |
| `OnSend` / `OnLink` / `OnUnlink` / `OnMonitor` / `OnDemonitor` / `OnSendExit` / `OnSendExitMeta` / `OnForward` | error-only ops | `Fail(err)`, `FailFunc(func() error)` |

`FailFunc` makes failure selective - a counter in the closure can fail only the second and fifth send:

```go
i := 0
sub.OnSend("svc").FailFunc(func() error {
    i++
    if i == 2 || i == 5 {
        return gen.ErrProcessMailboxFull
    }
    return nil
})
```

Two rules hold for every stub. First, whatever it decides, the action is still recorded - a stub shapes the return value, it does not hide the send from `ShouldSend`. Second, an unstubbed stub is permissive: an unstubbed `Call` returns `(nil, nil)`, a value-producing op returns a synthetic value, an error-only op succeeds. When several stubs match, the most recently registered one wins. The one loud exception is an unstubbed *resolve* through the mock network (see Service Discovery), because a forgotten discovery stub is almost always a bug.

### Stubbing Init-Time Egress: Prepare then Run

A stub only answers a call made after you set it. For a call from a handler that is enough - set the stub after `Spawn`, before the input that triggers it. But some actors call a dependency in `Init` itself (a supervisor that registers with a service as it starts). By the time `Spawn` returns, `Init` has already run, so a stub set on the returned `Subject` is too late.

Split the spawn. `Prepare` builds the actor without running `Init`; you set the stubs, then `Run` runs `Init` with them in place:

```go
sub := unit.Prepare(t, factoryRadarSup, gen.ProcessOptions{})
sub.OnCall("radar_health").Fail(gen.ErrProcessUnknown) // the service is not running
if err := sub.Run(); err != nil {
    t.Fatal(err)
}
```

`Spawn` is exactly `Prepare` followed by `Run`. Until `Run`, the actor is not initialized: any driver fails the test loudly rather than run against a half-built actor, and calling `Run` twice fails the same way.

### Controlling What the Actor Reads

The mirror of what the actor does is what it reads. What it reads about itself is an override on the `Subject`:

```go
sub.OnEnv(func(name gen.Env) (any, bool) { return "production", true })
```

Every non-egress accessor the actor calls on itself has such an override - `OnState`, `OnLog`, `OnUptime`, `OnInfo`, `OnLeader`, and the rest; the godoc has the full set. What it reads from its node is controlled through `sub.Node()`, a `*MockNode`:

```go
sub.Node().OnIsAlive(func() bool { return false })
```

### Service Discovery and Remote Nodes

The node carries a built-in mock network. Stub what the actor's resolver returns and you drive its routing with no registrar in sight:

```go
sub.Node().Network().Registrar().Resolver().
    OnResolveApplication("worker_app").
    Return(gen.ApplicationRoute{Node: "node1@localhost", State: gen.ApplicationStateRunning})
```

Resolver stubs offer `Return(...)` and `Fail(err)`. An unstubbed `Resolve` / `ResolveApplication` fails the test loudly (unlike other egress). `Network().FailRegistrar(err)` drives the no-registrar branch. Registrar events are configured with `Network().Registrar().OnEvent(event)` / `FailEvent(err)` and driven into the actor with `sub.DeliverRegistrarEvent(message)`.

`Network().OnGetNode(name)` returns a programmable `*MockRemoteNode`. Reaching it is a read, but the `Spawn` or application start the actor then issues on it is outward work, recorded as remote egress:

```go
peer := sub.Node().Network().OnGetNode("peer@localhost")
peer.OnSpawn("svc").Return(remotePID)
peer.OnApplicationStart("app").Fail(gen.ErrApplicationUnknown)

// after the actor acts:
sub.ShouldRemoteSpawn().To("peer@localhost").Name("svc").Once().Assert()
```

Until configured, `GetNode(name)` / `Node(name)` return `gen.ErrNoConnection`.

### Cron

The actor adds jobs via `Node().Cron().AddJob`, recorded as `AddCronJob`. The test fires one with `FireCron`, which delivers a `gen.MessageCron` to the subject:

```go
// the actor, in a handler: a.Node().Cron().AddJob(gen.CronJob{Name: "nightly", Spec: "0 0 * * *"})
sub.ShouldAddCronJob().Name("nightly").Spec("0 0 * * *").Once().Assert()

sub.FireCron("nightly") // delivers gen.MessageCron; fails loudly if the job is absent or disabled
sub.ShouldSend().To("reporter").Once().Assert()
```

There is no mock clock. A `SendAfter` is recorded as a `SendAfter` and delivered only when the test calls `FireTimers`, which fires every scheduled, uncancelled timer targeting the subject.

### Reading What the Actor Produced

The PIDs `unit` hands back are honest - every spawn gets a distinct, well-formed `gen.PID`, so you can capture a generated value and assert it flows correctly:

```go
sub.SendMessage(client, "spawn-worker")
spawn, _ := sub.ShouldSpawn().Once().Capture()
sub.ShouldSend().To("manager").Message(spawn.Child).Once().Assert()
```

Termination is recorded. When a callback returns an error, panics, or returns a stop reason, the actor terminates:

```go
sub.SendMessage(client, "self-destruct")
sub.ShouldTerminate().Reason(errBoom).Once().Assert()
check.True(t, sub.Terminated())
```

A panic in a callback is recovered into `gen.TerminateReasonPanic`, exactly as the real runtime does. `sub.Behavior()` returns the live behavior for a white-box look; `sub.PID()`, `sub.Terminated()`, and `sub.Reason()` read the subject directly.

### Faithful Runtime Semantics

The mock node enforces the rules a real process enforces, so a test catches the same misuse production would. Linking a process to itself is rejected with `gen.ErrNotAllowed`. `SetSendPriority` validates its argument and the priority is stateful. The logger gates by level, so a line below the configured level is never recorded. A message addressed by registered name dispatches to the name split-handler. You do not opt into any of this.

### Meta Processes

Meta processes - the I/O adapters behind TCP, UDP, web - have their own contract, and `unit` drives them. `SpawnMeta` instantiates a meta behavior under the actor, runs its `Init`, and returns a `*MetaSubject`:

```go
m, err := sub.SpawnMeta(&echoMeta{}, gen.MetaOptions{})
m.DeliverMessage(sub.PID(), "hello")
m.ShouldSend().To("client").Message("got:hello").Once().Assert()
```

A meta shares its parent's journal and its egress is observed as coming from the parent PID - scope with `Since(mark)` when both the parent and the meta send. Drivers are `DeliverMessage` (HandleMessage), `Request` (HandleCall, auto-recording the `SendResponse`), `Inspect` (HandleInspect), `Start`, and `Terminate`; the state gates apply, so `SendResponse` is rejected outside a running callback. `ID()`, `Parent()`, `State()`, and `Behavior()` read the meta. Its outbound calls are stubbed on its own scope (`m.OnSend`, `m.OnSpawnMeta`), not the parent's. To shape Init-time egress, use `sub.PrepareMeta(...)` then `m.Run()`.

## stage: The Live Multi-Node Harness

`stage` starts real nodes, runs real actors and applications, lets them talk over the real network, and observes what the live runtime does. Use it for what `unit` deliberately leaves out: the real scheduler, supervision and restarts, links and monitors across nodes, remote spawn, disconnects. Because everything runs concurrently, assertions wait with `Within` instead of reading a snapshot.

### The Shape of a Test

```go
s := stage.New(t)
n := s.StartNode("n")
ponger := n.Spawn(factoryPonger, gen.ProcessOptions{})

n.Send(ponger, ping{Seq: 1})
n.ShouldDeliver().To(ponger).Message(ping{Seq: 1}).Once().Within(time.Second).Assert()
```

`stage.New(t)` returns a `*Stage` - the owner of every node the test starts - and registers cleanup with the test, so the nodes stop automatically. `s.StartNode(prefix, ...)` starts one live node and returns a `*stage.Node`, a thin wrapper around a real `gen.Node`. It surfaces the operations a test reaches for - `Spawn`, `SpawnRegister`, `Send`, `Call`, `SendExit`, `Kill`, `EnableSpawn`, `EnableApplicationStart`, `ProcessPID`, `ProcessID`, `Name` - and carries the assertion grammar. For anything the wrapper does not cover, `n.Native()` returns the real `gen.Node`.

### Where Things Are Recorded

Each node keeps its own journal. Egress is recorded on the node the acting process runs on; ingress on the node hosting the receiver. One cross-node interaction leaves two traces:

```go
s := stage.New(t)
a, b := s.StartNode("a"), s.StartNode("b")

// a value that crosses the wire must be registered for transport, on both nodes
a.Native().Network().RegisterType(ping{})
b.Native().Network().RegisterType(ping{})

ponger := b.Spawn(factoryPonger, gen.ProcessOptions{})
pinger := a.Spawn(factoryPinger, gen.ProcessOptions{})

a.Send(pinger, sendPing{To: ponger, Seq: 1})

a.ShouldSend().From(pinger).Message(ping{Seq: 1}).Once().Within(time.Second).Must()   // egress, on a
b.ShouldDeliver().To(ponger).Message(ping{Seq: 1}).Once().Within(time.Second).Must()  // ingress, on b
```

The nodes were never explicitly connected - the first time `a` addressed a process on `b`, the runtime looked `b` up through the registrar and dialed it. This egress/ingress split is the half `unit` cannot show, which is why the ingress assertions (`ShouldDeliver`, `ShouldReceiveDown`, `ShouldReceiveExit`, `ShouldReceiveEvent`, `ShouldWireLink`, ...) live only in `stage`.

### Observing a Process Stop

Stage does not hand you a "terminated" record; a process ending is something other processes learn about through a monitor's `Down` or a link's `Exit`. So you observe a stop the way the system does:

```go
target := n.Spawn(factoryPonger, gen.ProcessOptions{})
w := n.Spawn(factoryWatcher, gen.ProcessOptions{}, target) // monitors target in Init

n.ShouldMonitor().From(w).Target(target).Once().Within(time.Second).Must()
n.Kill(target)
n.ShouldReceiveDown().To(w).About(target).Reason(gen.TerminateReasonKill).
    Once().Within(time.Second).Must()
```

The node logger is off in a stage and `SendAfter` timers fire for real, so there are no `Log`, `Terminated`, or `SendAfter` records - those facts a harness has to synthesize, which is `unit`'s job. Stage shows only what genuinely happened. A synchronous `Call` blocks for its reply and hands it back, so you check its return with `check.Equal`, no `Within`.

### Connecting on Purpose

Nodes connect themselves on first contact, so `s.Connect` is never required just to make traffic flow. Reach for it in two cases: to test connectivity itself (`s.Connect(a, b)` dials now and waits deterministically until both sides register the link), and for remote operations (`Connect` returns `a`'s view of `b` as a `gen.RemoteNode`, wrapped so its `Spawn` and application-start egress is recorded on `a`'s journal).

```go
b := s.StartNode("b", stage.NodeOptions{NetworkFlags: gen.NetworkFlags{Enable: true, EnableRemoteSpawn: true}})
b.EnableSpawn("worker", factoryWorker)

remote := s.Connect(a, b)
remote.Spawn("worker", gen.ProcessOptions{})

a.ShouldRemoteSpawn().To(b.Name()).Name("worker").Once().Within(time.Second).Assert()
```

For more than two nodes, `s.ConnectMesh(nodes...)` connects every pair at once and waits until the mesh settles.

### Configuring Nodes

`stage.NodeOptions` carries `Applications` to load, `Env`, a `Cookie`, network knobs (`MaxMessageSize`, `FragmentSize`, `NetworkFlags`, `PoolSize`, `Mode`), and `Security`. Loading an application tests framework-spawned, supervised, name-registered processes end to end. By default a node starts bare so a test can assert exact counts; add system services with `NodeOptions{EnableSystemApp: true}`. `stage.StageOptions` configures discovery: `RegistrarFull: true` upgrades the in-memory registrar to serve `ResolveApplication` and emit the canonical registrar event stream; `Registrar` supplies a real backend factory (etcd, for instance).

## mock: Standalone Fakes

Not all code that touches the framework is an actor. A helper that takes a `gen.Process`, a custom `gen.Resolver`, a constructor that reads a `gen.Node` - these are ordinary functions, tested the ordinary way. `mock` provides a controllable fake of each interface. A mock is not the `unit` harness: it runs no actor, starts no goroutine, and never fails your test on its own.

```go
func saveUser(db gen.Process, u User) error {
    _, err := db.Call("users", Insert{Name: u.Name})
    return err
}

func TestSaveUser(t *testing.T) {
    db := mock.NewProcess()
    db.OnCall(func(to, request any) (any, error) { return Row{ID: 7}, nil })

    check.NoError(t, saveUser(db, User{Name: "ann"}))
}
```

Every method has a matching `On<Method>` setter; a method you do not override returns a safe default (a query returns a zero value, an action reports success, an identifier is synthesized). This is the deliberate difference from `unit`: an unconfigured call is a no-op with a sensible result, not the bug under test.

Each type comes as a pair: the dumb constructor and a recording one whose name ends in `T` and takes the test's `T`. The recording form records every action and exposes the `check` grammar, so you can assert what the code did:

```go
node := mock.NewNodeT(t)
bootstrap(node) // the code under test calls node.Spawn three times
node.ShouldSpawn().Times(3).Assert()
```

On a recording mock, an override decides the return value while the action is still recorded - the same rule as stubbing in `unit`.

| Constructor | Interface |
|-------------|-----------|
| `NewNode` / `NewNodeT` | `gen.Node` |
| `NewProcess` / `NewProcessT` | `gen.Process` |
| `NewMeta` / `NewMetaT` | `gen.MetaProcess` |
| `NewLog` / `NewLogT` | `gen.Log` |
| `NewCron` / `NewCronT` | `gen.Cron` |
| `NewNetwork` / `NewNetworkT` | `gen.Network` |
| `NewRemoteNode` / `NewRemoteNodeT` | `gen.RemoteNode` |
| `NewRegistrar` / `NewRegistrarT` | `gen.Registrar` |
| `NewResolver` / `NewResolverT` | `gen.Resolver` |

Some interfaces hand back others (a `gen.Node` exposes a `gen.Log`, `gen.Network`, `gen.Cron`). A recording mock wires the sub-mocks it owns to share its recorder, so their actions collate into one journal and you assert through the parent. To steer a returned interface (a resolver that fails, say), build it from the bottom up and wire it in with the parent's `On*` override:

```go
resolver := mock.NewResolver()
resolver.OnResolve(func(node gen.Atom) ([]gen.Route, error) { return nil, gen.ErrNoRoute })

reg := mock.NewRegistrar()
reg.OnResolver(func() gen.Resolver { return resolver })

net := mock.NewNetwork()
net.OnRegistrar(func() (gen.Registrar, error) { return reg, nil })
```

## Complete Unit Test Example

```go
package orders_test

import (
    "testing"

    "ergo.services/ergo/act"
    "ergo.services/ergo/gen"
    "ergo.services/ergo/testing/check"
    "ergo.services/ergo/testing/unit"
)

type OrderActor struct {
    act.Actor
    received int
}

func factoryOrderActor() gen.ProcessBehavior { return &OrderActor{} }

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
    sub, err := unit.Spawn(t, factoryOrderActor, gen.ProcessOptions{})
    if err != nil {
        t.Fatal(err)
    }

    sub.ShouldLog().Level(gen.LogLevelInfo).Containing("orders started").Once().Assert()

    sub.SendMessage(gen.PID{}, PlaceOrder{ID: "order-42"})

    sub.ShouldSend().To("audit").Message(OrderPlaced{ID: "order-42"}).Once().Assert()

    behavior := sub.Behavior().(*OrderActor)
    check.Equal(t, 1, behavior.received)
}
```

## Anti-Patterns

- **Do not read the behavior's private fields to test it.** Assert on the records it produced; reach into `Behavior()` only for a targeted white-box check the grammar cannot express.
- **Do not depend on wall-clock time.** There is no mock clock in `unit` - schedule with `SendAfter` and drive it with `FireTimers`; add cron jobs and fire them with `FireCron`.
- **Do not forget to scope table-driven scenarios.** Use `Mark` + `Since(mark)` to count only what a phase produced, instead of resetting the journal.
- **Do not stub an Init-time call on the returned `Subject`.** By then `Init` has run - use `Prepare` then `Run` (and `PrepareMeta` then `Run` for a meta).
- **Do not use `Times`/`Once` with `Within` on a count that only grows** - it can overshoot n and never re-equal it; use `AtLeast`.
- **Do not test in `unit` what only the real runtime exhibits** - supervision restarts, cross-node links/monitors, real delivery. Those belong in `stage`.
- **Do not reach for `mock` when the thing under test is an actor** - use `unit`, which builds a controllable node around it and adds the input drivers.
```

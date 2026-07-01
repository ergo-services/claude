# Framework Internals

Runtime semantics that shape how diagnostics read. The system behaviours here are not surfaced by a single counter - you need the mental model to interpret the telemetry correctly.

## Important Delivery Errors

When a caller uses `SendImportant` or `CallImportant`, the framework returns one of:

| Error | Meaning | Investigate |
|-------|---------|-------------|
| `gen.ErrProcessUnknown` | Target name/PID/alias never registered - not found | Check the target's registration and application state |
| `gen.ErrProcessTerminated` | Target resolves but has already terminated / is not alive | The target's supervisor restart state and lifecycle, not its registration |
| `gen.ErrProcessMailboxFull` | Target mailbox saturated - backpressure | Target is overloaded; inspect its mailbox and RunningTime |
| `gen.ErrTimeout` | Remote node received but ACK did not arrive within 5s | Network congestion or remote overloaded |
| `gen.ErrNoConnection` | Cannot reach remote node at all | Connection lost; verify `network_nodes`, registrar |

`ErrProcessUnknown` and `ErrProcessTerminated` are distinct outcomes and need different remediation: unknown means the target was never there, terminated means it existed and died (look at its supervisor).

### Local vs Remote ACK

Important Delivery behaves differently depending on where the target lives:

- **Same node**: the routing error is returned synchronously with no round-trip. A local Important send that succeeds returns `nil` immediately.
- **Cross node**: the sender blocks up to `gen.DefaultRequestTimeout` (5s) for an ACK the remote node emits *after* it enqueues the message. The error you receive can be the remote's own routing error (`ErrProcessMailboxFull` / `ErrProcessUnknown` / `ErrProcessTerminated`) forwarded back, not only a local `ErrTimeout`. A bare `ErrTimeout` means the ACK itself never arrived.

Normal `Send` is fire-and-forget: all of the above produce silent drops. Systems that cross a node boundary and require a reliable delivery signal **must** use Important Delivery, and both nodes must set `NetworkFlags.EnableImportantDelivery = true`.

Cross-node `Call` without Important is ambiguous on failure: any failure becomes `ErrTimeout`, hiding whether the target exists.

## Supervisor Restart Intensity (Sliding Window)

`SupervisorRestart{Intensity: N, Period: S}` allows max `N` restarts within any rolling `S`-second window. Old restarts outside the window are discarded.

- `Intensity: 5, Period: 5` -> up to 5 restarts per 5 seconds; the 6th inside the window terminates the supervisor with `gen.ErrExceeded`.
- The failure cascades to the parent supervisor, which may restart the whole subtree, or also exceed its own intensity.
- `Intensity: 0` makes any single failure fatal - cascades immediately.

`process_inspect pid=<supervisor>` surfaces these fields from the supervisor's HandleInspect:
- `restarts_count` - current count within the window
- `strategy`, `intensity`, `period`
- child state (enabled/disabled, PIDs)

Cascade investigation pattern:
1. `sample_listen log_levels=["error","panic"] duration_sec=60` - capture the failure pattern.
2. `process_list max_uptime=10` - find young processes (fresh restarts).
3. `process_inspect` on each suspected supervisor.
4. If a supervisor vanished, its parent likely restarted; check the parent's `restarts_count`.

## Pool Backpressure

`act.Pool` capacity = `PoolSize * WorkerMailboxSize`. When all worker mailboxes are full:

- Normal-priority messages sent to the Pool are **dropped**, incrementing `messages_unhandled` (visible via `process_inspect` on the Pool PID).
- Messages are **not** queued at the Pool level; there is no Pool-side buffer.
- Urgent/System priority messages still reach the Pool's own `HandleMessage` (used for scale commands, introspection).

If `messages_unhandled` is growing:
- Increase `PoolSize` or `WorkerMailboxSize` (requires restart or an admin message to the Pool).
- Or shed load upstream - the Pool has no way to signal "slow down" to senders.

See `skills/framework/references/pool.md` for the design-side semantics.

## Connection Pool (Inter-Node TCP)

Each inter-node connection uses a pool of TCP connections. The pool size is determined by the **acceptor** during handshake:

- If node A (`PoolSize` setting = 10) dials node B (setting = 5), the resulting connection has **5** TCP links (B's setting, since B is the acceptor).
- If B dials A, the same pair has **10** TCP links.
- `PoolDSN` in `network_node_info` is non-nil on the dialing side.

### Ordering Invariant

Messages from the **same sender PID** always traverse the **same TCP connection** (the framework picks using an order byte derived from the sender). This guarantees per-sender ordering across the network.

Different senders may use different TCP connections in parallel; there is no cross-sender ordering guarantee.

### Reconnections

`Reconnections` in `network_node_info` counts pool-item replacements. Non-zero value means at least one TCP link in the pool has been broken and re-established. Growing = ongoing instability.

If `A.MessagesOut` to B significantly exceeds `B.MessagesIn` from A, a pool item likely dropped messages during a reconnect. Investigate physical network with external tools.

## Event Fanout

Event distribution scales with **subscribing nodes**, not subscriber count.

- Publishing to 1,000,000 subscribers all on the local node: 1,000,000 local deliveries, 0 network messages.
- Publishing to 1,000,000 subscribers spread across 10 nodes: 10 network messages (one per node), each fans out to its local subscribers.

Telemetry consequence:
- `MessagesRemoteSent` grows with **subscribing nodes x publishes**, not total subscribers.
- High `MessagesRemoteSent / MessagesPublished` = wide cluster fanout; reducing *node count* matters, not subscriber count.

Event bursts hit the publisher (CPU + memory) regardless of who subscribes; `Notify: true` lets producers skip expensive generation when nothing is listening.

## Shutdown Diagnostics

On `node.Stop()`, the framework sends shutdown signals to processes and waits up to `NodeOptions.ShutdownTimeout` (default 3 minutes) for them to terminate.

While it waits, a 5-second ticker logs a periodic **Warning** snapshot of the processes that have not terminated yet. This is a fixed poll interval, not a per-message "stuck > 5s" detector - every tick re-reads the current process table.

Each snapshot logs at most 10 running processes, plus a `...and N more` line if the total exceeds 10:

```
node mynode@host is still waiting for process(es) to terminate:
  <ABC.0.1005> (worker, myapp.Worker) state: running, queue: 3
  <ABC.0.1006> (myapp.DBConn) state: wait response, queue: 0
  ...and 4 more
```

Line format is `<pid> (<name>) state: <state>, queue: <n>`, where `<name>` is `name, behavior` when the process is registered, or just the behavior otherwise.

Reading the state field of each line:

| Log state | Queue | Meaning |
|-----------|-------|---------|
| `state: running, queue: 0` | | Stuck inside a callback - blocking operation |
| `state: running, queue: >0` | | Stuck and accumulating; same root cause, more urgent |
| `state: wait response, queue: N` | | Blocked in sync `Call`; remote not replying |
| `state: sleep, queue: 0` | | Idle; waiting for shutdown signal delivery (normal) |

### Escalation on timeout

If `ShutdownTimeout` expires with processes still alive, the framework escalates:

1. Logs an **Error**: `shutdown timeout <d> expired, killing N remaining process(es):` followed by the same snapshot format.
2. Force-kills every remaining process (`Kill` on each PID).
3. Waits a fixed 5 seconds for them to settle.
4. If any still survive, logs an **Error** `processes still alive after force kill, hard exit (N remaining):`, dumps a final snapshot, and calls `os.Exit(1)` - a hard, non-graceful exit.

Capture the whole sequence with `sample_listen log_levels=["warning","error"]`: the Warning snapshots reveal which processes are stuck, and an Error line means the node was force-killed rather than shut down cleanly.

## Meta Processes vs Actors (Mental Model)

Meta processes exist for blocking I/O (socket accept, port stdio, HTTP read). They run two goroutines:
- **Reader**: blocked on I/O syscall.
- **Dispatcher**: handles the meta's own message mailbox.

What matters for diagnostics:
- Meta processes **do** support synchronous `Call` via their Alias. `gen.MetaBehavior.HandleCall(from, ref, request)` runs on the dispatcher goroutine and its non-nil return value is routed back as the response; a meta can also use `SendResponse` / `SendResponseError` for deferred or out-of-band replies. Note: most built-in framework metas (port, tcp connection, udp server, web server) return `(nil, nil)` from `HandleCall`, which sends no response, so a synchronous `Call` to one of them blocks until the 5s request timeout (`ErrTimeout`). Only tcp server and web handler return `gen.ErrUnsupported`, and they return it as the reply VALUE with a nil error, so the caller receives `gen.ErrUnsupported` as the response payload, not as a Go error from `Call`. Custom metas may implement real request handling.
- No Link/Monitor initiation - the `gen.MetaProcess` interface has no Link/Monitor methods. A meta can **receive** them but cannot create them.
- Meta processes use `gen.Alias` as their primary address. Use `meta_inspect` (not `process_inspect`).

A meta process appearing in `pprof_goroutines` has two goroutines with labels (`-tags=pprof`):
- `{"meta":"Alias#...", "role":"reader"}`
- `{"meta":"Alias#...", "role":"handler"}`

## Sleep-State Goroutine Invisibility

A process in `sleep` state has its goroutine parked - it typically does not appear in `pprof_goroutines debug=1` summaries. This is normal, not a bug.

To catch a process's goroutine briefly visible on wake:

```
sample_start tool=pprof_goroutines arguments={"pid":"<ABC.0.1005>"} interval_ms=300 count=1 max_errors=0
```

- `interval_ms=300` - tight polling.
- `count=1` - stop on first successful capture.
- `max_errors=0` - keep retrying until the process wakes.

Requires `-tags=pprof` for per-PID goroutine lookup.

## Call Timeout Semantics

`gen.DefaultRequestTimeout = 5 seconds`. A process observed in `"wait response"` state for longer than 5s is abnormal:

- Remote target is stuck (find it: `pprof_goroutines debug=2 filter="<behavior>"`).
- Remote target is overloaded (find it: `process_list sort_by=mailbox`).
- Response was lost (check `network_node_info` both sides).
- Caller used `CallWithTimeout` with a longer timeout than default - that's by design.

## Registrar Failure Modes

A registrar returning `ErrUnsupported` for `RegisterProxy` is expected for etcd - it does not support proxy routes. `RegistrarInfo.SupportRegisterProxy` tells you whether to expect that.

If `registrar_info` reports a disconnected registrar and your cluster was previously discovering new nodes dynamically, new peers will fail to appear. Existing connections continue to work - the registrar is a discovery service, not a data path.

## Compression

Compression applies only to **cross-node** messages above the configured `Threshold`. Same-node sends never compress. Set threshold appropriately (default 1024 bytes); below that, compression overhead dominates.

Compression is configured per-process (`ProcessOptions.Compression` or runtime `SetCompression*`). `process_info` exposes the current compression configuration per process.

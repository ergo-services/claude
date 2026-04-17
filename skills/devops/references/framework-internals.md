# Framework Internals

Runtime semantics that shape how diagnostics read. The system behaviours here are not surfaced by a single counter — you need the mental model to interpret the telemetry correctly.

## Important Delivery Errors

When a caller uses `SendImportant` or `CallImportant`, the framework returns one of:

| Error | Meaning | Investigate |
|-------|---------|-------------|
| `gen.ErrProcessUnknown` | Target process does not exist on the remote node | Check the target's registration, application state |
| `gen.ErrProcessMailboxFull` | Target mailbox saturated — backpressure | Target is overloaded; inspect its mailbox and RunningTime |
| `gen.ErrTimeout` | Remote node received but ACK did not arrive within 5s | Network congestion or remote overloaded |
| `gen.ErrNoConnection` | Cannot reach remote node at all | Connection lost; verify `network_nodes`, registrar |

Normal `Send` is fire-and-forget: all of the above produce silent drops. Systems that cross a node boundary and require a reliable delivery signal **must** use Important Delivery, and both nodes must set `NetworkFlags.EnableImportantDelivery = true`.

Cross-node `Call` without Important is ambiguous on failure: any failure becomes `ErrTimeout`, hiding whether the target exists.

## Supervisor Restart Intensity (Sliding Window)

`SupervisorRestart{Intensity: N, Period: S}` allows max `N` restarts within any rolling `S`-second window. Old restarts outside the window are discarded.

- `Intensity: 5, Period: 5` → up to 5 restarts per 5 seconds; the 6th inside the window terminates the supervisor with `gen.ErrSupervisorRestartsExceeded`.
- The failure cascades to the parent supervisor, which may restart the whole subtree, or also exceed its own intensity.
- `Intensity: 0` makes any single failure fatal — cascades immediately.

`process_inspect pid=<supervisor>` surfaces these fields from the supervisor's HandleInspect:
- `restarts_count` — current count within the window
- `strategy`, `intensity`, `period`
- child state (enabled/disabled, PIDs)

Cascade investigation pattern:
1. `sample_listen log_levels=["error","panic"] duration_sec=60` — capture the failure pattern.
2. `process_list max_uptime=10` — find young processes (fresh restarts).
3. `process_inspect` on each suspected supervisor.
4. If a supervisor vanished, its parent likely restarted; check the parent's `restarts_count`.

## Pool Backpressure

`act.Pool` capacity = `PoolSize * WorkerMailboxSize`. When all worker mailboxes are full:

- Normal-priority messages sent to the Pool are **dropped**, incrementing `messages_unhandled` (visible via `process_inspect` on the Pool PID).
- Messages are **not** queued at the Pool level; there is no Pool-side buffer.
- Urgent/System priority messages still reach the Pool's own `HandleMessage` (used for scale commands, introspection).

If `messages_unhandled` is growing:
- Increase `PoolSize` or `WorkerMailboxSize` (requires restart or an admin message to the Pool).
- Or shed load upstream — the Pool has no way to signal "slow down" to senders.

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
- `MessagesRemoteSent` grows with **subscribing nodes × publishes**, not total subscribers.
- High `MessagesRemoteSent / MessagesPublished` = wide cluster fanout; reducing *node count* matters, not subscriber count.

Event bursts hit the publisher (CPU + memory) regardless of who subscribes; `Notify: true` lets producers skip expensive generation when nothing is listening.

## Shutdown Diagnostics

On `node.Stop()`, the framework sends shutdown signals to processes and waits `NodeOptions.ShutdownTimeout` (default 3 minutes). Processes stuck in a callback for > 5 seconds per message are logged with:

- `PID`
- State at the time of logging
- Current queue depth

Reading the shutdown log:

| Log state | Queue | Meaning |
|-----------|-------|---------|
| `state=running, queue=0` | | Process is stuck inside a callback — blocking operation |
| `state=running, queue>0` | | Stuck and accumulating; same root cause, more urgent |
| `state="wait response", queue=N` | | Blocked in sync `Call`; remote not replying |
| `state=sleep, queue=0` | | Idle; waiting for shutdown signal delivery (normal) |

Capture during shutdown with `sample_listen log_levels=["warning","error"]` if possible.

## Meta Processes vs Actors (Mental Model)

Meta processes exist for blocking I/O (socket accept, port stdio, HTTP read). They run two goroutines:
- **Reader**: blocked on I/O syscall.
- **Dispatcher**: handles the meta's own message mailbox.

Restrictions that matter for diagnostics:
- No `HandleCall` — there is no sync request channel into a meta process.
- No Link/Monitor initiation — a meta can **receive** them but cannot create them.
- Meta processes use `gen.Alias` as their primary address. Use `meta_inspect` (not `process_inspect`).

A meta process appearing in `pprof_goroutines` has two goroutines with labels (`-tags=pprof`):
- `{"meta":"Alias#...", "role":"reader"}`
- `{"meta":"Alias#...", "role":"handler"}`

## Sleep-State Goroutine Invisibility

A process in `sleep` state has its goroutine parked — it typically does not appear in `pprof_goroutines debug=1` summaries. This is normal, not a bug.

To catch a process's goroutine briefly visible on wake:

```
sample_start tool=pprof_goroutines arguments={"pid":"<ABC.0.1005>"} interval_ms=300 count=1 max_errors=0
```

- `interval_ms=300` — tight polling.
- `count=1` — stop on first successful capture.
- `max_errors=0` — keep retrying until the process wakes.

Requires `-tags=pprof` for per-PID goroutine lookup.

## Call Timeout Semantics

`gen.DefaultRequestTimeout = 5 seconds`. A process observed in `"wait response"` state for longer than 5s is abnormal:

- Remote target is stuck (find it: `pprof_goroutines debug=2 filter="<behavior>"`).
- Remote target is overloaded (find it: `process_list sort_by=mailbox`).
- Response was lost (check `network_node_info` both sides).
- Caller used `CallWithTimeout` with a longer timeout than default — that's by design.

## Registrar Failure Modes

A registrar returning `ErrUnsupported` for `RegisterProxy` is expected for etcd — it does not support proxy routes. `RegistrarInfo.SupportRegisterProxy` tells you whether to expect that.

If `registrar_info` reports a disconnected registrar and your cluster was previously discovering new nodes dynamically, new peers will fail to appear. Existing connections continue to work — the registrar is a discovery service, not a data path.

## Compression

Compression applies only to **cross-node** messages above the configured `Threshold`. Same-node sends never compress. Set threshold appropriately (default 1024 bytes); below that, compression overhead dominates.

Compression is configured per-process (`ProcessOptions.Compression` or runtime `SetCompression*`). `process_info` exposes the current compression configuration per process.

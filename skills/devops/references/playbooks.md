# Diagnostic Playbooks

Ten playbooks covering the most common Ergo cluster symptoms. Each follows the observe → hypothesize → test → confirm → recommend pattern. Commands use real MCP tool names and parameters from `tools.md`; counter meanings are in `counters.md`; underlying semantics in `framework-internals.md`.

When a playbook says "all nodes", fire the queries in parallel (one tool call per node in a single batch), never sequentially.

## 1. Cluster Health

**Symptoms:** "something feels off", periodic health checks, pre-deploy verification.

**Observe**
1. `cluster_nodes` — discover all nodes (self + connected + discovered).
2. Parallel for every node:
   - `node_info node=<N>`
   - `network_ping name=<N>` (from the MCP entry-point)
   - `app_list node=<N>`

**Present as a table:**

```
Node | Uptime | Processes | Zombee | Heap | SendErrorsLocal | Panics | RTT | Apps-Running/Total
```

**Flag anomalies**
- Uptime outlier — one node restarted.
- `ProcessesZombee > 0` — kill cascade or force-termination.
- Heap outlier — memory growth or leak.
- `SendErrorsLocal` or `LogMessages[4]` (Error) / `[5]` (Panic) growing — active incident.
- High ping RTT (>1s) — overloaded or network issues.
- `ApplicationsRunning < ApplicationsTotal` — something loaded but not running.
- Missing expected app in `app_list` — deployment gap.

**Confirm** by zooming into the outlier node's counters and samplers.

## 2. Performance Bottleneck

**Symptoms:** increasing request latency, specific endpoint slow, customer complaints.

**Observe**
```
process_list node=<N> sort_by=mailbox limit=10
```

**Hypothesize by pattern**

| Mailbox | Drain (MessagesIn/Wakeups) | Hypothesis | Next step |
|---------|----------------------------|-----------|-----------|
| High | `>> 1` | Message rate > processing speed | `pprof_cpu` on the actor; consider Pool |
| High | `≈ 1` | Slow callback | `pprof_cpu filter=<behavior>` to find hotspot |
| Low | High | Transient burst, handling OK | Monitor with sampler; no action |

**Test — CPU hotspot**
```
pprof_cpu node=<N> duration=5 exclude="runtime" limit=15 timeout=30
pprof_cpu node=<N> duration=5 filter="<app-package>" limit=10
```

**Test — liveness**
```
process_list node=<N> min_mailbox=1 min_uptime=60 min_messages_in=1 sort_by=mailbox_latency limit=10
```

For each candidate compute `Liveness = RunningTime_ns / (Uptime_sec * MailboxLatency_ns)`:
- `> 0.01` — overloaded but alive → scale out, add Pool, or shed load.
- `< 0.001` — callback blocked → next step below.

**Confirm blocked callback**
```
pprof_goroutines node=<N> debug=2 filter="<behavior>" limit=5 timeout=60
```

Look for `select (no cases)`, `chan receive`, `sync.(*Mutex).Lock`, blocking syscalls inside the actor's goroutine.

**Recommend**
- Overloaded but alive: scale (Pool size, more workers, shard by key).
- Blocked callback: root-cause and remove the blocking call; it violates the actor model.

## 3. CPU Profiling

**Snapshot (quick top-N):**
```
pprof_cpu node=<N> duration=5 limit=20
```

**Filtered (application code only):**
```
pprof_cpu node=<N> duration=5 filter="<app-pkg>" exclude="runtime" limit=10
```

**Continuous (goroutine sampling, surfaces which actors appear across samples):**
```
sample_start node=<N> tool=pprof_goroutines arguments={"debug":1,"filter":"ProcessRun","exclude":"toolPprof","limit":20} interval_ms=500 duration_sec=30 linger_sec=30
sample_read sampler_id=<id> node=<N>
```

This is not a true CPU profile, but an actor-goroutine presence map — useful when CPU time is spread across many short callbacks.

## 4. Memory Growth

**Symptoms:** RSS trending up, GC pauses, eventual OOM.

**Observe**
```
runtime_stats node=<N>              # heap_alloc, num_gc, last_gc_pause_ns
pprof_heap node=<N> limit=20        # top allocators
pprof_heap node=<N> limit=20 filter="<app-pkg>"  # only application allocators
```

**Trend**
```
sample_start node=<N> tool=runtime_stats interval_ms=5000 duration_sec=300 linger_sec=60
```

After collection: `sample_read sampler_id=<id> node=<N>` — plot `heap_alloc` and `heap_objects` over time.

**Interpret**
- `MemoryUsed` growing without recovery after GC → live leak.
- `MemoryAlloc - MemoryUsed` growing faster than `MemoryUsed` → high churn / GC pressure.
- Top allocator in `pprof_heap` from application code → leak suspect.
- Deep mailboxes (`process_list sort_by=mailbox`) — messages holding memory waiting to be processed.

**Recommend**
- Identify the allocator callsite, add pooling or bound the allocation.
- If mailbox growth is the cause, apply Pool with bounded `WorkerMailboxSize` or switch to `SendImportant` + backpressure at sender.

## 5. Process Leak

**Symptoms:** `ProcessesTotal` climbing, open FDs growing, RSS growing.

**Observe**
```
node_info node=<N>
```
Check: `ProcessesSpawned - ProcessesTerminated ≈ ProcessesTotal`. Spawned >> Terminated over time = leak.

**Find the leaking family**
```
process_list node=<N> max_uptime=60 limit=50              # recent spawns
process_list node=<N> sort_by=uptime limit=50             # oldest first; find unbounded families
```

Look for processes with no registered name (anonymous, dynamically spawned) accumulating under a specific Application or behavior.

**Find the parent**
```
process_children node=<N> parent=<supervisor-name> recursive=true
```

Count subtree size; compare against expected.

**Trend**
```
sample_start node=<N> tool=node_info interval_ms=10000 duration_sec=300
```

Watch `ProcessesSpawned` and `ProcessesTerminated` deltas.

**Recommend**
- If children are created on-demand without a cap, add a `Pool` or a `SimpleOneForOne` supervisor with explicit `StartChild` lifecycle.
- If they should self-terminate but don't, find the missing termination path (unhandled case in `HandleMessage`, monitor left in place, link that never triggers).

## 6. Restart Loop

**Symptoms:** error/panic log spikes, brief `ProcessesTotal` dip and recovery, child behavior "flapping".

**Observe**
```
process_list node=<N> max_uptime=10 limit=50         # very young processes
sample_listen node=<N> log_levels=["error","panic"] duration_sec=60 linger_sec=30
```

After `duration_sec`: `sample_read sampler_id=<id> node=<N>`.

**Identify the supervisor**
```
process_list node=<N> behavior=Supervisor limit=50
process_inspect node=<N> pid=<supervisor-pid>
```

Look in the inspect output for `restarts_count`, `strategy`, `intensity`, `period`. A supervisor at or near its intensity limit is a prime suspect.

**Increase logging if needed (requires user permission — it's an action tool)**
```
log_level_set node=<N> target=<behavior-or-pid> level=debug
```

**Recommend**
- Fix the root-cause panic/error (not the symptom).
- If restart intensity is too aggressive, loosen it — but only after fixing the underlying bug.
- If the child exits normally (expected) but the parent uses `Permanent` strategy, switch to `Transient`.

## 7. Zombie Processes

**Symptoms:** `ProcessesZombee > 0` in `node_info`, persistent.

**Observe**
```
process_list node=<N> state=zombee limit=50
```

For each zombie:
```
process_info node=<N> pid=<ZOMBIE-PID>
```

Fields of interest: `Parent`, `Behavior`, `MailboxQueues` (non-zero = still had messages when killed), last state transition.

**Find the stuck goroutine (requires -tags=pprof on target)**
```
pprof_goroutines node=<N> debug=2 filter="<behavior>" limit=5 timeout=60
```

Typical patterns:
- Goroutine in `select (no cases)` — leaked channel receive.
- Goroutine in `runtime_pollWait` — blocking syscall in a callback.
- Goroutine in `sync.(*Mutex).Lock` — lock contention.

All three are violations of the actor model.

**Recommend**
- Identify the blocking call site and remove it.
- Zombies do not self-clean; they persist until the node restarts. A growing zombie count is a leak.

## 8. Network Issues

**Symptoms:** cross-node calls fail, `CallErrorsRemote` growing, intermittent timeouts.

**Quick check**
```
network_ping name=<suspect>
```

Fail or RTT > 1s = problem. RTT measures the full path (flusher, TCP, remote MCP pool, response).

**Both-sided investigation — always compare**
```
network_node_info node=A name=B    # A's view of B
network_node_info node=B name=A    # B's view of A
```

**Interpret by symptom**

| Symptom | Root cause |
|---------|-----------|
| Ping fails entirely | No connection, or MCP app not running on target |
| Ping RTT > 1s | Target overloaded or network congested |
| `A.MessagesOut` >> `B.MessagesIn` | Silent data loss (pool item broken) |
| `Reconnections` > 0 and growing | Unstable connection; check physical network |
| `ConnectionUptime << Uptime` | Connection recently re-established (flapped) |
| `HandshakeErrors > 0` on `network_acceptors` | Cookie mismatch, version incompatibility, or scanning |

**Deep investigation**
```
network_info node=<N>                              # flags, max message size, routes
network_acceptors node=<N>                         # per-acceptor handshake errors
registrar_info node=<N>                            # registrar health
registrar_resolve node=<N> name=<target>           # can we discover the target at all?
```

**Recommend**
- Silent data loss: restart the connection (requires permission — `network_disconnect` + reconnect).
- Cookie mismatches: verify env vars, check `HandshakeErrors` after deploy.
- Chronic instability: check MTU, keep-alive settings, L4 load balancer idle timeouts.

## 9. Event System Issues

**Symptoms:** subscribers not receiving, fanout load, publishers saturating.

**Wasted publishing**
```
event_list node=<N> utilization_state=no_subscribers limit=10
```

Each entry is an event publishing into the void. Producers either:
- Should enable `Notify: true` and gate on `MessageEventStart/Stop`.
- Or stop publishing if nobody needs the event.

**Starved subscribers**
```
event_list node=<N> utilization_state=no_publishing limit=10
```

Subscribers exist but the producer is idle. Check producer process state.

**High fanout**
```
event_list node=<N> sort_by=subscribers limit=10
event_list node=<N> sort_by=remote_sent limit=10    # network fanout specifically
```

Remote fanout scales with **subscribing nodes**, not subscribers. If `remote_sent / published` is high, consider reducing the number of subscribing nodes (e.g., aggregate on the producer side).

**Inspect a specific event**
```
event_info node=<N> name=<event-name>
sample_listen node=<N> event=<event-name> duration_sec=30    # see actual published data
```

## 10. Goroutine Investigation

**Always filter on remote nodes.** Unfiltered goroutine dumps blow up the proxy transport.

**Actor summary (start here)**
```
pprof_goroutines node=<N> debug=1 filter="ProcessRun" exclude="toolPprof" limit=20
```

Shows actor goroutines only. Header tells total / matched / showing.

**Specific behavior**
```
pprof_goroutines node=<N> debug=2 filter="<behavior-type>" limit=5
```

`debug=2` includes full stacks — use for deep inspection of a few goroutines.

**Stuck in sync Call**
```
pprof_goroutines node=<N> debug=2 filter="waitResponse" limit=10
```

Finds processes blocked in `Call()` awaiting a response that never arrived.

**Exclude IO-waiting goroutines**
```
pprof_goroutines node=<N> debug=1 exclude="runtime_pollWait" limit=30
```

Surfaces non-network-IO goroutines.

**Large cluster — raise timeout**
```
pprof_goroutines node=<N> debug=1 filter="..." limit=20 timeout=60
```

**Per-PID (requires -tags=pprof on target)**
```
pprof_goroutines node=<N> pid=<PID>
```

If the process is in `sleep` state, its goroutine is parked — dump may be empty. Poll on wake:
```
sample_start node=<N> tool=pprof_goroutines arguments={"pid":"<PID>"} interval_ms=300 count=1 max_errors=0
```

## Cross-References

- Tool parameters: `tools.md`.
- Counter semantics: `counters.md`.
- Process-state / mailbox / liveness interpretation: `process-model.md`.
- Runtime internals (Important Delivery, restart intensity, pool backpressure, connection pool, event fanout): `framework-internals.md`.
- Samplers — active/passive, linger, proxy: `samplers.md`.
- Build tag requirements (`-tags=pprof`, `-tags=latency`): `build-tags.md`.

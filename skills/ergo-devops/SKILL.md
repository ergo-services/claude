---
name: ergo-devops
description: Diagnose production issues in running Ergo Framework nodes via MCP server. Covers performance bottlenecks, process leaks, memory issues, network problems, event fanout, restart loops, and stuck processes. Use when investigating issues on live Ergo nodes.
---

# Ergo MCP DevOps

Diagnose production issues in running Ergo Framework distributed systems via MCP.

## Network Transparency

Every tool accepts `node` parameter -- the framework establishes connections automatically. Never call `network_connect` before querying. Just pass `node=<name>`.

All nodes are equal. The MCP endpoint is just a proxy, not a primary node.

## MCP Connection

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp
```

## Process Model

**States:** Init -> Sleep <-> Running -> Terminated. WaitResponse during sync Call. Zombee on forced kill.

| Metric | Meaning |
|--------|---------|
| MessagesMailbox | Current queue depth |
| MailboxLatency | Age of oldest message (-tags=latency) |
| RunningTime/Uptime | Utilization ratio (NOT CPU) |
| Drain (MessagesIn/Wakeups) | ~1 = idle, >>1 = overloaded |

**MailboxSize = -1** means unlimited (default). Can accumulate messages indefinitely → memory growth. If > 0, `ErrProcessMailboxFull` on overflow.

**Default Call timeout = 5 seconds.** WaitResponse > 5s is abnormal.

**LogMessages in node_info:** `[0]=Trace [1]=Debug [2]=Info [3]=Warning [4]=Error [5]=Panic`. Watch `[4]` and `[5]`.

## Framework Internals

**Important Delivery errors:** `ErrProcessUnknown` = doesn't exist, `ErrProcessMailboxFull` = overloaded, `ErrTimeout` = ACK lost (5s), `ErrNoConnection` = unreachable.

**Supervisor restart intensity:** Sliding window (default 5 restarts / 5 sec). Exceeded = supervisor dies with `ErrSupervisorRestartsExceeded`, cascades up. Inspect supervisor for `restarts_count`.

**Pool backpressure:** Capacity = PoolSize * WorkerMailboxSize. All full = messages dropped. Inspect pool for `messages_unhandled`.

**Connection pool:** Default 3 TCP connections per peer. Same sender = same TCP (ordered). Different senders = parallel. Pool size determined by **acceptor** (receiving side) during handshake. `PoolDSN` non-null = this side dialed. `Reconnections` counter = instability.

**Goroutine labels (with -tags=pprof):** Actors: `{"pid":"<ABC.0.1005>"}`. Meta: `{"meta":"Alias#...", "role":"reader"}`.

**Error counters in node_info:** `SendErrorsLocal` (process unknown/terminated/mailbox full), `SendErrorsRemote` (connection failures), `CallErrorsLocal`, `CallErrorsRemote`. Non-zero = problems.

**Connection counters in network_info:** `ConnectionsEstablished - ConnectionsLost = active`. Growing `ConnectionsLost` = instability. `HandshakeErrors` = auth problems.

**Shutdown:** Processes >5s per message are logged. `state=running, queue=0` = stuck in callback.

## Tools Quick Reference

### Profiling (use filter/exclude on remote nodes)

```
pprof_goroutines debug=1 filter="ProcessRun" exclude="toolPprof" limit=20   # actor goroutines
pprof_goroutines debug=2 filter="<behavior>" limit=5                         # specific actor trace
pprof_cpu duration=5 exclude="runtime" limit=15                              # CPU hotspots
pprof_heap limit=20 filter="myapp"                                           # memory allocators
```

All pprof tools support `filter` (include matching) and `exclude` (remove matching) by substring.
Response header shows `total N, matched M, showing K`.

### Network

```
network_ping name=<node>                           # quick alive check with RTT
network_node_info node=A name=B                    # A's view of connection to B
network_node_info node=B name=A                    # B's view (always check BOTH sides)
```

Connection info includes `Reconnections` counter -- non-zero indicates instability.

### Samplers

```
sample_start tool=<tool> arguments={...} interval_ms=2000 duration_sec=60 linger_sec=30
sample_listen log_levels=["warning","error"] duration_sec=60 linger_sec=30
sample_read sampler_id=<id>
sample_list                                         # shows "completed, lingering Ns"
sample_stop sampler_id=<id>                         # immediate terminate
```

`linger_sec` (default 30): sampler stays alive after completion for data retrieval. No rush to read before duration expires.

Passive samplers also capture messages and calls sent to them -- use as test receivers: `send_message to=<sampler_id> type_name=X message={...}`, then `sample_read` to see captured entries with `from`, `type`, `message`.

**Stopping:** read data first (`sample_read`), present to user, then `sample_stop`. Exception: user says "just kill it".

### Typed Messages

```
message_types filter="order"                                    # discover types
message_type_info type_name="TestOrder"                         # inspect fields
send_message to=<target> type_name="TestOrder" message={...}    # send typed struct
call_process to=<target> type_name="GetStatusRequest" request={...}  # sync call
```

Short name works (e.g., `TestOrder`). Go→JSON: string→`"v"`, number→`42.5`, bool→`true`, *T→value or `null`, []T→`[...]`, map→`{...}`, struct→nested `{...}`.

### Proxy Timeout

All tools accept `timeout` parameter (seconds, default 30, max 120) for remote calls. Use higher timeout for heavy operations:
```
pprof_cpu node=X duration=10 timeout=60
pprof_goroutines node=X debug=2 limit=100 timeout=60
```

## Diagnostic Playbooks

### 1. Cluster Health

1. `cluster_nodes` -- discover all
2. In parallel: `node_info node=X` + `network_ping name=X` for every node
3. Table: Node, Uptime, Processes, Zombee, Heap, Errors, Ping RTT
4. Flag anomalies by comparing rows

### 2. Performance Bottleneck

1. `process_list sort_by=mailbox limit=10` -- find suspects
2. Pattern: High mailbox + high drain = overloaded. High mailbox + low drain = slow callbacks
3. `pprof_cpu duration=5 exclude="runtime"` -- where is CPU spent?
4. `process_info` on suspect -- confirm with queue sizes

### 3. CPU Profiling

**Snapshot:** `pprof_cpu duration=5 limit=20` -- top functions

**Application only:** `pprof_cpu duration=5 filter="myapp" exclude="runtime"` -- filter noise

**Continuous (goroutine sampling):**
```
sample_start tool=pprof_goroutines arguments={"debug":1,"filter":"ProcessRun","exclude":"toolPprof","limit":20} interval_ms=500 duration_sec=30
```

### 4. Memory Growth

1. `pprof_heap limit=20` -- top allocators (inuse + cumulative alloc columns)
2. `runtime_stats` -- heap_alloc, num_gc
3. Trend: `sample_start tool=runtime_stats interval_ms=5000 duration_sec=300`

### 5. Process Leak

1. `node_info` -- Spawned vs Terminated
2. `process_list max_uptime=60 limit=50` -- recent spawns
3. Trend: `sample_start tool=node_info interval_ms=10000`

### 6. Restart Loop

1. `process_list max_uptime=10 limit=50` -- very young processes
2. `sample_listen log_levels=["error","panic"] duration_sec=60`
3. `process_inspect` on supervisor -- restarts_count

### 7. Zombie Processes

1. `process_list state=zombee`
2. `pprof_goroutines debug=2 filter="<behavior_name>" limit=5` -- find by behavior name

### 8. Network Issues

**Always check both sides:**
```
network_node_info node=A name=B   # A's view
network_node_info node=B name=A   # B's view
```

| Symptom | Root Cause |
|---------|-----------|
| Ping fails or RTT > 1s | Connection problem or overloaded node |
| MessagesOut >> MessagesIn (cross-check) | Silent data loss |
| High Reconnections | Unstable connection |

### 9. Goroutine Investigation

```
pprof_goroutines debug=1 filter="ProcessRun" exclude="toolPprof" limit=20    # actors only
pprof_goroutines debug=2 filter="waitResponse" limit=10                       # stuck in Call
pprof_goroutines debug=1 exclude="runtime_pollWait" limit=30                  # non-IO goroutines
```

### 10. Event System

1. `event_list utilization_state=no_subscribers` -- wasted publishing
2. `event_list sort_by=subscribers limit=10` -- highest fanout
3. `sample_listen event=<name> duration_sec=30` -- see data

## Build Tags

| Tag | Enables | Without It |
|-----|---------|-----------|
| `-tags=pprof` | Per-process goroutine traces | pid param errors |
| `-tags=latency` | MailboxLatency measurement | Returns -1 |

## Common Mistakes

- **Connecting before querying** -- never use `network_connect` just to query, framework connects automatically
- **Unfiltered pprof on remote** -- too large, will timeout. Always use filter/exclude
- **One-sided network check** -- always compare both sides of connection
- **Snapshot vs trend** -- use samplers for trends, not just single queries
- **Missing timeout** -- use `timeout=60` for heavy remote operations
- **pprof "not found"** -- sleeping processes park goroutines, use sampler to poll
- **Dumping everything** -- always use sort_by, limit, filter

---
name: ergo-devops
description: Diagnose production issues in running Ergo Framework nodes via MCP server. Covers performance bottlenecks, process leaks, memory growth, restart loops, zombie processes, network failures, event system issues, goroutine deadlocks, and cluster health assessment. Use when investigating issues on live Ergo nodes.
---

# Ergo MCP DevOps

Diagnose and investigate production issues in running Ergo Framework distributed systems. Connect to live nodes via MCP server and run diagnostic sequences.

## When to Use This Skill

- Investigating slow responses or growing queues
- Finding process leaks (count growing without plateau)
- Debugging restart loops (processes keep crashing)
- Analyzing memory growth
- Diagnosing network connectivity issues between nodes
- Investigating event system problems (missing subscribers, fanout overload)
- Detecting goroutine leaks or deadlocks
- Running cluster-wide health checks
- Setting up real-time monitoring sessions

## MCP Connection

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp
```

All tools accept optional `node` parameter for cluster proxy -- request is forwarded to the remote node via native Ergo inter-node protocol.

## Process Model Quick Reference

**States:** Init -> Sleep <-> Running -> Terminated. WaitResponse during sync Call. Zombee on forced kill.

**Mailbox:** Urgent > System > Main > Log (priority order).

**Key metrics:**

| Metric | Meaning | Diagnostic Use |
|--------|---------|---------------|
| MessagesMailbox | Current queue depth | Backpressure indicator |
| MailboxLatency | Age of oldest message (needs -tags=latency) | Real latency, not just depth |
| RunningTime | Cumulative callback time (ns) | CPU consumption |
| Wakeups | Sleep->Running transitions | Activity level |
| Drain (MessagesIn/Wakeups) | Messages per wake cycle | ~1 = idle, >>1 = overloaded |
| InitTime | ProcessInit duration (ns) | Slow startup detection |

## Sampler Quick Reference

### Active (sample_start) -- periodic tool calls

```
sample_start tool=<any_mcp_tool> arguments={...} interval_ms=5000 duration_sec=60
```

- Calls any MCP tool periodically, stores results in ring buffer
- `count` -- stop after N successful results
- `max_errors=0` -- ignore errors, keep retrying (useful for polling rare conditions)
- Duration always applies as upper bound (default 60s, max 3600s)

### Passive (sample_listen) -- event-driven capture

```
sample_listen log_levels=["warning","error"] duration_sec=60
sample_listen event=my_event duration_sec=60
sample_listen log_levels=["error"] event=my_event duration_sec=60  # combined
```

### Reading

```
sample_read sampler_id=mcp_sampler_xxx since=0     # all buffered
sample_read sampler_id=mcp_sampler_xxx since=5     # incremental (seq > 5)
```

### Management

```
sample_list     # all active samplers with full inspect data
sample_stop sampler_id=mcp_sampler_xxx
```

## Diagnostic Playbooks

### 1. Performance Bottleneck

**Symptom:** slow responses, growing queues.

**Investigation:**
1. `process_list sort_by=mailbox limit=10` -- deepest mailboxes
2. `process_list sort_by=drain limit=10` -- highest drain ratio
3. `process_info` on suspect -- queue sizes, links
4. `runtime_stats` -- GC pressure
5. Trend: `sample_start tool=process_list arguments={"sort_by":"mailbox","limit":5} interval_ms=2000`

**Pattern matching:**

| Mailbox | Drain | Root Cause | Fix |
|---------|-------|-----------|-----|
| High | High (>>1) | Message rate > processing | Pool, load shedding |
| High | Low (~1) | Slow callbacks | Optimize code |
| Low | High | Burst handling OK | Monitor, may be transient |

### 2. Process Leak

**Symptom:** ProcessesTotal keeps growing.

**Investigation:**
1. `node_info` -- ProcessesSpawned vs ProcessesTerminated
2. `process_list max_uptime=60 limit=50` -- recently spawned
3. `process_children parent=<supervisor> recursive=true` -- subtree sizes
4. `app_list` -- per-app process counts
5. Trend: `sample_start tool=node_info interval_ms=10000 duration_sec=300`

**Red flags:** Spawned >> Terminated. Nameless processes accumulating. App with abnormal process count.

### 3. Restart Loop

**Symptom:** process keeps crashing.

**Investigation:**
1. `process_list max_uptime=10 limit=50` -- very young processes
2. `process_inspect` on supervisor -- restarts_count
3. `sample_listen log_levels=["error","panic"] duration_sec=60` -- crash patterns
4. `log_level_set target=<name> level=debug` -- increase logging

**Red flags:** Multiple processes with uptime < 10s. Growing restarts_count. Repeating error in log stream.

### 4. Zombie Processes

**Symptom:** processes in Zombee state.

**Investigation:**
1. `process_list state=zombee`
2. `process_info` on zombie -- parent, links
3. `process_state` on parent -- alive?
4. Catch stack: `sample_start tool=pprof_goroutines arguments={"pid":"<PID>"} interval_ms=300 count=1 max_errors=0 duration_sec=30`

**Red flags:** Goroutine stuck on channel/mutex = blocking in actor callback (forbidden). Parent in Running state = stuck callback.

### 5. Memory Growth

**Symptom:** heap_alloc keeps increasing.

**Investigation:**
1. `runtime_stats` -- snapshot
2. `pprof_heap` -- top allocators
3. Trend: `sample_start tool=runtime_stats interval_ms=5000 duration_sec=300`
4. `process_list sort_by=mailbox limit=10` -- message accumulation
5. `pprof_goroutines limit=200 debug=1` -- goroutine leak

**Red flags:** heap_alloc growing without GC recovery. heap_objects growing = object leak. Goroutine count growing. Deep mailboxes = messages piling up.

### 6. Network Issues

**Symptom:** remote calls timeout.

**Investigation:**
1. `cluster_nodes` -- overview
2. `network_nodes` -- traffic counts
3. `network_node_info name=<suspect>` -- connection details
4. `registrar_info` + `registrar_resolve name=<node>` -- route resolution
5. `network_connect name=<target>` -- try connecting

**Pattern matching:**

| Symptom | Root Cause |
|---------|-----------|
| Discovered, not connected | Route exists, connection fails |
| No bytes_in | Connection dead |
| Low ConnectionUptime | Flapping |
| Messages_out >> messages_in | One-directional failure |

### 7. Event System Issues

**Symptom:** events publishing to void, missing subscribers.

**Investigation:**
1. `event_list utilization_state=no_subscribers` -- wasted publishing
2. `event_list utilization_state=no_publishing` -- waiting subscribers
3. `event_list sort_by=subscribers limit=10` -- highest fanout
4. `event_list producer=<name>` -- all events owned by a specific process
5. `event_info name=<suspect>` -- producer, subscribers
6. `sample_listen event=<suspect> duration_sec=30` -- see published data

### 8. Goroutine Investigation

**Symptom:** deadlock suspected, goroutine count growing.

**Investigation:**
1. `runtime_stats` -- goroutine count
2. `pprof_goroutines debug=1 limit=100` -- summary by stack
3. `pprof_goroutines debug=2 limit=50` -- full traces
4. Catch specific actor: `sample_start tool=pprof_goroutines arguments={"pid":"<PID>"} interval_ms=300 count=1 max_errors=0 duration_sec=30`
5. `process_list state=wait_response` -- stuck in sync Call

**Red flags:** `chanrecv`/`chansend` = channel deadlock. `sync.Mutex.Lock` = forbidden in actors. Multiple wait_response = circular Call dependency.

### 9. Cluster Health Check

**Quick assessment:**
1. `cluster_nodes` -- all nodes
2. `node_info` -- local summary
3. `node_info` with `node` param -- for each remote node
4. `network_nodes` -- inter-node traffic
5. `registrar_resolve_app name=<critical_app>` -- deployment check

Compare across nodes: uptime, process counts, memory, application states.

## Real-Time Monitoring Setup

Start multiple samplers for continuous monitoring:

```
sample_start tool=node_info interval_ms=10000 duration_sec=600
sample_start tool=runtime_stats interval_ms=5000 duration_sec=600
sample_start tool=process_list arguments={"sort_by":"mailbox","limit":10} interval_ms=2000 duration_sec=600
sample_start tool=process_list arguments={"sort_by":"running_time","limit":10} interval_ms=5000 duration_sec=600
sample_listen log_levels=["error","panic"] duration_sec=600
sample_start tool=network_nodes interval_ms=30000 duration_sec=600
```

Read incrementally with `sample_read since=<last_sequence>`. Check status with `sample_list`.

## Build Tag Awareness

| Tag | Enables | Without It |
|-----|---------|-----------|
| `-tags=pprof` | `pprof_goroutines pid=...` for per-process stack traces | pid param returns error |
| `-tags=latency` | MailboxLatency measurement, `mailbox_latency` sort/filter | Latency fields return -1 |

**Sleeping process + pprof:** goroutine is parked, won't appear in dump. Use `sample_start tool=pprof_goroutines arguments={"pid":"..."} interval_ms=300 count=1 max_errors=0` to poll until it wakes up.

## Common Mistakes

- **Snapshot vs trend:** single `process_list` shows one moment. Use samplers for trends over time
- **pprof "not found" vs sleeping:** error says "pprof IS enabled" = process sleeping, not missing tag. Poll with sampler
- **Mailbox depth without drain:** same depth has different causes at different drain ratios
- **Action tools without permission:** never send_exit or process_kill without explicit user OK
- **Unbounded samplers:** always set duration_sec. Default is 60s, max 3600s
- **Dumping everything:** use sort_by and limit, not limit=0

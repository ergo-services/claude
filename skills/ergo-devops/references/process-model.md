# Process Model

The operational reference for interpreting `process_list`, `process_info`, `process_inspect`, and the metrics inside them. Covers states, mailbox queues, metric fields, the liveness formula, and how to read backpressure from telemetry.

## Process States

Returned as strings by `process_state` and inside `ProcessShortInfo.State`.

| State | Constant | Meaning | Allowed ops |
|-------|----------|---------|-------------|
| `init` | `ProcessStateInit` | Running `ProcessInit`. Not yet registered, no PID lookup. | Spawn, Send async, setters. No Call, Link, Monitor, RegisterName. |
| `sleep` | `ProcessStateSleep` | Idle, mailbox empty. Goroutine is parked. | All operations. |
| `running` | `ProcessStateRunning` | Dispatching a message. | All operations. |
| `"wait response"` | `ProcessStateWaitResponse` | Blocked in `Call()` awaiting reply. | Limited. Abnormal if > 5s (default Call timeout). |
| `terminated` | `ProcessStateTerminated` | Shutting down. | Send-only. |
| `zombee` | `ProcessStateZombee` | Killed forcibly. Goroutine orphaned. | Nothing. |

Zombies persist in the process table — they are a leak indicator. Investigate with `pprof_goroutines debug=2 filter=<behavior>` to find the stuck goroutine.

## Mailbox Queues (Priority)

Every process has four mailbox queues dispatched in priority order:

| Priority | Constant | Queue | Typical use |
|----------|----------|-------|-------------|
| Max | `MessagePriorityMax` | Urgent | Kill, admin controls |
| High | `MessagePriorityHigh` | System | Framework messages |
| Normal | `MessagePriorityNormal` | Main | Application messages (default) |
| — | — | Log | Log messages (if process is a logger) |

Senders pick a priority via `SendWithPriority` or the process's default `SendPriority`. Every pop checks Urgent → System → Main → Log and returns the first available message.

## MailboxSize

- `MailboxSize = 0` (default): **unlimited**. Mailbox can grow without bound → source of silent memory growth. Watch `MessagesMailbox` in `process_list sort_by=mailbox`.
- `MailboxSize > 0`: senders receive `gen.ErrProcessMailboxFull` on overflow, unless `ProcessFallback` is configured.

## ProcessShortInfo Fields (visible in process_list)

| Field | Meaning | How to use |
|-------|---------|------------|
| `PID` | Unique identifier | Always cite as `<ABC.0.1005>` |
| `Name` | Registered name | Empty if not registered |
| `Behavior` | Behavior type (e.g. `myapp.Worker`) | Useful in pprof goroutine filters |
| `Application` | Owning application (empty for root processes) | |
| `State` | See table above | |
| `Parent` | Parent PID | |
| `Leader` | Application leader PID | |
| `Uptime` | Seconds since spawn | |
| `MessagesIn` | Cumulative messages received | |
| `MessagesOut` | Cumulative messages sent | |
| `MessagesMailbox` | **Current** queue depth (all 4 queues combined) | Backpressure indicator |
| `MailboxLatency` | Age of oldest unhandled message (ns) | Requires `-tags=latency`; `-1` if disabled |
| `RunningTime` | Cumulative callback time (ns) | `RunningTime / Uptime` = utilization |
| `InitTime` | `ProcessInit` duration (ns) | Slow init detection |
| `Wakeups` | Sleep→Running transitions | Denominator for Drain |

## Drain (computed in sort_by)

```
Drain = MessagesIn / Wakeups
```

| Drain | Interpretation |
|-------|----------------|
| `≈ 1` | One message per wake. Idle-reactive; no backpressure. |
| `> 1` | Multiple messages per wake. Queue was building between wake-ups → overloaded but keeping up. |
| `>> 1` | Severe build-up; likely bottleneck. |

`sort_by=drain` in `process_list` surfaces the hottest actors.

## Utilization

```
Utilization = RunningTime_ns / (Uptime_sec * 1e9)
```

Interpretation:
- Close to 1.0: actor spends nearly all time in callbacks; bottleneck candidate.
- Close to 0: idle or very sparse load.

Utilization is **not** CPU. Two actors each with utilization 1.0 on a multi-core machine do not saturate the CPU — utilization is a per-actor busy ratio.

## MailboxLatency (requires -tags=latency)

`MailboxLatency_ns` is the age of the oldest message still in the queue (from enqueue to the current time). High latency + low utilization = the process is not processing its mailbox (stuck).

A transient non-zero latency (ms range) during active work is normal — it just means we caught the process mid-callback.

## Liveness Score

Detects blocked callbacks (actor holding a mutex, blocked on a channel, doing blocking I/O in a handler). Combines utilization with mailbox latency.

```
Liveness = RunningTime_ns / (Uptime_sec * MailboxLatency_ns)
```

Equivalent to: "how much callback work happens per unit of waiting message age".

### Collection

```
process_list min_mailbox=1 min_uptime=60 min_messages_in=1 sort_by=mailbox_latency limit=10
```

Filters: pending messages (`min_mailbox=1`), alive > 60s (warmup), has received any messages (excludes brand-new processes), sorted by oldest message age. Exclude processes in `zombee` state from results.

### Interpretation

| MailboxLatency | Liveness | Diagnosis | Action |
|----------------|----------|-----------|--------|
| High (seconds) | `> 0.01` | Overloaded but alive — can't drain fast enough | Scale: pool, more workers, or shed load |
| High (seconds) | `< 0.001` | Callback blocked — holding mailbox without processing | Find blocking call: `pprof_goroutines debug=2 filter=<behavior>` |
| 0 | n/a | Healthy, mailbox empty | No action |
| Transient (ms) | any | Caught mid-callback | Normal — re-sample |

**Worked example:** same 18s MailboxLatency, two different processes.
- `RunningTime = 5000s, Uptime = 600s` → `Liveness = 5000e9 / (600 * 18e9) ≈ 0.46` — heavily busy but alive.
- `RunningTime = 82µs, Uptime = 600s` → `Liveness = 82e3 / (600 * 18e9) ≈ 7.6e-12` — stuck.

## Call Timeout

`gen.DefaultRequestTimeout` = 5 seconds. A process stuck in `"wait response"` for > 5s is abnormal — either the remote never returned or the response was lost. Check with `pprof_goroutines debug=2 filter=waitResponse`.

## Restart Detection

- `max_uptime=10` returns processes younger than 10s. Many rows here during steady-state = restart loop somewhere.
- Inspect the parent supervisor with `process_inspect pid=<supervisor>` to see `restarts_count`, strategy, intensity, period.
- `ErrSupervisorRestartsExceeded` on a supervisor means its sliding window filled up — escalation propagates to the parent supervisor. See `framework-internals.md`.

## Meta Processes

Meta processes are identified by `gen.Alias`, not `gen.PID`. Use `meta_inspect alias=<alias>` (not `process_inspect`). Their info lives in a separate `MetaInfo` structure.

## Metrics to Watch per Symptom

| Symptom | Fields |
|---------|--------|
| Slow messages on one actor | `MessagesMailbox`, `MailboxLatency`, `RunningTime/Uptime`, Drain |
| Memory growth | `MessagesMailbox` across all (default unlimited mailbox), heap via `pprof_heap`, `node_info.MemoryUsed` |
| Stuck callbacks | Liveness score — low value with non-zero latency |
| Process leak | `node_info.ProcessesSpawned` vs `ProcessesTerminated` |
| Restart loop | `process_list max_uptime=10`, `process_inspect` on supervisor |
| Zombie accumulation | `process_list state=zombee` |
| Pool saturation | `process_inspect` on Pool PID — `messages_unhandled` counter |

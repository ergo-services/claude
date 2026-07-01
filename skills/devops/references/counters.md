# Counters

Every counter surfaced by MCP tools, grouped by the struct that carries it. Counters are either **cumulative** (monotonically increasing since node start; deltas over time reveal activity) or **instantaneous** (current value; snapshots reveal state).

## NodeInfo (from `node_info`)

### Process counters

| Counter | Kind | Meaning / Diagnostic use |
|---------|------|--------------------------|
| `ProcessesTotal` | instant | All processes on the node |
| `ProcessesRunning` | instant | Currently in Running state (dispatching) |
| `ProcessesWaitResponse` | instant | Blocked in sync `Call()`. Persistently high = stuck Calls. |
| `ProcessesZombee` | instant | Killed processes, goroutines orphaned. Should stay 0. Non-zero = leak. |
| `ProcessesSpawned` | cumulative | Total successful spawns |
| `ProcessesSpawnFailed` | cumulative | Failed spawns (factory errors, name taken, init errors) |
| `ProcessesTerminated` | cumulative | Total terminations |

**Process leak check:** `ProcessesSpawned - ProcessesTerminated ≈ ProcessesTotal`. Trend `ProcessesSpawned - ProcessesTerminated` over time with `sample_start tool=node_info interval_ms=10000`.

### Error counters

| Counter | Meaning | Root cause to investigate |
|---------|---------|---------------------------|
| `SendErrorsLocal` | Local `Send` failures | Target unknown / mailbox full / terminated |
| `SendErrorsRemote` | Remote `Send` failures | Connection problems, target missing on remote |
| `CallErrorsLocal` | Local `Call` failures | Same + timeouts |
| `CallErrorsRemote` | Remote `Call` failures | Same + network latency |

Non-zero values are always worth investigating. Spikes are always a problem.

### Registration counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `RegisteredNames` | instant | Currently registered process names |
| `RegisteredAliases` | instant | Live aliases |
| `RegisteredEvents` | instant | Registered events |

### Event pub/sub counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `EventsPublished` | cumulative | Local producers' publishes |
| `EventsReceived` | cumulative | Events arriving from remote nodes |
| `EventsLocalSent` | cumulative | Delivered to local subscribers |
| `EventsRemoteSent` | cumulative | Forwarded to remote subscriber nodes |

Fanout check: `EventsRemoteSent / EventsPublished` tells you how many remote deliveries each publish generates. High ratio + small cluster = wide fanout.

### Application counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `ApplicationsTotal` | instant | Loaded applications |
| `ApplicationsRunning` | instant | In Running state |

### Log and tracing counters

| Counter | Layout | Meaning |
|---------|--------|---------|
| `LogMessages` | `[6]uint64` | Cumulative per level: `[0]=Trace [1]=Debug [2]=Info [3]=Warning [4]=Error [5]=Panic` |
| `TracingSpans` | `[5]uint64` | `[0]=Send [1]=Request [2]=Response [3]=Spawn [4]=Terminate` |

Watch `LogMessages[4]` (Error) and `LogMessages[5]` (Panic) - growth indicates real problems. Compare across nodes in a cluster; one node with significantly more errors is an outlier.

### Runtime counters

| Counter | Meaning |
|---------|---------|
| `MemoryUsed` | `runtime.MemStats.Alloc` - current live bytes |
| `MemoryAlloc` | `runtime.MemStats.TotalAlloc` - cumulative bytes ever allocated |
| `UserTime` | User CPU time (ns) |
| `SystemTime` | System CPU time (ns) |
| `ServerTime` | Current wall-clock time with timezone |

`MemoryAlloc - MemoryUsed` = freed bytes. High ratio = allocation churn (GC pressure).

`ServerTime` helps correlate events across nodes in different timezones.

## NetworkInfo (from `network_info`)

| Counter | Kind | Meaning |
|---------|------|---------|
| `ConnectionsEstablished` | cumulative | Total connections ever opened |
| `ConnectionsLost` | cumulative | Total connections lost |

**Current active connections** = `ConnectionsEstablished - ConnectionsLost`. Growing `ConnectionsLost` = instability; pair with per-connection `Reconnections` counter (via `network_node_info`) to find the flaky peer.

## AcceptorInfo (from `network_acceptors`)

| Counter | Kind | Meaning |
|---------|------|---------|
| `HandshakeErrors` | cumulative | Failed handshakes on this acceptor |

Non-zero = cookie mismatches, version incompatibility, or scanning. Per-acceptor, so you can isolate which interface is being probed.

## RemoteNodeInfo (from `network_node_info`, `network_nodes`)

Per-connection view. **Always check both sides** (`network_node_info node=A name=B` and `network_node_info node=B name=A`) - divergence is silent data loss.

| Counter | Kind | Meaning |
|---------|------|---------|
| `Uptime` | instant | Remote node uptime (reported during handshake) |
| `ConnectionUptime` | instant | Age of this connection |
| `MessagesIn` | cumulative | Messages received from this peer |
| `MessagesOut` | cumulative | Messages sent to this peer |
| `BytesIn` / `BytesOut` | cumulative | Byte counters |
| `TransitBytesIn` / `TransitBytesOut` | cumulative | Proxy-transit traffic if this connection is a proxy |
| `Reconnections` | cumulative | Pool-item reconnections. Non-zero = at least one TCP link broke and was re-established; growing = ongoing flap. |

Cross-check rules:
- `A.MessagesOut` toward B should match `B.MessagesIn` from A. Significant divergence = data loss.
- `ConnectionUptime << Uptime` = connection was re-established (flap); combine with registrar / network_info trend.

### Fragmentation counters

Messages above the peer's `MaxMessageSize` are split into fragments and reassembled on the far side.

| Counter | Kind | Meaning |
|---------|------|---------|
| `FragmentsSent` | cumulative | Individual fragments sent |
| `FragmentMessagesSent` | cumulative | Messages that were fragmented for sending |
| `FragmentsReceived` | cumulative | Individual fragments received |
| `FragmentMessagesRecv` | cumulative | Fragmented messages successfully reassembled |
| `FragmentTimeouts` | cumulative | Reassemblies that timed out. **Non-zero is a fault** - a fragmented message never completed = dropped payload. |

`FragmentTimeouts > 0` is a direct fault signal: pair it with an `A.MessagesOut` / `B.MessagesIn` divergence to confirm dropped messages.

### Compression counters

Compression applies only to cross-node messages above the process compression threshold (see `framework-internals.md` §Compression).

| Counter | Kind | Meaning |
|---------|------|---------|
| `CompressedSent` | cumulative | Messages compressed on send |
| `CompressedBytesSent` | cumulative | Bytes after compression (wire size) |
| `CompressedOrigBytesSent` | cumulative | Bytes before compression (original size) |
| `DecompressedRecv` | cumulative | Messages decompressed on receive |
| `DecompressedBytesRecv` | cumulative | Bytes before decompression (wire size) |
| `DecompressedOrigRecv` | cumulative | Bytes after decompression (original size) |

Actual send-side compression ratio = `CompressedBytesSent / CompressedOrigBytesSent`. A ratio near 1.0 means compression is buying little - the threshold may be set too low for this traffic.

### Tracing and clock counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `TracedSent` | cumulative | Messages sent carrying a tracing wrapper |
| `TracedReceived` | cumulative | Messages received carrying a tracing wrapper |
| `ClockSkew` | instant (ns) | Estimated clock offset relative to the peer. Positive = remote clock ahead; `0` = not yet measured. |

`ClockSkew` complements `node_info.ServerTime` when correlating timestamped events across nodes - subtract it before comparing remote timestamps to local ones.

### Pool fields

Instantaneous, not counters, but essential for spotting a degraded connection pool:

| Field | Kind | Meaning |
|-------|------|---------|
| `PoolSize` | config | Configured **target** number of TCP connections (set by the acceptor during handshake) |
| `PoolLen` | instant | Current **live** number of TCP connections; reaches `PoolSize` once fully filled |
| `PoolDSN` | instant | Connection strings (`host:port`) for each pooled connection; non-nil means this side dialed out |

`PoolLen < PoolSize` means the pool has not fully filled - it is still filling or has been degraded by lost links. Cross-check with a growing `Reconnections` to confirm churn.

## ProcessInfo (from `process_info`)

### Mailbox queues

`MailboxQueues` has four depth counters and (with `-tags=latency`) four latency counters:

| Queue | Depth field | Latency field (ns) |
|-------|-------------|--------------------|
| Urgent | `MailboxQueues.Urgent` | `MailboxQueues.LatencyUrgent` |
| System | `MailboxQueues.System` | `MailboxQueues.LatencySystem` |
| Main | `MailboxQueues.Main` | `MailboxQueues.LatencyMain` |
| Log | `MailboxQueues.Log` | `MailboxQueues.LatencyLog` |

Latency is the age of the oldest message in that queue. `-1` means `-tags=latency` was not set.

### Message and time counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `MessagesIn` | cumulative | Messages the process received |
| `MessagesOut` | cumulative | Messages the process sent |
| `RunningTime` | cumulative (ns) | Time spent executing callbacks |
| `InitTime` | one-shot (ns) | Duration of `ProcessInit()` |
| `Wakeups` | cumulative | Sleep->Running transitions |
| `Uptime` | instant (sec) | Seconds since spawn |

Derivations used in diagnostics:

```
Utilization   = RunningTime / (Uptime_sec * 1e9)
Drain         = MessagesIn / Wakeups
Liveness      = RunningTime / (Uptime_sec * MailboxLatency_ns)
```

See `process-model.md` for how to read these values.

## EventInfo (from `event_info`, `event_list`)

| Field | Kind | Meaning |
|-------|------|---------|
| `Producer` | instant | PID of the registering process |
| `Subscribers` | instant | Current subscriber count (local + remote) |
| `Notify` | flag | Producer requested `MessageEventStart/Stop` notifications |
| `Open` | flag | Event accepts publishes from any local process (token check disabled) |
| `BufferSize` | config | Configured ring buffer size (from `RegisterEvent`) |
| `CurrentBuffer` | instant | Entries currently in the buffer |
| `CreatedAt` | timestamp | Unix nanosecond of registration |
| `MessagesPublished` | cumulative | Calls to `SendEvent` |
| `MessagesLocalSent` | cumulative | Delivered to local subscribers |
| `MessagesRemoteSent` | cumulative | Forwarded to remote subscribers |
| `LastPublishedAt` | timestamp | Unix nanosecond of the most recent `SendEvent`; `0` if nothing was ever published |

Cross-checks:
- `MessagesPublished > 0` and `Subscribers == 0` and `Notify == false` -> `utilization_state: no_subscribers` - publishing into the void.
- `Subscribers > 0` and `MessagesPublished == 0` and `Notify == false` -> `utilization_state: no_publishing` - subscribers starved.
- `MessagesPublished > 0` (it did publish once) with a **stale** `LastPublishedAt` -> a silent producer: it used to publish but has gone quiet. The two rules above only catch producers that *never* published (`MessagesPublished == 0`); `LastPublishedAt` is what catches one that *stopped*. Compare `LastPublishedAt` against `node_info.ServerTime` (or a peer's `ClockSkew`-adjusted clock) to gauge how long it has been silent.
- See `tools.md` §`event_list` for the full state table.

## Patterns for Investigation

### Silent message loss

Compare both sides of a connection:

```
A says MessagesOut=2363, B says MessagesIn=1
```

B never received what A sent. Inspect `Reconnections` and `FragmentTimeouts` on both sides (a reconnect mid-transfer or a failed fragment reassembly both drop payload). Check `SendErrorsRemote` on A's `node_info`.

### Growing heap

Trend `MemoryUsed` and `heap_alloc` (via `runtime_stats`) over `sample_start interval_ms=5000 duration_sec=300`. GC pressure shows as high `MemoryAlloc` increase without equivalent `MemoryUsed` growth.

### Restart loop

Growing `ProcessesSpawned` without matching `ProcessesTerminated`, combined with short-lived processes visible via `process_list max_uptime=10`.

### Handshake problems

`HandshakeErrors > 0` on one or more acceptors. Typical causes: wrong cookie, Erlang-vs-EDF protocol mismatch, TLS misconfiguration. Combine with log capture (`sample_listen log_levels=["error"]`).

### Event waste

`event_list utilization_state=no_subscribers sort_by=published` reveals events producing messages with nobody listening. Publishing cost is mostly wasted - producers should either gate on subscriber presence (`Notify: true`) or stop publishing.

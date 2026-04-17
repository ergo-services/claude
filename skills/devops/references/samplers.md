# Samplers

Samplers are first-class processes spawned by the MCP application that capture data into a ring buffer over time. Two modes: **active** (periodically call any MCP tool) and **passive** (subscribe to logs and/or events). Use them for trend analysis, catching transient states, and reproducing intermittent symptoms.

## Lifecycle

1. **Start** — `sample_start` (active) or `sample_listen` (passive) returns a `sampler_id`.
2. **Collect** — the sampler runs until `count` samples reached, `duration_sec` elapses, or `sample_stop` is called.
3. **Read** — `sample_read sampler_id=<id>` any time, including during collection.
4. **Linger** — after completion, the sampler stays alive for `linger_sec` (default 30) so data can still be read. `sample_list` shows `completed, lingering Ns` during this period.
5. **Terminate** — once linger expires or `sample_stop` is called, the sampler process exits.

## Active Samplers (`sample_start`)

Periodically call any MCP tool with fixed arguments. Results land in a ring buffer keyed by sequence number.

```
sample_start tool=runtime_stats interval_ms=5000 duration_sec=300 linger_sec=60
sample_start tool=process_list arguments={"sort_by":"mailbox","limit":5} interval_ms=2000 duration_sec=60
sample_start tool=node_info interval_ms=10000 duration_sec=600
```

Parameters (from `tools.md`):

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `tool` | required | Any MCP tool name |
| `arguments` | `{}` | Tool-specific arguments |
| `interval_ms` | 5000, min 100 | Period between calls |
| `count` | 0 | Stop after N samples (0 = unlimited within duration) |
| `duration_sec` | 60, max 3600 | Wall-clock cap |
| `buffer_size` | 256 | Ring buffer entries (oldest overwritten) |
| `max_errors` | 0 | 0 = retry forever |
| `linger_sec` | 30 | Keep alive after completion |

**`max_errors=0` is the default**. Use it when polling for a rare condition (e.g., catching a process stack when it briefly wakes from sleep) — the sampler keeps retrying until success.

## Passive Samplers (`sample_listen`)

Subscribe to logs and/or one event. Unlike active samplers, no periodic work — entries arrive as they are produced.

```
sample_listen log_levels=["warning","error","panic"] duration_sec=120
sample_listen log_levels=["debug"] log_source="process" duration_sec=60
sample_listen event=price_update event_node=feed@host duration_sec=30
sample_listen log_levels=["error"] event=transactions duration_sec=60
```

Parameters:

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `log_levels` | `[info, warning, error, panic]` | Subset of `trace, debug, info, warning, error, panic`. Empty = disable log capture |
| `log_source` | `""` (all) | `process | meta | node | network` |
| `event` | `""` | Event name to subscribe; empty = no event capture |
| `event_node` | local node | Node that owns the event |
| `duration_sec` | 60, max 3600 | |
| `buffer_size` | 256 | |
| `linger_sec` | 30 | |

Both log and event capture can run in the same sampler — combine as needed.

### Passive samplers as test receivers

A passive sampler is an ordinary process. You can `send_message` or `call_process` to its `sampler_id` and the delivery lands in the same ring buffer tagged with `from`, `type`, and `message`. Useful when you need a "message sink" to verify routing:

```
sample_listen duration_sec=60 linger_sec=60
# → sampler_id: mcp_sampler_AB12CD

send_message to=mcp_sampler_AB12CD type_name=PingRequest message={"Seq":1}
sample_read sampler_id=mcp_sampler_AB12CD
```

The captured entry shows the sender PID, Go type name, and payload.

## Reading Data

```
sample_read sampler_id=<id>              # all buffered entries (since=0)
sample_read sampler_id=<id> since=<seq>  # only entries newer than sequence <seq>
```

Each entry has a `sequence` field. For incremental polling, pass the last sequence you saw back as `since` on the next call.

The response includes the sampler's current status (running / completed / lingering / stopped) — useful for loop termination in automation.

## Managing Samplers

```
sample_list                 # all samplers on this node with config + progress
sample_stop sampler_id=<id> # graceful stop; buffered data remains readable during linger
```

`sample_stop` succeeds on a running sampler and on a stopped-but-still-lingering one. Once linger expires and the sampler process exits, `sample_stop` returns `sampler <id> not found`.

## Proxy Samplers (Cluster-Wide)

When you pass `node=X` to `sample_start` or `sample_listen`, the sampler is **spawned on node X**, not on the MCP entry-point. The tool runs in X's process space. Consequences:

- `sample_read`, `sample_list`, `sample_stop` for this sampler **must** also include `node=X` — the sampler process lives there.
- Cross-node sampling is symmetric: the entry-point has no special status.

Example — watch mailbox pressure on a remote node:

```
sample_start node=backend@host tool=process_list arguments={"sort_by":"mailbox","limit":5} interval_ms=2000 duration_sec=60

# Read later — from ANY MCP session, still addressing node=backend@host
sample_read node=backend@host sampler_id=<id>
```

### Why you almost always want proxy samplers

Each sampler call crosses a network boundary. For high-frequency (100ms–2s interval) or data-heavy (`process_list limit=50`) sampling on a remote node, running the sampler **on** the remote node avoids amplifying that network load.

## Ring Buffer Behaviour

- `buffer_size` (default 256) bounds memory. Once full, the oldest entries are overwritten.
- `sample_read since=0` returns whatever is currently in the buffer — not everything ever produced.
- For long captures on high-rate sources, increase `buffer_size` or periodically `sample_read` with advancing `since` to consume entries before they scroll.

## Common Recipes

### Trend heap over 5 minutes

```
sample_start tool=runtime_stats interval_ms=5000 duration_sec=300 linger_sec=60
# wait, then:
sample_read sampler_id=<id>
```

Plot `heap_alloc` / `heap_objects` over the sequence.

### Catch a goroutine at wake (sleeping process)

```
sample_start tool=pprof_goroutines arguments={"pid":"<PID>"} interval_ms=300 count=1 max_errors=0
```

Tight polling + stop on first success + never give up. `count=1` ends the sampler after one successful capture.

### Top-5 mailbox over a minute

```
sample_start tool=process_list arguments={"sort_by":"mailbox","limit":5} interval_ms=1000 duration_sec=60
```

### Log tail during a repro

```
sample_listen log_levels=["warning","error","panic"] duration_sec=300 linger_sec=60
```

Reproduce the issue in that window, then `sample_read`.

### Watch a specific event's published messages

```
sample_listen event=<name> event_node=<producer-node> duration_sec=120
```

See the actual payloads, not just counters.

## Anti-Patterns

- **No `duration_sec`** → the sampler runs up to the 3600s cap and leaks attention. Always set a bound.
- **High-frequency remote sampling without `node=X`** → amplifies network hops per sample.
- **Ignoring `linger_sec`** → sampler exits before you read; default 30s is usually enough, raise it for long operator sessions.
- **Starting many active samplers with overlapping arguments** → duplicated load. Use one sampler with a wider `limit` if possible.
- **Using `sample_read` in a tight loop** for real-time display → wasteful. Use the sequence-based `since` cursor to fetch only new entries.

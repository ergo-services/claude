# MCP Tool Catalog

Complete catalog of the 48 tools exposed by `ergo.services/application/mcp`. Grouped by category. Every tool accepts two universal proxy parameters alongside its natural arguments:

| Parameter | Purpose |
|-----------|---------|
| `node` (string) | Forward the call to node `<name>` via native Ergo networking. The framework connects automatically; do not call `network_connect` just to query. |
| `timeout` (int, seconds) | Per-call timeout for heavy remote operations. Raise for long `pprof_cpu` or large `pprof_goroutines debug=2`. Enforcement happens in the proxy forwarding layer; explicit value is optional. |

These two are never listed in each tool's schema — they are resolved by the MCP proxy layer before dispatch.

## Node (2)

### `node_info`
Returns node info: name, uptime, version, process counts, memory, CPU time, registered names/aliases/events counts, error counters, application counts, `LogMessages` (a `[6]uint64` array indexed `[0]=Trace [1]=Debug [2]=Info [3]=Warning [4]=Error [5]=Panic`).
No parameters.

### `node_env`
Returns the node's environment variables as key/value pairs.
No parameters.

## Process (7)

### `process_list`
Returns `ProcessShortInfo` for each matching process on the node. Supports filtering across all fields and sorting.

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `limit` | int | 100 (0 = all) | Max results |
| `application` | string | | Exact match on application name |
| `behavior` | string | | Substring match on behavior type |
| `state` | string | | `init | sleep | running | wait response | terminated | zombee` (note the space in "wait response") |
| `name` | string | | Substring match on registered name |
| `min_messages_in` | int | 0 | `MessagesIn ≥ value` |
| `min_messages_out` | int | 0 | `MessagesOut ≥ value` |
| `min_mailbox` | int | 0 | `MessagesMailbox ≥ value` |
| `min_mailbox_latency_ms` | number | 0 | Requires `-tags=latency` |
| `min_running_time_ms` | number | 0 | Cumulative callback time |
| `min_init_time_ms` | number | 0 | `InitTime ≥ value` |
| `min_wakeups` | int | 0 | `Wakeups ≥ value` |
| `min_uptime` | int | 0 | Seconds |
| `max_uptime` | int | 0 | Seconds — catch recently spawned |
| `sort_by` | enum | | `mailbox, mailbox_latency, running_time, init_time, wakeups, uptime, messages_in, messages_out, drain` (descending) |

### `process_children`
Direct children or full subtree of a parent process.

| Parameter | Type | Required | Meaning |
|-----------|------|----------|---------|
| `parent` | string | yes | Registered name or PID string |
| `recursive` | bool | no | Default `false` (direct only); `true` returns full subtree |

### `process_info`
Full `ProcessInfo` for one PID: mailbox queues, links, monitors, aliases, events, parent, leader, env.

| Parameter | Required |
|-----------|----------|
| `pid` | yes |

### `process_state`
Returns process state name: `init | sleep | running | wait response | terminated | zombee` (note the space in "wait response").

### `process_lookup`
Resolve name↔PID. Provide one of:

| Parameter | Meaning |
|-----------|---------|
| `name` | Resolves registered name to PID |
| `pid` | Resolves PID to its registered name (or empty) |

### `process_inspect`
Calls the process's `HandleInspect(from, items...)` callback, returning its key/value introspection map.

| Parameter | Type | Required |
|-----------|------|----------|
| `pid` | string | yes |
| `items` | []string | no |

### `meta_inspect`
Same as `process_inspect` but for meta processes, addressed by alias.

| Parameter | Type | Required |
|-----------|------|----------|
| `alias` | string | yes |
| `items` | []string | no |

## Profiling / Debug (4)

**On remote nodes always use `filter` / `exclude`.** Unfiltered dumps overwhelm the proxy transport.

### `pprof_goroutines`
Goroutine profile. Without `pid`: all goroutines (subject to `limit`). With `pid`: stack trace for one actor's goroutine (requires `-tags=pprof` on the target node). A sleeping process has a parked goroutine that may not appear in the dump.

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `pid` | string | | Requires `-tags=pprof`. Empty = all goroutines. |
| `limit` | int | 50 | Results after filtering. Ignored when `pid` is set. |
| `debug` | int | 2 | 1 = summary (count per stack), 2 = full traces |
| `filter` | string | | Include only matching substrings |
| `exclude` | string | | Exclude matching substrings (applied after filter) |

### `pprof_cpu`
CPU profile for `duration` seconds. Returns top functions by CPU. Worker is blocked during collection.

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `duration` | int | 5 (max 30) | Seconds |
| `limit` | int | 20 | Top N |
| `filter` | string | | Include only matching |
| `exclude` | string | | Exclude matching |

### `pprof_heap`
Heap profile: top allocators by `inuse` (and `alloc` for churn).

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `limit` | int | 20 | Top N |
| `filter` | string | | Include only matching |
| `exclude` | string | | Exclude matching |

### `runtime_stats`
Go runtime stats: goroutines, CPU count, `heap_alloc`, `heap_sys`, `heap_inuse`, `heap_objects`, `stack_inuse`, `total_alloc`, `sys`, `num_gc`, `last_gc_pause_ns`, `gc_cpu_percent`.
No parameters.

## Network (8)

### `network_info`
Network config: mode, flags, max message size, acceptors, routes, proxy routes.
No parameters.

### `network_nodes`
Connected remote nodes with per-connection info. Filters:

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `name` | | Substring match on node name |
| `min_uptime` | 0 | Node uptime seconds |
| `min_connection_uptime` | 0 | Connection age seconds |
| `min_messages_in` / `_out` | 0 | Counters |
| `min_bytes_in` / `_out` | 0 | Counters |

### `network_node_info`
Detailed `RemoteNodeInfo` for one peer: version, handshake/proto versions, flags, pool size, `PoolDSN`, messages/bytes in/out, uptimes.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

### `network_acceptors`
List of local acceptors with host, port, flags, TLS.
No parameters.

### `network_connect`
Establish connection via existing routes / registrar. If connected, returns current info.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

### `network_connect_route`
Connect using a custom ad-hoc route (host/port/TLS/cookie).

| Parameter | Type | Required |
|-----------|------|----------|
| `name` | string | yes |
| `host` | string | yes |
| `port` | int | yes |
| `tls` | bool | no |
| `cookie` | string | no |
| `insecure_skip_verify` | bool | no |

### `network_disconnect`
Disconnects from a named peer.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

### `network_ping`
Measures round-trip time through the full path (flusher → TCP → remote MCP → response). Requires the MCP application on the target node.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

## Samplers (5)

### `sample_start` (active)
Periodically call any MCP tool, store results in a ring buffer. Use for trend analysis.

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `tool` | string | yes | Tool name to call periodically |
| `arguments` | object | `{}` | Arguments for the tool on each call |
| `interval_ms` | int | 5000 (min 100) | |
| `count` | int | 0 | Stop after N samples (0 = until duration/stop) |
| `duration_sec` | int | 60 (max 3600) | 0 = until stopped (but still bounded by 3600) |
| `buffer_size` | int | 256 | Ring buffer entries |
| `max_errors` | int | 0 | 0 = retry forever; useful for polling rare events |
| `linger_sec` | int | 30 | Stay alive after completion so data can be read |

Returns `sampler_id` — use with `sample_read`.

### `sample_listen` (passive)
Passive sampler capturing log messages and/or published events.

| Parameter | Type | Meaning |
|-----------|------|---------|
| `log_levels` | []string | `[trace, debug, info, warning, error, panic]` subset. Default `[info, warning, error, panic]`. Empty = disable log capture. |
| `log_source` | enum | `process | meta | node | network`. Empty = all. |
| `event` | string | Event name to subscribe. Empty = no event capture. |
| `event_node` | string | Node owning the event (default: local) |
| `duration_sec` | int | Default 60, max 3600 |
| `buffer_size` | int | Default 256 |
| `linger_sec` | int | Default 30 |

Passive samplers also act as regular processes — you can `send_message`/`call_process` to them and the delivery is captured in the buffer (useful as a test receiver).

### `sample_read`
Read sampler buffer entries.

| Parameter | Type | Required | Meaning |
|-----------|------|----------|---------|
| `sampler_id` | string | yes | |
| `since` | int | no | Return entries with sequence `> since`. Default 0 = all buffered. |

### `sample_stop`
Terminates the sampler. Remaining data can still be read until `linger_sec` expires.

| Parameter | Required |
|-----------|----------|
| `sampler_id` | yes |

### `sample_list`
Lists all active/lingering samplers on the node with their configuration and progress.
No parameters.

**Proxy samplers:** if `sample_start` / `sample_listen` was invoked with `node=X`, the sampler lives on node `X`. `sample_read` / `sample_list` / `sample_stop` MUST also include `node=X` — the sampler is only visible on the owning node.

## Typed Messages and Actions (6)

Action tools (`send_message` with state change, `call_process`, `send_exit`, `process_kill`) **require explicit user permission**. Disabled entirely if the MCP app is started with `ReadOnly: true`.

### `message_types`
Lists EDF-registered type names.

| Parameter | Meaning |
|-----------|---------|
| `filter` | Substring match (case-insensitive) |

### `message_type_info`
Structure of a registered type — fields, types, JSON tags. Short name (`TestOrder`) or full EDF name (`#myapp/pkg/TestOrder`) accepted.

| Parameter | Required |
|-----------|----------|
| `type_name` | yes |

### `send_message`
Send an async message. With `type_name`, constructs a typed struct from JSON via the EDF registry; without, sends the raw JSON value.

| Parameter | Type | Required | Meaning |
|-----------|------|----------|---------|
| `to` | string | yes | Registered name or PID |
| `message` | object | yes | Payload (typed struct fields or raw JSON) |
| `type_name` | string | no | Enables typed construction |
| `important` | bool | no | Use Important Delivery; immediate `ErrProcessUnknown` if target missing |

### `call_process`
Synchronous request with response.

| Parameter | Type | Required | Meaning |
|-----------|------|----------|---------|
| `to` | string | yes | |
| `request` | object | yes | |
| `type_name` | string | no | |
| `timeout` | int | no | Seconds, default 5 |
| `important` | bool | no | `CallImportant` |

### `send_exit`
Delivers an exit signal. Target terminates unless it traps exits.

| Parameter | Type | Required | Meaning |
|-----------|------|----------|---------|
| `pid` | string | yes | |
| `reason` | string | no | `normal | shutdown | kill` or custom string; default `normal` |

### `process_kill`
Forcefully kills a process (`gen.ErrKill`). Target goes to Zombee state immediately.

| Parameter | Required |
|-----------|----------|
| `pid` | yes |

## Events (2)

### `event_list`
Registered events with statistics. Filters:

| Parameter | Meaning |
|-----------|---------|
| `limit` | Default 100 |
| `name` | Substring match |
| `producer` | Match by PID string or registered name (substring) |
| `notify` | Filter by Notify flag |
| `has_buffer` | Filter by buffer presence |
| `min_subscribers` / `max_subscribers` | |
| `min_published` / `max_published` | |
| `min_local_sent` | |
| `min_remote_sent` | |
| `utilization_state` | `active | on_demand | idle | no_subscribers | no_publishing` (see below) |
| `sort_by` | `subscribers | published | local_sent | remote_sent` (descending) |

**Utilization state derivation:**

| State | Condition |
|-------|-----------|
| `active` | `Subscribers > 0 AND MessagesPublished > 0` |
| `on_demand` | `Notify = true` (waiting for demand) |
| `idle` | `Subscribers = 0 AND MessagesPublished = 0 AND Notify = false` |
| `no_subscribers` | `MessagesPublished > 0 AND Subscribers = 0 AND Notify = false` (publishing to void) |
| `no_publishing` | `Subscribers > 0 AND MessagesPublished = 0 AND Notify = false` (subscribers waiting) |

### `event_info`
Full `EventInfo` for one event.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

## Applications (3)

### `app_list`

| Parameter | Meaning |
|-----------|---------|
| `state` | `loaded | running | stopped` |
| `mode` | `permanent | transient | temporary` |
| `name` | Substring match |
| `min_uptime` | Seconds |

### `app_info`
Full `ApplicationInfo` for one app: state, mode, version, description, Depends, Env, Group, Tags, Map.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

### `app_processes`

| Parameter | Type | Default |
|-----------|------|---------|
| `name` | string | required |
| `limit` | int | 100 |

## Cron (3)

### `cron_info`
Scheduler overview: next run time, queued jobs count, all job details.
No parameters.

### `cron_job`
Detailed info for one job: spec, location, last run, last error.

| Parameter | Required |
|-----------|----------|
| `name` | yes |

### `cron_schedule`
Jobs planned to fire within `duration_seconds` from now.

| Parameter | Default |
|-----------|---------|
| `duration_seconds` | 3600 |

## Log Level (3)

### `log_level_get`
Read current level for the node, a process (PID or registered name), or a meta process (alias).

| Parameter | Meaning |
|-----------|---------|
| `target` | `node` (default), PID, alias, or registered name |

### `log_level_set` (action)
Same target resolution; `level` required.

| Parameter | Required | Values |
|-----------|----------|--------|
| `target` | no (default `node`) | |
| `level` | yes | `trace, debug, info, warning, error, panic, disabled` |

### `loggers_list`
Registered loggers with their name and level filters.
No parameters.

## Registrar / Discovery (5)

### `registrar_info`
Registrar service info: server, version, supported features (proxy, encryption, applications).
No parameters.

### `registrar_resolve`

| Parameter | Required |
|-----------|----------|
| `name` | yes |

Returns `[]Route` for reaching the named node.

### `registrar_resolve_proxy`
Returns `[]ProxyRoute` (hops via intermediate nodes).

### `registrar_resolve_app`
Returns `[]ApplicationRoute` — which nodes run the named application, with tags/weight/mode/state.

### `cluster_nodes`
Combined list of self + connected + discovered nodes. Use first in any cluster-wide investigation.
No parameters.

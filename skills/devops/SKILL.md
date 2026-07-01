---
name: devops
description: Diagnose production issues on running Ergo Framework nodes via MCP server. Covers performance bottlenecks, process leaks, memory growth, network problems, event fanout, restart loops, and stuck processes. Provides 48 inspection tools, liveness analysis, active/passive sampling, and typed-message actions. Use when investigating issues on live Ergo nodes.
---

# Ergo DevOps

Diagnose production issues in running Ergo Framework distributed systems via the MCP application (`ergo.services/application/mcp`). This skill is the operational counterpart to the `framework` skill - it assumes the system is running and something is suspect.

## MCP Connection

Add the MCP endpoint to Claude Code:

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp
```

Or with authentication:

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp --header "Authorization: Bearer ${MCP_TOKEN}"
```

## Network Transparency

Every tool accepts a `node` parameter. When you pass `node=X`, the MCP entry-point forwards the request to node X via native Ergo inter-node protocol. The framework establishes the connection automatically — never call `network_connect` just to query.

The MCP entry-point node is not privileged. Always pass `node` explicitly even for it.

## Reference Files

| Topic | File | When to read |
|-------|------|--------------|
| Full tool catalog: 48 MCP tools across 11 categories with parameters, defaults, output shape | `references/tools.md` | Any investigation |
| Process model: states, mailbox queues, metrics (RunningTime, Drain, MailboxLatency), liveness formula | `references/process-model.md` | Interpreting `process_list` / `process_info` / `process_inspect` |
| Counters: `node_info`, `network_info`, `process_info`, `event_info` with per-field meaning and diagnostic use | `references/counters.md` | Reading any *_info output |
| 10 diagnostic playbooks (cluster health, bottleneck, CPU, memory, leak, restart loop, zombies, network, events, goroutines) | `references/playbooks.md` | Any symptom matching a playbook |
| Framework internals: Important-delivery errors, restart intensity sliding window, pool backpressure, connection pool, event fanout, shutdown signals | `references/framework-internals.md` | Explaining counter behavior or error semantics |
| Samplers: active (periodic tool), passive (log/event listen), proxy samplers, typed-message injection | `references/samplers.md` | Trend analysis, reproducing transient issues |
| Build tags: `pprof`, `latency`, `verbose`, `norecover`, `typestats`; what each enables and what breaks without them | `references/build-tags.md` | Interpreting missing metrics or goroutine data |

## Critical Rules

1. **Action tools require explicit user permission.** `send_message`, `send_exit`, `process_kill`, and state-changing `call_process` alter the running system and are refused when the MCP app runs with `ReadOnly: true`. `log_level_set` also changes the node but is registered unconditionally (not ReadOnly-gated), so it still requires explicit user permission.
2. **Always filter `pprof_*` on remote nodes.** Unfiltered goroutine / heap / CPU dumps overwhelm the proxy transport. Default: `filter="ProcessRun" exclude="toolPprof"`.
3. **Always sort and limit listings.** `process_list`, `event_list`, `pprof_*` all support `sort_by` + `limit`. Full dumps are useless signal-wise and slow to transport.
4. **Always `duration_sec` on samplers.** Undefined-duration samplers leak.
5. **Always check both sides of a connection** when diagnosing network. One-sided views miss silent data loss.

## Build Tag Shortcuts

| Tag | Effect | Default |
|-----|--------|---------|
| `-tags=pprof` | Labels actor/meta goroutines with PID/Alias for `pprof_goroutines pid=...` lookup, and starts an HTTP pprof server on localhost:9009 (`PPROF_HOST`/`PPROF_PORT`) | off (pid lookup errors; no pprof HTTP server) |
| `-tags=latency` | Measures `MailboxLatency` per queue | off (latency fields return `-1`) |
| `-tags=verbose` | Enables verbose internal logging via `lib.Verbose()` | off (internal traces silent) |
| `-tags=norecover` | **Disables** panic recovery so panics crash the node (debug only) | off - recovery is enabled by default |
| `-tags=typestats` | Per-type EDF encode/decode counters into `gen.RegisteredTypeStats` | off (no per-type stats; not surfaced by MCP) |

See `references/build-tags.md` for details on what breaks when each tag is missing.

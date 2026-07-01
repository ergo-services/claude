# Build Tags

Five build tags change what telemetry the Ergo runtime surfaces: `pprof`, `latency`, `verbose`, `norecover`, and `typestats`. The MCP application reads several of these conditionally - if a tag is absent, specific tools either return errors or report `-1` sentinel values. Two of them (`pprof`, `typestats`) also change runtime behaviour independent of MCP.

Always know which tags the target node was built with before diagnosing. Missing telemetry is not the same as a clean system.

## `-tags=pprof`

**What it enables.** Two things:

1. Labels each actor's goroutine with `pid` and each meta process's goroutine(s) with `meta` + `role`. Allows per-process goroutine lookup by PID.
2. Starts a standard `net/http/pprof` HTTP server on the node (see below).

**Go code gate.** Two variants of `process.run()` and `meta.start()`, plus one init file:
- Without tag: `node/process_run.go`, `node/meta_start.go` - plain spawn.
- With tag: `node/process_run_pprof.go`, `node/meta_start_pprof.go` - wraps the goroutine in `pprof.Do(labels, ...)`.
- With tag: `ergo/pprof.go` - an `init()` that starts the HTTP pprof server.

**Goroutine labels produced (with tag):**

| Process type | Labels |
|--------------|--------|
| Actor | `{"pid":"<ABC.0.1005>"}` |
| Meta (reader goroutine) | `{"meta":"Alias#...", "role":"reader"}` |
| Meta (handler goroutine) | `{"meta":"Alias#...", "role":"handler"}` |

**MCP tool behaviour with and without.**

| Tool | Without `pprof` | With `pprof` |
|------|-----------------|--------------|
| `pprof_goroutines` (no `pid`) | Full dump (unfiltered) - labels absent | Full dump with labels |
| `pprof_goroutines pid=<PID>` | Error: pid filter requires labels | Returns that process's goroutine(s) |
| `pprof_cpu` / `pprof_heap` | Works | Works |
| `runtime_stats` | Works | Works |

**HTTP pprof server.** Besides the goroutine labels, the tag's `init()` in `ergo/pprof.go` imports `net/http/pprof` and starts an HTTP server. This exposes the standard `/debug/pprof/` endpoints (`goroutine`, `heap`, `profile`, `allocs`, ...) directly on the node, independent of the MCP `pprof_*` tools. Bind address:

| Env var | Default | Purpose |
|---------|---------|---------|
| `PPROF_HOST` | `localhost` | Interface to bind |
| `PPROF_PORT` | `9009` | Port to bind |

So a pprof-built node with defaults serves `http://localhost:9009/debug/pprof/`. Point `go tool pprof` at it directly:

```bash
go tool pprof http://localhost:9009/debug/pprof/heap
go tool pprof http://localhost:9009/debug/pprof/profile   # 30s CPU profile
```

Because the default bind is `localhost`, the server is not reachable off-box unless you set `PPROF_HOST` (e.g. `PPROF_HOST=0.0.0.0`). Do that only on a trusted network - the endpoints are unauthenticated.

**When to enable.** Every non-dev deployment. The cost is low (one `pprof.Do` wrapper per spawn plus one HTTP listener) and it makes per-process introspection and off-band profiling possible.

## `-tags=latency`

**What it enables.** The mailbox MPSC queue tracks the enqueue time of the oldest undelivered message. Exposed as `MailboxLatency` in `ProcessShortInfo` and `LatencyMain/System/Urgent/Log` in `MailboxQueues`.

**Go code gate.** Two implementations of the MPSC queue:
- Without tag: `lib/mpsc.go` - plain enqueue/dequeue.
- With tag: `lib/mpsc_latency.go` - same API, records timestamps.

**Telemetry behaviour.**

| Without `latency` | With `latency` |
|-------------------|----------------|
| `MailboxLatency = -1` everywhere | `MailboxLatency` in nanoseconds |
| `min_mailbox_latency_ms` filter rejected with error | Filter works |
| `sort_by=mailbox_latency` rejected with error | Sort works |
| Liveness formula unusable | Liveness formula usable |

**Cost.** One timestamp per enqueue/dequeue. Non-trivial on very high-throughput mailboxes but useful for diagnostics. Production trade-off: enable in staging and on production nodes where latency analysis is expected; disable on hot-path nodes where every microsecond counts.

**When to enable.** Any environment where the liveness score matters - see `process-model.md`. Without it you cannot distinguish "overloaded but alive" from "callback blocked".

## `-tags=verbose`

**What it enables.** `lib.Verbose()` returns `true`. The framework uses this to gate additional internal logging around message routing, mailbox operations, and network events.

**Go code gate.**
- Without tag: `lib/noverbose.go` - `Verbose() = false`.
- With tag: `lib/verbose.go` - `Verbose() = true`.

Framework internals branch on `if lib.Verbose() { ... }` to emit extra trace-level messages.

**Telemetry behaviour.**

| Without `verbose` | With `verbose` |
|-------------------|----------------|
| Normal log volume | Much higher log volume, framework internals visible |

**When to enable.** Development and specific reproduction scenarios where you need to see framework-level message routing. Do **not** enable by default in production - log volume becomes unmanageable.

## `-tags=typestats`

**What it enables.** Per-type EDF serialization counters. With the tag, every root-level EDF encode/decode records its work against the registered type: how many times the type was encoded/decoded and how many wire bytes it accounted for (including the type-tag prefix). Counters live in `gen.RegisteredTypeStats`, embedded in each `gen.RegisteredTypeInfo`:

| Field | Kind | Meaning |
|-------|------|---------|
| `Enabled` | flag | `true` when built with `-tags=typestats`, else `false` |
| `Encoded` | cumulative | Root-level encodes of this type |
| `Decoded` | cumulative | Root-level decodes of this type |
| `EncodedBytes` | cumulative | Wire bytes produced encoding this type |
| `DecodedBytes` | cumulative | Wire bytes consumed decoding this type |

**Go code gate.** Two implementations of the encode/decode wrappers in `net/edf`:
- Without tag: `net/edf/typestats_off.go` - pass-through, `statsEnabled = false`.
- With tag: `net/edf/typestats_on.go` - wraps root encode/decode with `atomic.AddInt64` into the type's `Stats`; `statsEnabled = true`.

`statsEnabled` is read once at type registration to set `Stats.Enabled`, so the flag tells you whether the node was built with the tag.

**Telemetry behaviour.**

| Without `typestats` | With `typestats` |
|---------------------|------------------|
| `Enabled = false`, all counters stay `0` | `Enabled = true`, counters increment per encode/decode |

**How to read it.** These counters are **not** surfaced by any MCP tool - the MCP application reads `RegisteredTypes()` only for type names when building messages. Inspect them in-process instead, from any actor or from node startup code:

```go
for _, t := range edf.RegisteredTypes() {          // ergo.services/ergo/net/edf
    if t.Stats.Enabled {
        fmt.Printf("%s encoded=%d (%d B) decoded=%d (%d B)\n",
            t.Name, t.Stats.Encoded, t.Stats.EncodedBytes,
            t.Stats.Decoded, t.Stats.DecodedBytes)
    }
}
```

`gen.Node.Network().RegisteredTypes()` returns the same `[]gen.RegisteredTypeInfo` aggregated across protos. Each returned entry is a value snapshot taken at call time.

**When to enable.** When you need to know which message types dominate your serialization cost or wire volume - for example sizing a hot cross-node path or hunting an unexpectedly large message. It adds two atomic adds per root encode/decode, so leave it off on latency-critical hot paths unless you are actively measuring.

## `-tags=norecover`

**What it DISABLES.** Panic recovery. This is a reverse tag: by default, recovery is enabled; with this tag, it is disabled.

**Go code gate.**
- Without tag (default): `lib/norecover.go` - `Recover() = true`, panic recovery is active.
- With tag: `lib/recover.go` - `Recover() = false`, panics are not caught and propagate.

In actor code:
```go
if lib.Recover() {
    defer func() {
        if rcv := recover(); rcv != nil { /* log + terminate process */ }
    }()
}
```

Without `-tags=norecover` (default): actor panics are caught, the process terminates with `TerminateReasonPanic`, supervision kicks in.

With `-tags=norecover`: actor panics propagate up the goroutine and crash the node.

**When to enable.** Debugging a panic where the stack trace is more useful than graceful recovery. **Never** in production - a single panic takes the whole node down.

## Combining Tags

Tags are additive. Common combos:

```bash
go build -tags=pprof ./cmd                       # production-friendly: per-PID goroutines + HTTP pprof
go build -tags=pprof,latency ./cmd               # full diagnostic telemetry
go build -tags=pprof,latency,typestats ./cmd     # add per-type serialization counters
go build -tags=pprof,latency,verbose ./cmd       # noisy staging for deep repro
go build -tags=norecover ./cmd                   # debugging a panic
```

## Detecting Which Tags Are Built

There is no single MCP tool that reports the tag set directly. Infer from behaviour:

| Check | If... | Then |
|-------|-------|------|
| `pprof_goroutines pid=<PID>` returns an error about labels | pprof tag is missing | Rebuild with `-tags=pprof` |
| `http://<host>:9009/debug/pprof/` does not respond | pprof tag is missing (or bound elsewhere via `PPROF_HOST`/`PPROF_PORT`) | Rebuild with `-tags=pprof` |
| `process_info` shows `MailboxLatency: -1` | latency tag is missing | Rebuild with `-tags=latency` |
| Framework internals silent in `sample_listen log_levels=["trace"]` during known activity | verbose tag is missing | Rebuild with `-tags=verbose` (if you need that detail) |
| `RegisteredTypeStats.Enabled == false` (from `edf.RegisteredTypes()`, in-process) | typestats tag is missing | Rebuild with `-tags=typestats` |
| A panic crashed the node instead of terminating the process | norecover tag is set | Normal shutdown policy is not in effect |

`version.go` exposes the framework version but not the build tags of the consuming application. If you control deploys, maintain the tag set as part of the build manifest.

## Cross-References

- Per-PID goroutine lookup and labels: `framework-internals.md` §Meta Processes and §Sleep-State Goroutine Invisibility.
- Liveness score (requires `latency`): `process-model.md`.
- Goroutine investigation patterns: `playbooks.md` §10.

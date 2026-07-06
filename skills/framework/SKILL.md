---
name: framework
description: Build distributed actor-based systems with Ergo Framework in Go. Covers actor lifecycle, supervision, message patterns, meta processes for I/O, cluster configuration, EDF serialization, unit testing with fluent assertions, and extension libraries (health, leader, metrics, mcp, observer, pulse, radar). Use when implementing Ergo applications or working with actor-based distributed systems.
---

# Ergo Framework

Distributed, fault-tolerant actor-based systems in Go with Erlang-style reliability. This skill targets Ergo Framework v3.3+. It is organised as an index plus topic-focused reference files — load only what the current task needs.

When a signature or constant must be exact, verify at the source: `$(go env GOMODCACHE)/ergo.services/ergo@<version>/` (gen/, act/, app/, meta/, net/edf/).

## Reference Files

| Topic | File | When to read |
|-------|------|--------------|
| Actor lifecycle, mailbox, HandleMessage/Call/Event, Link/Monitor/Alias, Events, CoreEvent bus | `references/actors.md` | Implementing any actor |
| Supervisor types, restart strategies, intensity, Significant, KeepOrder, per-child restart, dynamic children | `references/supervision.md` | Designing supervision trees |
| Send/Call, important delivery, priorities, compression, fallbacks, MessageOptions | `references/messages.md` | Wiring messages between actors or nodes |
| Embed app.Application, Load(args), Init/Start/Stop lifecycle, ApplicationSpec, Depends, Tags, Map | `references/application.md` | Building an application |
| Pool, act.Router (message routing), act.WebWorker, AddWorkers/RemoveWorkers, when NOT to use | `references/pool.md` | High message load, routing, web handlers |
| TCPServer, TCPConnection, UDPServer, WebServer, Port, ProcessPool routing, meta limitations | `references/meta.md` | I/O integration |
| NodeOptions, NetworkOptions, NetworkFlags defaults, Security, CertManager/TLS, Events, TargetManager | `references/node.md` | Node configuration, security, TLS |
| gen.Log, LogLevel, structured fields, custom LoggerBehavior | `references/logging.md` | Logging from actors, custom loggers |
| Tracing samplers, business spans (StartTracingSpan), exporters, TracingFlags | `references/tracing.md` | Distributed tracing, custom spans |
| gen.Cron, CronJob, cron actions, MessageCron, fallbacks | `references/cron.md` | Scheduled jobs |
| gen sentinels, TerminateReason*, gen.Errorf, matching with errors.Is | `references/errors.md` | Matching errors and termination reasons |
| Network().RegisterType, schema evolution, ApplicationSpec.Network, visibility matrix | `references/edf.md` | Cross-node message types |
| etcd vs Saturn, Registrar interface, ResolveApplication filters, Routes, proxy, NetworkFlags | `references/cluster.md` | Distributed deployment |
| testing/unit (Spawn, Should*), testing/stage (multi-node), testing/check, testing/mock | `references/testing.md` | Writing actor tests |
| actor/health (K8s probes), actor/leader (Raft election), actor/metrics (Prometheus) | `references/actor-lib.md` | Operational actors |
| application/mcp (diagnostics), observer (web UI), pulse (OTLP), radar (bundle) | `references/applications-lib.md` | Observability and diagnostics |
| meta/websocket, meta/sse | `references/meta-lib.md` | WebSocket / SSE servers |
| registrar/etcd, registrar/saturn, logger/colored, logger/rotate, logger/sentry | `references/integrations.md` | External service integrations |
| proto/erlang23 (EPMD, ETF, DIST) | `references/erlang-protocol.md` | Interop with Erlang/OTP nodes |

## Critical Rules

Non-negotiable. A violation is always a bug.

1. **A callback must never block for an unbounded time.** It runs on the mailbox dispatcher; while it blocks, the actor handles nothing else and every Call to it stalls - so every wait needs a hard deadline. Bounded I/O (an HTTP/DB/socket call with an explicit timeout) is fine. A mutex or channel is a last resort and only if it cannot wait indefinitely: `TryLock`, a `select` with a timeout, a non-blocking send/receive - never a naked `Lock()` or a bare `<-ch`. Even bounded, it stalls the actor for the whole timeout and deadlocks if the releaser waits on this actor, so prefer lock-free structures (`sync.Map`, `atomic`) and message passing. Never: `time.Sleep`, any deadline-less wait, or blocking on a goroutine you spawned. Keep every bound as small as the actor's responsiveness needs - "bounded" is the floor, not a licence for a 30s stall.
2. **EDF types are registered in `init()` - never after node start.** Register nested types before parents. All cross-node struct fields must be Exported.
3. **Meta processes cannot initiate `Call` and cannot create Link/Monitor.** They can receive Call (`HandleCall`), receive links/monitors, and send async messages. A synchronous Call to a stock meta blocks (its default HandleCall sends no response).
4. **Production clusters use a central registrar.** Embedded registrar is dev-only. Pick etcd (≤ 70 nodes) or Saturn (100+).
5. **Actors do not share state.** Messages carry copies; never leak mutable struct pointers across actor boundaries.

## Naming and Style Conventions

- Async messages: `MessageXXX` (e.g., `MessageIncrement`, `MessageShutdown`).
- Sync calls: `XXXRequest` / `XXXResponse` (e.g., `GetStatusRequest` / `GetStatusResponse`).
- A reusable actor or application exposes a package-level helper API - `func Action(process gen.Process, ...) error` wrapping Send/Call (or `gen.Node` for node-level callers such as a web server) - so callers never hand-craft messages or hardcode process names. See `references/application.md`.
- Use `any`, not `interface{}`.
- No emojis in documentation.

## Version

Targets Ergo Framework v3.3+. Verify with:
```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/version.go
```

The framework version is `gen.Version` in `ergo/version.go`.

## Import Paths Quick Reference

| Package | Import |
|---------|--------|
| Core | `ergo.services/ergo` |
| Interfaces | `ergo.services/ergo/gen` |
| Behaviors | `ergo.services/ergo/act` |
| Application | `ergo.services/ergo/app` |
| Meta processes | `ergo.services/ergo/meta` |
| Node | `ergo.services/ergo/node` |
| EDF serialization | `ergo.services/ergo/net/edf` |
| etcd registrar | `ergo.services/registrar/etcd` |
| Saturn registrar | `ergo.services/registrar/saturn` |
| Colored logger | `ergo.services/logger/colored` |
| Rotating logger | `ergo.services/logger/rotate` |
| Sentry logger | `ergo.services/logger/sentry` |
| MCP diagnostics | `ergo.services/application/mcp` |
| Observer dashboard | `ergo.services/application/observer` |
| OTLP tracing | `ergo.services/application/pulse` |
| Health + metrics bundle | `ergo.services/application/radar` |
| K8s probes | `ergo.services/actor/health` |
| Leader election | `ergo.services/actor/leader` |
| Prometheus metrics | `ergo.services/actor/metrics` |
| WebSocket meta | `ergo.services/meta/websocket` |
| SSE meta | `ergo.services/meta/sse` |
| Erlang/OTP protocol | `ergo.services/proto/erlang23` |

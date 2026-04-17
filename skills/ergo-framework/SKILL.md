---
name: ergo-framework
description: Build distributed actor-based systems with Ergo Framework in Go. Covers actor lifecycle, supervision, message patterns, meta processes for I/O, cluster configuration, EDF serialization, unit testing with fluent assertions, and extension libraries (health, leader, metrics, mcp, observer, pulse, radar). Use when implementing Ergo applications or working with actor-based distributed systems.
---

# Ergo Framework

Distributed, fault-tolerant actor-based systems in Go with Erlang-style reliability. This skill targets Ergo Framework v3.3+. It is organised as an index plus topic-focused reference files — load only what the current task needs.

When a signature or constant must be exact, verify at the source: `$(go env GOMODCACHE)/ergo.services/ergo@<version>/` (gen/, act/, app/, meta/, net/edf/).

## Reference Files

| Topic | File | When to read |
|-------|------|--------------|
| Actor lifecycle, mailbox, HandleMessage/Call/Event, Link/Monitor/Alias, Events | `references/actors.md` | Implementing any actor |
| Supervisor types, restart strategies, intensity, Significant, KeepOrder, dynamic children | `references/supervision.md` | Designing supervision trees |
| Send/Call, SendImportant/CallImportant, FR-2PC, priorities, compression, fallbacks | `references/messages.md` | Wiring messages between actors or nodes |
| ApplicationSpec, Mode, Load(node, args), lifecycle, Depends, Tags, Map | `references/application.md` | Building an application |
| PoolOptions, AddWorkers/RemoveWorkers, worker mailbox, when NOT to use | `references/pool.md` | High message load |
| TCPServer, TCPConnection, UDPServer, WebServer, Port, ProcessPool routing, meta limitations | `references/meta.md` | I/O integration |
| NodeOptions, NetworkOptions, Security, Tracing, Cron, CertManager, TargetManager | `references/node.md` | Node configuration, security, observability |
| EDF RegisterTypeOf, Marshaler, size limits, visibility matrix | `references/edf.md` | Cross-node message types |
| etcd vs Saturn, Resolver, Routes, Tags, proxy, NetworkFlags | `references/cluster.md` | Distributed deployment |
| `ergo/testing/unit`: Spawn, Should*, matchers, failure injection, network simulation | `references/testing.md` | Writing actor tests |
| actor/health (K8s probes), actor/leader (Raft election), actor/metrics (Prometheus) | `references/actor-lib.md` | Operational actors |
| application/mcp (diagnostics), observer (web UI), pulse (OTLP), radar (bundle) | `references/applications-lib.md` | Observability and diagnostics |
| meta/websocket, meta/sse | `references/meta-lib.md` | WebSocket / SSE servers |
| registrar/etcd, registrar/saturn, logger/colored, logger/rotate | `references/integrations.md` | External service integrations |
| proto/erlang23 (EPMD, ETF, DIST) | `references/erlang-protocol.md` | Interop with Erlang/OTP nodes |

## Critical Rules

Non-negotiable. A violation is always a bug.

1. **No blocking inside actor callbacks.** No mutex, no channel send/receive, no blocking I/O, no `time.Sleep`, no spawned goroutines. Callbacks run on the mailbox dispatcher and must return quickly.
2. **EDF types are registered in `init()` — never after node start.** Register nested types before parents. All cross-node struct fields must be Exported.
3. **Meta processes cannot `Call`, cannot create Link/Monitor.** They can only receive links/monitors and send async messages.
4. **Production clusters use a central registrar.** Embedded registrar is dev-only. Pick etcd (≤ 70 nodes) or Saturn (100+).
5. **Actors do not share state.** Messages carry copies; never leak mutable struct pointers across actor boundaries.

## Naming and Style Conventions

- Async messages: `MessageXXX` (e.g., `MessageIncrement`, `MessageShutdown`).
- Sync calls: `XXXRequest` / `XXXResponse` (e.g., `GetStatusRequest` / `GetStatusResponse`).
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

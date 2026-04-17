# Ergo Framework Claude Code Integration

Claude Code agents and skills for building and operating distributed actor-based systems with Ergo Framework (v3.3+).

Two complementary pairs:

- **ergo-framework** — designing and implementing actor systems (build-time).
- **ergo-devops** — diagnosing running clusters via the MCP application (runtime).

Each pair is split into an **agent** (behaviour / policy / decision logic) and a **skill** (compact reference data, API surface, playbooks). The agent is slim and decides *what to do*; the skill is loaded progressively and answers *what the code actually says*.

## Contents

```
claude/
├── agents/
│   ├── ergo-framework-architect.md
│   └── ergo-devops.md
└── skills/
    ├── ergo-framework/
    │   ├── SKILL.md                       # navigation + critical rules
    │   └── references/
    │       ├── actors.md                  # lifecycle, mailbox, Link/Monitor, Alias, Events
    │       ├── supervision.md             # types, strategies, intensity, Significant, dynamic
    │       ├── messages.md                # Send/Call, Important, FR-2PC, priority, compression
    │       ├── application.md             # ApplicationSpec, Mode, Load(node, args), lifecycle
    │       ├── pool.md                    # PoolOptions, capacity, scale
    │       ├── meta.md                    # TCP/UDP/Web/Port, ProcessPool routing
    │       ├── node.md                    # NodeOptions, NetworkFlags, Security, EnableSpawn
    │       ├── edf.md                     # RegisterTypeOf, visibility matrix, limits
    │       ├── cluster.md                 # etcd/Saturn, Resolver, Tags, proxy
    │       ├── testing.md                 # unit harness: Spawn, Should*, matchers, injection
    │       ├── actor-lib.md               # actor/health, leader, metrics
    │       ├── applications-lib.md        # application/mcp, observer, pulse, radar
    │       ├── meta-lib.md                # meta/websocket, meta/sse
    │       ├── integrations.md            # registrar/etcd, saturn, logger/colored, rotate
    │       └── erlang-protocol.md         # proto/erlang23 (EPMD, ETF, DIST)
    └── ergo-devops/
        ├── SKILL.md                       # navigation + critical rules
        └── references/
            ├── tools.md                   # full MCP tool catalog (48 tools, 11 categories)
            ├── process-model.md           # states, mailbox, liveness formula
            ├── counters.md                # every field of *Info structs with meanings
            ├── framework-internals.md     # Important errors, restart intensity, pool, fanout
            ├── playbooks.md               # 10 diagnostic playbooks with exact commands
            ├── samplers.md                # active/passive, proxy, recipes
            └── build-tags.md              # pprof, latency, verbose, norecover
```

## How the Split Works

**Agents** (~6 KB each) carry only behavioural rules: trigger cues, diagnostic approach, decision tables, anti-patterns, permission rules. They contain no API signatures.

**Skills** use progressive disclosure — the top-level `SKILL.md` is a small (~3 KB) index with a navigation table plus the five non-negotiable rules. Concrete API details, tool schemas, formulas, and playbooks live in `references/*.md`, each ~150–500 lines and loaded on demand.

A typical task loads:
1. `SKILL.md` (auto) — ~3 KB.
2. One or two relevant `references/*.md` — ~5–10 KB.

Total working-set ≈ 8–13 KB vs ~17 KB monolithic; detail per topic is higher rather than lower.

## Pair Comparison

| | ergo-framework | ergo-devops |
|--|---|---|
| **Purpose** | Design and implement actor systems | Diagnose running clusters |
| **Invoked when** | Designing an actor architecture, writing a test, choosing a supervisor strategy | Investigating a production issue, reading counters, running `pprof` |
| **Style** | Design doc format + concrete code patterns | Hypothesis-driven investigation + MCP commands |
| **Requires** | Ergo Framework v3.3+ source or module cache | MCP application running on target nodes |

## Installation

### Recommended: Install as a Claude Code plugin

One-shot install from the official repo, updates handled by Claude Code:

```bash
/plugin marketplace add ergo-services/claude
/plugin install dev@ergo-services
```

After install, agents and skills are namespaced under the plugin — invoke the skill as `/ergo-framework` / `/ergo-devops`, agents by their trigger phrases.

Update later with `/plugin update`, remove with `/plugin uninstall dev@ergo-services`.

### Manual (development / fork / offline)

If you are editing the agents and skills locally, or running without network access, link or copy the files directly.

#### Symlink into `~/.claude`

```bash
cd path/to/ergo.services/claude

mkdir -p ~/.claude/agents ~/.claude/skills

# Agents
ln -sf $(pwd)/agents/ergo-framework-architect.md ~/.claude/agents/
ln -sf $(pwd)/agents/ergo-devops.md              ~/.claude/agents/

# Skills (directories — references/ is picked up transitively)
ln -sf $(pwd)/skills/ergo-framework ~/.claude/skills/
ln -sf $(pwd)/skills/ergo-devops    ~/.claude/skills/
```

#### Copy

```bash
cp -r agents/* ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

## Usage

### ergo-framework-architect (Agent)

Designs Ergo Framework applications with DDD bounded contexts, supervision trees, and cluster topology. Outputs an implementable design document.

**Trigger phrases**
- "design ergo application"
- "ergo architecture"
- "create ergo design document"
- "actor system design"

**Output** — a design document with bounded context, cluster topology, supervision tree, data structures, message flow, load analysis, and implementation phases.

### ergo-devops (Agent)

Connects to running Ergo nodes via the MCP application and runs hypothesis-driven investigations. Never mutates state without explicit user permission.

**Trigger phrases**
- "why is it slow"
- "find process leak"
- "check cluster health"
- "debug this node"
- "monitor for anomalies"

**Capabilities**
- Performance bottleneck investigation (mailbox depth, latency, drain ratio, liveness score).
- Process leak and zombie detection.
- Restart loop analysis.
- Memory growth investigation (heap profiling, GC pressure).
- Network connectivity and traffic analysis (both sides of every connection).
- Event / pub-sub diagnostics (fanout, publishing-to-void, starved subscribers).
- Goroutine debugging (deadlocks, leaks, per-process stack traces).
- Active and passive samplers for trend analysis and transient captures.
- Cluster-wide queries via the MCP proxy layer.

**Requires** — `ergo.services/application/mcp` running on each node under inspection.

### ergo-framework (Skill)

Reference for implementing Ergo Framework applications. Load via `/ergo-framework` or automatically when the topic matches. Follow the navigation table in `SKILL.md` to pull in only the needed `references/*.md`.

### ergo-devops (Skill)

Reference for diagnosing live Ergo nodes. Load via `/ergo-devops`. Navigation table lists 7 topic files — tool catalog, process model, counters, framework internals, playbooks, samplers, build tags.

## MCP Application Setup

The `ergo-devops` pair requires the MCP application on the target node:

```go
import "ergo.services/application/mcp"

node, _ := ergo.StartNode("mynode@host", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{
        mcp.CreateApp(mcp.Options{
            Port:     9922,                    // default
            Token:    os.Getenv("MCP_TOKEN"),  // Bearer auth; empty = no auth
            ReadOnly: true,                    // disables action tools (production)
        }),
    },
})
```

Connect from Claude Code:

```bash
claude mcp add --transport http ergo http://localhost:9922/mcp
# or with auth:
claude mcp add --transport http ergo http://localhost:9922/mcp \
    --header "Authorization: Bearer ${MCP_TOKEN}"
```

Every MCP tool accepts a `node` proxy parameter — pass `node=<name>` and the request is forwarded through native Ergo networking. The entry-point node is not privileged; always name the target explicitly.

## Build Tags for Better Diagnostics

| Tag | Effect |
|-----|--------|
| `-tags=pprof` | Per-process goroutine labels + pprof HTTP server at `localhost:9009` |
| `-tags=latency` | Enables `MailboxLatency` measurement (required for liveness score) |
| `-tags=verbose` | Verbose framework-internal logging |
| `-tags=norecover` | **Disables** panic recovery (debug only) |

See `skills/ergo-devops/references/build-tags.md` for detail.

## Reference

When a signature or constant must be exact, verify against the framework source:

```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/version.go
ls $(go env GOMODCACHE)/ergo.services/ergo@*/docs/
```

## Requirements

- Claude Code CLI
- Ergo Framework v3.3+
- For `ergo-devops`: `ergo.services/application/mcp` running on every node you want to inspect

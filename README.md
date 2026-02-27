# Ergo Framework Claude Code Integration

Claude Code agents and skills for building and operating distributed actor-based systems with Ergo Framework.

## Contents

```
claude/
├── agents/
│   ├── ergo-framework-architect.md   # Architecture design agent
│   └── ergo-devops.md                # DevOps diagnostics agent (MCP)
└── skills/
    ├── ergo-framework/
    │   └── SKILL.md                   # Framework reference (building)
    └── ergo-devops/
        └── SKILL.md                   # DevOps reference (operating)
```

## Agents vs Skills

**Agents** are interactive -- they run diagnostic sequences, make decisions, and drive conversations. Invoked via Claude Code agent selector or triggered by matching phrases.

**Skills** are reference material -- they provide context, playbooks, and quick lookups. Invoked via `/skill-name` or loaded automatically when relevant.

| | ergo-devops Agent | ergo-devops Skill |
|--|---|---|
| **Purpose** | Run a full investigation | Quick reference lookup |
| **Style** | Interactive, multi-step | Compact tables, recipes |
| **When** | "find why it's slow" | "/ergo-devops" for a playbook |
| **Has** | Behavioral rules, response approach | Pattern matching tables |

## Installation

### Option 1: Symlink to ~/.claude

```bash
mkdir -p ~/.claude/agents
mkdir -p ~/.claude/skills

# Agents
ln -sf $(pwd)/agents/ergo-framework-architect.md ~/.claude/agents/
ln -sf $(pwd)/agents/ergo-devops.md ~/.claude/agents/

# Skills
ln -sf $(pwd)/skills/ergo-framework ~/.claude/skills/
ln -sf $(pwd)/skills/ergo-devops ~/.claude/skills/
```

### Option 2: Copy files

```bash
cp -r agents/* ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

## Usage

### ergo-framework-architect (Agent)

Designs Ergo Framework applications with DDD bounded contexts, supervision trees, and cluster topology.

**Trigger phrases:**
- "design ergo application"
- "ergo architecture"
- "create ergo design document"
- "actor system design"

**Output:** Detailed design documents with data structures, message flows, supervision strategies, and implementation phases.

### ergo-devops (Agent)

Diagnostics agent for running Ergo nodes via MCP server. Connects to a node's MCP endpoint and runs diagnostic sequences to investigate production issues.

**Trigger phrases:**
- "why is it slow"
- "find process leak"
- "check cluster health"
- "debug this node"
- "monitor for anomalies"

**Capabilities:**
- Performance bottleneck investigation (mailbox depth, latency, drain ratio)
- Process leak and zombie detection
- Restart loop analysis
- Memory growth investigation (heap profiling, GC pressure)
- Network connectivity and traffic analysis
- Event/PubSub diagnostics (fanout overload, publishing to void)
- Goroutine debugging (deadlocks, leaks, per-process stack traces)
- Real-time monitoring with active and passive samplers
- Cluster-wide health checks via proxy

**Requires:** MCP application running on target node (`ergo.services/application/mcp`).

### ergo-framework (Skill)

Reference for implementing Ergo Framework applications: actors, supervision, messages, meta processes, EDF, clusters.

```
/ergo-framework
```

### ergo-devops (Skill)

Quick reference for diagnosing live Ergo nodes: playbooks, pattern matching tables, sampler recipes, build tag awareness.

```
/ergo-devops
```

## Reference

When uncertain about APIs, consult the Ergo Framework source and docs:

```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/docs/
```

## Requirements

- Claude Code CLI
- Ergo Framework v3.x
- For devops: MCP application (`ergo.services/application/mcp`) running on target node

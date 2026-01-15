# Ergo Framework Claude Code Integration

Claude Code agents and skills for building distributed actor-based systems with Ergo Framework.

## Contents

```
claude/
├── agents/
│   └── ergo-framework-architect.md   # Architecture design agent
└── skills/
    └── ergo-framework/
        └── SKILL.md                   # Reference documentation
```

## Installation

### Option 1: Symlink to ~/.claude

```bash
# Create directories if needed
mkdir -p ~/.claude/agents
mkdir -p ~/.claude/skills

# Symlink agent
ln -sf $(pwd)/agents/ergo-framework-architect.md ~/.claude/agents/

# Symlink skill
ln -sf $(pwd)/skills/ergo-framework ~/.claude/skills/
```

### Option 2: Copy files

```bash
cp -r agents/* ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

## Usage

### ergo-framework-architect Agent

Designs Ergo Framework applications with DDD bounded contexts, supervision trees, and cluster topology.

**Trigger phrases:**
- "design ergo application"
- "ergo architecture"
- "create ergo design document"
- "actor system design"

**Output:** Detailed design documents with data structures, message flows, supervision strategies, and implementation phases.

### ergo-framework Skill

Reference documentation for implementing Ergo Framework applications.

**Covers:**
- Actor lifecycle and message handling
- Supervision strategies
- Message patterns (Send, Call, Important Delivery)
- Application structure
- Meta processes for I/O
- EDF serialization
- Cluster configuration

## Reference

When uncertain about APIs, consult the Ergo Framework source and docs:

```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/docs/
```

## Requirements

- Claude Code CLI
- Ergo Framework v3.x

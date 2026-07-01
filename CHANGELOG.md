# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this plugin
follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-07-01

Skills actualized against Ergo Framework v3.3.x source. Every reference was
audited against the code: stale claims corrected, missing capabilities added,
thin sections deepened.

### Added

- **`framework` skill** - four new references: `tracing` (samplers, business
  spans, exporters, flags), `cron` (jobs, actions, `MessageCron`), `logging`
  (`gen.Log`, levels, custom `LoggerBehavior`), and `errors` (sentinels,
  `TerminateReason*`, `errors.Is` matching). The skill now carries 19 topic
  references.
- Helper-API pattern in `application`: package-level functions that address the
  local instance by name and keep message types private.

### Changed

- Rewrote `testing` to the real API across the four layers (`unit`, `stage`,
  `check`, `mock`).
- Corrected `application` (embed `app.Application`, `Load(args)` lifecycle),
  `node` (`NetworkFlags` defaults, error-returning introspection), `cluster`
  (`etcd.Create`, `Registrar` interface, `ResolveApplication` filters), `edf`
  (`Network().RegisterType`, schema evolution), `actor-lib` (metrics),
  `supervision` (per-child restart), `pool` (`act.Router`, `act.WebWorker`),
  `meta` (Call receiver), and the devops references.
- Added `logger/sentry` to `integrations`, the `typestats` build tag to
  `build-tags`, and refreshed both `SKILL.md` dispatchers and the `README`.

## [1.0.0] - 2026-04-17

Initial release of the Ergo Services Claude Code plugin — agents and skills
for building and operating distributed actor-based systems with Ergo
Framework (v3.3+).

### Added

- **`framework` skill** — reference for implementing Ergo applications.
  Progressive-disclosure index (`SKILL.md`) plus 15 topic references:
  `actors`, `supervision`, `messages`, `application`, `pool`, `meta`,
  `node`, `edf`, `cluster`, `testing`, `actor-lib`, `applications-lib`,
  `meta-lib`, `integrations`, `erlang-protocol`. Every API signature
  verified line-by-line against Ergo Framework v3.3 source.

- **`devops` skill** — reference for diagnosing live Ergo nodes via the
  MCP application. Progressive-disclosure index plus 7 topic references:
  `tools` (full 48-tool MCP catalog), `process-model`, `counters`,
  `framework-internals`, `playbooks` (10 diagnostic playbooks),
  `samplers`, `build-tags`.

- **`framework-architect` agent** — DDD-first architect for Ergo systems.
  Produces implementable design documents with bounded context, cluster
  topology, supervision trees, message flow, load analysis, and
  implementation phases.

- **`devops` agent** — SRE for running Ergo clusters. Hypothesis-driven
  investigations (observe -> hypothesize -> test -> confirm). Never
  mutates state without explicit permission.

- **Claude Code plugin manifest** — `.claude-plugin/plugin.json` and
  `.claude-plugin/marketplace.json`. Installable via
  `/plugin marketplace add ergo-services/claude` and
  `/plugin install ergo@ergo-services`.

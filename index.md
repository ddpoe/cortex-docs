# Cortex

**Project knowledge indexer for AI agents.**

Cortex scans a Python codebase and its documentation, extracts structured information about every process, entity, constraint, and document, and emits a typed graph-structured JSON index.

That index is the ground truth for agent context injection, database ingestion into [pev-agent-system](https://github.com/dap182/pev-agent-system), and visual rendering.

## Quick Links

- [MVP](mvp.md) — what's in v0.1, what's not, done criteria
- [Architecture](architecture.md) — system overview and design decisions
- [Ontology](ontology.md) — node types and edge types
- [Data Model](data_model.md) — `CortexNode` / `CortexEdge` / `CortexIndex` JSON spec
- [Integration](integration.md) — dFlow DB → cortex, cortex → pev-agent-system
- [Diagrams](diagrams.md) — system and graph diagrams
- [User Stories](user_stories.md) — what the system enables for agents and developers
- [Build Plan](build_plan.md) — three phases with acceptance criteria

## The Core Idea

Agents waste context reading code they don't need. Cortex indexes a project at **four resolution levels**:

| Level | Content | Tokens |
|---|---|---|
| 0 | Name only | ~5 |
| 1 | Signature + one-liner | ~30–60 |
| 2 | Full docstring | ~100–400 |
| 3 | Source location pointer | — |

An agent starts at Level 0, drills to Level 1 to find what's relevant, only reads Level 2 or 3 when it truly needs to. The graph structure (typed edges between nodes) lets it navigate without loading files.

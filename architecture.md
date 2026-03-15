# Cortex — Architecture

## What Cortex Is

Cortex is a **project knowledge indexer**. It scans a Python codebase and its documentation, extracts structured information about every process (function, module, package), entity (data artifact, config), constraint (test, decision), and document (markdown, docstring), and emits a typed, graph-structured JSON index.

That index is the ground truth. Everything else — agent context injection, database ingestion, visual rendering — is a consumer of that index.

Cortex is **not**:
- A workflow runner (that's pm_mvp / snakemake / nextflow)
- A workflow annotator (that's dFlow)
- A task/project manager (cortex doesn't track tasks, decisions, or blockers)
- A documentation renderer (that's MkDocs / Sphinx)

## The Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Project                             │
│   (e.g. pm_image_analysis)                                      │
│                                                                 │
│   Python modules          Markdown docs        Test modules     │
│   + dFlow decorators      (docs/ dir)          + @test_workflow │
└───────────────────────────────────┬────────────────────────────┘
                                    │ sources
                     ┌──────────────┴──────────────┐
                     │                             │
              dFlow (optional)            pure AST fallback
              dflow scan + assemble       (no dFlow needed)
              .dflow/workflow.db
                     │
                     └──────────────┬──────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                          Cortex                               │
│                                                               │
│   Scanners:                                                   │
│     module_scanner.py  — pure AST + dFlow DB reader          │
│     doc_scanner.py     — markdown → section tree             │
│                                                               │
│   Index:                                                      │
│     builder.py         — orchestrates scanners               │
│     writer.py          — writes .cortex/index.json           │
│                                                               │
│   Renderers / CLI:                                            │
│     agent.py           — level-0/1/2 text for agent context  │
│     cortex build / render / validate / check                 │
└───────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                        .cortex/index.json
                (the product — any tool can consume this)
```

## Ownership Boundaries

| Concern | Owner |
|---|---|
| Annotating workflows with `@task`/`@workflow`/`Step()` | dFlow |
| Storing annotation data (`.dflow/workflow.db`) | dFlow |
| Scanning source files and doc files into a graph index | Cortex |
| The ontology (node types, edge types, validation rules) | Cortex |
| The `CortexNode`/`CortexEdge`/`CortexIndex` data model | Cortex |
| Persisting / querying the index | Any tool, agent, or service |
| Visual graph rendering | workflow_canvas (future) |

## Key Design Decisions

**D1 — Cortex reads dFlow's DB, does not import dFlow.**
The integration is via SQL against `.dflow/workflow.db`. This means cortex has zero hard dependency on dFlow. If the DB doesn't exist, cortex falls back to pure AST scanning. More structure when dFlow annotations exist, graceful degradation without them.

**D2 — `.cortex/cortex.db` (SQLite) is the working store.**
All nodes, edges, hashes, and metadata live in a SQLite database under `.cortex/`. If it's lost or stale, re-run `cortex build`. SQLite enables indexed queries, incremental rescans (hash-check per file, upsert changed nodes only), graph traversal via JOINs, and full-text search over Level-1/2 content via the FTS5 extension — none of which are practical with a flat JSON file. `cortex export` produces a portable `.cortex/index.json` snapshot for external consumers (CI, pev-agent-system push, human review). The JSON export is derived from the DB and is never the authoritative store.

**D3 — Notebooks are documents, not code.**
Notebooks should contain only `@workflow` orchestration calls with `Step()`/`AutoStep()` markers — no heavy logic. The module scanner handles `.py` files. Notebooks are registered as `document` nodes that `documents` the processes they demonstrate. No notebook code scanner.

**D4 — Four resolution levels, level-3 is a pointer.**
Level-0 (name only), Level-1 (name + one-liner + signature), Level-2 (full docstring), Level-3 (file + line range). The agent fetches level-3 by reading the source file at the given location — cortex does not store code text in the index. This caps index size.

**D5 — Ontology is closed and validated at build time.**
Node types and edge types are a fixed enum. Invalid node types or edge type combinations fail the build, not silently produce junk. The ontology is defined in `cortex_ontology.yaml` and is the authoritative reference.

**D6 — Tags are open, project-scoped vocabulary.**
Tags are not part of the ontology. They are defined in `cortex_config.yaml` per project as a controlled vocabulary and are used for search/filtering, not structural reasoning. The scanner auto-suggests tags from docstring text; the human curates the vocabulary.

**D7 — Two-tier edge model: structural edges are computed, semantic edges are asserted.**
Structural edges (`composes`, `delegates_to`, `depends_on`, `consumes`, `produces`) are derived from AST and dFlow metadata on every `cortex scan`. They are always current and carry no confidence state. Semantic edges (`constrains`, `validates`, `documents`, `supersedes`) encode design intent that cannot be inferred mechanically — they are asserted by humans and stored with `asserted_at_hash` (the `source_hash` of the target node at assertion time). `cortex check` detects when target source has changed since assertion and marks the edge `UNVERIFIED`. This distinction is critical: structural edges are automatically maintained, semantic edges require human review when source changes. See [ontology.md](ontology.md#structural-vs-semantic-edges) for the full confidence state model.

**D8 — `cortex check` is a pre-flight tool, not a build step.**
`cortex check` rescans source files (AST only, no LLM), recomputes hashes, diffs against the stored index, and outputs a per-node / per-edge confidence report without modifying the index. It is designed to be called by agents before trusting the index, by CI to enforce freshness at PR boundaries, and interactively by developers to see what needs review. It does not fail silently — it returns a structured report with `CLEAN`, `DOC_STALE`, `STRUCTURAL_DRIFT`, `UNVERIFIED`, and `BROKEN` signals that consumers can act on programmatically.

**D9 — RAG: embed `level_1`, expanded nodes always return `level_2`.**
The vector store embeds `level_1` text for every node (code and document nodes share one embedding space). Primary query hits are returned at the caller-specified level. Graph-expanded neighbours (one hop of semantic edges) are always returned at `level_2` regardless of the query level — expanded context is supporting material and level_2 is the right depth for that role. Full source (level_3) is never returned automatically; the `level_3_location` pointer is included so callers can fetch it explicitly. One-hop expansion is the default; `--expand 2` is available but not recommended for routine use.

**D10 — Three distinct query modes: semantic (RAG), structural (graph traversal), and filtered list.**
These serve different intents and are not interchangeable. `cortex query` (RAG) is for discovery when you don't know what node you're looking for — it uses vector similarity. `cortex graph` is for deterministic traversal from a known node — it follows typed edges in a specified direction and depth. `cortex list` is for filtered enumeration — it returns all nodes or edges matching given criteria (type, tag, status, confidence). A typical agent session composes all three: RAG to find the relevant cluster, graph traversal to explore from the best hit, list to audit staleness or coverage. Each command is independently useful and the outputs are composable. See [integration.md](integration.md) for full usage.

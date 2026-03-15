# Build Plan

Three phases. Each is independently deployable and useful before the next begins.
Start with Phase 1 to validate the concept on real data before building any infrastructure.

---

## Phase 1 — Doc Scanner + Core Index (No DB, No Service)

**Goal:** Prove the resolution-level concept on `pm_image_analysis`'s existing markdown docs.
Run against real files today. Validate the JSON structure. No pev-agent-system changes needed.

**Acceptance criteria:**
- `cortex build R:/Dante/pdac/image_analysis/pm_image_analysis` produces `.cortex/cortex.db`
- All `docs/` markdown files appear as `document` nodes with section-level resolution
- `cortex render --level 0` prints just titles
- `cortex render --level 1` prints titles + first-paragraph summaries
- `cortex render --level 2` prints full section text
- `cortex export` writes `.cortex/index.json` from the DB for portability
- Zero crashes on the actual `pm_image_analysis` docs directory

**Files to create:**

```
cortex/
  __init__.py
  ontology.py               # NodeType, EdgeType enums — from cortex_ontology.yaml
  models.py                 # CortexNode, CortexEdge, CortexIndex dataclasses
  scanners/
    __init__.py
    doc_scanner.py          # markdown → document nodes + composes edges for sections
  index/
    __init__.py
    builder.py              # orchestrates scanners, validates ontology, upserts into DB
    db.py                   # SQLite schema, init, upsert, query helpers (stdlib sqlite3)
  renderers/
    __init__.py
    agent.py                # level-0/1/2 text output (queries DB)
  cli.py                    # cortex build / render / validate / export
pyproject.toml
```

**Key implementation notes:**

- `doc_scanner.py` uses Python's `re` / `markdown-it-py` to parse headings. No heavy deps.
- Level-0 = document title (H1). Level-1 = first paragraph of each section. Level-2 = full section text. Level-3 = file#heading-anchor.
- `ontology.py` loads `cortex_ontology.yaml` at import time and exposes `NodeType` and `EdgeType` enums + `validate_edge()`.
- `builder.py` calls each scanner, collects nodes + edges, runs ontology validation, upserts into SQLite via `db.py`.
- `db.py` creates `.cortex/cortex.db` with `nodes`, `edges`, `tags`, and `node_fts` (FTS5 over level_1 + level_2). All writes are upserts — re-runs are safe.
- `cortex_config.yaml` in the project root defines `tag_vocabulary`, `excluded_paths`, `project_id`.

**Dependencies:** Python stdlib (includes `sqlite3`) + `pyyaml` + `markdown-it-py`. Nothing else for Phase 1.

---

## Phase 2 — Module Scanner (Python AST + dFlow DB)

**Goal:** Add code structure to the index. Every Python module and its functions become typed process nodes. dFlow-annotated projects get richer Level-1 content automatically.

**Acceptance criteria:**
- All `.py` modules in `pm_image_analysis` appear as `composite_process` nodes
- All `@task` functions appear as `atomic_process` nodes with `level_1` from `purpose=`
- All `@workflow` functions appear as `composite_process` nodes
- All `@test_workflow`/`@test_suite` appear as `constraint:test` nodes
- All `coverage_links` rows produce `validates` edges
- `composes` edges connect modules to their functions
- `delegates_to` edges connect workflows to their called tasks (from AutoStep targets)
- Unannotated functions use first docstring sentence as `level_1`, full docstring as `level_2`

**Files to add:**

```
cortex/
  scanners/
    module_scanner.py       # pure Python AST walker
    dflow_reader.py         # SQL reader for .dflow/workflow.db — no dFlow import
```

**Key implementation notes:**

- `dflow_reader.py` checks if `.dflow/workflow.db` exists. If not, returns empty lists. No error.
- `module_scanner.py` does a two-pass scan: first pure AST for all functions, then merge with dFlow data (dFlow data wins on `level_1`/`level_2` when both exist).
- Entity nodes from `inputs=`/`outputs=` are best-effort — the string is parsed for names, created as `entity` nodes, hooked to `consumes`/`produces` edges. Marked `source: dflow_inferred`.
- Excluded paths from `cortex_config.yaml` are respected (skip `__pycache__`, `.venv`, `_archive`, etc).

**Dependencies:** Python stdlib `ast` only (no new packages).

---

## Phase 3 — pev-agent-system Ingestion + Query API

**Goal:** Push the `CortexIndex` into pev-agent-system's DB and extend the API with level-aware queries.

**Acceptance criteria:**
- `cortex push --api http://localhost:8100 <project_root>` succeeds and returns ingest stats
- All `CortexNode`s appear as `Artifact` rows in `knowledge_hub.db`
- All `CortexEdge`s appear as `ArtifactLink` rows
- `GET /artifacts?project_id=pm_image_analysis&level=0` returns names only
- `GET /artifacts?project_id=pm_image_analysis&level=1` returns names + level_1 + tags + linked_ids
- `GET /artifacts/{id}?level=2` returns full docstring
- `GET /artifacts/{id}?level=3` returns `{file, line_start, line_end}`
- Re-running push is idempotent (upsert, no duplicates)
- Manually-created edges (source=manual) are never overwritten by ingest

**Files to add to pev-agent-system:**

```
src/app/
  services/
    cortex_ingest.py        # CortexIndex → bulk upsert Artifact + ArtifactLink
  routers/
    ingest.py               # POST /ingest/cortex
```

**Files to modify in pev-agent-system:**

```
src/app/models/artifact_links.py   # Add COMPOSES, DELEGATES_TO, CONSUMES, PRODUCES to LinkType
src/app/main.py                    # Register ingest router
src/app/services/artifacts.py      # Add level param to read queries
```

**Migration:** Add `COMPOSES`, `DELEGATES_TO`, `CONSUMES`, `PRODUCES` to the `LinkType` enum and run an Alembic migration for the constraint check on `artifact_links.link_type`.

**Files to add to cortex:**

```
cortex/
  push.py                  # queries .cortex/cortex.db, POSTs to /ingest/cortex
  cli.py                   # add `cortex push` command
```

---

## Phase 4 (Future) — Search + Agent Tooling

Beyond the three phases above, these are the natural next steps once the graph is live and queryable:

- **BM25 / sqlite-vec search over Level-1 summaries** — `GET /artifacts/search?q=calibration&project=pm_image_analysis` returns ranked Level-1 results without embedding infrastructure
- **Automatic task generation** — pev-agent-system query for untested `atomic_process` nodes → bulk `POST /tasks`
- **Cortex diff** — compare two `index.json` snapshots, surface what changed (new functions, deleted functions, changed Level-1)
- **Cross-project edges** — nodes in one project's index referencing nodes in another (shared utility modules)
- **Notion sync** — push `document` nodes and `constraint:decision` nodes to Notion via pev-agent-system's existing sync machinery

---

## Package Setup

```toml
# pyproject.toml (cortex)
[project]
name = "cortex"
version = "0.1.0"
description = "Project knowledge indexer for AI agents"
requires-python = ">=3.11"
dependencies = [
    "pyyaml>=6.0",
    "markdown-it-py>=3.0",
    "click>=8.0",          # CLI
    "mcp>=1.0",            # MCP server (FastMCP)
]

[project.optional-dependencies]
push = [
    "httpx>=0.27",         # for cortex push
]

[project.scripts]
cortex = "cortex.cli:main"
```

No `dflow` dependency. No `sqlmodel`. No FastAPI. SQLite comes from Python's stdlib `sqlite3`. Cortex is a pure scanning/indexing library with a thin CLI and an MCP server. The service layer lives in pev-agent-system.

---

## Future Features (Backlog)

### Staleness as Git Age (Not Just CLEAN/DRIFT)

**Current behaviour:** `cortex_check` reports `CLEAN` vs `STRUCTURAL_DRIFT` based on whether the source file hash has changed since the node was last indexed.

**Proposed:** Surface *how stale* a node is — specifically, how many commits ago the file was last scanned. An agent seeing `STRUCTURAL_DRIFT (4 commits ago)` can make a much better decision about whether to re-read the source than one seeing just `STRUCTURAL_DRIFT`.

**Implementation sketch:**
- At build time, store `git rev-parse HEAD` in a `.cortex/meta.json` alongside the DB.
- `cortex_check` runs `git log --oneline <stored_hash>..HEAD -- <file>` and counts commits.
- Falls back gracefully if git is not available (non-git projects).

**Why it's parked:** Requires subprocess calls to `git` inside the scanner, which currently has zero subprocess use. Needs a clean abstraction boundary — probably a `cortex.vcs` module that is an optional dependency. Not worth the complexity until the core index is proven on real projects.

---

### Scanner Version Invalidation

**Problem:** The `source_hash` on each node tracks whether the *source file* has changed — not whether the *scanner* has changed. When new scanner features are added (e.g. raises extraction, entry-point tags), existing nodes in the DB have a matching `source_hash` so `upsert_node` skips the write. The new data never lands without manually deleting `.cortex/cortex.db`.

**Proposed:** Store a `scanner_version` string in `.cortex/meta.json` alongside the DB. At the start of `builder.build()`, compare the stored version against the current one. If they differ, drop and recreate the DB before scanning.

```json
// .cortex/meta.json
{ "scanner_version": "0.3.0", "built_at": "2026-03-02T..." }
```

`scanner_version` is bumped manually whenever a scanner change would produce different node content from the same source. Regular `upsert_node` hash-skipping continues to work within a version — only cross-version upgrades trigger a full rebuild.

**Why it's parked:** Low priority while there is a single user. The manual `db.unlink()` workaround is acceptable at this scale. Implement before cortex is used by multiple people or on CI.

---

### Cross-Project Edges

**Problem:** When `workflow_canvas` calls `from pm.method import pm_method`, cortex has no way to resolve that import to `pm_mvp::pm.method::pm_method`. It either drops the edge or emits a dangling `depends_on` to an unresolvable node ID.

**Proposed:** A project registry — a `cortex_workspace.yaml` at the multi-repo root — that maps package names to project roots:

```yaml
projects:
  pm_mvp:     c:/Users/dap182/Documents/git/temp/pm_mvp
  workflow_canvas: c:/Users/dap182/Documents/git/temp/workflow_canvas
```

The module scanner would consult this registry when resolving imports, emitting `depends_on` edges whose `to_id` resolves across project boundaries (e.g. `workflow_canvas::pm_provider --depends_on--> pm_mvp::pm.method`).

**Why it's parked:** Needs a workspace-level build command (`cortex build-workspace`) and a shared DB or a DB-of-DBs pattern. The current per-project build model doesn't support it. Design this when multi-repo navigation becomes the primary use case.

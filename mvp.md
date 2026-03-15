# Cortex MVP — v0.1

The MVP is the **structural layer only**: scan, index, store, serve. No pev-agent-system. No vectors. No hosted service. It is a local tool that an agent can call via MCP to navigate any Python project without reading source files.

---

## What's In

### Scanners (data sources)

| Scanner | What it produces |
|---|---|
| `doc_scanner.py` | `document` + `constraint:decision` nodes from `docs/` markdown |
| `module_scanner.py` | `atomic_process` + `composite_process` nodes from `.py` AST |

Only the Python stdlib `ast` module is used — no external annotations required. Functions containing `Step()` / `AutoStep()` calls get an optional `level_steps` field — see [Data Model](data_model.md#step-markers).

### Store

`.cortex/cortex.db` — SQLite, stdlib `sqlite3`, no ORM.

```
nodes    — one row per CortexNode
edges    — one row per CortexEdge
tags     — many-to-many node ↔ tag
node_fts — FTS5 virtual table over level_1 + level_2 for text search
```

Writes are always upserts. Running `cortex build` twice on the same project is safe and incremental: files are hash-checked, unchanged files produce no DB writes.

### CLI

| Command | What it does |
|---|---|
| `cortex build <project_root>` | Run all scanners, upsert into `.cortex/cortex.db` |
| `cortex render --level N [--id <node_id>]` | Print nodes at level 0/1/2/steps |
| `cortex list [--type <node_type>] [--tag <tag>]` | Filtered node enumeration |
| `cortex graph <node_id> [--direction in\|out] [--depth N]` | Traverse edges from a node |
| `cortex check` | Per-node staleness / confidence report (`CLEAN`, `DOC_STALE`, `STRUCTURAL_DRIFT`, `UNVERIFIED`) |
| `cortex export` | Write `.cortex/index.json` JSON snapshot from the DB |

### MCP Server

`cortex/mcp_server.py` — a [FastMCP](https://github.com/jlowin/fastmcp) server that wraps the same internals as the CLI. Registered in `.vscode/mcp.json` or Claude Desktop's config. No separate process to manage — `uvx cortex-mcp` starts it.

| MCP Tool | Maps to | User Story |
|---|---|---|
| `cortex_build(project_root)` | `cortex build` | Pre-flight: ensure index is fresh |
| `cortex_render(project_root, level, node_id?)` | `cortex render` | US-01, US-02: map before reading |
| `cortex_list(project_root, node_type?, tag?)` | `cortex list` | US-05: enumerate / audit |
| `cortex_graph(project_root, node_id, direction, depth)` | `cortex graph` | US-03, US-04, US-06, US-07: navigate edges |
| `cortex_search(project_root, query, level?)` | FTS5 query on `node_fts` | Fast keyword discovery without embeddings |
| `cortex_check(project_root)` | `cortex check` | Pre-flight: trust the index or rebuild |

---

## What's NOT In

| Capability | Why deferred |
|---|---|
| `cortex push` to pev-agent-system | Phase 3 — needs pev-agent-system REST changes |
| `cortex validate` (lint rules) | Post-MVP — requires defining and maintaining rule configs; not useful until the graph is mature |
| dFlow DB integration (`dflow_reader.py`) | Post-MVP — reading `.dflow/workflow.db` to enrich Level-1 from `purpose=` is valuable but not required; Step marker detection via AST is sufficient for v0.1 |
| Vector / RAG search | Phase 3 / Phase 4 — FTS5 covers the MVP agent use case |
| Cross-project edges | Future — single-project scope for v0.1 |
| Notebook code scanner | By design (D3) — notebooks are `document` nodes, not scanned for code |
| Notion sync | Phase 4 |

---

## File Tree

```
cortex/
  __init__.py
  ontology.py               # NodeType, EdgeType enums from cortex_ontology.yaml
  models.py                 # CortexNode, CortexEdge, CortexIndex dataclasses
  scanners/
    __init__.py
    doc_scanner.py          # markdown → document + constraint nodes
    module_scanner.py       # Python AST → process nodes + Step()/AutoStep() detection
  index/
    __init__.py
    builder.py              # orchestrates scanners, validates ontology, upserts DB
    db.py                   # schema init, upsert, query helpers, FTS5
  renderers/
    __init__.py
    agent.py                # level-0/1/2 text formatting for CLI and MCP output
  mcp_server.py             # FastMCP server — wraps builder + db query functions
  cli.py                    # click CLI: build / render / list / graph / check / validate / export
pyproject.toml
cortex_ontology.yaml        # authoritative node/edge type definitions
```

**Dependencies:** `pyyaml`, `markdown-it-py`, `click`, `mcp` (FastMCP). `sqlite3` and `ast` are stdlib. No `dflow`, no `sqlmodel`, no FastAPI.

---

## Build Order (for agents continuing this work)

Build in this exact sequence. Each step depends on everything above it. Mark items `[x]` as completed.

### Step 1 — Project scaffold + pyproject.toml `[x]`

Update `pyproject.toml` to add all dependencies and entry points. Create package skeleton (`__init__.py` files only — no logic yet) for every directory in the file tree:

```
cortex/__init__.py
cortex/scanners/__init__.py
cortex/index/__init__.py
cortex/renderers/__init__.py
```

Add to `pyproject.toml`:
- dependencies: `pyyaml`, `markdown-it-py`, `click`, `mcp[cli]` (FastMCP)
- `[tool.poetry.scripts]` entry: `cortex = "cortex.cli:main"`

---

### Step 2 — `cortex/ontology.py` `[x]`

Parse `cortex_ontology.yaml` at import time and expose:
- `NodeType` enum (values: `atomic_process`, `composite_process`, `entity`, `constraint`, `document`)
- `EdgeType` enum (values: `consumes`, `produces`, `composes`, `delegates_to`, `depends_on`, `constrains`, `validates`, `documents`, `supersedes`)
- `ONTOLOGY` dict (the raw parsed YAML, used by builder for validation)
- Helper: `valid_edge(edge_type, from_node_type, to_node_type) -> bool`

The YAML path is resolved relative to this file's package root (not CWD).

---

### Step 3 — `cortex/models.py` `[x]`

Pure dataclasses — no DB logic, no I/O.

```python
@dataclass
class StepMarker:
    step_num: float | int
    name: str
    purpose: str
    inputs: str | None = None
    outputs: str | None = None
    critical: str | None = None

@dataclass
class CortexNode:
    id: str                          # "{project_id}::{dotpath}::{name}"
    node_type: str                   # NodeType value
    subtype: str | None
    title: str
    tags: list[str]
    location: str                    # repo-relative path
    status: str                      # "active" | "deprecated" | "proposed"
    level_0: str
    level_1: str
    level_2: str | None
    level_3_location: str | None     # "path/file.py#L10-L45"
    level_steps: list[StepMarker] | None
    source: str                      # "ast" | "doc_scanner" | "manual"
    source_hash: str                 # SHA-256[:16] of raw source text
    doc_hash: str | None             # SHA-256[:16] of docstring only
    dflow_meta: dict | None

@dataclass
class CortexEdge:
    id: str                          # "{from_id}::{edge_type}::{to_id}"
    edge_type: str
    from_id: str
    to_id: str
    weight: float = 1.0
    meta: dict | None = None

@dataclass
class CortexIndex:
    cortex_version: str
    project_id: str
    project_root: str
    built_at: str                    # ISO 8601
    nodes: list[CortexNode]
    edges: list[CortexEdge]
```

---

### Step 4 — `cortex/index/db.py` `[x]`

SQLite layer. All functions take a `db_path: Path` argument. No global state.

**Schema** (create-if-not-exists):

```sql
CREATE TABLE IF NOT EXISTS nodes (
    id TEXT PRIMARY KEY,
    node_type TEXT NOT NULL,
    subtype TEXT,
    title TEXT NOT NULL,
    location TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    source TEXT NOT NULL,
    source_hash TEXT NOT NULL,
    doc_hash TEXT,
    level_0 TEXT NOT NULL,
    level_1 TEXT NOT NULL,
    level_2 TEXT,
    level_3_location TEXT,
    level_steps TEXT,       -- JSON array or NULL
    dflow_meta TEXT,        -- JSON object or NULL
    updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS edges (
    id TEXT PRIMARY KEY,
    edge_type TEXT NOT NULL,
    from_id TEXT NOT NULL,
    to_id TEXT NOT NULL,
    weight REAL NOT NULL DEFAULT 1.0,
    meta TEXT               -- JSON or NULL
);

CREATE TABLE IF NOT EXISTS tags (
    node_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    PRIMARY KEY (node_id, tag)
);

CREATE VIRTUAL TABLE IF NOT EXISTS node_fts USING fts5(
    id UNINDEXED,
    level_1,
    level_2,
    content='nodes',
    content_rowid='rowid'
);
```

**Functions to implement:**

- `init_db(db_path)` — create schema
- `upsert_node(db_path, node: CortexNode)` — INSERT OR REPLACE + sync tags + update FTS
- `upsert_edge(db_path, edge: CortexEdge)` — INSERT OR REPLACE
- `get_node(db_path, node_id) -> CortexNode | None`
- `get_source_hash(db_path, node_id) -> str | None` — used for incremental check
- `query_nodes(db_path, node_type=None, tag=None) -> list[CortexNode]`
- `query_edges(db_path, node_id, direction="out", depth=1) -> list[CortexEdge]`
- `fts_search(db_path, query, level=None) -> list[CortexNode]`
- `all_nodes(db_path) -> list[CortexNode]`
- `all_edges(db_path) -> list[CortexEdge]`

Incremental write logic: in `upsert_node`, compute `source_hash` on the incoming node, compare to `get_source_hash()` — if identical, skip the write entirely and return `False`. Return `True` when a write occurred.

---

### Step 5 — `cortex/scanners/module_scanner.py` `[x]`

**This is the hardest file.** Python `ast` only — no imports of the scanned project.

Entry point: `scan_module(file_path: Path, project_root: Path, project_id: str) -> tuple[list[CortexNode], list[CortexEdge]]`

**What it must produce:**

1. One `composite_process` node per `.py` file (the module itself).
   - `id` = `"{project_id}::{dotpath}"` where dotpath is the module's import path derived from `file_path` relative to `project_root`
   - `level_0` = module name (last component of dotpath)
   - `level_1` = first line of module docstring, or `"{module_name} module"`
   - `level_2` = full module docstring or `None`
   - `source_hash` = SHA-256[:16] of the entire file content

2. One `atomic_process` node per `FunctionDef` / `AsyncFunctionDef` at any nesting level.
   - `id` = `"{project_id}::{dotpath}::{function_name}"`
   - `level_0` = function name
   - `level_1` = `"{name}({args_signature})" + " — " + first_docstring_sentence` (or just signature if no docstring)
   - `level_2` = full docstring or `None`
   - `level_3_location` = `"{rel_path}#L{start}-L{end}"` from AST node `lineno` / `end_lineno`
   - `level_steps` = extracted from `Step()` / `AutoStep()` keyword args in the function body (see below)
   - `source_hash` = SHA-256[:16] of the function's raw source lines (use `ast.get_source_segment` or slice by line number)

3. `composes` edges: module → each of its top-level functions.

4. `depends_on` edges from import analysis: for each `import X` or `from X import Y`, if `X` resolves to another module **within `project_root`**, emit a `depends_on` edge from the current module node to the target module node. Use best-effort path resolution — no need to handle `sys.path` tricks.

**Step marker extraction** (AST pass inside each function body):

Walk the function's AST looking for `Assign` nodes where the value is a `Call` whose `func` is a `Name` with `id` in `{"Step", "AutoStep"}`. Extract keyword arguments as a dict. Build a `StepMarker` from the keywords. Collect all in order by `step_num`. If none found, `level_steps = None`.

**Args signature extraction:**

Use `ast.unparse(node.args)` (Python 3.9+) to get a clean signature string. Strip `self` / `cls` from the front for methods.

---

### Step 6 — `cortex/scanners/doc_scanner.py` `[x]`

Entry point: `scan_docs(docs_dir: Path, project_root: Path, project_id: str) -> tuple[list[CortexNode], list[CortexEdge]]`

Walk every `.md` file under `docs_dir`. For each file:

1. Emit one `document` node for the file itself.
   - `id` = `"{project_id}::docs::{stem}"` (e.g. `pm_mvp::docs::data_model`)
   - `level_0` = first H1 heading text, or filename stem
   - `level_1` = first paragraph of prose after the H1
   - `level_2` = full file text (truncated to 4000 chars)
   - `source_hash` = SHA-256[:16] of file content

2. For each H2 heading, emit a child `document` node:
   - `id` = `"{file_node_id}::{slugified_heading}"`
   - `level_0` = heading text
   - `level_1` = first paragraph under that heading
   - `documents` edge: child → file node

3. Pattern-match decision markers: lines matching `**D\d+` or `> D\d+` or `Decision D\d+` within heading sections. For each match, emit a `constraint` node (`subtype="decision"`) and a `constrains` edge from it to the enclosing document node.

Use `markdown-it-py` to tokenise. Slug headings with `re.sub(r'[^a-z0-9]+', '-', heading.lower()).strip('-')`.

---

### Step 7 — `cortex/index/builder.py` `[ ]`

Entry point: `build(project_root: Path, project_id: str | None = None) -> dict`

Steps:
1. Derive `project_id` from `project_root.name` if not provided.
2. Ensure `.cortex/` directory exists. Call `db.init_db(db_path)`.
3. Run `module_scanner.scan_module()` on every `.py` file under `project_root` (skip `.cortex/`, `__pycache__`, `.git`, `.venv`, `node_modules`).
4. Run `doc_scanner.scan_docs()` on `project_root / "docs"` if it exists.
5. For each node and edge: call `db.upsert_node()` / `db.upsert_edge()`. Count writes vs skips.
6. Validate ontology: for every edge, call `ontology.valid_edge()`. Log warnings for violations (do not fail the build).
7. Return a summary dict: `{"nodes_written": N, "nodes_skipped": M, "edges_written": E, "warnings": [...]}`

---

### Step 8 — `cortex/renderers/agent.py` `[ ]`

Entry point functions (all return `str`):

- `render_level_0(nodes: list[CortexNode]) -> str` — one line per node: `{id}`
- `render_level_1(nodes: list[CortexNode]) -> str` — one line per node: `{id}  {level_1}`
- `render_level_2(nodes: list[CortexNode]) -> str` — for each node: header + `level_2` block if present
- `render_steps(nodes: list[CortexNode]) -> str` — for each node with `level_steps`: numbered step list
- `render_graph(node: CortexNode, edges: list[CortexEdge], direction: str) -> str` — ASCII tree showing edge type + target id

---

### Step 9 — `cortex/cli.py` `[ ]`

Click CLI. Entry point registered as `cortex` in `pyproject.toml`. Implement all six commands:

```
cortex build <project_root> [--id <project_id>]
cortex render --level <0|1|2|steps> [--id <node_id>] <project_root>
cortex list [--type <node_type>] [--tag <tag>] <project_root>
cortex graph <node_id> [--direction in|out] [--depth N] <project_root>
cortex check <project_root>
cortex export <project_root>
```

`check` command logic: for each node, compare `source_hash` stored in DB to current file hash. If file has changed since last build, report `STRUCTURAL_DRIFT`. If node has `doc_hash` and docstring changed, report `DOC_STALE`. If neither, `CLEAN`. If node has no `source_hash`, `UNVERIFIED`.

`export` command: load all nodes + edges from DB, serialize to `CortexIndex` dataclass, write as JSON to `.cortex/index.json`.

---

### Step 10 — `cortex/mcp_server.py` `[ ]`

FastMCP server. Install `mcp[cli]` (FastMCP is bundled in the `mcp` package as of v1.x).

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("cortex")
```

Register six tools (thin wrappers — call the same functions as the CLI):
- `cortex_build(project_root: str) -> str`
- `cortex_render(project_root: str, level: int, node_id: str | None = None) -> str`
- `cortex_list(project_root: str, node_type: str | None = None, tag: str | None = None) -> str`
- `cortex_graph(project_root: str, node_id: str, direction: str = "out", depth: int = 1) -> str`
- `cortex_search(project_root: str, query: str, level: int | None = None) -> str`
- `cortex_check(project_root: str) -> str`

All tools return plain text (the same string the CLI would print). Errors are caught and returned as `"ERROR: {message}"` — never raise from a tool.

Add `[tool.poetry.scripts]` entry: `cortex-mcp = "cortex.mcp_server:mcp.run"` (or `mcp.run(transport="stdio")`).

---

### Step 11 — Smoke test on `pm_mvp` `[ ]`

From the `cortex/` repo root:

```bash
poetry install
cortex build ../pm_mvp
cortex render --level 1 ../pm_mvp
cortex list --type atomic_process ../pm_mvp
cortex export ../pm_mvp
```

All "Done When" checklist items must pass. Mark them in the section below.

---

## Done When

- [ ] `cortex build <project_root>` produces `.cortex/cortex.db` on `pm_mvp` with no crashes
- [ ] All `docs/` markdown sections appear as `document` nodes
- [ ] All `.py` modules appear as `composite_process` nodes
- [ ] All functions appear as `atomic_process` nodes with Level-1 from first docstring sentence
- [ ] `cortex render --level 0` / `--level 1` / `--level 2` / `--level steps` produce correct output
- [ ] `cortex graph <node_id>` returns correct edges at the right depth
- [ ] `cortex check` returns per-node confidence status
- [ ] MCP server starts with `uvx cortex-mcp` and all six tools respond correctly
- [ ] `cortex export` produces a valid `.cortex/index.json`
- [ ] Re-running `cortex build` on an unchanged project makes zero DB writes (incremental check works)

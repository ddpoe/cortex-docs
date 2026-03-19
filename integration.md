# Integration Guide

How cortex connects to dFlow and how to consume the output index.

---

## Part 1: dFlow → Cortex

### What dFlow Provides

When `dflow scan` + `dflow assemble` have been run in a project, `.dflow/workflow.db` exists with these tables:

```sql
modules          -- scanned source files
functions        -- all @workflow and @task decorated functions
steps            -- Step() and AutoStep() markers within workflows
workflow_entries -- entry-point workflows
classes          -- @test_suite decorated classes
coverage_links   -- test → function coverage (from covers= field)
```

### How Cortex Reads It

Cortex reads the DB via plain SQL. **No import of dFlow.** If `.dflow/workflow.db` does not exist, the module scanner falls back to pure AST and skips this step.

**Key queries:**

```sql
-- All annotated functions → CortexNode (atomic_process or composite_process)
SELECT
    f.name,
    f.signature,
    f.purpose,
    f.inputs,
    f.outputs,
    f.critical,
    f.decorator_type,   -- 'workflow' | 'task'
    f.line_start,
    f.line_end,
    m.file_path
FROM functions f
JOIN modules m ON f.module_id = m.id;

-- Steps within workflows → composes edges
SELECT
    f.name AS workflow_name,
    s.step_num,
    s.name AS step_name,
    s.purpose,
    s.called_function   -- AutoStep target, resolved by dflow assemble
FROM steps s
JOIN functions f ON s.function_id = f.id;

-- Coverage links → validates edges
SELECT
    cl.test_function_name,
    cl.test_module_path,
    cl.target_dotpath   -- e.g. "pm.database.get_engine"
FROM coverage_links cl;
```

### Mapping: dFlow → CortexNode

| dFlow | `node_type` | `level_1` source | `source` field |
|---|---|---|---|
| `@task` function | `atomic_process` | `purpose=` + signature | `dflow` |
| `@workflow` function | `composite_process` | `purpose=` + signature | `dflow` |
| `@test_workflow` function | `constraint` (subtype: `test`) | purpose string | `dflow` |
| `@test_suite` class | `constraint` (subtype: `test`) | class docstring | `dflow` |
| `inputs=`/`outputs=` values | `entity` (best-effort) | value string | `dflow` |

### Mapping: dFlow → CortexEdge

| dFlow table/field | `edge_type` | Direction |
|---|---|---|
| Module contains function | `composes` | module → function |
| `@workflow` → `Step`/`AutoStep` | `composes` | workflow → step node |
| AutoStep `called_function` | `delegates_to` | workflow → called task |
| `coverage_links.target_dotpath` | `validates` | test → function |
| `inputs=` on `@task` | `consumes` | function → entity (best-effort) |
| `outputs=` on `@task` | `produces` | function → entity (best-effort) |

---

## Part 2: Consuming the Index

`.cortex/index.json` is a self-contained artifact. Any tool or service that can read JSON can consume it. Two built-in consumption paths:

```bash
# Render as agent-readable text at a given level:
cortex render <project_root> --level 0
cortex render <project_root> --level 1 --id pm_image_analysis::methods.binary_label.run

# Validate the index against the ontology:
cortex validate <project_root>
```

The index format is specified in [Data Model](data_model.md). An agent can read rendered output directly, a database can ingest the JSON, a visualization layer can render the graph — those integrations are out of scope for this document.

### CLI Workflow (end-to-end)

```bash
# 1. (Optional) annotate with dFlow and build the DB:
dflow scan
dflow assemble

# 2. Build the cortex index:
cortex build <project_root>

# 3. Inspect and validate:
cortex validate <project_root>
cortex render <project_root> --level 1

# 4. Pre-flight staleness check (run before trusting the index):
cortex check <project_root>
# Returns per-node CLEAN / DOC_STALE / STRUCTURAL_DRIFT
# Returns per-edge VERIFIED / UNVERIFIED / BROKEN for semantic edges
```

---

## Part 3: RAG Query

Cortex supports graph-augmented retrieval on top of the index. Embeddings are stored in a LanceDB vector store at `.cortex/embeddings.lance` alongside `index.json`.

### What Gets Embedded

| Source | Text embedded | Why |
|---|---|---|
| Code nodes | `level_1` + `level_2` (signature + docstring) | Captures function intent and API contract without implementation noise |
| Doc sections | section heading + section content | Captures design rationale, ADR decisions, architectural context |

Raw source code is **not** embedded. Signatures and docstrings carry the semantic meaning; embedding implementation details would add token cost and dilute retrieval relevance.

Code nodes and doc sections live in the same embedding space — a query can surface a design decision or a doc section just as easily as a function.

### Section Length and Chunking

Embedding quality degrades on long text — the vector becomes a blurry average of all concepts rather than a sharp signal for any one. Doc sections are the natural chunking boundary, so keeping sections focused produces better embeddings.

**V1 — Embed as-is with warnings.** Sections are embedded whole. `cortex check` warns when a section exceeds ~300 words (suggesting a split). No automatic chunking.

**V2 — Intra-section chunking for oversized sections.** Sections that exceed the warning threshold are split into overlapping windows at embedding time. Because all content within a section is thematically related, overlaps can be aggressive (50%+) without introducing cross-topic noise. Each chunk gets its own vector but maps back to the same `section_id`, so retrieval still returns the full section.

### CLI Commands

```bash
# Build embeddings (run after cortex build, or when nodes change):
cortex embed <project_root>

# Query:
cortex query <project_root> "how does the classifier handle class imbalance" --level 2 --k 5
```

Embedding is incremental — only nodes whose `code_hash` or `desc_hash` changed since the last embed run are re-embedded.

### Query Phases

**Phase 1 — Vector retrieval**
The query string is embedded and matched against all node and doc-section embeddings. Top-k results are returned by cosine similarity.

**Phase 2 — Graph expansion**
For each retrieved node, one hop of semantic edges is followed automatically:

| Direction | Edge type | What you get |
|---|---|---|
| Inbound | `constrains` | Design decisions and schemas that restrict this node |
| Inbound | `documents` | Doc sections that describe this node |
| Outbound | `validates` | Tests that verify this node |

Structural edges (`delegates_to`, `composes`) are not followed by default. Pass `--expand-calls` to also include `delegates_to` neighbours.

**Phase 3 — Level resolution**
Primary hits are returned at the requested `--level`. Expanded (graph-neighbour) nodes are always returned at `level_2`, regardless of the query level. Expanded nodes are supporting context — level_2 is the right depth for that role: enough to understand the relationship without bloating the response with full source.

### Response Shape

```json
{
  "query": "how does the classifier handle class imbalance",
  "level": 2,
  "check_status": "CLEAN",
  "hits": [
    {
      "node_id": "pm_image_analysis::methods.binary_label.run::train_classifier",
      "score": 0.91,
      "confidence": "VERIFIED",
      "content": "<level_2 docstring>",
      "expanded": [
        {
          "node_id": "pm_image_analysis::D3",
          "edge_type": "constrains",
          "confidence": "VERIFIED",
          "content": "<level_2 of D3>"
        },
        {
          "node_id": "pm_image_analysis::docs.design.analysis_split#calibration",
          "edge_type": "documents",
          "confidence": "VERIFIED",
          "content": "<level_2 of doc section>"
        }
      ]
    }
  ]
}
```

`check_status` at the top level reflects the result of an implicit `cortex check` run before retrieval. If `STRUCTURAL_DRIFT` or `DESC_STALE` nodes exist, the status is surfaced here so the caller knows the index may not be fully current. Individual `confidence` fields on each hit and expanded node give finer-grained trust signals.

### Design Decisions

**Embed `level_1` + `level_2` for code, heading + content for doc sections.** These are the semantic summaries — clean, dense signal. Raw source is excluded.

**Incremental embedding.** Re-embed only when hashes change, piggy-backing on the existing hash-based change detection in `cortex build`.

**Section-length warnings before chunking.** V1 pushes quality upstream by encouraging well-scoped sections. V2 adds a safety net for sections that are legitimately long.

**Expanded nodes are always `level_2`.** Expanded context is supporting material. It should be readable and informative but not so deep it dominates the context window. Full source (level_3) is never returned automatically — the `level_3_location` pointer is included so the agent can fetch it explicitly if needed.

**One hop only by default.** Following two hops expands context exponentially and typically dilutes signal. `--expand 2` is available but not the default.

**`UNVERIFIED` nodes are returned, not suppressed.** Hiding stale results silently is worse than surfacing them with a flag. The caller decides how to handle low-confidence context.

---

## Part 4: Graph Traversal

Once you have a node ID (from RAG, from `cortex list`, or directly), `cortex graph` lets you navigate its relationships deterministically. This is **not** similarity search — it follows typed edges in the index exactly.

```bash
# What composes this module? (inbound composes edges)
cortex graph pm_image_analysis::methods.binary_label.run --rel composes --direction in

# What does this workflow delegate to? (outbound delegates_to)
cortex graph pm_image_analysis::run_pipeline --rel delegates_to --direction out

# What constrains this function? (inbound constrains)
cortex graph pm_image_analysis::methods.binary_label.run::train_classifier --rel constrains --direction in

# What does this decision constrain? (outbound constrains)
cortex graph pm_image_analysis::D3 --rel constrains --direction out

# All relationships one hop out, all edge types:
cortex graph pm_image_analysis::methods.binary_label.run::train_classifier --all --depth 1

# Two hops, semantic edges only:
cortex graph pm_image_analysis::methods.binary_label.run::train_classifier --semantic --depth 2
```

Edge direction matters. `constrains` goes *from* a decision/schema *to* a process — so "what constrains this function" requires `--direction in`, while "what does D3 constrain" requires `--direction out`.

Results are returned at `level_1` by default (enough to identify nodes). Pass `--level 2` to get full docstrings for all returned nodes.

---

## Part 5: Filtered List Queries

`cortex list` enumerates nodes or edges that match given criteria. It does not rank by relevance — it returns everything that passes the filter. Primary use cases: auditing, review queues, and feeding the GUI.

```bash
# All nodes with DOC_STALE status:
cortex list --status DOC_STALE

# All atomic_process nodes tagged "classification":
cortex list --type atomic_process --tag classification

# All semantic edges with UNVERIFIED confidence:
cortex list --edges --confidence UNVERIFIED

# All edges originating from a specific decision node:
cortex list --edges --source pm_image_analysis::D3

# All constraint nodes (decisions + tests + schemas):
cortex list --type constraint
```

This is the same query layer the GUI uses for its review queue — "show me everything that needs human attention" is just `cortex list --status DOC_STALE` plus `cortex list --edges --confidence UNVERIFIED` composed together.

---

## Composing Query Modes

The three modes are designed to be used together in sequence:

| Step | Command | Purpose |
|---|---|---|
| 1 | `cortex check` | Establish index trust level before starting |
| 2 | `cortex query "..."` | Find the relevant cluster semantically |
| 3 | `cortex graph <node_id> --all --depth 1` | Explore context from the best hit |
| 4 | `cortex render <node_id> --level 3` | Go deep on one specific node |
| 5 | `cortex list --edges --source <node_id>` | Audit what this node is connected to |

This flow moves from "find it" → "understand its context" → "read the code" → "verify its relationships". Each step narrows scope and increases depth.

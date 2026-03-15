# Cortex Ontology

The ontology defines all valid node types and edge types in the cortex knowledge graph. It is validated at build time — any violation fails the build rather than silently producing a malformed index.

The definitive source is [`cortex_ontology.yaml`](../cortex_ontology.yaml). This document is the human-readable explanation of the design rationale.

## Philosophical Basis

All code is a series of processes. This is scale-invariant:

- A **function** transforms inputs to outputs → Process
- A **module** orchestrates functions → Process
- A **package** groups modules → Process
- A **pipeline** orchestrates modules → Process
- A **notebook** demonstrates a workflow → Document (not a process)

There is no special "function type" or "module type" — they are all `atomic_process` or `composite_process`. This means the ontology doesn't need to change as you zoom in or out. The same graph traversal logic works at any granularity.

The ontology is grounded in [PROV-O (W3C Provenance Ontology)](https://www.w3.org/TR/prov-o/) principles — Activity (Process), Entity, and Agent — simplified and made concrete for software projects.

## Node Types

### `atomic_process`

The **leaf unit** of work. It does not contain other processes.

- A Python function or method
- A `@task` decorated function (dFlow)
- A pipeline step with no sub-steps

The key test: "Does this contain other named processes?" If yes, it's `composite_process`.

### `composite_process`

**Orchestrates or contains** other processes. Scale-invariant — a module, package, `@workflow`, and entire pipeline are all `composite_process`.

- A Python module (`analysis.py`)
- A Python package (`pm_image_analysis/`)
- A `@workflow` decorated function (dFlow)
- A Snakemake/Nextflow pipeline definition
- A dFlow `Step`/`AutoStep` block (a named stage within a workflow)

Edge: `composite_process --composes--> atomic_process/composite_process`

### `entity`

**Data or config** that processes act on. Something that exists and is consumed or produced.

Subtypes: `data_artifact`, `config_artifact`, `model_artifact`

- An AnnData object
- A config YAML file
- A trained model `.pkl`
- A results DataFrame

### `constraint`

**Restricts or validates** valid behavior of processes.

Subtypes: `test`, `decision`, `schema`, `contract`

- `test`: A `@test_workflow` or `@test_suite` (dFlow)
- `decision`: A design/architectural decision (D0, D1, ...) — typically a section of a design doc elevated to a first-class node
- `schema`: A data schema or validation rule
- `contract`: An interface contract

### `document`

**Human-readable description** of any other node. Documents do not execute.

- Markdown files in `docs/`
- `README.md` files
- Docstrings (rendered as document nodes linked to their parent function)
- Jupyter notebooks — treated as documentation, not code. Notebooks should only contain `@workflow` orchestration with `Step()`/`AutoStep()` — no heavy logic.

## Edge Types

### Data Flow
| Edge | From | To | Meaning |
|---|---|---|---|
| `consumes` | process | entity | Process takes entity as input |
| `produces` | process | entity | Process creates entity as output |

### Structural
| Edge | From | To | Meaning |
|---|---|---|---|
| `composes` | composite_process | any process | Static containment. Module composes its functions. |
| `delegates_to` | composite_process | any process | Runtime invocation. Workflow calls a task in another module. |
| `depends_on` | any process | any process or entity | Import or runtime dependency. |

### Semantic
| Edge | From | To | Meaning |
|---|---|---|---|
| `constrains` | constraint | any process or entity | A decision or schema restricts behavior. |
| `validates` | constraint (test) | any process | A test verifies correctness. |
| `documents` | document | any node | A doc describes another node. |
| `supersedes` | process or constraint | process or constraint | X replaces Y. |

## Proposed: `tested_by` Edge

> **Status: under consideration.** Not yet in the ontology. Recorded here for design review.

### Motivation

The existing `validates` edge runs *from* the test *to* the thing it tests:

```
test_get_module_contracts  --[validates]-->  _get_module_contracts
```

This answers: *"does this function have a test asserting its behaviour?"*

A `tested_by` edge would run in the opposite direction:

```
_get_module_contracts  --[tested_by]-->  test_get_module_contracts
```

This answers: *"if I change this function, which tests should I run?"*  — which is the more natural question during development and refactoring.

### Why not just reverse `validates`?

`validates` is a **semantic edge** (human-asserted, confidence-tracked). It encodes design intent: "this test was deliberately written to verify this contract."

`tested_by` would be a **structural edge** (auto-inferred from the call graph). It says: "this test file calls this function, therefore it exercises it, at least incidentally." The bar is lower and the inference is mechanical.

They are complementary:

| | `validates` | `tested_by` |
|---|---|---|
| Direction | test → source | source → test |
| Who asserts | Human (or dFlow `covers=`) | Scanner (call graph) |
| Confidence tracked | Yes — VERIFIED/UNVERIFIED/BROKEN | No — re-derived on every scan |
| Meaning | Deliberate contract coverage | Incidental or deliberate call coverage |

### Implementation options

**Option A — Call-graph heuristic (no dFlow required)**
The module scanner already produces `depends_on` edges from import analysis. Extending it to a lightweight call-graph pass (find all function call nodes in test files, resolve to their target module functions) would yield `tested_by` edges automatically. No metadata annotation needed from the user.

*Risk:* over-generation. A test that imports a utility function transitively "tests" everything that utility touches. Limiting to direct calls only would reduce noise.

**Option B — dFlow `covers=` metadata**
When dFlow is present, `@test_workflow(covers=["pm.method._get_module_contracts"])` already encodes explicit intent. The scanner could emit `tested_by` edges from each listed symbol back to the test function.

*Risk:* requires dFlow. Coverage annotations get stale if functions are renamed.

**Option C — Both, with a `source` field on the edge**
Emit `tested_by` edges from both sources, tagging them with `{"source": "call_graph"}` or `{"source": "covers_annotation"}`. Agents can weight them differently.

### Open questions

1. Should `tested_by` live in the ontology as a first-class edge type, or be modelled as a reverse-direction `validates` with a flag?
2. If using call-graph inference: how deep? Direct calls only, or transitive?
3. Does the agent UI surface `tested_by` edges differently from `validates`? (e.g. lower confidence badge)


## `composes` vs `delegates_to`

These two are easy to confuse.

**`composes`** is structural/static: "Y is defined inside X." `analysis_module.py` composes `train_classifier` because that function is defined in that module. This is the containment hierarchy.

**`delegates_to`** is runtime/dynamic: "X calls Y when it runs." A `@workflow` in `run_pipeline.py` delegates_to `train_classifier` in `analysis_module.py`. Particularly important for cross-module calls — the workflow `composes` its own steps but `delegates_to` functions defined elsewhere.

## Structural vs Semantic Edges

Edge types are divided into two tiers with fundamentally different reliability guarantees and maintenance requirements.

### Structural Edges (auto-generated)

| Edge type | How populated |
|---|---|
| `composes` | AST — static containment, always deterministic |
| `delegates_to` | AST — call graph analysis |
| `depends_on` | AST — import analysis |
| `consumes` / `produces` | dFlow `inputs=`/`outputs=` metadata, or best-effort from AST |

**These are recomputed on every `cortex scan`.** They are never stored with a confidence state because they are derived mechanically from source truth. If the source changes, the next scan updates them automatically. You do not assert these manually.

### Semantic Edges (human-asserted)

| Edge type | Who asserts it | What it encodes |
|---|---|---|
| `constrains` | Human | A design decision or schema restricts this process |
| `validates` | Human (or dFlow for `@test_workflow`) | A test verifies this process |
| `documents` | Human (or doc_scanner for explicit links) | A doc describes this node |
| `supersedes` | Human | X is the replacement for Y |

**These encode intent and design reasoning — things that cannot be inferred from static analysis alone.** They are asserted once and stored with provenance metadata (`asserted_at_hash`) so staleness can be detected when the underlying source changes.

### Confidence States for Semantic Edges

A semantic edge carries one of three confidence states:

| State | Meaning | Agent behaviour |
|---|---|---|
| `VERIFIED` | `source_hash` of the target node matches `asserted_at_hash` | Trust fully |
| `UNVERIFIED` | Source code changed since edge was last confirmed | Use cautiously, surface the uncertainty |
| `BROKEN` | Target node no longer exists in the index | Ignore; surfaced by `cortex check` |

`VERIFIED` is the default on initial assertion. `UNVERIFIED` is set automatically by `cortex check` when it detects the source hash has changed. A human must explicitly re-confirm an `UNVERIFIED` edge (via `cortex check --confirm` or the GUI) to restore `VERIFIED` status.

**Wrong semantic edges are worse than missing ones.** A missing edge reduces recall. A confidently wrong edge corrupts downstream reasoning. The confidence state exists so agents always know whether to trust or distrust a given semantic relationship.

## What the Ontology Does NOT Cover

- **Tags** — open vocabulary, project-scoped, defined in `cortex_config.yaml`. Used for search/filtering, not for structural reasoning.
- **Resolution levels** (level_0 through level_3) — these are properties of nodes, not part of the ontology. See [data_model.md](data_model.md).
- **Project-specific semantics** — decisions like "D3 constrains train_classifier" are recorded as edges but the meaning of D3 is encoded in the constraint node's content, not in the ontology.

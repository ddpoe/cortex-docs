# Cortex Data Model

The `CortexIndex` is the output of `cortex build`. It is a JSON file written to `.cortex/index.json` relative to the project root.

## Top-Level Structure

```json
{
  "cortex_version": "0.1.0",
  "project_id": "pm_image_analysis",
  "project_root": "R:/Dante/pdac/image_analysis/pm_image_analysis",
  "built_at": "2026-03-01T14:22:00Z",
  "nodes": [ ... ],
  "edges": [ ... ]
}
```

## `CortexNode`

```json
{
  "id": "pm_image_analysis::methods.binary_label.run::train_classifier",
  "node_type": "atomic_process",
  "subtype": null,
  "title": "train_classifier",
  "tags": ["classification", "ml", "calibration"],
  "location": "methods/binary_label/run.py",
  "status": "active",
  "level_0": "train_classifier",
  "level_1": "train_classifier(X, y, calibration='isotonic') — fits isotonic-calibrated SVM classifier",
  "level_2": "Fit and calibrate an SVM classifier.\n\nArgs:\n    X: Feature matrix (n_samples, n_features)\n    y: Binary labels (0/1)\n    calibration: Calibration method. Default 'isotonic'.\n\nReturns:\n    CalibratedClassifierCV fitted on (X, y)\n\nNotes:\n    Critical: calibration='isotonic' is required per decision D3.",
  "level_3_location": "methods/binary_label/run.py#L45-L120",
  "source": "dflow",
  "source_hash": "a3f2c1b4",
  "doc_hash": "e7d9a2f1",
  "last_modified_commit": "c4e8b2a1",
  "dflow_meta": {
    "purpose": "Fit and calibrate an SVM classifier for binary label classification",
    "inputs": "Feature matrix X, binary labels y",
    "outputs": "Fitted CalibratedClassifierCV",
    "critical": null
  }
}
```

### Field Reference

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Globally unique. Format: `{project_id}::{dotpath}::{name}` |
| `node_type` | string | yes | One of the ontology node_types enum |
| `subtype` | string | no | Ontology subtype (e.g. `test`, `decision`, `config`, `data_artifact`) |
| `title` | string | yes | Human-readable name |
| `tags` | list[string] | no | Project-scoped vocabulary from `cortex_config.yaml` |
| `location` | string | yes | Relative path to the file containing this node |
| `status` | string | no | `active`, `deprecated`, `proposed`. Default `active` |
| `level_0` | string | yes | Name only. Always populated. |
| `level_1` | string | yes | Signature + one-line summary. Always populated. |
| `level_2` | string | no | Full docstring. Populated when available. |
| `level_3_location` | string | no | `path/to/file.py#L45-L120`. Agent reads file at this location for full code. |
| `level_steps` | list[object] | no | Ordered step sequence extracted from `Step()` / `AutoStep()` calls in the function body. `null` when no step markers are present. See [Step Markers](#step-markers) below. |
| `source` | string | yes | `dflow`, `ast`, `doc_scanner`, `config_scanner`, `manual` |
| `source_hash` | string | yes | SHA-256 (first 16 chars) of the function's raw source. Recomputed on every `cortex scan`. |
| `doc_hash` | string | no | SHA-256 (first 16 chars) of the docstring only. Populated when a docstring exists. |
| `last_modified_commit` | string | no | Short commit SHA of the last commit that changed this node's source. Populated when git history is available. |
| `dflow_meta` | object | no | Raw dFlow decorator fields when `source = dflow` |

### Step Markers

`level_steps` is an optional field on `atomic_process` and `composite_process` nodes. It is populated by a second AST pass in `module_scanner.py` that detects `Step()` and `AutoStep()` call expressions in the function body and extracts their keyword arguments. No dFlow import or runtime execution is needed — pure AST.

```json
"level_steps": [
  {"step_num": 1, "name": "Load data",    "purpose": "Read raw measurements from disk"},
  {"step_num": 2, "name": "Filter cells",  "purpose": "Remove low-quality cells", "critical": "Removes 20-30% of data"},
  {"step_num": 3, "name": "Train model",   "purpose": "Fit classifier on filtered data"}
]
```

| Subfield | Required | Description |
|---|---|---|
| `step_num` | yes | Integer for major steps, float for sub-steps inside loops (e.g. `1.1`) |
| `name` | yes | Short display name |
| `purpose` | yes | What this block does |
| `inputs` | no | Inputs consumed at this step |
| `outputs` | no | Outputs produced at this step |
| `critical` | no | Warning text — flags long-running or irreversible operations |

An agent requesting `level="steps"` on a node gets this sequence without reading the source file. If `level_steps` is `null` the agent falls back to `level_2` or `level_3`.

---

### Hash Staleness Signals

`cortex check` compares stored hashes against a fresh scan and reports per-node status:

| `source_hash` changed | `doc_hash` changed | Status | Meaning |
|---|---|---|---|
| No | No | `CLEAN` | Fully up to date |
| Yes | Yes | `CLEAN` | Code and docs updated together |
| Yes | No | `DOC_STALE` | Code changed, docstring not updated — highest risk |
| No | Yes | `CLEAN` | Doc-only edit, no code change |

### ID Format

```
{project_id}::{module_dotpath}::{symbol_name}
```

Examples:
- `pm_image_analysis::methods.binary_label.run::train_classifier`
- `pm_image_analysis::methods.binary_label.run` (the module itself, no symbol)
- `pm_image_analysis::docs.design.analysis_split#implementation` (doc section)
- `pm_image_analysis::D3` (decision constraint — short IDs allowed for manually-created nodes)

### Resolution Levels

| Level | Content | Typical tokens | When to request |
|---|---|---|---|
| 0 | Name only (`level_0`) | ~5 | Building a map of what exists |
| 1 | Signature + one-liner (`level_1`) | ~30–60 | Finding what's relevant |
| 2 | Full docstring (`level_2`) | ~100–400 | Understanding contract/behavior |
| 3 | Full code | variable | Debugging, implementing, verifying |

**Level-3 is a pointer, not content.** `level_3_location` gives `file#L45-L120`. The agent reads the file directly at that range. Cortex never stores code text in the index — this keeps index size bounded.

## `CortexEdge`

```json
{
  "source_id": "pm_image_analysis::tests.test_fan_in::test_train_classifier",
  "target_id": "pm_image_analysis::methods.binary_label.run::train_classifier",
  "edge_type": "validates",
  "note": null,
  "source": "dflow",
  "asserted_at_hash": "a3f2c1b4",
  "confidence": "VERIFIED"
}
```

### Field Reference

| Field | Type | Required | Description |
|---|---|---|---|
| `source_id` | string | yes | ID of the originating node |
| `target_id` | string | yes | ID of the destination node |
| `edge_type` | string | yes | One of the ontology edge_types enum |
| `note` | string | no | Optional human explanation of why this edge exists |
| `source` | string | yes | `dflow`, `ast`, `doc_scanner`, `config_scanner`, `manual` |
| `asserted_at_hash` | string | no | `source_hash` of the target node at the time this edge was asserted. **Semantic edges only** — structural edges (`composes`, `delegates_to`, `depends_on`) are recomputed and never carry this field. |
| `confidence` | string | no | `VERIFIED`, `UNVERIFIED`, or `BROKEN`. **Semantic edges only.** Set and maintained by `cortex check`. See [ontology.md](ontology.md#structural-vs-semantic-edges). |

> **Structural edges have no `asserted_at_hash` or `confidence`** — they are recomputed deterministically from source on every `cortex scan` and are always implicitly current.

## Node Type Examples

### `composite_process` — Module

```json
{
  "id": "pm_image_analysis::methods.binary_label.run",
  "node_type": "composite_process",
  "title": "binary_label.run",
  "level_0": "binary_label.run",
  "level_1": "Binary label classification module — SVM training, calibration, evaluation",
  "level_2": "Module-level docstring if present...",
  "level_3_location": "methods/binary_label/run.py",
  "source": "ast"
}
```

### `constraint` (subtype: `decision`) — Design Decision

```json
{
  "id": "pm_image_analysis::D3",
  "node_type": "constraint",
  "subtype": "decision",
  "title": "D3 — Use isotonic calibration for SVM",
  "tags": ["classification", "calibration"],
  "location": "docs/design/analysis_module_split.md",
  "status": "accepted",
  "level_0": "D3",
  "level_1": "D3 (accepted) — SVM classifiers must use isotonic calibration",
  "level_2": "Isotonic calibration was chosen over Platt scaling after empirical comparison on the PDAC dataset. Platt scaling underestimated uncertainty in low-prevalence classes. Decision is binding for all binary classification tasks in this project.",
  "level_3_location": "docs/design/analysis_module_split.md#decision-d3",
  "source": "doc_scanner"
}
```

### `document` — Markdown Section

```json
{
  "id": "pm_image_analysis::docs.design.analysis_split#implementation",
  "node_type": "document",
  "title": "Analysis Module Split — Implementation",
  "location": "docs/design/analysis_module_split.md",
  "level_0": "Implementation",
  "level_1": "How the monolithic decision_boundary() was split into 3 methods",
  "level_2": "First paragraph of the section...",
  "level_3_location": "docs/design/analysis_module_split.md#implementation",
  "source": "doc_scanner"
}
```

### `entity` — Data Artifact

> Note: entities are best-effort, inferred from `inputs=`/`outputs=` dFlow metadata or import analysis. They are less reliably populated than process nodes.

```json
{
  "id": "pm_image_analysis::entity::AnnData_filtered",
  "node_type": "entity",
  "subtype": "data_artifact",
  "title": "Filtered AnnData object",
  "level_0": "AnnData_filtered",
  "level_1": "AnnData object after QC filtering (cells × genes)",
  "source": "dflow"
}
```

## The Full `CortexIndex` Shape

```json
{
  "cortex_version": "0.1.0",
  "project_id": "pm_image_analysis",
  "project_root": "R:/Dante/pdac/image_analysis/pm_image_analysis",
  "built_at": "2026-03-01T14:22:00Z",
  "config": {
    "tag_vocabulary": ["classification", "ml", "preprocessing", "qc", "image-analysis"],
    "excluded_paths": ["_archive/", "__pycache__/", ".venv/"]
  },
  "stats": {
    "node_count": 142,
    "edge_count": 318,
    "node_types": {
      "atomic_process": 87,
      "composite_process": 23,
      "entity": 18,
      "constraint": 9,
      "document": 5
    }
  },
  "nodes": [ ... CortexNode objects ... ],
  "edges": [ ... CortexEdge objects ... ]
}
```

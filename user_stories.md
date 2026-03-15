# User Stories

These stories motivate every design decision in cortex. Each maps to specific system capabilities.

---

## Agent Context Stories

### US-01 — Map Before Reading
> As an AI agent starting work on `pm_image_analysis`, I need to see all modules and their one-line summaries **without reading any code**, so I know where to look before touching any files.

**Satisfied by:** `GET /artifacts?project=pm_image_analysis&level=0` returns all node names and types in ~5 tokens each. Level-1 adds signatures and one-liners. Full project map with no code read.

**Without cortex:** The agent reads every `__init__.py`, every module docstring, and guesses structure from filenames.

---

### US-02 — Understand a Module Contract
> As an agent asked to add a new feature to the binary classification module, I need to understand what functions exist and their signatures **before reading the implementations**, so I can plan my changes without loading code into context.

**Satisfied by:** Level-1 on the `composite_process` node for the module returns all `composes` edge targets at Level-1. Agent sees all function signatures + one-liners in one response.

---

### US-03 — Find Design Decisions That Constrain a Function
> As an agent debugging why `train_classifier` uses `calibration='isotonic'`, I need to find any design decisions that `constrains` it, so I understand the intent without reading through all docs.

**Satisfied by:** Traversing inbound `constrains` edges on the `train_classifier` node returns `D3` at Level-1 ("D3 (accepted) — SVM classifiers must use isotonic calibration"). Agent gets the answer in one graph hop, no doc reading needed.

---

### US-04 — Check Test Coverage Before Modifying a Function
> As an agent about to refactor `evaluate_classifier`, I need to know whether it has tests, what they cover, and where they are, **before making changes**.

**Satisfied by:** Inbound `validates` edges on `evaluate_classifier` list all test constraint nodes at Level-1. If none exist, the agent knows it's untested and can flag this. The `report_untested_atomic` validation rule also surfaces this at build time.

---

### US-05 — Find All Untested Functions in a Project
> As a developer, I want to generate a task in pev-agent-system for "add tests for all `atomic_process` nodes with no inbound `validates` edge" **automatically**, without manually auditing the codebase.

**Satisfied by:** `cortex validate` runs the `report_untested_atomic` rule at build time and emits a list of `atomic_process` nodes with no inbound `validates` edge. That list appears in the build output and in `index.json` — any consumer (a CI step, a task manager, a developer reviewing the output) can act on it. No file scanning needed.

---

### US-06 — Understand Data Flow Through a Pipeline
> As an agent tracing why a downstream result is wrong, I need to follow `produces` and `consumes` edges backwards from the result entity to find which process produced the bad data.

**Satisfied by:** Graph traversal on `entity` nodes via `produces`/`consumes` edges. The agent follows the data lineage without reading function bodies.

---

### US-07 — Find Documentation for a Function
> As an agent writing user-facing docs, I need to find all `document` nodes that `documents` a given function, to avoid duplicating existing content.

**Satisfied by:** Inbound `documents` edges on the function node list all doc sections that reference it. Agent sees where it's already documented before writing.

---

## Developer Workflow Stories

### US-08 — Incremental Index Update
> As a developer who just added a new `@task` function, I want to update the knowledge graph **without rebuilding the entire index**, so CI stays fast.

**Satisfied by:** `cortex build --incremental` re-scans only files modified since last build (tracked via file mtime vs `built_at` in `index.json`). The ingest endpoint upserts — unchanged artifacts are untouched.

---

### US-09 — Cross-Project Linking
> As a developer maintaining `pm_image_analysis` and `agnostic_image_analysis`, I want constraints and documents in one project to reference processes in another, since some methods are shared.

**Satisfied by:** Node IDs include the `project_id` prefix — `pm_image_analysis::methods.binary_label.run` and `agnostic_image_analysis::methods.preprocess.run` are distinct, resolvable nodes. A `constrains` edge can reference a node in another project’s index by its full prefixed ID. Any consumer that loads multiple cortex indexes can resolve cross-project edges.

---

### US-10 — Validate Annotations Before Push
> As a developer, I want `cortex build` to fail with a clear error if I have an edge with an invalid `from`/`to` type combination, **before pushing to pev-agent-system**, so I catch mistakes early.

**Satisfied by:** Ontology validation in `builder.py` — edge type rules are checked during index construction. The build fails with a list of violations, not at ingest time.

---

## Documentation Stories

### US-11 — Progressive Disclosure for Docs
> As an agent reading project documentation, I need to see document titles and section headings **first**, then summaries of each section, **then** full text only when needed — so I don't spend context loading docs I don't need.

**Satisfied by:** `document` nodes have level_0 (title), level_1 (first paragraph / section heading list), level_2 (first paragraph of each section), level_3_location (file#section anchor for full text). Sections within a doc are their own `document` nodes linked with `composes` edges.

---

### US-12 — Link a Decision to the Code It Affects
> As a team member recording a new design decision D7, I want to link it to the functions it constrains **once**, so any agent or developer can find the constraint automatically later.

**Satisfied by:** Edges in the index have a `source` field. Cortex-generated edges use `source = cortex`; manually-authored edges use `source = manual`. `cortex build` preserves manual edges — they are not overwritten on rebuild. How manual edges are created depends on the consuming tool; they can be added directly to `index.json` or maintained in a `cortex_manual_edges.yaml` config file per project.

---

## Notebook Stories

### US-13 — Register a Notebook as Documentation
> As a developer with an analysis notebook that demonstrates the classification pipeline, I want it registered as a `document` node pointing at the processes it demonstrates, so agents know it's a use-case reference.

**Satisfied by:** The doc_scanner encounters `.ipynb` files, creates a `document` node, and creates `documents` edges to any `composite_process` or `atomic_process` nodes referenced in the notebook's markdown cells or function calls. No notebook code execution or code scanning required.

---

## Anti-Stories (Things Cortex Does Not Do)

### US-N01 — Run Workflows
Cortex indexes knowledge about workflows. It does not run them. Use pm_mvp / Snakemake / Nextflow.

### US-N02 — Annotate Code
Cortex reads annotations. It does not add them. Use dFlow's `@task`/`@workflow` decorators.

### US-N03 — Render User Documentation
Cortex can emit agent-facing text renderings. Human-facing docs (HTML, PDF) are MkDocs/Sphinx's job.

### US-N04 — Scan Notebook Code for Business Logic
Notebooks are documentation. If a notebook has heavy logic inside it, that logic belongs in a module. Cortex registers the notebook, it does not extract functions from it.

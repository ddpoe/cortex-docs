# DocJSON Format Guide

## What is DocJSON

DocJSON is the source format for documentation in Cortex. Each document is a `.json` file stored in the `docs/` directory of a project. When you run `cortex_build`, all JSON files in `docs/` are scanned, indexed as `document` nodes in the graph, and linked to any code nodes they reference.

DocJSON is the *source of truth* — not the rendered markdown. The rendered output (`cortex_render_doc`) is generated on demand from the live index, so linked node summaries always reflect the current state of the code.

## File Location and Naming

Place JSON files in the `docs/` directory at the project root:

```
my_project/
  docs/
    architecture.json
    mcp-tools.json
    data-model.json
  .cortex/
    cortex.db
```

The filename stem (`architecture`, `mcp-tools`) becomes part of the node ID: `{project_id}::docs.{stem}`. Use lowercase with hyphens — no spaces or underscores.

## Top-Level Structure

Every DocJSON file must have three required keys:

```json
{
  "id": "my-document",
  "title": "My Document Title",
  "sections": [ ... ]
}
```

And two optional keys:

```json
{
  "tags": ["optional", "list", "of", "tags"]
}
```

| Key | Required | Description |
|---|---|---|
| `id` | yes | Short identifier, matches the filename stem. Lowercase, hyphens only. |
| `title` | yes | Human-readable title. Becomes the H1 in rendered output. |
| `sections` | yes | Array of section objects (see below). Can be empty `[]`. |
| `tags` | no | String array. Stored in the index for filtering via `cortex_list`. |

## Section Structure

Each entry in `sections` is an object with two required keys and three optional keys:

```json
{
  "id": "my-section",
  "heading": "My Section Heading",
  "content": "Prose text for this section.",
  "level": 2,
  "tags": ["optional"],
  "links": [
    {"node_id": "myproject::mymodule::my_function"}
  ]
}
```

| Key | Required | Description |
|---|---|---|
| `id` | yes | Slug for this section. Combined with the doc id: `{project}::docs.{doc_id}::{section_id}`. Lowercase, hyphens only. |
| `heading` | yes | Section heading text. Rendered as `##` by default. |
| `content` | no | Prose content. Supports markdown formatting. Stored in the index as `level_2`. |
| `level` | no | Heading level 2–6 (default 2). Controls the `##` depth in rendered output. |
| `tags` | no | String array. Tags on the section node. |
| `links` | no | Array of `{"node_id": "..."}` objects linking this section to code nodes. |

## How Node IDs Work

Node IDs follow the pattern `{project_id}::{dotpath}` where `project_id` defaults to the project root directory name.

Examples for a project named `myproject`:

| What | Node ID |
|---|---|
| Module `myproject/core/db.py` | `myproject::myproject.core.db` |
| Function `upsert_node` in that module | `myproject::myproject.core.db::upsert_node` |
| Markdown doc `docs/architecture.md` | `myproject::docs.architecture` |
| H2 section "Overview" in that doc | `myproject::docs.architecture#overview` |
| JSON doc `docs/design.json` | `myproject::docs.design` |
| Section `database-layer` in that doc | `myproject::docs.design::database-layer` |

To find node IDs for your project, use `cortex_list` or `cortex_search`. If you link to a node ID that doesn't exist in the index, `cortex_build` won't fail — it will register the edge against whatever node is scanned later.

## Linking Sections to Code

Links connect a doc section to code nodes via a `documents` edge in the graph. This is how the rendered output knows what to show under **Linked nodes** and how `cortex_check` knows which sections to mark as `DESC_STALE` when the linked code changes.

Add links in two ways:

**1. In the JSON directly:**
```json
"links": [
  {"node_id": "cortex::cortex.index.db::upsert_node"},
  {"node_id": "cortex::cortex.index.db::upsert_edge"}
]
```

**2. Using `cortex_add_link` after the fact:**
```
cortex_add_link(
  project_root = "/path/to/project",
  section_id   = "cortex::docs.my-doc::my-section",
  node_id      = "cortex::cortex.index.db::upsert_node"
)
```
This mutates the JSON file and re-indexes automatically.

## Content Formatting

The `content` field is plain text that supports markdown. Common patterns:

**Inline code:** wrap with backticks — `` `my_function(x)` ``

**Code blocks:** use triple backtick fences:
```
"content": "Example:\n\n```python\ndef foo():\n    pass\n```"
```

**Tables:** standard markdown table syntax works in content.

**Escaping:** because `content` is a JSON string, newlines are `\n` and backslashes are `\\`.

**Length:** no hard limit, but keep `level_2` (the content stored in the index) meaningful for search. The first sentence is used as the node summary in `cortex_list` output.

## Full Example

A complete minimal document:

```json
{
  "id": "data-model",
  "title": "Data Model",
  "tags": ["database", "schema"],
  "sections": [
    {
      "id": "nodes-table",
      "heading": "Nodes Table",
      "content": "Every indexed entity is stored as a row in the `nodes` table. Each node has a unique `id`, a `node_type`, and up to four `level_*` text fields for progressive disclosure.",
      "links": [
        {"node_id": "myproject::myproject.index.db"}
      ]
    },
    {
      "id": "edges-table",
      "heading": "Edges Table",
      "content": "Relationships between nodes are stored in the `edges` table. Each edge has a `from_id`, `to_id`, and `edge_type`. Valid edge types are defined in the ontology YAML."
    }
  ]
}
```

After saving to `docs/data-model.json`, run `cortex_build` and then `cortex_render_doc` with `doc_id="{project}::docs.data-model"` to see the rendered output.

## Authoring Workflow

1. Create a new `.json` file in `docs/` following the format above
2. Run `cortex_build` to index it
3. Use `cortex_render_doc` to see the rendered output and check linked node summaries are correct
4. Use `cortex_list_undocumented` to find code nodes not yet covered by any doc section
5. Add `links` to connect sections to those nodes, then rebuild
6. Use `cortex_check` to monitor whether linked sections go stale as code evolves

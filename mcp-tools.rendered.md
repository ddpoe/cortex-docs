# Cortex MCP Tools

## Overview

Cortex exposes its index through a set of MCP tools. Agents connect via stdio transport and call these tools to build, query, and document a project's code index. All tools take `project_root` as their first argument and return plain text. Errors are returned as `ERROR: {message}` — tools never raise exceptions to the caller.

The tools fall into four groups: **index management**, **querying**, **staleness tracking**, and **documentation**.

## Index Management

`cortex_build(project_root)` — Scans all Python files and docs under the project root and upserts changed nodes and edges into the SQLite index. Run this after any code change. Returns a summary of nodes written/skipped and any ontology warnings.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_build` — cortex_build(project_root) — Scan a project directory and build (or refresh) the Cortex index.

## Querying: Render

`cortex_render(project_root, level, node_id=None)` — Renders one or all nodes at a chosen detail level.

- Level 0: IDs only — use for a quick inventory
- Level 1: ID + one-line summary — the default for orientation
- Level 2: full detail including level_2 documentation body
- Level 3: step markers only (dFlow-annotated workflows)

Pass a `node_id` to render a single node.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_render` — cortex_render(project_root, level, node_id=None) — Render nodes from the index at a given detail level.

## Querying: List and Graph

`cortex_list(project_root, node_type, tag, parent_id)` — Lists nodes filtered by type and/or tag. Pass `parent_id` to enumerate all functions inside a module via one-hop `composes` edges — faster than `cortex_graph` for simple child enumeration.

`cortex_graph(project_root, node_id, direction, depth)` — Traverses edges from a node. `direction` is `"out"`, `"in"`, or `"both"`. Use `depth > 1` to walk multi-hop relationships.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_graph` — cortex_graph(project_root, node_id, direction='out', depth=1) — Traverse and render the edge graph for a node.
- `cortex::cortex.mcp_server::cortex_list` — cortex_list(project_root, node_type=None, tag=None, parent_id=None) — List nodes, optionally filtered by node_type and/or tag.

## Querying: Search

`cortex_search(project_root, query, level)` — Full-text search over `level_1` and `level_2` fields. Uses FTS5 with ranked matching. Falls back to LIKE-AND then LIKE-OR if FTS returns nothing. The response header indicates which mode fired so agents can assess result confidence.

Set `level=1` to search summaries only, `level=2` to search documentation bodies only.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_search` — cortex_search(project_root, query, level=None) — Full-text search over node level_1 and level_2 fields.

## Staleness: Check and History

`cortex_check(project_root)` — Reports staleness status for every node: `CLEAN`, `DESC_STALE`, `STRUCTURAL_DRIFT`, or `UNVERIFIED`. Also appends a DOC SECTIONS block showing whether linked documentation sections are stale relative to their linked code nodes.

`cortex_history(project_root, node_id, limit)` — Shows the change history for a single node newest-first. Annotates the open stale window (code changed, description not updated) with the number of days it has been open.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_check` — cortex_check(project_root) — Report per-node staleness / confidence status.
- `cortex::cortex.mcp_server::cortex_history` — cortex_history(project_root, node_id, limit=10) — Show the change history for a single node.

## Staleness: Mark Clean

`cortex_mark_clean(project_root, node_id, reason)` — Records an `AGENT_VERIFIED` history row on a `DESC_STALE` node. Use only after reading both the current code and documentation and confirming they are consistent. The reason string is stored and surfaced in `cortex_history` for human review before push.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_mark_clean` — cortex_mark_clean(project_root, node_id, reason) — Mark a DESC_STALE node as agent-verified clean.

## Documentation: Write and Link

`cortex_write_doc(project_root, doc_json)` — Accepts a JSON string describing a documentation document, writes it to `docs/{id}.json`, and immediately indexes it. Validates that all linked `node_id` values exist; unknown IDs are returned as warnings (the write still succeeds).

`cortex_add_link(project_root, section_id, node_id)` — Appends a link from an existing doc section to a code node. Loads the JSON file, adds the link, writes it back, and re-indexes. Section IDs follow the pattern `{project_id}::docs.{doc_id}::{section_id}`.

`cortex_list_undocumented(project_root, node_type)` — Lists all nodes with no inbound `documents` edge. Use this to find coverage gaps.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_add_link` — cortex_add_link(project_root, section_id, node_id) — Add a link from a doc section to a code node.
- `cortex::cortex.mcp_server::cortex_list_undocumented` — cortex_list_undocumented(project_root, node_type=None) — List all nodes that have no inbound 'documents' edge.
- `cortex::cortex.mcp_server::cortex_write_doc` — cortex_write_doc(project_root, doc_json) — Write a DocJSON documentation file and register it in the index.

## Documentation: Render

`cortex_render_doc(project_root, doc_id)` — Renders a DocJSON document as Markdown, with linked node summaries listed under each section. Pass `doc_id="list"` to see all indexed document IDs.

**Linked nodes:**
- `cortex::cortex.mcp_server::cortex_render_doc` — cortex_render_doc(project_root, doc_id) — Render a DocJSON document as Markdown.

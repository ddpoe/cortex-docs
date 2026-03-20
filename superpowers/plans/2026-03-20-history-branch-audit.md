# History Branch Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Audit the `feature/history` branch — add missing ADR-011 tests, update stale docs, resolve all CONTENT_STALE nodes, and checkpoint.

**Architecture:** Six sequential tasks: rebuild index, write ADR-011 tests, update ADR-011 doc status, triage and fix stale doc nodes, stage the untracked test file, and run final verification + checkpoint.

**Tech Stack:** Python 3, pytest, SQLite3 (WAL mode), cortex MCP tools, poetry

---

### Task 1: Rebuild Index and Baseline

**Files:**
- None created/modified — operational commands only

- [ ] **Step 1: Rebuild the cortex index**

Run: `poetry run cortex build .`

This re-indexes all changed code files. Most code-node CONTENT_STALE should clear because the index will pick up the current file hashes.

- [ ] **Step 2: Check staleness after rebuild**

Run: `poetry run cortex check --verbose .`

Record which nodes are still CONTENT_STALE. Expected: code nodes mostly clear, doc nodes remain stale.

- [ ] **Step 3: Run full test suite to confirm green baseline**

Run: `poetry run python -m pytest tests/ -x -q`

Expected: 207 passed (same as current). If anything fails, stop and fix before proceeding.

---

### Task 2: Write ADR-011 Tests (WAL + Timeout + Structured Errors)

**Files:**
- Create: `tests/test_adr011_sqlite_resilience.py`
- Reference: `cortex/index/db.py:164-177` (`_connect`)
- Reference: `cortex/mcp_server.py` (MCP tool error handling)

These tests cover ADR-011 Steps 1-2 (WAL mode + timeout, structured error reporting). Step 3 (file locking) is not yet implemented, so no tests for that.

- [ ] **Step 1: Write the complete test file**

Write `tests/test_adr011_sqlite_resilience.py` as a single file. The helper `_make_node` is defined first because later tests depend on it.

```python
"""Tests for ADR-011: SQLite Concurrency Resilience.

Covers:
- WAL mode enabled on every connection
- timeout=5 so concurrent writers wait instead of failing
- foreign_keys=ON
- Rollback on exception
- MCP tools raise exceptions (not return error strings) on unexpected failures
- MCP input validation still returns ERROR strings (not exceptions)
"""

from __future__ import annotations

import sqlite3
import threading
import time
from pathlib import Path

import pytest

from cortex.index import db
from cortex.models import CortexNode
from cortex import mcp_server


# ---------------------------------------------------------------------------
# Helper
# ---------------------------------------------------------------------------

def _make_node(node_id: str, code_hash: str = "abc123") -> CortexNode:
    """Create a minimal CortexNode for testing."""
    parts = node_id.rsplit("::", 1)
    return CortexNode(
        id=node_id,
        title=parts[-1],
        node_type="atomic_process",
        location="test_file.py",
        source="ast",
        code_hash=code_hash,
        desc_hash="desc_hash_placeholder",
        level_0=node_id,
        level_1=f"{parts[-1]} — test node",
    )


# ---------------------------------------------------------------------------
# Connection layer tests
# ---------------------------------------------------------------------------

def test_connect_enables_wal_mode(db_path: Path) -> None:
    """_connect() sets journal_mode=WAL on every connection."""
    with db._connect(db_path) as conn:
        row = conn.execute("PRAGMA journal_mode").fetchone()
        assert row[0] == "wal"


def test_connect_uses_timeout(db_path: Path) -> None:
    """_connect() passes timeout=5 so a second writer waits instead of failing immediately."""
    # Hold an exclusive lock via a raw connection
    blocker = sqlite3.connect(str(db_path), timeout=0)
    blocker.execute("PRAGMA journal_mode=WAL")
    blocker.execute("BEGIN EXCLUSIVE")

    # Release the lock after a short delay from a thread.
    def release():
        time.sleep(0.3)
        blocker.rollback()
        blocker.close()

    threading.Thread(target=release, daemon=True).start()

    # _connect() should succeed (waits for lock release) not raise OperationalError
    with db._connect(db_path) as conn:
        conn.execute("SELECT count(*) FROM nodes")


def test_connect_enables_foreign_keys(db_path: Path) -> None:
    """_connect() sets PRAGMA foreign_keys=ON."""
    with db._connect(db_path) as conn:
        row = conn.execute("PRAGMA foreign_keys").fetchone()
        assert row[0] == 1


def test_connect_rollback_on_exception(db_path: Path) -> None:
    """_connect() rolls back the transaction when an exception is raised."""
    node = _make_node("pkg::mod::func_rollback", code_hash="aaa")
    db.upsert_node(db_path, node)

    with pytest.raises(RuntimeError):
        with db._connect(db_path) as conn:
            conn.execute(
                "UPDATE nodes SET code_hash = ? WHERE id = ?",
                ("zzz", "pkg::mod::func_rollback"),
            )
            raise RuntimeError("force rollback")

    # Verify the update was rolled back
    with db._connect(db_path) as conn:
        row = conn.execute(
            "SELECT code_hash FROM nodes WHERE id = ?",
            ("pkg::mod::func_rollback",),
        ).fetchone()
        assert row["code_hash"] == "aaa"


# ---------------------------------------------------------------------------
# MCP structured error tests
# ---------------------------------------------------------------------------

def test_mcp_tool_raises_on_unexpected_error(tmp_path: Path) -> None:
    """MCP tools re-raise unexpected exceptions (ADR-011 Step 2).

    Before ADR-011, tools returned 'ERROR: ...' strings. Now they raise,
    so FastMCP can return structured error responses to agents.
    """
    # cortex_build with a non-existent project should raise (no .cortex dir)
    fake_root = str(tmp_path / "nonexistent")
    with pytest.raises((FileNotFoundError, OSError)):
        mcp_server.cortex_build(fake_root)


def test_mcp_tool_input_validation_returns_error_string(mini_project: Path) -> None:
    """Input validation errors (bad node_id) still return ERROR strings.

    These are expected user errors, not unexpected exceptions.
    """
    result = mcp_server.cortex_render(str(mini_project), level=1, node_id="nonexistent::node")
    assert result.startswith("ERROR:")
```

- [ ] **Step 2: Run all ADR-011 tests**

Run: `poetry run python -m pytest tests/test_adr011_sqlite_resilience.py -v`

Expected: 6 tests pass.

- [ ] **Step 3: Run full test suite to confirm no regressions**

Run: `poetry run python -m pytest tests/ -x -q`

Expected: 213+ passed (previous 207 + 6 new tests).

- [ ] **Step 4: Commit**

```bash
git add tests/test_adr011_sqlite_resilience.py
git commit -m "test: add ADR-011 SQLite resilience tests (WAL, timeout, error handling)"
```

---

### Task 3: Update ADR-011 Status

**Files:**
- Modify: `docs/` submodule — via `cortex_update_section`

- [ ] **Step 1: Update ADR-011 implementation-order section**

Use `cortex_update_section` to update `cortex::docs.adrs.011-sqlite-concurrency-resilience::implementation-order` — mark Steps 1-2 as "Done", Steps 3-4 as "Deferred":

Content should note:
- Step 1 (WAL + timeout): **Done** — applied in `_connect()` and all direct `sqlite3.connect()` sites
- Step 2 (Structured MCP errors): **Done** — 19 `except→raise` conversions
- Step 3 (File locking): **Deferred** — read-modify-write race on JSON files not yet addressed
- Step 4 (Tests): **Partial** — connection-layer tests added, concurrency integration tests deferred with Step 3

- [ ] **Step 2: Fix stale module docstring in `cortex/mcp_server.py`**

The module docstring at line 24-25 says "Errors are caught and returned as 'ERROR: {message}' — tools never raise." This directly contradicts the ADR-011 `except Exception: raise` pattern. Update to:

```
All tools return plain text. Unexpected errors raise exceptions so FastMCP
returns structured error responses. Input validation errors return "ERROR: ..."
strings for user-facing messages.
```

- [ ] **Step 3: Rebuild index after doc update**

Run: `poetry run cortex build .`

---

### Task 4: Triage and Fix Stale Doc Nodes

**Files:**
- Modify: various doc sections via `cortex_update_section`

This task handles the remaining CONTENT_STALE doc nodes. The approach:
1. Run `cortex check --verbose` to get the current stale list
2. For each stale doc section, check if the prose still matches the code
3. If prose is correct (just hash mismatch from non-semantic changes): `cortex build` will clear it
4. If prose needs updating: use `cortex_update_section`

- [ ] **Step 1: Get current stale list after Task 1-3 rebuilds**

Use `cortex_check(verbose=True)` to see what's still stale.

- [ ] **Step 2: Categorize stale nodes**

Separate into:
- **Code nodes** still stale → need investigation (code changed after last build?)
- **Doc nodes with accurate prose** → just need re-index
- **Doc nodes needing prose updates** → update via `cortex_update_section`

- [ ] **Step 3: Update stale doc sections as needed**

For each doc section where the prose no longer matches the code:
- Read the doc section via `cortex_read_doc`
- Check the linked code via `cortex_source`
- Update the section content via `cortex_update_section`

Priority order (from audit protocol):
1. Interface specs (MCP tools, viz API, data model)
2. Design specs (history, staleness)
3. PRD capability tables
4. Templates and plans

- [ ] **Step 4: Rebuild and verify**

Run: `poetry run cortex build .`
Then: `cortex_check(verbose=True)` — target zero CONTENT_STALE.

---

### Task 5: Stage Untracked Test File

**Files:**
- Stage: `tests/test_e2e_git_lifecycle.py` (currently untracked, 803 lines)

- [ ] **Step 1: Verify the file is correct**

Run: `poetry run python -m pytest tests/test_e2e_git_lifecycle.py -v`

Expected: All tests in this file pass.

- [ ] **Step 2: Stage the file**

```bash
git add tests/test_e2e_git_lifecycle.py
```

---

### Task 6: Final Verification and Checkpoint

- [ ] **Step 1: Run full test suite**

Run: `poetry run python -m pytest tests/ -x -q`

Expected: All tests pass (207 + new ADR-011 tests).

- [ ] **Step 2: Final staleness check**

Use `cortex_check(verbose=True)` — target zero CONTENT_STALE / STRUCTURAL_DRIFT.

- [ ] **Step 3: Create audit checkpoint**

Run: `poetry run cortex history checkpoint . --message "history-branch-audit-complete"`

- [ ] **Step 4: Final commit with all changes**

Stage all remaining changes and commit. The pre-commit hook handles the docs submodule automatically.

```bash
git add cortex/mcp_server.py tests/test_e2e_git_lifecycle.py
git commit -m "audit: history branch audit — ADR-011 tests, doc updates, stale node resolution"
```

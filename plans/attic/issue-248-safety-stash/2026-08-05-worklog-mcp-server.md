# Worklog MCP Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #157 — Worklog REST + MCP server — expose work history as API
**Issue group:** #157

**Goal:** Expose worklog.db as 4 read-only MCP tools for Claude Code agents.

**Architecture:** Python FastMCP server (stdio transport) wrapping existing
`worklog.py` query functions. Follows `garden_mcp_server.py` pattern.
One production file, one test file.

**Tech Stack:** Python 3.14, FastMCP (`mcp.server.fastmcp`), SQLite (via
`worklog.py`), pytest

## Global Constraints

- Read-only — no write operations exposed via MCP
- `worklog.py` is the sole schema owner — server imports it directly
- Tests use temp SQLite databases via `monkeypatch` — never touch real worklog.db
- `WORKLOG_DB` env var read per call (not cached at import) for test isolation

---

### Task 1: Server and tests (TDD)

**Files:**
- Create: `scripts/worklog_mcp_server.py`
- Create: `tests/test_worklog_mcp_server.py`

**Interfaces:**
- Consumes: `worklog.connect(db_path)`, `worklog.active_work(conn)`,
  `worklog.event_log(conn, since, event_type, repo_path, limit)`,
  `worklog.work_item_timeline(conn, branch, repo_path)`,
  `worklog.slot_status(conn, family_root)`,
  `worklog.ensure_repo(conn, path)`,
  `worklog.record_work_start(conn, branch, repo_path, issue_number, issue_repo)`,
  `worklog.record_work_end(conn, branch, repo_path)`,
  `worklog.record_slot_create(conn, slot_number, family_root, repos, branch, issue_number, issue_repo)`
- Produces: 4 MCP tools (`worklog_active`, `worklog_events`,
  `worklog_timeline`, `worklog_slots`) callable as Python functions
  and via MCP stdio transport

- [ ] **Step 1: Write test file with all 15 test cases**

```python
"""Tests for worklog_mcp_server — 4 MCP tools over worklog.db."""

import json
import sys
from pathlib import Path
from unittest.mock import patch

import pytest

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "scripts"))
import worklog
import worklog_mcp_server


@pytest.fixture
def db(tmp_path, monkeypatch):
    """Create a temp worklog database and point the server at it."""
    db_path = str(tmp_path / "test-worklog.db")
    monkeypatch.setenv("WORKLOG_DB", db_path)
    conn = worklog.connect(db_path)
    worklog.ensure_repo(conn, "/repo/project", github_repo="Org/repo")
    yield conn
    conn.close()


def _start(db):
    """Record a work-start and return the connection."""
    worklog.record_work_start(
        db, "issue-42-fix", "/repo/project",
        issue_number=42, issue_repo="Org/repo",
    )


class TestWorklogActive:
    def test_returns_empty_on_fresh_db(self, db):
        result = worklog_mcp_server.worklog_active()
        assert result == []

    def test_returns_started_items(self, db):
        _start(db)
        result = worklog_mcp_server.worklog_active()
        assert len(result) == 1
        assert result[0]["branch"] == "issue-42-fix"
        assert result[0]["state"] == "active"

    def test_excludes_ended_items(self, db):
        _start(db)
        worklog.record_work_end(db, "issue-42-fix", "/repo/project")
        result = worklog_mcp_server.worklog_active()
        assert result == []

    def test_returns_error_dict_on_db_failure(self, monkeypatch):
        monkeypatch.setenv("WORKLOG_DB", "/nonexistent/path/db.sqlite")
        with patch("worklog.connect", side_effect=Exception("boom")):
            result = worklog_mcp_server.worklog_active()
        assert isinstance(result, dict)
        assert "error" in result


class TestWorklogEvents:
    def test_returns_all_when_unfiltered(self, db):
        _start(db)
        result = worklog_mcp_server.worklog_events()
        assert len(result) >= 1
        assert result[0]["event_type"] == "work-start"

    def test_filters_by_type(self, db):
        _start(db)
        worklog.record_work_end(db, "issue-42-fix", "/repo/project")
        result = worklog_mcp_server.worklog_events(event_type="work-end")
        assert all(e["event_type"] == "work-end" for e in result)

    def test_filters_by_since(self, db):
        _start(db)
        result = worklog_mcp_server.worklog_events(since="2099-01-01T00:00:00")
        assert result == []

    def test_respects_limit(self, db):
        _start(db)
        worklog.record_work_end(db, "issue-42-fix", "/repo/project")
        result = worklog_mcp_server.worklog_events(limit=1)
        assert len(result) == 1

    def test_normalizes_repo_path(self, db):
        _start(db)
        result = worklog_mcp_server.worklog_events(repo_path="/repo/project")
        assert len(result) >= 1

    def test_metadata_is_parsed_dict(self, db):
        _start(db)
        worklog.record_issue_activate(
            db, "issue-42-fix", "/repo/project", 42, "Org/repo",
        )
        result = worklog_mcp_server.worklog_events(event_type="issue-activate")
        assert len(result) >= 1
        meta = result[0]["metadata"]
        assert isinstance(meta, dict)
        assert meta["issue_number"] == 42


class TestWorklogTimeline:
    def test_returns_branch_events(self, db):
        _start(db)
        result = worklog_mcp_server.worklog_timeline(
            branch="issue-42-fix", repo_path="/repo/project",
        )
        assert len(result) >= 1
        assert result[0]["event_type"] == "work-start"

    def test_empty_for_unknown_branch(self, db):
        result = worklog_mcp_server.worklog_timeline(
            branch="nonexistent", repo_path="/repo/project",
        )
        assert result == []

    def test_metadata_is_parsed_dict(self, db):
        _start(db)
        worklog.record_issue_activate(
            db, "issue-42-fix", "/repo/project", 42, "Org/repo",
        )
        result = worklog_mcp_server.worklog_timeline(
            branch="issue-42-fix", repo_path="/repo/project",
        )
        activate_events = [e for e in result if e["event_type"] == "issue-activate"]
        assert len(activate_events) >= 1
        assert isinstance(activate_events[0]["metadata"], dict)


class TestWorklogSlots:
    def test_returns_all(self, db):
        worklog.record_slot_create(
            db, slot_number=1, family_root="/family",
            repos=["/family/repo"], branch="issue-50-epic",
            issue_number=50, issue_repo="Org/repo",
        )
        result = worklog_mcp_server.worklog_slots()
        assert len(result) == 1
        assert result[0]["slot_number"] == 1

    def test_filters_by_family_root(self, db):
        worklog.record_slot_create(
            db, slot_number=1, family_root="/family-a",
            repos=["/family-a/repo"], branch="issue-50-a",
            issue_number=50, issue_repo="Org/repo",
        )
        worklog.record_slot_create(
            db, slot_number=2, family_root="/family-b",
            repos=["/family-b/repo"], branch="issue-51-b",
            issue_number=51, issue_repo="Org/repo",
        )
        result = worklog_mcp_server.worklog_slots(family_root="/family-a")
        assert len(result) == 1
        assert result[0]["family_root"] == str(Path("/family-a").resolve())
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog_mcp_server.py -v 2>&1 | head -30`
Expected: ImportError or ModuleNotFoundError for `worklog_mcp_server`

- [ ] **Step 3: Write the server implementation**

Create `scripts/worklog_mcp_server.py` with the implementation from the
spec. One change from the spec: `_connect()` reads `WORKLOG_DB` per call
(not cached at module level) so `monkeypatch.setenv` works in tests:

```python
#!/usr/bin/env python3
"""
worklog_mcp_server.py — Hortora Worklog MCP Server.

Exposes 4 read-only tools over ~/.hortora/worklog.db via FastMCP:
  worklog_active   — active work items
  worklog_events   — filtered event log
  worklog_timeline — all events for a branch
  worklog_slots    — slot status

Usage (stdio transport):
  python3 worklog_mcp_server.py

Configure in Claude Code settings:
  {
    "mcpServers": {
      "hortora-worklog": {
        "command": "python3",
        "args": ["/path/to/soredium/scripts/worklog_mcp_server.py"]
      }
    }
  }
"""

import json
import os
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))

from mcp.server.fastmcp import FastMCP
import worklog

mcp = FastMCP("Hortora Worklog")


def _connect():
    db_path = os.environ.get('WORKLOG_DB')
    return worklog.connect(db_path) if db_path else worklog.connect()


def _parse_metadata(rows: list[dict]) -> list[dict]:
    for row in rows:
        raw = row.get("metadata")
        if isinstance(raw, str):
            try:
                row["metadata"] = json.loads(raw)
            except (json.JSONDecodeError, TypeError):
                pass
    return rows


def _norm_path(p: str | None) -> str | None:
    return str(Path(p).resolve()) if p else None


@mcp.tool()
def worklog_active() -> list | dict:
    """Return all active (non-ended) work items."""
    try:
        conn = _connect()
        try:
            return worklog.active_work(conn)
        finally:
            conn.close()
    except Exception as e:
        return {"error": type(e).__name__, "message": str(e)}


@mcp.tool()
def worklog_events(
    since: str = None,
    event_type: str = None,
    repo_path: str = None,
    limit: int = 100,
) -> list | dict:
    """Return filtered event log, newest first."""
    try:
        conn = _connect()
        try:
            results = worklog.event_log(conn, since=since,
                                         event_type=event_type,
                                         repo_path=_norm_path(repo_path),
                                         limit=limit)
            return _parse_metadata(results)
        finally:
            conn.close()
    except Exception as e:
        return {"error": type(e).__name__, "message": str(e)}


@mcp.tool()
def worklog_timeline(branch: str, repo_path: str) -> list | dict:
    """Return all events for a branch, oldest first."""
    try:
        conn = _connect()
        try:
            results = worklog.work_item_timeline(conn, branch, repo_path)
            return _parse_metadata(results)
        finally:
            conn.close()
    except Exception as e:
        return {"error": type(e).__name__, "message": str(e)}


@mcp.tool()
def worklog_slots(family_root: str = None) -> list | dict:
    """Return slot status, optionally filtered by family root."""
    try:
        conn = _connect()
        try:
            return worklog.slot_status(conn, family_root=family_root)
        finally:
            conn.close()
    except Exception as e:
        return {"error": type(e).__name__, "message": str(e)}


if __name__ == '__main__':
    mcp.run()
```

- [ ] **Step 4: Run tests to verify all pass**

Run: `python3 -m pytest tests/test_worklog_mcp_server.py -v`
Expected: 15 passed

- [ ] **Step 5: Commit**

```bash
git add scripts/worklog_mcp_server.py tests/test_worklog_mcp_server.py
git commit -m "feat(#157): add worklog MCP server with 4 read-only tools

FastMCP server (stdio) wrapping worklog.py query functions:
worklog_active, worklog_events, worklog_timeline, worklog_slots.

Includes repo_path normalization, metadata JSON parsing,
structured error handling. 15 tests.

Closes #157"
```

---

### Task 2: MCP configuration

**Files:**
- Modify: `.claude/settings.local.json`

- [ ] **Step 1: Add MCP server entry**

Add `hortora-worklog` to `mcpServers` in the project's local settings:

```json
{
  "mcpServers": {
    "hortora-worklog": {
      "command": "python3",
      "args": ["scripts/worklog_mcp_server.py"]
    }
  }
}
```

- [ ] **Step 2: Verify server starts**

Run: `python3 scripts/worklog_mcp_server.py` (should start and wait for
stdio input; Ctrl-C to exit). If it errors on import, fix the import path.

- [ ] **Step 3: Commit**

```bash
git add .claude/settings.local.json
git commit -m "chore(#157): configure worklog MCP server in project settings

Refs #157"
```

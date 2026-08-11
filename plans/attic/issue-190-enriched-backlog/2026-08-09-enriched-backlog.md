# Enriched Backlog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #190 — Local enriched backlog
**Issue group:** #191, #192, #193

**Goal:** Add enrichment schema, GitHub cache, trajectory capture, and
what-next recommendations to the soredium worklog DB.

**Architecture:** New `enrichment.py` module imports `worklog.connect()`
for shared DB access. Schema migration (v1→v2) lives in `worklog.py`.
Three new tables: `issue_enrichment`, `trajectory_notes`,
`github_issue_cache`. CLI interface for skill invocation. Skills call
Python scripts; Python scripts own all persistence logic.

**Tech Stack:** Python 3, SQLite (WAL mode), pytest, subprocess (gh CLI)

## Global Constraints

- All enum fields validated on write: `strategic_role`, `readiness`,
  `decay`, `blast_radius` have fixed vocabularies (see spec)
- Labels stored as JSON array of name strings
- All timestamps ISO 8601 UTC
- CLI output is always JSON, exit 0 on success, 1 on error
- Cache staleness is never fatal — failures preserve existing cache
- Protocol: every new .py script ships with tests (externalised-scripts-require-tests)

---

### Task 1: v2 schema migration

**Refs:** #191

**Files:**
- Modify: `scripts/worklog.py`
- Test: `tests/test_worklog.py`

**Interfaces:**
- Consumes: existing `worklog.connect()`, `_migrate()`
- Produces: three new tables available via any `worklog.connect()` call

- [ ] **Step 1: Write the failing test**

Add to `tests/test_worklog.py`:

```python
class TestV2Migration:
    def test_v2_tables_created(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        tables = [r[0] for r in conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        ).fetchall()]
        assert "issue_enrichment" in tables
        assert "trajectory_notes" in tables
        assert "github_issue_cache" in tables
        conn.close()

    def test_v2_schema_version(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        ver = conn.execute("PRAGMA user_version").fetchone()[0]
        assert ver == 2
        conn.close()

    def test_v1_to_v2_upgrade(self, tmp_path):
        db = tmp_path / "worklog.db"
        # Create a v1 database manually
        conn = sqlite3.connect(str(db))
        conn.executescript(worklog.SCHEMA_V1)
        conn.execute("PRAGMA user_version = 1")
        conn.commit()
        # Insert some v1 data
        conn.execute(
            "INSERT INTO repos (path) VALUES (?)", ("/test/repo",)
        )
        conn.commit()
        conn.close()
        # Now connect via worklog — should auto-upgrade to v2
        conn = worklog.connect(str(db))
        ver = conn.execute("PRAGMA user_version").fetchone()[0]
        assert ver == 2
        tables = [r[0] for r in conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        ).fetchall()]
        assert "issue_enrichment" in tables
        assert "trajectory_notes" in tables
        assert "github_issue_cache" in tables
        # v1 data preserved
        row = conn.execute("SELECT path FROM repos").fetchone()
        assert row[0] == "/test/repo"
        conn.close()

    def test_v2_indexes_created(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        indexes = [r[0] for r in conn.execute(
            "SELECT name FROM sqlite_master WHERE type='index' AND name LIKE 'idx_%' ORDER BY name"
        ).fetchall()]
        assert "idx_cache_repo" in indexes
        assert "idx_cache_staleness" in indexes
        assert "idx_enrichment_role" in indexes
        assert "idx_enrichment_decay" in indexes
        assert "idx_enrichment_readiness" in indexes
        assert "idx_trajectory_issue" in indexes
        conn.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::TestV2Migration -v`
Expected: FAIL — tables don't exist yet

- [ ] **Step 3: Write minimal implementation**

In `scripts/worklog.py`, add the v2 DDL constant after `SCHEMA_V1`:

```python
SCHEMA_V2 = """
CREATE TABLE IF NOT EXISTS issue_enrichment (
    issue_number   INTEGER NOT NULL,
    issue_repo     TEXT NOT NULL,
    strategic_role TEXT,
    readiness      TEXT,
    decay          TEXT,
    blast_radius   TEXT,
    cohesion       TEXT,
    updated_at     TEXT NOT NULL,
    PRIMARY KEY (issue_number, issue_repo)
);

CREATE TABLE IF NOT EXISTS trajectory_notes (
    id           INTEGER PRIMARY KEY,
    issue_number INTEGER NOT NULL,
    issue_repo   TEXT NOT NULL,
    note         TEXT NOT NULL,
    source_branch TEXT,
    created_at   TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_trajectory_issue
    ON trajectory_notes(issue_number, issue_repo);

CREATE TABLE IF NOT EXISTS github_issue_cache (
    issue_number INTEGER NOT NULL,
    issue_repo   TEXT NOT NULL,
    title        TEXT,
    state        TEXT,
    labels       TEXT,
    body         TEXT,
    cached_at    TEXT NOT NULL,
    PRIMARY KEY (issue_number, issue_repo)
);

CREATE INDEX IF NOT EXISTS idx_cache_repo ON github_issue_cache(issue_repo);
CREATE INDEX IF NOT EXISTS idx_cache_staleness ON github_issue_cache(issue_repo, cached_at);
CREATE INDEX IF NOT EXISTS idx_enrichment_role ON issue_enrichment(strategic_role);
CREATE INDEX IF NOT EXISTS idx_enrichment_decay ON issue_enrichment(decay);
CREATE INDEX IF NOT EXISTS idx_enrichment_readiness ON issue_enrichment(readiness);
"""
```

Update `SCHEMA_VERSION`:

```python
SCHEMA_VERSION = 2
```

Update `_migrate()`:

```python
def _migrate(conn: sqlite3.Connection) -> None:
    current = conn.execute("PRAGMA user_version").fetchone()[0]
    if current < 1:
        conn.executescript(SCHEMA_V1)
    if current < 2:
        conn.executescript(SCHEMA_V2)
    if current < SCHEMA_VERSION:
        conn.execute(f"PRAGMA user_version = {SCHEMA_VERSION}")
        conn.commit()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: ALL PASS (including existing v1 tests — `TestConnect.test_schema_version_set` needs updating to expect version 2)

Update the existing test:

```python
def test_schema_version_set(self, tmp_path):
    db = tmp_path / "worklog.db"
    conn = worklog.connect(str(db))
    ver = conn.execute("PRAGMA user_version").fetchone()[0]
    assert ver == 2  # was 1
    conn.close()
```

- [ ] **Step 5: Commit**

```bash
git add scripts/worklog.py tests/test_worklog.py
git commit -m "feat(#191): v2 schema migration — enrichment, trajectory_notes, github_issue_cache tables"
```

---

### Task 2: enrichment CRUD + trajectory notes

**Refs:** #191

**Files:**
- Create: `scripts/enrichment.py`
- Test: `tests/test_enrichment.py`

**Interfaces:**
- Consumes: `worklog.connect(db_path)` → `sqlite3.Connection`
- Produces:
  - `VALID_STRATEGIC_ROLES: set[str]`
  - `VALID_READINESS: set[str]`
  - `VALID_DECAY: set[str]`
  - `VALID_BLAST_RADIUS: set[str]`
  - `upsert_enrichment(conn, issue_number, issue_repo, **fields) → None`
  - `get_enrichment(conn, issue_number, issue_repo) → dict | None`
  - `list_enrichments(conn, issue_repo) → list[dict]`
  - `append_trajectory(conn, issue_number, issue_repo, note, source_branch=None) → int` (row id)
  - `get_trajectory(conn, issue_number, issue_repo, limit=10) → list[dict]`

- [ ] **Step 1: Write failing tests for enrichment CRUD**

Create `tests/test_enrichment.py`:

```python
"""Tests for scripts/enrichment.py"""

import json
import sqlite3
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))

import worklog
import enrichment


def _conn(tmp_path):
    return worklog.connect(str(tmp_path / "worklog.db"))


class TestUpsertEnrichment:
    def test_insert_new(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_enrichment(
            conn, 42, "Hortora/soredium",
            strategic_role="quick-win", readiness="ready",
        )
        row = conn.execute(
            "SELECT * FROM issue_enrichment WHERE issue_number=42"
        ).fetchone()
        assert row is not None
        assert row["strategic_role"] == "quick-win"
        assert row["readiness"] == "ready"
        assert row["updated_at"] is not None
        conn.close()

    def test_merge_existing(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_enrichment(
            conn, 42, "Hortora/soredium",
            strategic_role="quick-win", readiness="ready",
        )
        enrichment.upsert_enrichment(
            conn, 42, "Hortora/soredium",
            decay="compounding",
        )
        row = conn.execute(
            "SELECT * FROM issue_enrichment WHERE issue_number=42"
        ).fetchone()
        assert row["strategic_role"] == "quick-win"  # preserved
        assert row["readiness"] == "ready"            # preserved
        assert row["decay"] == "compounding"          # added
        conn.close()

    def test_validates_strategic_role(self, tmp_path):
        conn = _conn(tmp_path)
        import pytest
        with pytest.raises(ValueError, match="strategic_role"):
            enrichment.upsert_enrichment(
                conn, 42, "Hortora/soredium",
                strategic_role="invalid-role",
            )
        conn.close()

    def test_validates_readiness(self, tmp_path):
        conn = _conn(tmp_path)
        import pytest
        with pytest.raises(ValueError, match="readiness"):
            enrichment.upsert_enrichment(
                conn, 42, "Hortora/soredium",
                readiness="invalid",
            )
        conn.close()

    def test_validates_decay(self, tmp_path):
        conn = _conn(tmp_path)
        import pytest
        with pytest.raises(ValueError, match="decay"):
            enrichment.upsert_enrichment(
                conn, 42, "Hortora/soredium",
                decay="invalid",
            )
        conn.close()

    def test_validates_blast_radius(self, tmp_path):
        conn = _conn(tmp_path)
        import pytest
        with pytest.raises(ValueError, match="blast_radius"):
            enrichment.upsert_enrichment(
                conn, 42, "Hortora/soredium",
                blast_radius="invalid",
            )
        conn.close()

    def test_cohesion_is_freetext(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_enrichment(
            conn, 42, "Hortora/soredium",
            cohesion="lifecycle",
        )
        row = conn.execute(
            "SELECT cohesion FROM issue_enrichment WHERE issue_number=42"
        ).fetchone()
        assert row["cohesion"] == "lifecycle"
        conn.close()


class TestGetEnrichment:
    def test_returns_dict(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_enrichment(
            conn, 42, "Hortora/soredium",
            strategic_role="quick-win",
        )
        result = enrichment.get_enrichment(conn, 42, "Hortora/soredium")
        assert result is not None
        assert result["strategic_role"] == "quick-win"
        assert result["issue_number"] == 42
        conn.close()

    def test_returns_none_when_missing(self, tmp_path):
        conn = _conn(tmp_path)
        result = enrichment.get_enrichment(conn, 999, "Hortora/soredium")
        assert result is None
        conn.close()


class TestListEnrichments:
    def test_lists_all_for_repo(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_enrichment(conn, 1, "org/repo", strategic_role="quick-win")
        enrichment.upsert_enrichment(conn, 2, "org/repo", strategic_role="load-bearing")
        enrichment.upsert_enrichment(conn, 3, "org/other", strategic_role="quick-win")
        result = enrichment.list_enrichments(conn, "org/repo")
        assert len(result) == 2
        nums = {r["issue_number"] for r in result}
        assert nums == {1, 2}
        conn.close()


class TestTrajectoryNotes:
    def test_append_and_get(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.append_trajectory(conn, 42, "org/repo", "Schema landed, #43 is now ready")
        notes = enrichment.get_trajectory(conn, 42, "org/repo")
        assert len(notes) == 1
        assert notes[0]["note"] == "Schema landed, #43 is now ready"
        assert notes[0]["created_at"] is not None
        conn.close()

    def test_append_with_source_branch(self, tmp_path):
        conn = _conn(tmp_path)
        rid = enrichment.append_trajectory(
            conn, 42, "org/repo", "note text",
            source_branch="issue-42-schema",
        )
        assert rid is not None
        notes = enrichment.get_trajectory(conn, 42, "org/repo")
        assert notes[0]["source_branch"] == "issue-42-schema"
        conn.close()

    def test_accumulates_multiple_notes(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.append_trajectory(conn, 42, "org/repo", "first note")
        enrichment.append_trajectory(conn, 42, "org/repo", "second note")
        enrichment.append_trajectory(conn, 42, "org/repo", "third note")
        notes = enrichment.get_trajectory(conn, 42, "org/repo")
        assert len(notes) == 3
        assert notes[0]["note"] == "third note"  # most recent first
        assert notes[2]["note"] == "first note"
        conn.close()

    def test_limit(self, tmp_path):
        conn = _conn(tmp_path)
        for i in range(5):
            enrichment.append_trajectory(conn, 42, "org/repo", f"note {i}")
        notes = enrichment.get_trajectory(conn, 42, "org/repo", limit=2)
        assert len(notes) == 2
        conn.close()

    def test_scoped_to_issue(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.append_trajectory(conn, 42, "org/repo", "for 42")
        enrichment.append_trajectory(conn, 43, "org/repo", "for 43")
        notes = enrichment.get_trajectory(conn, 42, "org/repo")
        assert len(notes) == 1
        assert notes[0]["note"] == "for 42"
        conn.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_enrichment.py -v`
Expected: FAIL — `enrichment` module doesn't exist

- [ ] **Step 3: Write implementation**

Create `scripts/enrichment.py`:

```python
"""
enrichment.py — Issue enrichment and GitHub cache for what-next recommendations.

Extends the worklog DB (v2) with strategic classification, trajectory notes,
and cached GitHub issue state. Used by work-end (capture) and work-start (query).
"""

import datetime
import json
import sqlite3
import subprocess
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
import worklog

VALID_STRATEGIC_ROLES = {"quick-win", "load-bearing", "parallelizable", "dependency-unlocker", "consolidation"}
VALID_READINESS = {"ready", "needs-design", "needs-spike", "needs-decision"}
VALID_DECAY = {"stable", "compounding", "perishable"}
VALID_BLAST_RADIUS = {"isolated", "local", "cross-cutting", "foundational"}

_ENUM_FIELDS = {
    "strategic_role": VALID_STRATEGIC_ROLES,
    "readiness": VALID_READINESS,
    "decay": VALID_DECAY,
    "blast_radius": VALID_BLAST_RADIUS,
}

_ENRICHMENT_COLUMNS = [
    "strategic_role", "readiness", "decay", "blast_radius", "cohesion",
]


def _now() -> str:
    return datetime.datetime.now(datetime.timezone.utc).isoformat()


def _validate_enums(**fields) -> None:
    for field_name, valid_set in _ENUM_FIELDS.items():
        value = fields.get(field_name)
        if value is not None and value not in valid_set:
            raise ValueError(
                f"{field_name} must be one of {sorted(valid_set)}, got {value!r}"
            )


# --- Enrichment CRUD ---

def upsert_enrichment(conn: sqlite3.Connection, issue_number: int,
                      issue_repo: str, **fields) -> None:
    _validate_enums(**fields)
    existing = conn.execute(
        "SELECT * FROM issue_enrichment WHERE issue_number=? AND issue_repo=?",
        (issue_number, issue_repo),
    ).fetchone()
    merged = {}
    if existing:
        for col in _ENRICHMENT_COLUMNS:
            merged[col] = existing[col]
    for col in _ENRICHMENT_COLUMNS:
        if col in fields and fields[col] is not None:
            merged[col] = fields[col]
    conn.execute(
        "INSERT OR REPLACE INTO issue_enrichment "
        "(issue_number, issue_repo, strategic_role, readiness, decay, "
        "blast_radius, cohesion, updated_at) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
        (issue_number, issue_repo,
         merged.get("strategic_role"), merged.get("readiness"),
         merged.get("decay"), merged.get("blast_radius"),
         merged.get("cohesion"), _now()),
    )
    conn.commit()


def get_enrichment(conn: sqlite3.Connection, issue_number: int,
                   issue_repo: str) -> dict | None:
    row = conn.execute(
        "SELECT * FROM issue_enrichment WHERE issue_number=? AND issue_repo=?",
        (issue_number, issue_repo),
    ).fetchone()
    return dict(row) if row else None


def list_enrichments(conn: sqlite3.Connection,
                     issue_repo: str) -> list[dict]:
    rows = conn.execute(
        "SELECT * FROM issue_enrichment WHERE issue_repo=? ORDER BY issue_number",
        (issue_repo,),
    ).fetchall()
    return [dict(r) for r in rows]


# --- Trajectory Notes ---

def append_trajectory(conn: sqlite3.Connection, issue_number: int,
                      issue_repo: str, note: str,
                      source_branch: str | None = None) -> int:
    cur = conn.execute(
        "INSERT INTO trajectory_notes "
        "(issue_number, issue_repo, note, source_branch, created_at) "
        "VALUES (?, ?, ?, ?, ?)",
        (issue_number, issue_repo, note, source_branch, _now()),
    )
    conn.commit()
    return cur.lastrowid


def get_trajectory(conn: sqlite3.Connection, issue_number: int,
                   issue_repo: str, limit: int = 10) -> list[dict]:
    rows = conn.execute(
        "SELECT * FROM trajectory_notes "
        "WHERE issue_number=? AND issue_repo=? "
        "ORDER BY id DESC LIMIT ?",
        (issue_number, issue_repo, limit),
    ).fetchall()
    return [dict(r) for r in rows]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_enrichment.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/enrichment.py tests/test_enrichment.py
git commit -m "feat(#191): enrichment CRUD + trajectory notes — upsert, get, list, append, enum validation"
```

---

### Task 3: cache CRUD + refresh

**Refs:** #191

**Files:**
- Modify: `scripts/enrichment.py`
- Test: `tests/test_enrichment.py`

**Interfaces:**
- Consumes: `worklog.connect()`, `_now()`
- Produces:
  - `upsert_cached_issue(conn, issue_number, issue_repo, title, state, labels, body) → None`
  - `get_cached_issue(conn, issue_number, issue_repo) → dict | None`
  - `is_cache_fresh(conn, issue_repo, ttl_seconds=300) → bool`
  - `refresh_cache(conn, issue_repo, ttl_seconds=300) → int`

- [ ] **Step 1: Write failing tests for cache CRUD**

Add to `tests/test_enrichment.py`:

```python
from unittest.mock import patch, MagicMock
import time


class TestCachedIssueCRUD:
    def test_upsert_and_get(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 42, "org/repo",
            title="Fix bug", state="OPEN",
            labels=["enhancement", "scale:S"], body="Description",
        )
        result = enrichment.get_cached_issue(conn, 42, "org/repo")
        assert result is not None
        assert result["title"] == "Fix bug"
        assert result["state"] == "OPEN"
        assert json.loads(result["labels"]) == ["enhancement", "scale:S"]
        assert result["cached_at"] is not None
        conn.close()

    def test_upsert_replaces_existing(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 42, "org/repo",
            title="Old", state="OPEN", labels=[], body="",
        )
        enrichment.upsert_cached_issue(
            conn, 42, "org/repo",
            title="New", state="OPEN", labels=[], body="",
        )
        result = enrichment.get_cached_issue(conn, 42, "org/repo")
        assert result["title"] == "New"
        conn.close()

    def test_get_returns_none_when_missing(self, tmp_path):
        conn = _conn(tmp_path)
        result = enrichment.get_cached_issue(conn, 999, "org/repo")
        assert result is None
        conn.close()


class TestCacheFreshness:
    def test_fresh_when_within_ttl(self, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 42, "org/repo",
            title="t", state="OPEN", labels=[], body="",
        )
        assert enrichment.is_cache_fresh(conn, "org/repo", ttl_seconds=300)
        conn.close()

    def test_stale_when_no_rows(self, tmp_path):
        conn = _conn(tmp_path)
        assert not enrichment.is_cache_fresh(conn, "org/repo")
        conn.close()

    def test_stale_when_expired(self, tmp_path):
        conn = _conn(tmp_path)
        past = (
            datetime.datetime.now(datetime.timezone.utc)
            - datetime.timedelta(seconds=600)
        ).isoformat()
        conn.execute(
            "INSERT INTO github_issue_cache "
            "(issue_number, issue_repo, title, state, labels, body, cached_at) "
            "VALUES (?, ?, ?, ?, ?, ?, ?)",
            (42, "org/repo", "t", "OPEN", "[]", "", past),
        )
        conn.commit()
        assert not enrichment.is_cache_fresh(conn, "org/repo", ttl_seconds=300)
        conn.close()


class TestRefreshCache:
    def _gh_response(self):
        return json.dumps([
            {"number": 1, "title": "Issue 1", "state": "OPEN",
             "labels": [{"name": "enhancement"}], "body": "body1"},
            {"number": 2, "title": "Issue 2", "state": "OPEN",
             "labels": [{"name": "bug"}, {"name": "scale:S"}], "body": "body2"},
        ])

    @patch("enrichment.subprocess.run")
    def test_refresh_populates_cache(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        mock_run.return_value = MagicMock(
            returncode=0, stdout=self._gh_response(),
        )
        count = enrichment.refresh_cache(conn, "org/repo", ttl_seconds=0)
        assert count == 2
        r1 = enrichment.get_cached_issue(conn, 1, "org/repo")
        assert r1["title"] == "Issue 1"
        assert json.loads(r1["labels"]) == ["enhancement"]
        r2 = enrichment.get_cached_issue(conn, 2, "org/repo")
        assert json.loads(r2["labels"]) == ["bug", "scale:S"]
        conn.close()

    @patch("enrichment.subprocess.run")
    def test_refresh_skips_when_fresh(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 1, "org/repo", title="t", state="OPEN", labels=[], body="",
        )
        count = enrichment.refresh_cache(conn, "org/repo", ttl_seconds=9999)
        assert count == 0
        mock_run.assert_not_called()
        conn.close()

    @patch("enrichment.subprocess.run")
    def test_refresh_deletes_closed_issues(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        # Pre-populate issue 99 in cache
        enrichment.upsert_cached_issue(
            conn, 99, "org/repo", title="old", state="OPEN", labels=[], body="",
        )
        mock_run.return_value = MagicMock(
            returncode=0, stdout=self._gh_response(),  # only issues 1,2
        )
        enrichment.refresh_cache(conn, "org/repo", ttl_seconds=0)
        assert enrichment.get_cached_issue(conn, 99, "org/repo") is None
        conn.close()

    @patch("enrichment.subprocess.run")
    def test_refresh_preserves_cache_on_empty_response(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 1, "org/repo", title="existing", state="OPEN", labels=[], body="",
        )
        mock_run.return_value = MagicMock(returncode=0, stdout="[]")
        count = enrichment.refresh_cache(conn, "org/repo", ttl_seconds=0)
        assert count == 0
        assert enrichment.get_cached_issue(conn, 1, "org/repo") is not None
        conn.close()

    @patch("enrichment.subprocess.run")
    def test_refresh_handles_gh_failure(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        enrichment.upsert_cached_issue(
            conn, 1, "org/repo", title="existing", state="OPEN", labels=[], body="",
        )
        mock_run.side_effect = FileNotFoundError("gh not found")
        count = enrichment.refresh_cache(conn, "org/repo", ttl_seconds=0)
        assert count == 0
        assert enrichment.get_cached_issue(conn, 1, "org/repo") is not None
        conn.close()

    @patch("enrichment.subprocess.run")
    def test_refresh_handles_nonzero_exit(self, mock_run, tmp_path):
        conn = _conn(tmp_path)
        mock_run.return_value = MagicMock(returncode=1, stdout="", stderr="auth error")
        count = enrichment.refresh_cache(conn, "org/repo", ttl_seconds=0)
        assert count == 0
        conn.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_enrichment.py::TestCachedIssueCRUD tests/test_enrichment.py::TestCacheFreshness tests/test_enrichment.py::TestRefreshCache -v`
Expected: FAIL — functions don't exist

- [ ] **Step 3: Write implementation**

Add to `scripts/enrichment.py`:

```python
# --- Cache CRUD ---

def upsert_cached_issue(conn: sqlite3.Connection, issue_number: int,
                        issue_repo: str, title: str, state: str,
                        labels: list[str], body: str) -> None:
    conn.execute(
        "INSERT OR REPLACE INTO github_issue_cache "
        "(issue_number, issue_repo, title, state, labels, body, cached_at) "
        "VALUES (?, ?, ?, ?, ?, ?, ?)",
        (issue_number, issue_repo, title, state,
         json.dumps(labels), body, _now()),
    )
    conn.commit()


def get_cached_issue(conn: sqlite3.Connection, issue_number: int,
                     issue_repo: str) -> dict | None:
    row = conn.execute(
        "SELECT * FROM github_issue_cache WHERE issue_number=? AND issue_repo=?",
        (issue_number, issue_repo),
    ).fetchone()
    return dict(row) if row else None


def is_cache_fresh(conn: sqlite3.Connection, issue_repo: str,
                   ttl_seconds: int = 300) -> bool:
    row = conn.execute(
        "SELECT MIN(cached_at) as oldest FROM github_issue_cache WHERE issue_repo=?",
        (issue_repo,),
    ).fetchone()
    if row is None or row["oldest"] is None:
        return False
    oldest = datetime.datetime.fromisoformat(row["oldest"])
    cutoff = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(seconds=ttl_seconds)
    return oldest >= cutoff


# --- Cache Refresh ---

def refresh_cache(conn: sqlite3.Connection, issue_repo: str,
                  ttl_seconds: int = 300) -> int:
    if is_cache_fresh(conn, issue_repo, ttl_seconds):
        return 0
    try:
        result = subprocess.run(
            ["gh", "issue", "list", "--state", "open",
             "--json", "number,title,state,labels,body",
             "--limit", "500", "--repo", issue_repo],
            capture_output=True, text=True, timeout=30,
        )
        if result.returncode != 0:
            print(f"WARN=cache_refresh_failed repo={issue_repo} stderr={result.stderr.strip()}", file=sys.stderr)
            return 0
        issues = json.loads(result.stdout)
    except (FileNotFoundError, subprocess.TimeoutExpired, json.JSONDecodeError) as e:
        print(f"WARN=cache_refresh_error repo={issue_repo} detail={e}", file=sys.stderr)
        return 0
    if not issues:
        print(f"WARN=cache_refresh_empty repo={issue_repo}", file=sys.stderr)
        return 0
    fetched_numbers = {i["number"] for i in issues}
    with conn:
        for issue in issues:
            label_names = [l["name"] for l in issue.get("labels", [])]
            conn.execute(
                "INSERT OR REPLACE INTO github_issue_cache "
                "(issue_number, issue_repo, title, state, labels, body, cached_at) "
                "VALUES (?, ?, ?, ?, ?, ?, ?)",
                (issue["number"], issue_repo, issue["title"],
                 issue["state"], json.dumps(label_names),
                 issue.get("body", ""), _now()),
            )
        conn.execute(
            "DELETE FROM github_issue_cache "
            "WHERE issue_repo=? AND issue_number NOT IN ({})".format(
                ",".join("?" * len(fetched_numbers))
            ),
            (issue_repo, *sorted(fetched_numbers)),
        )
    return len(issues)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_enrichment.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/enrichment.py tests/test_enrichment.py
git commit -m "feat(#191): cache CRUD + refresh — upsert, get, freshness check, gh subprocess, empty response guard"
```

---

### Task 4: what_next query layer

**Refs:** #193

**Files:**
- Modify: `scripts/enrichment.py`
- Test: `tests/test_enrichment.py`

**Interfaces:**
- Consumes: `get_enrichment()`, `get_trajectory()`, `get_cached_issue()`, cache + enrichment tables
- Produces:
  - `what_next(conn, issue_repo, mode="general", cohesion_tag=None, limit=5) → list[dict]`

- [ ] **Step 1: Write failing tests**

Add to `tests/test_enrichment.py`:

```python
class TestWhatNext:
    def _seed(self, conn):
        """Seed cache and enrichment data for query tests."""
        # Cached issues
        enrichment.upsert_cached_issue(conn, 1, "org/repo", "Quick fix", "OPEN", ["scale:XS"], "")
        enrichment.upsert_cached_issue(conn, 2, "org/repo", "Big refactor", "OPEN", ["scale:L"], "")
        enrichment.upsert_cached_issue(conn, 3, "org/repo", "Urgent decay", "OPEN", [], "")
        enrichment.upsert_cached_issue(conn, 4, "org/repo", "Isolated task", "OPEN", [], "")
        enrichment.upsert_cached_issue(conn, 5, "org/repo", "Unenriched", "OPEN", [], "")
        # Enrichments
        enrichment.upsert_enrichment(conn, 1, "org/repo", strategic_role="quick-win", readiness="ready", decay="stable", blast_radius="isolated")
        enrichment.upsert_enrichment(conn, 2, "org/repo", strategic_role="load-bearing", readiness="ready", decay="stable", blast_radius="cross-cutting")
        enrichment.upsert_enrichment(conn, 3, "org/repo", strategic_role="consolidation", readiness="ready", decay="compounding", blast_radius="local")
        enrichment.upsert_enrichment(conn, 4, "org/repo", strategic_role="parallelizable", readiness="ready", decay="stable", blast_radius="isolated")
        # Trajectory
        enrichment.append_trajectory(conn, 1, "org/repo", "This unblocks #2")

    def test_general_returns_all_open(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="general")
        assert len(results) == 5
        assert all(r["issue_repo"] == "org/repo" for r in results)
        conn.close()

    def test_general_enriched_score_higher_than_unenriched(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="general")
        enriched_scores = [r["score"] for r in results if r["enriched"]]
        unenriched_scores = [r["score"] for r in results if not r["enriched"]]
        assert min(enriched_scores) > max(unenriched_scores)
        conn.close()

    def test_quick_wins_filters(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="quick-wins")
        assert len(results) >= 1
        assert all(r["strategic_role"] == "quick-win" for r in results if r["enriched"])
        conn.close()

    def test_compounding_filters(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="compounding")
        assert len(results) >= 1
        assert all(r["decay"] == "compounding" for r in results if r["enriched"])
        conn.close()

    def test_parallelizable_filters(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="parallelizable")
        enriched = [r for r in results if r["enriched"]]
        assert all(r["blast_radius"] == "isolated" for r in enriched)
        conn.close()

    def test_cohesion_filters(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        enrichment.upsert_enrichment(conn, 1, "org/repo", cohesion="lifecycle")
        enrichment.upsert_enrichment(conn, 4, "org/repo", cohesion="lifecycle")
        results = enrichment.what_next(conn, "org/repo", mode="cohesion", cohesion_tag="lifecycle")
        enriched = [r for r in results if r["enriched"]]
        assert all(r["cohesion"] == "lifecycle" for r in enriched)
        conn.close()

    def test_limit(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="general", limit=2)
        assert len(results) == 2
        conn.close()

    def test_unenriched_flagged(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="general")
        issue5 = [r for r in results if r["issue_number"] == 5][0]
        assert issue5["enriched"] is False
        assert issue5["score"] == 0
        conn.close()

    def test_includes_recent_trajectory(self, tmp_path):
        conn = _conn(tmp_path)
        self._seed(conn)
        results = enrichment.what_next(conn, "org/repo", mode="general")
        issue1 = [r for r in results if r["issue_number"] == 1][0]
        assert issue1["recent_trajectory"] is not None
        conn.close()

    def test_invalid_mode_raises(self, tmp_path):
        conn = _conn(tmp_path)
        import pytest
        with pytest.raises(ValueError, match="mode"):
            enrichment.what_next(conn, "org/repo", mode="invalid")
        conn.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_enrichment.py::TestWhatNext -v`
Expected: FAIL — `what_next` doesn't exist

- [ ] **Step 3: Write implementation**

Add to `scripts/enrichment.py`:

```python
# --- Query Layer ---

VALID_MODES = {"general", "quick-wins", "critical-path", "parallelizable", "compounding", "cohesion"}

_SCALE_ORDER = {"XS": 5, "S": 4, "M": 3, "L": 2, "XL": 1}
_DECAY_WEIGHT = {"compounding": 3, "perishable": 2, "stable": 1}
_ROLE_WEIGHT = {"dependency-unlocker": 4, "load-bearing": 3, "quick-win": 2, "consolidation": 1, "parallelizable": 1}
_READINESS_WEIGHT = {"ready": 3, "needs-spike": 2, "needs-design": 1, "needs-decision": 0}


def _parse_scale(labels_json: str | None) -> int:
    if not labels_json:
        return 0
    try:
        labels = json.loads(labels_json)
    except (json.JSONDecodeError, TypeError):
        return 0
    for label in labels:
        if ":" in label:
            prefix, value = label.split(":", 1)
            if prefix.lower() == "scale":
                return _SCALE_ORDER.get(value.upper(), 0)
    return 0


def _score_general(enrichment_row: dict | None, labels_json: str | None) -> int:
    if enrichment_row is None:
        return 0
    score = 0
    score += _ROLE_WEIGHT.get(enrichment_row.get("strategic_role", ""), 0)
    score += _READINESS_WEIGHT.get(enrichment_row.get("readiness", ""), 0)
    score += _DECAY_WEIGHT.get(enrichment_row.get("decay", ""), 0)
    score += _parse_scale(labels_json)
    return score


def what_next(conn: sqlite3.Connection, issue_repo: str,
              mode: str = "general", cohesion_tag: str | None = None,
              limit: int = 5) -> list[dict]:
    if mode not in VALID_MODES:
        raise ValueError(f"mode must be one of {sorted(VALID_MODES)}, got {mode!r}")

    rows = conn.execute(
        "SELECT c.issue_number, c.issue_repo, c.title, c.labels, "
        "e.strategic_role, e.readiness, e.decay, e.blast_radius, e.cohesion, e.updated_at "
        "FROM github_issue_cache c "
        "LEFT JOIN issue_enrichment e "
        "ON c.issue_number = e.issue_number AND c.issue_repo = e.issue_repo "
        "WHERE c.issue_repo=? AND c.state='OPEN'",
        (issue_repo,),
    ).fetchall()

    results = []
    for row in rows:
        row = dict(row)
        enriched = row.get("strategic_role") is not None
        enr = row if enriched else None

        if mode == "quick-wins" and enriched and row.get("strategic_role") != "quick-win":
            continue
        if mode == "quick-wins" and enriched and row.get("readiness") != "ready":
            continue
        if mode == "critical-path" and enriched and row.get("strategic_role") != "load-bearing":
            continue
        if mode == "parallelizable" and enriched:
            if row.get("blast_radius") != "isolated" or row.get("readiness") != "ready":
                continue
        if mode == "compounding" and enriched and row.get("decay") != "compounding":
            continue
        if mode == "cohesion" and enriched and row.get("cohesion") != cohesion_tag:
            continue

        score = _score_general(enr, row.get("labels")) if enriched else 0

        traj = conn.execute(
            "SELECT note, created_at FROM trajectory_notes "
            "WHERE issue_number=? AND issue_repo=? ORDER BY id DESC LIMIT 1",
            (row["issue_number"], row["issue_repo"]),
        ).fetchone()

        results.append({
            "issue_number": row["issue_number"],
            "issue_repo": row["issue_repo"],
            "title": row["title"],
            "score": score,
            "enriched": enriched,
            "strategic_role": row.get("strategic_role"),
            "readiness": row.get("readiness"),
            "decay": row.get("decay"),
            "blast_radius": row.get("blast_radius"),
            "cohesion": row.get("cohesion"),
            "recent_trajectory": dict(traj) if traj else None,
        })

    results.sort(key=lambda r: r["score"], reverse=True)
    return results[:limit]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_enrichment.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/enrichment.py tests/test_enrichment.py
git commit -m "feat(#193): what-next query layer — general, quick-wins, critical-path, parallelizable, compounding, cohesion modes"
```

---

### Task 5: CLI interface

**Refs:** #191

**Files:**
- Modify: `scripts/enrichment.py`
- Test: `tests/test_enrichment.py`

**Interfaces:**
- Consumes: all functions from Tasks 2-4
- Produces: CLI entry point `python3 scripts/enrichment.py <subcommand> [args]`

- [ ] **Step 1: Write failing tests**

Add to `tests/test_enrichment.py`:

```python
class TestCLI:
    def _run(self, *args, db_path=None):
        cmd = [sys.executable, str(Path(__file__).parent.parent / "scripts" / "enrichment.py")]
        cmd.extend(args)
        env = None
        if db_path:
            import os
            env = {**os.environ, "WORKLOG_DB": str(db_path)}
        result = subprocess.run(cmd, capture_output=True, text=True, env=env)
        return result

    def test_upsert_and_get(self, tmp_path):
        db = tmp_path / "worklog.db"
        worklog.connect(str(db)).close()
        r = self._run(
            "upsert", "--issue", "42", "--repo", "org/repo",
            "--role", "quick-win", "--readiness", "ready",
            db_path=db,
        )
        assert r.returncode == 0
        r = self._run("get", "--issue", "42", "--repo", "org/repo", db_path=db)
        assert r.returncode == 0
        data = json.loads(r.stdout)
        assert data["strategic_role"] == "quick-win"

    def test_trajectory_subcommand(self, tmp_path):
        db = tmp_path / "worklog.db"
        worklog.connect(str(db)).close()
        r = self._run(
            "trajectory", "--issue", "42", "--repo", "org/repo",
            "--text", "Schema landed",
            db_path=db,
        )
        assert r.returncode == 0

    def test_list_subcommand(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        enrichment.upsert_enrichment(conn, 1, "org/repo", strategic_role="quick-win")
        conn.close()
        r = self._run("list", "--repo", "org/repo", db_path=db)
        assert r.returncode == 0
        data = json.loads(r.stdout)
        assert len(data) == 1

    def test_invalid_subcommand(self, tmp_path):
        r = self._run("bogus")
        assert r.returncode != 0

    def test_upsert_invalid_enum(self, tmp_path):
        db = tmp_path / "worklog.db"
        worklog.connect(str(db)).close()
        r = self._run(
            "upsert", "--issue", "42", "--repo", "org/repo",
            "--role", "invalid",
            db_path=db,
        )
        assert r.returncode == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_enrichment.py::TestCLI -v`
Expected: FAIL — no CLI entry point

- [ ] **Step 3: Write implementation**

Add to `scripts/enrichment.py`:

```python
# --- CLI ---

import argparse
import os


def main():
    parser = argparse.ArgumentParser(description="Issue enrichment and GitHub cache")
    sub = parser.add_subparsers(dest="command", required=True)

    # upsert
    p_upsert = sub.add_parser("upsert")
    p_upsert.add_argument("--issue", type=int, required=True)
    p_upsert.add_argument("--repo", required=True)
    p_upsert.add_argument("--role")
    p_upsert.add_argument("--readiness")
    p_upsert.add_argument("--decay")
    p_upsert.add_argument("--blast-radius")
    p_upsert.add_argument("--cohesion")

    # trajectory
    p_traj = sub.add_parser("trajectory")
    p_traj.add_argument("--issue", type=int, required=True)
    p_traj.add_argument("--repo", required=True)
    p_traj.add_argument("--text", required=True)
    p_traj.add_argument("--branch")

    # get
    p_get = sub.add_parser("get")
    p_get.add_argument("--issue", type=int, required=True)
    p_get.add_argument("--repo", required=True)

    # list
    p_list = sub.add_parser("list")
    p_list.add_argument("--repo", required=True)

    # refresh
    p_refresh = sub.add_parser("refresh")
    p_refresh.add_argument("--repo", required=True)
    p_refresh.add_argument("--ttl", type=int, default=300)

    # what-next
    p_next = sub.add_parser("what-next")
    p_next.add_argument("--repo", required=True)
    p_next.add_argument("--mode", default="general")
    p_next.add_argument("--cohesion-tag")
    p_next.add_argument("--limit", type=int, default=5)

    args = parser.parse_args()
    db_path = os.environ.get("WORKLOG_DB")
    conn = worklog.connect(db_path)

    try:
        if args.command == "upsert":
            fields = {}
            if args.role:
                fields["strategic_role"] = args.role
            if args.readiness:
                fields["readiness"] = args.readiness
            if args.decay:
                fields["decay"] = args.decay
            if args.blast_radius:
                fields["blast_radius"] = args.blast_radius
            if args.cohesion:
                fields["cohesion"] = args.cohesion
            upsert_enrichment(conn, args.issue, args.repo, **fields)
            print(json.dumps({"ok": True}))

        elif args.command == "trajectory":
            rid = append_trajectory(conn, args.issue, args.repo, args.text,
                                    source_branch=args.branch)
            print(json.dumps({"ok": True, "id": rid}))

        elif args.command == "get":
            result = get_enrichment(conn, args.issue, args.repo)
            if result is None:
                print(json.dumps(None))
            else:
                print(json.dumps(result))

        elif args.command == "list":
            result = list_enrichments(conn, args.repo)
            print(json.dumps(result))

        elif args.command == "refresh":
            count = refresh_cache(conn, args.repo, ttl_seconds=args.ttl)
            print(json.dumps({"ok": True, "count": count}))

        elif args.command == "what-next":
            result = what_next(conn, args.repo, mode=args.mode,
                               cohesion_tag=args.cohesion_tag, limit=args.limit)
            print(json.dumps(result))

    except ValueError as e:
        print(json.dumps({"error": str(e)}), file=sys.stderr)
        sys.exit(1)
    finally:
        conn.close()


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_enrichment.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/enrichment.py tests/test_enrichment.py
git commit -m "feat(#191): CLI interface — upsert, trajectory, get, list, refresh, what-next subcommands"
```

---

### Task 6: work-end skill integration

**Refs:** #192

**Files:**
- Modify: `work-end/SKILL.md`

**Interfaces:**
- Consumes: `enrichment.py trajectory`, `enrichment.py upsert` CLI
- Produces: trajectory capture step in work-end lifecycle

This task edits a skill file (markdown), not Python code. No TDD applies.

- [ ] **Step 1: Read current work-end SKILL.md**

Read the work-end skill to identify the exact insertion point — after
`closing:promoted` but before `closing:pushed`.

- [ ] **Step 2: Add trajectory capture step**

Insert a new step between the promotion and push steps. The step should:

```markdown
### Step N: Trajectory capture (enrichment)

After artifacts are promoted and before the branch is pushed:

1. **Generate trajectory note** — using the full session context, draft a
   one-line trajectory note for each completed issue: "This work suggests
   X next because Y" (e.g., "Schema landed — #192 and #193 are now ready
   to implement").

2. **Propose enrichment updates** — assess how completed work shifts the
   strategic landscape for 2-3 sibling/related issues. Present in a table:

   | Issue | Field | Old | New | Reason |
   |-------|-------|-----|-----|--------|
   | #192 | readiness | needs-design | ready | Schema it depends on just landed |
   | #193 | readiness | needs-design | needs-design | Depends on #192 first |

3. **User confirms** — present the table. On YES:

   ```bash
   python3 scripts/enrichment.py trajectory --issue <N> --repo <REPO> --text "<note>" --branch <BRANCH>
   python3 scripts/enrichment.py upsert --issue <N> --repo <REPO> --readiness ready
   ```

4. **Failure is non-blocking** — if enrichment.py fails or the user
   declines, continue to the push step. Enrichment capture is valuable
   but never gates branch closure.
```

- [ ] **Step 3: Commit**

```bash
git add work-end/SKILL.md
git commit -m "feat(#192): trajectory capture step in work-end — enrichment updates after promotion, before push"
```

---

### Task 7: work-start what-next integration

**Refs:** #193

**Files:**
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: `enrichment.py refresh`, `enrichment.py what-next` CLI
- Produces: what-next recommendation in work router

This task edits a skill file (markdown), not Python code. No TDD applies.

- [ ] **Step 1: Read current work SKILL.md**

Read the work skill router to identify the insertion point — Step 1b
when the user is on main and hasn't specified an issue number.

- [ ] **Step 2: Add what-next recommendation**

In the `ROUTE=start` path (Step 2), before routing to work-start, add:

```markdown
**Step 2a: What-next recommendation (when no issue specified)**

If the user invoked `work` without an issue number:

1. Refresh the GitHub cache:
   ```bash
   python3 scripts/enrichment.py refresh --repo $OWNER_REPO
   ```

2. Query for recommendations:
   ```bash
   python3 scripts/enrichment.py what-next --repo $OWNER_REPO --mode general --limit 5
   ```

3. If results exist, present them:
   ```
   Recommended next:
     1. #42 — Fix caching bug (score: 12, quick-win, ready, compounding)
     2. #55 — Refactor auth (score: 8, load-bearing, ready, stable)
     3. #99 — Add tests (score: 0, not enriched)

   Pick a number, type an issue #, or describe what you want to work on.
   ```

4. If no enrichment data exists yet, skip silently — the feature
   bootstraps through work-end trajectory captures over time.

5. User picks → route to work-start with the selected issue number.
```

- [ ] **Step 3: Commit**

```bash
git add work/SKILL.md
git commit -m "feat(#193): what-next recommendation in work router — enrichment-powered backlog query before work-start"
```

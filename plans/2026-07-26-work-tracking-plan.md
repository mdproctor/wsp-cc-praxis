# Work Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** (to be created)
**Issue group:** single issue

**Goal:** Add SQLite-based lifecycle event tracking across all repos so
that active work, slot status, and historical events are queryable from
a single database.

**Architecture:** A `worklog.py` module in `scripts/` provides the write
API (called by lifecycle scripts) and query API (for future
project-manager skill). SQLite at `~/.hortora/worklog.db` with WAL mode.
Schema self-migrates via `PRAGMA user_version`. All writes are non-fatal
— worklog errors never block real operations.

**Tech Stack:** Python 3, sqlite3 (stdlib), pytest

## Global Constraints

- All timestamps UTC ISO 8601
- `worklog.py` must not import any non-stdlib module
- All write functions wrapped with `safe` decorator (non-fatal)
- Schema migrations embedded in module, triggered on `connect()`
- Tests use `tmp_path` fixtures, never hardcoded paths
- Per `externalised-scripts-require-tests` protocol: script + tests
  committed together

---

### Task 1: Core worklog module with schema and connection

**Files:**
- Create: `scripts/worklog.py`
- Create: `tests/test_worklog.py`

**Interfaces:**
- Produces: `connect(db_path=None) -> Connection`, `ensure_repo(conn, path, ...) -> int`, schema V1 tables

- [ ] **Step 1: Write failing test for connect + schema creation**

```python
# tests/test_worklog.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
import worklog


class TestConnect:
    def test_creates_db_and_tables(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        tables = [r[0] for r in conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        ).fetchall()]
        assert "repos" in tables
        assert "work_items" in tables
        assert "work_item_issues" in tables
        assert "slots" in tables
        assert "events" in tables
        conn.close()

    def test_wal_mode_enabled(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        mode = conn.execute("PRAGMA journal_mode").fetchone()[0]
        assert mode == "wal"
        conn.close()

    def test_creates_parent_directories(self, tmp_path):
        db = tmp_path / "sub" / "dir" / "worklog.db"
        conn = worklog.connect(str(db))
        assert db.exists()
        conn.close()

    def test_schema_version_set(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn = worklog.connect(str(db))
        ver = conn.execute("PRAGMA user_version").fetchone()[0]
        assert ver == 1
        conn.close()

    def test_idempotent_connect(self, tmp_path):
        db = tmp_path / "worklog.db"
        conn1 = worklog.connect(str(db))
        conn1.close()
        conn2 = worklog.connect(str(db))
        ver = conn2.execute("PRAGMA user_version").fetchone()[0]
        assert ver == 1
        conn2.close()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_worklog.py::TestConnect -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'worklog'`

- [ ] **Step 3: Implement connect() with schema V1**

```python
# scripts/worklog.py
"""
worklog.py — Cross-repo work lifecycle tracking

SQLite-based event log and state store for work-start, work-end,
work-pause, work-resume, slot-create, slot-merge, slot-archive events.
"""

import datetime
import functools
import json
import os
import sqlite3
from pathlib import Path

SCHEMA_VERSION = 1

DEFAULT_DB = os.path.expanduser("~/.hortora/worklog.db")

SCHEMA_V1 = """
CREATE TABLE IF NOT EXISTS repos (
    id           INTEGER PRIMARY KEY,
    path         TEXT UNIQUE NOT NULL,
    workspace    TEXT,
    family_root  TEXT,
    github_repo  TEXT,
    project_type TEXT
);

CREATE TABLE IF NOT EXISTS slots (
    id          INTEGER PRIMARY KEY,
    slot_number INTEGER NOT NULL,
    family_root TEXT NOT NULL,
    state       TEXT NOT NULL DEFAULT 'active',
    created_at  TEXT NOT NULL,
    archived_at TEXT,
    UNIQUE(slot_number, family_root)
);

CREATE TABLE IF NOT EXISTS work_items (
    id         INTEGER PRIMARY KEY,
    branch     TEXT NOT NULL,
    repo_id    INTEGER NOT NULL REFERENCES repos(id),
    state      TEXT NOT NULL DEFAULT 'active',
    location   TEXT NOT NULL DEFAULT 'primary',
    slot_id    INTEGER REFERENCES slots(id),
    work_path  TEXT,
    created_at TEXT NOT NULL,
    ended_at   TEXT,
    UNIQUE(branch, repo_id)
);

CREATE TABLE IF NOT EXISTS work_item_issues (
    work_item_id INTEGER NOT NULL REFERENCES work_items(id),
    issue_number INTEGER NOT NULL,
    issue_repo   TEXT NOT NULL,
    is_primary   INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (work_item_id, issue_number, issue_repo)
);

CREATE TABLE IF NOT EXISTS events (
    id           INTEGER PRIMARY KEY,
    timestamp    TEXT NOT NULL,
    event_type   TEXT NOT NULL,
    work_item_id INTEGER REFERENCES work_items(id),
    slot_id      INTEGER REFERENCES slots(id),
    repo_path    TEXT,
    metadata     TEXT
);

CREATE INDEX IF NOT EXISTS idx_events_type ON events(event_type);
CREATE INDEX IF NOT EXISTS idx_events_work_item ON events(work_item_id);
CREATE INDEX IF NOT EXISTS idx_events_slot ON events(slot_id);
CREATE INDEX IF NOT EXISTS idx_work_items_state ON work_items(state);
"""


def _now() -> str:
    return datetime.datetime.now(datetime.timezone.utc).isoformat()


def _migrate(conn: sqlite3.Connection) -> None:
    current = conn.execute("PRAGMA user_version").fetchone()[0]
    if current < 1:
        conn.executescript(SCHEMA_V1)
        conn.execute(f"PRAGMA user_version = {SCHEMA_VERSION}")
        conn.commit()


def connect(db_path: str | None = None) -> sqlite3.Connection:
    path = db_path or DEFAULT_DB
    Path(path).parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(path)
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    conn.row_factory = sqlite3.Row
    _migrate(conn)
    return conn


def safe(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        try:
            return fn(*args, **kwargs)
        except Exception as e:
            print(f"WARN=worklog_error detail={e}")
            return None
    return wrapper
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_worklog.py::TestConnect -v`
Expected: PASS (5 tests)

- [ ] **Step 5: Write failing test for ensure_repo**

```python
class TestEnsureRepo:
    def test_inserts_new_repo(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        rid = worklog.ensure_repo(conn, "/path/to/engine",
                                  workspace="/path/to/ws",
                                  github_repo="casehubio/engine")
        assert rid is not None
        row = conn.execute("SELECT * FROM repos WHERE id=?", (rid,)).fetchone()
        assert row["path"] == "/path/to/engine"
        assert row["github_repo"] == "casehubio/engine"
        conn.close()

    def test_returns_existing_repo_id(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        r1 = worklog.ensure_repo(conn, "/path/to/engine")
        r2 = worklog.ensure_repo(conn, "/path/to/engine")
        assert r1 == r2
        conn.close()

    def test_updates_fields_on_existing(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/path/to/engine")
        worklog.ensure_repo(conn, "/path/to/engine",
                            github_repo="casehubio/engine")
        row = conn.execute(
            "SELECT github_repo FROM repos WHERE path=?",
            ("/path/to/engine",)
        ).fetchone()
        assert row["github_repo"] == "casehubio/engine"
        conn.close()
```

- [ ] **Step 6: Implement ensure_repo**

```python
@safe
def ensure_repo(conn: sqlite3.Connection, path: str,
                workspace: str | None = None,
                family_root: str | None = None,
                github_repo: str | None = None,
                project_type: str | None = None) -> int | None:
    row = conn.execute("SELECT id FROM repos WHERE path=?", (path,)).fetchone()
    if row:
        updates = {}
        if workspace is not None:
            updates["workspace"] = workspace
        if family_root is not None:
            updates["family_root"] = family_root
        if github_repo is not None:
            updates["github_repo"] = github_repo
        if project_type is not None:
            updates["project_type"] = project_type
        if updates:
            sets = ", ".join(f"{k}=?" for k in updates)
            conn.execute(f"UPDATE repos SET {sets} WHERE id=?",
                         (*updates.values(), row["id"]))
            conn.commit()
        return row["id"]
    cur = conn.execute(
        "INSERT INTO repos (path, workspace, family_root, github_repo, project_type) "
        "VALUES (?, ?, ?, ?, ?)",
        (path, workspace, family_root, github_repo, project_type),
    )
    conn.commit()
    return cur.lastrowid
```

- [ ] **Step 7: Run all tests**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: PASS (8 tests)

- [ ] **Step 8: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/worklog.py tests/test_worklog.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): worklog module — schema V1, connect, ensure_repo"
```

---

### Task 2: Work item write functions

**Files:**
- Modify: `scripts/worklog.py`
- Modify: `tests/test_worklog.py`

**Interfaces:**
- Consumes: `connect()`, `ensure_repo()`, `safe`, `_now()`
- Produces: `record_work_start()`, `record_work_pause()`, `record_work_resume()`, `record_work_end()`

- [ ] **Step 1: Write failing test for record_work_start**

```python
class TestRecordWorkStart:
    def test_creates_work_item_and_event(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/repo/engine")
        wid = worklog.record_work_start(
            conn, "issue-42-spi", "/repo/engine",
            issue_number=42, issue_repo="casehubio/engine",
            covers="42,43",
        )
        assert wid is not None
        wi = conn.execute("SELECT * FROM work_items WHERE id=?", (wid,)).fetchone()
        assert wi["branch"] == "issue-42-spi"
        assert wi["state"] == "active"
        assert wi["location"] == "primary"
        issues = conn.execute(
            "SELECT * FROM work_item_issues WHERE work_item_id=? ORDER BY issue_number",
            (wid,)
        ).fetchall()
        assert len(issues) == 2
        assert issues[0]["issue_number"] == 42
        assert issues[0]["is_primary"] == 1
        assert issues[1]["issue_number"] == 43
        assert issues[1]["is_primary"] == 0
        evts = conn.execute(
            "SELECT * FROM events WHERE work_item_id=?", (wid,)
        ).fetchall()
        assert len(evts) == 1
        assert evts[0]["event_type"] == "work-start"
        conn.close()

    def test_single_issue(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/repo/engine")
        wid = worklog.record_work_start(
            conn, "issue-42-spi", "/repo/engine",
            issue_number=42, issue_repo="casehubio/engine",
        )
        issues = conn.execute(
            "SELECT * FROM work_item_issues WHERE work_item_id=?", (wid,)
        ).fetchall()
        assert len(issues) == 1
        assert issues[0]["is_primary"] == 1
        conn.close()

    def test_slot_location(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/repo/engine")
        worklog.record_slot_create(conn, 3, "/family",
                                   repos=["/repo/engine"],
                                   branch="issue-42-spi",
                                   issue_number=42, issue_repo="org/repo",
                                   covers="42")
        slot = conn.execute("SELECT id FROM slots WHERE slot_number=3").fetchone()
        wid = worklog.record_work_start(
            conn, "issue-42-spi", "/repo/engine",
            issue_number=42, issue_repo="org/repo",
            location="slot", slot_id=slot["id"],
        )
        wi = conn.execute("SELECT * FROM work_items WHERE id=?", (wid,)).fetchone()
        assert wi["location"] == "slot"
        assert wi["slot_id"] == slot["id"]
        conn.close()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_worklog.py::TestRecordWorkStart -v`
Expected: FAIL — `AttributeError: module 'worklog' has no attribute 'record_work_start'`

- [ ] **Step 3: Implement record_work_start**

```python
def _log_event(conn: sqlite3.Connection, event_type: str,
               work_item_id: int | None = None,
               slot_id: int | None = None,
               repo_path: str | None = None,
               metadata: dict | None = None) -> None:
    conn.execute(
        "INSERT INTO events (timestamp, event_type, work_item_id, slot_id, repo_path, metadata) "
        "VALUES (?, ?, ?, ?, ?, ?)",
        (_now(), event_type, work_item_id, slot_id, repo_path,
         json.dumps(metadata) if metadata else None),
    )


@safe
def record_work_start(conn: sqlite3.Connection, branch: str, repo_path: str,
                      issue_number: int, issue_repo: str,
                      covers: str | None = None,
                      location: str = "primary",
                      slot_id: int | None = None,
                      work_path: str | None = None) -> int | None:
    repo_id = ensure_repo(conn, repo_path)
    if repo_id is None:
        return None
    cur = conn.execute(
        "INSERT INTO work_items (branch, repo_id, state, location, slot_id, work_path, created_at) "
        "VALUES (?, ?, 'active', ?, ?, ?, ?)",
        (branch, repo_id, location, slot_id, work_path, _now()),
    )
    wid = cur.lastrowid
    issue_nums = [int(n.strip()) for n in (covers or str(issue_number)).split(",") if n.strip()]
    for num in issue_nums:
        conn.execute(
            "INSERT INTO work_item_issues (work_item_id, issue_number, issue_repo, is_primary) "
            "VALUES (?, ?, ?, ?)",
            (wid, num, issue_repo, 1 if num == issue_number else 0),
        )
    _log_event(conn, "work-start", work_item_id=wid, repo_path=repo_path)
    conn.commit()
    return wid
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_worklog.py::TestRecordWorkStart -v`
Expected: first two PASS, third may fail (needs record_slot_create — implement in Task 3)

- [ ] **Step 5: Write failing tests for pause, resume, end**

```python
class TestWorkItemLifecycle:
    def _start(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/repo/engine")
        wid = worklog.record_work_start(
            conn, "issue-42-spi", "/repo/engine",
            issue_number=42, issue_repo="org/repo",
        )
        return conn, wid

    def test_pause(self, tmp_path):
        conn, wid = self._start(tmp_path)
        worklog.record_work_pause(conn, "issue-42-spi", "/repo/engine")
        wi = conn.execute("SELECT state FROM work_items WHERE id=?", (wid,)).fetchone()
        assert wi["state"] == "paused"
        evts = conn.execute(
            "SELECT event_type FROM events WHERE work_item_id=? ORDER BY id",
            (wid,)
        ).fetchall()
        assert [e["event_type"] for e in evts] == ["work-start", "work-pause"]
        conn.close()

    def test_resume(self, tmp_path):
        conn, wid = self._start(tmp_path)
        worklog.record_work_pause(conn, "issue-42-spi", "/repo/engine")
        worklog.record_work_resume(conn, "issue-42-spi", "/repo/engine")
        wi = conn.execute("SELECT state FROM work_items WHERE id=?", (wid,)).fetchone()
        assert wi["state"] == "active"
        conn.close()

    def test_end(self, tmp_path):
        conn, wid = self._start(tmp_path)
        worklog.record_work_end(conn, "issue-42-spi", "/repo/engine",
                                landed_sha="abc123")
        wi = conn.execute("SELECT * FROM work_items WHERE id=?", (wid,)).fetchone()
        assert wi["state"] == "ended"
        assert wi["ended_at"] is not None
        evt = conn.execute(
            "SELECT metadata FROM events WHERE work_item_id=? AND event_type='work-end'",
            (wid,)
        ).fetchone()
        meta = json.loads(evt["metadata"])
        assert meta["landed_sha"] == "abc123"
        conn.close()

    def test_end_without_start_is_nonfatal(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        result = worklog.record_work_end(conn, "nonexistent", "/repo/x")
        assert result is None
        conn.close()
```

- [ ] **Step 6: Implement pause, resume, end**

```python
def _find_work_item(conn: sqlite3.Connection, branch: str,
                    repo_path: str) -> int | None:
    row = conn.execute(
        "SELECT wi.id FROM work_items wi "
        "JOIN repos r ON wi.repo_id = r.id "
        "WHERE wi.branch=? AND r.path=?",
        (branch, repo_path),
    ).fetchone()
    return row["id"] if row else None


@safe
def record_work_pause(conn: sqlite3.Connection, branch: str,
                      repo_path: str) -> None:
    wid = _find_work_item(conn, branch, repo_path)
    if wid is None:
        return
    conn.execute("UPDATE work_items SET state='paused' WHERE id=?", (wid,))
    _log_event(conn, "work-pause", work_item_id=wid, repo_path=repo_path)
    conn.commit()


@safe
def record_work_resume(conn: sqlite3.Connection, branch: str,
                       repo_path: str) -> None:
    wid = _find_work_item(conn, branch, repo_path)
    if wid is None:
        return
    conn.execute("UPDATE work_items SET state='active' WHERE id=?", (wid,))
    _log_event(conn, "work-resume", work_item_id=wid, repo_path=repo_path)
    conn.commit()


@safe
def record_work_end(conn: sqlite3.Connection, branch: str,
                    repo_path: str,
                    landed_sha: str | None = None) -> None:
    wid = _find_work_item(conn, branch, repo_path)
    if wid is None:
        return
    conn.execute(
        "UPDATE work_items SET state='ended', ended_at=? WHERE id=?",
        (_now(), wid),
    )
    meta = {"landed_sha": landed_sha} if landed_sha else None
    _log_event(conn, "work-end", work_item_id=wid, repo_path=repo_path,
               metadata=meta)
    conn.commit()
```

- [ ] **Step 7: Run all tests**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: PASS (all except slot-dependent test in TestRecordWorkStart)

- [ ] **Step 8: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/worklog.py tests/test_worklog.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): worklog work item lifecycle — start, pause, resume, end"
```

---

### Task 3: Slot write functions

**Files:**
- Modify: `scripts/worklog.py`
- Modify: `tests/test_worklog.py`

**Interfaces:**
- Consumes: `connect()`, `ensure_repo()`, `_log_event()`, `_now()`, `safe`
- Produces: `record_slot_create()`, `record_slot_phase_a()`, `record_slot_merge()`, `record_slot_archive()`

- [ ] **Step 1: Write failing tests for slot lifecycle**

```python
class TestSlotLifecycle:
    def _create(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/family/engine")
        worklog.ensure_repo(conn, "/family/iot")
        sid = worklog.record_slot_create(
            conn, 3, "/family",
            repos=["/family/engine", "/family/iot"],
            branch="issue-42-spi",
            issue_number=42, issue_repo="org/repo",
            covers="42,43",
        )
        return conn, sid

    def test_create(self, tmp_path):
        conn, sid = self._create(tmp_path)
        slot = conn.execute("SELECT * FROM slots WHERE id=?", (sid,)).fetchone()
        assert slot["slot_number"] == 3
        assert slot["family_root"] == "/family"
        assert slot["state"] == "active"
        wis = conn.execute(
            "SELECT * FROM work_items WHERE slot_id=?", (sid,)
        ).fetchall()
        assert len(wis) == 2
        assert all(w["location"] == "slot" for w in wis)
        evts = conn.execute(
            "SELECT * FROM events WHERE slot_id=?", (sid,)
        ).fetchall()
        assert any(e["event_type"] == "slot-create" for e in evts)
        conn.close()

    def test_phase_a(self, tmp_path):
        conn, sid = self._create(tmp_path)
        worklog.record_slot_phase_a(conn, 3, "/family")
        slot = conn.execute("SELECT state FROM slots WHERE id=?", (sid,)).fetchone()
        assert slot["state"] == "ready"
        conn.close()

    def test_merge(self, tmp_path):
        conn, sid = self._create(tmp_path)
        worklog.record_slot_merge(conn, 3, "/family",
                                  landed_shas={"engine": "abc", "iot": "def"})
        slot = conn.execute("SELECT state FROM slots WHERE id=?", (sid,)).fetchone()
        assert slot["state"] == "landed"
        wis = conn.execute(
            "SELECT state FROM work_items WHERE slot_id=?", (sid,)
        ).fetchall()
        assert all(w["state"] == "ended" for w in wis)
        conn.close()

    def test_archive(self, tmp_path):
        conn, sid = self._create(tmp_path)
        worklog.record_slot_merge(conn, 3, "/family",
                                  landed_shas={"engine": "abc"})
        worklog.record_slot_archive(conn, 3, "/family")
        slot = conn.execute("SELECT * FROM slots WHERE id=?", (sid,)).fetchone()
        assert slot["state"] == "archived"
        assert slot["archived_at"] is not None
        conn.close()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_worklog.py::TestSlotLifecycle -v`
Expected: FAIL — `AttributeError: module 'worklog' has no attribute 'record_slot_create'`

- [ ] **Step 3: Implement slot functions**

```python
def _find_slot(conn: sqlite3.Connection, slot_number: int,
               family_root: str) -> int | None:
    row = conn.execute(
        "SELECT id FROM slots WHERE slot_number=? AND family_root=?",
        (slot_number, family_root),
    ).fetchone()
    return row["id"] if row else None


@safe
def record_slot_create(conn: sqlite3.Connection, slot_number: int,
                       family_root: str, repos: list[str],
                       branch: str, issue_number: int,
                       issue_repo: str,
                       covers: str | None = None) -> int | None:
    cur = conn.execute(
        "INSERT INTO slots (slot_number, family_root, state, created_at) "
        "VALUES (?, ?, 'active', ?)",
        (slot_number, family_root, _now()),
    )
    sid = cur.lastrowid
    issue_nums = [int(n.strip()) for n in (covers or str(issue_number)).split(",") if n.strip()]
    for repo_path in repos:
        repo_id = ensure_repo(conn, repo_path, family_root=family_root)
        if repo_id is None:
            continue
        wi_cur = conn.execute(
            "INSERT INTO work_items (branch, repo_id, state, location, slot_id, created_at) "
            "VALUES (?, ?, 'active', 'slot', ?, ?)",
            (branch, repo_id, sid, _now()),
        )
        wid = wi_cur.lastrowid
        for num in issue_nums:
            conn.execute(
                "INSERT INTO work_item_issues (work_item_id, issue_number, issue_repo, is_primary) "
                "VALUES (?, ?, ?, ?)",
                (wid, num, issue_repo, 1 if num == issue_number else 0),
            )
    _log_event(conn, "slot-create", slot_id=sid,
               metadata={"repos": repos, "branch": branch})
    conn.commit()
    return sid


@safe
def record_slot_phase_a(conn: sqlite3.Connection, slot_number: int,
                        family_root: str) -> None:
    sid = _find_slot(conn, slot_number, family_root)
    if sid is None:
        return
    conn.execute("UPDATE slots SET state='ready' WHERE id=?", (sid,))
    _log_event(conn, "slot-phase-a", slot_id=sid)
    conn.commit()


@safe
def record_slot_merge(conn: sqlite3.Connection, slot_number: int,
                      family_root: str,
                      landed_shas: dict[str, str] | None = None) -> None:
    sid = _find_slot(conn, slot_number, family_root)
    if sid is None:
        return
    conn.execute("UPDATE slots SET state='landed' WHERE id=?", (sid,))
    conn.execute(
        "UPDATE work_items SET state='ended', ended_at=? WHERE slot_id=?",
        (_now(), sid),
    )
    _log_event(conn, "slot-merge", slot_id=sid,
               metadata={"landed_shas": landed_shas} if landed_shas else None)
    conn.commit()


@safe
def record_slot_archive(conn: sqlite3.Connection, slot_number: int,
                        family_root: str) -> None:
    sid = _find_slot(conn, slot_number, family_root)
    if sid is None:
        return
    conn.execute(
        "UPDATE slots SET state='archived', archived_at=? WHERE id=?",
        (_now(), sid),
    )
    _log_event(conn, "slot-archive", slot_id=sid)
    conn.commit()
```

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: PASS (all tests including the slot-dependent test from Task 2)

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/worklog.py tests/test_worklog.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): worklog slot lifecycle — create, phase-a, merge, archive"
```

---

### Task 4: Query functions

**Files:**
- Modify: `scripts/worklog.py`
- Modify: `tests/test_worklog.py`

**Interfaces:**
- Consumes: `connect()`, all `record_*` functions
- Produces: `active_work()`, `slot_status()`, `event_log()`, `work_item_timeline()`

- [ ] **Step 1: Write failing tests for queries**

```python
class TestQueries:
    def _setup(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/repo/engine", github_repo="org/engine")
        worklog.ensure_repo(conn, "/repo/iot", github_repo="org/iot")
        worklog.record_work_start(conn, "issue-42-spi", "/repo/engine",
                                  issue_number=42, issue_repo="org/engine")
        worklog.record_work_start(conn, "issue-55-fix", "/repo/iot",
                                  issue_number=55, issue_repo="org/iot")
        worklog.record_work_pause(conn, "issue-55-fix", "/repo/iot")
        return conn

    def test_active_work(self, tmp_path):
        conn = self._setup(tmp_path)
        result = worklog.active_work(conn)
        assert len(result) == 2
        branches = {r["branch"] for r in result}
        assert "issue-42-spi" in branches
        assert "issue-55-fix" in branches
        conn.close()

    def test_active_work_excludes_ended(self, tmp_path):
        conn = self._setup(tmp_path)
        worklog.record_work_end(conn, "issue-42-spi", "/repo/engine")
        result = worklog.active_work(conn)
        assert len(result) == 1
        assert result[0]["branch"] == "issue-55-fix"
        conn.close()

    def test_slot_status(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/family/engine")
        worklog.record_slot_create(conn, 1, "/family",
                                   repos=["/family/engine"],
                                   branch="issue-10-x",
                                   issue_number=10, issue_repo="org/r")
        result = worklog.slot_status(conn)
        assert len(result) == 1
        assert result[0]["slot_number"] == 1
        assert result[0]["state"] == "active"
        conn.close()

    def test_slot_status_filter_family(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        worklog.ensure_repo(conn, "/familyA/engine")
        worklog.ensure_repo(conn, "/familyB/engine")
        worklog.record_slot_create(conn, 1, "/familyA",
                                   repos=["/familyA/engine"],
                                   branch="b1", issue_number=1,
                                   issue_repo="org/r")
        worklog.record_slot_create(conn, 1, "/familyB",
                                   repos=["/familyB/engine"],
                                   branch="b2", issue_number=2,
                                   issue_repo="org/r")
        result = worklog.slot_status(conn, family_root="/familyA")
        assert len(result) == 1
        conn.close()

    def test_event_log(self, tmp_path):
        conn = self._setup(tmp_path)
        result = worklog.event_log(conn)
        assert len(result) >= 3
        conn.close()

    def test_event_log_filter_type(self, tmp_path):
        conn = self._setup(tmp_path)
        result = worklog.event_log(conn, event_type="work-pause")
        assert len(result) == 1
        conn.close()

    def test_work_item_timeline(self, tmp_path):
        conn = self._setup(tmp_path)
        worklog.record_work_resume(conn, "issue-55-fix", "/repo/iot")
        worklog.record_work_end(conn, "issue-55-fix", "/repo/iot")
        result = worklog.work_item_timeline(conn, "issue-55-fix", "/repo/iot")
        types = [r["event_type"] for r in result]
        assert types == ["work-start", "work-pause", "work-resume", "work-end"]
        conn.close()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_worklog.py::TestQueries -v`
Expected: FAIL — `AttributeError: module 'worklog' has no attribute 'active_work'`

- [ ] **Step 3: Implement query functions**

```python
def active_work(conn: sqlite3.Connection) -> list[dict]:
    rows = conn.execute(
        "SELECT wi.id, wi.branch, wi.state, wi.location, wi.slot_id, "
        "wi.created_at, r.path AS repo_path, r.github_repo "
        "FROM work_items wi JOIN repos r ON wi.repo_id = r.id "
        "WHERE wi.state != 'ended' "
        "ORDER BY wi.created_at",
    ).fetchall()
    return [dict(r) for r in rows]


def slot_status(conn: sqlite3.Connection,
                family_root: str | None = None) -> list[dict]:
    if family_root:
        rows = conn.execute(
            "SELECT * FROM slots WHERE family_root=? ORDER BY slot_number",
            (family_root,),
        ).fetchall()
    else:
        rows = conn.execute(
            "SELECT * FROM slots ORDER BY family_root, slot_number",
        ).fetchall()
    return [dict(r) for r in rows]


def event_log(conn: sqlite3.Connection,
              since: str | None = None,
              event_type: str | None = None,
              repo_path: str | None = None,
              limit: int = 100) -> list[dict]:
    clauses, params = [], []
    if since:
        clauses.append("timestamp >= ?")
        params.append(since)
    if event_type:
        clauses.append("event_type = ?")
        params.append(event_type)
    if repo_path:
        clauses.append("repo_path = ?")
        params.append(repo_path)
    where = (" WHERE " + " AND ".join(clauses)) if clauses else ""
    params.append(limit)
    rows = conn.execute(
        f"SELECT * FROM events{where} ORDER BY id DESC LIMIT ?",
        params,
    ).fetchall()
    return [dict(r) for r in rows]


def work_item_timeline(conn: sqlite3.Connection, branch: str,
                       repo_path: str) -> list[dict]:
    rows = conn.execute(
        "SELECT e.* FROM events e "
        "JOIN work_items wi ON e.work_item_id = wi.id "
        "JOIN repos r ON wi.repo_id = r.id "
        "WHERE wi.branch=? AND r.path=? "
        "ORDER BY e.id",
        (branch, repo_path),
    ).fetchall()
    return [dict(r) for r in rows]
```

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: PASS (all tests)

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/worklog.py tests/test_worklog.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): worklog query API — active_work, slot_status, event_log, timeline"
```

---

### Task 5: Failure isolation tests

**Files:**
- Modify: `tests/test_worklog.py`

**Interfaces:**
- Consumes: all `record_*` functions, `safe` decorator

- [ ] **Step 1: Write tests for non-fatal behavior**

```python
class TestFailureIsolation:
    def test_safe_decorator_catches_exceptions(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        conn.close()
        result = worklog.record_work_start(
            conn, "branch", "/repo", issue_number=1, issue_repo="org/r"
        )
        assert result is None

    def test_ensure_repo_on_closed_conn(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        conn.close()
        result = worklog.ensure_repo(conn, "/repo")
        assert result is None

    def test_record_pause_unknown_branch(self, tmp_path):
        conn = worklog.connect(str(tmp_path / "wl.db"))
        result = worklog.record_work_pause(conn, "nonexistent", "/repo")
        assert result is None
        conn.close()

    def test_concurrent_connections(self, tmp_path):
        db = str(tmp_path / "wl.db")
        conn1 = worklog.connect(db)
        conn2 = worklog.connect(db)
        worklog.ensure_repo(conn1, "/repo/a")
        worklog.ensure_repo(conn2, "/repo/b")
        repos = conn1.execute("SELECT COUNT(*) FROM repos").fetchone()[0]
        assert repos == 2
        conn1.close()
        conn2.close()
```

- [ ] **Step 2: Run tests**

Run: `python3 -m pytest tests/test_worklog.py::TestFailureIsolation -v`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/hortora/soredium add tests/test_worklog.py
git -C ~/claude/hortora/soredium commit -m "test(#N): worklog failure isolation and concurrency tests"
```

---

### Task 6: Wire into lifecycle scripts

**Files:**
- Modify: `work-start/scaffold.py` (add worklog call after .meta creation)
- Modify: `work-pause/pause_exec.py` (add worklog call after WIP commit)
- Modify: `work-resume/resume_exec.py` (add worklog call after branch checkout)
- Modify: `work-end/land_branch.py` (add worklog call after verified merge)
- Modify: `work-slot/slot_manager.py` (add worklog calls in create_slot, merge_slot, archive_slot)

**Interfaces:**
- Consumes: `worklog.connect()`, `worklog.record_*()`, `worklog.ensure_repo()`

The pattern is identical in every file:

```python
# At top of file, after other imports
try:
    import worklog as _wl
except ImportError:
    _wl = None

# At the integration point (after the operation succeeds)
if _wl:
    try:
        _conn = _wl.connect()
        _wl.record_work_start(_conn, branch, repo_path, ...)
        _conn.close()
    except Exception:
        pass
```

- [ ] **Step 1: Wire scaffold.py (work-start)**

In `scaffold.py`, after `print("CREATED=yes")` (line ~102), add:

```python
if _wl:
    try:
        _conn = _wl.connect()
        _wl.ensure_repo(_conn, args.get("project", ""),
                        github_repo=args.get("issue-repo", ""))
        _wl.record_work_start(
            _conn, args["branch"], args.get("project", ""),
            issue_number=int(args.get("issue", "0")),
            issue_repo=args.get("issue-repo", ""),
            covers=args.get("covers", ""),
        )
        _conn.close()
    except Exception:
        pass
```

- [ ] **Step 2: Wire pause_exec.py (work-pause)**

After `push_and_stack` succeeds, add worklog call with `record_work_pause`.

- [ ] **Step 3: Wire resume_exec.py (work-resume)**

After `checkout_branches` succeeds, add worklog call with `record_work_resume`.

- [ ] **Step 4: Wire land_branch.py (work-end)**

After `cmd_stamp` verifies and stamps, add worklog call with `record_work_end`.

- [ ] **Step 5: Wire slot_manager.py**

In `create_slot()` after `write_slot_md()`:
```python
if _wl:
    try:
        _conn = _wl.connect()
        repo_paths = [str(family_root / r) for r in repos]
        _wl.record_slot_create(_conn, slot_num, str(family_root),
                               repos=repo_paths, branch=branch,
                               issue_number=int(issue) if issue else 0,
                               issue_repo=issue_repo, covers=covers)
        _conn.close()
    except Exception:
        pass
```

In `merge_slot()` after `.landed` is written, add `record_slot_merge`.
In `archive_slot()` after `shutil.move`, add `record_slot_archive`.

- [ ] **Step 6: Run existing tests to verify nothing broke**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_worklog.py -v`
Expected: PASS (all tests — worklog import is optional, tests mock or don't use it)

- [ ] **Step 7: Commit**

```bash
git -C ~/claude/hortora/soredium add work-start/scaffold.py work-pause/pause_exec.py work-resume/resume_exec.py work-end/land_branch.py work-slot/slot_manager.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): wire worklog into lifecycle scripts"
```

---

### Task 7: Add worklog.py to sync-local

**Files:**
- Modify: `scripts/claude-skill` (cmd_sync_local function)

**Interfaces:**
- Consumes: `scripts/worklog.py` source file
- Produces: `~/.claude/lib/worklog.py` installed copy

- [ ] **Step 1: Write failing test**

```python
# In tests, verify the sync includes worklog.py
# Manual verification: after sync-local, ~/.claude/lib/worklog.py exists
```

- [ ] **Step 2: Add lib copy to cmd_sync_local**

At the end of `cmd_sync_local()` in `scripts/claude-skill`, after the
skills loop and before the hook sync, add:

```python
    lib_dir = Path.home() / ".claude" / "lib"
    lib_dir.mkdir(parents=True, exist_ok=True)
    worklog_src = repo_root / "scripts" / "worklog.py"
    if worklog_src.exists():
        worklog_dest = lib_dir / "worklog.py"
        shutil.copy2(str(worklog_src), str(worklog_dest))
        print("  ✅ worklog.py → ~/.claude/lib/")
```

- [ ] **Step 3: Add sys.path insert to lifecycle scripts**

Each lifecycle script's worklog import block becomes:

```python
import sys
from pathlib import Path
_lib = Path.home() / ".claude" / "lib"
if _lib.exists():
    sys.path.insert(0, str(_lib))
try:
    import worklog as _wl
except ImportError:
    _wl = None
```

- [ ] **Step 4: Run sync-local, verify worklog.py is copied**

Run: `python3 scripts/claude-skill sync-local --all -y`
Verify: `ls ~/.claude/lib/worklog.py`

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/claude-skill work-start/scaffold.py work-pause/pause_exec.py work-resume/resume_exec.py work-end/land_branch.py work-slot/slot_manager.py
git -C ~/claude/hortora/soredium commit -m "feat(#N): sync-local copies worklog.py to ~/.claude/lib/"
```

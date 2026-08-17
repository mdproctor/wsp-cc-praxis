# Worklog Event Emission Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #178 — Worklog event emission at state transitions
**Issue group:** #182, #178, #158, #141, #157

**Goal:** Wire automatic worklog event emission into `commit_transition()` so every lifecycle state change is logged without callers needing to remember.

**Architecture:** Add three public functions to `worklog.py` (extracted from existing internals). Add `repo_path` and `metadata` params to `commit_transition()` in `lifecycle.py`. After writing `.meta` state, `commit_transition()` calls worklog to update `work_items.state` and log a transition event. Worklog failures warn but never block.

**Tech Stack:** Python 3.14, pytest, SQLite (existing worklog.db schema — no migrations)

## Global Constraints

- No schema changes to worklog.db — use existing tables and columns
- Worklog emission is observability, never a gate — all worklog calls wrapped in try/except
- `externalised-scripts-require-tests` protocol — every script change ships with tests
- Existing `commit_transition(meta_path, result)` calls must continue to work (backward compat via optional params)

---

### Task 1: Extract public API from worklog.py

**Files:**
- Modify: `scripts/worklog.py`
- Test: `tests/test_worklog.py`

**Interfaces:**
- Produces:
  - `find_work_item(conn, branch, repo_path) -> int | None`
  - `update_work_item_state(conn, work_item_id, new_state) -> None`
  - `log_transition(conn, event_name, work_item_id=None, repo_path=None, metadata=None) -> None`

- [ ] **Step 1: Write failing tests for `find_work_item`**

```python
class TestFindWorkItem:
    def test_find_by_branch_and_repo(self, conn):
        repo_id = worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-1-foo", "/tmp/test-repo",
            issue_number=1, issue_repo="org/repo",
        )
        found = worklog.find_work_item(conn, "issue-1-foo", "/tmp/test-repo")
        assert found == wid

    def test_find_returns_none_when_missing(self, conn):
        found = worklog.find_work_item(conn, "nonexistent", "/tmp/nope")
        assert found is None

    def test_find_fallback_by_branch_only(self, conn):
        repo_id = worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-2-bar", "/tmp/test-repo",
            issue_number=2, issue_repo="org/repo",
        )
        found = worklog.find_work_item(conn, "issue-2-bar", "/tmp/other-path")
        assert found == wid
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::TestFindWorkItem -v`
Expected: FAIL with `AttributeError: module 'worklog' has no attribute 'find_work_item'`

- [ ] **Step 3: Implement `find_work_item`**

In `scripts/worklog.py`, make the existing `_find_work_item` public:

```python
def find_work_item(conn: sqlite3.Connection, branch: str,
                   repo_path: str) -> int | None:
    """Find work item ID by branch and repo path. Public API for lifecycle integration."""
    normalized = _norm(repo_path)
    row = conn.execute(
        "SELECT wi.id FROM work_items wi "
        "JOIN repos r ON wi.repo_id = r.id "
        "WHERE wi.branch=? AND r.path=?",
        (branch, normalized),
    ).fetchone()
    if row:
        return row["id"]
    row = conn.execute(
        "SELECT wi.id FROM work_items wi "
        "WHERE wi.branch=? AND wi.state != 'ended' "
        "ORDER BY wi.created_at DESC LIMIT 1",
        (branch,),
    ).fetchone()
    return row["id"] if row else None
```

Update `_find_work_item` to delegate: `_find_work_item = find_work_item`
(existing callers of `_find_work_item` continue to work).

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py::TestFindWorkItem -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for `update_work_item_state`**

```python
class TestUpdateWorkItemState:
    def test_update_to_paused(self, conn):
        worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-1-foo", "/tmp/test-repo",
            issue_number=1, issue_repo="org/repo",
        )
        worklog.update_work_item_state(conn, wid, "paused")
        row = conn.execute(
            "SELECT state FROM work_items WHERE id=?", (wid,)
        ).fetchone()
        assert row["state"] == "paused"

    def test_update_to_ended_sets_ended_at(self, conn):
        worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-1-foo", "/tmp/test-repo",
            issue_number=1, issue_repo="org/repo",
        )
        worklog.update_work_item_state(conn, wid, "ended")
        row = conn.execute(
            "SELECT state, ended_at FROM work_items WHERE id=?", (wid,)
        ).fetchone()
        assert row["state"] == "ended"
        assert row["ended_at"] is not None

    def test_update_to_active_clears_nothing(self, conn):
        worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-1-foo", "/tmp/test-repo",
            issue_number=1, issue_repo="org/repo",
        )
        worklog.update_work_item_state(conn, wid, "paused")
        worklog.update_work_item_state(conn, wid, "active")
        row = conn.execute(
            "SELECT state FROM work_items WHERE id=?", (wid,)
        ).fetchone()
        assert row["state"] == "active"
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::TestUpdateWorkItemState -v`
Expected: FAIL

- [ ] **Step 7: Implement `update_work_item_state`**

```python
@safe
def update_work_item_state(conn: sqlite3.Connection, work_item_id: int,
                           new_state: str) -> None:
    """Update work item state. Sets ended_at when state is 'ended'."""
    if new_state == "ended":
        conn.execute(
            "UPDATE work_items SET state=?, ended_at=? WHERE id=?",
            (new_state, _now(), work_item_id),
        )
    else:
        conn.execute(
            "UPDATE work_items SET state=? WHERE id=?",
            (new_state, work_item_id),
        )
    conn.commit()
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py::TestUpdateWorkItemState -v`
Expected: PASS

- [ ] **Step 9: Write failing tests for `log_transition`**

```python
class TestLogTransition:
    def test_log_transition_writes_event(self, conn):
        worklog.ensure_repo(conn, "/tmp/test-repo")
        wid = worklog.record_work_start(
            conn, "issue-1-foo", "/tmp/test-repo",
            issue_number=1, issue_repo="org/repo",
        )
        worklog.log_transition(
            conn, "work_pause", work_item_id=wid,
            repo_path="/tmp/test-repo",
            metadata={"from_state": "active", "to_state": "paused"},
        )
        events = worklog.event_log(conn, event_type="work_pause")
        assert len(events) >= 1
        last = events[0]
        assert last["work_item_id"] == wid
        meta = json.loads(last["metadata"])
        assert meta["from_state"] == "active"
        assert meta["to_state"] == "paused"

    def test_log_transition_without_work_item(self, conn):
        worklog.log_transition(
            conn, "work", work_item_id=None,
            repo_path="/tmp/test-repo",
            metadata={"from_state": "idle", "to_state": "scaffolded"},
        )
        events = worklog.event_log(conn, event_type="work")
        assert len(events) >= 1
        assert events[0]["work_item_id"] is None

    def test_log_transition_merges_caller_metadata(self, conn):
        worklog.log_transition(
            conn, "merge_pass", repo_path="/tmp/test-repo",
            metadata={
                "from_state": "closing:pushed",
                "to_state": "closing:merged",
                "landed_sha": "abc123",
            },
        )
        events = worklog.event_log(conn, event_type="merge_pass")
        meta = json.loads(events[0]["metadata"])
        assert meta["landed_sha"] == "abc123"
        assert meta["from_state"] == "closing:pushed"
```

- [ ] **Step 10: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::TestLogTransition -v`
Expected: FAIL

- [ ] **Step 11: Implement `log_transition`**

```python
@safe
def log_transition(conn: sqlite3.Connection, event_name: str,
                   work_item_id: int | None = None,
                   repo_path: str | None = None,
                   metadata: dict | None = None) -> None:
    """Log a lifecycle transition event."""
    _log_event(conn, event_name, work_item_id=work_item_id,
               repo_path=repo_path, metadata=metadata)
    conn.commit()
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py -v`
Expected: ALL PASS (including all existing tests)

- [ ] **Step 13: Commit**

```bash
git add scripts/worklog.py tests/test_worklog.py
git commit -m "feat(#178): extract public API from worklog.py — find_work_item, update_work_item_state, log_transition

Refs #178"
```

---

### Task 2: Wire worklog emission into commit_transition()

**Files:**
- Modify: `project/lifecycle.py`
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes:
  - `worklog.find_work_item(conn, branch, repo_path) -> int | None`
  - `worklog.update_work_item_state(conn, work_item_id, new_state) -> None`
  - `worklog.log_transition(conn, event_name, work_item_id, repo_path, metadata) -> None`
  - `worklog.connect() -> sqlite3.Connection`
- Produces:
  - `commit_transition(meta_path, result, repo_path=None, metadata=None) -> None`

- [ ] **Step 1: Write failing test — emission on normal transition**

```python
class TestWorklogEmission:
    @pytest.fixture
    def worklog_db(self, tmp_path):
        """Create a worklog DB with a work item for testing."""
        db_path = str(tmp_path / "worklog.db")
        sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
        import worklog
        conn = worklog.connect(db_path)
        worklog.ensure_repo(conn, str(tmp_path / "project"))
        worklog.record_work_start(
            conn, "issue-42-foo", str(tmp_path / "project"),
            issue_number=42, issue_repo="org/repo",
        )
        conn.close()
        return db_path

    def test_commit_emits_worklog_event(self, tmp_path, worklog_db, monkeypatch):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")

        monkeypatch.setenv("WORKLOG_DB", worklog_db)
        result = transition(meta, "work_pause")
        commit_transition(meta, result, repo_path=str(tmp_path / "project"))

        import worklog
        conn = worklog.connect(worklog_db)
        events = worklog.event_log(conn, event_type="work_pause")
        assert len(events) >= 1
        meta_json = json.loads(events[0]["metadata"])
        assert meta_json["from_state"] == "active"
        assert meta_json["to_state"] == "paused"
        conn.close()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle.py::TestWorklogEmission::test_commit_emits_worklog_event -v`
Expected: FAIL (commit_transition doesn't accept repo_path yet)

- [ ] **Step 3: Write failing test — work item state update**

```python
    def test_commit_updates_work_item_state(self, tmp_path, worklog_db, monkeypatch):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")

        monkeypatch.setenv("WORKLOG_DB", worklog_db)
        result = transition(meta, "work_pause")
        commit_transition(meta, result, repo_path=str(tmp_path / "project"))

        import worklog
        conn = worklog.connect(worklog_db)
        items = worklog.active_work(conn)
        paused = [i for i in items if i["branch"] == "issue-42-foo"]
        assert len(paused) == 1
        assert paused[0]["state"] == "paused"
        conn.close()
```

- [ ] **Step 4: Write failing test — caller metadata passed through**

```python
    def test_commit_passes_caller_metadata(self, tmp_path, worklog_db, monkeypatch):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: closing:pushed\ndate: 2026-08-03\n")

        monkeypatch.setenv("WORKLOG_DB", worklog_db)
        # Manually set work item to match the closing state
        import worklog
        conn = worklog.connect(worklog_db)
        wid = worklog.find_work_item(conn, "issue-42-foo", str(tmp_path / "project"))
        conn.close()

        result = transition(meta, "merge_pass")
        commit_transition(
            meta, result,
            repo_path=str(tmp_path / "project"),
            metadata={"landed_sha": "abc123"},
        )

        conn = worklog.connect(worklog_db)
        events = worklog.event_log(conn, event_type="merge_pass")
        assert len(events) >= 1
        meta_json = json.loads(events[0]["metadata"])
        assert meta_json["landed_sha"] == "abc123"
        assert meta_json["from_state"] == "closing:pushed"
        assert meta_json["to_state"] == "closing:merged"
        conn.close()
```

- [ ] **Step 5: Write failing test — backward compat (no repo_path)**

```python
    def test_commit_without_repo_path_skips_worklog(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")
        result = transition(meta, "work_pause")
        commit_transition(meta, result)  # no repo_path — backward compat
        assert read_state(meta) == "paused"  # .meta still written
```

- [ ] **Step 6: Write failing test — graceful degradation**

```python
    def test_commit_survives_worklog_failure(self, tmp_path, monkeypatch):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")

        monkeypatch.setenv("WORKLOG_DB", "/nonexistent/path/worklog.db")
        result = transition(meta, "work_pause")
        commit_transition(meta, result, repo_path=str(tmp_path / "project"))
        assert read_state(meta) == "paused"  # .meta write succeeded despite worklog error
```

- [ ] **Step 7: Write failing test — ended transition sets ended_at**

```python
    def test_commit_to_idle_sets_ended(self, tmp_path, worklog_db, monkeypatch):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: closing:stamped\ndate: 2026-08-03\n")

        monkeypatch.setenv("WORKLOG_DB", worklog_db)
        result = transition(meta, "cleanup_pass")
        commit_transition(meta, result, repo_path=str(tmp_path / "project"))

        import worklog
        conn = worklog.connect(worklog_db)
        row = conn.execute(
            "SELECT state, ended_at FROM work_items wi "
            "JOIN repos r ON wi.repo_id = r.id "
            "WHERE wi.branch='issue-42-foo'"
        ).fetchone()
        assert row["state"] == "ended"
        assert row["ended_at"] is not None
        conn.close()
```

- [ ] **Step 8: Run all new tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py::TestWorklogEmission -v`
Expected: FAIL

- [ ] **Step 9: Implement worklog emission in lifecycle.py**

Add the state mapping and emission function:

```python
_LIFECYCLE_TO_WORKLOG: dict[str, str | None] = {
    'scaffolded': 'active',
    'active': 'active',
    'transitioning': 'active',
    'paused': 'paused',
    'idle': 'ended',
}

def _read_branch(meta_path: Path) -> str | None:
    """Read branch name from .meta."""
    if not meta_path.exists():
        return None
    for line in meta_path.read_text().splitlines():
        if line.startswith('branch:'):
            return line.split(':', 1)[1].strip()
    return None

def _emit_to_worklog(
    meta_path: Path,
    result: TransitionResult,
    repo_path: str,
    metadata: dict | None,
) -> None:
    """Best-effort worklog emission. Never blocks."""
    try:
        import importlib
        import os
        scripts_dir = str(Path(__file__).resolve().parent.parent / "scripts")
        if scripts_dir not in sys.path:
            sys.path.insert(0, scripts_dir)
        import worklog

        db_path = os.environ.get("WORKLOG_DB")
        conn = worklog.connect(db_path) if db_path else worklog.connect()

        branch = _read_branch(meta_path)
        if not branch:
            conn.close()
            return

        wid = worklog.find_work_item(conn, branch, repo_path)

        wl_state = _LIFECYCLE_TO_WORKLOG.get(result.new_state)
        if wl_state and wid:
            worklog.update_work_item_state(conn, wid, wl_state)

        event_meta: dict = {
            'from_state': result.from_state,
            'to_state': result.new_state,
        }
        if metadata:
            event_meta.update(metadata)

        worklog.log_transition(
            conn, result.event,
            work_item_id=wid,
            repo_path=repo_path,
            metadata=event_meta,
        )
        conn.close()
    except Exception as e:
        print(f"WARN=worklog_error detail={e}")
```

Update `commit_transition()` signature and add the emission call:

```python
def commit_transition(
    meta_path: Path,
    result: TransitionResult,
    repo_path: str | None = None,
    metadata: dict | None = None,
) -> None:
    """Phase 2: Verify state unchanged, write atomically, emit worklog."""
    if result.from_state == 'idle':
        if not meta_path.exists():
            raise StateError(
                f".meta not created by write_meta effect at {meta_path}"
            )
        current = read_state(meta_path)
        if current != result.new_state:
            raise StateError(
                f"Expected '{result.new_state}' after scaffold, got '{current}'"
            )
    else:
        current = read_state(meta_path)
        if current != result.from_state:
            raise ConcurrentModification(
                expected=result.from_state, actual=current or 'None'
            )
        if result.new_state != 'idle':
            write_state(meta_path, result.new_state)

    if repo_path:
        _emit_to_worklog(meta_path, result, repo_path, metadata)
```

- [ ] **Step 10: Run all tests**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS (new tests + all existing tests)

- [ ] **Step 11: Run full test suite**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_worklog.py -v`
Expected: ALL PASS

- [ ] **Step 12: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle.py
git commit -m "feat(#178): wire worklog emission into commit_transition()

commit_transition() now automatically logs transition events and updates
work_items.state. Callers pass repo_path to enable emission; omitting it
preserves backward compatibility. Worklog failures warn but never block.

Refs #178"
```

---

### Task 3: Integration test — full lifecycle cycle

**Files:**
- Test: `tests/test_lifecycle.py` (add to existing)

**Interfaces:**
- Consumes: `commit_transition()` with `repo_path` from Task 2
- Produces: validation that a full idle→active→paused→active→closing→idle cycle produces correct event trail

- [ ] **Step 1: Write integration test**

```python
class TestWorklogIntegration:
    def test_full_lifecycle_produces_correct_event_trail(self, tmp_path):
        db_path = str(tmp_path / "worklog.db")
        project = tmp_path / "project"
        project.mkdir()
        repo_path = str(project)

        import os
        os.environ["WORKLOG_DB"] = db_path

        sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
        import worklog
        conn = worklog.connect(db_path)
        worklog.ensure_repo(conn, repo_path)
        conn.close()

        meta = tmp_path / ".meta"

        # idle → scaffolded
        result = transition(meta, "work")
        meta.write_text("branch: issue-99-test\nstate: scaffolded\ndate: 2026-08-04\n")
        conn = worklog.connect(db_path)
        worklog.record_work_start(
            conn, "issue-99-test", repo_path,
            issue_number=99, issue_repo="org/repo",
        )
        conn.close()
        commit_transition(meta, result, repo_path=repo_path)

        # scaffolded → active
        result = transition(meta, "auto_setup")
        commit_transition(meta, result, repo_path=repo_path)

        # active → paused
        result = transition(meta, "work_pause")
        commit_transition(meta, result, repo_path=repo_path)

        # paused → active
        result = transition(meta, "work_resume")
        commit_transition(meta, result, repo_path=repo_path)

        # active → closing:review
        result = transition(meta, "work_end")
        commit_transition(meta, result, repo_path=repo_path)

        # closing:review → closing:verified
        result = transition(meta, "review_pass")
        commit_transition(meta, result, repo_path=repo_path)

        # closing:verified → closing:promoted
        result = transition(meta, "promote_pass")
        commit_transition(meta, result, repo_path=repo_path)

        # closing:promoted → closing:pushed
        result = transition(meta, "push_pass")
        commit_transition(meta, result, repo_path=repo_path)

        # closing:pushed → closing:merged
        result = transition(meta, "merge_pass")
        commit_transition(
            meta, result, repo_path=repo_path,
            metadata={"landed_sha": "deadbeef"},
        )

        # closing:merged → closing:stamped
        result = transition(meta, "stamp_pass")
        commit_transition(meta, result, repo_path=repo_path)

        # closing:stamped → idle
        result = transition(meta, "cleanup_pass")
        commit_transition(meta, result, repo_path=repo_path)

        # Verify event trail
        conn = worklog.connect(db_path)
        events = worklog.event_log(conn, limit=50)
        conn.close()

        # Events are newest-first; reverse for chronological
        event_types = [e["event_type"] for e in reversed(events)]

        # work-start is from record_work_start, rest are from commit_transition
        assert "work-start" in event_types
        assert "work" in event_types
        assert "auto_setup" in event_types
        assert "work_pause" in event_types
        assert "work_resume" in event_types
        assert "work_end" in event_types
        assert "review_pass" in event_types
        assert "promote_pass" in event_types
        assert "push_pass" in event_types
        assert "merge_pass" in event_types
        assert "stamp_pass" in event_types
        assert "cleanup_pass" in event_types

        # Verify merge_pass has landed_sha
        merge_events = [
            e for e in events if e["event_type"] == "merge_pass"
        ]
        assert len(merge_events) == 1
        merge_meta = json.loads(merge_events[0]["metadata"])
        assert merge_meta["landed_sha"] == "deadbeef"

        # Verify work item ended
        conn = worklog.connect(db_path)
        row = conn.execute(
            "SELECT state, ended_at FROM work_items "
            "WHERE branch='issue-99-test'"
        ).fetchone()
        assert row["state"] == "ended"
        assert row["ended_at"] is not None
        conn.close()

        # Clean up env
        del os.environ["WORKLOG_DB"]
```

- [ ] **Step 2: Run integration test**

Run: `python3 -m pytest tests/test_lifecycle.py::TestWorklogIntegration -v`
Expected: PASS

- [ ] **Step 3: Run full test suite to confirm no regressions**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_worklog.py tests/test_work_lifecycle.py -v`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add tests/test_lifecycle.py
git commit -m "test(#178): integration test — full lifecycle cycle produces correct event trail

Refs #178"
```

---

### Task 4: Update docs/worklog.md

**Files:**
- Modify: `docs/worklog.md`

- [ ] **Step 1: Add Automatic Emission section**

After the "Event Types" section, add:

```markdown
## Automatic Emission

`lifecycle.commit_transition()` automatically logs a transition event and
updates `work_items.state` for every state change. Callers pass `repo_path`
to enable emission:

\`\`\`python
from lifecycle import transition, commit_transition

result = transition(meta_path, 'work_pause')
execute_effects(result.effects)
commit_transition(meta_path, result, repo_path="/path/to/repo")
\`\`\`

For transitions with domain metadata:

\`\`\`python
result = transition(meta_path, 'merge_pass')
sha = execute_merge(result.effects)
commit_transition(meta_path, result, repo_path="/path/to/repo",
    metadata={"landed_sha": sha})
\`\`\`

Worklog emission is best-effort — failures warn but never block the
`.meta` state write.

### Lifecycle-to-Worklog State Mapping

| Lifecycle state | work_items.state |
|-----------------|------------------|
| scaffolded, active, transitioning, closing:* | active |
| paused | paused |
| idle (from closing:stamped) | ended |
```

- [ ] **Step 2: Update Callers table**

Add `lifecycle.commit_transition()` to the Callers table:

```markdown
| `lifecycle.py` `commit_transition()` | All transition events (automatic) |
```

- [ ] **Step 3: Commit**

```bash
git add docs/worklog.md
git commit -m "docs(#178): document automatic worklog emission via commit_transition()

Refs #178"
```

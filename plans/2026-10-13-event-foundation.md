# Event Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #443 — Robust pipeline architecture: event-sourced lifecycle, classify-once context, outcome verification
**Issue group:** #443

**Goal:** Make the worklog DB events table the primary source of lifecycle state, close all `.plan` bypass writers, and add static analysis enforcement to prevent regression.

**Architecture:** Phase 1 of the robust pipeline architecture (spec: `2026-10-12-robust-pipeline-architecture-design.md`). Event emission becomes fail-loud (no silent drops). A new `get_lifecycle_state()` function queries DB events. `update_lifecycle_fields()` merges DB lifecycle state into `.plan`'s `## State` section while preserving queue data. All 14 bypass `.plan` writes across 7 files are routed through canonical writers (`plan_io.py`, `lifecycle.py`). A static analysis test prevents new bypasses.

**Tech Stack:** Python 3.12, SQLite (WAL mode), pytest

## Global Constraints

- All `.plan` lifecycle writes must route through `plan_io.write_fields()` or `plan_io.write_field()`
- All `.plan` removals must route through `plan_io.remove_plan()`
- Event emission failures must propagate — never silently dropped
- Existing tests must continue to pass after each task
- Protocol: `docs/protocols/git-util-single-source.md` pattern applies to plan_io (single source for `.plan` operations)

---

## Batch 1: Fail-Loud Event Emission

### Task 1: Remove @safe from worklog write path and add busy_timeout

**Files:**
- Modify: `scripts/worklog.py:165-184` (connect + @safe)
- Modify: `scripts/worklog.py:689-714` (update_work_item_state + log_transition)
- Modify: `project/lifecycle.py:285-328` (_emit_to_worklog)
- Test: `tests/test_worklog.py`
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes: existing `worklog.connect()`, `worklog.log_transition()`, `worklog.update_work_item_state()`
- Produces: same functions but fail-loud — errors propagate instead of being swallowed

- [ ] **Step 1: Write test proving @safe swallows errors**

```python
# tests/test_worklog.py — add to existing test file

def test_log_transition_propagates_db_errors(tmp_path):
    """log_transition must raise on DB errors, not swallow them."""
    import sqlite3
    conn = sqlite3.connect(":memory:")
    # Don't create tables — any insert will fail
    with pytest.raises(sqlite3.OperationalError):
        worklog.log_transition(conn, "started", work_item_id=1, repo_path="/fake")


def test_update_work_item_state_propagates_db_errors(tmp_path):
    """update_work_item_state must raise on DB errors, not swallow them."""
    import sqlite3
    conn = sqlite3.connect(":memory:")
    with pytest.raises(sqlite3.OperationalError):
        worklog.update_work_item_state(conn, 999, "active")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::test_log_transition_propagates_db_errors tests/test_worklog.py::test_update_work_item_state_propagates_db_errors -v`
Expected: FAIL — @safe catches the exception and returns None

- [ ] **Step 3: Remove @safe from log_transition and update_work_item_state**

In `scripts/worklog.py`:
- Line 689: remove `@safe` decorator from `update_work_item_state`
- Line 706: remove `@safe` decorator from `log_transition`
- Keep `@safe` on all OTHER functions (read-path functions remain best-effort)

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py::test_log_transition_propagates_db_errors tests/test_worklog.py::test_update_work_item_state_propagates_db_errors -v`
Expected: PASS

- [ ] **Step 5: Add busy_timeout to connect()**

In `scripts/worklog.py`, `connect()` function (line 165-173), add after WAL pragma:

```python
cur.execute("PRAGMA busy_timeout = 5000")
```

- [ ] **Step 6: Write test for busy_timeout**

```python
def test_connect_sets_busy_timeout(tmp_path):
    """connect() must set busy_timeout for write contention."""
    db_path = tmp_path / "test.db"
    conn = worklog.connect(db_path)
    cur = conn.execute("PRAGMA busy_timeout")
    timeout = cur.fetchone()[0]
    assert timeout == 5000
    conn.close()
```

- [ ] **Step 7: Run test to verify**

Run: `python3 -m pytest tests/test_worklog.py::test_connect_sets_busy_timeout -v`
Expected: PASS

- [ ] **Step 8: Remove blanket try/except from _emit_to_worklog**

In `project/lifecycle.py`, `_emit_to_worklog` (lines 285-328): replace the outer `try: ... except Exception as e: print(f"WARN=worklog_error detail={e}")` with direct calls. Keep the function structure but let exceptions propagate. Add a specific `sqlite3.OperationalError` catch ONLY for "database is locked" with a retry — all other errors propagate.

```python
def _emit_to_worklog(plan_path, result, repo_path=None):
    """Emit lifecycle event to worklog DB. Errors propagate."""
    from scripts import worklog
    db_path = _worklog_db_path()
    if not db_path:
        return
    conn = worklog.connect(db_path)
    try:
        wid = _resolve_work_item_id(conn, plan_path)
        if not wid:
            return
        wl_state = _map_lifecycle_to_worklog_state(result.to_state)
        worklog.update_work_item_state(conn, wid, wl_state)
        if repo_path:
            event_meta = {"from": result.from_state, "to": result.to_state}
            worklog.log_transition(
                conn, result.event,
                work_item_id=wid, repo_path=str(repo_path),
                metadata=event_meta
            )
    finally:
        conn.close()
```

- [ ] **Step 9: Write test for _emit_to_worklog propagation**

```python
# tests/test_lifecycle.py — add to existing

def test_emit_to_worklog_propagates_errors(tmp_path, monkeypatch):
    """_emit_to_worklog must not swallow DB errors."""
    from project.lifecycle import _emit_to_worklog
    from unittest.mock import MagicMock

    result = MagicMock()
    result.to_state = "active"
    result.from_state = "idle"
    result.event = "start"

    # Patch connect to return a broken connection
    import sqlite3
    bad_conn = sqlite3.connect(":memory:")  # no tables

    monkeypatch.setattr("project.lifecycle._worklog_db_path", lambda: tmp_path / "test.db")

    # Should propagate, not swallow
    # (exact error depends on implementation — the key assertion is it doesn't silently pass)
```

- [ ] **Step 10: Run full test suite to verify nothing broke**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_worklog.py -v`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add scripts/worklog.py project/lifecycle.py tests/test_worklog.py tests/test_lifecycle.py
git commit -m "feat(#443): make event emission fail-loud — remove @safe, add busy_timeout

Remove @safe from log_transition and update_work_item_state so DB
errors propagate. Remove blanket try/except from _emit_to_worklog.
Add busy_timeout=5000 for write contention. Prerequisite for
events-as-primary state source.

Refs #443"
```

---

## Batch 2: State Derivation Functions

### Task 2: Build get_lifecycle_state() query function

**Files:**
- Modify: `scripts/worklog.py` (add query function)
- Test: `tests/test_worklog.py`

**Interfaces:**
- Consumes: `worklog.connect()`, existing `events` table
- Produces: `get_lifecycle_state(conn, work_item_id, repo_path=None) -> str | None`

- [ ] **Step 1: Write failing test**

```python
# tests/test_worklog.py

def test_get_lifecycle_state_returns_latest(tmp_path):
    """get_lifecycle_state returns to_state from most recent event."""
    db_path = tmp_path / "test.db"
    conn = worklog.connect(db_path)

    # Create a work item
    wid = worklog.create_work_item(conn, repo_id=1, branch="test", title="test")

    # Log two transitions
    worklog.log_transition(conn, "start", work_item_id=wid, repo_path="/repo",
                          metadata={"from": "idle", "to": "active"})
    worklog.log_transition(conn, "review", work_item_id=wid, repo_path="/repo",
                          metadata={"from": "active", "to": "closing:review"})

    state = worklog.get_lifecycle_state(conn, wid, repo_path="/repo")
    assert state == "closing:review"
    conn.close()


def test_get_lifecycle_state_returns_none_when_no_events(tmp_path):
    """get_lifecycle_state returns None when no events exist."""
    db_path = tmp_path / "test.db"
    conn = worklog.connect(db_path)
    state = worklog.get_lifecycle_state(conn, 999)
    assert state is None
    conn.close()


def test_get_lifecycle_state_filters_by_repo(tmp_path):
    """get_lifecycle_state filters by repo_path when provided."""
    db_path = tmp_path / "test.db"
    conn = worklog.connect(db_path)
    wid = worklog.create_work_item(conn, repo_id=1, branch="test", title="test")

    worklog.log_transition(conn, "start", work_item_id=wid, repo_path="/repo-a",
                          metadata={"from": "idle", "to": "active"})
    worklog.log_transition(conn, "land", work_item_id=wid, repo_path="/repo-b",
                          metadata={"from": "idle", "to": "closing:pushed"})

    assert worklog.get_lifecycle_state(conn, wid, repo_path="/repo-a") == "active"
    assert worklog.get_lifecycle_state(conn, wid, repo_path="/repo-b") == "closing:pushed"
    conn.close()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog.py::test_get_lifecycle_state_returns_latest tests/test_worklog.py::test_get_lifecycle_state_returns_none_when_no_events tests/test_worklog.py::test_get_lifecycle_state_filters_by_repo -v`
Expected: FAIL — function doesn't exist

- [ ] **Step 3: Implement get_lifecycle_state**

Add to `scripts/worklog.py`:

```python
def get_lifecycle_state(conn, work_item_id: int, repo_path: str | None = None) -> str | None:
    """Query most recent lifecycle event for this work item + repo.
    Returns to_state from the latest event, or None if no events."""
    if repo_path:
        row = conn.execute(
            "SELECT json_extract(metadata, '$.to') FROM events "
            "WHERE work_item_id = ? AND repo_path = ? "
            "ORDER BY id DESC LIMIT 1",
            (work_item_id, repo_path)
        ).fetchone()
    else:
        row = conn.execute(
            "SELECT json_extract(metadata, '$.to') FROM events "
            "WHERE work_item_id = ? "
            "ORDER BY id DESC LIMIT 1",
            (work_item_id,)
        ).fetchone()
    return row[0] if row else None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog.py::test_get_lifecycle_state_returns_latest tests/test_worklog.py::test_get_lifecycle_state_returns_none_when_no_events tests/test_worklog.py::test_get_lifecycle_state_filters_by_repo -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/worklog.py tests/test_worklog.py
git commit -m "feat(#443): add get_lifecycle_state() query function

Queries most recent lifecycle event from worklog DB for a given
work_item_id + optional repo_path. Returns to_state or None.
Foundation for events-as-primary state source.

Refs #443"
```

### Task 3: Build update_lifecycle_fields() merge function

**Files:**
- Modify: `project/plan_io.py` (add update_lifecycle_fields)
- Test: `tests/test_plan_io.py`

**Interfaces:**
- Consumes: `plan_io.read_plan()`, `plan_io.write_fields()`, `worklog.get_lifecycle_state()`
- Produces: `update_lifecycle_fields(plan_path: Path, state: str, evidence_era: bool = False)` — updates lifecycle fields in `.plan` without touching queue sections

- [ ] **Step 1: Write failing test**

```python
# tests/test_plan_io.py

def test_update_lifecycle_fields_preserves_queue(tmp_path):
    """update_lifecycle_fields updates state: but preserves ## Queue."""
    plan = tmp_path / ".plan"
    plan.write_text(
        "## State\n"
        "state: active\n"
        "branch: issue-42-feature\n"
        "covers: 42\n"
        "\n"
        "## Queue\n"
        "- [x] #42 — implement widget ← active\n"
        "- [ ] #43 — add tests\n"
    )
    plan_io.update_lifecycle_fields(plan, state="closing:review")

    content = plan.read_text()
    assert "state: closing:review" in content
    assert "branch: issue-42-feature" in content  # preserved
    assert "- [x] #42" in content  # queue preserved
    assert "- [ ] #43" in content  # queue preserved


def test_update_lifecycle_fields_adds_evidence_era(tmp_path):
    """update_lifecycle_fields can set evidence-era field."""
    plan = tmp_path / ".plan"
    plan.write_text(
        "## State\n"
        "state: active\n"
        "branch: issue-42-feature\n"
    )
    plan_io.update_lifecycle_fields(plan, state="closing:review", evidence_era=True)

    content = plan.read_text()
    assert "state: closing:review" in content
    assert "evidence-era: true" in content


def test_update_lifecycle_fields_only_updates_state(tmp_path):
    """update_lifecycle_fields does not modify non-lifecycle fields."""
    plan = tmp_path / ".plan"
    plan.write_text(
        "## State\n"
        "state: active\n"
        "branch: issue-42-feature\n"
        "design-repo: casehubio/platform\n"
        "flyway-next-v: V042\n"
        "\n"
        "## Deferred\n"
        "- S: refactor the widget\n"
    )
    plan_io.update_lifecycle_fields(plan, state="closing:verified")

    content = plan.read_text()
    assert "state: closing:verified" in content
    assert "design-repo: casehubio/platform" in content
    assert "flyway-next-v: V042" in content
    assert "## Deferred" in content
    assert "refactor the widget" in content
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_io.py::test_update_lifecycle_fields_preserves_queue tests/test_plan_io.py::test_update_lifecycle_fields_adds_evidence_era tests/test_plan_io.py::test_update_lifecycle_fields_only_updates_state -v`
Expected: FAIL — function doesn't exist

- [ ] **Step 3: Implement update_lifecycle_fields**

Add to `project/plan_io.py`:

```python
def update_lifecycle_fields(plan_path: Path, state: str, evidence_era: bool = False):
    """Update lifecycle fields in .plan from authoritative source.
    Preserves queue sections, deferred items, and design fields."""
    updates = {"state": state}
    if evidence_era:
        updates["evidence-era"] = "true"
    write_fields(plan_path, updates)
```

This delegates to the existing `write_fields()` which already handles atomic writes and preserves non-matching lines.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_io.py::test_update_lifecycle_fields_preserves_queue tests/test_plan_io.py::test_update_lifecycle_fields_adds_evidence_era tests/test_plan_io.py::test_update_lifecycle_fields_only_updates_state -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add project/plan_io.py tests/test_plan_io.py
git commit -m "feat(#443): add update_lifecycle_fields() merge function

Updates lifecycle state fields in .plan via write_fields() without
touching queue sections, deferred items, or design fields. Used by
lifecycle.py after event emission to keep .plan snapshot current.

Refs #443"
```

---

## Batch 3: Bypass Closure — Lifecycle and Scaffold

### Task 4: Route lifecycle.py bypasses through plan_io

**Files:**
- Modify: `project/lifecycle.py:390-416` (_write_evidence_era → plan_io.write_field)
- Modify: `project/lifecycle.py:477` (commit_transition unlink → plan_io.remove_plan)
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes: `plan_io.write_field()`, `plan_io.remove_plan()`
- Produces: lifecycle.py no longer writes `.plan` directly

- [ ] **Step 1: Write test that _write_evidence_era uses plan_io**

```python
# tests/test_lifecycle.py

def test_write_evidence_era_uses_plan_io(tmp_path, monkeypatch):
    """_write_evidence_era must route through plan_io, not direct write."""
    from project import plan_io
    from project.lifecycle import _write_evidence_era

    plan = tmp_path / ".plan"
    plan.write_text("## State\nstate: active\nbranch: test\n")

    calls = []
    original_write_field = plan_io.write_field

    def tracking_write_field(path, field, value):
        calls.append((field, value))
        return original_write_field(path, field, value)

    monkeypatch.setattr("project.plan_io.write_field", tracking_write_field)
    _write_evidence_era(plan)

    assert ("evidence-era", "true") in calls
    assert "evidence-era: true" in plan.read_text()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle.py::test_write_evidence_era_uses_plan_io -v`
Expected: FAIL — _write_evidence_era writes directly, not through plan_io

- [ ] **Step 3: Replace _write_evidence_era with plan_io.write_field call**

In `project/lifecycle.py`, replace lines 390-416 (`_write_evidence_era` function body) with:

```python
def _write_evidence_era(plan_path: Path):
    """Set evidence-era: true in .plan via plan_io."""
    from project import plan_io
    plan_io.write_field(plan_path, "evidence-era", "true")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_lifecycle.py::test_write_evidence_era_uses_plan_io -v`
Expected: PASS

- [ ] **Step 5: Route commit_transition's unlink through plan_io.remove_plan**

In `project/lifecycle.py`, line 477, replace:
```python
plan_path.unlink(missing_ok=True)
```
with:
```python
from project import plan_io
plan_io.remove_plan(plan_path)
```

Verify `plan_io.remove_plan` handles missing files (check line 198 — it uses `unlink()` which will raise if missing). If needed, add `missing_ok=True` parameter to `remove_plan`.

- [ ] **Step 6: Run full lifecycle tests**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle.py
git commit -m "fix(#443): route lifecycle.py .plan writes through plan_io

Replace _write_evidence_era's direct file manipulation with
plan_io.write_field(). Replace commit_transition's plan_path.unlink()
with plan_io.remove_plan(). Closes 2 bypass paths.

Refs #443"
```

### Task 5: Route scaffold.py bypasses through plan_io

**Files:**
- Modify: `work-start/scaffold.py:95,109,130` (3 write_text calls → plan_io)
- Test: `tests/test_scaffold.py` or existing tests

**Interfaces:**
- Consumes: `plan_io.write_fields()`, `plan_io.remove_plan()`
- Produces: scaffold.py no longer writes `.plan` directly

- [ ] **Step 1: Read scaffold.py to understand the 3 write paths**

Read `work-start/scaffold.py` lines 80-140 to understand what each write does:
- Line 95: initial `.plan` creation with state fields
- Line 109: second write (variant path?)
- Line 130: third write

- [ ] **Step 2: Write test that scaffold uses plan_io for .plan creation**

```python
# In the appropriate test file

def test_scaffold_creates_plan_via_plan_io(tmp_path, monkeypatch):
    """scaffold must create .plan through plan_io, not direct write_text."""
    from project import plan_io

    writes = []
    original_write_fields = plan_io.write_fields

    def tracking_write(path, updates):
        writes.append(path.name)
        return original_write_fields(path, updates)

    monkeypatch.setattr("project.plan_io.write_fields", tracking_write)
    # Call scaffold with appropriate args (adjust based on reading step 1)
    # ...

    assert ".plan" in writes
```

- [ ] **Step 3: Replace write_text calls with plan_io.write_fields**

For each of the 3 `plan_path.write_text(...)` calls in scaffold.py:
1. Parse the content string to extract the key-value pairs
2. Replace with `plan_io.write_fields(plan_path, {...})` for creation
3. For initial creation (file doesn't exist), write the `## State` header first, then call `write_fields`

Implementation note: `plan_io.write_fields` expects an existing `## State` section. For initial creation, use a two-step approach:

```python
from project import plan_io

# Create minimal .plan with header
plan_path.write_text("## State\n\n## Queue\n")
# Then set fields through canonical writer
plan_io.write_fields(plan_path, {
    "state": "scaffolded",
    "branch": branch_name,
    "covers": covers_str,
    # ... other fields from the original write_text content
})
```

- [ ] **Step 4: Run scaffold-related tests**

Run: `python3 -m pytest tests/ -k scaffold -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add work-start/scaffold.py tests/
git commit -m "fix(#443): route scaffold.py .plan creation through plan_io

Replace 3 direct write_text() calls with plan_io.write_fields().
Initial .plan creation writes ## State header then sets fields via
canonical writer. Closes 3 bypass paths.

Refs #443"
```

---

## Batch 4: Bypass Closure — Orchestrator and Plan Manager

### Task 6: Route work_end_orchestrator.py bypasses through plan_io

**Files:**
- Modify: `work-end/work_end_orchestrator.py:556,573,600,684` (4 operations)
- Test: existing orchestrator tests

**Interfaces:**
- Consumes: `plan_io.remove_plan()`, `plan_io.copy_plan()`
- Produces: orchestrator no longer writes `.plan` directly. May require adding `copy_plan(src, dst)` to plan_io.

- [ ] **Step 1: Add copy_plan to plan_io**

The orchestrator does `shutil.copy2(plan_next, .plan)` and `shutil.copy2(ws_plan, slot_plan)`. Add a canonical copy function:

```python
# project/plan_io.py

def copy_plan(src: Path, dst: Path):
    """Copy a .plan file from src to dst atomically."""
    import shutil
    tmp = dst.with_suffix(".plan.tmp")
    shutil.copy2(src, tmp)
    tmp.replace(dst)
```

- [ ] **Step 2: Write test for copy_plan**

```python
# tests/test_plan_io.py

def test_copy_plan_preserves_content(tmp_path):
    """copy_plan copies .plan content atomically."""
    src = tmp_path / "src.plan"
    dst = tmp_path / "dst.plan"
    content = "## State\nstate: active\nbranch: test\n\n## Queue\n- [ ] #42\n"
    src.write_text(content)

    plan_io.copy_plan(src, dst)

    assert dst.read_text() == content
    assert not (tmp_path / "dst.plan.tmp").exists()  # temp cleaned up
```

- [ ] **Step 3: Run test**

Run: `python3 -m pytest tests/test_plan_io.py::test_copy_plan_preserves_content -v`
Expected: PASS

- [ ] **Step 4: Replace orchestrator bypass paths**

In `work-end/work_end_orchestrator.py`:
- Line 684 `_cleanup_close_artifacts`: replace `plan.unlink()` with `plan_io.remove_plan(plan)`
- Line 600 `_promote_plan_next_inline`: replace `_shutil.copy2(plan_next, ...)` with `plan_io.copy_plan(plan_next, ...)`
- Lines 556, 573 `_elevate_plan_inline`: replace `slot_plan.unlink()` with `plan_io.remove_plan(slot_plan)` and `_shutil.copy2(ws_plan, slot_plan)` with `plan_io.copy_plan(ws_plan, slot_plan)`

Add import at top of file:
```python
from project import plan_io
```

- [ ] **Step 5: Run orchestrator tests**

Run: `python3 -m pytest tests/ -k "work_end" -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/work_end_orchestrator.py project/plan_io.py tests/test_plan_io.py
git commit -m "fix(#443): route orchestrator .plan writes through plan_io

Replace 4 direct .plan operations in work_end_orchestrator.py with
plan_io.remove_plan() and new plan_io.copy_plan(). Add copy_plan()
for atomic .plan copying. Closes 4 bypass paths.

Refs #443"
```

### Task 7: Route plan_manager.py and migration script bypasses through plan_io

**Files:**
- Modify: `work-slot/plan_manager.py:414,713` (2 writes)
- Modify: `work-slot/plan_migrate.py:164,172` (2 writes)
- Modify: `project/migrate_meta.py:121` (1 write)
- Test: `tests/test_plan_manager.py`, `tests/test_plan_manager_no_set_state.py`

**Interfaces:**
- Consumes: `plan_io.write_fields()`
- Produces: plan_manager and migration scripts no longer write `.plan` directly

- [ ] **Step 1: Read plan_manager.py lines 410-420 and 710-720**

Understand what fields are being written at each bypass path.

- [ ] **Step 2: Replace plan_manager.py line 414**

This uses the `tmp_path` → `replace` atomic pattern. Replace with `plan_io.write_fields()`. The exact fields being written need to be extracted from the content string at this line.

- [ ] **Step 3: Replace plan_manager.py line 713**

This is a direct `plan_path.write_text(content)`. This is likely a full rewrite (queue manipulation changes the whole file). If so, this needs `plan_io.write_plan(plan_path, content)` — a new function that validates the content has `## State` before writing.

Add to `plan_io.py` if needed:

```python
def write_plan(plan_path: Path, content: str):
    """Write complete .plan content atomically. Validates ## State section exists."""
    if "## State" not in content:
        raise ValueError("Invalid .plan content: missing ## State section")
    tmp = plan_path.with_suffix(".plan.tmp")
    tmp.write_text(content)
    tmp.replace(plan_path)
```

- [ ] **Step 4: Replace plan_migrate.py lines 164, 172**

Route through `plan_io.write_plan()` for full content writes.

- [ ] **Step 5: Replace migrate_meta.py line 121**

Route through `plan_io.write_plan()`.

- [ ] **Step 6: Run plan_manager and migration tests**

Run: `python3 -m pytest tests/test_plan_manager.py tests/test_plan_manager_no_set_state.py -v`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add work-slot/plan_manager.py work-slot/plan_migrate.py project/migrate_meta.py project/plan_io.py tests/
git commit -m "fix(#443): route plan_manager and migration .plan writes through plan_io

Replace 5 direct .plan write_text() calls in plan_manager.py,
plan_migrate.py, and migrate_meta.py with plan_io functions. Add
write_plan() for full-content writes with section validation.
Closes remaining 5 bypass paths.

Refs #443"
```

---

## Batch 5: Enforcement and Verification

### Task 8: Static analysis bypass detection test

**Files:**
- Create: `tests/test_plan_bypass_enforcement.py`

**Interfaces:**
- Consumes: filesystem scan of Python files
- Produces: commit-tier test that fails if any `.plan` bypass exists

- [ ] **Step 1: Write the enforcement test**

```python
# tests/test_plan_bypass_enforcement.py
"""Static analysis: detect .plan bypass writers.

Runs at commit tier. Fails if any Python file outside the canonical
writers (plan_io.py, lifecycle.py) directly manipulates .plan files.
"""

import re
from pathlib import Path

import pytest

PROJECT_ROOT = Path(__file__).parent.parent

# Files allowed to write .plan
CANONICAL_WRITERS = {
    "project/plan_io.py",
    "project/lifecycle.py",
}

# Files exempt from enforcement (test fixtures only)
EXEMPT_FILES = {
    "work-end/audit_work_end.py",
    "tests/",  # all test files
}

# Patterns that indicate direct .plan manipulation
BYPASS_PATTERNS = [
    re.compile(r'plan_path\.write_text\('),
    re.compile(r'plan_path\.unlink\('),
    re.compile(r'\.plan.*\.write_text\('),
    re.compile(r'\.plan.*\.unlink\('),
    re.compile(r'\.plan.*\.replace\('),
    re.compile(r'shutil\.copy2\(.*\.plan'),
    re.compile(r'tmp_path\.replace\(.*plan'),
]

def _is_exempt(rel_path: str) -> bool:
    if rel_path in CANONICAL_WRITERS:
        return True
    for exempt in EXEMPT_FILES:
        if rel_path.startswith(exempt):
            return True
    return False


def test_no_plan_bypass_writers():
    """No Python file outside canonical writers may directly manipulate .plan."""
    violations = []

    for py_file in PROJECT_ROOT.rglob("*.py"):
        rel = str(py_file.relative_to(PROJECT_ROOT))
        if _is_exempt(rel):
            continue
        if "engine/" in rel:  # Java subproject, no .plan
            continue

        try:
            content = py_file.read_text()
        except UnicodeDecodeError:
            continue

        for i, line in enumerate(content.splitlines(), 1):
            for pattern in BYPASS_PATTERNS:
                if pattern.search(line):
                    violations.append(f"{rel}:{i}: {line.strip()}")

    assert violations == [], (
        f"Found {len(violations)} .plan bypass write(s) outside canonical writers:\n"
        + "\n".join(violations)
    )
```

- [ ] **Step 2: Run the enforcement test**

Run: `python3 -m pytest tests/test_plan_bypass_enforcement.py -v`
Expected: PASS (if all previous bypass closures are complete). If any remain, this test tells you exactly where.

- [ ] **Step 3: Commit**

```bash
git add tests/test_plan_bypass_enforcement.py
git commit -m "feat(#443): add static analysis test for .plan bypass enforcement

Commit-tier test that greps all Python files for direct .plan
manipulation outside canonical writers (plan_io.py, lifecycle.py).
Prevents regression permanently.

Refs #443"
```

### Task 9: Post-rebase .plan regeneration hook

**Files:**
- Modify: `project/lifecycle.py` (add regeneration call after rebase events)
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes: `plan_io.update_lifecycle_fields()`, `worklog.get_lifecycle_state()`
- Produces: after rebase/merge events, `.plan` lifecycle fields are regenerated from DB

- [ ] **Step 1: Write test for post-rebase regeneration**

```python
# tests/test_lifecycle.py

def test_plan_regenerated_after_rebase(tmp_path, monkeypatch):
    """After rebase corrupts .plan state, regeneration restores from DB."""
    from project import plan_io

    plan = tmp_path / ".plan"
    plan.write_text(
        "## State\n"
        "state: closing:review\n"
        "branch: issue-42\n"
        "\n"
        "## Queue\n"
        "- [x] #42 — work\n"
    )

    # Simulate rebase corruption — state field mangled
    plan.write_text(
        "## State\n"
        "state: active\n"  # wrong — should be closing:review
        "branch: issue-42\n"
        "\n"
        "## Queue\n"
        "- [x] #42 — work\n"
    )

    # Regenerate from DB state
    plan_io.update_lifecycle_fields(plan, state="closing:review")

    content = plan.read_text()
    assert "state: closing:review" in content
    assert "- [x] #42" in content  # queue preserved
```

- [ ] **Step 2: Run test**

Run: `python3 -m pytest tests/test_lifecycle.py::test_plan_regenerated_after_rebase -v`
Expected: PASS (this uses update_lifecycle_fields from Task 3)

- [ ] **Step 3: Add regeneration call to commit_transition**

In `project/lifecycle.py`, in `commit_transition()`, after `_emit_to_worklog()` call (line ~474), add:

```python
# Regenerate .plan lifecycle fields from DB state
if result.to_state != "idle":  # idle transitions remove .plan
    plan_io.update_lifecycle_fields(plan_path, state=result.to_state)
```

This ensures `.plan` is always consistent with DB after every lifecycle transition — including after rebase corruption.

- [ ] **Step 4: Run full test suite**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_plan_io.py -v`
Expected: ALL PASS

- [ ] **Step 5: Run the full project test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle.py
git commit -m "feat(#443): regenerate .plan lifecycle fields after every transition

After each lifecycle event emission, update_lifecycle_fields()
restores .plan state from the authoritative transition result.
Handles post-rebase corruption: any .plan field mangled by git
operations is overwritten by the correct DB-derived value.

Refs #443"
```

---

## References

- [2026-10-12-robust-pipeline-architecture-design.md] — design spec this plan implements (Phase 1)
- project/lifecycle.py:285-477 — _emit_to_worklog, _write_evidence_era, commit_transition
- scripts/worklog.py:165-714 — connect, @safe, log_transition, update_work_item_state, events table
- project/plan_io.py:40-198 — read_plan, write_fields, write_field, remove_plan
- work-start/scaffold.py:95-130 — 3 bypass writes
- work-end/work_end_orchestrator.py:556-684 — 4 bypass operations
- work-slot/plan_manager.py:414-713 — 2 bypass writes
- work-slot/plan_migrate.py:164-172 — 2 bypass writes
- project/migrate_meta.py:121 — 1 bypass write
- docs/protocols/git-util-single-source.md — single-source protocol (applied to plan_io)
- docs/protocols/evidence-before-claims.md — verification protocol
- GitHub #443 — parent epic

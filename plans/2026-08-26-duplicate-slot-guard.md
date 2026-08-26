# Duplicate Slot Guard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #291 — Guard against duplicate slot creation and clean up partial failures
**Issue group:** #291

**Goal:** Prevent duplicate slot creation, clean up partial failures automatically, reuse abandoned slot numbers, and quarantine ghost directories.

**Architecture:** Four changes across three files: (1) `worklog.py` gets `fail_slot` and `find_reusable_slot` functions, (2) `slot_manager.py` gets `SlotCreationError`, `find_slot_by_branch`, refactored error paths, try/except rollback, pending reuse, and a `list_slots` ghost filter, (3) `reconcile_slots.py` gets content-aware quarantine strategy and `.claude/projects` relocation.

**Tech Stack:** Python 3, SQLite (worklog DB), pytest, pathlib

## Global Constraints

- All new `.py` functions ship with pytest tests in the same commit (protocol: `externalised-scripts-require-tests`)
- No `ON DELETE CASCADE` on the `events.slot_id` FK — never delete from `slots` table; use state transitions
- Concurrent slot creation is not supported — guards protect against sequential retry only
- `worklog.py` lives at `~/.claude/lib/worklog.py` — the installed copy, not in this repo. Changes to worklog functions are tested via the repo's `scripts/worklog.py` (same file, synced)

---

## Batch 1: DB layer — fail_slot and find_reusable_slot

### Task 1: Add `fail_slot` and `find_reusable_slot` to worklog.py

**Files:**
- Modify: `scripts/worklog.py` (add two functions after `confirm_slot_create`)
- Create: `tests/test_worklog_slot_guards.py`

**Interfaces:**
- Produces: `fail_slot(conn, slot_number, family_root)` — transitions any slot to `state='failed'`
- Produces: `find_reusable_slot(conn, family_root)` → `tuple[int, list[int]] | None` — returns `(highest_pending_or_failed, [other_nums])` or `None`

- [ ] **Step 1: Write failing tests for `fail_slot`**

```python
# tests/test_worklog_slot_guards.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
import worklog as wl

import pytest


@pytest.fixture
def db(tmp_path, monkeypatch):
    db_path = tmp_path / "test.db"
    monkeypatch.setattr(wl, "DEFAULT_DB", str(db_path))
    conn = wl.connect()
    yield conn
    conn.close()


class TestFailSlot:
    def test_transitions_pending_to_failed(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (1, '/tmp/family', 'pending', '2026-01-01')")
        db.commit()
        wl.fail_slot(db, 1, "/tmp/family")
        row = db.execute("SELECT state FROM slots WHERE slot_number=1 AND family_root='/tmp/family'").fetchone()
        assert row["state"] == "failed"

    def test_transitions_active_to_failed(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (2, '/tmp/family', 'active', '2026-01-01')")
        db.commit()
        wl.fail_slot(db, 2, "/tmp/family")
        row = db.execute("SELECT state FROM slots WHERE slot_number=2 AND family_root='/tmp/family'").fetchone()
        assert row["state"] == "failed"

    def test_preserves_events(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (3, '/tmp/family', 'active', '2026-01-01')")
        db.commit()
        sid = db.execute("SELECT id FROM slots WHERE slot_number=3").fetchone()["id"]
        db.execute(
            "INSERT INTO events (event_type, timestamp, slot_id) "
            "VALUES ('slot-create', '2026-01-01', ?)", (sid,))
        db.commit()
        wl.fail_slot(db, 3, "/tmp/family")
        event = db.execute("SELECT * FROM events WHERE slot_id=?", (sid,)).fetchone()
        assert event is not None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog_slot_guards.py -v`
Expected: FAIL with `AttributeError: module 'worklog' has no attribute 'fail_slot'`

- [ ] **Step 3: Implement `fail_slot`**

Add to `scripts/worklog.py` after `confirm_slot_create` (around line 267):

```python
def fail_slot(conn: sqlite3.Connection, slot_number: int,
              family_root: str) -> None:
    """Transition a slot to failed state. Works for both pending and active.
    Preserves audit trail — no deletion. No @safe."""
    family_root = _norm(family_root)
    conn.execute(
        "UPDATE slots SET state='failed' WHERE slot_number=? AND family_root=?",
        (slot_number, family_root),
    )
    conn.commit()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog_slot_guards.py::TestFailSlot -v`
Expected: 3 PASS

- [ ] **Step 5: Write failing tests for `find_reusable_slot`**

Append to `tests/test_worklog_slot_guards.py`:

```python
class TestFindReusableSlot:
    def test_returns_highest_pending(self, db):
        for n in (1, 3, 5):
            db.execute(
                "INSERT INTO slots (slot_number, family_root, state, created_at) "
                "VALUES (?, '/tmp/family', 'pending', '2026-01-01')", (n,))
        db.commit()
        result = wl.find_reusable_slot(db, "/tmp/family")
        assert result is not None
        highest, others = result
        assert highest == 5
        assert sorted(others) == [1, 3]

    def test_includes_failed_slots(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (10, '/tmp/family', 'failed', '2026-01-01')")
        db.commit()
        result = wl.find_reusable_slot(db, "/tmp/family")
        assert result is not None
        assert result[0] == 10

    def test_returns_none_when_no_pending(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (1, '/tmp/family', 'active', '2026-01-01')")
        db.commit()
        assert wl.find_reusable_slot(db, "/tmp/family") is None

    def test_ignores_active_slots(self, db):
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (1, '/tmp/family', 'active', '2026-01-01')")
        db.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (2, '/tmp/family', 'pending', '2026-01-01')")
        db.commit()
        result = wl.find_reusable_slot(db, "/tmp/family")
        assert result is not None
        assert result[0] == 2
        assert result[1] == []
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_worklog_slot_guards.py::TestFindReusableSlot -v`
Expected: FAIL with `AttributeError: module 'worklog' has no attribute 'find_reusable_slot'`

- [ ] **Step 7: Implement `find_reusable_slot`**

Add to `scripts/worklog.py` after `fail_slot`:

```python
def find_reusable_slot(conn: sqlite3.Connection,
                       family_root: str) -> tuple[int, list[int]] | None:
    """Find reusable pending/failed slots for a family_root.
    Returns (highest_number, [other_numbers]) or None."""
    family_root = _norm(family_root)
    rows = conn.execute(
        "SELECT slot_number FROM slots "
        "WHERE family_root=? AND state IN ('pending', 'failed') "
        "ORDER BY slot_number DESC",
        (family_root,),
    ).fetchall()
    if not rows:
        return None
    highest = rows[0][0]
    others = [r[0] for r in rows[1:]]
    return highest, others
```

- [ ] **Step 8: Run all tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog_slot_guards.py -v`
Expected: 7 PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/worklog.py tests/test_worklog_slot_guards.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: add fail_slot and find_reusable_slot to worklog Refs #291"
```

---

## Batch 2: Slot manager — guard, rollback, reuse, ghost filter

### Task 2: Add `SlotCreationError`, `find_slot_by_branch`, and ghost filter

**Files:**
- Modify: `work-slot/slot_manager.py` (add exception class, guard function, list_slots filter)
- Modify: `tests/test_slot_manager.py` (add test classes)

**Interfaces:**
- Produces: `SlotCreationError(Exception)` — raised on guard failures and creation errors
- Produces: `find_slot_by_branch(family_root, branch)` → `tuple[int, bool] | None` — `(slot_num, is_landed)` or `None`

- [ ] **Step 1: Write failing tests for `find_slot_by_branch`**

Append to `tests/test_slot_manager.py`:

```python
class TestFindSlotByBranch:
    def test_finds_match(self, tmp_path):
        slots_dir = tmp_path / "slots" / "1"
        slots_dir.mkdir(parents=True)
        (slots_dir / ".slot").write_text("# Slot 1 — issue-42-feature\n\n## Repos\n- myrepo\n")
        result = slot_manager.find_slot_by_branch(tmp_path, "issue-42-feature")
        assert result is not None
        assert result == (1, False)

    def test_returns_landed_flag(self, tmp_path):
        slots_dir = tmp_path / "slots" / "1"
        slots_dir.mkdir(parents=True)
        (slots_dir / ".slot").write_text("# Slot 1 — issue-42-feature\n\n## Repos\n- myrepo\n")
        (slots_dir / ".landed").write_text("landed_shas=myrepo:abc123\n")
        result = slot_manager.find_slot_by_branch(tmp_path, "issue-42-feature")
        assert result == (1, True)

    def test_no_match(self, tmp_path):
        slots_dir = tmp_path / "slots" / "1"
        slots_dir.mkdir(parents=True)
        (slots_dir / ".slot").write_text("# Slot 1 — issue-42-feature\n\n## Repos\n- myrepo\n")
        assert slot_manager.find_slot_by_branch(tmp_path, "other-branch") is None

    def test_ignores_attic(self, tmp_path):
        attic = tmp_path / "slots" / "attic" / "1"
        attic.mkdir(parents=True)
        (attic / ".slot").write_text("# Slot 1 — issue-42-feature\n\n## Repos\n- myrepo\n")
        assert slot_manager.find_slot_by_branch(tmp_path, "issue-42-feature") is None

    def test_ignores_ghost_dirs(self, tmp_path):
        slots_dir = tmp_path / "slots" / "1"
        slots_dir.mkdir(parents=True)
        # No .slot file — ghost directory
        assert slot_manager.find_slot_by_branch(tmp_path, "anything") is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestFindSlotByBranch -v`
Expected: FAIL with `AttributeError`

- [ ] **Step 3: Implement `SlotCreationError` and `find_slot_by_branch`**

Add to `work-slot/slot_manager.py` after the constants (around line 57):

```python
class SlotCreationError(Exception):
    """Raised when slot creation fails at any point."""
    pass
```

Add `find_slot_by_branch` after `_resolve_slot_dir_for_number` (around line 78):

```python
def find_slot_by_branch(family_root: Path, branch: str) -> tuple[int, bool] | None:
    """Check if an active slot already uses this branch name.
    Returns (slot_number, is_landed) or None."""
    for dir_name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
        slots_dir = family_root / dir_name
        if not slots_dir.exists():
            continue
        for d in sorted(slots_dir.iterdir()):
            if not d.is_dir() or not d.name.isdigit() or d.name == "attic":
                continue
            info = parse_slot_md(d)
            if info.get("branch") == branch:
                landed = (d / ".landed").exists()
                return int(d.name), landed
    return None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestFindSlotByBranch -v`
Expected: 5 PASS

- [ ] **Step 5: Write failing test for `list_slots` ghost filter**

Append to `tests/test_slot_manager.py`:

```python
class TestListSlotsGhostFilter:
    def test_skips_ghost_directories(self, tmp_path, monkeypatch):
        monkeypatch.setattr(slot_manager, "_wl", None)
        # Real slot with .slot file
        real = tmp_path / "slots" / "1"
        real.mkdir(parents=True)
        (real / ".slot").write_text("# Slot 1 — issue-1-real\n\n## Repos\n- myrepo\n")
        init_repo(real / "myrepo")
        # Ghost directory with no .slot
        ghost = tmp_path / "slots" / "2"
        ghost.mkdir(parents=True)
        (ghost / "somedir").mkdir()

        slots = slot_manager.list_slots(tmp_path)
        nums = [s["number"] for s in slots]
        assert 1 in nums
        assert 2 not in nums
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestListSlotsGhostFilter -v`
Expected: FAIL — ghost slot 2 appears in results

- [ ] **Step 7: Add ghost filter to `list_slots`**

In `work-slot/slot_manager.py`, inside `list_slots()`, after the line `if not d.is_dir() or not d.name.isdigit():` (around line 1647), add:

```python
            if not (d / ".slot").exists():
                continue
```

- [ ] **Step 8: Run tests to verify all pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestListSlotsGhostFilter tests/test_slot_manager.py::TestFindSlotByBranch -v`
Expected: 6 PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: add duplicate branch guard, SlotCreationError, and list_slots ghost filter Refs #291"
```

### Task 3: Refactor `create_slot` error paths and add rollback

**Files:**
- Modify: `work-slot/slot_manager.py` (refactor `create_slot`, update `allocate_slot_number`, update `main`)
- Modify: `tests/test_slot_manager.py` (add rollback and reuse tests)

**Interfaces:**
- Consumes: `SlotCreationError` from Task 2
- Consumes: `find_slot_by_branch` from Task 2
- Consumes: `fail_slot`, `find_reusable_slot` from Task 1

- [ ] **Step 1: Write failing tests for duplicate guard integration**

Append to `tests/test_slot_manager.py`:

```python
class TestCreateSlotDuplicateGuard:
    def _setup_db(self, tmp_path, monkeypatch):
        scripts_dir = Path(__file__).parent.parent / "scripts"
        sys.path.insert(0, str(scripts_dir))
        import worklog as _wl_mod
        db_path = tmp_path / "guard_test.db"
        monkeypatch.setattr(slot_manager, "_wl", _wl_mod)
        monkeypatch.setattr(_wl_mod, "DEFAULT_DB", str(db_path))
        return _wl_mod

    def test_duplicate_branch_raises(self, tmp_path, monkeypatch):
        self._setup_db(tmp_path, monkeypatch)
        family = tmp_path / "family"
        family.mkdir()
        repo = init_repo(family / "myrepo")
        # Create a fake existing slot with .slot file
        existing = family / "slots" / "1"
        existing.mkdir(parents=True)
        (existing / ".slot").write_text("# Slot 1 — my-branch\n\n## Repos\n- myrepo\n")
        with pytest.raises(slot_manager.SlotCreationError, match="already has branch"):
            slot_manager.create_slot(family, ["myrepo"], "my-branch",
                                     issue="1", issue_repo="org/repo",
                                     covers="1", context="test")

    def test_duplicate_landed_branch_message(self, tmp_path, monkeypatch):
        self._setup_db(tmp_path, monkeypatch)
        family = tmp_path / "family"
        family.mkdir()
        init_repo(family / "myrepo")
        existing = family / "slots" / "1"
        existing.mkdir(parents=True)
        (existing / ".slot").write_text("# Slot 1 — my-branch\n\n## Repos\n- myrepo\n")
        (existing / ".landed").write_text("landed_shas=myrepo:abc\n")
        with pytest.raises(slot_manager.SlotCreationError, match="landed.*Archive it"):
            slot_manager.create_slot(family, ["myrepo"], "my-branch",
                                     issue="1", issue_repo="org/repo",
                                     covers="1", context="test")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotDuplicateGuard -v`
Expected: FAIL — `create_slot` doesn't raise `SlotCreationError`

- [ ] **Step 3: Write failing tests for rollback**

Append to `tests/test_slot_manager.py`:

```python
class TestCreateSlotRollback:
    def _setup_db(self, tmp_path, monkeypatch):
        scripts_dir = Path(__file__).parent.parent / "scripts"
        sys.path.insert(0, str(scripts_dir))
        import worklog as _wl_mod
        db_path = tmp_path / "rollback_test.db"
        monkeypatch.setattr(slot_manager, "_wl", _wl_mod)
        monkeypatch.setattr(_wl_mod, "DEFAULT_DB", str(db_path))
        return _wl_mod

    def test_clone_failure_cleans_up_dir(self, tmp_path, monkeypatch):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        family = tmp_path / "family"
        family.mkdir()
        # No repo exists — clone will fail
        with pytest.raises(slot_manager.SlotCreationError):
            slot_manager.create_slot(family, ["nonexistent"], "test-branch",
                                     issue="1", issue_repo="org/repo",
                                     covers="1", context="test")
        # Verify no slot directory remains
        slots_dir = family / "slots"
        remaining = [d for d in slots_dir.iterdir() if d.is_dir() and d.name.isdigit()] if slots_dir.exists() else []
        assert len(remaining) == 0

    def test_clone_failure_transitions_db_to_failed(self, tmp_path, monkeypatch):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        family = tmp_path / "family"
        family.mkdir()
        with pytest.raises(slot_manager.SlotCreationError):
            slot_manager.create_slot(family, ["nonexistent"], "test-branch",
                                     issue="1", issue_repo="org/repo",
                                     covers="1", context="test")
        conn = _wl_mod.connect()
        row = conn.execute(
            "SELECT state FROM slots WHERE family_root=?",
            (_wl_mod._norm(str(family)),)
        ).fetchone()
        conn.close()
        assert row is not None
        assert row["state"] == "failed"
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotRollback -v`
Expected: FAIL — `create_slot` calls `sys.exit` instead of raising

- [ ] **Step 5: Write failing tests for pending reuse**

Append to `tests/test_slot_manager.py`:

```python
class TestAllocateSlotReuse:
    def _setup_db(self, tmp_path, monkeypatch):
        scripts_dir = Path(__file__).parent.parent / "scripts"
        sys.path.insert(0, str(scripts_dir))
        import worklog as _wl_mod
        db_path = tmp_path / "reuse_test.db"
        monkeypatch.setattr(slot_manager, "_wl", _wl_mod)
        monkeypatch.setattr(_wl_mod, "DEFAULT_DB", str(db_path))
        return _wl_mod

    def test_reuses_pending(self, tmp_path, monkeypatch, capsys):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        conn = _wl_mod.connect()
        conn.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (5, ?, 'pending', '2026-01-01')",
            (_wl_mod._norm(str(tmp_path)),))
        conn.commit()
        conn.close()
        result = slot_manager.allocate_slot_number(tmp_path)
        assert result == 5
        captured = capsys.readouterr()
        assert "REUSED_PENDING=5" in captured.out

    def test_reuses_failed(self, tmp_path, monkeypatch, capsys):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        conn = _wl_mod.connect()
        conn.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (7, ?, 'failed', '2026-01-01')",
            (_wl_mod._norm(str(tmp_path)),))
        conn.commit()
        conn.close()
        result = slot_manager.allocate_slot_number(tmp_path)
        assert result == 7

    def test_cleans_debris_on_reuse(self, tmp_path, monkeypatch):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        conn = _wl_mod.connect()
        conn.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (3, ?, 'pending', '2026-01-01')",
            (_wl_mod._norm(str(tmp_path)),))
        conn.commit()
        conn.close()
        debris = tmp_path / "slots" / "3"
        debris.mkdir(parents=True)
        (debris / ".m2").mkdir()
        slot_manager.allocate_slot_number(tmp_path)
        assert not debris.exists()

    def test_cleans_older_pending_slots(self, tmp_path, monkeypatch):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        conn = _wl_mod.connect()
        for n in (1, 3, 5):
            conn.execute(
                "INSERT INTO slots (slot_number, family_root, state, created_at) "
                "VALUES (?, ?, 'pending', '2026-01-01')",
                (n, _wl_mod._norm(str(tmp_path))))
        conn.commit()
        conn.close()
        result = slot_manager.allocate_slot_number(tmp_path)
        assert result == 5
        conn = _wl_mod.connect()
        states = {r["slot_number"]: r["state"] for r in conn.execute(
            "SELECT slot_number, state FROM slots WHERE family_root=?",
            (_wl_mod._norm(str(tmp_path)),)).fetchall()}
        conn.close()
        assert states[5] == "pending"  # reused, still pending until confirmed
        assert states[1] == "failed"
        assert states[3] == "failed"

    def test_fresh_when_no_pending(self, tmp_path, monkeypatch):
        _wl_mod = self._setup_db(tmp_path, monkeypatch)
        conn = _wl_mod.connect()
        conn.execute(
            "INSERT INTO slots (slot_number, family_root, state, created_at) "
            "VALUES (10, ?, 'active', '2026-01-01')",
            (_wl_mod._norm(str(tmp_path)),))
        conn.commit()
        conn.close()
        result = slot_manager.allocate_slot_number(tmp_path)
        assert result == 11
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestAllocateSlotReuse -v`
Expected: FAIL — `allocate_slot_number` doesn't check for pending slots

- [ ] **Step 7: Refactor `create_slot` — replace `sys.exit` with exceptions, add guard and rollback**

This is the main refactoring step. In `work-slot/slot_manager.py`:

1. At the top of `create_slot()`, add the duplicate branch guard:
```python
result = find_slot_by_branch(family_root, branch)
if result is not None:
    existing_num, landed = result
    if landed:
        raise SlotCreationError(
            f"Slot {existing_num} has branch `{branch}` (landed, not yet archived). "
            f"Archive it first.")
    raise SlotCreationError(
        f"Slot {existing_num} already has branch `{branch}`. "
        f"Use that slot or archive it first.")
```

2. Replace every `sys.exit(1)` inside `create_slot` with `raise SlotCreationError(message)`. The error message should include the `ERROR=` and `ERROR_DETAIL=` info from the print statements preceding the exit.

3. Wrap the body (after guard, after `allocate_slot_number`) in try/except:
```python
slot_num = allocate_slot_number(family_root)
slot_dir = slots_dir / str(slot_num)
try:
    slot_dir.mkdir()
    # ... rest of creation logic ...
    return {... }
except Exception:
    if slot_dir.exists():
        shutil.rmtree(str(slot_dir), ignore_errors=True)
    if _wl:
        try:
            conn = _wl.connect()
            _wl.fail_slot(conn, slot_num, str(family_root))
            conn.close()
        except Exception:
            pass
    raise
```

4. Update `main()` to catch `SlotCreationError`:
```python
# In main(), around the create-slot subcommand handler:
try:
    result = create_slot(...)
except SlotCreationError as e:
    print(f"ERROR={e}")
    sys.exit(1)
```

- [ ] **Step 8: Refactor `allocate_slot_number` — add pending reuse**

Replace the body of `allocate_slot_number()` with:

```python
def allocate_slot_number(family_root: Path) -> int:
    if _wl is None:
        print("ERROR=worklog_unavailable")
        print("ERROR_DETAIL=worklog module required for slot numbering — "
              "ensure scripts/worklog.py is importable")
        sys.exit(1)
    conn = _wl.connect()
    try:
        reusable = _wl.find_reusable_slot(conn, str(family_root))
        if reusable is not None:
            slot_num, others = reusable
            for dir_name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
                debris = family_root / dir_name / str(slot_num)
                if debris.exists():
                    shutil.rmtree(str(debris), ignore_errors=True)
            for other_num in others:
                for dir_name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
                    debris = family_root / dir_name / str(other_num)
                    if debris.exists():
                        shutil.rmtree(str(debris), ignore_errors=True)
                _wl.fail_slot(conn, other_num, str(family_root))
            print(f"REUSED_PENDING={slot_num}")
            return slot_num
        slot_num = _wl.reserve_slot_number(conn, str(family_root))
    finally:
        conn.close()
    return slot_num
```

- [ ] **Step 9: Run all tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: all tests PASS including the new guard, rollback, and reuse tests

- [ ] **Step 10: Run existing test suite to check for regressions**

Run: `python3 -m pytest tests/ -v --timeout=60 -x`
Expected: no regressions — existing tests pass

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: create_slot rollback, pending reuse, duplicate guard integration Refs #291"
```

---

## Batch 3: Reconcile enhancements — content-aware quarantine

### Task 4: Enhance reconcile quarantine with content checking and `.claude/projects` relocation

**Files:**
- Modify: `scripts/reconcile_slots.py` (enhance `strategy` and `execute`)
- Modify: `tests/test_reconcile_slots.py` (add new test cases)

**Interfaces:**
- Consumes: `relocate_claude_projects` from `slot_manager.py` (already imported)

- [ ] **Step 1: Write failing tests for content-aware strategy**

Append to `tests/test_reconcile_slots.py`:

```python
class TestQuarantineContentCheck:
    def test_ghost_with_commits_reported_in_strategy(self, tmp_path):
        family = tmp_path / "family"
        slots_dir = family / "slots"
        ghost = slots_dir / "54"
        ghost.mkdir(parents=True)
        repo = ghost / "myrepo"
        repo.mkdir()
        (repo / ".git").mkdir()
        # Simulate repo with content
        (repo / "file.txt").write_text("work in progress")

        divergences = [{
            "slot": 54,
            "class": "ghost",
            "disk_path": str(ghost),
            "disk_contents": ["myrepo"],
            "db_state": None,
            "detail": "directory with no .slot file",
        }]
        actions = reconcile_slots.strategy(divergences)
        assert len(actions) == 1
        assert actions[0]["action"] == "quarantine"
        assert "content" in actions[0]

    def test_empty_ghost_reported_as_empty(self, tmp_path):
        family = tmp_path / "family"
        ghost = family / "slots" / "55"
        ghost.mkdir(parents=True)

        divergences = [{
            "slot": 55,
            "class": "ghost",
            "disk_path": str(ghost),
            "disk_contents": [],
            "db_state": None,
            "detail": "directory with no .slot file",
        }]
        actions = reconcile_slots.strategy(divergences)
        assert len(actions) == 1
        assert actions[0]["content"] == "empty"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_reconcile_slots.py::TestQuarantineContentCheck -v`
Expected: FAIL — `strategy` output has no `content` key

- [ ] **Step 3: Enhance `strategy` function**

In `scripts/reconcile_slots.py`, update the ghost handler in `strategy()`:

```python
if cls == "ghost":
    contents = d.get("disk_contents", [])
    has_content = len(contents) > 0
    db_state = d.get("db_state")
    content_summary = "empty"
    if has_content:
        content_summary = f"contains: {', '.join(contents)}"
        if db_state:
            content_summary += f", DB state: {db_state}"
    actions.append({
        "slot": d["slot"],
        "action": "quarantine",
        "source": d["disk_path"],
        "content": content_summary,
        "detail": f"move to quarantine/ — {content_summary}",
        "risk": "medium" if has_content else "low",
    })
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_reconcile_slots.py::TestQuarantineContentCheck -v`
Expected: 2 PASS

- [ ] **Step 5: Write failing test for `.claude/projects` relocation during quarantine**

Append to `tests/test_reconcile_slots.py`:

```python
class TestQuarantineClaudeProjects:
    def test_relocates_claude_projects(self, tmp_path, monkeypatch):
        family = tmp_path / "family"
        ghost_dir = family / "slots" / "99"
        ghost_dir.mkdir(parents=True)
        quarantine_dir = family / "slots" / "quarantine"

        # Mock relocate_claude_projects to track calls
        calls = []
        monkeypatch.setattr(reconcile_slots, "relocate_claude_projects",
                            lambda src, dst: calls.append((str(src), str(dst))) or 0)

        actions = [{
            "slot": 99,
            "action": "quarantine",
            "source": str(ghost_dir),
            "content": "empty",
            "detail": "test",
            "risk": "low",
        }]
        results = reconcile_slots.execute(actions, family)
        assert len(results) == 1
        assert results[0]["status"] == "done"
        assert len(calls) == 1
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_reconcile_slots.py::TestQuarantineClaudeProjects -v`
Expected: FAIL — quarantine execute doesn't call `relocate_claude_projects`

- [ ] **Step 7: Add `.claude/projects` relocation to quarantine execute**

In `scripts/reconcile_slots.py`, update the quarantine handler in `execute()`:

```python
if a["action"] == "quarantine":
    quarantine_dir = family_root / "slots" / "quarantine"
    quarantine_dir.mkdir(parents=True, exist_ok=True)
    dest = quarantine_dir / str(a["slot"])
    if dest.exists():
        results.append({"slot": a["slot"], "action": a["action"],
                        "status": "skipped", "detail": "quarantine dest exists"})
        continue
    shutil.move(a["source"], str(dest))
    if relocate_claude_projects:
        relocate_claude_projects(Path(a["source"]), dest)
    results.append({"slot": a["slot"], "action": a["action"], "status": "done"})
```

- [ ] **Step 8: Run all reconcile tests**

Run: `python3 -m pytest tests/test_reconcile_slots.py -v`
Expected: all PASS

- [ ] **Step 9: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=60 -x`
Expected: no regressions

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/reconcile_slots.py tests/test_reconcile_slots.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: content-aware quarantine with claude projects relocation Refs #291"
```

---

## Batch 4: Sync and verify

### Task 5: Sync worklog.py, update _states_compatible, run full verification

**Files:**
- Modify: `scripts/reconcile_slots.py` (`_states_compatible` to handle `failed` state)
- Modify: `work-slot/slot_manager.py` (`_map_db_to_disk_state` to handle `failed` state)

- [ ] **Step 1: Update `_states_compatible` in reconcile_slots.py**

Add `failed` handling:

```python
def _states_compatible(db_state: str, disk_state: str) -> bool:
    if db_state == disk_state:
        return True
    if db_state == "pending" and disk_state == "active":
        return True
    if db_state == "failed":
        return True  # failed slots have no expected disk state
    if db_state == "ready" and disk_state in ("active", "ready"):
        return True
    return False
```

- [ ] **Step 2: Update `_map_db_to_disk_state` in slot_manager.py**

Add `failed` to the mapping:

```python
def _map_db_to_disk_state(db_state: str) -> str:
    mapping = {
        "active": "active",
        "pending": "active",
        "failed": "failed",
        "ready": "ready to land",
        "landed": "landed",
        "archived": "archived",
    }
    return mapping.get(db_state, db_state)
```

- [ ] **Step 3: Sync worklog.py to ~/.claude/lib/**

```bash
cp /Users/mdproctor/claude/hortora/soredium/scripts/worklog.py /Users/mdproctor/.claude/lib/worklog.py
```

- [ ] **Step 4: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=60`
Expected: all tests pass

- [ ] **Step 5: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: no CRITICAL findings

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/reconcile_slots.py work-slot/slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: handle failed state in reconcile compatibility and drift mapping Refs #291"
```

## References

- [2026-08-26-duplicate-slot-guard-design.md] — design spec this plan implements
- [work-slot/slot_manager.py] — primary implementation target (create_slot, allocate_slot_number, list_slots)
- [scripts/worklog.py] — DB layer (reserve_slot_number, confirm_slot_create, slots table)
- [scripts/reconcile_slots.py] — existing quarantine flow (audit, strategy, execute)
- [tests/test_slot_manager.py] — existing slot tests
- [tests/test_reconcile_slots.py] — existing reconcile tests
- [docs/protocols/externalised-scripts-require-tests.md] — tests required in same commit
- [GitHub #291] — Guard against duplicate slot creation and clean up partial failures

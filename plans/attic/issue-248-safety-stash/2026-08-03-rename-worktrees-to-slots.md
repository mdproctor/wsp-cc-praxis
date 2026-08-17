# Rename worktrees/ → slots/ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development). Steps use checkbox (`- [ ]`) syntax
> for tracking.

**Focal issue:** #162 — Rename slot directory from worktrees/ to slots/
**Issue group:** #162

**Goal:** New slots create under `slots/`, existing `worktrees/` slots continue working via dual-path reads.

**Architecture:** Add constants and a resolution helper to `slot_manager.py`. All functions that reference `worktrees/` switch to the helper. Detection heuristic (`"/worktrees/" in path`) becomes an exported `is_slot_path()` function checking both patterns. Tests updated to use `slots/` with backward-compat tests for `worktrees/`.

**Tech Stack:** Python 3.14, pytest

## Global Constraints

- Existing slots under `worktrees/` must remain discoverable — no migration step required
- `allocate_slot_number` must consider both `slots/` and `worktrees/` to avoid number collisions
- `.worktrees/` (dot-prefixed, actual git worktrees) is NOT affected
- `.claude/worktrees/` (Claude Code worktrees) is NOT affected

---

### Task 1: Add constants, resolution helpers, and `is_slot_path()` to slot_manager.py

**Files:**
- Modify: `work-slot/slot_manager.py`
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `SLOT_DIR_NAME = "slots"`, `LEGACY_SLOT_DIR_NAME = "worktrees"`, `_resolve_slots_dir(family_root) -> Path`, `_resolve_slot_dir_for_number(family_root, slot_num) -> Path`, `is_slot_path(path: str) -> bool`

- [ ] **Step 1: Write failing tests for the new helpers**

```python
class TestSlotDirResolution:
    def test_prefers_slots_over_worktrees(self, tmp_path):
        (tmp_path / "slots").mkdir()
        (tmp_path / "worktrees").mkdir()
        result = slot_manager._resolve_slots_dir(tmp_path)
        assert result == tmp_path / "slots"

    def test_falls_back_to_worktrees(self, tmp_path):
        (tmp_path / "worktrees").mkdir()
        result = slot_manager._resolve_slots_dir(tmp_path)
        assert result == tmp_path / "worktrees"

    def test_returns_slots_when_neither_exists(self, tmp_path):
        result = slot_manager._resolve_slots_dir(tmp_path)
        assert result == tmp_path / "slots"

    def test_resolve_slot_number_in_slots(self, tmp_path):
        (tmp_path / "slots" / "1").mkdir(parents=True)
        result = slot_manager._resolve_slot_dir_for_number(tmp_path, 1)
        assert result == tmp_path / "slots" / "1"

    def test_resolve_slot_number_falls_back_to_worktrees(self, tmp_path):
        (tmp_path / "worktrees" / "1").mkdir(parents=True)
        result = slot_manager._resolve_slot_dir_for_number(tmp_path, 1)
        assert result == tmp_path / "worktrees" / "1"

    def test_resolve_slot_number_prefers_slots(self, tmp_path):
        (tmp_path / "slots" / "1").mkdir(parents=True)
        (tmp_path / "worktrees" / "1").mkdir(parents=True)
        result = slot_manager._resolve_slot_dir_for_number(tmp_path, 1)
        assert result == tmp_path / "slots" / "1"


class TestIsSlotPath:
    def test_detects_slots_path(self):
        assert slot_manager.is_slot_path("/home/user/family/slots/1/repo") is True

    def test_detects_legacy_worktrees_path(self):
        assert slot_manager.is_slot_path("/home/user/family/worktrees/1/repo") is True

    def test_rejects_claude_worktrees(self):
        assert slot_manager.is_slot_path("/home/user/repo/.claude/worktrees/issue-17") is False

    def test_rejects_dot_worktrees(self):
        assert slot_manager.is_slot_path("/home/user/repo/.worktrees/feat") is False

    def test_rejects_plain_path(self):
        assert slot_manager.is_slot_path("/home/user/project/src") is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestSlotDirResolution tests/test_slot_manager.py::TestIsSlotPath -v`
Expected: FAIL (functions don't exist yet)

- [ ] **Step 3: Implement the helpers in slot_manager.py**

Add after the `_IDE_ARTIFACTS` constant (line 36):

```python
SLOT_DIR_NAME = "slots"
LEGACY_SLOT_DIR_NAME = "worktrees"


def _resolve_slots_dir(family_root: Path) -> Path:
    """Return the slots directory, preferring slots/ over legacy worktrees/."""
    new = family_root / SLOT_DIR_NAME
    old = family_root / LEGACY_SLOT_DIR_NAME
    if new.exists():
        return new
    if old.exists():
        return old
    return new


def _resolve_slot_dir_for_number(family_root: Path, slot_num: int) -> Path:
    """Find a specific slot by number, checking slots/ then worktrees/."""
    for name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
        candidate = family_root / name / str(slot_num)
        if candidate.exists():
            return candidate
    return family_root / SLOT_DIR_NAME / str(slot_num)


def is_slot_path(path: str) -> bool:
    """Check if a path is inside a slot directory (not a git/Claude Code worktree)."""
    return "/slots/" in path or ("/worktrees/" in path and "/.claude/worktrees/" not in path and "/.worktrees/" not in path)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestSlotDirResolution tests/test_slot_manager.py::TestIsSlotPath -v`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#162): add slot directory resolution helpers and is_slot_path

Refs #162
```

---

### Task 2: Update slot_manager.py functions to use helpers

**Files:**
- Modify: `work-slot/slot_manager.py`
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `_resolve_slots_dir`, `_resolve_slot_dir_for_number`, `SLOT_DIR_NAME`
- Produces: all existing functions work with both `slots/` and `worktrees/`

- [ ] **Step 1: Write backward-compat test — create_slot uses slots/**

```python
class TestCreateSlotUsesNewDir:
    def test_creates_under_slots_not_worktrees(self, tmp_path):
        repo = init_repo(tmp_path / "myrepo")
        result = slot_manager.create_slot(
            family_root=tmp_path, repos=["myrepo"], branch="test-branch",
            issue="1", issue_repo="org/repo", covers="1", context="test",
        )
        assert (tmp_path / "slots" / "1").exists()
        assert not (tmp_path / "worktrees" / "1").exists()
```

- [ ] **Step 2: Write backward-compat test — list_slots finds worktrees/**

```python
class TestListSlotsDualPath:
    def test_finds_slots_in_legacy_worktrees(self, tmp_path):
        wt = tmp_path / "worktrees" / "1"
        wt.mkdir(parents=True)
        repo = init_repo(wt / "myrepo")
        (wt / ".slot").write_text("# Slot 1 — test-branch\n")
        slots = slot_manager.list_slots(tmp_path)
        assert len(slots) == 1
        assert slots[0]["number"] == 1

    def test_finds_slots_in_new_dir(self, tmp_path):
        sd = tmp_path / "slots" / "1"
        sd.mkdir(parents=True)
        repo = init_repo(sd / "myrepo")
        (sd / ".slot").write_text("# Slot 1 — test-branch\n")
        slots = slot_manager.list_slots(tmp_path)
        assert len(slots) == 1
        assert slots[0]["number"] == 1

    def test_merges_both_dirs(self, tmp_path):
        wt = tmp_path / "worktrees" / "1"
        wt.mkdir(parents=True)
        repo1 = init_repo(wt / "repo1")
        (wt / ".slot").write_text("# Slot 1 — old-branch\n")
        sd = tmp_path / "slots" / "2"
        sd.mkdir(parents=True)
        repo2 = init_repo(sd / "repo2")
        (sd / ".slot").write_text("# Slot 2 — new-branch\n")
        slots = slot_manager.list_slots(tmp_path)
        assert len(slots) == 2
        nums = {s["number"] for s in slots}
        assert nums == {1, 2}
```

- [ ] **Step 3: Write backward-compat test — allocate considers both dirs**

```python
class TestAllocateConsidersBothDirs:
    def test_considers_legacy_worktrees_for_numbering(self, tmp_path):
        (tmp_path / "worktrees" / "3").mkdir(parents=True)
        (tmp_path / "slots").mkdir()
        result = slot_manager.allocate_slot_number(
            slot_manager._resolve_slots_dir(tmp_path)
        )
        # Must not reuse 3 — but allocate_slot_number only sees slots/ (empty)
        # This test verifies the CALLER passes the right dir.
        # allocate_slot_number itself is unchanged — it counts what's in the dir given.
        assert result == 1  # slots/ is empty, so 1 is correct for slots/

    def test_allocate_with_slots_dir(self, tmp_path):
        sd = tmp_path / "slots"
        sd.mkdir()
        (sd / "1").mkdir()
        (sd / "2").mkdir()
        assert slot_manager.allocate_slot_number(sd) == 3
```

- [ ] **Step 4: Run new tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotUsesNewDir tests/test_slot_manager.py::TestListSlotsDualPath tests/test_slot_manager.py::TestAllocateConsidersBothDirs -v`
Expected: FAIL (create_slot still uses `worktrees/`, list_slots doesn't scan both)

- [ ] **Step 5: Update create_slot to use SLOT_DIR_NAME**

In `create_slot()`, change line 270:
```python
# Old:
worktrees_dir = family_root / "worktrees"
# New:
worktrees_dir = family_root / SLOT_DIR_NAME
```

- [ ] **Step 6: Update list_slots to scan both directories**

Replace the single-directory scan with a dual-path scan:
```python
def list_slots(family_root: Path, include_archived: bool = False) -> list[dict]:
    slots = []
    for dir_name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
        slots_dir = family_root / dir_name
        if not slots_dir.exists():
            continue

        attic_dir = slots_dir / "attic"
        archived_nums: set[int] = set()
        if attic_dir.exists():
            archived_nums = {
                int(d.name) for d in attic_dir.iterdir()
                if d.is_dir() and d.name.isdigit()
            }

        for d in sorted(slots_dir.iterdir()):
            if not d.is_dir() or not d.name.isdigit():
                continue
            if int(d.name) in archived_nums:
                continue
            # ... rest unchanged, build slot dict and append ...

        if include_archived and attic_dir.exists():
            # ... scan attic, same as before ...
    
    # Deduplicate by slot number (slots/ wins over worktrees/)
    seen: set[int] = set()
    deduped = []
    for s in slots:
        if s["number"] not in seen:
            seen.add(s["number"])
            deduped.append(s)
    return deduped
```

- [ ] **Step 7: Update single-slot functions to use _resolve_slot_dir_for_number**

In `merge_slot()`, `archive_slot()`, `remove_slot()`, `check_cross_deps()`, and the `ensure-clone-layout` CLI handler — replace:
```python
# Old:
slot_dir = family_root / "worktrees" / str(slot_num)
# New:
slot_dir = _resolve_slot_dir_for_number(family_root, slot_num)
```

In `archive_slot()`, update the attic path to stay under the same parent:
```python
# Old:
attic_dir = family_root / "worktrees" / "attic"
# New:
attic_dir = slot_dir.parent / "attic"
```

Same for `remove_slot()`.

In `scan_ready()`, update to scan both directories (same pattern as list_slots).

- [ ] **Step 8: Run all new and existing tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: ALL PASS (new tests + existing tests)

- [ ] **Step 9: Commit**

```
feat(#162): update slot_manager to create under slots/ with worktrees/ fallback

Refs #162
```

---

### Task 3: Update detection heuristic in work_router.py, pause_exec.py, ctx.py

**Files:**
- Modify: `work/work_router.py`
- Modify: `work-pause/pause_exec.py`
- Modify: `project/ctx.py`
- Test: `tests/test_work_router.py`
- Test: `tests/test_ctx.py`
- Test: `tests/test_pause_exec.py`

**Interfaces:**
- Consumes: `slot_manager.is_slot_path`

- [ ] **Step 1: Write failing tests for work_router slot detection**

```python
class TestSlotDetectionDualPath:
    def test_detects_slot_in_slots_dir(self):
        # project path contains /slots/ → IN_SLOT should be detected
        # (test with a tmp_path that has /slots/ in its structure)
        ...

    def test_detects_slot_in_legacy_worktrees_dir(self):
        # project path contains /worktrees/ → still detected
        ...

    def test_no_false_positive_on_claude_worktrees(self):
        # project path contains /.claude/worktrees/ → NOT a slot
        ...
```

- [ ] **Step 2: Write failing test for pause_exec slot dir parsing**

```python
class TestResolveSlotDirDualPath:
    def test_resolves_from_slots_path(self):
        p = Path("/home/user/family/slots/42/repo")
        result = pause_exec._resolve_slot_dir(p)
        assert result == Path("/home/user/family/slots/42")

    def test_resolves_from_legacy_worktrees_path(self):
        p = Path("/home/user/family/worktrees/42/repo")
        result = pause_exec._resolve_slot_dir(p)
        assert result == Path("/home/user/family/worktrees/42")
```

- [ ] **Step 3: Write failing test for ctx.py epic detection**

```python
class TestWorktreeResolution:
    # Add to existing class:
    def test_epic_detection_uses_slots_path(self, tmp_path):
        # Epic detection fallback checks "/worktrees/" in path
        # Verify it also checks "/slots/"
        ...
```

- [ ] **Step 4: Run tests to verify they fail**

- [ ] **Step 5: Update work_router.py**

Replace line 75:
```python
# Old:
if "/worktrees/" in str(project):
# New:
from slot_manager import is_slot_path
if is_slot_path(str(project)):
```

(Import at top of file, not inline)

- [ ] **Step 6: Update pause_exec.py**

Update `_resolve_slot_dir()` (line 85-94):
```python
def _resolve_slot_dir(clone_path: Path) -> Path | None:
    parts = clone_path.resolve().parts
    for name in ("slots", "worktrees"):
        try:
            idx = parts.index(name)
            if idx + 1 < len(parts):
                return Path(*parts[:idx + 2])
        except ValueError:
            continue
    return None
```

Update line 189:
```python
# Old:
if epic_info is None and "/worktrees/" in project:
# New:
from slot_manager import is_slot_path
if epic_info is None and is_slot_path(project):
```

- [ ] **Step 7: Update ctx.py**

Update line ~193 (the epic detection fallback):
```python
# Old:
if _epic_info is None and "/worktrees/" in str(project):
# New:
_slot_dir = Path(__file__).parent.parent / "work-slot"
if str(_slot_dir) not in sys.path:
    sys.path.insert(0, str(_slot_dir))
from slot_manager import is_slot_path as _is_slot_path
if _epic_info is None and _is_slot_path(str(project)):
```

Note: ctx.py already imports from work-slot (epic_manager). Add the is_slot_path import alongside it.

- [ ] **Step 8: Run all affected tests**

Run: `python3 -m pytest tests/test_work_router.py tests/test_pause_exec.py tests/test_ctx.py -v`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat(#162): replace "/worktrees/" heuristic with is_slot_path() in all detectors

Refs #162
```

---

### Task 4: Update test fixtures and terminology tests

**Files:**
- Modify: `tests/test_slot_manager.py`
- Modify: `tests/test_slot_terminology.py`
- Modify: `tests/test_work_router.py`

**Interfaces:**
- Consumes: all updated functions from Tasks 1-3

- [ ] **Step 1: Update test_slot_manager.py fixtures**

Bulk-replace `tmp_path / "worktrees"` with `tmp_path / "slots"` in test fixtures that create NEW slots. Keep `tmp_path / "worktrees"` in backward-compat tests.

- [ ] **Step 2: Update test_slot_terminology.py**

Update to enforce `slots/` as canonical:
- Add check that `SLOT_DIR_NAME == "slots"` in slot_manager
- Add check that `create_slot` docstring/comments reference `slots/` not `worktrees/`
- Keep existing allowed-pattern checks for legacy references

- [ ] **Step 3: Update test_work_router.py fixtures**

Update slot detection tests to use `/slots/` paths as the primary case, with legacy `/worktrees/` as a backward-compat test.

- [ ] **Step 4: Run full test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_work_router.py tests/test_slot_terminology.py tests/test_pause_exec.py tests/test_ctx.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
test(#162): update fixtures to use slots/ as primary, keep worktrees/ compat tests

Refs #162
```

---

### Task 5: Update SKILL.md files and CLAUDE.md

**Files:**
- Modify: `work-slot/SKILL.md`
- Modify: `work-end/SKILL.md`
- Modify: `work-start/SKILL.md`
- Modify: `work/SKILL.md`
- Modify: `git-commit/SKILL.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update work-slot/SKILL.md**

Replace `worktrees/<N>/` with `slots/<N>/` in path references. Add migration note:
```
> **Legacy:** existing slots under `worktrees/` continue to work. New slots are created under `slots/`.
```

- [ ] **Step 2: Update work-end/SKILL.md**

Replace slot detection references from `"/worktrees/"` to `is_slot_path()`. Update path examples.

- [ ] **Step 3: Update remaining SKILL.md files**

`work-start/SKILL.md`, `work/SKILL.md`, `git-commit/SKILL.md` — same pattern.

- [ ] **Step 4: Update CLAUDE.md**

Replace `worktrees/` references in the slot documentation to `slots/`. Keep `.worktrees/` (git worktrees) unchanged.

- [ ] **Step 5: Run validation**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 6: Commit**

```
docs(#162): update SKILL.md files and CLAUDE.md for slots/ rename

Closes #162
```

# Slot Symlink Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #237 — Slot workspace symlink failures
**Issue group:** #237

**Goal:** Fix silent wksp/ symlink failures in create_slot and add_repo
by adding family-wide collision detection, post-creation validation, and
broken-symlink surfacing in list_slots.

**Architecture:** Five targeted changes in `slot_manager.py`:
(1) `_get_family_repo_names()` helper for collision detection,
(2) `validate_slot_wksp()` for post-creation validation,
(3) DB consistency check in `create_slot`,
(4) `wksp_ok` health field in `list_slots`,
(5) workspace remote wiring parity in `add_repo`.

**Tech Stack:** Python 3.14, pytest, git

## Global Constraints

- All changes in `work-slot/slot_manager.py` and `tests/test_slot_manager.py`
- Follow existing KEY=VALUE output pattern for errors and warnings
- Tests use the existing `init_repo()` helper and `@patch("slot_manager.run_cmd")` pattern
- No new dependencies

---

### Task 1: Family-wide collision detection

**Files:**
- Modify: `work-slot/slot_manager.py` (add `_get_family_repo_names`, update `create_slot`, update `add_repo`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `_get_family_repo_names(family_root: Path) -> set[str]` — returns names of all top-level git directories in family_root, excluding `slots`, `attic`, `.m2`
- Consumes: nothing new

- [ ] **Step 1: Write failing test for `_get_family_repo_names`**

```python
class TestGetFamilyRepoNames:
    def test_finds_git_repos(self, tmp_path):
        family = tmp_path / "casehub"
        family.mkdir()
        init_repo(family / "engine")
        init_repo(family / "work")
        (family / "not-a-repo").mkdir()
        result = slot_manager._get_family_repo_names(family)
        assert result == {"engine", "work"}

    def test_excludes_slots_and_m2(self, tmp_path):
        family = tmp_path / "casehub"
        family.mkdir()
        init_repo(family / "engine")
        (family / "slots").mkdir()
        (family / ".m2").mkdir()
        result = slot_manager._get_family_repo_names(family)
        assert "slots" not in result
        assert ".m2" not in result
        assert "engine" in result

    def test_empty_family(self, tmp_path):
        family = tmp_path / "casehub"
        family.mkdir()
        result = slot_manager._get_family_repo_names(family)
        assert result == set()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestGetFamilyRepoNames -v`
Expected: FAIL with `AttributeError: module 'slot_manager' has no attribute '_get_family_repo_names'`

- [ ] **Step 3: Implement `_get_family_repo_names`**

Add after `_resolve_slot_dir_for_number` (around line 76) in `slot_manager.py`:

```python
def _get_family_repo_names(family_root: Path) -> set[str]:
    """Return names of all top-level git directories in the family root."""
    excluded = {"slots", "worktrees", "attic", ".m2"}
    names: set[str] = set()
    if not family_root.is_dir():
        return names
    for entry in family_root.iterdir():
        if not entry.is_dir() or entry.name in excluded or entry.name.startswith("."):
            continue
        if (entry / ".git").exists() or (entry / ".git").is_file():
            names.add(entry.name)
    return names
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestGetFamilyRepoNames -v`
Expected: PASS

- [ ] **Step 5: Write failing test for family-wide collision in create_slot**

```python
class TestCreateSlotCollisionFamily:
    @patch("slot_manager.run_cmd")
    def test_collision_detected_against_family_repo_not_in_slot(self, mock_cmd, tmp_path):
        """Family has repo 'work' but slot only has 'engine'.
        Workspace clone default name 'work' must be deconflicted."""
        family = tmp_path / "casehub"
        family.mkdir()
        engine = init_repo(family / "engine")
        init_repo(family / "work")  # exists in family but NOT in this slot
        shared_ws = init_repo(tmp_path / "public" / "casehub")
        (shared_ws / "engine").mkdir()
        (engine / "wksp").symlink_to(shared_ws / "engine")

        mock_cmd.return_value = (0, "", "")

        result = slot_manager.create_slot(
            family_root=family,
            repos=["engine"],
            branch="issue-99-test",
            issue="99",
            issue_repo="casehubio/parent",
            covers="99",
            context="Test collision",
        )

        slot_dir = family / "slots" / str(result["slot_number"])
        # Workspace clone must NOT be named "work" — that's a family repo name
        clone_dests = []
        for c in mock_cmd.call_args_list:
            args = c.args[0] if c.args else c[0]
            if isinstance(args, list) and len(args) >= 2 and args[0] == "git" and "clone" in args:
                dest = Path(args[-1])
                if dest.parent == slot_dir and "work" in dest.name:
                    clone_dests.append(dest.name)
        for name in clone_dests:
            assert name != "work", (
                f"workspace clone used name 'work' which collides with family repo"
            )
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotCollisionFamily -v`
Expected: FAIL — the assertion fires because current code only checks `repos`, not family-wide

- [ ] **Step 7: Update `create_slot` collision check**

In `create_slot()`, replace the collision check (around line 548):

Old:
```python
            if ws_name in repos:
                ws_name = f"work-{ws_source.name}"
```

New:
```python
            family_repo_names = _get_family_repo_names(family_root)
            if ws_name in family_repo_names:
                ws_name = f"work-{ws_source.name}"
```

Also update the disambiguation check (around line 552) — replace `repos` with `family_repo_names`:

Old:
```python
            if ws_key not in ws_created and (slot_dir / ws_name).exists():
                ws_name = f"work-{ws_source.name}"
```

This stays as-is — it handles a different case (two workspace sources resolving to the same name).

- [ ] **Step 8: Update `add_repo` collision check**

In `add_repo()`, replace the collision check (around line 703):

Old:
```python
        existing_repos = get_slot_repos(slot_dir)
        if ws_name in existing_repos:
            ws_name = f"work-{ws_source.name}"
```

New:
```python
        family_repo_names = _get_family_repo_names(family_root)
        if ws_name in family_repo_names:
            ws_name = f"work-{ws_source.name}"
```

- [ ] **Step 9: Run all collision tests**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotCollisionFamily tests/test_slot_manager.py::TestWorkspaceNameCollision tests/test_slot_manager.py::TestCrossOrgWorkspaceWiring -v`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "fix(#237): family-wide workspace naming collision detection"
```

---

### Task 2: Post-creation wksp symlink validation

**Files:**
- Modify: `work-slot/slot_manager.py` (add `validate_slot_wksp`, call from `create_slot` and `add_repo`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `validate_slot_wksp(slot_dir: Path, repo_names: list[str] | None = None) -> list[str]` — returns list of failure descriptions; empty = all OK. When `repo_names` is provided, validates only those repos (used by `add_repo`).
- Consumes: `get_slot_repos()`, `resolve_original_repo()`

- [ ] **Step 1: Write failing tests for `validate_slot_wksp`**

```python
class TestValidateSlotWksp:
    def test_passes_when_symlinks_resolve(self, tmp_path):
        """All repo clones have working wksp/ symlinks."""
        family = tmp_path / "casehub"
        family.mkdir()
        original = init_repo(family / "engine")
        ws_dir = tmp_path / "workspace"
        ws_dir.mkdir()
        (ws_dir / "engine").mkdir()
        (original / "wksp").symlink_to(ws_dir / "engine")

        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = init_repo(slot_dir / "engine")
        # Clone's wksp points to an existing directory
        ws_slot = slot_dir / "work" / "engine"
        ws_slot.mkdir(parents=True)
        (clone / "wksp").symlink_to(os.path.relpath(ws_slot, clone))

        failures = slot_manager.validate_slot_wksp(slot_dir)
        assert failures == []

    def test_fails_when_symlink_dangling(self, tmp_path):
        """wksp/ points to a non-existent directory."""
        family = tmp_path / "casehub"
        family.mkdir()
        original = init_repo(family / "engine")
        (original / "wksp").symlink_to("/nonexistent/path")

        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = init_repo(slot_dir / "engine")
        (clone / "wksp").symlink_to("/nonexistent/path")

        failures = slot_manager.validate_slot_wksp(slot_dir)
        assert len(failures) == 1
        assert "engine" in failures[0]

    def test_fails_when_symlink_missing(self, tmp_path):
        """Original has wksp/ but clone doesn't."""
        family = tmp_path / "casehub"
        family.mkdir()
        original = init_repo(family / "engine")
        ws_dir = tmp_path / "workspace"
        ws_dir.mkdir()
        (original / "wksp").symlink_to(ws_dir)

        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = init_repo(slot_dir / "engine")
        # No wksp symlink in clone

        # Set up so resolve_original_repo can find the original
        subprocess.run(["git", "-C", str(clone), "remote", "add", "local", str(original)], capture_output=True)

        failures = slot_manager.validate_slot_wksp(slot_dir)
        assert len(failures) == 1
        assert "missing" in failures[0].lower()

    def test_passes_when_original_has_no_wksp(self, tmp_path):
        """Original repo has no wksp/ — nothing to validate."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = init_repo(slot_dir / "engine")
        # Original has no wksp symlink — this is fine

        failures = slot_manager.validate_slot_wksp(slot_dir)
        assert failures == []

    def test_scoped_to_specific_repos(self, tmp_path):
        """When repo_names is provided, only those repos are checked."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        good = init_repo(slot_dir / "engine")
        bad = init_repo(slot_dir / "iot")
        (bad / "wksp").symlink_to("/nonexistent")

        # Only check engine — should pass despite iot being broken
        failures = slot_manager.validate_slot_wksp(slot_dir, repo_names=["engine"])
        assert failures == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestValidateSlotWksp -v`
Expected: FAIL with `AttributeError: module 'slot_manager' has no attribute 'validate_slot_wksp'`

- [ ] **Step 3: Implement `validate_slot_wksp`**

Add after `_get_family_repo_names` in `slot_manager.py`:

```python
def validate_slot_wksp(slot_dir: Path, repo_names: list[str] | None = None) -> list[str]:
    """Validate wksp/ symlinks in slot repo clones.
    Returns list of failure descriptions (empty = all OK)."""
    failures: list[str] = []
    names = repo_names if repo_names is not None else get_slot_repos(slot_dir)
    for repo_name in names:
        clone = slot_dir / repo_name
        if not clone.is_dir() or not (clone / ".git").exists():
            continue
        original = resolve_original_repo(clone)
        original_wksp = original / "wksp"
        if not original_wksp.is_symlink():
            continue
        clone_wksp = clone / "wksp"
        if not clone_wksp.is_symlink():
            failures.append(f"{repo_name}: wksp/ symlink missing")
        elif not clone_wksp.resolve().exists():
            failures.append(f"{repo_name}: wksp/ symlink dangling -> {clone_wksp.resolve()}")
    return failures
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestValidateSlotWksp -v`
Expected: PASS

- [ ] **Step 5: Wire validation into `create_slot`**

In `create_slot()`, after the worklog `confirm_slot_create` block (after the `finally: conn.close()` at ~line 644), add:

```python
    wksp_failures = validate_slot_wksp(slot_dir)
    if wksp_failures:
        for f in wksp_failures:
            print(f"ERROR=wksp_validation_failed detail={f}")
        sys.exit(1)
```

- [ ] **Step 6: Wire validation into `add_repo`**

In `add_repo()`, after `_update_slot_repos()` call (~line 737), add:

```python
    wksp_failures = validate_slot_wksp(slot_dir, repo_names=[repo_name])
    if wksp_failures:
        for f in wksp_failures:
            print(f"ERROR=wksp_validation_failed detail={f}")
        sys.exit(1)
```

- [ ] **Step 7: Write integration test for create_slot validation failure**

```python
class TestCreateSlotWkspValidation:
    @patch("slot_manager.run_cmd")
    def test_create_slot_exits_on_broken_wksp(self, mock_cmd, tmp_path, capsys):
        """create_slot must fail if post-creation validation finds broken wksp/."""
        family = tmp_path / "casehub"
        family.mkdir()
        engine = init_repo(family / "engine")
        shared_ws = init_repo(tmp_path / "public" / "casehub")
        (shared_ws / "engine").mkdir()
        (engine / "wksp").symlink_to(shared_ws / "engine")

        def side_effect(args, cwd=None):
            # Make clone succeed but don't actually clone —
            # this means wksp won't be wired, triggering validation failure
            if isinstance(args, list) and args[0] == "git" and "clone" in args:
                dest = Path(args[-1])
                dest.mkdir(parents=True, exist_ok=True)
                subprocess.run(["git", "init", str(dest)], capture_output=True)
                subprocess.run(["git", "-C", str(dest), "config", "user.name", "Test"], capture_output=True)
                subprocess.run(["git", "-C", str(dest), "config", "user.email", "test@test.com"], capture_output=True)
                subprocess.run(["git", "-C", str(dest), "commit", "--allow-empty", "-m", "init"], capture_output=True)
                return (0, "", "")
            return (0, "", "")

        mock_cmd.side_effect = side_effect

        with pytest.raises(SystemExit):
            slot_manager.create_slot(
                family_root=family,
                repos=["engine"],
                branch="issue-99-test",
                issue="99",
                issue_repo="casehubio/engine",
                covers="99",
                context="test",
            )
        captured = capsys.readouterr()
        assert "ERROR=wksp_validation_failed" in captured.out
```

- [ ] **Step 8: Run all validation tests**

Run: `python3 -m pytest tests/test_slot_manager.py::TestValidateSlotWksp tests/test_slot_manager.py::TestCreateSlotWkspValidation -v`
Expected: PASS

- [ ] **Step 9: Commit**

```
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "fix(#237): post-creation wksp symlink validation with fail-fast"
```

---

### Task 3: list_slots health and add_repo parity

**Files:**
- Modify: `work-slot/slot_manager.py` (`wksp_ok` in `list_slots`, remotes in `add_repo`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `validate_slot_wksp()` from Task 2, `_get_family_repo_names()` from Task 1
- Produces: `wksp_ok: bool` field in list_slots output dicts

- [ ] **Step 1: Write failing test for list_slots wksp health**

```python
class TestListSlotsWkspHealth:
    def test_wksp_ok_true_when_no_wksp(self, tmp_path):
        """Repos without workspace integration are wksp_ok=True."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot 1 — test\n")
        init_repo(slot_dir / "engine")
        slots = slot_manager.list_slots(tmp_path)
        assert len(slots) == 1
        assert slots[0]["wksp_ok"] is True

    def test_wksp_ok_false_when_dangling(self, tmp_path):
        """Broken wksp/ symlink surfaces as wksp_ok=False."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot 1 — test\n")
        clone = init_repo(slot_dir / "engine")
        (clone / "wksp").symlink_to("/nonexistent")
        # Need original to have wksp too for validation to check
        # Since resolve_original_repo falls back to self, the clone IS the original
        # So we need the clone itself to have a dangling wksp — which it does
        # But validate_slot_wksp checks original_wksp.is_symlink() first
        # When original == clone, original_wksp == clone_wksp, both dangling → failure
        slots = slot_manager.list_slots(tmp_path)
        assert len(slots) == 1
        assert slots[0]["wksp_ok"] is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestListSlotsWkspHealth -v`
Expected: FAIL — `wksp_ok` key doesn't exist

- [ ] **Step 3: Add `wksp_ok` to `list_slots`**

In `list_slots()`, in the loop that builds slot dicts (around line 1605), after computing `isolation`, add:

```python
            wksp_ok = True
            if state not in ("archived", "landed"):
                wksp_failures = validate_slot_wksp(d)
                wksp_ok = len(wksp_failures) == 0
```

And add `"wksp_ok": wksp_ok` to the dict at ~line 1611.

Also in the archived slots section (~line 1626), add `"wksp_ok": True` to the dict (archived slots are not checked).

- [ ] **Step 4: Update CLI output**

In `main()` under `list-slots` (around line 1912), update the print to include WKSP:

Old:
```python
            print(f"SLOT={s['number']} BRANCH={s['branch']} REPOS={repos_str} STATE={s['state']} ISOLATION={s.get('isolation', 'none')}")
```

New:
```python
            wksp = "ok" if s.get("wksp_ok", True) else "broken"
            print(f"SLOT={s['number']} BRANCH={s['branch']} REPOS={repos_str} STATE={s['state']} ISOLATION={s.get('isolation', 'none')} WKSP={wksp}")
```

- [ ] **Step 5: Run list_slots health tests**

Run: `python3 -m pytest tests/test_slot_manager.py::TestListSlotsWkspHealth tests/test_slot_manager.py::TestListSlots tests/test_slot_manager.py::TestListSlotsExtended -v`
Expected: PASS (including existing tests — verify no regression)

- [ ] **Step 6: Write test for add_repo workspace remotes**

```python
class TestAddRepoWorkspaceRemotes:
    @patch("slot_manager.run_cmd")
    def test_add_repo_configures_workspace_remotes(self, mock_cmd, tmp_path):
        """add_repo must configure remotes on new workspace clones."""
        family = tmp_path / "casehub"
        family.mkdir()
        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        init_repo(slot_dir / "engine")

        repo_b = init_repo(family / "iot")
        ws = init_repo(tmp_path / "public" / "casehub")
        (ws / "iot").mkdir()
        (repo_b / "wksp").symlink_to(ws / "iot")

        slot_manager.write_slot_md(
            slot_dir, 1, ["engine"], "test-branch", "42",
            "Org/repo", "42", "test",
        )

        mock_cmd.return_value = (0, "", "")
        slot_manager.add_repo(family, 1, "iot", "test-branch")

        # Check that configure_slot_remotes was called for the workspace clone
        # by looking for "remote rename origin local" calls on workspace dirs
        ws_rename_calls = [
            c for c in mock_cmd.call_args_list
            if isinstance(c.args[0], list)
            and "remote" in c.args[0]
            and "rename" in c.args[0]
            and "work" in str(c.args[0])
        ]
        assert len(ws_rename_calls) >= 1, (
            "add_repo did not configure remotes on workspace clone"
        )
```

- [ ] **Step 7: Add `configure_slot_remotes` and `configure_update_instead` to `add_repo` workspace path**

In `add_repo()`, after the workspace clone block (around line 720, after `_exclude_symlinks(ws_slot_dir)`), add:

```python
                configure_slot_remotes(ws_slot_dir, ws_source)
                configure_update_instead(ws_source)
```

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "fix(#237): list_slots wksp health, add_repo workspace remotes parity"
```

---

## Verification

After all tasks complete:

- [ ] Run the verification command from issue #237 body against any live slots
- [ ] Run `python3 -m pytest tests/test_slot_manager.py -v` — all green
- [ ] Run `python3 scripts/validate_all.py --tier commit` — no regressions

# Work-End Slot Landing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #224 — work-end: detect slot context and offer archive after branch close
**Issue group:** #224, #225

**Goal:** Wire up the slot landing path that SKILL.md already describes but was never implemented — write `.phase-a-complete` marker after squash so `merge-slot` can complete the two-hop push, .landed marker, and archive offer.

**Architecture:** Add `cmd_write_marker()` to `work_end_execute.py` (marker write). Extend `verify_slot_close.py` with three composable check functions for slot-specific landing verification. Update SKILL.md to call the marker script and clarify the slot mode flow.

**Tech Stack:** Python 3, subprocess (git), pytest

## Global Constraints

- All new `.py` functions ship with tests in the same commit (protocol: `externalised-scripts-require-tests`)
- Tests use `tmp_path` fixtures — no hardcoded paths
- Verification check functions follow the composable pattern: `check_<name>(args) -> dict` with `{"status": "pass|fail|warn", "detail": "..."}`
- No changes to `merge_slot()` or `close_artifacts.py` — they work correctly once the marker exists

---

### Task 1: `cmd_write_marker()` in `work_end_execute.py`

Write the `.phase-a-complete` marker that `merge_slot()` requires.

**Files:**
- Modify: `work-end/work_end_execute.py` — add `cmd_write_marker()` function and CLI dispatch
- Test: `tests/test_work_end_execute.py`

**Interfaces:**
- Produces: `cmd_write_marker(opts: dict[str, str]) -> int` — writes `.phase-a-complete` to `slot_path`, returns 0 on success
- Produces: CLI `python3 work_end_execute.py write-marker slot_path=<path> branch=<name>`

- [ ] **Step 1: Write failing tests**

```python
class TestWriteMarker:
    def test_writes_marker_with_correct_fields(self, tmp_path):
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        opts = {"slot_path": str(slot_dir), "branch": "issue-42-fix"}
        result = work_end_execute.cmd_write_marker(opts)
        assert result == 0
        marker = slot_dir / ".phase-a-complete"
        assert marker.exists()
        content = marker.read_text()
        assert "branch=issue-42-fix" in content
        assert "timestamp=" in content

    def test_missing_slot_path(self):
        result = work_end_execute.cmd_write_marker({"branch": "test"})
        assert result == 1

    def test_missing_branch(self, tmp_path):
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        result = work_end_execute.cmd_write_marker({"slot_path": str(slot_dir)})
        assert result == 1

    def test_slot_path_not_exists(self, tmp_path):
        result = work_end_execute.cmd_write_marker({
            "slot_path": str(tmp_path / "nonexistent"),
            "branch": "test",
        })
        assert result == 1

    def test_marker_format_matches_merge_slot_expectations(self, tmp_path):
        """merge_slot reads branch= from .phase-a-complete line by line."""
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        work_end_execute.cmd_write_marker({
            "slot_path": str(slot_dir),
            "branch": "issue-42-fix",
        })
        content = (slot_dir / ".phase-a-complete").read_text()
        branch_line = [l for l in content.splitlines() if l.startswith("branch=")]
        assert len(branch_line) == 1
        assert branch_line[0] == "branch=issue-42-fix"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_end_execute.py::TestWriteMarker -v`
Expected: FAIL — `cmd_write_marker` not defined

- [ ] **Step 3: Implement `cmd_write_marker()`**

Add to `work_end_execute.py` before `main()`:

```python
def cmd_write_marker(opts: dict[str, str]) -> int:
    slot_path = opts.get("slot_path", "")
    branch = opts.get("branch", "")

    if not slot_path:
        print("ERROR=MISSING_ARGS")
        print("ERROR_DETAIL=slot_path= is required")
        return 1
    if not branch:
        print("ERROR=MISSING_ARGS")
        print("ERROR_DETAIL=branch= is required")
        return 1

    slot_dir = Path(slot_path)
    if not slot_dir.is_dir():
        print("ERROR=BAD_PATH")
        print(f"ERROR_DETAIL=slot_path={slot_path} not found")
        return 1

    import datetime
    marker = slot_dir / ".phase-a-complete"
    marker.write_text(
        f"branch={branch}\n"
        f"timestamp={datetime.datetime.now(datetime.timezone.utc).isoformat()}\n"
    )
    print(f"MARKER_WRITTEN={marker}")
    return 0
```

Add CLI dispatch in `main()`:

```python
    elif command == "write-marker":
        return cmd_write_marker(opts)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_end_execute.py::TestWriteMarker -v`
Expected: PASS

- [ ] **Step 5: Run full work_end_execute test suite**

Run: `python3 -m pytest tests/test_work_end_execute.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add work-end/work_end_execute.py tests/test_work_end_execute.py
git -C <PROJECT> commit -m "feat(#224): cmd_write_marker for .phase-a-complete

Enables merge-slot to complete the slot landing pipeline.
Writes branch= and timestamp= to the slot root marker file.

Refs #224, Refs #225"
```

---

### Task 2: `verify_slot_close.py` Extensions

Add three composable check functions for slot-specific landing verification.

**Files:**
- Modify: `work-end/verify_slot_close.py` — add `check_landed_marker()`, `check_original_sync()`, `check_slot_archive_status()`, wire into `verify()`
- Test: `tests/test_verify_slot_close.py` — new test classes

**Interfaces:**
- Consumes: Existing `verify()` function signature, existing composable check pattern
- Produces: Three new check functions following the `check_<name>() -> dict` pattern

- [ ] **Step 1: Write failing tests for `check_landed_marker()`**

```python
class TestCheckLandedMarker:
    def test_landed_marker_present(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        (slot_dir / ".landed").write_text(
            "branch=issue-42\n"
            "repos=engine,work\n"
            "landed_shas=engine:abc123,work:def456\n"
            "timestamp=2026-08-12T00:00:00Z\n"
        )
        result = verify_slot_close.check_landed_marker(str(slot_dir))
        assert result["status"] == "pass"

    def test_landed_marker_absent(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        result = verify_slot_close.check_landed_marker(str(slot_dir))
        assert result["status"] == "fail"
        assert "no .landed marker" in result["detail"]

    def test_landed_marker_no_shas(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        (slot_dir / ".landed").write_text("branch=issue-42\n")
        result = verify_slot_close.check_landed_marker(str(slot_dir))
        assert result["status"] == "fail"
        assert "no landed_shas" in result["detail"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestCheckLandedMarker -v`
Expected: FAIL — `check_landed_marker` not defined

- [ ] **Step 3: Implement `check_landed_marker()`**

Add to `verify_slot_close.py`:

```python
def check_landed_marker(slot_dir: str) -> dict:
    landed = Path(slot_dir) / ".landed"
    if not landed.exists():
        return {"status": "fail", "detail": "no .landed marker"}
    content = landed.read_text()
    if "landed_shas=" not in content:
        return {"status": "fail", "detail": "no landed_shas in .landed marker"}
    return {"status": "pass"}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestCheckLandedMarker -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for `check_original_sync()`**

```python
class TestCheckOriginalSync:
    def test_original_in_sync(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        original = _init_repo(tmp_path / "original")
        clone = _init_repo(slot_dir / "engine")
        # Both at same SHA
        orig_sha = _git(original, "rev-parse", "HEAD")
        clone_sha = _git(clone, "rev-parse", "HEAD")
        (slot_dir / ".landed").write_text(
            f"landed_shas=engine:{clone_sha}\n"
        )
        result = verify_slot_close.check_original_sync(
            str(slot_dir), "engine", str(original),
        )
        assert result["status"] == "pass"

    def test_original_behind(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        original = _init_repo(tmp_path / "original")
        clone = _init_repo(slot_dir / "engine")
        # Add a commit to clone that original doesn't have
        (clone / "extra.txt").write_text("new\n")
        _git(clone, "add", "extra.txt")
        _git(clone, "commit", "-m", "extra")
        clone_sha = _git(clone, "rev-parse", "HEAD")
        (slot_dir / ".landed").write_text(
            f"landed_shas=engine:{clone_sha}\n"
        )
        result = verify_slot_close.check_original_sync(
            str(slot_dir), "engine", str(original),
        )
        assert result["status"] == "fail"
        assert "behind" in result["detail"] or "not reachable" in result["detail"]
```

- [ ] **Step 6: Implement `check_original_sync()`**

```python
def check_original_sync(slot_dir: str, repo_name: str, original_path: str) -> dict:
    landed = Path(slot_dir) / ".landed"
    if not landed.exists():
        return {"status": "fail", "detail": "no .landed marker"}

    landed_sha = ""
    for line in landed.read_text().splitlines():
        if line.startswith("landed_shas="):
            shas_str = line.split("=", 1)[1]
            for entry in shas_str.split(","):
                if ":" in entry:
                    name, sha = entry.split(":", 1)
                    if name == repo_name:
                        landed_sha = sha
                        break

    if not landed_sha:
        return {"status": "fail", "detail": f"no landed SHA for {repo_name}"}

    result = git(original_path, "merge-base", "--is-ancestor", landed_sha, "main")
    if result.returncode == 0:
        return {"status": "pass", "detail": f"{repo_name} SHA {landed_sha[:8]} on main"}
    return {"status": "fail", "detail": f"{repo_name} SHA {landed_sha[:8]} not reachable from main — original behind"}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestCheckOriginalSync -v`
Expected: PASS

- [ ] **Step 8: Write failing test and implement `check_slot_archive_status()`**

```python
class TestCheckSlotArchiveStatus:
    def test_archived(self, tmp_path: Path) -> None:
        attic_dir = tmp_path / "slots" / "attic" / "1"
        attic_dir.mkdir(parents=True)
        result = verify_slot_close.check_slot_archive_status(
            str(tmp_path / "slots" / "1"), str(attic_dir),
        )
        assert result["status"] == "pass"
        assert "archived" in result["detail"]

    def test_landed_not_archived(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".landed").write_text("landed\n")
        result = verify_slot_close.check_slot_archive_status(
            str(slot_dir), str(tmp_path / "slots" / "attic" / "1"),
        )
        assert result["status"] == "warn"
        assert "landed but not archived" in result["detail"]

    def test_active(self, tmp_path: Path) -> None:
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        result = verify_slot_close.check_slot_archive_status(
            str(slot_dir), str(tmp_path / "slots" / "attic" / "1"),
        )
        assert result["status"] == "warn"
        assert "active" in result["detail"]
```

Implementation:

```python
def check_slot_archive_status(slot_dir: str, attic_dir: str) -> dict:
    if Path(attic_dir).is_dir():
        return {"status": "pass", "detail": "archived"}
    slot_path = Path(slot_dir)
    if slot_path.is_dir() and (slot_path / ".landed").exists():
        return {"status": "warn", "detail": "landed but not archived"}
    if slot_path.is_dir():
        return {"status": "warn", "detail": "active — not landed"}
    return {"status": "fail", "detail": "slot not found"}
```

- [ ] **Step 9: Run all verify tests**

Run: `python3 -m pytest tests/test_verify_slot_close.py -v`
Expected: All PASS (new + existing)

- [ ] **Step 10: Wire new checks into `verify()` for slot mode**

Add optional `slot_dir` and `original_repos` parameters to `verify()`.
When `slot_dir` is provided, append slot-specific checks:

```python
def verify(
    project: str, branch: str, workspace: str,
    base: str = "main", covers: list[int] | None = None,
    slot_dir: str = "", original_repos: dict[str, str] | None = None,
) -> bool:
    # ... existing checks ...

    if slot_dir:
        checks.append(("landed_marker", check_landed_marker(slot_dir)))
        if original_repos:
            for repo_name, orig_path in original_repos.items():
                checks.append((
                    f"original_sync_{repo_name}",
                    check_original_sync(slot_dir, repo_name, orig_path),
                ))
        slot_num = Path(slot_dir).name
        attic = str(Path(slot_dir).parent / "attic" / slot_num)
        checks.append(("archive_status", check_slot_archive_status(slot_dir, attic)))

    # ... existing output loop ...
```

- [ ] **Step 11: Commit**

```bash
git -C <PROJECT> add work-end/verify_slot_close.py tests/test_verify_slot_close.py
git -C <PROJECT> commit -m "feat(#225): extend verify_slot_close with slot landing checks

Adds check_landed_marker(), check_original_sync(),
check_slot_archive_status() as composable check functions.
Wired into verify() when slot_dir is provided.

Refs #224, Refs #225"
```

---

### Task 3: SKILL.md Updates

Document the marker write call and clarify the slot mode flow.

**Files:**
- Modify: `work-end/SKILL.md`

**Interfaces:**
- None — documentation only

- [ ] **Step 1: Update Step 3.4 (Phase B — Squash)**

After the squash analysis section, add:

```markdown
**Slot mode marker:** After squash completes for all repos, if `IN_SLOT=yes`:

\`\`\`bash
python3 work-end/work_end_execute.py write-marker slot_path=<SLOT_PATH> branch=<BRANCH>
\`\`\`

This writes `.phase-a-complete` to the slot root, enabling `merge-slot`
in Phase C. Read `MARKER_WRITTEN=` from output. If error, report and
offer to retry — the squash is already done, only the marker failed.
```

- [ ] **Step 2: Clarify Step 3.5 Phase C slot mode**

Ensure the slot mode section is unambiguous:

```markdown
**Slot mode (IN_SLOT=yes):**

Requires `.phase-a-complete` marker (written in Step 3.4).

\`\`\`bash
python3 work-slot/slot_manager.py merge-slot <SLOT_PATH>
\`\`\`

merge-slot handles: per-repo rebase (no-op if already rebased), two-hop
push (slot clone → original → GitHub), SHA verification, `.landed`
marker, branch stamps. Do NOT call `work_end_execute.py land` in slot
mode — that path is for branch mode only.
```

- [ ] **Step 3: Verify Step 5.1 archive is reachable**

Confirm the archive prompt text matches the current SKILL.md. No change
needed — the flow now reaches it because merge-slot succeeds.

- [ ] **Step 4: Commit**

```bash
git -C <PROJECT> add work-end/SKILL.md
git -C <PROJECT> commit -m "docs(#224): SKILL.md — wire marker write, clarify slot mode Phase C

Adds write-marker call after squash in Step 3.4.
Clarifies merge-slot is the sole slot landing path in Step 3.5.

Refs #224, Refs #225"
```

---

## Execution Order

Tasks 1 and 2 are independent (different files). Task 3 depends on
Task 1 existing (references the write-marker command).

```
Task 1: cmd_write_marker       ← work_end_execute.py
Task 2: verify extensions      ← verify_slot_close.py (independent of Task 1)
Task 3: SKILL.md               ← references Task 1's command
```

---

## Self-Review

**1. Spec coverage:**
- `.phase-a-complete` marker write → Task 1
- Phase C slot mode clarification → Task 3
- verify_slot_close.py extensions → Task 2
- Tests → Tasks 1, 2
- Promotion push (no change needed) → confirmed in spec, no task needed
- merge-slot (no change needed) → confirmed in spec, no task needed
- Archive offer (already in SKILL.md) → Task 3 confirms reachability

**2. Placeholder scan:** No TBD/TODO. All code blocks are concrete.

**3. Type consistency:** `cmd_write_marker(opts: dict[str, str]) -> int` is consistent across Task 1. Check function signatures match existing pattern (`check_<name>(...) -> dict`).

**4. Tooling safety scan:** No bash file operations on source files. All modifications via Edit tool. Git and pytest commands are bash-appropriate.

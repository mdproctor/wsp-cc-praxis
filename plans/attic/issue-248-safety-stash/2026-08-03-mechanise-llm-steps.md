# Mechanise LLM-Dependent Steps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #151 — Mechanise LLM-dependent steps in work-end and slot-merge
**Issue group:** #151

**Goal:** Replace LLM-dependent steps with mechanical scripts, add epic
lifecycle awareness at every work transition point, and suppress mid-work
pushes in slot mode.

**Architecture:** Consolidate epic detection into `epic_manager.detect()` as
the single source of truth. Add a `check` CLI subcommand for gates. Enrich
ctx.py and work_router.py by delegation. Add phase_b_gate.py for Phase B
verification. Update stack.py for epic field propagation. Update 6 skill
files for display and push discipline.

**Tech Stack:** Python 3.14, pytest, gh CLI (mocked in tests)

## Global Constraints

- All scripts follow KEY=VALUE stdout protocol
- GitHub API failures are non-fatal (warn and continue) except in phase_b_gate
  where they produce GATE=warn (not GATE=fail)
- `externalised-scripts-require-tests` protocol: every new/modified function
  gets unit tests
- No AI attribution in commits
- All commits reference #151 (`Refs #151`)

---

### Task 1: epic_manager.py — detect() + check + safe_exit fix

**Files:**
- Modify: `work-slot/epic_manager.py`
- Modify: `tests/test_epic_manager.py`

**Interfaces:**
- Consumes: `parse_batch_plan(epic_path: Path) -> dict` (existing)
- Consumes: `status(epic_path: Path) -> dict` (existing)
- Produces: `detect(path: Path) -> dict | None` — returns enriched
  parse_batch_plan dict with `epic_path` key, or None
- Produces: `check` CLI subcommand — KEY=VALUE output on stdout
- Produces: `tick_epic_checkboxes(issue_repo: str, epic_number: int, completed_issues: list[int]) -> bool`

- [ ] **Step 1: Write failing tests for detect()**

```python
# tests/test_epic_manager.py — add to existing file

class TestDetect:
    def test_detects_single_repo_epic(self, tmp_path):
        epic_dir = tmp_path / "design"
        epic_dir.mkdir()
        (epic_dir / ".epic").write_text(
            "## Issue\nHortora/soredium#99\nType: epic\n\n"
            "## Batch Plan\n\n### Batch 1 — Setup\n"
            "- [ ] #100 — First task ← active\n"
        )
        result = detect(tmp_path)
        assert result is not None
        assert result["is_epic"] is True
        assert result["epic_path"] == epic_dir / ".epic"
        assert result["current_issue"] == 100

    def test_detects_slot_epic(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 72\n\n## Issue\nHortora/soredium#50\nType: epic\n\n"
            "## Batch Plan\n\n### Batch 1 — Work\n"
            "- [x] #51 — Done\n- [ ] #52 — Active ← active\n"
        )
        result = detect(tmp_path)
        assert result is not None
        assert result["is_epic"] is True
        assert result["epic_path"] == tmp_path / ".slot"

    def test_detects_project_inside_slot(self, tmp_path):
        slot_dir = tmp_path
        (slot_dir / ".slot").write_text(
            "# Slot 72\n\n## Issue\nHortora/soredium#50\nType: epic\n\n"
            "## Batch Plan\n\n### Batch 1 — Work\n"
            "- [ ] #51 — Task ← active\n"
        )
        project_dir = slot_dir / "engine"
        project_dir.mkdir()
        result = detect(project_dir)
        assert result is not None
        assert result["epic_path"] == slot_dir / ".slot"

    def test_returns_none_no_epic(self, tmp_path):
        assert detect(tmp_path) is None

    def test_returns_none_non_epic_slot(self, tmp_path):
        (tmp_path / ".slot").write_text("# Slot 5\n\n## Issue\nrepo#10\n")
        assert detect(tmp_path) is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_epic_manager.py::TestDetect -v`
Expected: FAIL — `detect` not importable

- [ ] **Step 3: Implement detect()**

Add to `work-slot/epic_manager.py` after `parse_batch_plan`:

```python
def detect(path: Path) -> dict | None:
    """Find and parse an epic file from the given path.

    Search order:
    1. path/design/.epic (single-repo workspace)
    2. path/.slot with Type: epic (slot directory)
    3. path.parent/.slot with Type: epic (project inside slot)

    Returns enriched parse_batch_plan dict with epic_path, or None.
    """
    candidates = [
        path / "design" / ".epic",
        path / ".slot",
        path.parent / ".slot",
    ]
    for candidate in candidates:
        if candidate.exists():
            result = parse_batch_plan(candidate)
            if result.get("is_epic"):
                result["epic_path"] = candidate
                return result
    return None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestDetect -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for safe_exit fix**

```python
class TestSafeExit:
    def test_safe_exit_false_mid_batch_with_prior_complete(self, tmp_path):
        """safe_exit must be False when mid-batch, even if batch 1 completed."""
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n\n"
            "### Batch 1 — Done\n- [x] #10 — A\n- [x] #11 — B\n\n"
            "### Batch 2 — In progress\n- [x] #12 — C\n- [ ] #13 — D ← active\n"
        )
        result = status(epic)
        assert result["safe_exit"] is False

    def test_safe_exit_true_at_batch_boundary(self, tmp_path):
        """safe_exit True when all prior batches complete and current batch not started."""
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n\n"
            "### Batch 1 — Done\n- [x] #10 — A\n- [x] #11 — B\n\n"
            "### Batch 2 — Next\n- [ ] #12 — C ← active\n- [ ] #13 — D\n"
        )
        result = status(epic)
        assert result["safe_exit"] is True

    def test_safe_exit_false_no_batches_complete(self, tmp_path):
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n\n"
            "### Batch 1 — Work\n- [ ] #10 — A ← active\n- [ ] #11 — B\n"
        )
        result = status(epic)
        assert result["safe_exit"] is False
```

- [ ] **Step 6: Run to verify fail**

Run: `python3 -m pytest tests/test_epic_manager.py::TestSafeExit -v`
Expected: `test_safe_exit_false_mid_batch_with_prior_complete` FAILS (current code returns True)

- [ ] **Step 7: Fix safe_exit in status()**

In `status()`, replace line 274:
```python
# OLD: "safe_exit": completed_batches > 0,
```
with:
```python
at_batch_boundary = completed_batches > 0
if at_batch_boundary and current_issue != 0:
    current_batch_obj = next(
        (b for b in plan["batches"] if b["number"] == plan["current_batch"]),
        None,
    )
    if current_batch_obj:
        first_in_batch = current_batch_obj["issues"][0]["number"] if current_batch_obj["issues"] else 0
        at_batch_boundary = (current_issue == first_in_batch and
                             not current_batch_obj["issues"][0]["done"])

# ...
"safe_exit": at_batch_boundary,
```

The logic: safe_exit is True when (a) at least one batch is complete AND (b) the
active issue is the first undone issue in its batch with no done issues in that batch.

- [ ] **Step 8: Run to verify pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestSafeExit -v`
Expected: PASS

- [ ] **Step 9: Write failing tests for check subcommand**

```python
class TestCheckSubcommand:
    def test_check_output_format(self, tmp_path):
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n\n"
            "### Batch 1 — Work\n- [x] #10 — A\n- [ ] #11 — B ← active\n\n"
            "### Batch 2 — More\n- [ ] #12 — C\n"
        )
        import subprocess
        result = subprocess.run(
            [sys.executable, str(Path(__file__).parent.parent / "work-slot" / "epic_manager.py"),
             "check", str(epic)],
            capture_output=True, text=True,
        )
        lines = dict(l.split("=", 1) for l in result.stdout.strip().split("\n") if "=" in l)
        assert lines["IS_EPIC"] == "yes"
        assert lines["EPIC_COMPLETE"] == "no"
        assert lines["SAFE_EXIT"] == "no"  # mid-batch
        assert lines["CURRENT_BATCH"] == "1"
        assert lines["TOTAL_BATCHES"] == "2"
        assert lines["ACTIVE_ISSUE"] == "11"

    def test_check_epic_complete(self, tmp_path):
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n\n"
            "### Batch 1 — Done\n- [x] #10 — A\n"
        )
        import subprocess
        result = subprocess.run(
            [sys.executable, str(Path(__file__).parent.parent / "work-slot" / "epic_manager.py"),
             "check", str(epic)],
            capture_output=True, text=True,
        )
        lines = dict(l.split("=", 1) for l in result.stdout.strip().split("\n") if "=" in l)
        assert lines["EPIC_COMPLETE"] == "yes"

    def test_check_empty_plan_not_complete(self, tmp_path):
        epic = tmp_path / ".epic"
        epic.write_text(
            "## Issue\nrepo#1\nType: epic\n\n## Batch Plan\n"
        )
        import subprocess
        result = subprocess.run(
            [sys.executable, str(Path(__file__).parent.parent / "work-slot" / "epic_manager.py"),
             "check", str(epic)],
            capture_output=True, text=True,
        )
        lines = dict(l.split("=", 1) for l in result.stdout.strip().split("\n") if "=" in l)
        assert lines["EPIC_COMPLETE"] == "no"
```

- [ ] **Step 10: Run to verify fail**

Run: `python3 -m pytest tests/test_epic_manager.py::TestCheckSubcommand -v`
Expected: FAIL — `check` subcommand not implemented

- [ ] **Step 11: Implement check subcommand**

Add to `main()` in epic_manager.py, after the `status` elif:

```python
elif command == "check":
    result = status(epic_path)
    if not result.get("is_epic"):
        print("IS_EPIC=no")
    else:
        total = result["total_issues"]
        completed = result["completed_count"]
        epic_complete = total > 0 and completed == total
        print("IS_EPIC=yes")
        print(f"EPIC_COMPLETE={'yes' if epic_complete else 'no'}")
        print(f"SAFE_EXIT={'yes' if result['safe_exit'] else 'no'}")
        print(f"CURRENT_BATCH={result['current_batch']}")
        print(f"TOTAL_BATCHES={result['total_batches']}")
        print(f"ACTIVE_ISSUE={result['current_issue']}")
        print(f"COMPLETED_COUNT={completed}")
        print(f"TOTAL_COUNT={total}")
```

- [ ] **Step 12: Run to verify pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestCheckSubcommand -v`
Expected: PASS

- [ ] **Step 13: Write failing tests for tick_epic_checkboxes()**

```python
class TestTickEpicCheckboxes:
    def test_ticks_matching_checkboxes(self):
        body = "## Scope\n- [ ] #83 — Fix auth\n- [ ] #84 — Add tests\n- [x] #85 — Done\n"
        result = _tick_checkboxes_in_body(body, [83])
        assert "- [x] #83 — Fix auth" in result
        assert "- [ ] #84 — Add tests" in result
        assert "- [x] #85 — Done" in result

    def test_idempotent(self):
        body = "- [x] #83 — Already done\n- [ ] #84 — Not done\n"
        result = _tick_checkboxes_in_body(body, [83])
        assert result == body

    def test_handles_url_format(self):
        body = "- [ ] https://github.com/org/repo/issues/83 title\n"
        result = _tick_checkboxes_in_body(body, [83])
        assert "- [x] https://github.com/org/repo/issues/83" in result

    def test_multiple_issues(self):
        body = "- [ ] #10 — A\n- [ ] #11 — B\n- [ ] #12 — C\n"
        result = _tick_checkboxes_in_body(body, [10, 12])
        assert "- [x] #10" in result
        assert "- [ ] #11" in result
        assert "- [x] #12" in result
```

- [ ] **Step 14: Run to verify fail**

Run: `python3 -m pytest tests/test_epic_manager.py::TestTickEpicCheckboxes -v`
Expected: FAIL — function not defined

- [ ] **Step 15: Implement _tick_checkboxes_in_body() and tick_epic_checkboxes()**

Add to epic_manager.py:

```python
import subprocess as _sp

def _tick_checkboxes_in_body(body: str, issues: list[int]) -> str:
    """Replace - [ ] #N with - [x] #N for each issue number in the list."""
    lines = body.splitlines()
    result = []
    for line in lines:
        for n in issues:
            if re.match(rf"^- \[ \] #?{n}\b", line) or re.match(rf"^- \[ \] https://github\.com/.+/issues/{n}\b", line):
                line = line.replace("- [ ]", "- [x]", 1)
                break
        result.append(line)
    return "\n".join(result) + ("\n" if body.endswith("\n") else "")


def tick_epic_checkboxes(issue_repo: str, epic_number: int,
                         completed_issues: list[int]) -> bool:
    """Tick checkboxes on the GitHub epic issue body. Returns True on success."""
    try:
        r = _sp.run(
            ["gh", "api", f"repos/{issue_repo}/issues/{epic_number}",
             "--jq", ".body"],
            capture_output=True, text=True, timeout=30,
        )
        if r.returncode != 0:
            return False
        body = r.stdout
        updated = _tick_checkboxes_in_body(body, completed_issues)
        if updated == body:
            return True
        r = _sp.run(
            ["gh", "api", "-X", "PATCH",
             f"repos/{issue_repo}/issues/{epic_number}",
             "-f", f"body={updated}"],
            capture_output=True, text=True, timeout=30,
        )
        return r.returncode == 0
    except Exception:
        return False
```

- [ ] **Step 16: Run to verify pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestTickEpicCheckboxes -v`
Expected: PASS

- [ ] **Step 17: Add tick CLI subcommand to main()**

```python
elif command == "tick":
    kv = _parse_kv_args(sys.argv[3:])
    issue_repo = kv.get("issue-repo", "")
    epic_number = int(kv.get("epic", "0"))
    issues_str = kv.get("issues", "")
    completed = [int(x) for x in issues_str.split(",") if x.strip()]
    if not issue_repo or not epic_number or not completed:
        print("ERROR=missing_args")
        sys.exit(1)
    ok = tick_epic_checkboxes(issue_repo, epic_number, completed)
    print(f"TICK={'ok' if ok else 'failed'}")
```

- [ ] **Step 18: Run all epic_manager tests**

Run: `python3 -m pytest tests/test_epic_manager.py -v`
Expected: ALL PASS (existing + new)

- [ ] **Step 19: Commit**

```bash
git add work-slot/epic_manager.py tests/test_epic_manager.py
git commit -m "feat(#151): add detect(), check, tick + fix safe_exit in epic_manager  Refs #151"
```

---

### Task 2: ctx.py enrichment + work_router.py consolidation

**Files:**
- Modify: `project/ctx.py`
- Modify: `work/work_router.py`
- Modify: `tests/test_ctx.py` (if exists, else create)
- Modify: `tests/test_work_router.py` (if exists)

**Interfaces:**
- Consumes: `epic_manager.detect(path: Path) -> dict | None` (Task 1)
- Produces: `IS_EPIC`, `EPIC_PATH`, `EPIC_BATCH`, `EPIC_ACTIVE_ISSUE` in ctx.py output
- Produces: Same KEY=VALUE output from work_router.py via detect() delegation

- [ ] **Step 1: Write failing test for ctx.py slot-mode epic detection**

Check if `tests/test_ctx.py` exists, create if needed. Write a test that
sets up a slot-mode directory structure with a `.slot` epic file, runs ctx.py,
and asserts `EPIC_BATCH` and `EPIC_ACTIVE_ISSUE` appear in output.

- [ ] **Step 2: Run to verify fail**

Run: `python3 -m pytest tests/test_ctx.py -v -k epic`
Expected: FAIL

- [ ] **Step 3: Modify ctx.py to use detect()**

Replace lines 165-170 (inline epic detection) with:

```python
sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
from epic_manager import detect as _epic_detect

epic_info = _epic_detect(Path(workspace))
if epic_info is None and "/worktrees/" in str(project):
    epic_info = _epic_detect(Path(project))

is_epic = epic_info is not None
epic_path_str = str(epic_info["epic_path"]) if epic_info else ""
epic_batch = ""
epic_active_issue = ""
if epic_info:
    current = epic_info.get("current_batch", 0)
    total = len(epic_info.get("batches", []))
    epic_batch = f"{current} of {total}" if total else ""
    epic_active_issue = str(epic_info.get("current_issue", ""))
```

Update the output section to add:
```python
print(f"EPIC_BATCH={epic_batch}")
print(f"EPIC_ACTIVE_ISSUE={epic_active_issue}")
```

- [ ] **Step 4: Run to verify pass**

Run: `python3 -m pytest tests/test_ctx.py -v -k epic`
Expected: PASS

- [ ] **Step 5: Write failing test for work_router.py detect() delegation**

Write a test that creates a slot directory with an epic `.slot` file, runs
work_router.py, and asserts IS_EPIC=yes, EPIC_BATCH, EPIC_ACTIVE_ISSUE
match the expected values. The test should verify identical output to what
the current inline parser produces.

- [ ] **Step 6: Run to verify fail (may pass if interface matches)**

- [ ] **Step 7: Replace work_router.py inline epic parsing with detect()**

Replace lines 70-137 with:

```python
sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
from epic_manager import detect as _epic_detect

if "/worktrees/" in str(project):
    epic_info = _epic_detect(project.parent)
    if epic_info:
        in_slot = True
        is_epic = True
        slot_path = str(epic_info["epic_path"])
        current = epic_info.get("current_batch", 0)
        total = len(epic_info.get("batches", []))
        epic_batch = f"{current} of {total}" if total else ""
        epic_active_issue = str(epic_info.get("current_issue", ""))
    elif (project.parent / ".slot").exists():
        in_slot = True
        slot_path = str(project.parent / ".slot")

if not in_slot:
    epic_info = _epic_detect(workspace)
    if epic_info:
        is_epic = True
        epic_file_path = str(epic_info["epic_path"])
        current = epic_info.get("current_batch", 0)
        total = len(epic_info.get("batches", []))
        epic_batch = f"{current} of {total}" if total else ""
        epic_active_issue = str(epic_info.get("current_issue", ""))
```

- [ ] **Step 8: Run all work_router tests**

Run: `python3 -m pytest tests/test_work_router.py -v`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add project/ctx.py work/work_router.py tests/test_ctx.py tests/test_work_router.py
git commit -m "refactor(#151): consolidate epic detection via epic_manager.detect()  Refs #151"
```

---

### Task 3: merge_slot() epic check + GitHub checkbox tick

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `epic_manager.status(epic_path: Path) -> dict` (existing)
- Consumes: `epic_manager.tick_epic_checkboxes(issue_repo, epic_number, completed)` (Task 1)
- Consumes: `slot_manager.parse_slot_md(slot_dir: Path) -> dict` (existing)
- Produces: `EPIC_STATUS=batch N/M, N completed, N remaining` in merge_slot output

- [ ] **Step 1: Write failing test for merge_slot epic status**

Add to `tests/test_slot_manager.py` — a test that creates an epic slot
with `.phase-a-complete`, runs merge_slot (with mocked git), and asserts
`EPIC_STATUS=` appears in stdout.

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Add epic check to merge_slot()**

After the `.phase-a-complete` check (line 602-604), add:

```python
slot_info = parse_slot_md(slot_dir)
if slot_info.get("is_epic"):
    sys.path.insert(0, str(Path(__file__).parent))
    from epic_manager import status as _epic_status, tick_epic_checkboxes
    epic_status = _epic_status(slot_dir / ".slot")
    if epic_status.get("is_epic"):
        total = epic_status["total_issues"]
        done = epic_status["completed_count"]
        batch = epic_status["current_batch"]
        total_b = epic_status["total_batches"]
        print(f"EPIC_STATUS=batch {batch}/{total_b}, {done} completed, {total - done} remaining")
```

- [ ] **Step 4: Run to verify pass**

- [ ] **Step 5: Write failing test for post-merge tick**

Test that after merge_slot writes `.landed`, it calls tick_epic_checkboxes.
Mock the function and assert it was called with the right arguments.

- [ ] **Step 6: Run to verify fail**

- [ ] **Step 7: Add tick call after .landed is written**

After line 703 (`.landed` file written), add:

```python
if slot_info.get("is_epic"):
    epic_num = int(slot_info.get("issue", "0"))
    epic_repo = slot_info.get("issue_repo", "")
    covers_str = slot_info.get("covers", "")
    completed = [int(x) for x in covers_str.split(",") if x.strip()]
    if epic_num and epic_repo and completed:
        ok = tick_epic_checkboxes(epic_repo, epic_num, completed)
        if not ok:
            print("WARN=epic_tick_failed")
```

- [ ] **Step 8: Run all slot_manager tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "feat(#151): add epic check + GitHub tick to merge_slot  Refs #151"
```

---

### Task 4: archive_slot() checkbox verification + catch-up tick

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `parse_slot_md(slot_dir: Path) -> dict` (existing)
- Consumes: `tick_epic_checkboxes(issue_repo, epic_number, completed)` (Task 1)

- [ ] **Step 1: Write failing test for stale checkbox auto-fix**

Create an epic `.slot` with `- [ ] #83` but issue 83 would be closed.
Since we can't call GitHub in unit tests, test the local `.slot` rewrite
logic directly: write a helper `_fix_stale_checkboxes(slot_path, issues_to_tick)`
and test it.

```python
def test_archive_fixes_stale_checkboxes(self, tmp_path):
    slot_dir = tmp_path / "worktrees" / "72"
    slot_dir.mkdir(parents=True)
    (slot_dir / ".slot").write_text(
        "# Slot 72\n\n## Issue\nrepo#50\nType: epic\nCovers: 83,84\n\n"
        "## Batch Plan\n\n### Batch 1 — Work\n"
        "- [ ] #83 — Task A\n- [x] #84 — Task B\n"
    )
    (slot_dir / ".landed").write_text("branch=x\n")
    _fix_stale_checkboxes(slot_dir / ".slot", [83])
    content = (slot_dir / ".slot").read_text()
    assert "- [x] #83" in content
```

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Implement _fix_stale_checkboxes()**

Add to slot_manager.py:

```python
def _fix_stale_checkboxes(slot_path: Path, issues_to_tick: list[int]) -> int:
    """Tick unchecked boxes for completed issues. Returns count fixed."""
    content = slot_path.read_text()
    fixed = 0
    lines = content.splitlines()
    result = []
    for line in lines:
        for n in issues_to_tick:
            if f"- [ ] #{n} " in line or f"- [ ] #{n}\n" in line.rstrip():
                line = line.replace("- [ ]", "- [x]", 1)
                fixed += 1
                break
        result.append(line)
    if fixed:
        slot_path.write_text("\n".join(result))
    return fixed
```

- [ ] **Step 4: Run to verify pass**

- [ ] **Step 5: Wire into archive_slot()**

In `archive_slot()`, after `ensure_clone_layout(slot_dir)` (line 836) and
before the attic move, add:

```python
slot_info = parse_slot_md(slot_dir)
if slot_info.get("is_epic"):
    covers_str = slot_info.get("covers", "")
    completed = [int(x) for x in covers_str.split(",") if x.strip()]
    if completed:
        fixed = _fix_stale_checkboxes(slot_dir / ".slot", completed)
        if fixed:
            print(f"CHECKBOXES_FIXED={fixed}")
            print(f"WARN=stale_checkboxes issues={','.join(str(c) for c in completed[:fixed])}")
        epic_num = int(slot_info.get("issue", "0"))
        epic_repo = slot_info.get("issue_repo", "")
        if epic_num and epic_repo:
            try:
                from epic_manager import tick_epic_checkboxes
                ok = tick_epic_checkboxes(epic_repo, epic_num, completed)
                if not ok:
                    print("WARN=github_unreachable_for_checkbox_verify")
            except Exception:
                print("WARN=github_unreachable_for_checkbox_verify")
```

- [ ] **Step 6: Run all slot_manager tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "feat(#151): archive_slot verifies + fixes stale epic checkboxes  Refs #151"
```

---

### Task 5: phase_b_gate.py

**Files:**
- Create: `work-end/phase_b_gate.py`
- Create: `tests/test_phase_b_gate.py`

**Interfaces:**
- Consumes: filesystem state (stamps, .artifacts-promoted, attic)
- Consumes: `gh issue view` for issue state (mocked in tests)
- Produces: `GATE=pass|fail|warn` + `MISSING=` + `REASON=` on stdout

- [ ] **Step 1: Write failing tests**

```python
import subprocess
import sys
from pathlib import Path
from unittest.mock import patch

SCRIPT = Path(__file__).parent.parent / "work-end" / "phase_b_gate.py"


class TestPhaseBGate:
    def test_all_pass(self, tmp_path):
        # Setup: stamps exist, attic exists, mock gh returns CLOSED
        slot_dir = tmp_path / "worktrees" / "attic" / "72"
        slot_dir.mkdir(parents=True)
        repo = slot_dir / "engine"
        repo.mkdir()
        (repo / ".git").mkdir()
        # Fake stamp commit via a marker file
        (slot_dir / ".stamps-verified").write_text("engine\n")
        (slot_dir / "design" / ".artifacts-promoted").parent.mkdir(parents=True)
        (slot_dir / "design" / ".artifacts-promoted").write_text("ok\n")

        # Test via import, not subprocess, for mockability
        # ... (implementation detail)

    def test_missing_stamps(self, tmp_path):
        # No stamp markers → GATE=fail MISSING=stamps:engine
        pass

    def test_github_unreachable(self, tmp_path):
        # gh returns error → GATE=warn REASON=github_unreachable
        pass

    def test_not_archived(self, tmp_path):
        # Slot in worktrees/72 not worktrees/attic/72 → GATE=fail MISSING=archive
        pass
```

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Implement phase_b_gate.py**

Create `work-end/phase_b_gate.py`:

```python
#!/usr/bin/env python3
"""
Phase B completion gate for slot-mode work-end.

Usage:
    python3 phase_b_gate.py <slot_dir> <family_root> covers=N,M issue-repo=org/repo

Output:
    GATE=pass | GATE=fail MISSING=... | GATE=warn MISSING=... REASON=...
"""
import subprocess
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
from common import parse_args


def _check_stamps(slot_dir: Path, repos: list[str]) -> list[str]:
    missing = []
    for repo_name in repos:
        repo = slot_dir / repo_name
        if not repo.is_dir():
            continue
        r = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%s"],
            capture_output=True, text=True,
        )
        if r.returncode != 0 or not r.stdout.strip().startswith("chore: branch closed"):
            missing.append(repo_name)
    return missing


def _check_issues(issue_repo: str, covers: list[int]) -> tuple[list[int], bool]:
    open_issues = []
    unreachable = False
    for n in covers:
        r = subprocess.run(
            ["gh", "issue", "view", str(n), "--repo", issue_repo,
             "--json", "state", "--jq", ".state"],
            capture_output=True, text=True, timeout=15,
        )
        if r.returncode != 0:
            unreachable = True
            open_issues.append(n)
        elif r.stdout.strip() != "CLOSED":
            open_issues.append(n)
    return open_issues, unreachable


def _check_promoted(slot_dir: Path) -> bool:
    for sub in slot_dir.iterdir():
        if sub.is_dir():
            if (sub / "design" / ".artifacts-promoted").exists():
                return True
    return (slot_dir / "design" / ".artifacts-promoted").exists()


def _check_archived(slot_dir: Path) -> bool:
    return "attic" in slot_dir.parts


def main() -> int:
    if len(sys.argv) < 3:
        print(__doc__)
        return 1

    slot_dir = Path(sys.argv[1])
    family_root = Path(sys.argv[2])
    params = parse_args(sys.argv[3:])
    covers_str = params.get("covers", "")
    issue_repo = params.get("issue-repo", "")
    covers = [int(x) for x in covers_str.split(",") if x.strip()]

    repos = [d.name for d in slot_dir.iterdir()
             if d.is_dir() and (d / ".git").exists()]

    missing = []
    reason = ""

    stamp_missing = _check_stamps(slot_dir, repos)
    if stamp_missing:
        missing.append(f"stamps:{','.join(stamp_missing)}")

    if covers and issue_repo:
        open_issues, unreachable = _check_issues(issue_repo, covers)
        if open_issues:
            missing.append(f"issues:{','.join(str(i) for i in open_issues)}")
        if unreachable:
            reason = "github_unreachable"

    if not _check_promoted(slot_dir):
        missing.append("promotion")

    if not _check_archived(slot_dir):
        missing.append("archive")

    if not missing:
        print("GATE=pass")
    elif reason:
        print("GATE=warn")
        print(f"MISSING={','.join(missing)}")
        print(f"REASON={reason}")
    else:
        print("GATE=fail")
        print(f"MISSING={','.join(missing)}")

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run to verify pass**

Run: `python3 -m pytest tests/test_phase_b_gate.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/phase_b_gate.py tests/test_phase_b_gate.py
git commit -m "feat(#151): add phase_b_gate.py — mechanical Phase B verification  Refs #151"
```

---

### Task 6: stack.py epic fields + forward-compatible serializer

**Files:**
- Modify: `project/stack.py`
- Modify or create: `tests/test_stack.py`

**Interfaces:**
- Consumes: `push` CLI args (key=value pairs including `epic_batch`, `epic_active_issue`)
- Produces: `ENTRY_N_EPIC_BATCH=`, `ENTRY_N_EPIC_ACTIVE_ISSUE=` in `list` output

- [ ] **Step 1: Write failing tests**

```python
class TestStackEpicFields:
    def test_push_with_epic_fields(self, tmp_path):
        stack_file = tmp_path / ".pause-stack"
        from stack import cmd_push
        cmd_push(stack_file, [
            "branch=issue-50-epic", "issue=50", "epic_batch=2/4",
            "epic_active_issue=83",
        ])
        content = stack_file.read_text()
        assert "epic_batch: 2/4" in content
        assert "epic_active_issue: 83" in content

    def test_roundtrip_unknown_keys(self, tmp_path):
        stack_file = tmp_path / ".pause-stack"
        from stack import cmd_push, _read_entries
        cmd_push(stack_file, [
            "branch=test", "issue=1", "future_field=value",
        ])
        entries = _read_entries(stack_file)
        assert entries[0]["future_field"] == "value"

    def test_list_includes_epic_fields(self, tmp_path, capsys):
        stack_file = tmp_path / ".pause-stack"
        from stack import cmd_push, cmd_list
        cmd_push(stack_file, [
            "branch=epic-branch", "issue=50",
            "epic_batch=1/3", "epic_active_issue=101",
        ])
        cmd_list(stack_file)
        out = capsys.readouterr().out
        assert "ENTRY_1_EPIC_BATCH=1/3" in out
        assert "ENTRY_1_EPIC_ACTIVE_ISSUE=101" in out

    def test_backward_compat_no_epic_fields(self, tmp_path, capsys):
        stack_file = tmp_path / ".pause-stack"
        stack_file.write_text("- branch: old-branch\n  issue: 5\n  paused: 2026-01-01\n")
        from stack import cmd_list
        cmd_list(stack_file)
        out = capsys.readouterr().out
        assert "ENTRY_1_BRANCH=old-branch" in out
        assert "ENTRY_1_EPIC_BATCH=" in out  # empty, not missing
```

- [ ] **Step 2: Run to verify fail**

- [ ] **Step 3: Fix _entries_to_text() to serialize all keys**

Replace the hardcoded tuple with:

```python
def _entries_to_text(entries: list[dict]) -> str:
    """Serialise entries back to YAML-block format."""
    known_order = ("issue", "paused", "wip_project", "wip_workspace", "slot",
                   "epic_batch", "epic_active_issue")
    lines = []
    for e in entries:
        lines.append(f"- branch: {e.get('branch', '')}")
        written = set()
        for key in known_order:
            if key in e:
                lines.append(f"  {key}: {e[key]}")
                written.add(key)
        for key in sorted(e.keys()):
            if key not in written and key != "branch":
                lines.append(f"  {key}: {e[key]}")
    return "\n".join(lines) + ("\n" if lines else "")
```

- [ ] **Step 4: Fix cmd_list() to output epic fields**

Replace the hardcoded output with:

```python
def cmd_list(stack_file: Path) -> int:
    entries = _read_entries(stack_file)
    print(f"ENTRY_COUNT={len(entries)}")
    known_keys = ("BRANCH", "ISSUE", "PAUSED", "WIP_PROJECT", "WIP_WORKSPACE",
                  "SLOT", "EPIC_BATCH", "EPIC_ACTIVE_ISSUE")
    for i, e in enumerate(entries, 1):
        for display_key in known_keys:
            dict_key = display_key.lower()
            print(f"ENTRY_{i}_{display_key}={e.get(dict_key, '')}")
    return 0
```

- [ ] **Step 5: Run to verify pass**

Run: `python3 -m pytest tests/test_stack.py -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add project/stack.py tests/test_stack.py
git commit -m "feat(#151): stack.py — epic fields + forward-compatible serializer  Refs #151"
```

---

### Task 7: Skill documentation updates (J1-J5, C, K)

**Files:**
- Modify: `work-end/SKILL.md` — Step 0b epic gate + stamp verification note
- Modify: `work/SKILL.md` — Step 4 "end" epic annotation
- Modify: `work-start/SKILL.md` — Epic Overlay → Step 3d
- Modify: `work-resume/SKILL.md` — Step 9b epic context
- Modify: `work-pause/SKILL.md` — epic state in push call
- Modify: `work-slot/SKILL.md` — remove push after work-slot next
- Modify: `git-commit/SKILL.md` — slot mode push suppression

No TDD for skill documentation — these are markdown instruction changes.

- [ ] **Step 1: work-end/SKILL.md — Add Step 0b**

After the branch divergence check in Pre-conditions, add:

```markdown
0b. **Epic confirmation gate** — read `IS_EPIC` from ctx.py output.
   If `no`, skip. If `yes`, run:
   ```bash
   python3 ~/.claude/skills/work-slot/epic_manager.py check <EPIC_PATH>
   ```
   Read `EPIC_COMPLETE`, `SAFE_EXIT`, `CURRENT_BATCH`, `TOTAL_BATCHES`,
   `ACTIVE_ISSUE` from output.

   | Check (in order) | UX |
   |-------------------|----|
   | `EPIC_COMPLETE=yes` | Proceed silently |
   | `SAFE_EXIT=yes` | "Batch N/M complete. Safe exit point — close? (y/n)" |
   | Neither | "⚠ Mid-batch (issue #X of batch N/M). Partial close. Continue? (y/confirm-partial)" |
```

- [ ] **Step 2: work-end/SKILL.md — Add stamp verification note**

After the Phase B stamp section, add a note explaining why merge_slot
stamps without verify_stamp.py (push success = content landed).

- [ ] **Step 3: work/SKILL.md — Step 4 "end" epic annotation**

In Step 4, change the "end" option:
```markdown
If `IS_EPIC=yes` and not `EPIC_COMPLETE`:
> N+1. **end** — ⚠️ epic Batch N of M — close this branch, merge, push, return to main
```

- [ ] **Step 4: work-start/SKILL.md — Number Epic Overlay as Step 3d**

Change "## Epic Overlay" heading to "### Step 3d — Epic Overlay" and update
the text from "After the standard resume steps complete" to "After Step 3c,
check for epic context:".

- [ ] **Step 5: work-resume/SKILL.md — Add Step 9b**

After Step 9 (work-start pre-checks), add:
```markdown
### Step 9b — Epic context
If the stack entry has `epic_batch`: display:
> `Epic — Batch N, active: #M`
```

- [ ] **Step 6: work-pause/SKILL.md — Add epic fields to push**

In the push step, add `epic_batch` and `epic_active_issue` to the
key=value args when IS_EPIC (read from ctx.py or .meta).

- [ ] **Step 7: work-slot/SKILL.md — Remove push after work-slot next**

In `work-slot next`, after Step 3 (GitHub checkbox), ensure there is NO
instruction to push branches. Add:
```markdown
**Slot mode:** commits stay local in the clone. No pushes until Phase A.
```

- [ ] **Step 8: git-commit/SKILL.md — Slot mode push suppression**

Add a section or note:
```markdown
**Slot mode:** When `IN_SLOT=yes` (detected via `/worktrees/` in project
path), do not offer to push after committing. Commits accumulate locally
in the clone until Phase A squashes and pushes.
```

- [ ] **Step 9: work-start/SKILL.md — Slot mode scaffold push skip**

In Step 10 (Commit and push scaffold), add:
```markdown
**Slot mode:** When `IN_SLOT=yes`, skip the push. The scaffold lives in
the clone only until Phase A.
```

- [ ] **Step 10: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 11: Commit**

```bash
git add work-end/SKILL.md work/SKILL.md work-start/SKILL.md work-resume/SKILL.md work-pause/SKILL.md work-slot/SKILL.md git-commit/SKILL.md
git commit -m "docs(#151): epic lifecycle awareness + slot push discipline in skills  Refs #151"
```

---

### Task 8: Stamp verification audit (K)

No code changes. Read-only verification.

- [ ] **Step 1: Verify land_branch.py always calls verify_stamp.py**

Read `work-end/land_branch.py` `cmd_stamp()` — confirm verify_stamp.py is
called unconditionally (line 144-160). Verified: no conditional bypass.

- [ ] **Step 2: Verify merge_slot() stamp path**

Read `work-slot/slot_manager.py` `merge_slot()` lines 705-716 — stamps are
written only after successful push. Push success = content landed. Different
verification mechanism, both valid.

- [ ] **Step 3: Note in commit message**

Already documented in work-end/SKILL.md (Task 7, Step 2).

---

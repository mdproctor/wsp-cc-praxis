# State Hardening Implementation Plan (Phase 1 of #382)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #382 — Unified mechanical pipeline
**Issue group:** #382

**Goal:** Make every state write go through one validated gate, commit state changes to git so they survive branch operations, and close all bypass paths that allow invalid states.

**Architecture:** `commit_transition()` becomes the sole gate for state writes. It validates against VALID_STATES, writes to disk, AND commits to git. Bypass paths (plan_manager set-state, elevate_plan_inline, branch_cleanup _reset_plan_state) are replaced with proper lifecycle transitions. `write_field` adds a safety-net validation for the `state` field. `.plan` is removed from the LIFECYCLE_FILES strip list so post-land lifecycle transitions can complete.

**Tech Stack:** Python 3, pytest, subprocess (git CLI), pathlib

## Global Constraints

- All state writes must go through `lifecycle.py commit_transition()`
- `write_field` rejects state values not in `VALID_STATES` (safety net)
- `.plan` stays on the feature branch permanently (not stripped before merge)
- Cleanup removes `.plan` from main after merge
- Existing test suites must continue passing
- No changes to the lifecycle state machine semantics (states and transitions stay the same)

---

## Batch 1: Core — commit state to git + validate writes

After this batch: `commit_transition` commits `.plan` to git. `write_field`
rejects invalid state values. These are the two foundational changes.

### Task 1: commit_transition commits .plan to git

**Files:**
- Modify: `project/lifecycle.py` — add git commit after write_state
- Create: `tests/test_lifecycle_git_commit.py` — test that commit_transition creates a git commit

**Interfaces:**
- Consumes: `write_state()`, subprocess git calls
- Produces: Every `commit_transition` call creates a git commit on the workspace branch

- [ ] **Step 1: Write failing test**

```python
# tests/test_lifecycle_git_commit.py
"""Tests that commit_transition commits .plan to git."""

import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from lifecycle import (
    TransitionResult,
    commit_transition,
    write_state,
)
from plan_io import write_fields


def _init_git_repo(path: Path) -> None:
    """Initialize a git repo with a committed .plan."""
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "test@test.com"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
    plan = path / ".plan"
    plan.write_text(
        "# Work Plan\n\n## State\n"
        "branch: test-branch\nstate: active\ncovers: 1\n\n## Queue\n- [ ] test#1 — Test\n"
    )
    subprocess.run(["git", "-C", str(path), "add", ".plan"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "initial"], capture_output=True)


class TestCommitTransitionGitCommit:
    def test_creates_git_commit_on_transition(self, tmp_path):
        _init_git_repo(tmp_path)
        plan = tmp_path / ".plan"

        result = TransitionResult(
            from_state="active",
            new_state="closing:review",
            event="work_end",
        )

        commit_transition(plan, result)

        log = subprocess.run(
            ["git", "-C", str(tmp_path), "log", "--oneline", "-1"],
            capture_output=True, text=True,
        )
        assert "lifecycle" in log.stdout.lower() or "closing:review" in log.stdout

        show = subprocess.run(
            ["git", "-C", str(tmp_path), "show", "HEAD:.plan"],
            capture_output=True, text=True,
        )
        assert "state: closing:review" in show.stdout

    def test_state_survives_checkout(self, tmp_path):
        _init_git_repo(tmp_path)
        plan = tmp_path / ".plan"

        result = TransitionResult(
            from_state="active",
            new_state="closing:review",
            event="work_end",
        )
        commit_transition(plan, result)

        subprocess.run(
            ["git", "-C", str(tmp_path), "checkout", "-b", "other"],
            capture_output=True,
        )
        subprocess.run(
            ["git", "-C", str(tmp_path), "checkout", "-"],
            capture_output=True,
        )

        content = plan.read_text()
        assert "state: closing:review" in content

    def test_no_commit_when_state_is_idle(self, tmp_path):
        """idle transition deletes .plan — no commit needed."""
        _init_git_repo(tmp_path)
        plan = tmp_path / ".plan"

        before_count = subprocess.run(
            ["git", "-C", str(tmp_path), "rev-list", "--count", "HEAD"],
            capture_output=True, text=True,
        ).stdout.strip()

        result = TransitionResult(
            from_state="closing:stamped",
            new_state="idle",
            event="cleanup_pass",
        )
        # idle transition expects certain state
        write_state(plan, "closing:stamped")
        subprocess.run(["git", "-C", str(tmp_path), "add", ".plan"], capture_output=True)
        subprocess.run(["git", "-C", str(tmp_path), "commit", "-m", "prep"], capture_output=True)

        commit_transition(plan, result, evidence={"repos_on_main": {"test": True}, "work_items_ended": True})

        # .plan should be deleted for idle transition
        assert not plan.exists()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle_git_commit.py -v --tb=short`
Expected: FAIL — commit_transition doesn't create git commits yet

- [ ] **Step 3: Implement git commit in commit_transition**

In `project/lifecycle.py`, add git commit after `write_state` (line 427):

```python
        if result.new_state != 'idle':
            write_state(plan_path, result.new_state)
            # Commit state change to git — makes state durable across branch ops
            _commit_plan_to_git(plan_path, result)
```

Add the helper function:

```python
def _commit_plan_to_git(plan_path: Path, result: TransitionResult) -> None:
    """Commit .plan state change to git. Best-effort — never blocks."""
    workspace = plan_path.parent
    try:
        _sp.run(
            ["git", "-C", str(workspace), "add", str(plan_path.name)],
            capture_output=True, timeout=10,
        )
        _sp.run(
            ["git", "-C", str(workspace), "commit", "-m",
             f"chore: lifecycle {result.from_state} → {result.new_state}"],
            capture_output=True, timeout=10,
        )
    except Exception:
        pass  # best-effort — don't block lifecycle on git failure
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_lifecycle_git_commit.py tests/test_lifecycle.py -v --tb=short`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle_git_commit.py
git commit -m "feat(#382): commit_transition commits .plan to git  Refs #382"
```

### Task 2: write_field validates state field

**Files:**
- Modify: `project/plan_io.py` — add VALID_STATES validation
- Modify: `tests/test_lifecycle.py` or create `tests/test_plan_io_validation.py`

**Interfaces:**
- Consumes: `VALID_STATES` from `lifecycle.py`
- Produces: `write_field` raises `ValueError` when writing invalid state

- [ ] **Step 1: Write failing test**

```python
# tests/test_plan_io_validation.py
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from plan_io import write_field


class TestWriteFieldStateValidation:
    def test_rejects_invalid_state(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nstate: active\n\n## Queue\n")
        with pytest.raises(ValueError, match="Invalid state"):
            write_field(plan, "state", "closing:landed")

    def test_accepts_valid_state(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nstate: active\n\n## Queue\n")
        write_field(plan, "state", "closing:review")
        assert "state: closing:review" in plan.read_text()

    def test_allows_non_state_fields(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nbranch: old\n\n## Queue\n")
        write_field(plan, "branch", "anything-goes")
        assert "branch: anything-goes" in plan.read_text()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_plan_io_validation.py -v --tb=short`
Expected: FAIL — `test_rejects_invalid_state` does NOT raise

- [ ] **Step 3: Add validation to write_field**

In `project/plan_io.py`, modify `write_field`:

```python
def write_field(plan_path: Path, field_name: str, value: str) -> None:
    if field_name == "state":
        _validate_state_value(value)
    write_fields(plan_path, {field_name: value})


def _validate_state_value(value: str) -> None:
    """Reject state values not in VALID_STATES. Safety net for bypass paths."""
    try:
        from lifecycle import VALID_STATES
        if value not in VALID_STATES:
            raise ValueError(
                f"Invalid state '{value}'. Valid states: {sorted(VALID_STATES)}"
            )
    except ImportError:
        pass  # lifecycle not available — skip validation
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_plan_io_validation.py tests/test_lifecycle.py -v --tb=short`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add project/plan_io.py tests/test_plan_io_validation.py
git commit -m "feat(#382): write_field validates state against VALID_STATES  Refs #382"
```

---

## Batch 2: Close bypass paths

After this batch: all code paths that write state go through `commit_transition`.
No more direct string manipulation of the state field.

### Task 3: Remove .plan from LIFECYCLE_FILES + fix cleanup

**Files:**
- Modify: `work-end/land_flow.py` — remove `.plan` from LIFECYCLE_FILES
- Modify: `work-end/branch_cleanup.py` — ensure cleanup removes `.plan` from main
- Modify: `tests/test_work_end_scripts.py` — update tests

**Interfaces:**
- Consumes: LIFECYCLE_FILES list
- Produces: `.plan` not stripped before merge; cleaned from main during cleanup

- [ ] **Step 1: Write failing test**

```python
# Add to tests/test_work_end_scripts.py or create tests/test_plan_not_stripped.py

class TestPlanNotInLifecycleFiles:
    def test_plan_not_in_lifecycle_files(self):
        import sys
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        from land_flow import LIFECYCLE_FILES
        assert ".plan" not in LIFECYCLE_FILES, ".plan should not be stripped before merge"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_plan_not_stripped.py -v`
Expected: FAIL — `.plan` IS in LIFECYCLE_FILES

- [ ] **Step 3: Remove .plan from LIFECYCLE_FILES**

In `work-end/land_flow.py`, remove `.plan` from the list:

```python
LIFECYCLE_FILES = [
    "JOURNAL.md", ".execute-progress",
    ".land-ledger.jsonl", ".artifacts-promoted",
    ".close-progress", ".close-report.json",
    ".close-log.jsonl", ".wrap-log.jsonl",
]
```

- [ ] **Step 4: Verify cleanup still removes .plan from main**

Check `branch_cleanup.py` `cleanup_scaffold` — `.plan` is already handled:
line 96-97 adds `.plan` to `scaffold_names` when `preserve_plan` is False.
This means cleanup removes `.plan` from main when no queue items remain. ✅

- [ ] **Step 5: Run tests**

Run: `python3 -m pytest tests/test_plan_not_stripped.py tests/test_work_end_orchestrator.py -v --tb=short`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/land_flow.py tests/test_plan_not_stripped.py
git commit -m "fix(#382): remove .plan from LIFECYCLE_FILES strip list  Refs #382"
```

### Task 4: Replace elevate_plan_inline and _reset_plan_state with lifecycle transitions

**Files:**
- Modify: `project/lifecycle.py` — add `elevate` transition to TRANSITION_TABLE
- Modify: `work-end/work_end_orchestrator.py` — replace string manipulation with commit_transition call
- Modify: `work-end/branch_cleanup.py` — replace `_reset_plan_state` with commit_transition call

**Interfaces:**
- Consumes: New `elevate` and `reset_for_next` transitions in TRANSITION_TABLE
- Produces: No more direct state string manipulation

- [ ] **Step 1: Write failing test for elevate transition**

```python
# Add to tests/test_lifecycle.py

class TestElevateTransition:
    def test_elevate_transition_exists(self):
        from lifecycle import TRANSITION_TABLE
        assert ("closing:stamped", "elevate") in TRANSITION_TABLE

    def test_elevate_goes_to_active(self):
        from lifecycle import TRANSITION_TABLE
        new_state, _, _ = TRANSITION_TABLE[("closing:stamped", "elevate")]
        assert new_state == "active"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle.py::TestElevateTransition -v`
Expected: FAIL — no `elevate` transition in table

- [ ] **Step 3: Add transitions to TRANSITION_TABLE**

In `project/lifecycle.py`, add to TRANSITION_TABLE:

```python
    # Elevate (slot plan promotion — reset closing state to active)
    ('closing:stamped', 'elevate'):      ('active',            ['elevate_plan_to_slot'],          []),
    # Reset for next issue cycle (cleanup resets closing → active when queue has items)
    ('closing:review', 'reset_for_next'):  ('active',          ['clear_closing_markers'],         []),
    ('closing:verified', 'reset_for_next'):('active',          ['clear_closing_markers'],         []),
    ('closing:promoted', 'reset_for_next'):('active',          ['clear_closing_markers'],         []),
    ('closing:pushed', 'reset_for_next'):  ('active',          ['clear_closing_markers'],         []),
    ('closing:merged', 'reset_for_next'):  ('active',          ['clear_closing_markers'],         []),
    ('closing:stamped', 'reset_for_next'): ('active',          ['clear_closing_markers'],         []),
```

- [ ] **Step 4: Replace elevate_plan_inline in work_end_orchestrator.py**

Replace the string manipulation at line 510-518 with a `commit_transition` call:

```python
def _elevate_plan_inline(ctx: OrchestratorContext) -> dict[str, str]:
    import shutil
    ws_plan = ctx.workspace / ".plan"
    slot_plan = ctx.slot_path / ".plan"

    if ctx.slot_path:
        _slot_dir = str(Path(__file__).resolve().parent.parent / "work-slot")
        if _slot_dir not in sys.path:
            sys.path.insert(0, _slot_dir)
        from slot_claude import write_occupant_pid
        if not ctx.dry_run:
            write_occupant_pid(ctx.slot_path)
        elif ctx.call_log is not None:
            ctx.call_log.append(["(internal)", "write_occupant_pid", str(ctx.slot_path)])

    if not ws_plan.exists():
        return {"ELEVATED": "no", "REASON": "no_plan"}

    _project_dir = str(Path(__file__).resolve().parent.parent / "project")
    if _project_dir not in sys.path:
        sys.path.insert(0, _project_dir)
    try:
        from plan_io import read_plan, has_uncompleted_items
        state = read_plan(ws_plan)
        if state is None or not has_uncompleted_items(state):
            if slot_plan.exists() and not ctx.dry_run:
                slot_plan.unlink()
            return {"ELEVATED": "no", "REASON": "queue_drained"}
    except Exception:
        return {"ELEVATED": "no", "REASON": "parse_error"}

    if not ctx.dry_run:
        # Use lifecycle transition instead of string manipulation
        from lifecycle import TransitionResult, commit_transition, read_state
        current = read_state(ws_plan)
        if current and current.startswith("closing:"):
            result = TransitionResult(
                from_state=current, new_state="active", event="elevate",
            )
            try:
                commit_transition(ws_plan, result)
            except Exception:
                pass  # fall back to copy as-is
        shutil.copy2(ws_plan, slot_plan)
    elif ctx.call_log is not None:
        ctx.call_log.append(["(internal)", "elevate_plan", str(slot_plan)])

    return {"ELEVATED": "yes", "SLOT_PLAN": str(slot_plan)}
```

- [ ] **Step 5: Replace _reset_plan_state in branch_cleanup.py**

Replace the string manipulation at line 62-73:

```python
def _reset_plan_state(plan_path: Path) -> None:
    """Reset .plan state from closing:* to active via lifecycle transition."""
    if not plan_path.exists():
        return
    _project_dir = str(Path(__file__).resolve().parent.parent / "project")
    if _project_dir not in sys.path:
        sys.path.insert(0, _project_dir)
    from lifecycle import TransitionResult, commit_transition, read_state
    current = read_state(plan_path)
    if current and current.startswith("closing:"):
        result = TransitionResult(
            from_state=current, new_state="active", event="reset_for_next",
        )
        try:
            commit_transition(plan_path, result)
        except Exception:
            pass  # best-effort
```

- [ ] **Step 6: Run tests**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_work_end_orchestrator.py -v --tb=short`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add project/lifecycle.py work-end/work_end_orchestrator.py work-end/branch_cleanup.py
git commit -m "fix(#382): replace state bypass paths with lifecycle transitions  Refs #382"
```

### Task 5: Remove plan_manager set-state command

**Files:**
- Modify: `work-slot/plan_manager.py` — remove `set-state` command
- Modify: tests if any use `set-state`

**Interfaces:**
- Consumes: Nothing new
- Produces: `set-state` command returns an error pointing to `lifecycle.py commit-transition`

- [ ] **Step 1: Write failing test**

```python
# tests/test_plan_manager_no_set_state.py
import subprocess
import sys
from pathlib import Path

def test_set_state_rejected(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nstate: active\n\n## Queue\n")
    result = subprocess.run(
        [sys.executable, str(Path(__file__).parent.parent / "work-slot" / "plan_manager.py"),
         "set-state", str(plan), "key=state", "value=closing:landed"],
        capture_output=True, text=True,
    )
    assert result.returncode != 0 or "removed" in result.stdout.lower() or "lifecycle" in result.stdout.lower()
```

- [ ] **Step 2: Replace set-state with error message**

In `work-slot/plan_manager.py`, replace the `set-state` handler:

```python
    elif command == "set-state":
        key = opts.get("key", "")
        if key == "state":
            print("ERROR=state writes must go through lifecycle.py commit-transition")
            print("ERROR_DETAIL=plan_manager set-state for the 'state' field is removed. Use: python3 project/lifecycle.py commit-transition <plan> from_state=<current> new_state=<target> event=<event>")
            return 1
        # Allow non-state field writes (branch, covers, etc.)
        tree = parse_plan(plan_path)
        tree.state[key] = value
        rewrite_plan(plan_path, tree)
        print(f"SET={key}={value}")
        return 0
```

- [ ] **Step 3: Run tests**

Run: `python3 -m pytest tests/test_plan_manager_no_set_state.py -v`
Expected: PASS

- [ ] **Step 4: Check if any existing code calls set-state with key=state**

```bash
grep -rn "set-state.*key=state\|set-state.*state=" tests/ work-end/ work-slot/ project/ --include="*.py"
```

Fix any callers found.

- [ ] **Step 5: Commit**

```bash
git add work-slot/plan_manager.py tests/test_plan_manager_no_set_state.py
git commit -m "fix(#382): block state writes via plan_manager set-state  Refs #382"
```

---

## Batch 3: Integration verification

After this batch: end-to-end tests prove state survives branch operations
and invalid states are rejected at all entry points.

### Task 6: State durability integration tests

**Files:**
- Create: `tests/test_state_durability.py`

**Interfaces:**
- Consumes: `commit_transition`, `write_field`, git operations
- Produces: Tests proving state survives rebase, checkout, and merge

- [ ] **Step 1: Write integration tests**

```python
# tests/test_state_durability.py
"""Integration tests for state durability across git operations."""

import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from lifecycle import TransitionResult, commit_transition, read_state, write_state
from plan_io import write_field


def _git(repo, *args):
    return subprocess.run(
        ["git", "-C", str(repo), *args],
        capture_output=True, text=True, timeout=10,
    )


def _init_workspace(path):
    _git(path, "init")
    _git(path, "config", "user.email", "test@test.com")
    _git(path, "config", "user.name", "Test")
    plan = path / ".plan"
    plan.write_text(
        "# Work Plan\n\n## State\n"
        "branch: test-branch\nstate: active\ncovers: 1\n\n"
        "## Queue\n- [ ] test#1 — Test\n"
    )
    _git(path, "add", ".plan")
    _git(path, "commit", "-m", "scaffold")
    _git(path, "checkout", "-b", "test-branch")


class TestStateSurvivesBranchOps:
    def test_state_survives_checkout_and_return(self, tmp_path):
        _init_workspace(tmp_path)
        plan = tmp_path / ".plan"

        result = TransitionResult("active", "closing:review", "work_end")
        commit_transition(plan, result)
        assert read_state(plan) == "closing:review"

        _git(tmp_path, "checkout", "main")
        _git(tmp_path, "checkout", "test-branch")

        assert read_state(plan) == "closing:review"

    def test_state_survives_stash(self, tmp_path):
        _init_workspace(tmp_path)
        plan = tmp_path / ".plan"

        result = TransitionResult("active", "closing:review", "work_end")
        commit_transition(plan, result)

        (tmp_path / "other.txt").write_text("change")
        _git(tmp_path, "add", "other.txt")
        _git(tmp_path, "stash", "push", "-u")
        _git(tmp_path, "stash", "pop")

        assert read_state(plan) == "closing:review"


class TestInvalidStateRejection:
    def test_write_field_rejects_closing_landed(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nstate: active\n\n## Queue\n")
        with pytest.raises(ValueError):
            write_field(plan, "state", "closing:landed")

    def test_write_field_rejects_arbitrary_string(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nstate: active\n\n## Queue\n")
        with pytest.raises(ValueError):
            write_field(plan, "state", "banana")

    def test_write_field_allows_valid_closing_states(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("## State\nstate: active\n\n## Queue\n")
        write_field(plan, "state", "closing:review")
        assert "state: closing:review" in plan.read_text()
```

- [ ] **Step 2: Run tests**

Run: `python3 -m pytest tests/test_state_durability.py -v --tb=short`
Expected: All PASS (if previous tasks are complete)

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: No regressions

- [ ] **Step 4: Commit**

```bash
git add tests/test_state_durability.py
git commit -m "test(#382): state durability integration tests  Refs #382"
```

---

## References

- `specs/issue-382-unified-pipeline/2026-09-27-unified-pipeline-design.md` — design spec (Section A)
- `project/lifecycle.py` — state machine, commit_transition (lines 383-441)
- `project/plan_io.py` — write_field, write_fields (lines 141-190)
- `work-end/land_flow.py` — LIFECYCLE_FILES (lines 98-103)
- `work-end/work_end_orchestrator.py` — elevate_plan_inline (lines 475-522)
- `work-end/branch_cleanup.py` — _reset_plan_state (lines 62-73), cleanup_scaffold (lines 76-119)
- `work-slot/plan_manager.py` — set-state command (lines 1339-1349)
- `tests/test_lifecycle.py` — existing lifecycle test patterns
- `docs/protocols/evidence-before-claims.md`
- `docs/protocols/externalised-scripts-require-tests.md`
- GitHub #382 — unified mechanical pipeline
- Incident: `closing:landed` from plan_manager set-state bypass
- Incident: orchestrator infinite loop from uncommitted state changes

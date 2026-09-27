# Idempotent Work-End Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #379 — Make work-end pipeline crash-safe through idempotent steps
**Issue group:** #379

**Goal:** Make every mechanical step in the work-end pipeline follow check-execute-verify so that re-running after any failure converges to the correct end state.

**Architecture:** Add `postcondition_fn` to `StepDef`. The orchestrator engine's `run_loop` calls the postcondition before executing (skip if met) and after executing (fail if not met). Postcondition functions live in `verification/step_postconditions.py`. Disk markers (.landed, .artifacts-promoted) are kept but only written after postcondition verification.

**Tech Stack:** Python 3, pytest, subprocess (git CLI), pathlib

## Global Constraints

- All new `.py` scripts must ship with pytest tests (protocol: externalised-scripts-require-tests)
- Postcondition functions are pure checks — no side effects, no fixes
- Steps without a postcondition_fn follow the current flow unchanged (incremental adoption)
- Existing test suites must continue passing (`python3 -m pytest tests/ -v`)
- No changes to the SKILL.md orchestration flow or lifecycle state machine

---

## Batch 1: Postcondition Library

After this batch: `verification/step_postconditions.py` exists with all postcondition
functions, fully tested. No behavioral changes yet — the functions are not wired in.

### Task 1: Create step postcondition functions + tests

**Files:**
- Create: `verification/step_postconditions.py`
- Create: `tests/test_step_postconditions.py`

**Interfaces:**
- Consumes: `verification.git()` helper from `verification/__init__.py`
- Produces: Functions with signature `(ctx) -> bool` where `ctx` is any object with
  attributes `project: Path`, `workspace: Path`, `branch: str`, `base_branch: str`,
  `slot_path: Path | None`, `family_root: Path | None`, `slot_num: str`,
  `landed_shas: dict[str, str]`, `covers: str`, `issue_repo: str`

- [ ] **Step 1: Write failing tests for rebase_postcondition**

```python
# tests/test_step_postconditions.py
import subprocess
import sys
from dataclasses import dataclass, field
from pathlib import Path
from unittest.mock import patch, MagicMock

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "verification"))


@dataclass
class FakeCtx:
    project: Path
    workspace: Path
    branch: str = "issue-379-test"
    base_branch: str = "main"
    slot_path: Path | None = None
    family_root: Path | None = None
    slot_num: str = ""
    landed_shas: dict = field(default_factory=dict)
    covers: str = ""
    issue_repo: str = ""
    in_slot: bool = False
    on_main: bool = False
    current_repo_project: Path | None = None


class TestRebasePostcondition:
    def test_returns_true_when_base_is_ancestor(self, tmp_path):
        from step_postconditions import rebase_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0)
            assert rebase_postcondition(ctx) is True

    def test_returns_false_when_base_not_ancestor(self, tmp_path):
        from step_postconditions import rebase_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=1)
            assert rebase_postcondition(ctx) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_step_postconditions.py::TestRebasePostcondition -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'step_postconditions'`

- [ ] **Step 3: Write failing tests for remaining postconditions**

Add these test classes to `tests/test_step_postconditions.py`:

```python
class TestPushPostcondition:
    def test_returns_true_when_sha_on_remote(self, tmp_path):
        from step_postconditions import push_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path,
                      landed_shas={"myrepo": "abc123"})
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0)
            assert push_postcondition(ctx) is True

    def test_returns_false_when_no_landed_sha(self, tmp_path):
        from step_postconditions import push_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        assert push_postcondition(ctx) is False

    def test_returns_false_when_sha_not_on_remote(self, tmp_path):
        from step_postconditions import push_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path,
                      landed_shas={"myrepo": "abc123"})
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=1)
            assert push_postcondition(ctx) is False


class TestStampPostcondition:
    def test_returns_true_when_stamp_with_valid_sha(self, tmp_path):
        from step_postconditions import stamp_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        with patch("step_postconditions._git") as mock_git:
            def side_effect(*args):
                if "log" in args:
                    return MagicMock(returncode=0,
                                    stdout="chore: branch closed — landed as abc123 on main")
                if "merge-base" in args:
                    return MagicMock(returncode=0)
                return MagicMock(returncode=0)
            mock_git.side_effect = side_effect
            assert stamp_postcondition(ctx) is True

    def test_returns_false_when_no_stamp(self, tmp_path):
        from step_postconditions import stamp_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0,
                                              stdout="feat: some feature")
            assert stamp_postcondition(ctx) is False


class TestPromotePostcondition:
    def test_returns_true_when_stamp_exists(self, tmp_path):
        from step_postconditions import promote_postcondition
        (tmp_path / ".artifacts-promoted").write_text("timestamp=2026-01-01\n")
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        assert promote_postcondition(ctx) is True

    def test_returns_false_when_no_stamp(self, tmp_path):
        from step_postconditions import promote_postcondition
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path)
        assert promote_postcondition(ctx) is False


class TestArchiveMovePostcondition:
    def test_returns_true_when_in_attic(self, tmp_path):
        from step_postconditions import archive_move_postcondition
        family = tmp_path / "family"
        family.mkdir()
        attic = family / "attic" / "42"
        attic.mkdir(parents=True)
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path,
                      slot_path=family / "slots" / "42",
                      family_root=family, slot_num="42")
        assert archive_move_postcondition(ctx) is True

    def test_returns_false_when_still_in_slots(self, tmp_path):
        from step_postconditions import archive_move_postcondition
        family = tmp_path / "family"
        slots = family / "slots" / "42"
        slots.mkdir(parents=True)
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path,
                      slot_path=slots, family_root=family, slot_num="42")
        assert archive_move_postcondition(ctx) is False


class TestLandedMarkerPostcondition:
    def test_returns_true_with_populated_shas(self, tmp_path):
        from step_postconditions import landed_marker_postcondition
        slot = tmp_path / "slot"
        slot.mkdir()
        (slot / ".landed").write_text("landed_shas=repo1:abc123,repo2:def456\n")
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path, slot_path=slot)
        assert landed_marker_postcondition(ctx) is True

    def test_returns_false_with_empty_shas(self, tmp_path):
        from step_postconditions import landed_marker_postcondition
        slot = tmp_path / "slot"
        slot.mkdir()
        (slot / ".landed").write_text("landed_shas=\n")
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path, slot_path=slot)
        assert landed_marker_postcondition(ctx) is False

    def test_returns_false_when_no_file(self, tmp_path):
        from step_postconditions import landed_marker_postcondition
        slot = tmp_path / "slot"
        slot.mkdir()
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path, slot_path=slot)
        assert landed_marker_postcondition(ctx) is False


class TestCheckoutMainPostcondition:
    def test_returns_true_when_both_on_main(self, tmp_path):
        from step_postconditions import checkout_main_postcondition
        proj = tmp_path / "proj"
        ws = tmp_path / "ws"
        proj.mkdir()
        ws.mkdir()
        ctx = FakeCtx(project=proj, workspace=ws)
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="main\n")
            assert checkout_main_postcondition(ctx) is True

    def test_returns_false_when_project_on_branch(self, tmp_path):
        from step_postconditions import checkout_main_postcondition
        proj = tmp_path / "proj"
        ws = tmp_path / "ws"
        proj.mkdir()
        ws.mkdir()
        ctx = FakeCtx(project=proj, workspace=ws)
        with patch("step_postconditions._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="issue-379\n")
            assert checkout_main_postcondition(ctx) is False


class TestWriteMarkerPostcondition:
    def test_returns_true_when_marker_exists(self, tmp_path):
        from step_postconditions import write_marker_postcondition
        slot = tmp_path / "slot"
        slot.mkdir()
        (slot / ".phase-a-complete").write_text("branch=test\n")
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path, slot_path=slot)
        assert write_marker_postcondition(ctx) is True

    def test_returns_false_when_no_marker(self, tmp_path):
        from step_postconditions import write_marker_postcondition
        slot = tmp_path / "slot"
        slot.mkdir()
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path, slot_path=slot)
        assert write_marker_postcondition(ctx) is False
```

- [ ] **Step 4: Write the implementation**

```python
# verification/step_postconditions.py
"""Step-level postcondition checks for work-end pipeline idempotency.

Each function checks whether a specific step's side effect has already
been achieved. Used by the orchestrator engine in the check-execute-verify
pattern: check before executing (skip if met), verify after executing
(fail if not met).

All functions: (ctx) -> bool. Pure checks, no side effects.
"""
from __future__ import annotations

import re
import subprocess
from pathlib import Path


def _git(repo_path, *args: str) -> subprocess.CompletedProcess:
    return subprocess.run(
        ["git", "-C", str(repo_path), *args],
        capture_output=True, text=True, timeout=10,
    )


def rebase_postcondition(ctx) -> bool:
    project = getattr(ctx, "current_repo_project", None) or ctx.project
    result = _git(project, "merge-base", "--is-ancestor",
                  ctx.base_branch, ctx.branch)
    return result.returncode == 0


def push_postcondition(ctx) -> bool:
    project = getattr(ctx, "current_repo_project", None) or ctx.project
    repo_name = project.name
    sha = ctx.landed_shas.get(repo_name, "")
    if not sha:
        return False
    result = _git(project, "merge-base", "--is-ancestor",
                  sha, f"origin/{ctx.base_branch}")
    return result.returncode == 0


def stamp_postcondition(ctx) -> bool:
    project = getattr(ctx, "current_repo_project", None) or ctx.project
    tip = _git(project, "log", "-1", "--format=%s", ctx.branch)
    if tip.returncode != 0:
        return False
    if not tip.stdout.strip().startswith("chore: branch closed"):
        return False
    sha_match = re.search(r"landed as ([0-9a-f]+)", tip.stdout.strip())
    if not sha_match:
        return True
    sha = sha_match.group(1)
    check = _git(project, "merge-base", "--is-ancestor", sha, ctx.base_branch)
    return check.returncode == 0


def promote_postcondition(ctx) -> bool:
    ws = getattr(ctx, "current_repo_workspace", None) or ctx.workspace
    return (ws / ".artifacts-promoted").exists()


def archive_move_postcondition(ctx) -> bool:
    if not ctx.slot_path or not ctx.family_root:
        return False
    attic = ctx.family_root / "attic" / ctx.slot_num
    return attic.is_dir() and not ctx.slot_path.is_dir()


def landed_marker_postcondition(ctx) -> bool:
    if not ctx.slot_path:
        return False
    landed = ctx.slot_path / ".landed"
    if not landed.exists():
        return False
    content = landed.read_text()
    if "landed_shas=" not in content:
        return False
    for line in content.splitlines():
        if line.startswith("landed_shas="):
            shas = line.split("=", 1)[1]
            if not shas.strip():
                return False
            pairs = [p for p in shas.split(",") if ":" in p]
            return len(pairs) > 0 and all(
                p.split(":", 1)[1].strip() for p in pairs
            )
    return False


def checkout_main_postcondition(ctx) -> bool:
    for repo in [ctx.project, ctx.workspace]:
        result = _git(repo, "rev-parse", "--abbrev-ref", "HEAD")
        if result.returncode != 0 or result.stdout.strip() != "main":
            return False
    return True


def write_marker_postcondition(ctx) -> bool:
    if not ctx.slot_path:
        return False
    return (ctx.slot_path / ".phase-a-complete").exists()


def issues_closed_postcondition(ctx) -> bool:
    if not ctx.covers or not ctx.issue_repo:
        return True
    import sys
    _project_dir = str(Path(__file__).resolve().parent.parent / "project")
    if _project_dir not in sys.path:
        sys.path.insert(0, _project_dir)
    from plan_io import parse_covers
    for issue_num in parse_covers(ctx.covers):
        result = subprocess.run(
            ["gh", "issue", "view", str(issue_num), "--repo", ctx.issue_repo,
             "--json", "state", "--jq", ".state"],
            capture_output=True, text=True, timeout=10,
        )
        if result.returncode != 0 or result.stdout.strip() != "CLOSED":
            return False
    return True


def cleanup_scaffold_postcondition(ctx) -> bool:
    ws = ctx.workspace
    scaffold = ["JOURNAL.md", ".execute-progress", ".land-ledger.jsonl",
                ".artifacts-promoted"]
    return not any((ws / f).exists() for f in scaffold)
```

- [ ] **Step 5: Run all postcondition tests**

Run: `python3 -m pytest tests/test_step_postconditions.py -v`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add verification/step_postconditions.py tests/test_step_postconditions.py
git commit -m "feat(#379): add step postcondition functions for check-execute-verify  Refs #379"
```

---

## Batch 2: Engine Extension + Wiring

After this batch: `run_loop` enforces check-execute-verify for all mechanical steps
that have a `postcondition_fn`. Steps without one behave as before. The pipeline
skips already-completed steps and fails explicitly when postconditions aren't met
after execution.

### Task 2: Extend StepDef and run_loop with postcondition support

**Files:**
- Modify: `work-end/shared_steps.py` — add `postcondition_fn` to StepDef
- Modify: `work-end/work_end_orchestrator.py` — add `postcondition_fn` to local StepDef
- Modify: `work-end/orchestrator_engine.py` — check-execute-verify in `run_loop`
- Modify: `tests/test_work_end_orchestrator.py` — add tests for postcondition behavior
- Create: `tests/test_orchestrator_engine_postconditions.py`

**Interfaces:**
- Consumes: `postcondition_fn: Callable | None` on StepDef
- Produces: Modified `run_loop` that calls `postcondition_fn` before and after mechanical step execution

- [ ] **Step 1: Write failing test for postcondition skip**

```python
# tests/test_orchestrator_engine_postconditions.py
import sys
from dataclasses import dataclass, field
from pathlib import Path
from unittest.mock import MagicMock

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from shared_steps import StepDef


@dataclass
class FakeEngineCtx:
    workspace: Path
    project: Path
    branch: str = "test"
    base_branch: str = "main"
    on_main: bool = False
    in_slot: bool = False
    covers: str = ""
    issue_repo: str = ""
    progress: dict = field(default_factory=dict)
    dry_run: bool = False
    call_log: list = field(default_factory=list)
    plan_path: Path | None = None
    slot_path: Path | None = None
    family_root: Path | None = None
    slot_num: str = ""
    last_output: dict = field(default_factory=dict)
    steps_executed: list = field(default_factory=list)

    def done(self, step: str) -> bool:
        return self.progress.get(step) in ("done", "skipped", "skipped_error")


class TestPostconditionSkip:
    """When postcondition is already met, step is skipped."""

    def test_skips_when_postcondition_met(self, tmp_path):
        from orchestrator_engine import run_loop
        from close_progress import write_close_progress

        script_called = []

        def fake_script(ctx):
            script_called.append(True)
            return ["echo", "should not run"]

        step = StepDef(
            name="test_step", phase="test", step_type="mechanical",
            script_fn=fake_script,
            postcondition_fn=lambda ctx: True,  # already done
        )

        ctx = FakeEngineCtx(workspace=tmp_path, project=tmp_path)
        result = run_loop([step], ctx)

        assert result["ACTION"] == "complete"
        assert len(script_called) == 0  # script was NOT called


class TestPostconditionVerify:
    """When postcondition fails after execution, step returns error."""

    def test_fails_when_postcondition_not_met_after_execute(self, tmp_path):
        from orchestrator_engine import run_loop

        step = StepDef(
            name="test_step", phase="test", step_type="mechanical",
            script_fn=lambda ctx: ["echo", "ok"],
            postcondition_fn=lambda ctx: False,  # never met
        )

        ctx = FakeEngineCtx(workspace=tmp_path, project=tmp_path)

        def mock_execute(step, ctx):
            return {}  # "succeeds" but postcondition will fail

        result = run_loop([step], ctx, execute_mechanical_fn=mock_execute)
        assert "ERROR" in result or result.get("ACTION") == "error"


class TestPostconditionNone:
    """Steps without postcondition_fn behave as before."""

    def test_no_postcondition_normal_flow(self, tmp_path):
        from orchestrator_engine import run_loop

        step = StepDef(
            name="test_step", phase="test", step_type="mechanical",
            postcondition_fn=None,
        )

        ctx = FakeEngineCtx(workspace=tmp_path, project=tmp_path)

        def mock_execute(step, ctx):
            return {}

        result = run_loop([step], ctx, execute_mechanical_fn=mock_execute)
        assert result["ACTION"] == "complete"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_orchestrator_engine_postconditions.py -v`
Expected: FAIL — `postcondition_fn` not a valid field for StepDef

- [ ] **Step 3: Add postcondition_fn to StepDef in shared_steps.py**

In `work-end/shared_steps.py`, add the field to the StepDef dataclass:

```python
@dataclass
class StepDef:
    name: str
    phase: str
    step_type: str
    script_fn: Callable | None = None
    skip_fn: Callable | None = None
    action_context_fn: Callable | None = None
    verify_fn: Callable | None = None
    postcondition_fn: Callable | None = None  # NEW
    from_state: str | None = None
    to_state: str | None = None
    event: str | None = None
```

In `work-end/work_end_orchestrator.py`, add the same field to the local StepDef:

```python
@dataclass
class StepDef:
    name: str
    phase: str
    step_type: str
    script_fn: Callable | None = None
    skip_fn: Callable | None = None
    action_context_fn: Callable | None = None
    verify_fn: Callable | None = None
    postcondition_fn: Callable | None = None  # NEW
    from_state: str | None = None
    to_state: str | None = None
    event: str | None = None
```

- [ ] **Step 4: Modify run_loop for check-execute-verify**

In `work-end/orchestrator_engine.py`, modify the mechanical step handling in `run_loop`:

```python
# Replace the mechanical step block (lines ~212-255) with:
        if step.step_type == "mechanical":
            if per_repo_mechanical:
                handled = per_repo_mechanical(step, ctx)
                if handled is not None:
                    if handled:
                        return handled
                    continue

            if ctx.done(step.name):
                continue

            # CHECK: postcondition already met? skip.
            if step.postcondition_fn and step.postcondition_fn(ctx):
                update_close_progress(ctx.workspace, step.name, "done")
                ctx.steps_executed.append(f"{step.name}:postcondition_skip")
                continue

            attempt_key = f"{step.name}_mechanical_attempt"
            attempt = int(ctx.progress.get(attempt_key, "0"))

            if execute_mechanical_fn:
                result = execute_mechanical_fn(step, ctx)
            else:
                result = _default_execute_mechanical(step, ctx)

            if result and "ERROR" in result:
                if on_mechanical_error:
                    override = on_mechanical_error(step, ctx, result)
                    if override is not None:
                        ctx.steps_executed.append(f"{step.name}:ERROR:classified")
                        return override
                attempt += 1
                update_close_progress(ctx.workspace, attempt_key, str(attempt))
                if attempt >= MAX_MECHANICAL_RETRIES:
                    update_close_progress(ctx.workspace, step.name, "skipped_error")
                    ctx.steps_executed.append(f"{step.name}:SKIPPED_ERROR")
                    continue
                ctx.steps_executed.append(f"{step.name}:ERROR:{attempt}")
                return _make_error_result(step.name, attempt, result)

            # VERIFY: postcondition met after execution?
            if step.postcondition_fn and not step.postcondition_fn(ctx):
                attempt += 1
                update_close_progress(ctx.workspace, attempt_key, str(attempt))
                if attempt >= MAX_MECHANICAL_RETRIES:
                    update_close_progress(ctx.workspace, step.name, "skipped_error")
                    ctx.steps_executed.append(f"{step.name}:POSTCONDITION_FAIL:skipped")
                    continue
                ctx.steps_executed.append(f"{step.name}:POSTCONDITION_FAIL:{attempt}")
                return {
                    "ACTION": "error",
                    "ERROR": "postcondition_failed",
                    "STEP": step.name,
                    "RETRY": str(attempt),
                    "REASON": f"Step '{step.name}' executed but postcondition not met",
                }

            ctx.last_output = result or {}
            if on_step_done:
                on_step_done(step, ctx, result or {})
            update_close_progress(ctx.workspace, step.name, "done")
            ctx.steps_executed.append(step.name)
            continue
```

- [ ] **Step 5: Run tests**

Run: `python3 -m pytest tests/test_orchestrator_engine_postconditions.py tests/test_work_end_orchestrator.py -v`
Expected: All tests PASS (new postcondition tests + existing orchestrator tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/shared_steps.py work-end/orchestrator_engine.py work-end/work_end_orchestrator.py tests/test_orchestrator_engine_postconditions.py
git commit -m "feat(#379): extend StepDef and run_loop with check-execute-verify  Refs #379"
```

### Task 3: Wire postcondition_fn to orchestrator STEPS

**Files:**
- Modify: `work-end/work_end_orchestrator.py` — add postcondition_fn to each mechanical StepDef in STEPS

**Interfaces:**
- Consumes: All postcondition functions from `verification/step_postconditions.py`
- Produces: Every mechanical StepDef in STEPS has a `postcondition_fn`

- [ ] **Step 1: Write failing test verifying postconditions are wired**

Add to `tests/test_work_end_orchestrator.py`:

```python
class TestPostconditionsWired:
    """Every mechanical step should have a postcondition_fn."""

    def test_all_mechanical_steps_have_postcondition(self):
        from work_end_orchestrator import STEPS
        mechanical = [s for s in STEPS if s.step_type == "mechanical"]
        # report_ steps and delete_progress are meta-steps, exempt
        exempt = {s.name for s in mechanical
                  if s.name.startswith("report_") or s.name == "delete_progress"}
        required = [s for s in mechanical if s.name not in exempt]
        missing = [s.name for s in required if s.postcondition_fn is None]
        assert missing == [], f"Mechanical steps missing postcondition_fn: {missing}"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestPostconditionsWired -v`
Expected: FAIL — lists all mechanical steps missing postcondition_fn

- [ ] **Step 3: Wire postconditions to STEPS**

In `work-end/work_end_orchestrator.py`, add the import at the top:

```python
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "verification"))
from step_postconditions import (
    rebase_postcondition,
    push_postcondition,
    stamp_postcondition,
    promote_postcondition,
    archive_move_postcondition,
    landed_marker_postcondition,
    checkout_main_postcondition,
    write_marker_postcondition,
    cleanup_scaffold_postcondition,
    issues_closed_postcondition,
)
```

Then update each mechanical StepDef in the STEPS list to add `postcondition_fn=`:

| Step name | postcondition_fn |
|-----------|-----------------|
| `promote` | `promote_postcondition` |
| `rebase` | `rebase_postcondition` |
| `write_marker` | `write_marker_postcondition` |
| `land` | `push_postcondition` |
| `write_landed` | `landed_marker_postcondition` |
| `close_issues` | `issues_closed_postcondition` |
| `archive_slot` | `archive_move_postcondition` |
| `checkout_main` | `checkout_main_postcondition` |
| `cleanup` | `cleanup_scaffold_postcondition` |
| `upstream_push` | `push_postcondition` |
| `cleanup_stack` | `None` (no meaningful postcondition — stack entry removal is trivial) |
| `elevate_plan` | `None` (inline execution, not script-based) |

For each, add the kwarg. Example for promote:

```python
    StepDef("promote", "closing:verified", "mechanical",
            script_fn=_promote_script,
            postcondition_fn=promote_postcondition),
```

- [ ] **Step 4: Run the wiring test + full test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v`
Expected: All PASS including `TestPostconditionsWired`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat(#379): wire postcondition checks to all mechanical steps  Refs #379"
```

---

## Batch 3: Honest Reporting

After this batch: all success output (prints, marker writes, DB transitions) happens
only after postcondition verification. The optimistic reporting root cause is eliminated.

### Task 4: Fix optimistic reporting in land_flow.py

**Files:**
- Modify: `work-end/land_flow.py` — move progress writes after verification
- Modify: `tests/test_work_end_execute.py` — add test for honest reporting

**Interfaces:**
- Consumes: Existing `_write_progress`, `_merge_and_push_two_hop`, `_merge_and_push_direct`
- Produces: Same functions but with progress writes AFTER verification, not before

- [ ] **Step 1: Write failing test**

Add to `tests/test_work_end_execute.py`:

```python
class TestHonestReporting:
    """Progress is only written after verification."""

    def test_progress_not_written_on_push_failure(self, tmp_path):
        progress_file = tmp_path / ".execute-progress"
        from land_flow import _write_progress, _read_progress

        # Simulate: merged but push failed
        _write_progress(progress_file, "repo:branch", "merged")

        # After a push failure, progress should NOT advance to "pushed"
        progress = _read_progress(progress_file)
        assert progress.get("repo:branch") == "merged"
        assert "pushed" not in str(progress)
```

- [ ] **Step 2: Review and fix land_flow.py**

In `_merge_and_push_two_hop` (line ~438-472): the current code writes
`_write_progress(progress_file, key, "merged")` immediately after merge
but before push verification. Move the "pushed" write to after the
verification check passes.

In `_merge_and_push_direct` (line ~476-540): same pattern — move
"pushed" write to after the push+verify sequence completes.

Key changes:
1. `_merge_and_push_two_hop`: Move `_write_progress(progress_file, key, "pushed")`
   to after `ls-remote` verification confirms SHA on remote
2. `_merge_and_push_direct`: Move `_write_progress(progress_file, key, "pushed")`
   to after `merge-base --is-ancestor` verification

- [ ] **Step 3: Run tests**

Run: `python3 -m pytest tests/test_work_end_execute.py -v`
Expected: All PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/land_flow.py tests/test_work_end_execute.py
git commit -m "fix(#379): move progress writes after verification in land_flow  Refs #379"
```

### Task 5: Fix optimistic reporting in work_end_execute.py and close_artifacts.py

**Files:**
- Modify: `work-end/work_end_execute.py` — verify before reporting in cmd_archive_slot, cmd_write_marker
- Modify: `work-end/close_artifacts.py` — verify stamp content after promotion

**Interfaces:**
- Consumes: Existing functions
- Produces: Same functions but with output after verification

- [ ] **Step 1: Write failing test for archive verification**

```python
# Add to tests/test_work_end_execute.py
class TestArchiveHonestReporting:
    def test_archive_reports_after_verification(self, tmp_path):
        """ARCHIVED= should only print after physical move is confirmed."""
        # This is tested via the postcondition — archive_move_postcondition
        # returns False if the slot is still in slots/
        from step_postconditions import archive_move_postcondition
        from test_step_postconditions import FakeCtx

        family = tmp_path / "family"
        slots = family / "slots" / "42"
        slots.mkdir(parents=True)
        ctx = FakeCtx(project=tmp_path, workspace=tmp_path,
                      slot_path=slots, family_root=family, slot_num="42")

        # Slot still in slots/ — postcondition should fail
        assert archive_move_postcondition(ctx) is False
```

- [ ] **Step 2: Fix cmd_write_marker verification**

In `work_end_execute.py` `cmd_write_marker`: add verification that the marker
file was actually written before printing `MARKER_WRITTEN`:

```python
def cmd_write_marker(opts: dict[str, str]) -> int:
    # ... existing validation ...
    marker = slot_dir / ".phase-a-complete"
    marker.write_text(
        f"branch={branch}\n"
        f"timestamp={datetime.datetime.now(datetime.timezone.utc).isoformat()}\n"
    )
    if not marker.exists():
        print("ERROR=MARKER_WRITE_FAILED")
        return 1
    print(f"MARKER_WRITTEN={marker}")
    return 0
```

- [ ] **Step 3: Fix close_artifacts.py stamp verification**

In `close_artifacts.py` `write_stamp`: verify the stamp was written:

```python
def write_stamp(workspace: Path, branch: str, results: dict[str, str]) -> Path:
    stamp_path = workspace / ".artifacts-promoted"
    # ... existing write logic ...
    stamp_path.write_text("\n".join(lines) + "\n")
    if not stamp_path.exists():
        raise RuntimeError(f"Failed to write stamp to {stamp_path}")
    # ... existing git add/commit ...
    return stamp_path
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_work_end_execute.py tests/test_step_postconditions.py -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_execute.py work-end/close_artifacts.py tests/test_work_end_execute.py
git commit -m "fix(#379): verify operations before reporting success  Refs #379"
```

---

## Batch 4: Convergence Testing

After this batch: integration-style tests prove that re-running the pipeline after
any mid-step failure converges to the correct end state.

### Task 6: Write convergence tests

**Files:**
- Create: `tests/test_convergence.py`

**Interfaces:**
- Consumes: `run_orchestrator` from `work_end_orchestrator.py`, all postcondition functions
- Produces: Test suite proving idempotent convergence

- [ ] **Step 1: Write convergence test scaffold with helper**

```python
# tests/test_convergence.py
"""Convergence tests — verify re-running work-end after failures converges."""

import sys
from dataclasses import dataclass, field
from pathlib import Path
from unittest.mock import patch, MagicMock, call

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
sys.path.insert(0, str(Path(__file__).parent.parent / "verification"))

from close_progress import write_close_progress, read_close_progress


def _make_args(tmp_path, **overrides):
    """Build orchestrator args dict with tmp_path workspace."""
    base = {
        "workspace": str(tmp_path),
        "project": str(tmp_path / "project"),
        "branch": "issue-379-test",
        "base_branch": "main",
        "meta_state": "closing:promoted",
        "on_main": "no",
        "in_slot": "no",
        "covers": "379",
        "issue_repo": "Hortora/soredium",
    }
    base.update(overrides)
    return base


def _prefill_progress(tmp_path, **entries):
    """Write .close-progress with given entries + branch marker."""
    data = {"_branch": "issue-379-test"}
    data.update(entries)
    write_close_progress(tmp_path, data)
```

- [ ] **Step 2: Write test — full pipeline re-run is no-op**

```python
class TestIdempotentRerun:
    def test_all_steps_done_returns_complete(self, tmp_path, monkeypatch):
        """When all postconditions are met, pipeline skips everything."""
        monkeypatch.setattr("work_end_orchestrator._run_script",
                            lambda cmd, ws, **kw: {})

        # Pre-fill all steps as done
        done_steps = [
            "report_init", "code_review", "branch_audit_conformance",
            "branch_audit_coherence", "branch_audit_structure",
            "branch_audit_robustness", "loose_ends", "forcing_function",
            "sweep_config", "review_pass",
            "promote", "report_promote", "promote_pass",
            "trajectory", "rebase", "report_rebase",
            "squash", "report_squash", "land", "report_land",
            "push_pass", "merge_pass", "stamp_pass",
            "close_issues", "report_close_issues",
            "verify", "report_verify",
            "checkout_main", "cleanup_stack", "cleanup",
            "report_scaffold", "arc42_scan", "session_rename",
            "garden_feedback", "notes", "cleanup_pass",
            "delete_progress", "report_render",
        ]
        entries = {s: "done" for s in done_steps}
        _prefill_progress(tmp_path, **entries)

        from work_end_orchestrator import run_orchestrator
        result = run_orchestrator(_make_args(tmp_path))
        assert result["ACTION"] == "complete"
```

- [ ] **Step 3: Write test — resume after push failure**

```python
    def test_resume_after_push_failure(self, tmp_path, monkeypatch):
        """After push fails, re-run skips rebase (postcondition met) and retries push."""
        call_log = []

        def mock_run_script(cmd, ws, **kw):
            cmd_str = " ".join(str(c) for c in cmd)
            call_log.append(cmd_str)
            if "land" in cmd_str:
                return {"LANDED": "yes", "LANDED_SHA": "abc123"}
            return {}

        monkeypatch.setattr("work_end_orchestrator._run_script", mock_run_script)

        # Pre-fill: everything up to and including rebase is done
        _prefill_progress(tmp_path,
            report_init="done",
            code_review="done", branch_audit_conformance="done",
            branch_audit_coherence="done", branch_audit_structure="done",
            branch_audit_robustness="done", loose_ends="done",
            forcing_function="done", sweep_config="done",
            review_pass="done",
            promote="done", report_promote="done", promote_pass="done",
            trajectory="done",
            rebase="done", report_rebase="done",
            squash="done", report_squash="done",
        )

        from work_end_orchestrator import run_orchestrator

        # Mock all postconditions to return True (rebase already done)
        with patch("work_end_orchestrator.rebase_postcondition", return_value=True):
            result = run_orchestrator(_make_args(tmp_path))

        # Should execute land, not re-run rebase
        rebase_calls = [c for c in call_log if "rebase" in c]
        assert len(rebase_calls) == 0, "Rebase should not be re-run"
```

- [ ] **Step 4: Write test — postcondition skip on re-entry**

```python
    def test_postcondition_skip_on_reentry(self, tmp_path, monkeypatch):
        """Promote postcondition met on re-entry -> step skipped."""
        monkeypatch.setattr("work_end_orchestrator._run_script",
                            lambda cmd, ws, **kw: {})

        # Pre-fill through review
        _prefill_progress(tmp_path,
            report_init="done",
            code_review="done", branch_audit_conformance="done",
            branch_audit_coherence="done", branch_audit_structure="done",
            branch_audit_robustness="done", loose_ends="done",
            forcing_function="done", sweep_config="done",
            review_pass="done",
        )

        # Simulate: .artifacts-promoted already exists (promotion done externally)
        (tmp_path / ".artifacts-promoted").write_text("timestamp=2026-01-01\n")

        from work_end_orchestrator import run_orchestrator
        result = run_orchestrator(_make_args(tmp_path))

        # Promote step should have been skipped via postcondition
        progress = read_close_progress(tmp_path)
        assert progress.get("promote") == "done"
```

- [ ] **Step 5: Run convergence tests**

Run: `python3 -m pytest tests/test_convergence.py -v`
Expected: All PASS

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: All tests PASS, no regressions

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/test_convergence.py
git commit -m "test(#379): convergence tests for idempotent pipeline re-runs  Refs #379"
```

---

## References

- `specs/issue-379-idempotent-work-end/2026-09-25-idempotent-work-end-design.md` — design spec
- `work-end/orchestrator_engine.py` — run_loop implementation (lines 171-299)
- `work-end/work_end_orchestrator.py` — STEPS list (lines 865-1012), OrchestratorContext
- `work-end/shared_steps.py` — StepDef dataclass (lines 18-28)
- `work-end/land_flow.py` — land_batch flow with partial idempotency
- `work-end/close_artifacts.py` — artifact promotion and stamp writing
- `work-end/verify_slot_close.py` — existing post-close verification
- `verification/__init__.py` — Finding, git() helper
- `verification/postconditions.py` — existing lifecycle gate checks
- `tests/test_work_end_orchestrator.py` — existing test patterns (monkeypatch _run_script)
- `docs/protocols/evidence-before-claims.md` — "run the command, read the output, THEN claim"
- `docs/protocols/externalised-scripts-require-tests.md` — scripts ship with tests
- GitHub #379 — issue with detailed failure analysis

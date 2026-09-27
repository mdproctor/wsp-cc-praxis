# Continuation Pattern Implementation Plan (Phase 3 of #382)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #382 — Unified mechanical pipeline
**Issue group:** #382

**Goal:** Add mechanical promotion of `.plan-next` → `.plan` and `HANDOFF-next.md` → `HANDOFF.md` as a pipeline step, so continuation artifacts built before ceremony survive and get promoted atomically after sync or end completes.

**Architecture:** Two pieces: (1) `plan_manager.py build-next` creates `.plan-next` from the current `.plan` by extracting uncompleted issues into a fresh queue. (2) A new `promote_plan_next` mechanical step in the orchestrator runs after `sync_pass` (sync mode) or before `cleanup` (end mode). It copies `-next` files over active files, writes `state: active`, commits, and deletes the `-next` originals. A postcondition verifies `.plan-next` no longer exists after promotion.

**Tech Stack:** Python 3, pytest, shutil, subprocess (git CLI), pathlib

## Global Constraints

- Promotion must be a mechanical step with a postcondition — not LLM-driven
- `.plan-next` and `HANDOFF-next.md` are committed to the branch before ceremony starts (by the LLM or `build-next`)
- Promotion commits to git (same pattern as `commit_transition`)
- If no `-next` files exist, the step is a no-op (not an error)
- End mode: promotion runs before cleanup (cleanup removes `.plan` if no remaining items)
- Sync mode: promotion runs after `sync_pass` (state is already `active`)
- Existing `cleanup_scaffold` behavior must not change when no `.plan-next` exists

---

## Batch 1: build-next command and promote step

### Task 1: Add build-next command to plan_manager.py

**Files:**
- Modify: `work-slot/plan_manager.py:1401+` — add `build-next` command handler
- Create: `tests/test_build_next.py` — test build-next produces correct `.plan-next`

**Interfaces:**
- Consumes: `parse_plan()` from plan_manager.py, `PlanState` / `QueueItem` from `plan_io.py`
- Produces: `build-next` command that creates `.plan-next` from uncompleted items in `.plan`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_build_next.py
"""Tests for plan_manager build-next command."""

import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


class TestBuildNext:
    def _run(self, plan_path: Path) -> subprocess.CompletedProcess:
        return subprocess.run(
            [sys.executable,
             str(Path(__file__).parent.parent / "work-slot" / "plan_manager.py"),
             "build-next", str(plan_path)],
            capture_output=True, text=True,
        )

    def test_creates_plan_next(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: test-branch\nstate: active\ncovers: 1,2,3\n\n"
            "## Queue\n"
            "- [x] repo#1 — First issue\n"
            "- [ ] repo#2 — Second issue ← active\n"
            "- [ ] repo#3 — Third issue\n"
        )
        result = self._run(plan)
        assert result.returncode == 0

        plan_next = tmp_path / ".plan-next"
        assert plan_next.exists()
        content = plan_next.read_text()
        assert "repo#2" in content
        assert "repo#3" in content
        assert "repo#1" not in content

    def test_plan_next_has_active_state(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: test-branch\nstate: active\ncovers: 1,2\n\n"
            "## Queue\n"
            "- [x] repo#1 — Done\n"
            "- [ ] repo#2 — Remaining ← active\n"
        )
        self._run(plan)
        content = (tmp_path / ".plan-next").read_text()
        assert "state: active" in content

    def test_plan_next_preserves_branch(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: issue-382\nstate: active\ncovers: 1,2\n\n"
            "## Queue\n"
            "- [x] repo#1 — Done\n"
            "- [ ] repo#2 — Remaining ← active\n"
        )
        self._run(plan)
        content = (tmp_path / ".plan-next").read_text()
        assert "branch: issue-382" in content

    def test_no_plan_next_when_all_completed(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: test-branch\nstate: active\ncovers: 1\n\n"
            "## Queue\n"
            "- [x] repo#1 — All done\n"
        )
        result = self._run(plan)
        assert result.returncode == 0
        assert not (tmp_path / ".plan-next").exists()
        assert "NO_REMAINING=yes" in result.stdout

    def test_no_plan_next_when_plan_missing(self, tmp_path):
        result = self._run(tmp_path / ".plan")
        assert result.returncode != 0

    def test_first_uncompleted_is_active(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: test-branch\nstate: active\ncovers: 1,2,3\n\n"
            "## Queue\n"
            "- [x] repo#1 — Done\n"
            "- [ ] repo#2 — Second\n"
            "- [ ] repo#3 — Third\n"
        )
        self._run(plan)
        content = (tmp_path / ".plan-next").read_text()
        assert "← active" in content
        lines = [l for l in content.splitlines() if "← active" in l]
        assert len(lines) == 1
        assert "repo#2" in lines[0]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_build_next.py -v --tb=short`
Expected: FAIL — `build-next` command not recognized

- [ ] **Step 3: Implement build-next command**

In `work-slot/plan_manager.py`, add a new command handler after the `uncheck` handler:

```python
    elif command == "build-next":
        if not plan_path or not plan_path.exists():
            print("ERROR=plan_not_found")
            return 1
        tree = parse_plan(plan_path)
        uncompleted = [item for item in tree.queue if not _is_checked(item)]
        if not uncompleted:
            print("NO_REMAINING=yes")
            return 0
        # Build .plan-next with uncompleted items
        next_path = plan_path.parent / ".plan-next"
        branch = tree.state.get("branch", "")
        covers_nums = [_extract_issue_number(item) for item in uncompleted]
        covers_str = ",".join(str(n) for n in covers_nums if n)
        lines = ["# Work Plan\n", "", "## State"]
        if branch:
            lines.append(f"branch: {branch}")
        lines.append("state: active")
        if covers_str:
            lines.append(f"covers: {covers_str}")
        lines.append("")
        lines.append("## Queue")
        first = True
        for item in uncompleted:
            text = item.rstrip()
            # Ensure first uncompleted item is active
            if first:
                if "← active" not in text:
                    text = text + " ← active"
                first = False
            else:
                text = text.replace(" ← active", "")
            lines.append(text)
        next_path.write_text("\n".join(lines) + "\n")
        print(f"BUILT={next_path}")
        print(f"ITEMS={len(uncompleted)}")
        return 0
```

Helper functions needed (add near the top of the file if not already present):

```python
def _is_checked(line: str) -> bool:
    stripped = line.strip()
    return stripped.startswith("- [x]") or stripped.startswith("- [X]")

def _extract_issue_number(line: str) -> int | None:
    import re
    m = re.search(r'#(\d+)', line)
    return int(m.group(1)) if m else None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_build_next.py -v --tb=short`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_build_next.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add build-next command to plan_manager

Refs #382"
```

### Task 2: Add promote_plan_next step to orchestrator

**Files:**
- Modify: `work-end/work_end_orchestrator.py:475-528` — add `_promote_plan_next_inline` function
- Modify: `work-end/work_end_orchestrator.py:1049-1064` — add `promote_plan_next` step to STEPS
- Modify: `work-end/close_progress.py:58+` — add `promote_plan_next` to STEP_TO_PHASE
- Modify: `verification/step_postconditions.py` — add `promote_plan_next_postcondition`
- Create: `tests/test_promote_plan_next.py` — test promotion logic

**Interfaces:**
- Consumes: `OrchestratorContext`, `_skip_sync_mode`, `_skip_not_sync_mode`, `sync_pass` step, `cleanup` step
- Produces: `_promote_plan_next_inline(ctx)` function, `promote_plan_next` StepDef, `promote_plan_next_postcondition`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_promote_plan_next.py
"""Tests for promote_plan_next orchestrator step."""

import shutil
import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from work_end_orchestrator import (
    OrchestratorContext,
    STEPS,
    _promote_plan_next_inline,
)


def _init_git_repo(path: Path) -> None:
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "test@test.com"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
    (path / ".gitkeep").write_text("")
    subprocess.run(["git", "-C", str(path), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "initial"], capture_output=True)


def _make_ctx(workspace: Path, **overrides) -> OrchestratorContext:
    defaults = dict(
        workspace=workspace,
        project=workspace,
        branch="test-branch",
        base_branch="main",
        meta_state="active",
        on_main=False,
        in_slot=False,
        covers="1",
        issue_repo="test/repo",
        progress={},
        mode="sync",
    )
    defaults.update(overrides)
    return OrchestratorContext(**defaults)


class TestPromotePlanNextInline:
    def test_promotes_plan_next_to_plan(self, tmp_path):
        _init_git_repo(tmp_path)
        plan = tmp_path / ".plan"
        plan.write_text("# Old Plan\n\n## State\nstate: active\n\n## Queue\n- [x] done#1 — Done\n")
        plan_next = tmp_path / ".plan-next"
        plan_next.write_text("# Work Plan\n\n## State\nstate: active\n\n## Queue\n- [ ] repo#2 — Next\n")
        subprocess.run(["git", "-C", str(tmp_path), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(tmp_path), "commit", "-m", "scaffold"], capture_output=True)

        ctx = _make_ctx(tmp_path)
        result = _promote_plan_next_inline(ctx)

        assert result["PROMOTED"] == "yes"
        assert plan.exists()
        assert not plan_next.exists()
        assert "repo#2" in plan.read_text()

    def test_promotes_handoff_next(self, tmp_path):
        _init_git_repo(tmp_path)
        handoff = tmp_path / "HANDOFF.md"
        handoff.write_text("# Old Handoff\n")
        handoff_next = tmp_path / "HANDOFF-next.md"
        handoff_next.write_text("# New Handoff\n")
        subprocess.run(["git", "-C", str(tmp_path), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(tmp_path), "commit", "-m", "scaffold"], capture_output=True)

        ctx = _make_ctx(tmp_path)
        result = _promote_plan_next_inline(ctx)

        assert handoff.exists()
        assert not handoff_next.exists()
        assert "New Handoff" in handoff.read_text()

    def test_noop_when_no_next_files(self, tmp_path):
        _init_git_repo(tmp_path)
        ctx = _make_ctx(tmp_path)
        result = _promote_plan_next_inline(ctx)

        assert result["PROMOTED"] == "no"
        assert result["REASON"] == "no_next_files"

    def test_commits_to_git(self, tmp_path):
        _init_git_repo(tmp_path)
        plan_next = tmp_path / ".plan-next"
        plan_next.write_text("# Work Plan\n\n## State\nstate: active\n\n## Queue\n- [ ] repo#1 — Work\n")
        subprocess.run(["git", "-C", str(tmp_path), "add", ".plan-next"], capture_output=True)
        subprocess.run(["git", "-C", str(tmp_path), "commit", "-m", "add next"], capture_output=True)

        ctx = _make_ctx(tmp_path)
        _promote_plan_next_inline(ctx)

        log = subprocess.run(
            ["git", "-C", str(tmp_path), "log", "--oneline", "-1"],
            capture_output=True, text=True,
        )
        assert "promote" in log.stdout.lower()

    def test_dry_run_does_not_modify(self, tmp_path):
        _init_git_repo(tmp_path)
        plan_next = tmp_path / ".plan-next"
        plan_next.write_text("# Next\n")
        subprocess.run(["git", "-C", str(tmp_path), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(tmp_path), "commit", "-m", "add"], capture_output=True)

        ctx = _make_ctx(tmp_path, dry_run=True)
        result = _promote_plan_next_inline(ctx)

        assert plan_next.exists()
        assert result["PROMOTED"] == "yes"


class TestPromotePlanNextStep:
    def test_step_exists_in_steps(self):
        names = [s.name for s in STEPS]
        assert "promote_plan_next" in names

    def test_step_is_mechanical(self):
        step = next(s for s in STEPS if s.name == "promote_plan_next")
        assert step.step_type == "mechanical"

    def test_step_runs_in_both_modes(self):
        """promote_plan_next runs in both sync and end mode."""
        step = next(s for s in STEPS if s.name == "promote_plan_next")
        # Should not have a mode-based skip — runs in both
        assert step.skip_fn is None or step.skip_fn.__name__ not in (
            "_skip_sync_mode", "_skip_not_sync_mode"
        )
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_promote_plan_next.py -v --tb=short`
Expected: FAIL — `_promote_plan_next_inline` doesn't exist

- [ ] **Step 3: Implement _promote_plan_next_inline**

In `work-end/work_end_orchestrator.py`, add after `_elevate_plan_inline` (around line 528):

```python
def _promote_plan_next_inline(ctx: OrchestratorContext) -> dict[str, str]:
    """Promote .plan-next → .plan and HANDOFF-next.md → HANDOFF.md.

    Runs after sync_pass (sync mode) or before cleanup (end mode).
    Mechanical step — no LLM judgment needed.
    """
    plan_next = ctx.workspace / ".plan-next"
    handoff_next = ctx.workspace / "HANDOFF-next.md"

    if not plan_next.exists() and not handoff_next.exists():
        return {"PROMOTED": "no", "REASON": "no_next_files"}

    if ctx.dry_run:
        if ctx.call_log is not None:
            if plan_next.exists():
                ctx.call_log.append(["(internal)", "promote_plan_next", str(plan_next)])
            if handoff_next.exists():
                ctx.call_log.append(["(internal)", "promote_handoff_next", str(handoff_next)])
        return {"PROMOTED": "yes", "DRY_RUN": "yes"}

    promoted = []

    if plan_next.exists():
        import shutil as _shutil
        _shutil.copy2(plan_next, ctx.workspace / ".plan")
        plan_next.unlink()
        subprocess.run(
            ["git", "-C", str(ctx.workspace), "add", ".plan"],
            capture_output=True, timeout=10,
        )
        subprocess.run(
            ["git", "-C", str(ctx.workspace), "rm", "--ignore-unmatch", "-f", ".plan-next"],
            capture_output=True, timeout=10,
        )
        promoted.append(".plan-next")

    if handoff_next.exists():
        import shutil as _shutil
        _shutil.copy2(handoff_next, ctx.workspace / "HANDOFF.md")
        handoff_next.unlink()
        subprocess.run(
            ["git", "-C", str(ctx.workspace), "add", "HANDOFF.md"],
            capture_output=True, timeout=10,
        )
        subprocess.run(
            ["git", "-C", str(ctx.workspace), "rm", "--ignore-unmatch", "-f", "HANDOFF-next.md"],
            capture_output=True, timeout=10,
        )
        promoted.append("HANDOFF-next.md")

    if promoted:
        subprocess.run(
            ["git", "-C", str(ctx.workspace), "commit", "--no-verify", "-m",
             f"chore: promote {', '.join(promoted)}"],
            capture_output=True, timeout=10,
        )

    return {"PROMOTED": "yes", "FILES": ",".join(promoted)}
```

- [ ] **Step 4: Add promote_plan_next step to STEPS**

In `work-end/work_end_orchestrator.py`, add after `sync_pass` and before `cleanup_pass`:

```python
    # --- continuation promotion (runs in both sync and end modes) ---
    StepDef("promote_plan_next", "closing:stamped", "mechanical",
            script_fn=lambda ctx: None),
```

The `script_fn` returns None because the step uses inline execution via
`_promote_plan_next_inline`. The engine's `_next_action` function needs to
call inline for this step. However, looking at how `elevate_plan` works
(it also has `script_fn=_elevate_plan_script` which returns None, and the
engine calls `_elevate_plan_inline` directly), the same pattern applies.

Actually — the engine calls script_fn, and if it returns None, the step
auto-completes. For inline execution, we need to wire it through the
engine's inline step handling. Let me check how elevate_plan is wired:

The `elevate_plan` step has `script_fn=_elevate_plan_script` which returns
None, so the engine auto-completes it. But `_elevate_plan_inline` is called
separately in `_next_action`. Let me follow that pattern.

Add to `_next_action` (or the inline step handler) — check the orchestrator
engine for how inline steps work. The simplest approach: give
`promote_plan_next` a `script_fn` that calls the inline function directly.

```python
def _promote_plan_next_script(ctx):
    result = _promote_plan_next_inline(ctx)
    ctx.last_output.update(result)
    return None
```

Update the step definition:
```python
    StepDef("promote_plan_next", "closing:stamped", "mechanical",
            script_fn=_promote_plan_next_script),
```

- [ ] **Step 5: Add postcondition**

In `verification/step_postconditions.py`, add:

```python
def promote_plan_next_postcondition(workspace: Path, ctx=None) -> str | None:
    plan_next = workspace / ".plan-next"
    if plan_next.exists():
        return ".plan-next still exists after promotion — retry"
    return None
```

Wire it in the step definition:
```python
    StepDef("promote_plan_next", "closing:stamped", "mechanical",
            script_fn=_promote_plan_next_script,
            postcondition_fn=promote_plan_next_postcondition),
```

Import in `work_end_orchestrator.py`:
```python
from step_postconditions import (
    ...,
    promote_plan_next_postcondition,
)
```

- [ ] **Step 6: Add to STEP_TO_PHASE**

In `work-end/close_progress.py`, add to `STEP_TO_PHASE`:

```python
    "promote_plan_next": "closing:stamped",
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_promote_plan_next.py tests/test_sync_orchestrator.py tests/test_work_end_orchestrator.py -v --tb=short`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py work-end/close_progress.py verification/step_postconditions.py tests/test_promote_plan_next.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add promote_plan_next pipeline step

Refs #382"
```

---

## References

- `specs/issue-382-unified-pipeline/2026-09-27-unified-pipeline-design.md` — design spec (Section C: Continuation Pattern)
- `specs/issue-382-unified-pipeline/decisions.md` — D5 (.plan-next / HANDOFF-next pattern)
- `work-slot/plan_manager.py:1401+` — existing command handlers
- `work-end/work_end_orchestrator.py:475-528` — elevate_plan_inline (pattern to follow)
- `work-end/work_end_orchestrator.py:1049-1064` — STEPS terminal section
- `work-end/close_progress.py:58-62` — STEP_TO_PHASE closing:stamped entries
- `work-end/branch_cleanup.py:81-120` — cleanup_scaffold (preserve_plan logic)
- `verification/step_postconditions.py` — existing postcondition functions
- `project/plan_io.py:40-86` — read_plan, QueueItem, PlanState
- GitHub #382 — unified mechanical pipeline

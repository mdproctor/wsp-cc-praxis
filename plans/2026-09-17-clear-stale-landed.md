# Queue-Aware Work-End Cycle Mode — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #373 — Clear stale .landed marker when slot is re-activated for new work

**Goal:** Make work-end queue-aware so it cycles (syncs without landing) when the slot has remaining work, and only terminates when the queue is empty.

**Architecture:** Add a `_skip_cycle_mode` predicate to the orchestrator that checks `.plan` for uncompleted items. Terminal steps (`.landed`, stamp, archive, checkout, cleanup) use this predicate to auto-skip. A new `cycle_pass` lifecycle step transitions `closing:merged → active` when cycling. Belt-and-suspenders: `append_to_queue()` also clears stale `.landed` retroactively.

**Tech Stack:** Python 3, pytest (tmp_path fixtures), existing plan_io/lifecycle/slot_state modules

## Global Constraints

- All new functions require tests committed together (protocol: externalised-scripts-require-tests)
- No new dependencies — only import existing project modules
- `plan_manager.py` currently has zero imports from `slot_state` or `slot_metadata` — the retroactive cleanup (Task 5) adds a minimal import

---

## Batch 1: Foundation — skip predicates, lifecycle transition, slot state

### Task 1: Add skip predicates and combiner to orchestrator

**Files:**
- Modify: `work-end/work_end_orchestrator.py:217-254` (add after existing skip predicates)
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes: `plan_io.read_plan(path) -> PlanState | None`, `plan_io.has_uncompleted_items(state) -> bool` (existing, used at orchestrator line 460)
- Produces: `_or_skip(*fns) -> Callable`, `_skip_cycle_mode(ctx) -> bool`, `_skip_not_cycle_mode(ctx) -> bool` — used by Task 4

- [ ] **Step 1: Write failing tests**

Add to `tests/test_work_end_orchestrator.py`:

```python
import pytest
from pathlib import Path
from unittest.mock import MagicMock


class TestSkipCycleMode:
    """Tests for _skip_cycle_mode and _skip_not_cycle_mode."""

    def _make_ctx(self, tmp_path, plan_content=None):
        ctx = MagicMock()
        ctx.workspace = str(tmp_path)
        if plan_content is not None:
            (tmp_path / ".plan").write_text(plan_content)
        return ctx

    def test_skip_cycle_mode_with_uncompleted_items(self, tmp_path):
        from work_end_orchestrator import _skip_cycle_mode
        plan = "## Queue\n- [ ] item1 ← active\n- [ ] item2\n"
        ctx = self._make_ctx(tmp_path, plan)
        assert _skip_cycle_mode(ctx) is True

    def test_skip_cycle_mode_with_all_completed(self, tmp_path):
        from work_end_orchestrator import _skip_cycle_mode
        plan = "## Queue\n- [x] item1\n"
        ctx = self._make_ctx(tmp_path, plan)
        assert _skip_cycle_mode(ctx) is False

    def test_skip_cycle_mode_with_no_plan(self, tmp_path):
        from work_end_orchestrator import _skip_cycle_mode
        ctx = self._make_ctx(tmp_path, plan_content=None)
        assert _skip_cycle_mode(ctx) is False

    def test_skip_not_cycle_mode_inverse(self, tmp_path):
        from work_end_orchestrator import _skip_not_cycle_mode
        plan = "## Queue\n- [ ] item1 ← active\n- [ ] item2\n"
        ctx = self._make_ctx(tmp_path, plan)
        assert _skip_not_cycle_mode(ctx) is False


class TestOrSkip:
    """Tests for _or_skip combiner."""

    def test_any_true_returns_true(self):
        from work_end_orchestrator import _or_skip
        always_true = lambda ctx: True
        always_false = lambda ctx: False
        combined = _or_skip(always_false, always_true)
        assert combined(None) is True

    def test_all_false_returns_false(self):
        from work_end_orchestrator import _or_skip
        always_false = lambda ctx: False
        combined = _or_skip(always_false, always_false)
        assert combined(None) is False

    def test_single_predicate(self):
        from work_end_orchestrator import _or_skip
        always_true = lambda ctx: True
        combined = _or_skip(always_true)
        assert combined(None) is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestSkipCycleMode -v --no-header 2>&1 | tail -20`
Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestOrSkip -v --no-header 2>&1 | tail -20`
Expected: ImportError — `_skip_cycle_mode`, `_or_skip` not defined

- [ ] **Step 3: Implement skip predicates**

Add after the existing skip predicates (after line 254 in `work-end/work_end_orchestrator.py`):

```python
def _or_skip(*fns):
    """Skip if ANY predicate returns True."""
    def combined(ctx):
        return any(fn(ctx) for fn in fns)
    return combined


def _skip_cycle_mode(ctx) -> bool:
    """Skip terminal steps when .plan has remaining items (cycle mode)."""
    from plan_io import read_plan, has_uncompleted_items
    ws_plan = Path(ctx.workspace) / ".plan"
    state = read_plan(ws_plan)
    return state is not None and has_uncompleted_items(state)


def _skip_not_cycle_mode(ctx) -> bool:
    """Skip cycle step when in terminal mode."""
    return not _skip_cycle_mode(ctx)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestSkipCycleMode tests/test_work_end_orchestrator.py::TestOrSkip -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat(#373): add _or_skip combiner and cycle mode skip predicates

Refs #373"
```

---

### Task 2: Add `issue_cycle` lifecycle transition

**Files:**
- Modify: `project/lifecycle.py:109-132` (add transition to TRANSITION_TABLE)
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes: nothing new — extends existing TRANSITION_TABLE
- Produces: `('closing:merged', 'issue_cycle') → ('active', ...)` transition — used by Task 4's `cycle_pass` step

- [ ] **Step 1: Write failing test**

Add to `tests/test_lifecycle.py`:

```python
class TestIssueCycleTransition:
    """Tests for the issue_cycle lifecycle event."""

    def test_issue_cycle_from_closing_merged(self, tmp_path):
        from lifecycle import transition
        plan = tmp_path / ".plan"
        _write_plan(plan, "closing:merged")
        result = transition(plan, "issue_cycle")
        assert result.new_state == "active"
        assert "advance_issue" in result.effects
        assert "clear_closing_markers" in result.effects

    def test_issue_cycle_from_wrong_state_fails(self, tmp_path):
        from lifecycle import transition, InvalidTransition
        plan = tmp_path / ".plan"
        _write_plan(plan, "active")
        with pytest.raises(InvalidTransition):
            transition(plan, "issue_cycle")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle.py::TestIssueCycleTransition -v --no-header 2>&1 | tail -20`
Expected: FAIL — InvalidTransition for `closing:merged` + `issue_cycle` (not in table)

- [ ] **Step 3: Add transition to TRANSITION_TABLE**

In `project/lifecycle.py`, add to `TRANSITION_TABLE` (after the `stamp_pass` entry at line ~125):

```python
    ('closing:merged', 'issue_cycle'):   ('active',           ['advance_issue', 'clear_closing_markers'], []),
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_lifecycle.py::TestIssueCycleTransition -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 5: Run full lifecycle test suite for regressions**

Run: `python3 -m pytest tests/test_lifecycle.py -v --no-header 2>&1 | tail -30`
Expected: all PASSED

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_lifecycle.py
git commit -m "feat(#373): add issue_cycle lifecycle transition (closing:merged → active)

Refs #373"
```

---

### Task 3: Add `landed → active` slot state transition

**Files:**
- Modify: `work-slot/slot_state.py:47-55` (add `"active"` to landed's transitions)
- Test: `tests/test_slot_state.py`

**Interfaces:**
- Consumes: nothing new — extends existing VALID_TRANSITIONS
- Produces: `landed → active` validation — used by Task 5's retroactive cleanup

- [ ] **Step 1: Write failing test**

Add to `tests/test_slot_state.py`:

```python
class TestLandedToActiveTransition:
    """Tests for landed → active slot state transition."""

    def test_landed_to_active_valid(self, tmp_path):
        from slot_state import transition
        slot_dir = tmp_path / "slot-1"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text("state: landed\n")
        transition(slot_dir, "active")
        content = (slot_dir / ".slot").read_text()
        assert "state: active" in content

    def test_landed_to_archived_still_valid(self, tmp_path):
        from slot_state import transition
        slot_dir = tmp_path / "slot-1"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text("state: landed\n")
        transition(slot_dir, "archived")
        content = (slot_dir / ".slot").read_text()
        assert "state: archived" in content
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_state.py::TestLandedToActiveTransition::test_landed_to_active_valid -v --no-header 2>&1 | tail -20`
Expected: FAIL — InvalidTransition (landed → active not in VALID_TRANSITIONS)

- [ ] **Step 3: Add `active` to landed's transitions**

In `work-slot/slot_state.py`, change line ~51:

```python
    "landed":    frozenset({"archived", "active"}),
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_state.py::TestLandedToActiveTransition -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 5: Run full slot state test suite for regressions**

Run: `python3 -m pytest tests/test_slot_state.py -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_state.py tests/test_slot_state.py
git commit -m "feat(#373): allow landed → active slot state transition

Refs #373"
```

---

## Batch 2: Wiring — orchestrator step changes and retroactive cleanup

### Task 4: Add `cycle_pass` step and update terminal step skip predicates

**Files:**
- Modify: `work-end/work_end_orchestrator.py:839-980` (STEPS list — insert step, update skip_fns)
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes: `_or_skip`, `_skip_cycle_mode`, `_skip_not_cycle_mode` (from Task 1); `issue_cycle` transition (from Task 2)
- Produces: orchestrator cycle behavior — no downstream consumers

- [ ] **Step 1: Write failing tests**

Add to `tests/test_work_end_orchestrator.py`:

```python
class TestCyclePassStep:
    """Tests for the cycle_pass step in STEPS list."""

    def test_cycle_pass_exists_in_steps(self):
        from work_end_orchestrator import STEPS
        names = [s.name for s in STEPS]
        assert "cycle_pass" in names

    def test_cycle_pass_before_stamp_pass(self):
        from work_end_orchestrator import STEPS
        names = [s.name for s in STEPS]
        cycle_idx = names.index("cycle_pass")
        stamp_idx = names.index("stamp_pass")
        assert cycle_idx < stamp_idx

    def test_cycle_pass_properties(self):
        from work_end_orchestrator import STEPS
        step = next(s for s in STEPS if s.name == "cycle_pass")
        assert step.phase == "closing:merged"
        assert step.step_type == "lifecycle"
        assert step.from_state == "closing:merged"
        assert step.to_state == "active"
        assert step.event == "issue_cycle"

    def test_terminal_steps_have_cycle_skip(self):
        from work_end_orchestrator import STEPS
        terminal_steps = {"write_marker", "stamp_pass", "write_landed",
                          "archive_slot", "report_archive", "checkout_main",
                          "cleanup_stack", "cleanup", "cleanup_pass",
                          "cleanup_main"}
        for step in STEPS:
            if step.name in terminal_steps:
                assert step.skip_fn is not None, (
                    f"Step {step.name} must have a skip_fn for cycle mode"
                )
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestCyclePassStep -v --no-header 2>&1 | tail -20`
Expected: FAIL — `cycle_pass` not in STEPS

- [ ] **Step 3: Insert `cycle_pass` step and update skip predicates**

In `work-end/work_end_orchestrator.py`, in the STEPS list:

**Insert `cycle_pass` after `merge_pass` (line ~919) and before `stamp_pass` (line ~920):**

```python
    StepDef("cycle_pass", "closing:merged", "lifecycle",
            skip_fn=_skip_not_cycle_mode,
            from_state="closing:merged", to_state="active",
            event="issue_cycle"),
```

**Update skip_fn on terminal steps:**

| Step (approx line) | Change |
|---------------------|--------|
| `write_marker` (~909) | `skip_fn=_or_skip(_skip_not_slot, _skip_cycle_mode)` |
| `stamp_pass` (~920) | `skip_fn=_skip_cycle_mode` |
| `write_landed` (~924) | `skip_fn=_or_skip(_skip_not_slot, _skip_cycle_mode)` |
| `archive_slot` (~940) | `skip_fn=_or_skip(_skip_not_slot, _skip_cycle_mode)` |
| `report_archive` (~943) | `skip_fn=_or_skip(_skip_not_slot, _skip_cycle_mode)` |
| `checkout_main` (~949) | `skip_fn=_or_skip(_skip_on_main, _skip_cycle_mode)` |
| `cleanup_stack` (~952) | `skip_fn=_or_skip(_skip_on_main, _skip_cycle_mode)` |
| `cleanup` (~955) | `skip_fn=_skip_cycle_mode` |
| `cleanup_pass` (~969) | `skip_fn=_or_skip(_skip_on_main, _skip_cycle_mode)` |
| `cleanup_main` (~972) | `skip_fn=_or_skip(_skip_not_main, _skip_cycle_mode)` |

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestCyclePassStep -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 5: Run full orchestrator test suite for regressions**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v --no-header 2>&1 | tail -30`
Expected: all PASSED

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat(#373): add cycle_pass step and wire cycle-mode skip predicates

Refs #373"
```

---

### Task 5: Retroactive `.landed` cleanup in `append_to_queue()`

**Files:**
- Modify: `work-slot/plan_manager.py:831-871` (add `slot_path` parameter and cleanup logic)
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `slot_state.transition(slot_dir, "active")` (from Task 3)
- Produces: retroactive `.landed` cleanup — no downstream consumers

- [ ] **Step 1: Write failing tests**

Add to `tests/test_plan_manager.py`:

```python
class TestAppendClearsLanded:
    """Tests for retroactive .landed cleanup in append_to_queue."""

    def _write_plan(self, path, content=None):
        if content is None:
            content = (
                "## State\nbranch: test-branch\nstate: active\n"
                "issue-repo: Org/repo\ncovers: 1\n\n"
                "## Queue\n- [x] org/repo#1 — Done ← active\n"
            )
        path.write_text(content)

    def test_append_clears_landed_file(self, tmp_path):
        from plan_manager import append_to_queue, QueueItem
        plan = tmp_path / ".plan"
        self._write_plan(plan)
        slot_dir = tmp_path / "slot-1"
        slot_dir.mkdir()
        (slot_dir / ".landed").write_text("landed_shas=abc\n")
        (slot_dir / ".slot").write_text("state: landed\n")
        append_to_queue(plan, [QueueItem(repo="org/repo", number=2, title="New")],
                        slot_path=slot_dir)
        assert not (slot_dir / ".landed").exists()

    def test_append_transitions_slot_state(self, tmp_path):
        from plan_manager import append_to_queue, QueueItem
        plan = tmp_path / ".plan"
        self._write_plan(plan)
        slot_dir = tmp_path / "slot-1"
        slot_dir.mkdir()
        (slot_dir / ".landed").write_text("landed_shas=abc\n")
        (slot_dir / ".slot").write_text("state: landed\n")
        append_to_queue(plan, [QueueItem(repo="org/repo", number=2, title="New")],
                        slot_path=slot_dir)
        content = (slot_dir / ".slot").read_text()
        assert "state: active" in content

    def test_append_no_slot_path_no_crash(self, tmp_path):
        from plan_manager import append_to_queue, QueueItem
        plan = tmp_path / ".plan"
        self._write_plan(plan)
        result = append_to_queue(plan, [QueueItem(repo="org/repo", number=2, title="New")])
        assert len(result) == 1

    def test_append_slot_path_no_landed_no_crash(self, tmp_path):
        from plan_manager import append_to_queue, QueueItem
        plan = tmp_path / ".plan"
        self._write_plan(plan)
        slot_dir = tmp_path / "slot-1"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text("state: active\n")
        result = append_to_queue(plan, [QueueItem(repo="org/repo", number=2, title="New")],
                                 slot_path=slot_dir)
        assert len(result) == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAppendClearsLanded -v --no-header 2>&1 | tail -20`
Expected: FAIL — `append_to_queue` does not accept `slot_path`

- [ ] **Step 3: Add `slot_path` parameter and cleanup logic**

In `work-slot/plan_manager.py`, modify `append_to_queue` (line ~831):

Change the signature to:
```python
def append_to_queue(plan_path: Path, new_items: list[QueueItem],
                    position: int | None = None,
                    skip_state_check: bool = False,
                    slot_path: Path | None = None) -> list[QueueItem]:
```

Add cleanup logic at the start of the function body (before the existing logic):
```python
    if slot_path is not None:
        landed = slot_path / ".landed"
        if landed.exists():
            landed.unlink()
            slot_file = slot_path / ".slot"
            if slot_file.exists():
                content = slot_file.read_text()
                if "state: landed" in content:
                    updated = content.replace("state: landed", "state: active")
                    slot_file.write_text(updated)
            import logging
            logging.getLogger(__name__).info(
                "Cleared stale .landed — slot re-activated with new work"
            )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAppendClearsLanded -v --no-header 2>&1 | tail -20`
Expected: all PASSED

- [ ] **Step 5: Run full plan_manager test suite for regressions**

Run: `python3 -m pytest tests/test_plan_manager.py -v --no-header 2>&1 | tail -30`
Expected: all PASSED

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git commit -m "feat(#373): clear stale .landed when appending new work to slot queue

Refs #373"
```

---

## References

- [2026-09-17-clear-stale-landed-design.md] — design spec this plan implements
- [decisions.md] — D1-D4 design rationale
- [work_end_orchestrator.py:203-254] — StepDef, existing skip predicates
- [work_end_orchestrator.py:433-480] — elevate_plan_inline (cycle detection pattern)
- [work_end_orchestrator.py:839-980] — STEPS list
- [project/lifecycle.py:109-132] — TRANSITION_TABLE
- [work-slot/slot_state.py:47-55] — VALID_TRANSITIONS
- [work-slot/plan_manager.py:831-871] — append_to_queue
- [orchestrator_engine.py:213] — skip_fn evaluation in run_loop
- [externalised-scripts-require-tests.md] — protocol: tests with scripts
- [evidence-before-claims.md] — protocol: verify before claiming done
- [GitHub #373] — slot 198 incident

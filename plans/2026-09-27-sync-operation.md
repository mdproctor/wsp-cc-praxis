# Sync Operation Implementation Plan (Phase 2 of #382)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #382 — Unified mechanical pipeline
**Issue group:** #382

**Goal:** Add `work sync` — a command that runs the full close ceremony (review, promote, rebase, squash, push, merge, close issues) but returns to `active` instead of stamping/archiving/checking out main. This lets users land completed work without closing the branch.

**Architecture:** Sync reuses the existing close pipeline. Two new lifecycle transitions (`work_sync` → `closing:review`, `sync_pass` → `active`) share the entire closing sequence with `work end`. The divergence point is after `closing:merged`: sync fires `sync_pass` (returns to active) instead of `stamp_pass` (continues to stamped). The orchestrator gains a `mode` field (`end`/`sync`) that skip predicates use to select the right terminal path. `work_chain.py` gains a `sync` evaluator that guards against syncing with no work to land.

**Tech Stack:** Python 3, pytest, subprocess (git CLI), pathlib

## Global Constraints

- No new lifecycle states — sync reuses `active` and the existing `closing:*` sequence
- `work_sync` entry transition must go through `commit_transition` (Phase 1 invariant)
- `sync_pass` must go through `commit_transition` with evidence gating
- Existing `work end` behavior must not change — sync is additive
- All skip predicates must compose cleanly with existing `_skip_cycle_mode` and `_skip_on_main`
- The `mode` field must default to `end` so all existing callers are unaffected

---

## Batch 1: Lifecycle transitions

After this batch: the state machine accepts `work_sync` and `sync_pass`
events. No orchestrator or chaining changes yet — this is the foundation.

### Task 1: Add work_sync and sync_pass transitions to lifecycle

**Files:**
- Modify: `project/lifecycle.py:109-142` — add transitions to TRANSITION_TABLE
- Modify: `project/lifecycle.py:144-173` — add INVALID_MESSAGES for sync
- Modify: `project/lifecycle.py:323-333` — add sync_pass to EVIDENCE_GATES
- Create: `tests/test_sync_lifecycle.py` — test new transitions

**Interfaces:**
- Consumes: `TRANSITION_TABLE`, `transition()`, `commit_transition()`, `EVIDENCE_GATES`
- Produces: `('active', 'work_sync')` and `('closing:stamped', 'sync_pass')` transitions

- [ ] **Step 1: Write failing tests**

```python
# tests/test_sync_lifecycle.py
"""Tests for work_sync and sync_pass lifecycle transitions."""

import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from lifecycle import (
    TRANSITION_TABLE,
    EVIDENCE_GATES,
    InvalidTransition,
    TransitionResult,
    commit_transition,
    transition,
    read_state,
    write_state,
)


def _init_git_repo(path: Path, state: str = "active") -> Path:
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "test@test.com"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
    plan = path / ".plan"
    plan.write_text(
        f"# Work Plan\n\n## State\nbranch: test-branch\nstate: {state}\ncovers: 1\n\n"
        f"## Queue\n- [ ] test#1 — Test\n"
    )
    subprocess.run(["git", "-C", str(path), "add", ".plan"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "initial"], capture_output=True)
    return plan


class TestWorkSyncTransition:
    def test_work_sync_in_transition_table(self):
        assert ("active", "work_sync") in TRANSITION_TABLE

    def test_work_sync_goes_to_closing_review(self):
        new_state, effects, _ = TRANSITION_TABLE[("active", "work_sync")]
        assert new_state == "closing:review"

    def test_work_sync_has_pre_close_sweep_effect(self):
        _, effects, _ = TRANSITION_TABLE[("active", "work_sync")]
        assert "pre_close_sweep" in effects

    def test_work_sync_transition_from_active(self, tmp_path):
        plan = _init_git_repo(tmp_path, "active")
        result = transition(plan, "work_sync")
        assert result.from_state == "active"
        assert result.new_state == "closing:review"
        assert result.event == "work_sync"

    def test_work_sync_rejected_from_idle(self, tmp_path):
        plan = tmp_path / ".plan"
        with pytest.raises(InvalidTransition):
            transition(plan, "work_sync")

    def test_work_sync_rejected_from_paused(self, tmp_path):
        plan = _init_git_repo(tmp_path, "paused")
        with pytest.raises(InvalidTransition):
            transition(plan, "work_sync")

    def test_work_sync_rejected_from_closing(self, tmp_path):
        plan = _init_git_repo(tmp_path, "closing:review")
        with pytest.raises(InvalidTransition):
            transition(plan, "work_sync")


class TestSyncPassTransition:
    def test_sync_pass_in_transition_table(self):
        assert ("closing:stamped", "sync_pass") in TRANSITION_TABLE

    def test_sync_pass_goes_to_active(self):
        new_state, effects, _ = TRANSITION_TABLE[("closing:stamped", "sync_pass")]
        assert new_state == "active"

    def test_sync_pass_has_clear_closing_markers_effect(self):
        _, effects, _ = TRANSITION_TABLE[("closing:stamped", "sync_pass")]
        assert "clear_closing_markers" in effects

    def test_sync_pass_transition_from_stamped(self, tmp_path):
        plan = _init_git_repo(tmp_path, "closing:stamped")
        result = transition(plan, "sync_pass")
        assert result.from_state == "closing:stamped"
        assert result.new_state == "active"

    def test_sync_pass_commits_state(self, tmp_path):
        plan = _init_git_repo(tmp_path, "closing:stamped")
        result = TransitionResult(
            from_state="closing:stamped",
            new_state="active",
            event="sync_pass",
        )
        commit_transition(plan, result, evidence={"stamp_shas": {}})
        assert read_state(plan) == "active"
        # Verify committed to git
        show = subprocess.run(
            ["git", "-C", str(tmp_path), "show", "HEAD:.plan"],
            capture_output=True, text=True,
        )
        assert "state: active" in show.stdout

    def test_sync_pass_rejected_from_active(self, tmp_path):
        plan = _init_git_repo(tmp_path, "active")
        with pytest.raises(InvalidTransition):
            transition(plan, "sync_pass")


class TestSyncPassEvidenceGating:
    def test_sync_pass_in_evidence_gates(self):
        assert "sync_pass" in EVIDENCE_GATES

    def test_sync_pass_requires_stamp_shas(self):
        assert "stamp_shas" in EVIDENCE_GATES["sync_pass"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_sync_lifecycle.py -v --tb=short`
Expected: FAIL — `work_sync` and `sync_pass` not in TRANSITION_TABLE

- [ ] **Step 3: Add transitions and evidence gate**

In `project/lifecycle.py`, add to TRANSITION_TABLE (after line 141):

```python
    # Sync (land without closing — returns to active after stamped)
    ('active', 'work_sync'):                 ('closing:review',    ['pre_close_sweep'],               []),
    ('closing:stamped', 'sync_pass'):        ('active',            ['clear_closing_markers'],          []),
```

Add to INVALID_MESSAGES (after line 172):

```python
    ('idle', 'work_sync'):       "Cannot sync — no active branch. Start work first.",
    ('paused', 'work_sync'):     "Cannot sync — branch is paused. Resume first.",
    ('drained', 'work_sync'):    "Cannot sync — queue is drained.",
    ('scaffolded', 'work_sync'): "Cannot sync — branch not yet active.",
    ('transitioning', 'work_sync'): "Cannot sync — issue transition in progress.",
```

Add to EVIDENCE_GATES (after line 331):

```python
    'sync_pass':     ['stamp_shas'],
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_sync_lifecycle.py tests/test_lifecycle.py -v --tb=short`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_sync_lifecycle.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add work_sync and sync_pass lifecycle transitions

Refs #382"
```

---

## Batch 2: Orchestrator sync mode

After this batch: the orchestrator accepts `mode=sync` and skips terminal
steps (stamp, archive, checkout main, cleanup). A new `sync_pass` step
fires the `sync_pass` lifecycle transition after merge.

### Task 2: Add mode parameter and sync skip predicates to orchestrator

**Files:**
- Modify: `work-end/work_end_orchestrator.py:172-196` — add `mode` field to OrchestratorContext
- Modify: `work-end/work_end_orchestrator.py:233-297` — add `_skip_sync_mode` and `_skip_not_sync_mode` predicates
- Modify: `work-end/work_end_orchestrator.py:886-1043` — update STEPS with sync skip predicates and new sync_pass step
- Modify: `work-end/work_end_orchestrator.py:1072-1198` — pass mode through run_orchestrator
- Modify: `work-end/close_progress.py:30-62` — add sync_pass to STEP_TO_PHASE
- Create: `tests/test_sync_orchestrator.py` — test sync mode step selection

**Interfaces:**
- Consumes: `OrchestratorContext`, `StepDef`, `_skip_cycle_mode`, `_or_skip`, `_fire_lifecycle`, `_build_evidence`
- Produces: `mode` field on `OrchestratorContext`, `_skip_sync_mode()`, `_skip_not_sync_mode()`, `sync_pass` StepDef

- [ ] **Step 1: Write failing tests**

```python
# tests/test_sync_orchestrator.py
"""Tests for sync mode in the work-end orchestrator."""

import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from work_end_orchestrator import (
    OrchestratorContext,
    STEPS,
    _skip_sync_mode,
    _skip_not_sync_mode,
)


def _make_ctx(mode: str = "end", **overrides) -> OrchestratorContext:
    defaults = dict(
        workspace=Path("/tmp/ws"),
        project=Path("/tmp/proj"),
        branch="test-branch",
        base_branch="main",
        meta_state="closing:stamped",
        on_main=False,
        in_slot=False,
        covers="1",
        issue_repo="test/repo",
        progress={},
        mode=mode,
    )
    defaults.update(overrides)
    return OrchestratorContext(**defaults)


class TestSyncSkipPredicates:
    def test_skip_sync_mode_true_when_sync(self):
        ctx = _make_ctx(mode="sync")
        assert _skip_sync_mode(ctx) is True

    def test_skip_sync_mode_false_when_end(self):
        ctx = _make_ctx(mode="end")
        assert _skip_sync_mode(ctx) is False

    def test_skip_not_sync_mode_true_when_end(self):
        ctx = _make_ctx(mode="end")
        assert _skip_not_sync_mode(ctx) is True

    def test_skip_not_sync_mode_false_when_sync(self):
        ctx = _make_ctx(mode="sync")
        assert _skip_not_sync_mode(ctx) is False


class TestSyncPassStepExists:
    def test_sync_pass_step_in_steps(self):
        step_names = [s.name for s in STEPS]
        assert "sync_pass" in step_names

    def test_sync_pass_is_lifecycle_step(self):
        step = next(s for s in STEPS if s.name == "sync_pass")
        assert step.step_type == "lifecycle"
        assert step.from_state == "closing:stamped"
        assert step.to_state == "active"
        assert step.event == "sync_pass"

    def test_sync_pass_skipped_when_not_sync(self):
        step = next(s for s in STEPS if s.name == "sync_pass")
        ctx = _make_ctx(mode="end")
        assert step.skip_fn(ctx) is True

    def test_sync_pass_not_skipped_when_sync(self):
        step = next(s for s in STEPS if s.name == "sync_pass")
        ctx = _make_ctx(mode="sync")
        assert step.skip_fn(ctx) is False


class TestSyncModeSkipsTerminalSteps:
    """In sync mode, stamp/archive/checkout/cleanup steps are skipped."""

    TERMINAL_STEPS = [
        "stamp_pass", "archive_slot", "report_archive",
        "checkout_main", "cleanup_stack", "cleanup",
        "cleanup_pass", "cleanup_main",
    ]

    def test_terminal_steps_skipped_in_sync(self):
        ctx = _make_ctx(mode="sync")
        for step_name in self.TERMINAL_STEPS:
            step = next((s for s in STEPS if s.name == step_name), None)
            if step and step.skip_fn:
                assert step.skip_fn(ctx) is True, (
                    f"Step '{step_name}' should be skipped in sync mode"
                )


class TestEndModeUnchanged:
    """End mode behavior must not change."""

    def test_stamp_pass_not_skipped_in_end_mode(self):
        ctx = _make_ctx(mode="end")
        step = next(s for s in STEPS if s.name == "stamp_pass")
        # stamp_pass is skipped by _skip_cycle_mode, not by mode
        # In end mode with no uncompleted items, it should NOT be skipped
        # (This test validates end mode is unaffected by sync changes)
        # Note: _skip_cycle_mode checks the .plan file which doesn't exist
        # in this test context, so it returns False (not cycle mode)
        assert step.skip_fn(ctx) is False

    def test_sync_pass_skipped_in_end_mode(self):
        ctx = _make_ctx(mode="end")
        step = next(s for s in STEPS if s.name == "sync_pass")
        assert step.skip_fn(ctx) is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_sync_orchestrator.py -v --tb=short`
Expected: FAIL — `_skip_sync_mode` doesn't exist, `mode` not on OrchestratorContext

- [ ] **Step 3: Add mode to OrchestratorContext**

In `work-end/work_end_orchestrator.py`, add `mode` field to `OrchestratorContext` (after line 189):

```python
    mode: str = "end"  # "end" or "sync"
```

- [ ] **Step 4: Add sync skip predicates**

In `work-end/work_end_orchestrator.py`, add after `_skip_not_cycle_mode` (line 296):

```python
def _skip_sync_mode(ctx) -> bool:
    """Skip terminal steps when in sync mode (land without closing)."""
    return ctx.mode == "sync"


def _skip_not_sync_mode(ctx) -> bool:
    """Skip sync step when in end mode."""
    return ctx.mode != "sync"
```

- [ ] **Step 5: Add sync_pass step and update terminal step skip predicates**

In the STEPS list, add `sync_pass` step after `stamp_pass` (line 977):

```python
    StepDef("sync_pass", "closing:stamped", "lifecycle",
            skip_fn=_skip_not_sync_mode,
            from_state="closing:stamped", to_state="active", event="sync_pass"),
```

Update terminal steps to also skip in sync mode. Change these skip_fn entries:

| Step | Current skip_fn | New skip_fn |
|------|----------------|-------------|
| `stamp_pass` (line 975) | `_skip_cycle_mode` | `_or_skip(_skip_cycle_mode, _skip_sync_mode)` |
| `write_landed` (line 980) | `_or_skip(_skip_not_slot, _skip_cycle_mode)` | `_or_skip(_skip_not_slot, _skip_cycle_mode, _skip_sync_mode)` |
| `archive_slot` (line 999) | `_or_skip(_skip_not_slot, _skip_cycle_mode)` | `_or_skip(_skip_not_slot, _skip_cycle_mode, _skip_sync_mode)` |
| `report_archive` (line 1003) | `_or_skip(_skip_not_slot, _skip_cycle_mode)` | `_or_skip(_skip_not_slot, _skip_cycle_mode, _skip_sync_mode)` |
| `elevate_plan` (line 1006) | `_skip_not_slot` | `_or_skip(_skip_not_slot, _skip_sync_mode)` |
| `checkout_main` (line 1009) | `_or_skip(_skip_on_main, _skip_cycle_mode)` | `_or_skip(_skip_on_main, _skip_cycle_mode, _skip_sync_mode)` |
| `cleanup_stack` (line 1013) | `_or_skip(_skip_on_main, _skip_cycle_mode)` | `_or_skip(_skip_on_main, _skip_cycle_mode, _skip_sync_mode)` |
| `cleanup` (line 1016) | `_skip_cycle_mode` | `_or_skip(_skip_cycle_mode, _skip_sync_mode)` |
| `cleanup_pass` (line 1032) | `_or_skip(_skip_on_main, _skip_cycle_mode)` | `_or_skip(_skip_on_main, _skip_cycle_mode, _skip_sync_mode)` |
| `cleanup_main` (line 1035) | `_or_skip(_skip_not_main, _skip_cycle_mode)` | `_or_skip(_skip_not_main, _skip_cycle_mode, _skip_sync_mode)` |

Steps that should NOT skip in sync mode (they run in both modes):
- `close_issues` — issues were completed, close them
- `verify` — verify the landing
- `report_close_issues`, `report_verify` — reporting
- `upstream_push` — push upstream if configured
- `report_scaffold`, `arc42_scan`, `session_rename`, `garden_feedback`, `notes` — judgment wrap-up steps that execute before the terminal lifecycle transition

Note: `close_issues`, `verify`, the report steps, and `upstream_push` run before `sync_pass`/`stamp_pass` in the step list (lines 984–998), so they execute in both modes. The judgment steps (`arc42_scan`, `session_rename`, `garden_feedback`, `notes`) also run before the terminal lifecycle steps. These should be skipped in sync mode since they're session-end activities:

| Step | Current skip_fn | New skip_fn |
|------|----------------|-------------|
| `arc42_scan` (line 1022) | None | `_skip_sync_mode` |
| `session_rename` (line 1024) | None | `_skip_sync_mode` |
| `garden_feedback` (line 1026) | None | `_skip_sync_mode` |
| `notes` (line 1028) | None | `_skip_sync_mode` |
| `report_scaffold` (line 1020) | None | `_skip_sync_mode` |

- [ ] **Step 6: Add sync_pass to STEP_TO_PHASE and _build_evidence**

In `work-end/close_progress.py`, add to `STEP_TO_PHASE` (after line 62):

```python
    "sync_pass": "closing:stamped",
```

In `work-end/work_end_orchestrator.py`, add to `_build_evidence()` (after line 655):

```python
    if event == "sync_pass":
        return {"stamp_shas": ctx.landed_shas or {}}
```

- [ ] **Step 7: Pass mode through run_orchestrator**

In `work-end/work_end_orchestrator.py`, in `run_orchestrator()`:

Parse mode from args (after line 1087):
```python
    mode = args.get("mode", "end")
```

Add to OrchestratorContext construction (around line 1194):
```python
        mode=mode,
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_sync_orchestrator.py tests/test_work_end_orchestrator.py -v --tb=short`
Expected: All PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py work-end/close_progress.py tests/test_sync_orchestrator.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add sync mode to orchestrator with skip predicates

Refs #382"
```

---

## Batch 3: Chaining and routing

After this batch: `work sync` is a user-invocable command. The chaining
engine evaluates it, the work skill routes it, and the orchestrator runs
the close pipeline in sync mode.

### Task 3: Add sync command to work_chain.py

**Files:**
- Modify: `project/work_chain.py:44-72` — add `sync` branch to `evaluate()`
- Modify: `project/work_chain.py` — add `_evaluate_sync()` function
- Create: `tests/test_sync_chain.py` — test sync evaluation

**Interfaces:**
- Consumes: `evaluate()`, `_check_issue_state()`, `_parse_remaining()`
- Produces: `evaluate("sync", ctx)` returns appropriate `CHAIN_DIRECTIVE`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_sync_chain.py
"""Tests for sync command evaluation in work_chain.py."""

import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from work_chain import evaluate


class TestSyncEvaluation:
    def test_sync_proceeds_when_active_with_work(self):
        ctx = {
            "META_STATE": "active",
            "HAS_PLAN": "yes",
            "ACTIVE_ISSUE": "test/repo#1",
            "PLAN_POSITION": "0/3",
            "ON_MAIN": "no",
        }
        result = evaluate("sync", ctx, issue_state="OPEN")
        assert result["DIRECTIVE"] == "proceed"

    def test_sync_blocked_on_main(self):
        ctx = {
            "META_STATE": "active",
            "HAS_PLAN": "yes",
            "ACTIVE_ISSUE": "test/repo#1",
            "PLAN_POSITION": "0/1",
            "ON_MAIN": "yes",
        }
        result = evaluate("sync", ctx)
        assert result["DIRECTIVE"] != "proceed"

    def test_sync_blocked_when_no_active_issue(self):
        ctx = {
            "META_STATE": "active",
            "HAS_PLAN": "no",
            "ACTIVE_ISSUE": "",
            "PLAN_POSITION": "",
            "ON_MAIN": "no",
        }
        result = evaluate("sync", ctx)
        assert result["DIRECTIVE"] != "proceed"

    def test_sync_blocked_when_drained(self):
        ctx = {
            "META_STATE": "drained",
            "HAS_PLAN": "yes",
            "ACTIVE_ISSUE": "",
            "PLAN_POSITION": "",
            "ON_MAIN": "no",
        }
        result = evaluate("sync", ctx)
        assert result["DIRECTIVE"] != "proceed"

    def test_sync_blocked_when_paused(self):
        ctx = {
            "META_STATE": "paused",
            "HAS_PLAN": "yes",
            "ACTIVE_ISSUE": "test/repo#1",
            "PLAN_POSITION": "0/1",
            "ON_MAIN": "no",
        }
        result = evaluate("sync", ctx)
        assert result["DIRECTIVE"] != "proceed"

    def test_sync_includes_base_fields(self):
        ctx = {
            "META_STATE": "active",
            "HAS_PLAN": "yes",
            "ACTIVE_ISSUE": "test/repo#1",
            "PLAN_POSITION": "0/3",
            "ON_MAIN": "no",
        }
        result = evaluate("sync", ctx, issue_state="OPEN")
        assert "ACTIVE_ISSUE" in result
        assert "ON_MAIN" in result
        assert "QUEUE_REMAINING" in result
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_sync_chain.py -v --tb=short`
Expected: FAIL — `sync` command falls through to default `proceed` (wrong for blocked cases)

- [ ] **Step 3: Add _evaluate_sync and sync branch**

In `project/work_chain.py`, add the sync branch in `evaluate()` (after line 70):

```python
    elif command == "sync":
        return {**base, **_evaluate_sync(state, has_plan, active_issue, on_main)}
```

Add the evaluator function (after `_evaluate_find`, around line 114):

```python
def _evaluate_sync(
    state: str, has_plan: bool, active_issue: str, on_main: bool,
) -> dict:
    if on_main:
        return {"DIRECTIVE": "chain_to_end", "REASON": "sync_requires_branch"}
    if state in ("drained", "paused", "idle", "scaffolded", "transitioning"):
        return {"DIRECTIVE": "chain_to_end", "REASON": f"cannot_sync_from_{state}"}
    if not active_issue:
        return {"DIRECTIVE": "chain_to_end", "REASON": "no_active_work"}
    return {"DIRECTIVE": "proceed", "REASON": "ready_to_sync"}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_sync_chain.py tests/test_work_end_orchestrator.py -v --tb=short`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_chain.py tests/test_sync_chain.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add sync command to work_chain.py

Refs #382"
```

### Task 4: Add work sync routing to work/SKILL.md

**Files:**
- Modify: `work/SKILL.md` — add `work sync` to routing table and Step 1

**Interfaces:**
- Consumes: `ctx.py` output (CHAIN_DIRECTIVE), orchestrator `mode=sync` parameter
- Produces: `work sync` command available to users

- [ ] **Step 1: Add work sync to routing table**

In `work/SKILL.md`, add to the routing table (Step 1):

```markdown
| `work sync` | → read `CHAIN_DIRECTIVE` from ctx.py → follow directive (Step 1c) |
```

- [ ] **Step 2: Add Step 7 — work sync execution**

Add after Step 6 in work/SKILL.md:

```markdown
**Step 7 — `work sync` (land without closing)**

Lands completed work via the close ceremony but returns to active instead
of stamping/archiving. The branch stays open for continued work.

1. Run ctx.py. Read `CHAIN_DIRECTIVE` — if not `proceed`, follow the
   directive (Step 1c).
2. Fire transition:
   ```bash
   python3 ~/.claude/skills/project/lifecycle.py transition <PLAN_PATH> work_sync
   ```
3. Commit transition:
   ```bash
   python3 ~/.claude/skills/project/lifecycle.py commit-transition <PLAN_PATH> from_state=active new_state=closing:review event=work_sync
   ```
4. Run the orchestrator in sync mode:
   ```bash
   python3 work-end/work_end_orchestrator.py \
       workspace=<ws> project=<proj> branch=<branch> \
       base_branch=<base> meta_state=closing:review \
       mode=sync \
       [covers=...] [issue_repo=...] [plan_path=...]
   ```
   The orchestrator runs the full close sequence. After `closing:merged`,
   it fires `sync_pass` (→ `active`) instead of `stamp_pass` (→ `closing:stamped`).
5. After sync completes, state is `active`. Branch stays open.
   Report: "Work synced — landed on main, branch still active."
```

- [ ] **Step 3: Add work sync to Step 4 options**

In Step 4 (on feature branch: contextual options), add sync option:

```markdown
If `HAS_PLAN=yes` or multiple issues on branch:
> N. **sync** — land completed work, keep branch open
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#382): add work sync routing to work skill

Refs #382"
```

---

## Batch 4: Integration test

After this batch: end-to-end validation that sync mode runs the pipeline
correctly and leaves state as `active`.

### Task 5: Sync mode integration test

**Files:**
- Create: `tests/test_sync_integration.py` — end-to-end dry-run test

**Interfaces:**
- Consumes: `run_orchestrator()`, `read_close_progress()`, `STEPS`
- Produces: Tests proving sync runs close sequence and returns to active

- [ ] **Step 1: Write integration test**

```python
# tests/test_sync_integration.py
"""Integration test for sync mode orchestrator dry-run."""

import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

from work_end_orchestrator import run_orchestrator, STEPS


class TestSyncDryRun:
    def _run_sync_loop(self, tmp_path):
        """Run the orchestrator in sync+dry_run mode, collecting actions."""
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        plan = ws / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: test-branch\nstate: active\ncovers: 1\n\n"
            "## Queue\n- [ ] test#1 — Test\n"
        )

        args = {
            "workspace": str(ws),
            "project": str(proj),
            "branch": "test-branch",
            "base_branch": "main",
            "meta_state": "closing:review",
            "mode": "sync",
            "on_main": "no",
            "in_slot": "no",
            "covers": "1",
            "issue_repo": "test/repo",
            "plan_path": str(plan),
            "dry_run": "yes",
        }

        actions = []
        max_iterations = 100
        for _ in range(max_iterations):
            result = run_orchestrator(dict(args))
            action = result.get("ACTION", "")
            actions.append(result)

            if action == "complete":
                break
            elif action == "error":
                break
            elif result.get("STEP"):
                step_name = result["STEP"]
                args["step_done"] = step_name
                if result.get("VERIFY_REQUIRED") == "yes":
                    args["produced"] = "0"
            elif action in ("run_step",):
                step_name = result.get("STEP", "")
                args["step_done"] = step_name

        return actions

    def test_sync_reaches_complete(self, tmp_path):
        actions = self._run_sync_loop(tmp_path)
        last = actions[-1]
        assert last.get("ACTION") == "complete" or last.get("FINAL_STATE") is not None

    def test_sync_skips_stamp_pass(self, tmp_path):
        actions = self._run_sync_loop(tmp_path)
        steps_run = [a.get("STEP", "") for a in actions]
        assert "stamp_pass" not in steps_run

    def test_sync_skips_checkout_main(self, tmp_path):
        actions = self._run_sync_loop(tmp_path)
        steps_run = [a.get("STEP", "") for a in actions]
        assert "checkout_main" not in steps_run

    def test_sync_skips_cleanup_pass(self, tmp_path):
        actions = self._run_sync_loop(tmp_path)
        steps_run = [a.get("STEP", "") for a in actions]
        assert "cleanup_pass" not in steps_run
```

- [ ] **Step 2: Run tests**

Run: `python3 -m pytest tests/test_sync_integration.py -v --tb=short`
Expected: All PASS

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: No regressions

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/test_sync_integration.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "test(#382): sync mode integration tests

Refs #382"
```

---

## References

- `specs/issue-382-unified-pipeline/2026-09-27-unified-pipeline-design.md` — design spec (Section B: Sync Operation)
- `specs/issue-382-unified-pipeline/decisions.md` — D4 (sync as first-class), D5 (.plan-next pattern)
- `project/lifecycle.py:109-142` — TRANSITION_TABLE
- `project/lifecycle.py:323-333` — EVIDENCE_GATES
- `project/lifecycle.py:410-468` — commit_transition
- `project/work_chain.py:44-72` — evaluate() routing
- `work-end/work_end_orchestrator.py:172-196` — OrchestratorContext
- `work-end/work_end_orchestrator.py:233-297` — skip predicates
- `work-end/work_end_orchestrator.py:886-1043` — STEPS list
- `work-end/work_end_orchestrator.py:1072-1198` — run_orchestrator
- `work-end/close_progress.py:18-62` — LIFECYCLE_PHASE_ORDER, STEP_TO_PHASE
- GitHub #382 — unified mechanical pipeline

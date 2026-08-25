# Unified Work Queue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #261 — Unified work queue: bidirectional chaining for continue/next/end/find
**Issue group:** #261

**Goal:** Unify the work lifecycle commands with bidirectional chaining and `.plan` on main, so no matter which command is invoked, the user lands at the right place.

**Architecture:** New `work_chain.py` module evaluates chaining directives in Python (deterministic, testable). Lifecycle state machine gets a `drained` state for "queue empty, ceremony done, plan persists." All hardcoded `"main"` comparisons replaced with `BASE_BRANCH` parameter. Skills read Python directives instead of making routing decisions.

**Tech Stack:** Python 3.12, pytest, lifecycle state machine (lifecycle.py), plan_manager.py, ctx.py

## Global Constraints

- All chaining decisions in Python — LLM reads directives, never decides routing
- `drained` state (not `ended`) — avoids worklog collision
- No self-transitions for routing — state machine events for actual state changes only
- Hardcoded `"main"` replaced everywhere with `BASE_BRANCH`
- All new scripts require tests (protocol: externalised-scripts-require-tests)

---

## Batch 1: Lifecycle foundation

### Task 1: Add `drained` state to lifecycle state machine

**Files:**
- Modify: `project/lifecycle.py:25-35` (VALID_STATES, TRANSIENT_STATES, RESTING_STATES)
- Modify: `project/lifecycle.py:87-108` (TRANSITION_TABLE)
- Modify: `project/lifecycle.py:110-133` (INVALID_MESSAGES)
- Modify: `project/lifecycle.py:258-264` (_LIFECYCLE_TO_WORKLOG)
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Produces: `drained` in VALID_STATES, transitions `(closing:stamped, cleanup_main) → drained`, `(drained, work_find) → transitioning`, INVALID_MESSAGES for drained, worklog mapping `drained → idle`

- [ ] **Step 1: Write failing tests for drained state transitions**

Add to `tests/test_lifecycle.py`:

```python
def test_cleanup_main_transitions_to_drained(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: closing:stamped\n")
    result = transition(plan, "cleanup_main")
    assert result.from_state == "closing:stamped"
    assert result.new_state == "drained"
    assert "write_plan_drained" in result.effects
    assert "write_handoff" in result.post_commit_effects
    assert "return_to_main" not in result.post_commit_effects


def test_work_find_transitions_drained_to_transitioning(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n")
    result = transition(plan, "work_find")
    assert result.from_state == "drained"
    assert result.new_state == "transitioning"
    assert "queue_populated" in result.effects


def test_drained_rejects_work_end(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n")
    with pytest.raises(InvalidTransition, match="Already drained"):
        transition(plan, "work_end")


def test_drained_rejects_work_pause(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n")
    with pytest.raises(InvalidTransition, match="Nothing active to pause"):
        transition(plan, "work_pause")


def test_drained_rejects_work_resume(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n")
    with pytest.raises(InvalidTransition, match="work find"):
        transition(plan, "work_resume")


def test_drained_rejects_work_next(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n")
    with pytest.raises(InvalidTransition, match="Queue is empty"):
        transition(plan, "work_next")


def test_drained_in_valid_states():
    from lifecycle import VALID_STATES, RESTING_STATES
    assert "drained" in VALID_STATES
    assert "drained" in RESTING_STATES


def test_drained_worklog_mapping():
    from lifecycle import _LIFECYCLE_TO_WORKLOG
    assert _LIFECYCLE_TO_WORKLOG["drained"] == "idle"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py -v -k "drained"`
Expected: FAIL — `drained` not in VALID_STATES, transitions not defined

- [ ] **Step 3: Add drained to VALID_STATES**

In `project/lifecycle.py`, add `'drained'` to `VALID_STATES` frozenset (line 25-30). It is a resting state, not transient.

```python
VALID_STATES = frozenset({
    'idle', 'scaffolded', 'active', 'transitioning', 'paused', 'drained',
    'closing:review', 'closing:verified', 'closing:promoted',
    'closing:pushed', 'closing:merged', 'closing:stamped',
})
```

- [ ] **Step 4: Add transitions to TRANSITION_TABLE**

After line 104 (`closing:stamped`, `cleanup_pass`), add:

```python
    ('closing:stamped', 'cleanup_main'):    ('drained',        ['write_plan_drained'],                                                ['write_handoff']),
    # Re-activation from drained via work find
    ('drained', 'work_find'):               ('transitioning',  ['queue_populated'],                                                    []),
```

- [ ] **Step 5: Add INVALID_MESSAGES for drained**

After line 132 (last abort entry), add:

```python
    ('drained', 'work_end'):       "Already drained. Use `work find` to discover new work.",
    ('drained', 'work_pause'):     "Nothing active to pause.",
    ('drained', 'work_resume'):    "Nothing to resume. Use `work find` to discover new work.",
    ('drained', 'work_next'):      "Queue is empty. Use `work find` to discover new work.",
    ('drained', 'work_continue'):  "Queue is drained. Use `work find` to discover new work.",
    ('drained', 'work'):           "Queue is drained. Use `work find` to discover new work, or `work start #N` for a specific issue.",
```

- [ ] **Step 6: Add worklog mapping**

In `_LIFECYCLE_TO_WORKLOG` (line 258-264), add:

```python
    'drained': 'idle',
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_lifecycle.py -v -k "drained"`
Expected: ALL PASS

- [ ] **Step 8: Run full lifecycle test suite**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS — no regressions

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_lifecycle.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): add drained state to lifecycle — transitions, messages, worklog mapping"
```

---

## Batch 2: Chaining engine

### Task 2: Create work_chain.py with bidirectional chaining

**Files:**
- Create: `project/work_chain.py`
- Test: `tests/test_work_chain.py`

**Interfaces:**
- Consumes: `plan_manager.detect()` → `dict | None`, `lifecycle.read_state()` → `str | None`, `gh issue view` for issue state
- Produces: `evaluate(command, ctx) → dict` with `DIRECTIVE`, `REASON`, `ACTIVE_ISSUE`, `ISSUE_STATE`, `QUEUE_REMAINING`, `ON_MAIN` keys. Called by ctx.py.

- [ ] **Step 1: Write failing tests for evaluate_continue**

Create `tests/test_work_chain.py`:

```python
import pytest
from pathlib import Path

from work_chain import evaluate


def test_continue_with_active_open_issue():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "0/3",
        "ON_MAIN": "no",
    }
    result = evaluate("continue", ctx, issue_state="OPEN")
    assert result["DIRECTIVE"] == "proceed"


def test_continue_with_active_closed_issue():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "1/3",
        "ON_MAIN": "no",
    }
    result = evaluate("continue", ctx, issue_state="CLOSED")
    assert result["DIRECTIVE"] == "chain_to_next"
    assert result["REASON"] == "active_issue_done"


def test_continue_with_no_plan():
    ctx = {
        "META_STATE": "",
        "HAS_PLAN": "no",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "",
        "ON_MAIN": "yes",
    }
    result = evaluate("continue", ctx)
    assert result["DIRECTIVE"] == "chain_to_next"
    assert result["REASON"] == "no_active_work"


def test_continue_with_drained_state():
    ctx = {
        "META_STATE": "drained",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "3/3",
        "ON_MAIN": "yes",
    }
    result = evaluate("continue", ctx)
    assert result["DIRECTIVE"] == "chain_to_next"
    assert result["REASON"] == "queue_drained"
```

- [ ] **Step 2: Write failing tests for evaluate_next**

Append to `tests/test_work_chain.py`:

```python
def test_next_with_open_issue_guards_back():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "0/3",
        "ON_MAIN": "no",
    }
    result = evaluate("next", ctx, issue_state="OPEN")
    assert result["DIRECTIVE"] == "guard_continue"
    assert result["REASON"] == "issue_still_open"
    assert result["ACTIVE_ISSUE"] == "42"


def test_next_with_closed_issue_proceeds():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "1/3",
        "ON_MAIN": "no",
    }
    result = evaluate("next", ctx, issue_state="CLOSED")
    assert result["DIRECTIVE"] == "proceed"


def test_next_with_empty_queue_chains_to_end():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "3/3",
        "ON_MAIN": "no",
    }
    result = evaluate("next", ctx)
    assert result["DIRECTIVE"] == "chain_to_end"
    assert result["REASON"] == "queue_empty"


def test_next_with_no_plan_chains_to_end():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "no",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "",
        "ON_MAIN": "no",
    }
    result = evaluate("next", ctx)
    assert result["DIRECTIVE"] == "chain_to_end"
    assert result["REASON"] == "no_plan"
```

- [ ] **Step 3: Write failing tests for evaluate_end**

Append to `tests/test_work_chain.py`:

```python
def test_end_with_remaining_queue_guards_back():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "1/3",
        "ON_MAIN": "no",
    }
    result = evaluate("end", ctx, issue_state="OPEN")
    assert result["DIRECTIVE"] == "guard_next"
    assert result["REASON"] == "queue_not_empty"
    assert result["QUEUE_REMAINING"] == "2"


def test_end_with_empty_queue_proceeds():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "3/3",
        "ON_MAIN": "no",
    }
    result = evaluate("end", ctx)
    assert result["DIRECTIVE"] == "proceed"


def test_end_with_no_plan_proceeds():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "no",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "",
        "ON_MAIN": "no",
    }
    result = evaluate("end", ctx)
    assert result["DIRECTIVE"] == "proceed"
```

- [ ] **Step 4: Write failing tests for evaluate_find**

Append to `tests/test_work_chain.py`:

```python
def test_find_with_active_work_guards_back():
    ctx = {
        "META_STATE": "active",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "42",
        "PLAN_POSITION": "0/3",
        "ON_MAIN": "no",
    }
    result = evaluate("find", ctx, issue_state="OPEN")
    assert result["DIRECTIVE"] == "guard_next"
    assert result["REASON"] == "unfinished_work"


def test_find_with_drained_state_proceeds():
    ctx = {
        "META_STATE": "drained",
        "HAS_PLAN": "yes",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "3/3",
        "ON_MAIN": "yes",
    }
    result = evaluate("find", ctx)
    assert result["DIRECTIVE"] == "proceed"


def test_find_with_no_plan_proceeds():
    ctx = {
        "META_STATE": "",
        "HAS_PLAN": "no",
        "ACTIVE_ISSUE": "",
        "PLAN_POSITION": "",
        "ON_MAIN": "yes",
    }
    result = evaluate("find", ctx)
    assert result["DIRECTIVE"] == "proceed"
```

- [ ] **Step 5: Run all tests to verify they fail**

Run: `python3 -m pytest tests/test_work_chain.py -v`
Expected: FAIL — module `work_chain` not found

- [ ] **Step 6: Implement work_chain.py**

Create `project/work_chain.py`:

```python
"""Bidirectional chaining for work lifecycle commands.

Evaluates the current state and returns a directive telling the LLM
which action to take. All routing decisions are deterministic — the
LLM reads the directive and follows it, never decides routing itself.

Chain: continue ←→ next ←→ end ←→ find
  →  "nothing to do, cascade forward"
  ←  "not ready, go back"
"""

from __future__ import annotations

import subprocess


def _check_issue_state(issue_number: str, issue_repo: str) -> str:
    if not issue_number or not issue_repo:
        return "UNKNOWN"
    result = subprocess.run(
        ["gh", "issue", "view", issue_number, "--repo", issue_repo,
         "--json", "state", "--jq", ".state"],
        capture_output=True, text=True, timeout=10,
    )
    return result.stdout.strip() if result.returncode == 0 else "UNKNOWN"


def _parse_remaining(position: str) -> int:
    if not position or "/" not in position:
        return 0
    parts = position.split("/")
    try:
        completed, total = int(parts[0]), int(parts[1])
        return max(0, total - completed)
    except (ValueError, IndexError):
        return 0


def _has_active_work(ctx: dict, issue_state: str | None = None) -> bool:
    if ctx.get("ACTIVE_ISSUE") and issue_state in ("OPEN", "UNKNOWN", None):
        return True
    return False


def _queue_is_empty(ctx: dict) -> bool:
    if ctx.get("HAS_PLAN") != "yes":
        return True
    if not ctx.get("ACTIVE_ISSUE"):
        return True
    return False


def evaluate(
    command: str,
    ctx: dict,
    issue_state: str | None = None,
) -> dict:
    state = ctx.get("META_STATE", "")
    has_plan = ctx.get("HAS_PLAN") == "yes"
    active_issue = ctx.get("ACTIVE_ISSUE", "")
    position = ctx.get("PLAN_POSITION", "")
    on_main = ctx.get("ON_MAIN") == "yes"
    remaining = _parse_remaining(position)

    base = {
        "ACTIVE_ISSUE": active_issue,
        "ISSUE_STATE": issue_state or "",
        "QUEUE_REMAINING": str(remaining),
        "ON_MAIN": "yes" if on_main else "no",
    }

    if command == "continue":
        return {**base, **_evaluate_continue(state, has_plan, active_issue, issue_state)}
    elif command == "next":
        return {**base, **_evaluate_next(state, has_plan, active_issue, issue_state, remaining)}
    elif command == "end":
        return {**base, **_evaluate_end(state, has_plan, active_issue, issue_state, remaining)}
    elif command == "find":
        return {**base, **_evaluate_find(state, has_plan, active_issue, issue_state)}
    else:
        return {**base, "DIRECTIVE": "proceed", "REASON": "unknown_command"}


def _evaluate_continue(
    state: str, has_plan: bool, active_issue: str, issue_state: str | None,
) -> dict:
    if state == "drained":
        return {"DIRECTIVE": "chain_to_next", "REASON": "queue_drained"}
    if not has_plan or not active_issue:
        return {"DIRECTIVE": "chain_to_next", "REASON": "no_active_work"}
    if issue_state == "CLOSED":
        return {"DIRECTIVE": "chain_to_next", "REASON": "active_issue_done"}
    return {"DIRECTIVE": "proceed", "REASON": "active_work_exists"}


def _evaluate_next(
    state: str, has_plan: bool, active_issue: str,
    issue_state: str | None, remaining: int,
) -> dict:
    if not has_plan:
        return {"DIRECTIVE": "chain_to_end", "REASON": "no_plan"}
    if not active_issue:
        return {"DIRECTIVE": "chain_to_end", "REASON": "queue_empty"}
    if issue_state == "OPEN":
        return {"DIRECTIVE": "guard_continue", "REASON": "issue_still_open"}
    return {"DIRECTIVE": "proceed", "REASON": "issue_complete"}


def _evaluate_end(
    state: str, has_plan: bool, active_issue: str,
    issue_state: str | None, remaining: int,
) -> dict:
    if has_plan and active_issue and issue_state == "OPEN":
        return {"DIRECTIVE": "guard_next", "REASON": "queue_not_empty"}
    if has_plan and remaining > 0 and active_issue:
        return {"DIRECTIVE": "guard_next", "REASON": "queue_not_empty"}
    return {"DIRECTIVE": "proceed", "REASON": "ready_to_close"}


def _evaluate_find(
    state: str, has_plan: bool, active_issue: str, issue_state: str | None,
) -> dict:
    if active_issue and issue_state in ("OPEN", "UNKNOWN"):
        return {"DIRECTIVE": "guard_next", "REASON": "unfinished_work"}
    return {"DIRECTIVE": "proceed", "REASON": "ready_to_find"}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_chain.py -v`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_chain.py tests/test_work_chain.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): work_chain.py — bidirectional chaining directives"
```

---

## Batch 3: Routing infrastructure

### Task 3: Fix hardcoded "main" and add drained route

**Files:**
- Modify: `project/work_state.py:48-129` (detect function)
- Modify: `project/work_health.py:69-98,165-177` (meta_consistency, stale_scaffold)
- Test: `tests/test_work_state.py` (if exists, otherwise `tests/test_lifecycle.py`)

**Interfaces:**
- Consumes: `BASE_BRANCH` from ctx.py CLAUDE.md parsing, `lifecycle.read_state()` for drained detection
- Produces: Updated `WorkState.route` with new `"drained"` value, `work_state.detect()` accepts `base_branch` parameter

- [ ] **Step 1: Write failing tests for drained route and base_branch parameter**

```python
def test_detect_drained_route_on_main(tmp_path):
    """On main with .plan in drained state, route should be 'drained'."""
    plan = tmp_path / ".plan"
    plan.write_text("## State\nbranch: main\nstate: drained\n\n## Queue\n")
    # Mock topology with workspace = tmp_path
    topo = make_test_topo(tmp_path, branch="main")
    state = detect(topo, base_branch="main")
    assert state.route == "drained"


def test_detect_uses_base_branch_not_hardcoded_main(tmp_path):
    """Projects with base branch 'develop' should detect on_main correctly."""
    topo = make_test_topo(tmp_path, branch="develop")
    state = detect(topo, base_branch="develop")
    assert state.on_main is True


def test_detect_on_main_false_for_feature_branch(tmp_path):
    topo = make_test_topo(tmp_path, branch="issue-42-foo")
    state = detect(topo, base_branch="main")
    assert state.on_main is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_state.py -v -k "drained or base_branch"`
Expected: FAIL

- [ ] **Step 3: Update work_state.detect() to accept base_branch parameter**

In `project/work_state.py`, modify the `detect` function signature and on_main check:

```python
def detect(topo: Topology, base_branch: str = "main") -> WorkState:
    workspace = str(topo.workspace)
    project = str(topo.project)
    current_branch = _run("git", "-C", workspace, "branch", "--show-current")
    on_main = current_branch == base_branch
```

- [ ] **Step 4: Add drained route detection**

In the routing section of `detect()` (lines 124-129), add drained check:

```python
    if on_main:
        if meta_state == "drained":
            route = "drained"
        elif stack_depth > 0:
            route = "resume_stack"
        else:
            route = "start"
    elif workspace_dirty:
        route = "workspace_dirty"
    else:
        route = "resume_branch"
```

- [ ] **Step 5: Update ctx.py to pass BASE_BRANCH to detect()**

In `project/ctx.py`, find where `work_state.detect(topo)` is called and pass `base_branch`:

```python
ws = work_state.detect(topo, base_branch=base_branch)
```

Where `base_branch` is already parsed from CLAUDE.md (line ~107).

- [ ] **Step 6: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_state.py -v -k "drained or base_branch"`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_state.py project/ctx.py tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): fix hardcoded main, add drained route detection"
```

### Task 4: Update health checks for drained .plan on main

**Files:**
- Modify: `project/work_health.py:69-98` (check_meta_consistency)
- Modify: `project/work_health.py:165-177` (check_stale_scaffold_on_main)
- Test: `tests/test_work_health.py` (if exists)

**Interfaces:**
- Consumes: `lifecycle.read_state()` to check if `.plan` has `drained` state
- Produces: Health checks that accept `drained` state `.plan` on main as valid

- [ ] **Step 1: Write failing tests**

```python
def test_stale_scaffold_skips_drained_plan(tmp_path):
    """A .plan with state: drained on main is intentional, not stale."""
    (tmp_path / ".plan").write_text("## State\nbranch: main\nstate: drained\n")
    result = check_stale_scaffold_on_main(str(tmp_path), str(tmp_path))
    assert "STATUS=ok" in result


def test_meta_consistency_accepts_branch_main_on_main(tmp_path):
    """A .plan with branch: main is valid when on main."""
    (tmp_path / ".plan").write_text("## State\nbranch: main\nstate: drained\n")
    result = check_meta_consistency(str(tmp_path), str(tmp_path))
    assert "STATUS=ok" in result
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: `check_stale_scaffold_on_main` warns about `.plan` on main

- [ ] **Step 3: Update check_stale_scaffold_on_main**

In `project/work_health.py`, modify the function to read state before flagging `.plan`:

```python
def check_stale_scaffold_on_main(project, workspace):
    current, _ = _git(workspace, "branch", "--show-current")
    if current != "main":
        return "CHECK=stale_scaffold_on_main STATUS=ok"
    ws = Path(workspace)
    # A .plan with state: drained is intentional — not stale
    plan_path = ws / ".plan"
    if plan_path.exists():
        from lifecycle import read_state
        state = read_state(plan_path)
        if state in ("drained", "active"):
            # Intentional main .plan — skip it in the stale check
            stale = []
            for name in (".meta", ".epic", "JOURNAL.md", ".execute-progress"):
                if (ws / name).exists():
                    stale.append(name)
            if stale:
                return (f"CHECK=stale_scaffold_on_main STATUS=warn "
                        f"DETAIL=workspace main has stale scaffold: {', '.join(stale)} — run cleanup-scaffold")
            return "CHECK=stale_scaffold_on_main STATUS=ok"
    stale = []
    for name in (".plan", ".meta", ".epic", "JOURNAL.md", ".execute-progress"):
        if (ws / name).exists():
            stale.append(name)
    if stale:
        return (f"CHECK=stale_scaffold_on_main STATUS=warn "
                f"DETAIL=workspace main has stale scaffold: {', '.join(stale)} — run cleanup-scaffold")
    return "CHECK=stale_scaffold_on_main STATUS=ok"
```

- [ ] **Step 4: Update check_meta_consistency**

In `project/work_health.py`, modify line 93 to accept `branch: main` when on main:

```python
    if current == "main" and meta_branch == "main":
        return "CHECK=meta_consistency STATUS=ok"
    if current == "main" and meta_branch != "main":
        return (f"CHECK=meta_consistency STATUS=warn "
                f"DETAIL=.plan says branch '{meta_branch}' but on main — orphaned .plan")
```

- [ ] **Step 5: Fix hardcoded "main" in work_health.py**

Replace all `== "main"` and `!= "main"` comparisons with a `base_branch` parameter passed through the call chain from `run_checks()`. Add `base_branch="main"` parameter to `check_meta_consistency`, `check_stale_scaffold_on_main`, `check_dirty_main`, and update `run_checks` to accept and pass it.

- [ ] **Step 6: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: ALL PASS

- [ ] **Step 7: Run full test suite for regressions**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_work_chain.py -v`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_health.py tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): health checks accept drained .plan on main, fix hardcoded main"
```

---

## Batch 4: Skill integration

### Task 5: Update work/SKILL.md routing table

**Files:**
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: `CHAIN_DIRECTIVE` from ctx.py / work_chain.py output
- Produces: Updated routing table with `work find`, `drained` route, chaining directives

- [ ] **Step 1: Add `work find` to routing table (Step 1)**

In `work/SKILL.md`, add to the invocation table:

```markdown
| `work find` | → discover work and populate `.plan` queue (Step 6) |
```

- [ ] **Step 2: Replace Step 1c error handling with chaining**

Remove the existing Step 1c error table entries for `work continue` on main. Replace with:

```markdown
**Step 1c — Chaining (replaces error handling)**

Before dispatching, read `CHAIN_DIRECTIVE` from ctx.py output. Follow it:

| DIRECTIVE | Action |
|-----------|--------|
| `proceed` | Continue with the invoked command |
| `chain_to_next` | Redirect to `work next` (Step 5) |
| `chain_to_end` | Redirect to `work end` (invoke work-end) |
| `chain_to_find` | Redirect to `work find` (Step 6) |
| `guard_continue` | "Issue #N still open. Continue working." → run continue |
| `guard_next` | "Queue has remaining items. Advance or continue?" → present choice |
| `no_work_found` | "No work found." → stay in current state |
```

- [ ] **Step 3: Add `drained` route to Step 2**

Add to the routing table:

```markdown
| `drained` | → "Queue drained. Run `work find` to discover new work, or `work start #N` for a specific issue." |
```

- [ ] **Step 4: Add Step 6 — `work find`**

Add new section after Step 5:

```markdown
**Step 6 — `work find` (discover and populate queue)**

Discovers candidate work items and appends them to the `.plan` queue.

1. Run ctx.py. Read `CHAIN_DIRECTIVE` — if not `proceed`, follow the directive.
2. Refresh GitHub cache:
   ```bash
   python3 scripts/enrichment.py refresh --repo $OWNER_REPO
   ```
3. Query for recommendations:
   ```bash
   python3 scripts/enrichment.py what-next --repo $OWNER_REPO --mode general --limit 5
   ```
4. Also check HANDOFF.md What's Next section (existing Step 2a logic).
5. Present candidates. User selects items.
6. If items selected:
   - `plan_manager append` adds to queue
   - If state is `drained`: fire `work_find` transition (drained → transitioning)
   - Then `auto_refresh` (transitioning → active) with full context loading
7. If zero items selected: stay in current state. Output: "No work found."
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): work/SKILL.md — routing table, work find, chaining directives"
```

### Task 6: Update work-end/SKILL.md for main support

**Files:**
- Modify: `work-end/SKILL.md`

**Interfaces:**
- Consumes: `ON_MAIN` from ctx.py, `drained-sha` from `.plan`
- Produces: Main-mode work-end that skips branch-specific steps

- [ ] **Step 1: Remove branch-only precondition**

In frontmatter description, remove "Must be invoked from the working branch, not main." Replace with:

```markdown
Must be invoked from a branch with active work or from main with a .plan.
On main, skips branch-specific steps (merge, stamp, rebase, squash).
```

- [ ] **Step 2: Add main-mode conditional at Step 1**

After context checks, add:

```markdown
**Main-mode detection:**

Read `ON_MAIN` from ctx.py. If `ON_MAIN=yes`:
- Skip Step 4.2 (Rebase) — already on main
- Skip Step 4.3 (Squash) — no branch commits
- Step 4.4 (Land) → push only, no merge/stamp
- Step 5 (Verify) → check pushed, not merged/stamped
- Skip Step 6.2 (Checkout main) — already there
- Step 6.2b (Cleanup) → fire `cleanup_main` event (sets state to `drained`)
  instead of removing .plan

Textual guidance: "Consider a feature branch for non-trivial work."
```

- [ ] **Step 3: Add diff base for main-mode review**

In Step 2 (Review), add:

```markdown
**Main-mode diff base:** Read `drained-sha` from `.plan`'s `## State`.
Diff against this SHA. If no `drained-sha` exists (first close), diff
against `project-sha` from `.plan`. Command:
```bash
git -C "$PROJECT" diff <drained-sha>..HEAD
```
```

- [ ] **Step 4: Update lifecycle transitions section**

Add `cleanup_main` alongside `cleanup_pass`:

```markdown
- If `ON_MAIN=yes`: fire `cleanup_main` → state becomes `drained`
- If `ON_MAIN=no`: fire `cleanup_pass` → state becomes `idle` (unchanged)
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#261): work-end/SKILL.md — main support, conditional steps, drained state"
```

- [ ] **Step 6: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_work_chain.py -v`
Expected: ALL PASS — no regressions from skill changes

## References

- [2026-08-20-unified-work-queue-design.md] — design spec this plan implements
- [project/lifecycle.py:87-108] — transition table
- [project/lifecycle.py:110-133] — INVALID_MESSAGES
- [project/lifecycle.py:258-264] — worklog mapping
- [project/work_state.py:48-129] — detect() function, routing logic
- [project/work_health.py:69-98,165-177] — health check functions
- [project/ctx.py] — context output, BASE_BRANCH parsing
- [work/SKILL.md] — current routing table
- [work-end/SKILL.md] — current branch-only preconditions
- [ADR-0013] — unified .plan file design
- [#262] — corruption recovery (separate issue, not in this plan)
- [evidence-before-claims] — verification protocol
- [externalised-scripts-require-tests] — test requirement for new scripts

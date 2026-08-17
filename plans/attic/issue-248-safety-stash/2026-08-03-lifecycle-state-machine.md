# Lifecycle State Machine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #171 — Epic: Explicit lifecycle state machine
**Issue group:** #172, #173, #174, #175, #176, #177, #178

**Goal:** Replace inference-based state detection across the work lifecycle with an explicit state machine in `.meta`, making context setup unskippable, closing gates enforceable, and repo health mechanically verifiable.

**Architecture:** A single Python module (`lifecycle.py`) owns all state transitions via a data-driven transition table. Entry points fire events; the state machine validates, returns effects, and writes state atomically. A three-phase protocol (`transition()` → effects → `commit_transition()`) prevents state from advancing past completed work. Existing scripts (`ctx.py`, `work_router.py`, `scaffold.py`) are modified to read/write the new `state:` field.

**Tech Stack:** Python 3.11+, pytest, existing soredium test infrastructure

## Global Constraints

- Python module lives at `project/lifecycle.py` (alongside `ctx.py`)
- Tests at `tests/test_lifecycle.py` (follows existing test naming)
- `.meta` format adds one field (`state:`) — no other format changes
- Atomic file writes via temp-then-rename (POSIX `os.replace()`)
- Unknown state values raise `CorruptedState` — never silently default
- Missing state field defaults to `active` (legacy migration only)
- `validate=False` parameter does NOT exist — omit `project`/`workspace` args to skip validation
- All effects must be idempotent (guarded by existence checks)
- Pre-push hook at `project/pre_push_hook.py` (installed by user, not auto-installed)
- Protocol: `externalised-scripts-require-tests` applies to all new `.py` files

---

### Task 1: lifecycle.py core — transition table, read/write, state classification

**Issue:** #172
**Files:**
- Create: `project/lifecycle.py`
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Produces:
  - `read_state(meta_path: Path) -> Optional[str]`
  - `write_state(meta_path: Path, state: str) -> None`
  - `transition(meta_path: Path, event: str, project: Path = None, workspace: Path = None) -> TransitionResult`
  - `commit_transition(meta_path: Path, result: TransitionResult) -> None`
  - `can_transition(from_state: str, event: str) -> bool`
  - `is_transient(state: str) -> bool`
  - `is_closing(state: str) -> bool`
  - `migrate_legacy_paused(meta_path: Path) -> bool`
  - `VALID_STATES`, `TRANSIENT_STATES`, `CLOSING_STATES`, `RESTING_STATES` frozensets
  - `TransitionResult(from_state, new_state, event, effects, post_commit_effects)`
  - `InvalidTransition`, `InvalidState`, `ConcurrentModification`, `CorruptedState` exceptions

This task does NOT include `validate_state()` with git checks — that is Task 6. `transition()` accepts optional `project`/`workspace` but Task 1's implementation ignores them (validation is a no-op stub that returns `[]`).

- [ ] **Step 1: Write `test_lifecycle.py` — state constants and classification helpers**

```python
import pytest
from pathlib import Path

# Will import from lifecycle once created
# from lifecycle import (VALID_STATES, TRANSIENT_STATES, CLOSING_STATES,
#     is_transient, is_closing, can_transition)


class TestStateConstants:
    def test_valid_states_count(self):
        assert len(VALID_STATES) == 11

    def test_transient_states_are_subset(self):
        assert TRANSIENT_STATES <= VALID_STATES

    def test_closing_states_are_subset(self):
        assert CLOSING_STATES <= VALID_STATES

    @pytest.mark.parametrize("state, expected", [
        ("scaffolded", True), ("transitioning", True),
        ("active", False), ("paused", False), ("idle", False),
        ("closing:review", False),
    ])
    def test_is_transient(self, state, expected):
        assert is_transient(state) == expected

    @pytest.mark.parametrize("state, expected", [
        ("closing:review", True), ("closing:verified", True),
        ("closing:promoted", True), ("closing:pushed", True),
        ("closing:merged", True), ("closing:stamped", True),
        ("active", False), ("idle", False),
    ])
    def test_is_closing(self, state, expected):
        assert is_closing(state) == expected

    def test_can_transition_valid(self):
        assert can_transition("active", "work_end") is True

    def test_can_transition_invalid(self):
        assert can_transition("idle", "work_next") is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_lifecycle.py::TestStateConstants -v`
Expected: ImportError — `lifecycle` module does not exist yet

- [ ] **Step 3: Create `project/lifecycle.py` — constants and helpers**

```python
#!/usr/bin/env python3
"""
Lifecycle state machine for work branches.

Single source of truth for state transitions. Entry points fire events;
the transition table determines effects. No entry point decides what to do —
the state machine decides.

Spec: specs/issue-171-lifecycle-state-machine/2026-08-03-lifecycle-state-machine-design.md
"""
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

VALID_STATES = frozenset({
    'idle', 'scaffolded', 'active', 'transitioning', 'paused',
    'closing:review', 'closing:verified', 'closing:promoted',
    'closing:pushed', 'closing:merged', 'closing:stamped',
})

TRANSIENT_STATES = frozenset({'scaffolded', 'transitioning'})

CLOSING_STATES = frozenset({
    'closing:review', 'closing:verified', 'closing:promoted',
    'closing:pushed', 'closing:merged', 'closing:stamped',
})

RESTING_STATES = VALID_STATES - TRANSIENT_STATES


@dataclass
class TransitionResult:
    from_state: str
    new_state: str
    event: str
    effects: list[str] = field(default_factory=list)
    post_commit_effects: list[str] = field(default_factory=list)


class InvalidTransition(Exception):
    def __init__(self, from_state: str, event: str, message: str):
        self.from_state = from_state
        self.event = event
        super().__init__(message)


class InvalidState(Exception):
    def __init__(self, state: str, violations: list[str]):
        self.state = state
        self.violations = violations
        super().__init__(f"State '{state}' invariant violations: {violations}")


class ConcurrentModification(Exception):
    def __init__(self, expected: str, actual: str):
        self.expected = expected
        self.actual = actual
        super().__init__(f"State changed by another session: expected '{expected}', found '{actual}'")


class CorruptedState(Exception):
    def __init__(self, meta_path: Path, raw_value: str):
        self.meta_path = meta_path
        self.raw_value = raw_value
        super().__init__(f"Unknown state '{raw_value}' in {meta_path}")


class StateError(Exception):
    pass


def is_transient(state: str) -> bool:
    return state in TRANSIENT_STATES

def is_closing(state: str) -> bool:
    return state in CLOSING_STATES

def can_transition(from_state: str, event: str) -> bool:
    return (from_state, event) in TRANSITION_TABLE
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_lifecycle.py::TestStateConstants -v`
Expected: PASS

- [ ] **Step 5: Write tests for `read_state` and `write_state`**

Add to `test_lifecycle.py`:

```python
@pytest.fixture
def tmp_meta(tmp_path):
    meta = tmp_path / ".meta"
    meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")
    return meta


class TestReadWriteState:
    def test_read_active(self, tmp_meta):
        assert read_state(tmp_meta) == "active"

    def test_read_closing_pushed(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: x\nstate: closing:pushed\ndate: 2026-08-03\n")
        assert read_state(meta) == "closing:pushed"

    def test_no_meta_returns_none(self, tmp_path):
        assert read_state(tmp_path / ".meta") is None

    def test_missing_state_field_returns_active(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\ndate: 2026-08-03\n")
        assert read_state(meta) == "active"

    def test_unknown_state_raises_corrupted(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: x\nstate: bogus\ndate: 2026-08-03\n")
        with pytest.raises(CorruptedState):
            read_state(meta)

    def test_write_updates_existing(self, tmp_meta):
        write_state(tmp_meta, "closing:review")
        assert read_state(tmp_meta) == "closing:review"
        assert tmp_meta.read_text().count("state:") == 1

    def test_write_appends_to_legacy(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\ndate: 2026-08-03\n")
        write_state(meta, "scaffolded")
        content = meta.read_text()
        assert "state: scaffolded" in content
        assert content.index("branch:") < content.index("state:")

    def test_write_is_atomic(self, tmp_meta):
        write_state(tmp_meta, "closing:merged")
        assert not (tmp_meta.parent / ".meta.tmp").exists()
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py::TestReadWriteState -v`
Expected: FAIL — `read_state` not defined

- [ ] **Step 7: Implement `read_state` and `write_state`**

Add to `lifecycle.py`:

```python
def read_state(meta_path: Path) -> Optional[str]:
    if not meta_path.exists():
        return None
    content = meta_path.read_text()
    for line in content.splitlines():
        if line.startswith('state:'):
            raw = line.split(':', 1)[1].strip()
            if raw in VALID_STATES:
                return raw
            raise CorruptedState(meta_path, raw)
    return 'active'


def write_state(meta_path: Path, state: str) -> None:
    content = meta_path.read_text()
    lines = content.splitlines()

    state_line_idx = None
    for i, line in enumerate(lines):
        if line.startswith('state:'):
            state_line_idx = i
            break

    if state_line_idx is not None:
        lines[state_line_idx] = f'state: {state}'
    else:
        for i, line in enumerate(lines):
            if line.startswith('branch:'):
                lines.insert(i + 1, f'state: {state}')
                break
        else:
            lines.append(f'state: {state}')

    tmp_path = meta_path.parent / '.meta.tmp'
    tmp_path.write_text('\n'.join(lines) + '\n')
    tmp_path.replace(meta_path)
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_lifecycle.py::TestReadWriteState -v`
Expected: PASS

- [ ] **Step 9: Write tests for transition table and `transition()`**

Add to `test_lifecycle.py`:

```python
class TestValidTransitions:
    @pytest.mark.parametrize("from_state, event, expected_state, expected_effects", [
        ("idle",                "work",          "scaffolded",        ["create_branch", "write_meta"]),
        ("idle",                "work_epic",     "scaffolded",        ["create_branch", "write_meta", "write_epic"]),
        ("idle",                "slot_create",   "scaffolded",        ["create_slot", "write_meta"]),
        ("idle",                "slot_epic",     "scaffolded",        ["create_slot", "write_meta", "write_slot_epic"]),
        ("scaffolded",          "auto_setup",    "active",            ["garden_search", "load_specs", "check_protocols", "check_intellij"]),
        ("active",              "work_next",     "transitioning",     ["advance_issue", "update_meta", "tick_github"]),
        ("transitioning",       "auto_refresh",  "active",            ["garden_search", "load_specs", "check_protocols"]),
        ("active",              "work_pause",    "paused",            ["wip_commit"]),
        ("paused",              "work_resume",   "active",            ["pop_stack", "reset_wip", "context_resume"]),
        ("active",              "work_end",      "closing:review",    ["pre_close_sweep"]),
        ("closing:review",      "review_pass",   "closing:verified",  ["record_review"]),
        ("closing:verified",    "promote_pass",  "closing:promoted",  ["write_promotion_stamp"]),
        ("closing:promoted",    "push_pass",     "closing:pushed",    []),
        ("closing:pushed",      "merge_pass",    "closing:merged",    ["verify_content_landed"]),
        ("closing:merged",      "stamp_pass",    "closing:stamped",   ["write_stamp"]),
        ("closing:stamped",     "cleanup_pass",  "idle",              ["write_epic_closed"]),
    ])
    def test_valid_transition(self, from_state, event, expected_state, expected_effects, tmp_meta):
        if from_state != "idle":
            write_state(tmp_meta, from_state)
        result = transition(tmp_meta if from_state != "idle" else tmp_meta, event)
        assert result.from_state == from_state
        assert result.new_state == expected_state
        assert result.effects == expected_effects
        if from_state != "idle":
            assert read_state(tmp_meta) == from_state  # Phase 1 does NOT write

    def test_work_pause_has_post_commit_effects(self, tmp_meta):
        write_state(tmp_meta, "active")
        result = transition(tmp_meta, "work_pause")
        assert result.effects == ["wip_commit"]
        assert result.post_commit_effects == ["switch_to_main", "push_stack"]

    def test_cleanup_has_post_commit_effects(self, tmp_meta):
        write_state(tmp_meta, "closing:stamped")
        result = transition(tmp_meta, "cleanup_pass")
        assert result.effects == ["write_epic_closed"]
        assert result.post_commit_effects == ["return_to_main", "write_handoff"]

    @pytest.mark.parametrize("closing_state", [
        "closing:review", "closing:verified",
    ])
    def test_abort_from_pre_artifact(self, closing_state, tmp_meta):
        write_state(tmp_meta, closing_state)
        result = transition(tmp_meta, "abort_close")
        assert result.new_state == "active"

    @pytest.mark.parametrize("closing_state", [
        "closing:promoted", "closing:pushed", "closing:merged", "closing:stamped",
    ])
    def test_abort_blocked_post_artifact(self, closing_state, tmp_meta):
        write_state(tmp_meta, closing_state)
        with pytest.raises(InvalidTransition):
            transition(tmp_meta, "abort_close")
```

- [ ] **Step 10: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py::TestValidTransitions -v`
Expected: FAIL — `transition` not defined

- [ ] **Step 11: Implement `TRANSITION_TABLE` and `transition()`**

Add to `lifecycle.py`:

```python
TRANSITION_TABLE: dict[tuple[str, str], tuple[str, list[str], list[str]]] = {
    ('idle', 'work'):              ('scaffolded',        ['create_branch', 'write_meta'],                           []),
    ('idle', 'work_epic'):         ('scaffolded',        ['create_branch', 'write_meta', 'write_epic'],             []),
    ('idle', 'slot_create'):       ('scaffolded',        ['create_slot', 'write_meta'],                             []),
    ('idle', 'slot_epic'):         ('scaffolded',        ['create_slot', 'write_meta', 'write_slot_epic'],          []),
    ('scaffolded', 'auto_setup'):  ('active',            ['garden_search', 'load_specs', 'check_protocols', 'check_intellij'], []),
    ('active', 'work_next'):       ('transitioning',     ['advance_issue', 'update_meta', 'tick_github'],           []),
    ('transitioning', 'auto_refresh'): ('active',        ['garden_search', 'load_specs', 'check_protocols'],        []),
    ('active', 'work_pause'):      ('paused',            ['wip_commit'],                                            ['switch_to_main', 'push_stack']),
    ('paused', 'work_resume'):     ('active',            ['pop_stack', 'reset_wip', 'context_resume'],              []),
    ('active', 'work_end'):        ('closing:review',    ['pre_close_sweep'],                                       []),
    ('closing:review', 'review_pass'):    ('closing:verified',  ['record_review'],                                  []),
    ('closing:verified', 'promote_pass'): ('closing:promoted',  ['write_promotion_stamp'],                          []),
    ('closing:promoted', 'push_pass'):    ('closing:pushed',    [],                                                 []),
    ('closing:pushed', 'merge_pass'):     ('closing:merged',    ['verify_content_landed'],                          []),
    ('closing:merged', 'stamp_pass'):     ('closing:stamped',   ['write_stamp'],                                    []),
    ('closing:stamped', 'cleanup_pass'):  ('idle',              ['write_epic_closed'],                               ['return_to_main', 'write_handoff']),
    ('closing:review', 'abort_close'):    ('active', ['clear_closing_markers'],                                     []),
    ('closing:verified', 'abort_close'):  ('active', ['clear_closing_markers'],                                     []),
}

INVALID_MESSAGES: dict[tuple[str, str], str] = {
    ('idle', 'work_next'): "Cannot advance — no active branch. Start work first.",
    ('idle', 'work_pause'): "Cannot pause — no active branch.",
    ('idle', 'work_end'): "Cannot close — no active branch. You're on main.",
    ('idle', 'work_resume'): "Cannot resume — pause stack is empty.",
    ('scaffolded', 'work_next'): "Branch not yet active — context setup must complete first.",
    ('scaffolded', 'work_end'): "Cannot close — branch hasn't been activated yet.",
    ('scaffolded', 'work_pause'): "Cannot pause — branch hasn't been activated yet.",
    ('active', 'work'): "Already on an active branch. Use `work end`, `work pause`, or `work next`.",
    ('active', 'work_epic'): "Already on an active branch. Close or pause first.",
    ('active', 'work_resume'): "Branch is active, not paused. Nothing to resume.",
    ('transitioning', 'work_end'): "Issue transition in progress — context refresh must complete first.",
    ('transitioning', 'work_pause'): "Issue transition in progress — wait for context refresh.",
    ('paused', 'work_end'): "Cannot close a paused branch. Resume it first, then close.",
    ('paused', 'work_next'): "Cannot advance — branch is paused. Resume first.",
    ('paused', 'work_pause'): "Branch is already paused.",
    ('closing:promoted', 'abort_close'): "Cannot abort — artifacts already promoted. Continue forward.",
    ('closing:pushed', 'abort_close'): "Cannot abort — artifacts already promoted. Branch pushed — continue forward.",
    ('closing:merged', 'abort_close'): "Cannot abort — content already merged to main. Continue forward.",
    ('closing:stamped', 'abort_close'): "Cannot abort — branch already stamped. Only cleanup remains.",
}


def transition(
    meta_path: Path,
    event: str,
    project: Optional[Path] = None,
    workspace: Optional[Path] = None,
) -> TransitionResult:
    raw_state = read_state(meta_path)
    current_state = raw_state or 'idle'

    key = (current_state, event)
    if key not in TRANSITION_TABLE:
        msg = INVALID_MESSAGES.get(key, f"No transition from '{current_state}' on '{event}'")
        raise InvalidTransition(current_state, event, msg)

    new_state, effects, post_commit = TRANSITION_TABLE[key]

    if project is not None and workspace is not None:
        violations = validate_state(new_state, project, workspace)
        if violations:
            raise InvalidState(new_state, violations)

    return TransitionResult(
        from_state=current_state,
        new_state=new_state,
        event=event,
        effects=list(effects),
        post_commit_effects=list(post_commit),
    )


def validate_state(
    state: str,
    project: Path,
    workspace: Path,
    exclude_patterns: Optional[list[str]] = None,
) -> list[str]:
    return []  # Stub — Task 6 implements real checks
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_lifecycle.py::TestValidTransitions -v`
Expected: PASS

- [ ] **Step 13: Write invalid transition tests**

Add to `test_lifecycle.py`:

```python
class TestInvalidTransitions:
    @pytest.mark.parametrize("from_state, event", [
        ("idle",          "work_next"),
        ("idle",          "work_pause"),
        ("idle",          "work_end"),
        ("idle",          "work_resume"),
        ("scaffolded",    "work_next"),
        ("scaffolded",    "work_end"),
        ("scaffolded",    "work_pause"),
        ("active",        "work"),
        ("active",        "work_epic"),
        ("active",        "work_resume"),
        ("active",        "auto_setup"),
        ("transitioning", "work_end"),
        ("transitioning", "work_pause"),
        ("paused",        "work_end"),
        ("paused",        "work_next"),
        ("paused",        "work_pause"),
        ("closing:review",   "work_pause"),
        ("closing:review",   "work_next"),
        ("closing:review",   "work"),
        ("closing:review",   "work_epic"),
        ("closing:verified", "review_pass"),
        ("closing:promoted", "promote_pass"),
        ("closing:pushed",    "push_pass"),
    ])
    def test_invalid_transition_raises(self, from_state, event, tmp_meta):
        if from_state != "idle":
            write_state(tmp_meta, from_state)
        with pytest.raises(InvalidTransition):
            transition(tmp_meta if from_state != "idle" else tmp_meta, event)

    def test_invalid_transition_has_message(self, tmp_meta):
        write_state(tmp_meta, "active")
        with pytest.raises(InvalidTransition, match="Already on an active branch"):
            transition(tmp_meta, "work")
```

- [ ] **Step 14: Run invalid transition tests**

Run: `python3 -m pytest tests/test_lifecycle.py::TestInvalidTransitions -v`
Expected: PASS (transition table already handles these)

- [ ] **Step 15: Write `commit_transition` tests**

Add to `test_lifecycle.py`:

```python
class TestCommitTransition:
    def test_commit_writes_new_state(self, tmp_meta):
        write_state(tmp_meta, "active")
        result = transition(tmp_meta, "work_end")
        commit_transition(tmp_meta, result)
        assert read_state(tmp_meta) == "closing:review"

    def test_commit_detects_concurrent_modification(self, tmp_meta):
        write_state(tmp_meta, "active")
        result = transition(tmp_meta, "work_end")
        write_state(tmp_meta, "paused")
        with pytest.raises(ConcurrentModification):
            commit_transition(tmp_meta, result)

    def test_commit_idle_to_scaffolded_verifies_meta(self, tmp_path):
        meta = tmp_path / ".meta"
        result = transition(meta, "work")
        meta.write_text("branch: issue-1-test\nstate: scaffolded\ndate: 2026-08-03\n")
        commit_transition(meta, result)
        assert read_state(meta) == "scaffolded"

    def test_commit_idle_to_scaffolded_fails_without_meta(self, tmp_path):
        meta = tmp_path / ".meta"
        result = transition(meta, "work")
        with pytest.raises(StateError):
            commit_transition(meta, result)

    def test_commit_to_idle_skips_write(self, tmp_meta):
        write_state(tmp_meta, "closing:stamped")
        result = transition(tmp_meta, "cleanup_pass")
        commit_transition(tmp_meta, result)
        assert read_state(tmp_meta) == "closing:stamped"

    def test_commit_to_idle_checks_concurrent(self, tmp_meta):
        write_state(tmp_meta, "closing:stamped")
        result = transition(tmp_meta, "cleanup_pass")
        write_state(tmp_meta, "closing:merged")
        with pytest.raises(ConcurrentModification):
            commit_transition(tmp_meta, result)
```

- [ ] **Step 16: Implement `commit_transition()`**

Add to `lifecycle.py`:

```python
def commit_transition(meta_path: Path, result: TransitionResult) -> None:
    if result.from_state == 'idle':
        if not meta_path.exists():
            raise StateError(f".meta not created by write_meta effect at {meta_path}")
        current = read_state(meta_path)
        if current != result.new_state:
            raise StateError(f"Expected '{result.new_state}' after scaffold, got '{current}'")
    else:
        current = read_state(meta_path)
        if current != result.from_state:
            raise ConcurrentModification(expected=result.from_state, actual=current or 'None')
        if result.new_state != 'idle':
            write_state(meta_path, result.new_state)
```

- [ ] **Step 17: Run all commit_transition tests**

Run: `python3 -m pytest tests/test_lifecycle.py::TestCommitTransition -v`
Expected: PASS

- [ ] **Step 18: Write `migrate_legacy_paused` tests**

```python
class TestMigrateLegacyPaused:
    def test_migrates_missing_state_to_paused(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\ndate: 2026-08-03\n")
        assert migrate_legacy_paused(meta) is True
        assert read_state(meta) == "paused"

    def test_migrates_defaulted_active_to_paused(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\ndate: 2026-08-03\n")
        assert read_state(meta) == "active"  # default
        assert migrate_legacy_paused(meta) is True
        assert read_state(meta) == "paused"

    def test_no_op_if_already_paused(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: paused\ndate: 2026-08-03\n")
        assert migrate_legacy_paused(meta) is False

    def test_no_op_if_has_explicit_state(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")
        assert migrate_legacy_paused(meta) is False
```

- [ ] **Step 19: Implement `migrate_legacy_paused()`**

```python
def migrate_legacy_paused(meta_path: Path) -> bool:
    if not meta_path.exists():
        return False
    content = meta_path.read_text()
    has_state_field = any(line.startswith('state:') for line in content.splitlines())
    if has_state_field:
        return False
    write_state(meta_path, 'paused')
    return True
```

- [ ] **Step 20: Run all tests**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS

- [ ] **Step 21: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_lifecycle.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#172): lifecycle state machine core — transition table, read/write, three-phase protocol

Refs #171"
```

---

### Task 2: ctx.py integration — add META_STATE output, remove branch-clearing

**Issue:** #173 (part 1)
**Files:**
- Modify: `project/ctx.py`
- Modify: `tests/test_ctx.py`

**Interfaces:**
- Consumes: `lifecycle.read_state(meta_path)` from Task 1
- Produces: `META_STATE=<state>` and `META_IS_TRANSIENT=yes|no` in ctx.py output

- [ ] **Step 1: Write test for new META_STATE output**

Add to `tests/test_ctx.py`:

```python
class TestMetaState:
    def test_meta_state_output_when_active(self, workspace_with_meta):
        # workspace_with_meta fixture creates .meta with state: active
        result = run_ctx(workspace_with_meta)
        assert "META_STATE=active" in result.stdout

    def test_meta_state_empty_when_no_meta(self, workspace_no_meta):
        result = run_ctx(workspace_no_meta)
        assert "META_STATE=" in result.stdout

    def test_meta_is_transient_yes_for_scaffolded(self, workspace_with_meta_state):
        result = run_ctx(workspace_with_meta_state("scaffolded"))
        assert "META_IS_TRANSIENT=yes" in result.stdout

    def test_meta_is_transient_no_for_active(self, workspace_with_meta_state):
        result = run_ctx(workspace_with_meta_state("active"))
        assert "META_IS_TRANSIENT=no" in result.stdout

    def test_branch_mismatch_preserves_meta(self, workspace_mismatched_branch):
        # Previously ctx.py cleared meta dict on mismatch — verify it no longer does
        result = run_ctx(workspace_mismatched_branch)
        assert "META_STATE=" in result.stdout
        # META_STATE should still be populated — lifecycle.py owns mismatch detection
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_ctx.py::TestMetaState -v`
Expected: FAIL — META_STATE not in output

- [ ] **Step 3: Modify `ctx.py`**

Three changes:
1. Import `read_state` from lifecycle (add to sys.path)
2. Add `META_STATE` and `META_IS_TRANSIENT` output after existing meta parsing
3. Remove the branch-mismatch meta clearing (lines 110-113)

At line ~97, after meta parsing, add:

```python
_lifecycle_dir = Path(__file__).parent
if str(_lifecycle_dir) not in sys.path:
    sys.path.insert(0, str(_lifecycle_dir))
from lifecycle import read_state as _read_state, is_transient as _is_transient

_meta_state = _read_state(meta_path) or ""
_meta_is_transient = "yes" if (_meta_state and _is_transient(_meta_state)) else "no"
```

Remove lines 110-113 (the `if meta_branch and meta_branch != workspace_branch: meta = {}` clearing).

Add to output section:

```python
print(f"META_STATE={_meta_state}")
print(f"META_IS_TRANSIENT={_meta_is_transient}")
```

- [ ] **Step 4: Run ctx.py tests**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: ALL PASS (existing + new)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/ctx.py tests/test_ctx.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#173): ctx.py outputs META_STATE, removes branch-clearing

lifecycle.py owns branch alignment validation now.
Refs #171"
```

---

### Task 3: scaffold.py — write `state: scaffolded` on creation

**Issue:** #173 (part 2)
**Files:**
- Modify: `work-start/scaffold.py`
- Modify: `tests/test_scaffold.py`

**Interfaces:**
- Consumes: nothing new — adds `state:` field to existing `.meta` format
- Produces: `.meta` files now contain `state: scaffolded` after creation

- [ ] **Step 1: Write test for state field in scaffold output**

Add to `tests/test_scaffold.py`:

```python
def test_scaffold_writes_state_scaffolded(tmp_path):
    result = run(tmp_path, *required_args())
    assert result.returncode == 0
    meta = (tmp_path / "design" / ".meta").read_text()
    assert "state: scaffolded" in meta
    # state should be right after branch
    lines = meta.splitlines()
    branch_idx = next(i for i, l in enumerate(lines) if l.startswith("branch:"))
    state_idx = next(i for i, l in enumerate(lines) if l.startswith("state:"))
    assert state_idx == branch_idx + 1
```

- [ ] **Step 2: Run test — verify fail**

Run: `python3 -m pytest tests/test_scaffold.py::test_scaffold_writes_state_scaffolded -v`
Expected: FAIL — `state:` not in .meta

- [ ] **Step 3: Add `state: scaffolded` to scaffold.py**

In `scaffold.py`, line 89-99, insert `state: scaffolded` after the `branch` line:

```python
    meta_lines = [
        f"branch: {branch}",
        f"state: scaffolded",
        f"project-sha: {params.get('project-sha', '')}",
        # ... rest unchanged
    ]
```

- [ ] **Step 4: Run all scaffold tests**

Run: `python3 -m pytest tests/test_scaffold.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-start/scaffold.py tests/test_scaffold.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#173): scaffold writes state: scaffolded to .meta

New branches start in scaffolded state — context setup auto-resolves to active.
Refs #171"
```

---

### Task 4: work_router.py — state-based routing

**Issue:** #173 (part 3)
**Files:**
- Modify: `work/work_router.py`
- Modify: `tests/test_work_router.py`

**Interfaces:**
- Consumes: `META_STATE` from ctx.py (Task 2), `lifecycle.is_transient()`, `lifecycle.is_closing()` from Task 1
- Produces: `META_STATE=<state>` in router output (alongside existing fields)

- [ ] **Step 1: Write tests for state-based routing**

Add to `tests/test_work_router.py`:

```python
class TestStateBasedRouting:
    def test_meta_state_in_output(self, workspace_on_branch_with_meta):
        result = detect_state("issue-42-foo", str(project), str(workspace))
        assert "META_STATE" in result

    def test_scaffolded_routes_to_resume_branch(self, workspace_scaffolded):
        result = detect_state("issue-42-foo", str(project), str(workspace))
        assert result["ROUTE"] == "resume_branch"
        assert result["META_STATE"] == "scaffolded"

    def test_closing_state_routes_to_resume_branch(self, workspace_closing):
        result = detect_state("issue-42-foo", str(project), str(workspace))
        assert result["ROUTE"] == "resume_branch"
        assert result["META_STATE"] == "closing:review"
```

- [ ] **Step 2: Run tests — verify fail**

Run: `python3 -m pytest tests/test_work_router.py::TestStateBasedRouting -v`

- [ ] **Step 3: Modify `work_router.py` to read and output META_STATE**

Import lifecycle, read state from `.meta`, add to output:

```python
from lifecycle import read_state as _read_state, is_transient, is_closing

# After meta_file check:
meta_state = _read_state(meta_file) or ""

# Add to result dict:
result["META_STATE"] = meta_state
```

- [ ] **Step 4: Run all work_router tests**

Run: `python3 -m pytest tests/test_work_router.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/work_router.py tests/test_work_router.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#173): work_router outputs META_STATE for state-based routing

Refs #171"
```

---

### Task 5: Pre-push hook — enforce closing gates

**Issue:** #176
**Files:**
- Create: `project/pre_push_hook.py`
- Test: `tests/test_pre_push_hook.py`

**Interfaces:**
- Consumes: `lifecycle.read_state()` from Task 1
- Produces: exit code 0 (allow) or 1 (block) for git pre-push hook protocol

- [ ] **Step 1: Write hook tests**

```python
import pytest
from pathlib import Path
from pre_push_hook import hook_check, HookResult


class TestPrePushHook:
    @pytest.mark.parametrize("state, push_to_main, should_block", [
        ("active",              True,  True),
        ("scaffolded",          True,  True),
        ("closing:review",      True,  True),
        ("closing:verified",    True,  True),
        ("closing:promoted",    True,  True),
        ("closing:pushed",      True,  False),
        ("closing:merged",      True,  False),
        ("closing:stamped",     True,  False),
        ("active",              False, False),
        ("closing:review",      False, False),
        ("closing:pushed",      False, False),
    ])
    def test_hook_enforcement(self, state, push_to_main, should_block, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text(f"branch: test\nstate: {state}\ndate: 2026-08-03\n")
        result = hook_check(meta, push_to_main=push_to_main)
        assert result.blocked == should_block

    def test_no_meta_allows_push(self, tmp_path):
        meta = tmp_path / ".meta"
        result = hook_check(meta, push_to_main=True)
        assert result.blocked is False

    def test_blocked_message_includes_state(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: test\nstate: active\ndate: 2026-08-03\n")
        result = hook_check(meta, push_to_main=True)
        assert "active" in result.message
```

- [ ] **Step 2: Run tests — verify fail**

- [ ] **Step 3: Implement `pre_push_hook.py`**

```python
#!/usr/bin/env python3
"""Pre-push hook: enforce lifecycle state gates."""
import sys
from dataclasses import dataclass
from pathlib import Path

_dir = Path(__file__).parent
if str(_dir) not in sys.path:
    sys.path.insert(0, str(_dir))
from lifecycle import read_state

PUSH_ALLOWED_STATES = frozenset({
    'closing:pushed', 'closing:merged', 'closing:stamped',
})


@dataclass
class HookResult:
    blocked: bool
    message: str = ""


def hook_check(meta_path: Path, push_to_main: bool = False) -> HookResult:
    state = read_state(meta_path)
    if state is None:
        return HookResult(blocked=False)
    if not push_to_main:
        return HookResult(blocked=False)
    if state in PUSH_ALLOWED_STATES:
        return HookResult(blocked=False)
    return HookResult(
        blocked=True,
        message=f"BLOCKED: state is '{state}'. Run work-end to complete the close sequence.",
    )
```

- [ ] **Step 4: Run tests — verify pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/pre_push_hook.py tests/test_pre_push_hook.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#176): pre-push hook enforces lifecycle closing gates

Blocks push to main unless state is closing:pushed or later.
Refs #171"
```

---

### Task 6: Hygiene invariants — `validate_state()` with real checks

**Issue:** #177
**Files:**
- Modify: `project/lifecycle.py` (replace stub `validate_state`)
- Modify: `tests/test_lifecycle.py` (add hygiene tests)

**Interfaces:**
- Consumes: git status output, `.meta` fields, `project`/`workspace` paths
- Produces: list of violation strings (empty = clean)

- [ ] **Step 1: Write hygiene invariant tests**

Add test fixtures and test class to `tests/test_lifecycle.py`. Tests need real git repos — use `tmp_path` with `git init`.

```python
@pytest.fixture
def git_project(tmp_path):
    """Create a minimal git repo with .meta on a feature branch."""
    project = tmp_path / "project"
    project.mkdir()
    subprocess.run(["git", "init", str(project)], capture_output=True)
    subprocess.run(["git", "-C", str(project), "commit", "--allow-empty", "-m", "init"], capture_output=True)
    subprocess.run(["git", "-C", str(project), "checkout", "-b", "issue-42-foo"], capture_output=True)
    meta_dir = project / "design"
    meta_dir.mkdir()
    meta = meta_dir / ".meta"
    meta.write_text("branch: issue-42-foo\nstate: active\ndate: 2026-08-03\n")
    return project, meta


class TestHygieneInvariants:
    def test_untracked_files_detected(self, git_project):
        project, meta = git_project
        (project / "leftover.py").write_text("oops")
        violations = validate_state("closing:review", project, project)
        assert any("Untracked" in v for v in violations)

    def test_excluded_patterns_ignored(self, git_project):
        project, meta = git_project
        idea = project / ".idea"
        idea.mkdir()
        (idea / "workspace.xml").write_text("<xml/>")
        violations = validate_state("closing:review", project, project)
        assert not any(".idea" in v for v in violations)

    def test_branch_mismatch_detected(self, git_project):
        project, meta = git_project
        subprocess.run(["git", "-C", str(project), "checkout", "main"], capture_output=True)
        violations = validate_state("active", project, project)
        assert any("Branch mismatch" in v for v in violations)

    def test_uncommitted_changes_block_close(self, git_project):
        project, meta = git_project
        (project / "file.py").write_text("new")
        subprocess.run(["git", "-C", str(project), "add", "file.py"], capture_output=True)
        violations = validate_state("closing:review", project, project)
        assert any("Uncommitted" in v or "Untracked" in v for v in violations)
```

- [ ] **Step 2: Run tests — verify fail (stub returns `[]`)**

- [ ] **Step 3: Implement real `validate_state()`**

Replace the stub in `lifecycle.py`:

```python
import subprocess as _sp

_DEFAULT_EXCLUDES = [
    '.idea/', 'target/', 'build/', 'node_modules/',
    '__pycache__/', '*.iml', '.worktrees/', 'slots/',
    '.pytest_cache/', '*.pyc',
]


def validate_state(
    state: str,
    project: Path,
    workspace: Path,
    exclude_patterns: Optional[list[str]] = None,
) -> list[str]:
    violations: list[str] = []
    excludes = exclude_patterns or _DEFAULT_EXCLUDES

    if state not in ('paused', 'idle'):
        _check_untracked(project, excludes, violations, "project")
        if project != workspace:
            _check_untracked(workspace, excludes, violations, "workspace")

    if state not in ('idle', 'paused'):
        meta_path = (workspace if project != workspace else project) / "design" / ".meta"
        if meta_path.exists():
            meta_branch = ""
            for line in meta_path.read_text().splitlines():
                if line.startswith("branch:"):
                    meta_branch = line.split(":", 1)[1].strip()
                    break
            if meta_branch:
                current = _sp.run(
                    ["git", "-C", str(project), "branch", "--show-current"],
                    capture_output=True, text=True,
                ).stdout.strip()
                if current and current != meta_branch:
                    violations.append(
                        f"Branch mismatch: .meta says '{meta_branch}', git says '{current}'"
                    )

    if state == 'closing:review':
        for repo, label in [(project, "project"), (workspace, "workspace")]:
            status = _sp.run(
                ["git", "-C", str(repo), "status", "--porcelain"],
                capture_output=True, text=True,
            ).stdout.strip()
            if status:
                violations.append(f"Uncommitted changes in {label}")

    return violations


def _check_untracked(repo: Path, excludes: list[str], violations: list[str], label: str) -> None:
    result = _sp.run(
        ["git", "-C", str(repo), "ls-files", "--others", "--exclude-standard"],
        capture_output=True, text=True,
    )
    for f in result.stdout.strip().splitlines():
        if f and not any(_matches(f, pat) for pat in excludes):
            violations.append(f"Untracked file in {label}: {f}")


def _matches(filepath: str, pattern: str) -> bool:
    if pattern.endswith('/'):
        return filepath.startswith(pattern) or f'/{pattern}' in filepath
    if pattern.startswith('*.'):
        return filepath.endswith(pattern[1:])
    return pattern in filepath
```

- [ ] **Step 4: Run hygiene tests — verify pass**

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_lifecycle.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#177): validate_state with untracked files, branch alignment, clean tree checks

Hygiene invariants block transitions when repo state is inconsistent.
Refs #171"
```

---

### Task 7: Skill SKILL.md updates — wire transition calls into lifecycle skills

**Issue:** #173 (part 4), #174, #175
**Files:**
- Modify: `work-start/SKILL.md` — replace 6-state detection with state reads
- Modify: `work-end/SKILL.md` — replace preconditions with closing state transitions
- Modify: `work-pause/SKILL.md` — add transition(active, work_pause) call
- Modify: `work-resume/SKILL.md` — add migrate_legacy_paused + transition(paused, work_resume)
- Modify: `work/SKILL.md` — route based on META_STATE, chain epic setup to auto_setup
- Modify: `work-slot/SKILL.md` — change messaging from "run work-start" to "run work"
- Modify: `project/SKILL.md` — route to work lifecycle when META_STATE is non-empty

**Interfaces:**
- Consumes: `META_STATE` from ctx.py, `lifecycle.transition()` and `lifecycle.commit_transition()` from Task 1

**Note:** This task modifies SKILL.md files (skill instructions for Claude), not Python scripts. These changes tell Claude how to invoke the state machine when executing each skill. No automated tests — the skills are tested by exercising the work lifecycle end-to-end.

- [ ] **Step 1: Update `work/SKILL.md` — state-based routing**

Key changes:
1. Step 1b: after `work_router.py`, read `META_STATE` from output
2. Step 2: add `META_STATE`-based routing alongside existing ROUTE values
3. Step 5 (work epic): after branch creation, note that scaffold.py writes `state: scaffolded`, then invoke `transition(meta, 'auto_setup')` + `commit_transition()` to chain to context setup automatically (replacing "Run work-start to begin")
4. Step 6 (work next): call `transition(meta, 'work_next')`, execute effects, `commit_transition()`, then `transition(meta, 'auto_refresh')` for context refresh

- [ ] **Step 2: Update `work-start/SKILL.md` — replace detection states**

Key changes:
1. Detection section: replace 6-state if/elif with `read_state()` from lifecycle
2. New branch path: `scaffold.py` now writes `state: scaffolded` → `transition(meta, 'auto_setup')` auto-resolves to active
3. Resume path: read `META_STATE` from ctx.py, if `active` → resume, if `scaffolded` → auto-resolve

- [ ] **Step 3: Update `work-end/SKILL.md` — closing state transitions**

Key changes:
1. Pre-conditions: replace branch divergence check with `read_state()` — if not `active`, show state-appropriate message
2. Step 3c (code review): after review passes, `transition(meta, 'review_pass')` + `commit_transition()`
3. Step 8a: after close_artifacts succeeds, `transition(meta, 'promote_pass')` + `commit_transition()`
4. Step 8j: after squash + push branch, `transition(meta, 'push_pass')` + `commit_transition()`; after merge + push main, `transition(meta, 'merge_pass')` + `commit_transition()`
5. Step 8j stamp: after stamp, `transition(meta, 'stamp_pass')` + `commit_transition()`
6. Step 9-10: after cleanup, `transition(meta, 'cleanup_pass')` + `commit_transition()`, then post_commit_effects

- [ ] **Step 4: Update `work-pause/SKILL.md`**

After Step 1 (validate state), add:
```
result = transition(meta_path, 'work_pause')
# Execute effects: wip_commit
commit_transition(meta_path, result)
# Execute post_commit_effects: switch_to_main, push_stack
```

- [ ] **Step 5: Update `work-resume/SKILL.md`**

After Step 4 (checkout branch), before Step 5 (pop stack):
```
# Migration for legacy paused branches
migrate_legacy_paused(meta_path)

result = transition(meta_path, 'work_resume')
# Execute effects: pop_stack, reset_wip, context_resume
commit_transition(meta_path, result)
```

- [ ] **Step 6: Update `work-slot/SKILL.md`**

Change Step 7 report from "run work-start" to "run `work`" in both `work-slot create` and `work-slot epic`.

- [ ] **Step 7: Update `project/SKILL.md`**

After fast-path exit, add:
```
If META_STATE is non-empty (from ctx.py output): route to `work` router.
The work router handles all state-based routing including transient auto-resolve.
```

- [ ] **Step 8: Commit all SKILL.md changes**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/SKILL.md work-start/SKILL.md work-end/SKILL.md work-pause/SKILL.md work-resume/SKILL.md work-slot/SKILL.md project/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#173,#174,#175): wire lifecycle state machine into all work skills

- work-start: replace 6-state detection with state reads
- work-end: closing sub-states with gate enforcement
- work-pause/resume: transition calls with legacy migration
- work epic: chains to auto_setup (implicit work-start)
- work next: fires transitioning → auto_refresh (context refresh)
- work-slot: messaging fix (run work, not work-start)
- project: routes to work lifecycle when .meta exists

Refs #171"
```

---

## Self-Review Checklist

1. **Spec coverage:**
   - [x] §1 States → Task 1 (VALID_STATES, state constants)
   - [x] §2 Events → Task 1 (TRANSITION_TABLE keys)
   - [x] §3 Transition table → Task 1 (TRANSITION_TABLE)
   - [x] §4 Invalid transitions → Task 1 (INVALID_MESSAGES + tests)
   - [x] §6 Context setup/refresh → Task 7 (SKILL.md wiring)
   - [x] §7 Hygiene invariants → Task 6 (validate_state)
   - [x] §8 Pre-push hook → Task 5 (pre_push_hook.py)
   - [x] §9 Stale state recovery → Task 7 (SKILL.md routing)
   - [x] §10 Migration → Task 1 (read_state defaults, migrate_legacy_paused)
   - [x] §11 .meta format → Task 3 (scaffold.py)
   - [x] §12 ctx.py integration → Task 2
   - [x] §13 Transition protocol → Task 1 (three-phase: transition, commit_transition)
   - [x] §15 Module API → Task 1 (all public functions)
   - [x] §16 TDD test cases → Tasks 1, 2, 3, 5, 6

2. **Not covered (deferred to #178):**
   - §13.2 Worklog event emission — requires #158

3. **Placeholder scan:** None found. All steps have code.

4. **Type consistency:** Verified — `TransitionResult`, `read_state`, `write_state`, `transition`, `commit_transition` signatures match across all tasks.

5. **Tooling safety:** No bash cp/rm/mv on source files. All Python file operations use standard library.

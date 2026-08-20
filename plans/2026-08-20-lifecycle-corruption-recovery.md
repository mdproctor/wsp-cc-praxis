# Lifecycle Corruption Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #262 — Lifecycle corruption recovery: Python-based state triage
**Issue group:** #262, #263

**Goal:** Detect inconsistent lifecycle states and present structured
recovery directives instead of crashing or silently defaulting.

**Architecture:** New `project/corruption.py` module with pure detection
functions that check state-vs-environment coherence. Called by `ctx.py`
at session start. Returns `Finding` dataclasses serialised as KEY=VALUE
lines. `work_health.py` delegates overlapping checks to corruption.py
and gets `base_branch` threading (#263).

**Tech Stack:** Python 3, pytest, subprocess (git CLI), gh CLI (GitHub)

## Global Constraints

- All detection functions are pure — no state mutation (D2)
- Each `check_*` function returns `Optional[Finding]` — never raises
- `diagnose()` catches all exceptions internally — never crashes ctx.py
- `base_branch` parameter everywhere — never compare against literal `"main"`
- Tests use `tmp_path` fixtures and `monkeypatch` for subprocess mocking
- Follow existing `_write_plan()` helper pattern from `tests/test_lifecycle.py`
- Follow existing `_init_repo()` pattern from `tests/test_work_health.py`

---

## Batch 1: Corruption detection module

### Task 1: Finding dataclass + file-level checks (S1, S2)

**Files:**
- Create: `project/corruption.py`
- Create: `tests/test_corruption.py`

**Interfaces:**
- Consumes: `lifecycle.VALID_STATES` (frozenset of valid state strings)
- Produces: `Finding` dataclass, `check_missing_state(plan_path) -> Optional[Finding]`, `check_invalid_state(meta_state, plan_path) -> Optional[Finding]`

- [ ] **Step 1: Write failing tests for Finding dataclass and S1**

Create `tests/test_corruption.py`:

```python
#!/usr/bin/env python3
"""Tests for project/corruption.py — lifecycle corruption detection."""

import subprocess
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


def _write_plan(path: Path, state: str = "active", branch: str = "issue-42-foo", **extra):
    defaults = {"date": "2026-08-20", "issue-repo": "Hortora/soredium", "covers": "42"}
    defaults.update(extra)
    lines = ["# Work Plan — test", "", "## State",
             f"branch: {branch}", f"state: {state}"]
    for k, v in defaults.items():
        lines.append(f"{k}: {v}")
    lines.extend(["", "## Queue", "- [ ] #42 — Fix foo ← active", ""])
    path.write_text("\n".join(lines))


def _write_plan_no_state(path: Path, branch: str = "issue-42-foo"):
    lines = ["# Work Plan — test", "", "## State",
             f"branch: {branch}", "date: 2026-08-20",
             "", "## Queue", "- [ ] #42 — Fix foo ← active", ""]
    path.write_text("\n".join(lines))


class TestFinding:
    def test_finding_has_required_fields(self):
        from corruption import Finding
        f = Finding(
            scenario="S1_MISSING_STATE",
            severity="warning",
            detail="test detail",
            actions=["accept_default"],
        )
        assert f.scenario == "S1_MISSING_STATE"
        assert f.severity == "warning"
        assert f.detail == "test detail"
        assert f.actions == ["accept_default"]


class TestS1MissingState:
    def test_missing_state_field_returns_warning(self, tmp_path):
        from corruption import check_missing_state
        plan = tmp_path / ".plan"
        _write_plan_no_state(plan)
        finding = check_missing_state(plan)
        assert finding is not None
        assert finding.scenario == "S1_MISSING_STATE"
        assert finding.severity == "warning"
        assert "accept_default" in finding.actions

    def test_present_state_field_returns_none(self, tmp_path):
        from corruption import check_missing_state
        plan = tmp_path / ".plan"
        _write_plan(plan)
        assert check_missing_state(plan) is None

    def test_no_plan_returns_none(self, tmp_path):
        from corruption import check_missing_state
        assert check_missing_state(tmp_path / ".plan") is None


class TestS2InvalidState:
    def test_corrupted_prefix_returns_error(self, tmp_path):
        from corruption import check_invalid_state
        plan = tmp_path / ".plan"
        _write_plan(plan, state="closing:pro")
        finding = check_invalid_state("corrupted:closing:pro", plan)
        assert finding is not None
        assert finding.scenario == "S2_INVALID_STATE"
        assert finding.severity == "error"
        assert "write_active" in finding.actions
        assert "remove_plan" in finding.actions

    def test_valid_state_returns_none(self, tmp_path):
        from corruption import check_invalid_state
        plan = tmp_path / ".plan"
        _write_plan(plan)
        assert check_invalid_state("active", plan) is None

    def test_empty_state_returns_none(self, tmp_path):
        from corruption import check_invalid_state
        plan = tmp_path / ".plan"
        _write_plan(plan)
        assert check_invalid_state("", plan) is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_corruption.py::TestFinding tests/test_corruption.py::TestS1MissingState tests/test_corruption.py::TestS2InvalidState -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'corruption'`

- [ ] **Step 3: Implement Finding dataclass and S1/S2 checks**

Create `project/corruption.py`:

```python
#!/usr/bin/env python3
"""Lifecycle corruption detection.

Checks state-vs-environment coherence. Called by ctx.py at session start.
Returns Finding objects — never mutates state, never raises.

Spec: issue-262-lifecycle-corruption-recovery
"""
from __future__ import annotations

import subprocess
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

import sys
_project_dir = Path(__file__).parent
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))

from lifecycle import VALID_STATES


@dataclass
class Finding:
    scenario: str
    severity: str
    detail: str
    actions: list[str] = field(default_factory=list)


def _git(repo: Path, *args: str, timeout: int = 10) -> tuple[str, int]:
    try:
        result = subprocess.run(
            ["git", "-C", str(repo)] + list(args),
            capture_output=True, text=True, timeout=timeout,
        )
        return result.stdout.strip(), result.returncode
    except subprocess.TimeoutExpired:
        return "", 1


def _read_plan_field(plan_path: Path, field_name: str) -> Optional[str]:
    if not plan_path.exists():
        return None
    in_state = False
    has_sections = False
    for line in plan_path.read_text().splitlines():
        if line.strip() == "## State":
            in_state = True
            has_sections = True
            continue
        if line.startswith("## "):
            in_state = False
            continue
        should_check = in_state if has_sections else True
        if should_check and line.startswith(f"{field_name}:"):
            return line.split(":", 1)[1].strip()
    return None


def check_missing_state(plan_path: Path) -> Optional[Finding]:
    if not plan_path.exists():
        return None
    in_state = False
    has_sections = False
    for line in plan_path.read_text().splitlines():
        if line.strip() == "## State":
            in_state = True
            has_sections = True
            continue
        if line.startswith("## "):
            in_state = False
            continue
        should_check = in_state if has_sections else True
        if should_check and line.startswith("state:"):
            return None
    if not has_sections:
        return None
    return Finding(
        scenario="S1_MISSING_STATE",
        severity="warning",
        detail="state: field missing from .plan — defaulted to 'active' (legacy migration)",
        actions=["accept_default", "write_scaffolded"],
    )


def check_invalid_state(meta_state: str, plan_path: Path) -> Optional[Finding]:
    if not meta_state or not meta_state.startswith("corrupted:"):
        return None
    raw_value = meta_state[len("corrupted:"):]
    return Finding(
        scenario="S2_INVALID_STATE",
        severity="error",
        detail=f"Unknown state '{raw_value}' in .plan — file may be truncated or hand-edited",
        actions=["write_active", "infer_from_environment", "remove_plan"],
    )
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_corruption.py::TestFinding tests/test_corruption.py::TestS1MissingState tests/test_corruption.py::TestS2InvalidState -v`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/corruption.py tests/test_corruption.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#262): Finding dataclass + S1/S2 corruption checks

Refs #262"
```

---

### Task 2: Git-local checks (S4, S5, S6, S7)

**Files:**
- Modify: `project/corruption.py`
- Modify: `tests/test_corruption.py`

**Interfaces:**
- Consumes: `Finding`, `_git()`, `_read_plan_field()` from Task 1
- Produces: `check_closing_postconditions()`, `check_branch_mismatch()`, `check_branch_exists()`, `check_stale_plan_on_main()`

- [ ] **Step 1: Write failing tests for S5 (branch mismatch)**

Append to `tests/test_corruption.py`:

```python
class TestS5BranchMismatch:
    def test_mismatch_returns_error(self, tmp_path):
        from corruption import check_branch_mismatch
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        finding = check_branch_mismatch(plan, tmp_path, current_branch="main", base_branch="main")
        assert finding is not None
        assert finding.scenario == "S5_BRANCH_MISMATCH"
        assert finding.severity == "error"
        assert "switch_to_plan_branch" in finding.actions

    def test_matching_branches_returns_none(self, tmp_path):
        from corruption import check_branch_mismatch
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        finding = check_branch_mismatch(plan, tmp_path, current_branch="issue-42-foo", base_branch="main")
        assert finding is None

    def test_main_plan_on_main_returns_none(self, tmp_path):
        from corruption import check_branch_mismatch
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="main")
        finding = check_branch_mismatch(plan, tmp_path, current_branch="main", base_branch="main")
        assert finding is None

    def test_no_plan_returns_none(self, tmp_path):
        from corruption import check_branch_mismatch
        finding = check_branch_mismatch(tmp_path / ".plan", tmp_path, current_branch="main", base_branch="main")
        assert finding is None

    def test_nonmain_base_branch(self, tmp_path):
        from corruption import check_branch_mismatch
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="develop")
        finding = check_branch_mismatch(plan, tmp_path, current_branch="develop", base_branch="develop")
        assert finding is None
```

- [ ] **Step 2: Write failing tests for S7 (stale plan on main)**

Append to `tests/test_corruption.py`:

```python
class TestS7StalePlanOnMain:
    def test_stale_plan_on_main(self, tmp_path):
        from corruption import check_stale_plan_on_main
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo", state="active")
        finding = check_stale_plan_on_main(plan, meta_state="active", base_branch="main", on_main=True)
        assert finding is not None
        assert finding.scenario == "S7_STALE_PLAN_ON_MAIN"

    def test_drained_plan_on_main_is_ok(self, tmp_path):
        from corruption import check_stale_plan_on_main
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="main", state="drained")
        finding = check_stale_plan_on_main(plan, meta_state="drained", base_branch="main", on_main=True)
        assert finding is None

    def test_main_plan_on_main_is_ok(self, tmp_path):
        from corruption import check_stale_plan_on_main
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="main", state="active")
        finding = check_stale_plan_on_main(plan, meta_state="active", base_branch="main", on_main=True)
        assert finding is None

    def test_not_on_main_returns_none(self, tmp_path):
        from corruption import check_stale_plan_on_main
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        finding = check_stale_plan_on_main(plan, meta_state="active", base_branch="main", on_main=False)
        assert finding is None
```

- [ ] **Step 3: Write failing tests for S6 (branch not exist) and S4 (closing postconditions)**

Append to `tests/test_corruption.py`:

```python
class TestS6BranchNotExist:
    def test_missing_branch_returns_error(self, tmp_path, monkeypatch):
        from corruption import check_branch_exists
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': '', 'returncode': 0,
        })())
        finding = check_branch_exists(plan, tmp_path)
        assert finding is not None
        assert finding.scenario == "S6_BRANCH_NOT_EXIST"
        assert finding.severity == "error"
        assert "remove_plan" in finding.actions

    def test_existing_branch_returns_none(self, tmp_path, monkeypatch):
        from corruption import check_branch_exists
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': 'issue-42-foo', 'returncode': 0,
        })())
        finding = check_branch_exists(plan, tmp_path)
        assert finding is None

    def test_no_plan_returns_none(self, tmp_path):
        from corruption import check_branch_exists
        assert check_branch_exists(tmp_path / ".plan", tmp_path) is None


class TestS4ClosingPostconditions:
    def test_closing_stamped_without_stamp_commit(self, tmp_path, monkeypatch):
        from corruption import check_closing_postconditions
        plan = tmp_path / ".plan"
        _write_plan(plan, state="closing:stamped", branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': 'abc1234 feat: add feature\n', 'returncode': 0,
        })())
        finding = check_closing_postconditions("closing:stamped", plan, tmp_path, tmp_path, "main")
        assert finding is not None
        assert finding.scenario == "S4_CLOSING_POSTCONDITION"
        assert "continue_close" in finding.actions

    def test_closing_stamped_with_stamp_commit(self, tmp_path, monkeypatch):
        from corruption import check_closing_postconditions
        plan = tmp_path / ".plan"
        _write_plan(plan, state="closing:stamped", branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': 'chore: branch closed — landed as abc on main\n', 'returncode': 0,
        })())
        finding = check_closing_postconditions("closing:stamped", plan, tmp_path, tmp_path, "main")
        assert finding is None

    def test_non_closing_state_returns_none(self, tmp_path):
        from corruption import check_closing_postconditions
        plan = tmp_path / ".plan"
        _write_plan(plan)
        finding = check_closing_postconditions("active", plan, tmp_path, tmp_path, "main")
        assert finding is None

    def test_closing_review_no_postcondition_check(self, tmp_path):
        from corruption import check_closing_postconditions
        plan = tmp_path / ".plan"
        _write_plan(plan, state="closing:review")
        finding = check_closing_postconditions("closing:review", plan, tmp_path, tmp_path, "main")
        assert finding is None
```

- [ ] **Step 4: Run all new tests to verify they fail**

Run: `python3 -m pytest tests/test_corruption.py::TestS5BranchMismatch tests/test_corruption.py::TestS7StalePlanOnMain tests/test_corruption.py::TestS6BranchNotExist tests/test_corruption.py::TestS4ClosingPostconditions -v`
Expected: FAIL

- [ ] **Step 5: Implement S5, S7, S6, S4 check functions**

Append to `project/corruption.py`:

```python
def check_branch_mismatch(
    plan_path: Path, workspace: Path,
    current_branch: str, base_branch: str,
) -> Optional[Finding]:
    if not plan_path.exists():
        return None
    plan_branch = _read_plan_field(plan_path, "branch")
    if not plan_branch:
        return None
    if plan_branch == current_branch:
        return None
    if plan_branch == base_branch and current_branch == base_branch:
        return None
    actions = ["switch_to_plan_branch", "update_plan_branch", "remove_plan"]
    branch_out, _ = _git(workspace, "branch", "--list", plan_branch)
    if not branch_out.strip():
        actions = ["update_plan_branch", "remove_plan"]
    return Finding(
        scenario="S5_BRANCH_MISMATCH",
        severity="error",
        detail=f".plan says branch '{plan_branch}', git says '{current_branch}'",
        actions=actions,
    )


def check_stale_plan_on_main(
    plan_path: Path, meta_state: str, base_branch: str, on_main: bool,
) -> Optional[Finding]:
    if not on_main or not plan_path.exists():
        return None
    plan_branch = _read_plan_field(plan_path, "branch")
    if not plan_branch or plan_branch == base_branch:
        return None
    if meta_state == "drained":
        return None
    return Finding(
        scenario="S7_STALE_PLAN_ON_MAIN",
        severity="warning",
        detail=f"stale .plan on {base_branch} — references branch '{plan_branch}' with state '{meta_state}'",
        actions=["switch_to_branch", "remove_plan"],
    )


def check_branch_exists(plan_path: Path, project: Path) -> Optional[Finding]:
    if not plan_path.exists():
        return None
    plan_branch = _read_plan_field(plan_path, "branch")
    if not plan_branch:
        return None
    local_out, _ = _git(project, "branch", "--list", plan_branch)
    if local_out.strip():
        return None
    remote_out, rc = _git(project, "ls-remote", "--heads", "origin", plan_branch)
    if rc == 0 and remote_out.strip():
        return Finding(
            scenario="S6_BRANCH_NOT_EXIST",
            severity="warning",
            detail=f"branch '{plan_branch}' not local but exists on remote",
            actions=["fetch_and_checkout", "remove_plan"],
        )
    return Finding(
        scenario="S6_BRANCH_NOT_EXIST",
        severity="error",
        detail=f"branch '{plan_branch}' doesn't exist locally or on remote",
        actions=["remove_plan", "recreate_branch"],
    )


def check_closing_postconditions(
    meta_state: str, plan_path: Path,
    project: Path, workspace: Path, base_branch: str,
) -> Optional[Finding]:
    if not meta_state.startswith("closing:"):
        return None
    sub = meta_state.split(":", 1)[1]
    plan_branch = _read_plan_field(plan_path, "branch") or ""

    checks: dict[str, tuple[list[str], str]] = {
        "promoted": (
            ["git", "-C", str(workspace), "log", "--oneline", "-1", "--",
             ".artifacts-promoted"],
            "no .artifacts-promoted stamp found",
        ),
        "pushed": (
            ["git", "-C", str(project), "ls-remote", "--heads", "origin", plan_branch],
            f"branch '{plan_branch}' not on remote",
        ),
        "merged": (
            ["git", "-C", str(project), "log", "--oneline",
             f"{base_branch}..{plan_branch}"],
            f"branch '{plan_branch}' has unmerged commits",
        ),
        "stamped": (
            ["git", "-C", str(project), "log", "-1", "--format=%s", plan_branch],
            "last commit is not a closure stamp",
        ),
    }
    if sub not in checks:
        return None
    cmd, failure_detail = checks[sub]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
    except subprocess.TimeoutExpired:
        return None
    stdout = result.stdout.strip()

    failed = False
    if sub == "promoted":
        failed = not stdout
    elif sub == "pushed":
        failed = not stdout
    elif sub == "merged":
        non_stamp = [l for l in stdout.splitlines()
                     if not l.split(" ", 1)[-1].startswith("chore: branch closed")]
        failed = bool(non_stamp)
    elif sub == "stamped":
        failed = not stdout.startswith("chore: branch closed")

    if not failed:
        return None

    actions = ["continue_close"]
    if sub in ("review", "verified"):
        actions.append("rollback_to_active")
    return Finding(
        scenario="S4_CLOSING_POSTCONDITION",
        severity="error",
        detail=f"state: {meta_state} but {failure_detail} — ceremony was interrupted",
        actions=actions,
    )
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_corruption.py -v`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/corruption.py tests/test_corruption.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#262): S4/S5/S6/S7 git-local corruption checks

Refs #262"
```

---

### Task 3: GitHub checks (S3, S8) + diagnose() orchestrator

**Files:**
- Modify: `project/corruption.py`
- Modify: `tests/test_corruption.py`

**Interfaces:**
- Consumes: `Finding`, `_git()`, `_read_plan_field()`, all `check_*` from Tasks 1-2
- Produces: `check_active_all_closed()`, `check_queue_consistency()`, `diagnose()`

- [ ] **Step 1: Write failing tests for S3 (active all closed)**

Append to `tests/test_corruption.py`:

```python
class TestS3ActiveAllClosed:
    def test_skipped_without_owner_repo(self, tmp_path):
        from corruption import check_active_all_closed
        plan = tmp_path / ".plan"
        _write_plan(plan)
        assert check_active_all_closed(plan, "active", owner_repo="") is None

    def test_skipped_for_non_active_state(self, tmp_path):
        from corruption import check_active_all_closed
        plan = tmp_path / ".plan"
        _write_plan(plan, state="closing:review")
        assert check_active_all_closed(plan, "closing:review", owner_repo="Org/repo") is None

    def test_all_closed_returns_warning(self, tmp_path, monkeypatch):
        from corruption import check_active_all_closed
        plan = tmp_path / ".plan"
        _write_plan(plan)

        def mock_run(*args, **kwargs):
            cmd = args[0] if args else kwargs.get("args", [])
            if "gh" in cmd:
                return type('R', (), {'stdout': 'CLOSED\n', 'returncode': 0})()
            return type('R', (), {'stdout': '', 'returncode': 0})()

        monkeypatch.setattr("corruption.subprocess.run", mock_run)
        finding = check_active_all_closed(plan, "active", owner_repo="Hortora/soredium")
        assert finding is not None
        assert finding.scenario == "S3_ACTIVE_ALL_CLOSED"
        assert finding.severity == "warning"


class TestS8QueueConsistency:
    def test_skipped_without_owner_repo(self, tmp_path):
        from corruption import check_queue_consistency
        plan = tmp_path / ".plan"
        _write_plan(plan)
        assert check_queue_consistency(plan, owner_repo="") is None

    def test_consistent_queue_returns_none(self, tmp_path, monkeypatch):
        from corruption import check_queue_consistency
        plan = tmp_path / ".plan"
        _write_plan(plan)

        def mock_run(*args, **kwargs):
            cmd = args[0] if args else kwargs.get("args", [])
            if "gh" in cmd:
                return type('R', (), {'stdout': 'OPEN\n', 'returncode': 0})()
            return type('R', (), {'stdout': '', 'returncode': 0})()

        monkeypatch.setattr("corruption.subprocess.run", mock_run)
        finding = check_queue_consistency(plan, owner_repo="Hortora/soredium")
        assert finding is None
```

- [ ] **Step 2: Write failing tests for diagnose() orchestrator**

Append to `tests/test_corruption.py`:

```python
class TestDiagnose:
    def test_healthy_state_returns_empty(self, tmp_path, monkeypatch):
        from corruption import diagnose
        plan = tmp_path / ".plan"
        _write_plan(plan, branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': 'issue-42-foo', 'returncode': 0,
        })())
        findings = diagnose(
            plan_path=plan, meta_state="active",
            project=tmp_path, workspace=tmp_path,
            base_branch="main", current_branch="issue-42-foo", on_main=False,
        )
        assert findings == []

    def test_no_plan_returns_empty(self, tmp_path):
        from corruption import diagnose
        findings = diagnose(
            plan_path=None, meta_state="",
            project=tmp_path, workspace=tmp_path,
        )
        assert findings == []

    def test_invalid_state_short_circuits(self, tmp_path, monkeypatch):
        from corruption import diagnose
        plan = tmp_path / ".plan"
        _write_plan(plan, state="bogus", branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': 'issue-42-foo', 'returncode': 0,
        })())
        findings = diagnose(
            plan_path=plan, meta_state="corrupted:bogus",
            project=tmp_path, workspace=tmp_path,
            current_branch="issue-42-foo",
        )
        scenarios = {f.scenario for f in findings}
        assert "S2_INVALID_STATE" in scenarios
        assert "S3_ACTIVE_ALL_CLOSED" not in scenarios
        assert "S4_CLOSING_POSTCONDITION" not in scenarios
        assert "S6_BRANCH_NOT_EXIST" not in scenarios

    def test_multiple_findings_returned(self, tmp_path, monkeypatch):
        from corruption import diagnose
        plan = tmp_path / ".plan"
        _write_plan_no_state(plan, branch="issue-42-foo")
        monkeypatch.setattr("corruption.subprocess.run", lambda *a, **kw: type('R', (), {
            'stdout': '', 'returncode': 0,
        })())
        findings = diagnose(
            plan_path=plan, meta_state="active",
            project=tmp_path, workspace=tmp_path,
            current_branch="main", on_main=True, base_branch="main",
        )
        scenarios = {f.scenario for f in findings}
        assert "S1_MISSING_STATE" in scenarios
        assert "S7_STALE_PLAN_ON_MAIN" in scenarios
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_corruption.py::TestS3ActiveAllClosed tests/test_corruption.py::TestS8QueueConsistency tests/test_corruption.py::TestDiagnose -v`
Expected: FAIL

- [ ] **Step 4: Implement S3, S8, and diagnose()**

Append to `project/corruption.py`:

```python
def check_active_all_closed(
    plan_path: Path, meta_state: str, owner_repo: str,
) -> Optional[Finding]:
    if not owner_repo or meta_state != "active":
        return None
    if not plan_path.exists():
        return None
    covers = _read_plan_field(plan_path, "covers")
    if not covers:
        return None
    issue_repo = _read_plan_field(plan_path, "issue-repo") or owner_repo
    issue_nums = [n.strip() for n in covers.split(",") if n.strip()]
    all_closed = True
    for num in issue_nums:
        try:
            result = subprocess.run(
                ["gh", "issue", "view", num, "--repo", issue_repo,
                 "--json", "state", "--jq", ".state"],
                capture_output=True, text=True, timeout=5,
            )
            if result.returncode != 0 or result.stdout.strip() != "CLOSED":
                all_closed = False
                break
        except subprocess.TimeoutExpired:
            return None
    if not all_closed:
        return None
    return Finding(
        scenario="S3_ACTIVE_ALL_CLOSED",
        severity="warning",
        detail=f"state: active but all issues in covers ({covers}) are CLOSED on GitHub",
        actions=["transition_to_drained", "mark_complete_and_next", "reopen_issues"],
    )


def check_queue_consistency(plan_path: Path, owner_repo: str) -> Optional[Finding]:
    if not owner_repo or not plan_path.exists():
        return None
    content = plan_path.read_text()
    in_queue = False
    issues: list[tuple[int, bool]] = []
    for line in content.splitlines():
        if line.strip() == "## Queue":
            in_queue = True
            continue
        if line.startswith("## "):
            in_queue = False
            continue
        if not in_queue:
            continue
        import re
        m = re.match(r'\s*- \[([ x])\] #(\d+)', line)
        if m:
            completed = m.group(1) == "x"
            issue_num = int(m.group(2))
            issues.append((issue_num, completed))

    if not issues:
        return None

    issue_repo = _read_plan_field(plan_path, "issue-repo") or owner_repo
    inconsistencies = []
    for num, completed in issues:
        try:
            result = subprocess.run(
                ["gh", "issue", "view", str(num), "--repo", issue_repo,
                 "--json", "state", "--jq", ".state"],
                capture_output=True, text=True, timeout=5,
            )
            if result.returncode != 0:
                continue
            gh_state = result.stdout.strip()
            if not completed and gh_state == "CLOSED":
                inconsistencies.append(f"#{num} unchecked but CLOSED")
            elif completed and gh_state == "OPEN":
                inconsistencies.append(f"#{num} checked but OPEN")
        except subprocess.TimeoutExpired:
            return None

    if not inconsistencies:
        return None
    return Finding(
        scenario="S8_QUEUE_INCONSISTENT",
        severity="warning",
        detail=f"queue inconsistency: {len(inconsistencies)} issue(s) differ from GitHub — {', '.join(inconsistencies)}",
        actions=["sync_plan_with_github", "ignore"],
    )


def diagnose(
    plan_path: Optional[Path],
    meta_state: str,
    project: Path,
    workspace: Path,
    base_branch: str = "main",
    current_branch: str = "",
    on_main: bool = False,
    owner_repo: str = "",
) -> list[Finding]:
    if plan_path is None or not plan_path.exists():
        return []
    findings: list[Finding] = []
    try:
        s1 = check_missing_state(plan_path)
        if s1:
            findings.append(s1)

        s2 = check_invalid_state(meta_state, plan_path)
        if s2:
            findings.append(s2)
            return findings

        s5 = check_branch_mismatch(plan_path, workspace, current_branch, base_branch)
        if s5:
            findings.append(s5)

        if not s5:
            s6 = check_branch_exists(plan_path, project)
            if s6:
                findings.append(s6)

        s7 = check_stale_plan_on_main(plan_path, meta_state, base_branch, on_main)
        if s7:
            findings.append(s7)

        s4 = check_closing_postconditions(meta_state, plan_path, project, workspace, base_branch)
        if s4:
            findings.append(s4)

        s3 = check_active_all_closed(plan_path, meta_state, owner_repo)
        if s3:
            findings.append(s3)

        s8 = check_queue_consistency(plan_path, owner_repo)
        if s8:
            findings.append(s8)
    except Exception:
        pass

    return findings
```

- [ ] **Step 5: Run all corruption tests**

Run: `python3 -m pytest tests/test_corruption.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/corruption.py tests/test_corruption.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#262): S3/S8 GitHub checks + diagnose() orchestrator

Refs #262"
```

---

## Batch 2: Integration

### Task 4: work_state.py CorruptedState handling + ctx.py corruption output

**Files:**
- Modify: `project/work_state.py:111`
- Modify: `project/ctx.py:236-297`
- Modify: `tests/test_work_state.py`
- Modify: `tests/test_ctx.py`

**Interfaces:**
- Consumes: `corruption.diagnose()`, `corruption.Finding` from Batch 1
- Produces: `CORRUPTION_COUNT`, `CORRUPTION_N`, `CORRUPTION_N_SEVERITY`, `CORRUPTION_N_DETAIL`, `CORRUPTION_N_ACTIONS` in ctx.py output

- [ ] **Step 1: Write failing test for work_state.py CorruptedState catch**

Add to `tests/test_work_state.py`:

```python
class TestCorruptedStateHandling:
    def test_corrupted_state_stored_as_prefix(self, tmp_path):
        """work_state.detect() catches CorruptedState and stores corrupted:<value>"""
        # Create a .plan with invalid state
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: issue-42-foo\nstate: bogus\ndate: 2026-08-20\n\n"
            "## Queue\n"
        )
        # This requires setting up a Topology — check existing test patterns
        # and adapt. The key assertion:
        # state.meta_state should be "corrupted:bogus", not raise
```

Note: adapt to match the existing Topology fixture pattern in `tests/test_work_state.py`. The key behavior: `detect()` must not raise when `.plan` has an invalid state — it must store `"corrupted:<value>"` in `meta_state`.

- [ ] **Step 2: Implement CorruptedState catch in work_state.py**

Edit `project/work_state.py:110-113`:

```python
# Before:
    meta_state = _read_state(state_file) if state_file else ""
    meta_state = meta_state or ""

# After:
    from lifecycle import CorruptedState
    try:
        meta_state = _read_state(state_file) if state_file else ""
        meta_state = meta_state or ""
    except CorruptedState as e:
        meta_state = f"corrupted:{e.raw_value}"
```

- [ ] **Step 3: Write failing test for ctx.py corruption output**

Add to `tests/test_ctx.py` (adapt to existing pattern):

```python
class TestCorruptionOutput:
    def test_corruption_count_zero_when_healthy(self, ...):
        # Set up healthy state, call resolve()
        # Assert "CORRUPTION_COUNT" in result and value is "0"
        pass

    def test_corruption_findings_serialised(self, ...):
        # Set up corrupted state, call resolve()
        # Assert CORRUPTION_0, CORRUPTION_0_SEVERITY, etc.
        pass
```

- [ ] **Step 4: Implement ctx.py corruption integration**

Edit `project/ctx.py`. After the `state = ws_detect(...)` line (~107), add the corruption call. Before the `return {}` dict (~236), add the corruption fields.

Add import at top:
```python
from corruption import diagnose as _diagnose
```

After `state = ws_detect(topo, base_branch=base_branch)` (line 107):
```python
    _plan_file = Path(state.plan_path) if state.plan_path else None
    corruption_findings = _diagnose(
        plan_path=_plan_file,
        meta_state=state.meta_state,
        project=topo.project,
        workspace=topo.workspace,
        base_branch=base_branch,
        current_branch=current_branch if 'current_branch' in dir() else "",
        on_main=state.on_main,
        owner_repo=owner_repo if 'owner_repo' in dir() else "",
    )
```

Note: `current_branch` and `owner_repo` are computed later in `resolve()`. Move the corruption call after both are available (after line 176 where `current_branch` is set, and after line 102 where `owner_repo` is set). The exact placement:

After line 178 (`current_branch = workspace_branch`), add the corruption call.

Add to the return dict:
```python
        "CORRUPTION_COUNT": str(len(corruption_findings)),
        **{
            f"CORRUPTION_{i}": f.scenario
            for i, f in enumerate(corruption_findings)
        },
        **{
            f"CORRUPTION_{i}_SEVERITY": f.severity
            for i, f in enumerate(corruption_findings)
        },
        **{
            f"CORRUPTION_{i}_DETAIL": f.detail
            for i, f in enumerate(corruption_findings)
        },
        **{
            f"CORRUPTION_{i}_ACTIONS": ",".join(f.actions)
            for i, f in enumerate(corruption_findings)
        },
```

- [ ] **Step 5: Run tests**

Run: `python3 -m pytest tests/test_work_state.py tests/test_ctx.py tests/test_corruption.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_state.py project/ctx.py tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#262): integrate corruption detection into ctx.py pipeline

work_state.py catches CorruptedState, ctx.py calls diagnose() and
emits CORRUPTION_* KEY=VALUE lines.

Refs #262"
```

---

### Task 5: base_branch threading in work_health.py (#263) + delegation

**Files:**
- Modify: `project/work_health.py:69-101,167-193,196-203,491-535`
- Modify: `tests/test_work_health.py`

**Interfaces:**
- Consumes: `corruption.check_branch_mismatch`, `corruption.check_stale_plan_on_main` from Batch 1
- Produces: Updated `check_meta_consistency(project, workspace, base_branch)`, `check_stale_scaffold_on_main(project, workspace, base_branch)`, `check_dirty_main(project, workspace, base_branch)`, `run_checks(..., base_branch)`

- [ ] **Step 1: Write failing tests for base_branch parameter**

Add to `tests/test_work_health.py`:

```python
class TestBaseBranchThreading:
    def test_meta_consistency_uses_base_branch(self, tmp_path):
        workspace = _init_repo(tmp_path / "wksp")
        _git(workspace, "checkout", "-b", "develop")
        plan = workspace / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\n"
            "branch: develop\nstate: active\ndate: 2026-08-20\n\n"
            "## Queue\n"
        )
        _git(workspace, "add", ".")
        _git(workspace, "commit", "-m", "scaffold")
        result = check_meta_consistency(str(tmp_path / "proj"), str(workspace), base_branch="develop")
        assert "STATUS=ok" in result

    def test_stale_scaffold_uses_base_branch(self, tmp_path):
        workspace = _init_repo(tmp_path / "wksp")
        _git(workspace, "checkout", "-b", "develop")
        _git(workspace, "checkout", "develop")
        result = check_stale_scaffold_on_main(str(tmp_path / "proj"), str(workspace), base_branch="develop")
        # Not on develop (we're on develop but it's the base), should be ok
        assert "STATUS=ok" in result

    def test_dirty_main_uses_base_branch(self, tmp_path):
        workspace = _init_repo(tmp_path / "wksp")
        _git(workspace, "checkout", "-b", "develop")
        (workspace / "dirty.txt").write_text("dirty")
        result = check_dirty_main(str(tmp_path / "proj"), str(workspace), base_branch="develop")
        # On develop (base branch) with uncommitted changes
        assert "STATUS=warn" in result

    def test_run_checks_accepts_base_branch(self, tmp_path):
        workspace = _init_repo(tmp_path / "wksp")
        project = _init_repo(tmp_path / "proj")
        # Should not crash
        run_checks("entry", str(project), str(workspace), base_branch="develop")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_health.py::TestBaseBranchThreading -v`
Expected: FAIL (TypeError: unexpected keyword argument 'base_branch')

- [ ] **Step 3: Add base_branch parameter to all affected functions**

Edit `project/work_health.py`:

1. `check_meta_consistency(project, workspace)` → `check_meta_consistency(project, workspace, base_branch="main")`
   - Line 93: `current == "main"` → `current == base_branch`
   - Line 95: `current == "main"` → `current == base_branch`

2. `check_stale_scaffold_on_main(project, workspace)` → `check_stale_scaffold_on_main(project, workspace, base_branch="main")`
   - Line 169: `current != "main"` → `current != base_branch`

3. `check_dirty_main(project, workspace)` → `check_dirty_main(project, workspace, base_branch="main")`
   - Line 199: `current != "main"` → `current != base_branch`

4. `run_checks(scope, project, workspace, branch=None, owner_repo=None)` → add `base_branch="main"`
   - Thread `base_branch` to each check in the lambda lists

5. `main()` — add `--base-branch` argument:
   ```python
   parser.add_argument("--base-branch", default="main")
   ```
   Pass to `run_checks()`.

6. Update ENTRY_CHECKS and WRAP_CHECKS lambdas to pass base_branch:
   ```python
   ENTRY_CHECKS = [
       lambda p, w, bb="main": check_meta_consistency(p, w, bb),
       ...
   ```
   This is a structural change — the lambdas need a third parameter. Change `run_checks` to pass `base_branch` to each check function:
   ```python
   for check_fn in checks:
       result = check_fn(project, workspace, base_branch)
   ```
   And update all lambdas to accept `(p, w, bb)`.

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All PASS

- [ ] **Step 5: Run full test suite to check for regressions**

Run: `python3 -m pytest tests/test_corruption.py tests/test_work_health.py tests/test_lifecycle.py tests/test_work_state.py tests/test_ctx.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_health.py tests/test_work_health.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix(#263): thread base_branch through work_health.py

Replace hardcoded 'main' comparisons with base_branch parameter in
check_meta_consistency, check_stale_scaffold_on_main, check_dirty_main,
and run_checks.

Closes #263"
```

---

### Task 6: work/SKILL.md triage routing

**Files:**
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: `CORRUPTION_COUNT`, `CORRUPTION_*` lines from ctx.py (Task 4)
- Produces: Triage flow in the work router

- [ ] **Step 1: Add corruption triage to Step 1b**

Edit `work/SKILL.md`. After the `python3 ~/.claude/skills/project/ctx.py` instruction in Step 1b, add:

```markdown
**Step 1b-pre — Corruption triage (before normal routing)**

If `CORRUPTION_COUNT` > 0, enter triage flow instead of normal routing:

```
⚠️ Lifecycle corruption detected (N finding(s)):

  1. [SEVERITY] SCENARIO — DETAIL
     Actions:
       a. action_1 (Recommended)
       b. action_2
       c. action_3

Pick actions (e.g. "1a 2a") or describe what you want:
```

After the user confirms actions, execute them:

| Action | Command |
|--------|---------|
| `accept_default` | No-op — continue with defaulted state |
| `write_active` | `python3 ~/.claude/skills/project/lifecycle.py commit-transition <PLAN_PATH> from_state=idle new_state=active event=work` |
| `write_scaffolded` | Same pattern with `new_state=scaffolded` |
| `switch_to_plan_branch` | Branch Switch Helper from work-start |
| `update_plan_branch` | `python3 ~/.claude/skills/work-slot/plan_manager.py set-state <PLAN_PATH> key=branch value=<current_branch>` |
| `remove_plan` | `rm <PLAN_PATH>` and commit |
| `continue_close` | Route to work-end at current gate |
| `rollback_to_active` | `python3 ~/.claude/skills/project/lifecycle.py commit-transition <PLAN_PATH> from_state=<current> new_state=active event=abort_close` |
| `transition_to_drained` | Route to work-end |
| `sync_plan_with_github` | `python3 project/work_health.py --scope entry --project $PROJECT --workspace $WORKSPACE --owner-repo $OWNER_REPO` |
| `fetch_and_checkout` | `git -C $PROJECT fetch origin <branch> && git -C $PROJECT checkout <branch>` |
| `recreate_branch` | Route to work-start with the issue number |
| `ignore` | No-op — continue normally |

After executing actions, re-run `ctx.py` to verify corruption is resolved.
If `CORRUPTION_COUNT` is still > 0, report remaining findings.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#262): triage routing in work/SKILL.md for corruption findings

Refs #262"
```

---

## References

- specs/issue-262-lifecycle-corruption-recovery/2026-08-20-lifecycle-corruption-recovery-design.md — design spec
- project/lifecycle.py — state machine, CorruptedState, VALID_STATES, read_state
- project/work_state.py:111 — latent CorruptedState crash
- project/ctx.py — KEY=VALUE output, integration point
- project/work_health.py:93,169,199 — hardcoded "main" (#263)
- tests/test_lifecycle.py — _write_plan helper, test conventions
- tests/test_work_health.py — _init_repo helper, git test patterns
- GE-20260803-263c2c — explicit state machine pattern
- docs/protocols/externalised-scripts-require-tests.md
- docs/protocols/evidence-before-claims.md
- GitHub #262, #263

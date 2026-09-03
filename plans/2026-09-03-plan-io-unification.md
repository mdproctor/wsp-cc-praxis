# Plan I/O Unification — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #327 — S3_ACTIVE_ALL_CLOSED fires incorrectly for cross-repo queues
**Issue group:** #327

**Goal:** Extract `project/plan_io.py` as a single source of truth for all `.plan` file operations, fix the S3/S8 false positives, and migrate all consumers.

**Architecture:** One new module (`project/plan_io.py`) owns all CRUD operations on `.plan` files — reading fields, parsing queues, parsing covers, writing fields, and removing plans. Every consumer that currently reimplements section walking or covers splitting imports from this module instead. The queue regex matches `plan_manager.py`'s canonical `_ITEM_RE` pattern.

**Tech Stack:** Python 3, pytest, pathlib

## Global Constraints

- `plan_io.py` lives in `project/` — same directory as `corruption.py`, `lifecycle.py`, `ctx.py`
- Queue regex must handle both `#N` and `owner/repo#N` formats
- `QueueItem` is read-only (frozen dataclass) with 5 fields — intentionally simpler than `plan_manager.QueueItem`
- Atomic writes via `.plan.tmp` → rename (matches existing lifecycle.py pattern)
- All consumers outside `project/` must add `project/` to `sys.path` before importing

---

## Batch 1: plan_io.py foundation

### Task 1: Read API — read_plan, read_field, parse_covers, has_uncompleted_items

**Files:**
- Create: `project/plan_io.py`
- Create: `tests/test_plan_io.py`

**Interfaces:**
- Produces: `PlanState`, `QueueItem`, `read_plan(Path) -> PlanState | None`, `read_field(Path, str) -> str | None`, `parse_covers(str) -> list[int]`, `has_uncompleted_items(PlanState) -> bool`

- [ ] **Step 1: Write failing tests for data types and read_plan**

```python
# tests/test_plan_io.py
import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


def _make_plan(tmp_path, content):
    p = tmp_path / ".plan"
    p.write_text(content)
    return p


BASIC_PLAN = """\
# Work Plan — test

## State
branch: issue-42-foo
state: active
date: 2026-08-20
issue-repo: Hortora/soredium
covers: 42

## Queue
- [ ] Hortora/soredium#42 — Fix foo ← active
"""

CROSS_REPO_PLAN = """\
# Work Plan — test

## State
branch: issue-74-design
state: active
date: 2026-08-20
issue-repo: Hortora/soredium
covers: 74

## Queue
- [x] Hortora/soredium#74 — Design session
- [ ] casehubio/blocks#231 — Extract summarisation types ← active
- [ ] casehubio/blocks#233 — Refactor pipeline
"""

PLAN_WITH_UNPARSED = """\
# Work Plan — test

## State
branch: issue-99-test
state: active
covers: 99

## Queue
- [ ] Hortora/soredium#99 — Valid item
- This line doesn't match the regex
- [ ] #100 — Bare issue ref without repo
"""


class TestReadPlan:
    def test_basic_plan(self, tmp_path):
        from plan_io import read_plan
        plan = _make_plan(tmp_path, BASIC_PLAN)
        state = read_plan(plan)
        assert state is not None
        assert state.fields["branch"] == "issue-42-foo"
        assert state.fields["state"] == "active"
        assert state.fields["covers"] == "42"
        assert len(state.queue_items) == 1
        assert state.queue_items[0].repo == "Hortora/soredium"
        assert state.queue_items[0].number == 42
        assert state.queue_items[0].completed is False
        assert state.queue_items[0].active is True
        assert state.unparsed_lines == []

    def test_cross_repo_plan(self, tmp_path):
        from plan_io import read_plan
        plan = _make_plan(tmp_path, CROSS_REPO_PLAN)
        state = read_plan(plan)
        assert len(state.queue_items) == 3
        assert state.queue_items[0].completed is True
        assert state.queue_items[1].repo == "casehubio/blocks"
        assert state.queue_items[1].number == 231
        assert state.queue_items[1].completed is False
        assert state.queue_items[1].active is True
        assert state.queue_items[2].number == 233

    def test_unparsed_lines_tracked(self, tmp_path):
        from plan_io import read_plan
        plan = _make_plan(tmp_path, PLAN_WITH_UNPARSED)
        state = read_plan(plan)
        assert len(state.queue_items) == 1
        assert state.queue_items[0].number == 99
        assert len(state.unparsed_lines) == 2
        assert "This line doesn't match" in state.unparsed_lines[0]
        assert "#100" in state.unparsed_lines[1]

    def test_nonexistent_file_returns_none(self, tmp_path):
        from plan_io import read_plan
        assert read_plan(tmp_path / "nonexistent") is None

    def test_plan_without_queue_section(self, tmp_path):
        from plan_io import read_plan
        plan = _make_plan(tmp_path, "# Plan\n\n## State\nbranch: foo\nstate: active\n")
        state = read_plan(plan)
        assert state is not None
        assert state.fields["branch"] == "foo"
        assert state.queue_items == []

    def test_plan_without_state_section(self, tmp_path):
        from plan_io import read_plan
        plan = _make_plan(tmp_path, "# Plan\nbranch: foo\nstate: active\n")
        state = read_plan(plan)
        assert state is not None
        assert state.fields["branch"] == "foo"

    def test_epic_and_active_markers(self, tmp_path):
        from plan_io import read_plan
        content = "# Plan\n\n## State\nstate: active\n\n## Queue\n- [ ] org/repo#5 — Big feature (epic) ← active\n"
        plan = _make_plan(tmp_path, content)
        state = read_plan(plan)
        assert len(state.queue_items) == 1
        assert state.queue_items[0].active is True
        assert "epic" not in state.queue_items[0].title.lower()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_io.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'plan_io'`

- [ ] **Step 3: Implement plan_io.py read operations**

```python
# project/plan_io.py
"""Single source of truth for .plan file I/O.

Owns all CRUD operations: create, read, update, delete.
Every consumer imports from here — no inline section walking.
"""
from __future__ import annotations

import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional


@dataclass(frozen=True)
class QueueItem:
    repo: str
    number: int
    title: str
    completed: bool
    active: bool


@dataclass(frozen=True)
class PlanState:
    fields: dict[str, str]
    queue_items: list[QueueItem]
    unparsed_lines: list[str]


_ITEM_RE = re.compile(
    r'^(\s*)- \[([ x])\] '
    r'(?:([A-Za-z0-9._-]+/[A-Za-z0-9._-]+))?#(\d+)'
    r'\s*—\s*(.+?)(?:\s*\(epic\))?(?:\s*←\s*active)?$'
)

_ACTIVE_RE = re.compile(r'←\s*active')
_EPIC_RE = re.compile(r'\(epic\)')


def read_plan(plan_path: Path) -> Optional[PlanState]:
    if not plan_path.exists():
        return None
    content = plan_path.read_text()
    lines = content.splitlines()

    fields: dict[str, str] = {}
    queue_items: list[QueueItem] = []
    unparsed_lines: list[str] = []

    section = None
    has_sections = False

    for line in lines:
        stripped = line.strip()
        if stripped.startswith("## "):
            section = stripped
            has_sections = True
            continue

        if section == "## State" or (not has_sections and ":" in line and not line.startswith("#")):
            if ":" in line and not line.startswith("#") and not line.startswith("-"):
                k, _, v = line.partition(":")
                k = k.strip()
                if k:
                    fields[k] = v.strip()

        elif section == "## Queue":
            if not stripped or stripped.startswith("###") or stripped.startswith("("):
                continue
            m = _ITEM_RE.match(line)
            if m:
                repo = m.group(3) or ""
                title_raw = m.group(5).strip()
                title = _EPIC_RE.sub("", title_raw).strip()
                title = _ACTIVE_RE.sub("", title).strip()
                queue_items.append(QueueItem(
                    repo=repo,
                    number=int(m.group(4)),
                    title=title,
                    completed=m.group(2) == "x",
                    active=bool(_ACTIVE_RE.search(line)),
                ))
            elif stripped.startswith("- "):
                unparsed_lines.append(stripped)

    return PlanState(fields=fields, queue_items=queue_items, unparsed_lines=unparsed_lines)


def read_field(plan_path: Path, field_name: str) -> Optional[str]:
    state = read_plan(plan_path)
    if state is None:
        return None
    return state.fields.get(field_name)


def parse_covers(covers_str: str) -> list[int]:
    result = []
    for part in covers_str.split(","):
        part = part.strip()
        if part.isdigit():
            result.append(int(part))
    return result


def has_uncompleted_items(plan_state: PlanState) -> bool:
    return any(not item.completed for item in plan_state.queue_items)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_io.py -v`
Expected: all PASS

- [ ] **Step 5: Write tests for read_field, parse_covers, has_uncompleted_items**

Add to `tests/test_plan_io.py`:

```python
class TestReadField:
    def test_existing_field(self, tmp_path):
        from plan_io import read_field
        plan = _make_plan(tmp_path, BASIC_PLAN)
        assert read_field(plan, "branch") == "issue-42-foo"
        assert read_field(plan, "state") == "active"

    def test_missing_field(self, tmp_path):
        from plan_io import read_field
        plan = _make_plan(tmp_path, BASIC_PLAN)
        assert read_field(plan, "nonexistent") is None

    def test_nonexistent_file(self, tmp_path):
        from plan_io import read_field
        assert read_field(tmp_path / "nope", "branch") is None


class TestParseCovers:
    def test_single(self):
        from plan_io import parse_covers
        assert parse_covers("42") == [42]

    def test_multiple(self):
        from plan_io import parse_covers
        assert parse_covers("42,19,32") == [42, 19, 32]

    def test_whitespace(self):
        from plan_io import parse_covers
        assert parse_covers(" 42 , 19 , 32 ") == [42, 19, 32]

    def test_empty(self):
        from plan_io import parse_covers
        assert parse_covers("") == []

    def test_non_numeric_skipped(self):
        from plan_io import parse_covers
        assert parse_covers("42,abc,19") == [42, 19]


class TestHasUncompletedItems:
    def test_has_uncompleted(self, tmp_path):
        from plan_io import read_plan, has_uncompleted_items
        plan = _make_plan(tmp_path, CROSS_REPO_PLAN)
        state = read_plan(plan)
        assert has_uncompleted_items(state) is True

    def test_all_completed(self, tmp_path):
        from plan_io import read_plan, has_uncompleted_items
        content = "# Plan\n\n## State\nstate: active\n\n## Queue\n- [x] org/repo#1 — Done\n"
        plan = _make_plan(tmp_path, content)
        state = read_plan(plan)
        assert has_uncompleted_items(state) is False

    def test_empty_queue(self, tmp_path):
        from plan_io import read_plan, has_uncompleted_items
        plan = _make_plan(tmp_path, "# Plan\n\n## State\nstate: active\n\n## Queue\n")
        state = read_plan(plan)
        assert has_uncompleted_items(state) is False
```

- [ ] **Step 6: Run all plan_io tests**

Run: `python3 -m pytest tests/test_plan_io.py -v`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add project/plan_io.py tests/test_plan_io.py
git -C <PROJECT> commit -m "feat(#327): add project/plan_io.py read API — single source of truth for .plan parsing Refs #327"
```

---

### Task 2: Write API — write_field, write_fields, remove_plan

**Files:**
- Modify: `project/plan_io.py`
- Modify: `tests/test_plan_io.py`

**Interfaces:**
- Consumes: `read_plan(Path) -> PlanState | None` from Task 1
- Produces: `write_field(Path, str, str) -> None`, `write_fields(Path, dict[str, str]) -> None`, `remove_plan(Path) -> None`

- [ ] **Step 1: Write failing tests for write operations**

Add to `tests/test_plan_io.py`:

```python
class TestWriteField:
    def test_update_existing_field(self, tmp_path):
        from plan_io import write_field, read_field
        plan = _make_plan(tmp_path, BASIC_PLAN)
        write_field(plan, "state", "closing:review")
        assert read_field(plan, "state") == "closing:review"
        assert read_field(plan, "branch") == "issue-42-foo"

    def test_add_new_field(self, tmp_path):
        from plan_io import write_field, read_field
        plan = _make_plan(tmp_path, BASIC_PLAN)
        write_field(plan, "new-field", "new-value")
        assert read_field(plan, "new-field") == "new-value"
        assert read_field(plan, "state") == "active"

    def test_preserves_queue(self, tmp_path):
        from plan_io import write_field, read_plan
        plan = _make_plan(tmp_path, BASIC_PLAN)
        write_field(plan, "state", "drained")
        state = read_plan(plan)
        assert len(state.queue_items) == 1
        assert state.queue_items[0].number == 42

    def test_atomic_write(self, tmp_path):
        from plan_io import write_field
        plan = _make_plan(tmp_path, BASIC_PLAN)
        write_field(plan, "state", "paused")
        assert not (tmp_path / ".plan.tmp").exists()


class TestWriteFields:
    def test_batch_update(self, tmp_path):
        from plan_io import write_fields, read_plan
        plan = _make_plan(tmp_path, BASIC_PLAN)
        write_fields(plan, {"state": "drained", "branch": "main"})
        state = read_plan(plan)
        assert state.fields["state"] == "drained"
        assert state.fields["branch"] == "main"


class TestRemovePlan:
    def test_removes_file(self, tmp_path):
        from plan_io import remove_plan
        plan = _make_plan(tmp_path, BASIC_PLAN)
        assert plan.exists()
        remove_plan(plan)
        assert not plan.exists()

    def test_nonexistent_is_noop(self, tmp_path):
        from plan_io import remove_plan
        remove_plan(tmp_path / "nonexistent")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_io.py::TestWriteField -v`
Expected: FAIL — `ImportError: cannot import name 'write_field'`

- [ ] **Step 3: Implement write operations**

Add to `project/plan_io.py`:

```python
def write_field(plan_path: Path, field_name: str, value: str) -> None:
    write_fields(plan_path, {field_name: value})


def write_fields(plan_path: Path, updates: dict[str, str]) -> None:
    content = plan_path.read_text()
    lines = content.splitlines()

    in_state = False
    has_sections = False
    state_end = len(lines)
    updated = set()

    for i, line in enumerate(lines):
        stripped = line.strip()
        if stripped == "## State":
            in_state = True
            has_sections = True
            continue
        if stripped.startswith("## ") and in_state:
            state_end = i
            in_state = False
            continue

        check = in_state if has_sections else (
            ":" in line and not line.startswith("#") and not line.startswith("-")
        )
        if check:
            k = line.split(":", 1)[0].strip()
            if k in updates:
                lines[i] = f"{k}: {updates[k]}"
                updated.add(k)

    for k, v in updates.items():
        if k not in updated:
            lines.insert(state_end, f"{k}: {v}")
            state_end += 1

    tmp = plan_path.parent / ".plan.tmp"
    tmp.write_text("\n".join(lines) + "\n")
    tmp.replace(plan_path)


def remove_plan(plan_path: Path) -> None:
    if plan_path.exists():
        plan_path.unlink()
```

- [ ] **Step 4: Run all plan_io tests**

Run: `python3 -m pytest tests/test_plan_io.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add project/plan_io.py tests/test_plan_io.py
git -C <PROJECT> commit -m "feat(#327): add plan_io write API — write_field, write_fields, remove_plan Refs #327"
```

---

## Batch 2: Fix #327 — corruption.py

### Task 3: Migrate corruption.py to plan_io, fix S3 and S8

**Files:**
- Modify: `project/corruption.py:44-60` (remove `_read_plan_field`)
- Modify: `project/corruption.py:63-86` (simplify `check_missing_state`)
- Modify: `project/corruption.py:236-268` (fix `check_active_all_closed`)
- Modify: `project/corruption.py:271-340` (fix `check_queue_consistency`)
- Modify: `tests/test_corruption.py`

**Interfaces:**
- Consumes: `read_plan`, `read_field`, `parse_covers`, `has_uncompleted_items` from Task 1

- [ ] **Step 1: Write failing test for S3 with cross-repo uncompleted queue items**

Add to `tests/test_corruption.py` in `TestS3ActiveAllClosed`:

```python
def test_all_closed_but_queue_has_uncompleted_returns_none(self, tmp_path, monkeypatch):
    """#327: design issue closed, cross-repo implementation continues."""
    from corruption import check_active_all_closed
    plan = tmp_path / ".plan"
    lines = [
        "# Work Plan — test", "", "## State",
        "branch: issue-74-design", "state: active",
        "date: 2026-08-20", "issue-repo: Hortora/soredium",
        "covers: 74", "",
        "## Queue",
        "- [x] Hortora/soredium#74 — Design session",
        "- [ ] casehubio/blocks#231 — Extract summarisation types ← active",
        "- [ ] casehubio/blocks#233 — Refactor pipeline",
        "",
    ]
    plan.write_text("\n".join(lines))

    def mock_run(*args, **kwargs):
        return type('R', (), {'stdout': 'CLOSED\n', 'returncode': 0})()

    monkeypatch.setattr("corruption.subprocess.run", mock_run)
    finding = check_active_all_closed(plan, "active", owner_repo="Hortora/soredium")
    assert finding is None, "Should not fire when queue has uncompleted cross-repo items"
```

- [ ] **Step 2: Write failing test for S3 with empty queue still fires**

Add to `tests/test_corruption.py` in `TestS3ActiveAllClosed`:

```python
def test_all_closed_empty_queue_still_fires(self, tmp_path, monkeypatch):
    """Regression guard: S3 fires when covers closed AND queue is empty."""
    from corruption import check_active_all_closed
    plan = tmp_path / ".plan"
    lines = [
        "# Work Plan — test", "", "## State",
        "branch: issue-74-design", "state: active",
        "date: 2026-08-20", "issue-repo: Hortora/soredium",
        "covers: 74", "",
        "## Queue", "",
    ]
    plan.write_text("\n".join(lines))

    def mock_run(*args, **kwargs):
        return type('R', (), {'stdout': 'CLOSED\n', 'returncode': 0})()

    monkeypatch.setattr("corruption.subprocess.run", mock_run)
    finding = check_active_all_closed(plan, "active", owner_repo="Hortora/soredium")
    assert finding is not None, "Should fire when covers closed and queue is empty"
    assert finding.scenario == "S3_ACTIVE_ALL_CLOSED"
```

- [ ] **Step 3: Write failing test for S8 cross-repo queue parsing**

Add to `tests/test_corruption.py` in `TestS8QueueConsistency`:

```python
def test_cross_repo_queue_items_parsed(self, tmp_path, monkeypatch):
    """#327: cross-repo queue items must be visible to consistency check."""
    from corruption import check_queue_consistency
    plan = tmp_path / ".plan"
    lines = [
        "# Work Plan — test", "", "## State",
        "branch: issue-74-design", "state: active",
        "date: 2026-08-20", "issue-repo: Hortora/soredium",
        "covers: 74", "",
        "## Queue",
        "- [ ] casehubio/blocks#231 — Extract summarisation types ← active",
        "",
    ]
    plan.write_text("\n".join(lines))

    def mock_run(*args, **kwargs):
        return type('R', (), {'stdout': 'OPEN\tExtract summarisation types\n', 'returncode': 0})()

    monkeypatch.setattr("corruption.subprocess.run", mock_run)
    result = check_queue_consistency(plan, owner_repo="Hortora/soredium")
    assert result is None, "Cross-repo queue items should be parsed and checked"
```

- [ ] **Step 4: Run new tests to verify they fail**

Run: `python3 -m pytest tests/test_corruption.py::TestS3ActiveAllClosed::test_all_closed_but_queue_has_uncompleted_returns_none tests/test_corruption.py::TestS8QueueConsistency::test_cross_repo_queue_items_parsed -v`
Expected: FAIL

- [ ] **Step 5: Migrate corruption.py to plan_io**

Replace `_read_plan_field` and all inline parsing with plan_io imports. Fix `check_active_all_closed` to check queue. Fix `check_queue_consistency` to use `read_plan`.

In `project/corruption.py`:

1. Add import at top (after existing imports):
```python
from plan_io import read_plan, read_field, parse_covers, has_uncompleted_items
```

2. Delete `_read_plan_field()` function (lines 44-60)

3. Replace `check_missing_state()` (lines 63-86):
```python
def check_missing_state(plan_path: Path) -> Optional[Finding]:
    state = read_plan(plan_path)
    if state is None:
        return None
    if not state.fields:
        return None
    if "state" in state.fields:
        return None
    return Finding(
        scenario="S1_MISSING_STATE",
        severity="warning",
        detail="state: field missing from .plan — defaulted to 'active' (legacy migration)",
        actions=["accept_default", "write_scaffolded"],
    )
```

4. Update all `_read_plan_field(plan_path, "X")` calls to `read_field(plan_path, "X")` — these appear in `check_branch_mismatch`, `check_stale_plan_on_main`, `check_branch_exists`, `check_closing_postconditions`, `check_active_all_closed`, `check_queue_consistency`.

5. Replace `check_active_all_closed()` (lines 236-268):
```python
def check_active_all_closed(
    plan_path: Path, meta_state: str, owner_repo: str,
) -> Optional[Finding]:
    if not owner_repo or meta_state != "active":
        return None
    if not plan_path.exists():
        return None
    covers = read_field(plan_path, "covers")
    if not covers:
        return None
    issue_repo = read_field(plan_path, "issue-repo") or owner_repo
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
    plan_state = read_plan(plan_path)
    if plan_state and has_uncompleted_items(plan_state):
        return None
    total = len(plan_state.queue_items) if plan_state else 0
    return Finding(
        scenario="S3_ACTIVE_ALL_CLOSED",
        severity="warning",
        detail=f"state: active, all covers ({covers}) CLOSED, queue: 0 uncompleted / {total} total items",
        actions=["transition_to_drained", "mark_complete_and_next", "reopen_issues"],
    )
```

6. Replace `check_queue_consistency()` queue parsing (lines 271-340):
```python
def check_queue_consistency(plan_path: Path, owner_repo: str) -> Optional[Finding]:
    if not owner_repo or not plan_path.exists():
        return None
    plan_state = read_plan(plan_path)
    if plan_state is None or not plan_state.queue_items:
        return None

    issue_repo = plan_state.fields.get("issue-repo", owner_repo)
    covers_raw = plan_state.fields.get("covers", "")
    covers_nums = set(parse_covers(covers_raw))

    inconsistencies: list[str] = []
    for item in plan_state.queue_items:
        try:
            result = subprocess.run(
                ["gh", "issue", "view", str(item.number), "--repo", issue_repo,
                 "--json", "state,title", "--jq", "[.state, .title] | @tsv"],
                capture_output=True, text=True, timeout=5,
            )
            if result.returncode != 0:
                continue
            parts = result.stdout.strip().split("\t", 1)
            if len(parts) != 2:
                continue
            gh_state, gh_title = parts

            if item.title and gh_title and item.title.lower() not in gh_title.lower() and gh_title.lower() not in item.title.lower():
                if owner_repo != issue_repo:
                    continue

            if not item.completed and gh_state == "CLOSED":
                inconsistencies.append(f"#{item.number} unchecked but CLOSED")
            elif item.completed and gh_state == "OPEN":
                if item.number in covers_nums:
                    continue
                inconsistencies.append(f"#{item.number} checked but OPEN")
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
```

- [ ] **Step 6: Run all corruption tests**

Run: `python3 -m pytest tests/test_corruption.py -v`
Expected: all PASS (including new tests)

- [ ] **Step 7: Run plan_io tests too (no regressions)**

Run: `python3 -m pytest tests/test_plan_io.py tests/test_corruption.py -v`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C <PROJECT> add project/corruption.py tests/test_corruption.py
git -C <PROJECT> commit -m "fix(#327): S3/S8 false positives — migrate corruption.py to plan_io, check queue before S3 fires Refs #327"
```

---

## Batch 3: Consumer migrations

### Task 4: Migrate project/ files — lifecycle.py, work_health.py, ctx.py, close_progress.py

**Files:**
- Modify: `project/lifecycle.py:167-196` (replace `read_state` section parser)
- Modify: `project/lifecycle.py:199-239` (replace `write_state` section parser)
- Modify: `project/lifecycle.py:242-252` (replace `write_branch` section parser)
- Modify: `project/lifecycle.py:309-330` (replace `_read_branch` section parser)
- Modify: `project/work_health.py:76-89` (replace inline parser in `check_meta_consistency`)
- Modify: `project/work_health.py:250-262` (replace inline parser in `check_plan_queue`)
- Modify: `project/ctx.py:168` (replace covers split)
- Modify: `work-end/close_progress.py:105-111` (replace `_read_plan_state`)

**Interfaces:**
- Consumes: `read_field`, `read_plan`, `parse_covers`, `write_field` from Tasks 1-2

- [ ] **Step 1: Migrate lifecycle.py read functions**

In `project/lifecycle.py`:

Add import:
```python
from plan_io import read_field, write_field
```

Replace `read_state()` (lines 167-196):
```python
def read_state(plan_path: Path) -> Optional[str]:
    """Read lifecycle state from .plan's ## State section."""
    if not plan_path.exists():
        return None
    raw = read_field(plan_path, "state")
    if raw is None:
        return "active"
    if raw in VALID_STATES:
        return raw
    raise CorruptedState(plan_path, raw)
```

Replace `_read_branch()` (lines 309-330):
```python
def _read_branch(plan_path: Path) -> Optional[str]:
    """Read branch name from .plan's ## State section."""
    return read_field(plan_path, "branch")
```

- [ ] **Step 2: Migrate lifecycle.py write functions**

Replace `write_state()` (lines 199-239):
```python
def write_state(plan_path: Path, state: str) -> None:
    """Write lifecycle state to .plan's ## State section atomically."""
    write_field(plan_path, "state", state)
```

Replace `write_branch()` (lines 242-252):
```python
def write_branch(plan_path: Path, branch: str) -> None:
    """Write branch field to .plan's ## State section atomically."""
    write_field(plan_path, "branch", branch)
```

- [ ] **Step 3: Run lifecycle tests**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: all PASS

- [ ] **Step 4: Migrate work_health.py**

In `project/work_health.py`:

Add import:
```python
from plan_io import read_field
```

Replace lines 75-89 (inline parser in `check_meta_consistency`):
```python
    plan_path = Path(workspace) / ".plan"
    meta_path = Path(workspace) / "design" / ".meta"
    state_file = plan_path if plan_path.exists() else meta_path
    if not state_file.exists():
        return "CHECK=meta_consistency STATUS=ok"
    meta_branch = read_field(state_file, "branch")
    if not meta_branch:
        return "CHECK=meta_consistency STATUS=ok"
```

Replace lines 248-262 (inline parser in `check_plan_queue`):
```python
    state_file = Path(workspace) / ".plan"
    if state_file.exists():
        plan_branch = read_field(state_file, "branch")
        if plan_branch:
            branches_to_check.add(plan_branch)
```

- [ ] **Step 5: Migrate ctx.py**

In `project/ctx.py`, line 168:

Add import:
```python
from plan_io import parse_covers
```

Replace:
```python
issue_n = covers.split(",")[0].strip() if covers else ""
```
With:
```python
issue_n = str(parse_covers(covers)[0]) if covers else ""
```

- [ ] **Step 6: Migrate close_progress.py**

In `work-end/close_progress.py`:

Add import (after existing sys.path setup):
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import read_field
```

Replace `_read_plan_state()` (lines 105-111):
```python
def _read_plan_state(plan_path: Path) -> str:
    """Read the lifecycle state from a .plan file."""
    return read_field(plan_path, "state") or ""
```

- [ ] **Step 7: Run affected tests**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_corruption.py tests/test_plan_io.py -v`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C <PROJECT> add project/lifecycle.py project/work_health.py project/ctx.py work-end/close_progress.py
git -C <PROJECT> commit -m "refactor(#327): migrate project/ and close_progress to plan_io Refs #327"
```

---

### Task 5: Migrate work-end/ files

**Files:**
- Modify: `work-end/work_end_context.py:59-74` (replace inline dict parser)
- Modify: `work-end/artifact_promote.py:229,255` (replace covers splits)
- Modify: `work-end/branch_recon.py:52` (replace covers split)
- Modify: `work-end/verify_slot_close.py:417` (replace covers split)
- Modify: `work-end/work_end_orchestrator.py:391,1114` (replace covers splits)

**Interfaces:**
- Consumes: `read_plan`, `parse_covers` from Task 1

- [ ] **Step 1: Migrate work_end_context.py**

In `work-end/work_end_context.py`:

Add import (the file already has sys.path setup at line 23):
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import read_plan
```

Replace lines 59-74 (inline dict parser in `check_meta_exists`):
```python
    state = read_plan(target)
    if state is None:
        return {"status": "needs_input", "detail": "no-meta"}
    meta_data = state.fields
```

- [ ] **Step 2: Migrate artifact_promote.py**

Add import:
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 229 — replace:
```python
issue_ref = f"  Refs #{covers.split(',')[0]}" if covers else ""
```
With:
```python
issue_ref = f"  Refs #{parse_covers(covers)[0]}" if covers else ""
```

Line 255 — replace:
```python
issues = [n.strip() for n in covers_str.split(",") if n.strip()]
```
With:
```python
issues = [str(n) for n in parse_covers(covers_str)]
```

- [ ] **Step 3: Migrate branch_recon.py**

Add import:
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 52 — replace:
```python
numbers = [n.strip().lstrip("#") for n in covers.split(",") if n.strip()]
```
With:
```python
numbers = [str(n) for n in parse_covers(covers)]
```

- [ ] **Step 4: Migrate verify_slot_close.py**

Add import:
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 417 — replace:
```python
covers = [int(x) for x in covers_str.split(",") if x.strip()] if covers_str else None
```
With:
```python
covers = parse_covers(covers_str) if covers_str else None
```

- [ ] **Step 5: Migrate work_end_orchestrator.py**

File already has `sys.path.insert(0, str(_project))` at line 783, where `_project` is `Path(__file__).resolve().parent.parent / "project"`. Add import after that:
```python
from plan_io import parse_covers
```

Line 391 — replace:
```python
existing = set(n.strip() for n in covers.split(",") if n.strip())
```
With:
```python
existing = set(parse_covers(covers))
```

Line 1114 — replace:
```python
issue_number=int(covers.split(",")[0]) if covers else 0,
```
With:
```python
issue_number=parse_covers(covers)[0] if covers else 0,
```

- [ ] **Step 6: Run work-end tests**

Run: `python3 -m pytest tests/ -k "work_end or artifact or branch_recon or verify_slot" -v`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add work-end/work_end_context.py work-end/artifact_promote.py work-end/branch_recon.py work-end/verify_slot_close.py work-end/work_end_orchestrator.py
git -C <PROJECT> commit -m "refactor(#327): migrate work-end/ files to plan_io.parse_covers Refs #327"
```

---

### Task 6: Migrate remaining consumers

**Files:**
- Modify: `work-slot/slot_lifecycle.py:235,564,641` (3 covers splits)
- Modify: `scripts/worklog.py:263,401,490` (3 covers splits)
- Modify: `handover/wrap_orchestrator.py:220` (covers split)
- Modify: `work-start/scaffold.py:161` (covers split in duplicate check)

**Interfaces:**
- Consumes: `parse_covers` from Task 1

- [ ] **Step 1: Migrate slot_lifecycle.py**

In `work-slot/slot_lifecycle.py`:

Add import (file already has `sys.path.insert(0, str(_lib))` at line 15 — add project path):
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 235 — replace:
```python
cover_list = [c.strip() for c in covers.split(",") if c.strip()]
```
With:
```python
cover_list = [str(c) for c in parse_covers(covers)]
```

Lines 564, 641 — replace both instances of:
```python
completed = [int(x) for x in covers_str.split(",") if x.strip()]
```
With:
```python
completed = parse_covers(covers_str)
```

- [ ] **Step 2: Migrate worklog.py**

In `scripts/worklog.py`:

Add import:
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Lines 263, 401, 490 — replace all three instances of:
```python
issue_nums = [int(n.strip()) for n in (covers or str(issue_number)).split(",") if n.strip()]
```
With:
```python
issue_nums = parse_covers(covers) if covers else [issue_number]
```

- [ ] **Step 3: Migrate wrap_orchestrator.py**

In `handover/wrap_orchestrator.py`:

Add import (file already has sys.path setup):
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 220 — replace:
```python
issue_number=int(covers.split(",")[0]) if covers else 0,
```
With:
```python
issue_number=parse_covers(covers)[0] if covers else 0,
```

- [ ] **Step 4: Migrate scaffold.py**

In `work-start/scaffold.py`:

Add import:
```python
_project_dir = Path(__file__).resolve().parent.parent / "project"
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import parse_covers
```

Line 161 — replace:
```python
for n in covers.split(","):
    n = n.strip()
    if n.isdigit():
```
With:
```python
for n in parse_covers(covers):
```

Adjust the loop body to use `n` as an `int` (it already calls `int(n)` later — now unnecessary since `parse_covers` returns ints).

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest tests/ -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add work-slot/slot_lifecycle.py scripts/worklog.py handover/wrap_orchestrator.py work-start/scaffold.py
git -C <PROJECT> commit -m "refactor(#327): migrate remaining consumers to plan_io.parse_covers Refs #327"
```

---

## References

- [2026-09-03-plan-io-unification-design.md] — design spec this plan implements
- [project/corruption.py] — 13 field-reader copies, inline queue regex (primary #327 bug site)
- [work-slot/plan_manager.py:103-105] — canonical `_ITEM_RE` regex
- [project/lifecycle.py:167-330] — 4 section parser copies (read_state, write_state, write_branch, _read_branch)
- [project/work_health.py:76-89,250-262] — 2 inline section parsers
- [externalised-scripts-require-tests] — protocol: scripts ship with tests
- [evidence-before-claims] — protocol: verify before claiming done
- [GitHub #327] — focal issue

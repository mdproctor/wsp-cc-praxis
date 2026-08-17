# Unify Queue Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #238 — Unify queue tracking: .plan always, kill .epic, drop issue: from .meta

**Goal:** Merge `.meta` and `.plan` into a single unified `.plan` file. One file owns all branch state — identity, lifecycle, and work queue. Kill `.epic`. Migrate existing branches on read.

**Architecture:** `.meta` and `.plan` are always created together, deleted together, live in the same directory, and no consumer reads one without the other. Two files means two parsers, two paths to resolve, two places for state to diverge. Merge them into a single `.plan` with a `## State` section (replacing `.meta`) and a `## Queue` section (existing). `## Session State` is absorbed into `## State`. `.epic` is removed entirely — `.plan` already handles nested epics. `ctx.py` output contract changes: `PLAN_ACTIVE_ISSUE` + `EPIC_ACTIVE_ISSUE` collapse to `ACTIVE_ISSUE`; all `EPIC_*` fields removed; `ISSUE_N` derived from `covers:` in `## State`.

**Tech Stack:** Python 3, pytest, skill markdown

## Global Constraints

- All changes must pass `python3 -m pytest tests/ -v`
- `ctx.py` output is a public contract — all skill markdown consuming renamed fields must be updated
- Migration-on-read must handle three scenarios: `.meta` only, `.meta` + old `.plan`, `.meta` + `.epic`
- The unified `.plan` format is detected by the presence of `## State` section
- `lifecycle.py` functions (`read_state`, `write_state`, `commit_transition`) change from operating on `.meta` to operating on `.plan`'s `## State` section — same key-value format, different file

## Unified `.plan` format

```markdown
# Work Plan — issue-42-fix-auth

## State
branch: issue-42-fix-auth
state: active
project-sha: abc123
date: 2026-08-14
issue-repo: Hortora/soredium
covers: 42
design-repo: workspace
design-section-hashes:
flyway-next-v: unknown
last-wrap:

## Queue
- [ ] #42 — Fix auth ← active

## Deferred
(only present if deferred items exist)
```

**Section responsibilities:**
- `## State` — branch identity + lifecycle state (was `.meta` + `## Session State`)
- `## Queue` — ordered work items with `← active` marker (unchanged)
- `## Deferred` — parked items with scale/complexity/repos (unchanged)

**What collapsed:**
- `## Session State`'s `Started:` → `date:` in `## State` (same value at creation time)
- `## Session State`'s `Current: #N` → dropped (derived from `← active` marker)
- `## Session State`'s `Last wrap:` → `last-wrap:` in `## State`
- `.meta`'s `issue:` → dropped (queue position is `← active` in `## Queue`)
- `.meta`'s `plan: yes` → dropped (dead field — never read)

---

### Task 1: Extend plan_manager to parse/write `## State` section

Add state awareness to the existing plan parser. The `## State` section uses the same key-value format as `.meta` — `key: value` lines. `PlanTree` gains a `state` dict. `build_plan_content()` writes `## State` before `## Queue`. `## Session State` is removed.

**Files:**
- Modify: `work-slot/plan_manager.py:131-191` — `parse_plan()` to read `## State`
- Modify: `work-slot/plan_manager.py:314-347` — `build_plan_content()` to write `## State`
- Modify: `work-slot/plan_manager.py:36-44` — `PlanTree` dataclass
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Produces: `PlanTree.state: dict[str, str]` — key-value pairs from `## State`
- Produces: `build_plan_content()` accepts `state: dict[str, str]` parameter
- Produces: `parse_plan()` returns `PlanTree` with populated `state` dict

- [ ] **Step 1: Write failing test — parse_plan reads ## State**

```python
def test_parse_plan_reads_state_section(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text(
        "# Work Plan — issue-42-fix-auth\n\n"
        "## State\n"
        "branch: issue-42-fix-auth\n"
        "state: active\n"
        "date: 2026-08-14\n"
        "covers: 42\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix auth ← active\n"
    )
    tree = parse_plan(plan)
    assert tree.state["branch"] == "issue-42-fix-auth"
    assert tree.state["state"] == "active"
    assert tree.state["covers"] == "42"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_plan_manager.py::test_parse_plan_reads_state_section -v`
Expected: FAIL — `PlanTree` has no `state` attribute

- [ ] **Step 3: Add `state` to PlanTree**

```python
@dataclass
class PlanTree:
    heading: str
    queue: list[QueueItem]
    current_issue: int | None
    started: str
    last_wrap: str | None = None
    deferred: list[DeferredItem] = field(default_factory=list)
    state: dict[str, str] = field(default_factory=dict)
```

- [ ] **Step 4: Parse `## State` in parse_plan()**

Add a `in_state` flag alongside `in_queue`, `in_deferred`, `in_session`. When in the `## State` section, parse `key: value` lines into `tree.state`:

```python
if line.strip() == "## State":
    in_state = True
    in_queue = False
    in_deferred = False
    in_session = False
    continue
```

And in the section body:

```python
if in_state:
    stripped = line.strip()
    if ':' in stripped:
        k, _, v = stripped.partition(':')
        state_dict[k.strip()] = v.strip()
```

Also migrate `started` and `last_wrap` to come from `state` when present:
- `started = state_dict.get("date", "")` if `Started:` not found in `## Session State`
- `last_wrap = state_dict.get("last-wrap")` if `Last wrap:` not found

- [ ] **Step 5: Update build_plan_content() to write ## State**

Add a `state` parameter. Write `## State` section before `## Queue`:

```python
def build_plan_content(branch_slug: str, items: list[QueueItem], date: str,
                       last_wrap: str | None = None,
                       deferred: list[DeferredItem] | None = None,
                       state: dict[str, str] | None = None) -> str:
    lines = [f"# Work Plan — {branch_slug}"]

    if state:
        lines.append("")
        lines.append("## State")
        for k, v in state.items():
            lines.append(f"{k}: {v}")

    lines.extend(["", "## Queue"])
    # ... rest unchanged ...
```

Remove the `## Session State` section generation. `Current: #N` is dropped (redundant with `← active`). `Started:` and `Last wrap:` move to `## State` as `date:` and `last-wrap:`.

- [ ] **Step 6: Write test for roundtrip — parse then rewrite preserves state**

```python
def test_plan_roundtrip_preserves_state(tmp_path):
    state = {"branch": "issue-42", "state": "active", "date": "2026-08-14", "covers": "42"}
    items = [QueueItem(issue_number=42, title="Fix auth", active=True)]
    content = build_plan_content("issue-42", items, "2026-08-14", state=state)
    plan = tmp_path / ".plan"
    plan.write_text(content)
    tree = parse_plan(plan)
    assert tree.state == state
    assert len(tree.queue) == 1
    assert tree.queue[0].active
```

- [ ] **Step 7: Update rewrite_plan() to preserve state**

`rewrite_plan()` calls `build_plan_content()`. Pass `tree.state` through:

```python
def rewrite_plan(plan_path: Path, tree: PlanTree) -> None:
    content = build_plan_content(
        tree.heading.replace("Work Plan — ", ""),
        tree.queue,
        tree.started,
        last_wrap=tree.last_wrap,
        deferred=tree.deferred,
        state=tree.state,
    )
    # Atomic write — .plan now holds lifecycle state + queue
    tmp_path = plan_path.parent / '.plan.tmp'
    tmp_path.write_text(content)
    tmp_path.replace(plan_path)
```

- [ ] **Step 8: Update detect() to return state fields**

The `detect()` function (line 606) returns a dict. Add state fields:

```python
return {
    "has_plan": True,
    "plan_path": str(plan_path),
    "active_issue": active.issue_number if active else None,
    "active_title": active.title if active else None,
    "completed_count": completed_count,
    "total_count": total_count,
    "current_batch": batch,
    "state": tree.state,  # new: full state dict
}
```

- [ ] **Step 9: Ensure backward compat — parse old .plan without ## State**

Write a test that parses an old-format `.plan` (no `## State`, has `## Session State`):

```python
def test_parse_old_plan_without_state_section(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text(
        "# Work Plan — issue-42\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix auth ← active\n\n"
        "## Session State\n"
        "Current: #42 — Fix auth\n"
        "Started: 2026-08-14\n"
    )
    tree = parse_plan(plan)
    assert tree.state == {}  # no ## State section
    assert tree.started == "2026-08-14"
    assert len(tree.queue) == 1
```

- [ ] **Step 10: Run all plan_manager tests**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add work-slot/plan_manager.py tests/test_plan_manager.py
git commit -m "feat(#238): plan_manager parses/writes ## State section — foundation for unified .plan"
```

---

### Task 2: Scaffold creates unified `.plan` — no more `.meta`

The scaffold currently creates `.meta` and optionally `.plan`. Change it to create a single unified `.plan` with `## State` + `## Queue`. No `.meta` is written. `ScaffoldResult` drops `meta_path` — only `plan_path` is returned.

**Files:**
- Modify: `work-start/scaffold.py:49-127` — `scaffold()` function, `ScaffoldResult`
- Modify: `work-start/scaffold.py:143-213` — CLI entry point output
- Modify: `work-start/branch_create.py:98` — commit-scaffold file list
- Test: `tests/test_scaffold_api.py`
- Test: `tests/test_scaffold.py`

**Interfaces:**
- Consumes: Task 1's `build_plan_content()` with `state` parameter
- Produces: `ScaffoldResult` has `plan_path: str` (always set), no `meta_path`
- Produces: CLI outputs `PLAN_PATH=` instead of `META_PATH=`

- [ ] **Step 1: Write failing test — scaffold creates unified .plan**

```python
def test_scaffold_creates_unified_plan(tmp_path):
    result = scaffold(
        workspace=tmp_path,
        branch="issue-42-fix-auth",
        project_sha="abc123",
        issue="42",
        issue_repo="Hortora/soredium",
        today="2026-08-14",
    )
    assert result.plan_path is not None
    plan = Path(result.plan_path)
    assert plan.exists()
    content = plan.read_text()
    # Has ## State section with identity fields
    assert "## State" in content
    assert "branch: issue-42-fix-auth" in content
    assert "state: scaffolded" in content
    assert "covers: 42" in content
    # Has ## Queue section with active issue
    assert "## Queue" in content
    assert "← active" in content
    assert "#42" in content
    # No separate .meta
    assert not (tmp_path / "design" / ".meta").exists()
```

- [ ] **Step 2: Write test — scaffold without issue creates empty queue**

```python
def test_scaffold_without_issue_creates_empty_queue(tmp_path):
    result = scaffold(
        workspace=tmp_path,
        branch="spike-explore",
        project_sha="abc123",
        today="2026-08-14",
    )
    plan = Path(result.plan_path)
    content = plan.read_text()
    assert "## State" in content
    assert "branch: spike-explore" in content
    assert "(empty" in content
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_scaffold_api.py::test_scaffold_creates_unified_plan tests/test_scaffold_api.py::test_scaffold_without_issue_creates_empty_queue -v`
Expected: FAIL

- [ ] **Step 4: Implement unified scaffold**

Rewrite `scaffold()` in `work-start/scaffold.py`:

```python
@dataclass
class ScaffoldResult:
    plan_path: str
    journal_path: str
    created: bool


def scaffold(workspace: Path, branch: str, project_sha: str,
             issue: str = "", issue_repo: str = "", covers: str = "",
             today: str | None = None, flyway_next_v: str = "unknown",
             design_repo: str = "project",
             design_section_hashes: str = "",
             plan_content: str = "",
             force: bool = False) -> ScaffoldResult:
    design_dir = workspace / "design"
    design_dir.mkdir(parents=True, exist_ok=True)

    plan_path = design_dir / ".plan"
    journal_path = design_dir / "JOURNAL.md"

    if not force and plan_path.exists() and journal_path.exists():
        return ScaffoldResult(
            plan_path=str(plan_path),
            journal_path=str(journal_path),
            created=False,
        )

    if today is None:
        from datetime import date
        today = date.today().isoformat()

    if not covers:
        covers = issue

    # Build unified .plan
    if plan_content:
        plan_path.write_text(plan_content)
    else:
        state = {
            "branch": branch,
            "state": "scaffolded",
            "project-sha": project_sha,
            "date": today,
            "issue-repo": issue_repo,
            "covers": covers,
            "design-repo": design_repo,
            "design-section-hashes": design_section_hashes,
            "flyway-next-v": flyway_next_v,
        }

        _slot_dir = str(Path(__file__).parent.parent / "work-slot")
        if _slot_dir not in sys.path:
            sys.path.insert(0, _slot_dir)
        from plan_manager import build_plan_content, QueueItem

        if issue:
            items = [QueueItem(issue_number=int(issue),
                               title=f"Issue #{issue}", active=True)]
        else:
            items = []
        plan_path.write_text(build_plan_content(branch, items, today, state=state))

    journal_path.write_text(f"# Design Journal — {branch}\n")

    return ScaffoldResult(
        plan_path=str(plan_path),
        journal_path=str(journal_path),
        created=True,
    )
```

Drop the `plan` and `plan_content` params (now `plan_content` only — `.plan` is always created). Remove `meta_path` from `ScaffoldResult`. Remove the old `.meta` writing code.

- [ ] **Step 5: Update CLI output**

Change `print(f"META_PATH={result.meta_path}")` to `print(f"PLAN_PATH={result.plan_path}")`. (The `PLAN_PATH` line may already exist — deduplicate.)

- [ ] **Step 6: Update branch_create.py commit-scaffold**

In `work-start/branch_create.py:98`, change the staged files list from `["design/JOURNAL.md", "design/.meta"]` to `["design/JOURNAL.md", "design/.plan"]`.

- [ ] **Step 7: Fix existing scaffold tests**

Update all tests in `test_scaffold_api.py` and `test_scaffold.py`:
- Replace `result.meta_path` with reading state from `result.plan_path`
- Remove `plan=True` parameters (no longer needed)
- Assert `.plan` exists instead of `.meta`

- [ ] **Step 8: Update worklog integration in scaffold**

The `main()` function (lines 186-207) calls `worklog.record_work_start()` and `worklog.record_issue_activate()`. These don't depend on `.meta` — they use params directly. No change needed, just verify they still work.

- [ ] **Step 9: Run all scaffold tests**

Run: `python3 -m pytest tests/test_scaffold_api.py tests/test_scaffold.py -v`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add work-start/scaffold.py work-start/branch_create.py tests/test_scaffold_api.py tests/test_scaffold.py
git commit -m "feat(#238): scaffold creates unified .plan — no more .meta"
```

---

### Task 3: Migration-on-read — detect old format and merge

When `work_state.detect()` encounters old-format files (`.meta` exists), transparently migrate to the unified `.plan` format. Three scenarios:

1. **`.meta` + old `.plan`** — merge `.meta` fields into `.plan`'s new `## State` section, delete `.meta`
2. **`.meta` alone** — create unified `.plan` from `.meta` fields + single-entry queue
3. **`.meta` + `.epic`** — convert `.epic` queue to `.plan` format, merge `.meta`, delete both old files

Migration is atomic: write new `.plan`, then delete old files. If migration fails, old files remain untouched.

**Files:**
- Create: `work-slot/plan_migrate.py` — migration function
- Modify: `project/work_state.py` — call migration before plan detection
- Test: `tests/test_plan_migrate.py` (new)

**Interfaces:**
- Consumes: Task 1's `parse_plan()`, `build_plan_content()`, `PlanTree`
- Produces: `migrate_if_needed(topo: Topology) -> bool` — returns True if migration occurred

- [ ] **Step 1: Write failing test — migrate .meta + old .plan**

```python
def test_migrate_meta_plus_old_plan(tmp_path):
    design = tmp_path / "design"
    design.mkdir()
    meta = design / ".meta"
    meta.write_text(
        "branch: issue-42-fix-auth\n"
        "state: active\n"
        "project-sha: abc123\n"
        "date: 2026-08-14\n"
        "issue: 42\n"
        "issue-repo: Hortora/soredium\n"
        "covers: 42\n"
        "design-repo: workspace\n"
    )
    plan = design / ".plan"
    plan.write_text(
        "# Work Plan — issue-42-fix-auth\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix auth ← active\n\n"
        "## Session State\n"
        "Current: #42 — Fix auth\n"
        "Started: 2026-08-14\n"
    )
    result = migrate_if_needed(design)
    assert result is True
    assert not meta.exists()  # .meta deleted
    assert plan.exists()
    content = plan.read_text()
    assert "## State" in content
    assert "branch: issue-42-fix-auth" in content
    assert "state: active" in content
    assert "## Queue" in content
    assert "← active" in content
```

- [ ] **Step 2: Write test — migrate .meta alone (no .plan)**

```python
def test_migrate_meta_alone(tmp_path):
    design = tmp_path / "design"
    design.mkdir()
    meta = design / ".meta"
    meta.write_text(
        "branch: issue-99-solo\n"
        "state: active\n"
        "project-sha: def456\n"
        "date: 2026-08-10\n"
        "issue: 99\n"
        "issue-repo: Hortora/soredium\n"
        "covers: 99\n"
    )
    result = migrate_if_needed(design)
    assert result is True
    assert not meta.exists()
    plan = design / ".plan"
    assert plan.exists()
    content = plan.read_text()
    assert "## State" in content
    assert "## Queue" in content
    assert "#99" in content
    assert "← active" in content
```

- [ ] **Step 2b: Write test — migrate multi-issue .meta alone**

```python
def test_migrate_multi_issue_meta_alone(tmp_path):
    design = tmp_path / "design"
    design.mkdir()
    meta = design / ".meta"
    meta.write_text(
        "branch: issue-42-batch\n"
        "state: active\n"
        "project-sha: abc123\n"
        "date: 2026-08-10\n"
        "issue: 42\n"
        "issue-repo: Hortora/soredium\n"
        "covers: 42,43,44\n"
    )
    result = migrate_if_needed(design)
    assert result is True
    plan = design / ".plan"
    content = plan.read_text()
    assert "#42" in content
    assert "#43" in content
    assert "#44" in content
    # First issue is active
    assert "← active" in content
```

- [ ] **Step 3: Write test — no migration needed (already unified)**

```python
def test_no_migration_when_unified(tmp_path):
    design = tmp_path / "design"
    design.mkdir()
    plan = design / ".plan"
    plan.write_text(
        "# Work Plan — issue-42\n\n"
        "## State\n"
        "branch: issue-42\n"
        "state: active\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix ← active\n"
    )
    result = migrate_if_needed(design)
    assert result is False  # no migration needed
```

- [ ] **Step 4: Write test — no migration on main (no .meta, no .plan)**

```python
def test_no_migration_when_nothing_exists(tmp_path):
    design = tmp_path / "design"
    design.mkdir()
    result = migrate_if_needed(design)
    assert result is False
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_migrate.py -v`
Expected: FAIL — `migrate_if_needed` doesn't exist

- [ ] **Step 6: Implement plan_migrate.py**

```python
"""
plan_migrate.py — one-time migration from .meta (+ optional old .plan/.epic) to unified .plan.

Called by work_state.detect() before plan detection. If .meta exists alongside
.plan without ## State, merges .meta into .plan and deletes .meta. If .meta
exists alone, creates a unified .plan from covers: field.

epic parsing is inlined (not imported from epic_manager.py) so migration
survives after epic_manager.py is deleted in Task 6.
"""
import re
import sys
from pathlib import Path

_slot_dir = str(Path(__file__).parent)
if _slot_dir not in sys.path:
    sys.path.insert(0, _slot_dir)


def _parse_meta(meta_path: Path) -> dict[str, str]:
    fields = {}
    for line in meta_path.read_text().splitlines():
        if ':' in line:
            k, _, v = line.partition(':')
            fields[k.strip()] = v.strip()
    return fields


def _has_state_section(plan_path: Path) -> bool:
    for line in plan_path.read_text().splitlines():
        if line.strip() == "## State":
            return True
    return False


def _queue_items_from_covers(covers: str):
    """Build QueueItem list from comma-separated covers string."""
    from plan_manager import QueueItem
    items = []
    for i, part in enumerate(covers.split(",")):
        num = part.strip()
        if num and num.isdigit():
            items.append(QueueItem(
                issue_number=int(num),
                title=f"Issue #{num}",
                active=(i == 0),
            ))
    return items


_EPIC_ITEM_RE = re.compile(
    r'^(\s*)- \[([ x])\] #(\d+)\s*—\s*(.+?)(?:\s*←\s*(?:active|current))?$'
)


def _queue_items_from_epic(epic_path: Path):
    """Inline epic parser — extracts queue items without importing epic_manager."""
    from plan_manager import QueueItem
    items = []
    first_uncompleted = True
    for line in epic_path.read_text().splitlines():
        m = _EPIC_ITEM_RE.match(line)
        if m:
            completed = m.group(2) == "x"
            issue_num = int(m.group(3))
            title = m.group(4).strip()
            active = not completed and first_uncompleted
            if active:
                first_uncompleted = False
            items.append(QueueItem(issue_num, title, completed=completed, active=active))
    return items


def migrate_if_needed(design_dir: Path) -> bool:
    meta_path = design_dir / ".meta"
    plan_path = design_dir / ".plan"
    epic_path = design_dir / ".epic"

    if not meta_path.exists():
        return False

    if plan_path.exists() and _has_state_section(plan_path):
        meta_path.unlink()
        return True

    meta = _parse_meta(meta_path)
    meta.pop("issue", None)
    meta.pop("plan", None)

    from plan_manager import parse_plan, build_plan_content, rewrite_plan

    if plan_path.exists():
        tree = parse_plan(plan_path)
        tree.state = meta
        if tree.started and "date" not in meta:
            tree.state["date"] = tree.started
        if tree.last_wrap:
            tree.state["last-wrap"] = tree.last_wrap
        rewrite_plan(plan_path, tree)
    elif epic_path.exists():
        items = _queue_items_from_epic(epic_path)
        if not items:
            items = _queue_items_from_covers(meta.get("covers", ""))
        content = build_plan_content(
            meta.get("branch", "migrated"), items,
            meta.get("date", ""), state=meta)
        plan_path.write_text(content)
        epic_path.unlink()
    else:
        items = _queue_items_from_covers(meta.get("covers", ""))
        content = build_plan_content(
            meta.get("branch", "migrated"), items,
            meta.get("date", ""), state=meta)
        plan_path.write_text(content)

    meta_path.unlink()
    return True
```

- [ ] **Step 7: Wire migration into work_state.detect()**

In `project/work_state.py`, before plan detection (line 68), call migration:

```python
# Migration — convert old .meta to unified .plan
from plan_migrate import migrate_if_needed
meta_file = find_design_file(".meta", topo)
if meta_file:
    migrate_if_needed(meta_file.parent)
```

- [ ] **Step 8: Run migration tests**

Run: `python3 -m pytest tests/test_plan_migrate.py -v`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add work-slot/plan_migrate.py project/work_state.py tests/test_plan_migrate.py
git commit -m "feat(#238): migration-on-read — old .meta transparently merged into unified .plan"
```

---

### Task 4: Update lifecycle.py — read/write state from `.plan`

`lifecycle.py` functions (`read_state`, `write_state`, `_read_branch`, `commit_transition`) currently operate on `.meta`. Change them to operate on `.plan`'s `## State` section. The key-value format is identical — the only difference is finding the right section in the file.

**Files:**
- Modify: `project/lifecycle.py:139-179` — `read_state()`, `write_state()`
- Modify: `project/lifecycle.py:235-242` — `_read_branch()`
- Modify: `project/lifecycle.py:291-318` — `commit_transition()`
- Modify: `project/lifecycle.py:87-108` — transition table (remove `update_meta` effect, rename `write_meta` to `write_plan`)
- Test: `tests/test_lifecycle.py` — ~40 fixture updates (`.meta` → `.plan` with `## State`)

**Interfaces:**
- Consumes: Task 1's unified `.plan` format
- Produces: `read_state(plan_path)` reads `state:` from `## State` section
- Produces: `write_state(plan_path, state)` writes `state:` within `## State` section
- Produces: transition table effect `write_meta` renamed to `write_plan`; `update_meta` removed

- [ ] **Step 1: Write failing test — read_state from .plan**

```python
def test_read_state_from_plan(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text(
        "# Work Plan — test\n\n"
        "## State\n"
        "branch: test\n"
        "state: active\n"
        "date: 2026-08-14\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix ← active\n"
    )
    assert read_state(plan) == "active"
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `read_state` reads the whole file, finds `state:` anywhere, but we need it to work with `.plan` format. Actually `read_state` does a simple line scan for `state:` — it would find it in `.plan` too since the line format is identical. But we need to verify it doesn't accidentally match `## Session State` content. Write a more specific test:

```python
def test_read_state_ignores_non_state_section(tmp_path):
    plan = tmp_path / ".plan"
    plan.write_text(
        "# Work Plan\n\n"
        "## State\n"
        "branch: test\n"
        "state: closing:review\n\n"
        "## Queue\n"
        "- [ ] #42 — Fix ← active\n"
    )
    assert read_state(plan) == "closing:review"
```

- [ ] **Step 3: Update read_state**

The current `read_state` scans all lines for `state:`. Scope it to `## State` section to avoid matching queue items:

```python
def read_state(plan_path: Path) -> Optional[str]:
    """Read lifecycle state from .plan's ## State section.
    Returns None if file doesn't exist.
    Raises CorruptedState if state: field has unrecognised value."""
    if not plan_path.exists():
        return None
    in_state_section = False
    for line in plan_path.read_text().splitlines():
        if line.strip() == '## State':
            in_state_section = True
            continue
        if line.startswith('## '):
            in_state_section = False
            continue
        if in_state_section and line.startswith('state:'):
            raw = line.split(':', 1)[1].strip()
            if raw in VALID_STATES:
                return raw
            raise CorruptedState(plan_path, raw)
    return 'active'
```

Parameter rename: `meta_path` → `plan_path` throughout `lifecycle.py`. Also scope `_read_branch` the same way (read `branch:` only from `## State` section).

- [ ] **Step 4: Update write_state for .plan**

The current `write_state` scans all lines for `state:`. In `.meta` this was safe (only key-value lines). In `.plan`, a queue item titled "state: machine refactor" would match. Scope the scan to lines between `## State` and the next `##`:

```python
def write_state(plan_path: Path, state: str) -> None:
    content = plan_path.read_text()
    lines = content.splitlines()

    in_state_section = False
    state_line_idx = None
    for i, line in enumerate(lines):
        if line.strip() == '## State':
            in_state_section = True
            continue
        if line.startswith('## '):
            in_state_section = False
            continue
        if in_state_section and line.startswith('state:'):
            state_line_idx = i
            break

    if state_line_idx is not None:
        lines[state_line_idx] = f'state: {state}'
    else:
        # Fallback: insert after branch: line
        for i, line in enumerate(lines):
            if line.startswith('branch:'):
                lines.insert(i + 1, f'state: {state}')
                break

    tmp_path = plan_path.parent / '.plan.tmp'
    tmp_path.write_text('\n'.join(lines) + '\n')
    tmp_path.replace(plan_path)
```

- [ ] **Step 5: Update _read_branch**

Same pattern — rename parameter, same logic:

```python
def _read_branch(plan_path: Path) -> Optional[str]:
    if not plan_path.exists():
        return None
    for line in plan_path.read_text().splitlines():
        if line.startswith('branch:'):
            return line.split(':', 1)[1].strip()
    return None
```

- [ ] **Step 6: Update commit_transition**

Rename `meta_path` → `plan_path` parameter. All internal calls already use `read_state` and `write_state` which are updated.

- [ ] **Step 7: Update transition table**

```python
('idle', 'work'):        ('scaffolded', ['create_branch', 'write_plan', 'build_plan'], []),
('idle', 'slot_create'): ('scaffolded', ['create_slot', 'write_plan', 'build_plan'], []),
('active', 'work_next'): ('transitioning', ['advance_issue', 'tick_github'], []),
```

Changes:
- `write_meta` → `write_plan` (scaffolding now writes `.plan`)
- `update_meta` removed from `work_next` effects (no longer needed — `.plan` `← active` marker is the source of truth, updated by `advance_issue`)

- [ ] **Step 8: Fix test_lifecycle.py fixtures**

All ~40 test fixtures create `.meta` files. Update each to create `.plan` files with `## State` section instead. The test logic is the same — only the file format changes.

Pattern for each test:
```python
# Before:
meta = tmp_path / ".meta"
meta.write_text("branch: test\nstate: active\n")

# After:
plan = tmp_path / ".plan"
plan.write_text(
    "# Work Plan — test\n\n"
    "## State\n"
    "branch: test\n"
    "state: active\n\n"
    "## Queue\n"
    "(empty)\n"
)
```

Create a test helper to reduce boilerplate:
```python
def _write_plan(path: Path, state: str = "active", branch: str = "test", **extra):
    lines = ["# Work Plan — test", "", "## State",
             f"branch: {branch}", f"state: {state}"]
    for k, v in extra.items():
        lines.append(f"{k}: {v}")
    lines.extend(["", "## Queue", "(empty)", ""])
    path.write_text("\n".join(lines))
```

- [ ] **Step 9: Run lifecycle tests**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle.py
git commit -m "refactor(#238): lifecycle reads/writes state from unified .plan — no more .meta"
```

---

### Task 5: Update ctx.py and work_state.py — unified output

ctx.py reads identity fields from `.meta` via `_parse_meta()`. Change it to read from `.plan`'s `## State` section via `plan_manager.detect()` (which now returns state fields from Task 1). Unify output fields.

**Files:**
- Modify: `project/ctx.py:30-42` — remove `_parse_meta()` (replaced by plan state)
- Modify: `project/ctx.py:103-114` — read identity from plan state, not .meta
- Modify: `project/ctx.py:183-246` — output dict renames
- Modify: `project/work_state.py` — remove epic fields, rename active_issue, remove `.meta` lookup
- Modify: `work-end/work_end_context.py:53-58` — read from .plan not .meta
- Modify: `work-end/branch_cleanup.py:54-59` — remove .meta from cleanup (replace with .plan)
- Modify: `git-squash/ctx.py:69-72` — read from .plan not .meta
- Test: `tests/test_ctx.py`
- Test: `tests/test_work_state.py`
- Test: `tests/test_work_end_context.py`

**Interfaces:**
- Consumes: Task 1's `detect()` returning `state` dict
- Consumes: Task 4's lifecycle functions taking `plan_path`
- Produces: `ACTIVE_ISSUE` replaces `PLAN_ACTIVE_ISSUE` + `EPIC_ACTIVE_ISSUE`
- Produces: `ISSUE_N` derived from `covers:` first entry in `## State`
- Produces: `IS_EPIC`, `EPIC_PATH`, `EPIC_BATCH`, `EPIC_ACTIVE_ISSUE` removed
- Produces: `META_PATH` removed from ctx output (consumers use `PLAN_PATH`)

- [ ] **Step 1: Write failing test**

```python
def test_ctx_unified_output(workspace_on_branch):
    result = resolve(cwd=str(workspace_on_branch))
    assert "ACTIVE_ISSUE" in result
    assert "PLAN_ACTIVE_ISSUE" not in result
    assert "EPIC_ACTIVE_ISSUE" not in result
    assert "IS_EPIC" not in result
```

- [ ] **Step 2: Update work_state.py**

Remove `is_epic`, `epic_path`, `epic_batch`, `epic_active_issue` from `WorkState`. Rename `plan_active_issue` → `active_issue`. Remove the entire epic detection block (lines 89-106). Remove `from epic_manager import detect` import.

- [ ] **Step 3: Update ctx.py — read identity from .plan state**

Replace the `.meta` parsing block (lines 103-114) with plan state reading:

```python
# Identity — from .plan's ## State section (via plan_manager.detect)
plan_state = {}
if state.has_plan and state.plan_path:
    from plan_manager import detect as _plan_detect
    plan_dir = Path(state.plan_path).parent
    detect_base = plan_dir.parent if plan_dir.name == "design" else plan_dir
    plan_info = _plan_detect(detect_base)
    if plan_info:
        plan_state = plan_info.get("state", {})

branch_name = plan_state.get("branch", "")
project_sha = plan_state.get("project-sha", "")
covers = plan_state.get("covers", "")
issue_n = covers.split(",")[0].strip() if covers else ""
issue_repo = plan_state.get("issue-repo", owner_repo)
design_repo_key = plan_state.get("design-repo", "")
flyway_next_v = plan_state.get("flyway-next-v", "")
meta_section_hashes = plan_state.get("design-section-hashes", "")
has_meta = "yes" if plan_state else "no"  # backward compat field name
```

Also handle the fallback case when no `.plan` exists (on main, or in unexpected states):

```python
# Fallback to .meta if it still exists (pre-migration branch)
if not plan_state:
    meta_path = find_design_file(".meta", topo)
    if meta_path:
        meta = _parse_meta(meta_path)
        branch_name = meta.get("branch", "")
        # ... etc
```

Keep `_parse_meta()` for the fallback path — it gets exercised less over time but prevents hard breakage.

- [ ] **Step 4: Update ctx.py output dict**

```python
"ACTIVE_ISSUE": state.active_issue,  # was PLAN_ACTIVE_ISSUE
# Remove: IS_EPIC, EPIC_PATH, EPIC_BATCH, EPIC_ACTIVE_ISSUE
# ISSUE_N now from covers: first entry
"ISSUE_N": issue_n,
```

- [ ] **Step 5: Update work_end_context.py**

Change `.meta` reading (line 53-58) to read from `.plan`'s `## State` section. Same key-value parsing, different file path.

- [ ] **Step 6: Update branch_cleanup.py**

Change `.meta` removal (lines 54-59) to `.plan` removal. (Or keep `.plan` — work-end may want to preserve it. Check what the cleanup intent is.)

Actually, `branch_cleanup.py` removes `design/.meta` and `design/JOURNAL.md` at branch close to prevent stale state. After the merge, it should remove `design/.plan` and `design/JOURNAL.md` instead.

- [ ] **Step 7: Update git-squash/ctx.py**

Change `.meta` reading (lines 69-72) to read from `.plan`'s `## State` section.

- [ ] **Step 8: Fix test_ctx.py**

Update all `PLAN_ACTIVE_ISSUE` → `ACTIVE_ISSUE`, remove `EPIC_*` assertions, update `ISSUE_N` expectations, update fixtures that create `.meta` to create unified `.plan` instead.

- [ ] **Step 9: Run tests**

Run: `python3 -m pytest tests/test_ctx.py tests/test_work_state.py tests/test_work_end_context.py -v`
Expected: ALL PASS

- [ ] **Step 10: Run full test suite**

Run: `python3 -m pytest tests/ -v`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add project/ctx.py project/work_state.py work-end/work_end_context.py work-end/branch_cleanup.py git-squash/ctx.py tests/
git commit -m "refactor(#238): ctx.py reads identity from unified .plan — ACTIVE_ISSUE replaces PLAN_ACTIVE_ISSUE"
```

---

### Task 6: Kill `.epic` and clean advance signatures

Remove `epic_manager.py`, its tests, and clean the `advance_issue()` signature.

**Files:**
- Delete: `work-slot/epic_manager.py` (531 lines)
- Delete: `tests/test_epic_manager.py` (709 lines)
- Modify: `work-slot/plan_manager.py:456-466` — simplify `advance_issue()`, drop `epic_path` param
- Modify: `work-slot/plan_manager.py:392` — drop `meta_path` from `advance()` signature (no more `_update_meta_issue`)
- Modify: `work-slot/plan_manager.py:469` — drop `meta_path` from `complete_active_issue()`

**Interfaces:**
- Produces: `advance(plan_path, repo_path=None)` — no `meta_path` (state changes happen via lifecycle, not advance)
- Produces: `advance_issue(plan_path, repo_path=None)` — no `epic_path`, no `meta_path`
- Produces: `complete_active_issue(plan_path, repo_path)` — no `meta_path`

- [ ] **Step 1: Update advance() signature**

Remove `meta_path` parameter. The `_update_meta_issue()` call was already the only use of it (removed conceptually in Task 4). The `_emit_issue_events()` call reads branch/issue-repo from `.plan`'s `## State` via `_read_meta_fields()` — update this to read from plan's state section instead:

```python
def _read_plan_state(plan_path: Path) -> dict[str, str]:
    """Read ## State section key-values from unified .plan."""
    fields = {}
    in_state = False
    for line in plan_path.read_text().splitlines():
        if line.strip() == "## State":
            in_state = True
            continue
        if line.startswith("## "):
            in_state = False
            continue
        if in_state and ':' in line:
            k, _, v = line.partition(':')
            fields[k.strip()] = v.strip()
    return fields


def advance(plan_path: Path, repo_path: str | None = None) -> AdvanceResult:
    # ... existing logic, minus _update_meta_issue() call ...
    if repo_path:
        try:
            fields = _read_plan_state(plan_path)
            # _emit_issue_events now reads from plan state
```

- [ ] **Step 2: Simplify advance_issue()**

```python
def advance_issue(plan_path: Path,
                  repo_path: str | None = None) -> AdvanceResult:
    if plan_path and plan_path.exists():
        return advance(plan_path, repo_path=repo_path)
    raise NoQueueFile("No .plan found")
```

- [ ] **Step 3: Simplify complete_active_issue()**

```python
def complete_active_issue(plan_path: Path,
                          repo_path: str) -> int | None:
    tree = parse_plan(plan_path)
    active = _find_active_leaf(tree.queue)
    if not active:
        return None
    fields = _read_plan_state(plan_path)
    _emit_issue_events_from_fields(fields, repo_path, active.issue_number, next_issue=None)
    return active.issue_number
```

- [ ] **Step 4: Delete epic_manager.py and test_epic_manager.py**

```bash
git rm work-slot/epic_manager.py tests/test_epic_manager.py
```

- [ ] **Step 5: Update plan_manager tests for new signatures**

Fix all calls to `advance()`, `advance_issue()`, `complete_active_issue()` in `tests/test_plan_manager.py` to use the new signatures (no `meta_path`, no `epic_path`).

- [ ] **Step 6: Run tests**

Run: `python3 -m pytest tests/test_plan_manager.py tests/ -v`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor(#238): kill epic_manager.py, clean advance signatures — .plan is the only queue"
```

---

### Task 7: Update skill markdown

All skills that read `ctx.py` output or reference `.meta`/`.epic` need updating.

**Files:**
- Modify: `work/SKILL.md` — `PLAN_ACTIVE_ISSUE` → `ACTIVE_ISSUE`, remove EPIC refs, remove `.meta` refs, add D4
- Modify: `work-end/SKILL.md` — update ctx.py field list, remove `.meta` refs, strengthen queue gate
- Modify: `work-start/SKILL.md` — update scaffold step (no `.meta`, unified `.plan`), remove `.epic` refs
- Modify: `executing-plans/SKILL.md` — update queue check field names
- Modify: `subagent-driven-development/SKILL.md` — update flowchart field names
- Modify: `handover/SKILL.md` — update field refs if present
- Modify: `brief/SKILL.md` — update field refs if present
- Modify: `work-pause/SKILL.md` — update `.meta` refs if present
- Modify: `work-resume/SKILL.md` — update `.meta` refs if present

- [ ] **Step 1: Global find of old field names and file references**

```bash
python3 -c '
import os
for root, dirs, files in os.walk("."):
    dirs[:] = [d for d in dirs if d not in ("__pycache__", ".git", "engine", "node_modules", ".worktrees", "tests")]
    for f in files:
        if f.endswith(".md"):
            path = os.path.join(root, f)
            with open(path) as fh:
                for i, line in enumerate(fh, 1):
                    for term in ["PLAN_ACTIVE_ISSUE", "EPIC_ACTIVE_ISSUE", "EPIC_PATH", "EPIC_BATCH", "IS_EPIC", "META_PATH", "meta_path", ".meta", ".epic", "epic_manager"]:
                        if term in line:
                            print(f"{path}:{i}: [{term}] {line.rstrip()}")
                            break
'
```

- [ ] **Step 2: Update work/SKILL.md**

Key changes:
- `PLAN_ACTIVE_ISSUE` → `ACTIVE_ISSUE` everywhere
- Remove all `.epic` references
- Remove `.meta` references — replace with `.plan`'s `## State` section
- D3 block: `ACTIVE_ISSUE` instead of `PLAN_ACTIVE_ISSUE`
- Step 5 (`work next`): `plan_manager.advance(<PLAN_PATH>)` — no `meta_path` arg
- Add D4 mid-session queue check:

```markdown
**Mid-session issue completion (D4):** When the active issue is completed
during a session (GitHub issue closed, user says "that's done", execution
skill reports all tasks done, or a `Closes #N` commit is made), ALWAYS
check queue state before suggesting next action:

1. Run `python3 ~/.claude/skills/project/ctx.py`
2. Read `HAS_PLAN` and `ACTIVE_ISSUE`
3. If `HAS_PLAN=yes`:
   - If `ACTIVE_ISSUE` is non-empty → more work remains.
     Suggest `next` to advance, NOT `work end`.
   - If `ACTIVE_ISSUE` is empty → queue is exhausted.
     Suggest `work end`.
4. If `HAS_PLAN=no` → suggest `work end`.

**Never suggest work-end when the queue has remaining issues.**
```

- [ ] **Step 3: Update work-end/SKILL.md**

- Field list: replace `META_STATE` with reading from `.plan`'s `## State`
- Remove `.meta` references (branch_cleanup removes `.plan` now)
- Strengthen queue gate:

```markdown
**Queue gate** (if `HAS_PLAN=yes`): Run `plan_manager.py detect` to check
queue state. If mid-queue (remaining uncompleted items exist), STOP and
redirect: "Queue has N remaining issues. Run `work next` to advance, or
pass `confirm-partial` to close the branch with remaining work."
```

- [ ] **Step 4: Update work-start/SKILL.md**

- Step 9 scaffold: no `.meta` arg, just `.plan` with `## State`
- Remove `META_PATH` from output parsing — use `PLAN_PATH`
- Remove `.epic` references
- Remove `plan=yes` conditional — `.plan` is always created

- [ ] **Step 5: Update executing-plans and SDD**

- `PLAN_ACTIVE_ISSUE` → `ACTIVE_ISSUE`
- Queue check uses `ACTIVE_ISSUE`

- [ ] **Step 6: Update remaining skills**

Scan and update: `handover/`, `brief/`, `work-pause/`, `work-resume/`, `work-slot/`.

- [ ] **Step 7: Final scan for stragglers**

Run the search from Step 1 again. Fix any remaining hits.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "docs(#238): update all skill markdown — unified .plan, ACTIVE_ISSUE, D4 queue check"
```

---

### Task 8: Update CLAUDE.md, run validation, update issue

**Files:**
- Modify: `CLAUDE.md` — document unified `.plan`, remove `.meta`/`.epic` references
- Run: `python3 scripts/validate_all.py --tier commit`
- Run: `python3 -m pytest tests/ -v`

- [ ] **Step 1: Update CLAUDE.md**

Key sections to update:
- Canonical Path Resolution Block — remove `.meta` references
- Key Skills — update work-start (`.plan` always created, no `.meta`), work-end (reads `.plan` not `.meta`)
- Any reference to `.meta`/`.epic` format or separation
- Document the unified model: `.plan` owns identity, lifecycle state, and work queue

- [ ] **Step 2: Run commit-tier validation**

```bash
python3 scripts/validate_all.py --tier commit
```

Expected: PASS

- [ ] **Step 3: Run full test suite**

```bash
python3 -m pytest tests/ -v
```

Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#238): update CLAUDE.md — unified .plan replaces .meta + .plan + .epic"
```

- [ ] **Step 5: Close issue**

```bash
gh issue close 238 --repo Hortora/soredium --comment "Landed: unified .plan replaces .meta + .plan + .epic. One file, one parser, one source of truth."
```

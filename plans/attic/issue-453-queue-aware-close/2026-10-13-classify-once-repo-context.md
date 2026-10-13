# Classify-Once RepoContext Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #449 — Phase 2: Classify-once RepoContext — eliminate distributed git probes
**Issue group:** #449, #450, #451

**Goal:** Build `RepoContext` that wraps `RepoState` + verification fields, classify once at pipeline entry, and propagate the frozen context to all consumers — eliminating the N×M edge-case whack-a-mole pattern.

**Architecture:** Two-phase construction (classify → enrich). `classify_repo()` stays unchanged returning `RepoState`. New `build_repo_context()` wraps it with `tree_sha`, `main_tree_sha`, `remote_synced`, `lifecycle_state`. `OrchestratorContext.repos` becomes `list[RepoContext]`. New `dispositions` and `disposition_repo_types` fields on `StepDef` replace skip predicates with declarative filtering in the engine.

**Tech Stack:** Python 3.12, SQLite (worklog DB), pytest

## Global Constraints

- All code in `project/` or `verification/` — not `work-end/` (which re-exports from `project/`)
- `worklog.py` lives at `~/.claude/lib/worklog.py` — changes shared across all projects
- `frozen=True` on all new dataclasses — RepoContext is immutable
- No `@safe` on new DB query functions — callers handle failures explicitly
- `git_util.git()` is the single source for git subprocess calls — never `subprocess.run(["git", ...])` directly
- Test with in-memory SQLite (`:memory:`) for DB tests, `tmp_path` fixtures for git repos

---

## Batch 1: Foundation — RepoContext and get_lifecycle_state

### Task 1: Add get_lifecycle_state() to worklog.py

**Files:**
- Modify: `~/.claude/lib/worklog.py` (add function near line 758, after active_work)
- Modify: `scripts/worklog.py` (sync copy if it differs from lib)
- Test: `tests/test_worklog_lifecycle_state.py`

**Interfaces:**
- Consumes: `worklog.connect()`, `worklog._log_event()` (for test setup)
- Produces: `get_lifecycle_state(conn: sqlite3.Connection, work_item_id: int) -> str | None`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_worklog_lifecycle_state.py
"""Tests for get_lifecycle_state() — Phase 1b prerequisite."""
import json
import sqlite3
import sys
from pathlib import Path

_lib = str(Path.home() / ".claude" / "lib")
if _lib not in sys.path:
    sys.path.insert(0, _lib)

import worklog


def _make_db():
    conn = worklog.connect(":memory:")
    return conn


def _insert_event(conn, work_item_id, event_type, to_state, from_state="idle"):
    metadata = json.dumps({"from_state": from_state, "to_state": to_state})
    conn.execute(
        "INSERT INTO events (timestamp, event_type, work_item_id, metadata) "
        "VALUES (datetime('now'), ?, ?, ?)",
        (event_type, work_item_id, metadata),
    )
    conn.commit()


def test_no_events_returns_none():
    conn = _make_db()
    result = worklog.get_lifecycle_state(conn, 999)
    assert result is None


def test_single_event_returns_state():
    conn = _make_db()
    _insert_event(conn, 1, "scaffolded", "scaffolded")
    result = worklog.get_lifecycle_state(conn, 1)
    assert result == "scaffolded"


def test_multiple_events_returns_most_recent():
    conn = _make_db()
    _insert_event(conn, 1, "scaffolded", "scaffolded")
    _insert_event(conn, 1, "activate", "active", from_state="scaffolded")
    _insert_event(conn, 1, "review_pass", "closing:review", from_state="active")
    result = worklog.get_lifecycle_state(conn, 1)
    assert result == "closing:review"


def test_different_work_items_isolated():
    conn = _make_db()
    _insert_event(conn, 1, "activate", "active")
    _insert_event(conn, 2, "review_pass", "closing:review")
    assert worklog.get_lifecycle_state(conn, 1) == "active"
    assert worklog.get_lifecycle_state(conn, 2) == "closing:review"


def test_event_without_to_state_skipped():
    conn = _make_db()
    # Insert event with metadata missing to_state
    conn.execute(
        "INSERT INTO events (timestamp, event_type, work_item_id, metadata) "
        "VALUES (datetime('now'), 'info', 1, ?)",
        (json.dumps({"note": "no state"}),),
    )
    conn.commit()
    result = worklog.get_lifecycle_state(conn, 1)
    assert result is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_worklog_lifecycle_state.py -v`
Expected: FAIL — `AttributeError: module 'worklog' has no attribute 'get_lifecycle_state'`

- [ ] **Step 3: Implement get_lifecycle_state in worklog.py**

Add to `~/.claude/lib/worklog.py` after the `active_work` function (around line 758):

```python
def get_lifecycle_state(conn: sqlite3.Connection,
                        work_item_id: int) -> str | None:
    """Most recent lifecycle state from events for this work item.
    Returns to_state from the latest event, or None if no events."""
    row = conn.execute(
        "SELECT json_extract(metadata, '$.to_state') AS to_state "
        "FROM events WHERE work_item_id=? "
        "AND json_extract(metadata, '$.to_state') IS NOT NULL "
        "ORDER BY id DESC LIMIT 1",
        (work_item_id,),
    ).fetchone()
    return row["to_state"] if row else None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_worklog_lifecycle_state.py -v`
Expected: all 5 PASS

- [ ] **Step 5: Commit**

```bash
git add tests/test_worklog_lifecycle_state.py
git commit -m "feat(#449): add get_lifecycle_state() to worklog.py — Phase 1b Refs #449"
```

### Task 2: Build RepoContext dataclass and build_repo_context()

**Files:**
- Modify: `project/repo_state.py` (add RepoContext, build_repo_context, build_repo_contexts after existing code)
- Test: `tests/test_repo_context.py`

**Interfaces:**
- Consumes: `RepoState` (from repo_state.py), `git()` (from git_util.py), `worklog.get_lifecycle_state()`, `worklog.find_work_item()`, `worklog.connect()`
- Produces:
  - `RepoContext(state: RepoState, tree_sha: str, main_tree_sha: str, remote_synced: bool, lifecycle_state: str | None)` — frozen dataclass
  - `build_repo_context(state: RepoState, base_branch: str, conn=None, work_item_id=None) -> RepoContext`
  - `build_repo_contexts(repos: list[RepoState], base_branch: str, conn=None, work_item_id=None) -> list[RepoContext]`

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_repo_context.py
"""Tests for RepoContext — classify-once frozen context."""
import json
import subprocess
import sys
from pathlib import Path

import pytest

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)

from repo_state import (
    RepoContext, RepoState, RepoType, RepoDisposition,
    build_repo_context, build_repo_contexts,
)


def _make_repo(tmp_path, name="test-repo", base_branch="main"):
    """Create a minimal git repo with a main branch."""
    repo = tmp_path / name
    repo.mkdir()
    subprocess.run(["git", "init", "-b", base_branch, str(repo)],
                   capture_output=True, check=True)
    (repo / "file.txt").write_text("content")
    subprocess.run(["git", "-C", str(repo), "add", "."],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(repo), "commit", "-m", "init"],
                   capture_output=True, check=True)
    return repo


def _make_state(repo_path, branch="main", base_branch="main",
                disposition=RepoDisposition.HAS_WORK,
                repo_type=RepoType.PROJECT, commits_ahead=1):
    return RepoState(
        path=repo_path, name=repo_path.name, repo_type=repo_type,
        disposition=disposition, branch=branch, commits_ahead=commits_ahead,
        is_dirty=False, has_remote=False, remote_branch_exists=False,
        landing_sha=None, has_upstream=False, push_remote="origin",
        canonical_path=None, is_slot_clone=False,
    )


def test_repo_context_is_frozen(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo)
    ctx = build_repo_context(state, "main")
    with pytest.raises(AttributeError):
        ctx.tree_sha = "changed"


def test_tree_sha_computed(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo, branch="main")
    ctx = build_repo_context(state, "main")
    assert ctx.tree_sha != ""
    assert len(ctx.tree_sha) == 40


def test_main_tree_sha_computed(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo, branch="main")
    ctx = build_repo_context(state, "main")
    assert ctx.main_tree_sha != ""
    assert len(ctx.main_tree_sha) == 40


def test_no_branch_gives_empty_tree_sha(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo, branch=None, disposition=RepoDisposition.NO_BRANCH)
    ctx = build_repo_context(state, "main")
    assert ctx.tree_sha == ""


def test_remote_synced_false_without_remote(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo)
    ctx = build_repo_context(state, "main")
    assert ctx.remote_synced is False


def test_lifecycle_state_none_without_conn(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo)
    ctx = build_repo_context(state, "main")
    assert ctx.lifecycle_state is None


def test_lifecycle_state_from_db(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo)

    _lib = str(Path.home() / ".claude" / "lib")
    if _lib not in sys.path:
        sys.path.insert(0, _lib)
    import worklog
    conn = worklog.connect(":memory:")
    metadata = json.dumps({"from_state": "idle", "to_state": "active"})
    conn.execute(
        "INSERT INTO events (timestamp, event_type, work_item_id, metadata) "
        "VALUES (datetime('now'), 'activate', 1, ?)",
        (metadata,),
    )
    conn.commit()

    ctx = build_repo_context(state, "main", conn=conn, work_item_id=1)
    assert ctx.lifecycle_state == "active"


def test_build_repo_contexts_batch(tmp_path):
    repo1 = _make_repo(tmp_path, "repo1")
    repo2 = _make_repo(tmp_path, "repo2")
    states = [_make_state(repo1), _make_state(repo2)]
    contexts = build_repo_contexts(states, "main")
    assert len(contexts) == 2
    assert all(isinstance(c, RepoContext) for c in contexts)


def test_state_composition(tmp_path):
    repo = _make_repo(tmp_path)
    state = _make_state(repo, disposition=RepoDisposition.EMPTY_BRANCH)
    ctx = build_repo_context(state, "main")
    assert ctx.state is state
    assert ctx.state.disposition == RepoDisposition.EMPTY_BRANCH
    assert ctx.state.path == repo
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_repo_context.py -v`
Expected: FAIL — `ImportError: cannot import name 'RepoContext' from 'repo_state'`

- [ ] **Step 3: Implement RepoContext and builders in repo_state.py**

Add to `project/repo_state.py` after `classify_repos()`:

```python
@dataclass(frozen=True)
class RepoContext:
    state: RepoState
    tree_sha: str
    main_tree_sha: str
    remote_synced: bool
    lifecycle_state: str | None


def build_repo_context(
    state: RepoState,
    base_branch: str,
    conn=None,
    work_item_id: int | None = None,
) -> RepoContext:
    tree_sha = ""
    if state.branch:
        result = git(state.path, "rev-parse", f"{state.branch}^{{tree}}")
        if result.returncode == 0:
            tree_sha = result.stdout.strip()

    main_result = git(state.path, "rev-parse", f"{base_branch}^{{tree}}")
    main_tree_sha = main_result.stdout.strip() if main_result.returncode == 0 else ""

    local_main = git(state.path, "rev-parse", base_branch)
    origin_main = git(state.path, "rev-parse", f"origin/{base_branch}")
    remote_synced = (
        local_main.returncode == 0
        and origin_main.returncode == 0
        and local_main.stdout.strip() == origin_main.stdout.strip()
    )

    lifecycle_state = None
    if conn is not None and work_item_id is not None:
        try:
            import sys as _sys
            _lib = str(Path.home() / ".claude" / "lib")
            if _lib not in _sys.path:
                _sys.path.insert(0, _lib)
            import worklog
            lifecycle_state = worklog.get_lifecycle_state(conn, work_item_id)
        except Exception:
            pass

    return RepoContext(
        state=state,
        tree_sha=tree_sha,
        main_tree_sha=main_tree_sha,
        remote_synced=remote_synced,
        lifecycle_state=lifecycle_state,
    )


def build_repo_contexts(
    repos: list[RepoState],
    base_branch: str,
    conn=None,
    work_item_id: int | None = None,
) -> list[RepoContext]:
    return [build_repo_context(r, base_branch, conn, work_item_id)
            for r in repos]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_repo_context.py -v`
Expected: all 10 PASS

- [ ] **Step 5: Commit**

```bash
git add project/repo_state.py tests/test_repo_context.py
git commit -m "feat(#449): add RepoContext dataclass and build functions Refs #449"
```

---

## Batch 2: Engine — disposition filtering on StepDef

### Task 3: Add dispositions fields to StepDef and engine evaluation

**Files:**
- Modify: `project/shared_steps.py` (add `ALL_DISPOSITIONS`, `ALL_REPO_TYPES`, two fields to `StepDef`)
- Modify: `project/orchestrator_engine.py` (add disposition filtering in `run_loop` between modes and skip_fn)
- Test: `tests/test_disposition_filtering.py`

**Interfaces:**
- Consumes: `RepoDisposition`, `RepoType` (from repo_state.py), `RepoContext` (from Task 2)
- Produces:
  - `ALL_DISPOSITIONS: frozenset[RepoDisposition]` — all enum values
  - `ALL_REPO_TYPES: frozenset[RepoType]` — all enum values
  - `StepDef.dispositions: frozenset[RepoDisposition]` — defaults to `ALL_DISPOSITIONS`
  - `StepDef.disposition_repo_types: frozenset[RepoType]` — defaults to `ALL_REPO_TYPES`
  - Engine evaluates: modes → dispositions → skip_fn (D10)

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_disposition_filtering.py
"""Tests for StepDef disposition filtering in the engine."""
import sys
from dataclasses import dataclass, field
from pathlib import Path
from unittest.mock import MagicMock

import pytest

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)

from repo_state import RepoDisposition, RepoType, RepoState, RepoContext
from shared_steps import StepDef, ALL_MODES, PipelineMode


def _make_context(state, tree_sha="abc123", main_tree_sha="def456"):
    return RepoContext(
        state=state, tree_sha=tree_sha, main_tree_sha=main_tree_sha,
        remote_synced=False, lifecycle_state=None,
    )


def _make_state(disposition, repo_type=RepoType.PROJECT, name="test"):
    return RepoState(
        path=Path(f"/tmp/{name}"), name=name, repo_type=repo_type,
        disposition=disposition, branch="feature", commits_ahead=1,
        is_dirty=False, has_remote=False, remote_branch_exists=False,
        landing_sha=None, has_upstream=False, push_remote="origin",
        canonical_path=None, is_slot_clone=False,
    )


def test_default_dispositions_includes_all():
    from shared_steps import ALL_DISPOSITIONS, ALL_REPO_TYPES
    step = StepDef("test", "phase", "mechanical")
    assert step.dispositions == ALL_DISPOSITIONS
    assert step.disposition_repo_types == ALL_REPO_TYPES


def test_step_with_restricted_dispositions():
    step = StepDef(
        "review", "review", "judgment",
        dispositions=frozenset({RepoDisposition.HAS_WORK}),
        disposition_repo_types=frozenset({RepoType.PROJECT}),
    )
    assert RepoDisposition.HAS_WORK in step.dispositions
    assert RepoDisposition.EMPTY_BRANCH not in step.dispositions


def test_engine_skips_step_when_no_matching_repos(tmp_path):
    from shared_steps import ALL_DISPOSITIONS, ALL_REPO_TYPES
    from orchestrator_engine import run_loop
    from progress import Progress

    step = StepDef(
        "review", "review", "judgment",
        dispositions=frozenset({RepoDisposition.HAS_WORK}),
        disposition_repo_types=frozenset({RepoType.PROJECT}),
    )

    empty_state = _make_state(RepoDisposition.EMPTY_BRANCH)
    empty_ctx = _make_context(empty_state)

    ctx = MagicMock()
    ctx.workspace = tmp_path
    ctx.pipeline_mode = PipelineMode.BRANCH
    ctx.progress = Progress()
    ctx.repos = [empty_ctx]
    ctx.steps_executed = []
    ctx.done = lambda s: False
    ctx.last_output = {}

    result = run_loop([step], ctx)
    assert result.get("ACTION") == "complete"


def test_engine_runs_step_when_matching_repos(tmp_path):
    from orchestrator_engine import run_loop
    from progress import Progress

    step = StepDef(
        "review", "review", "judgment",
        dispositions=frozenset({RepoDisposition.HAS_WORK}),
        disposition_repo_types=frozenset({RepoType.PROJECT}),
    )

    work_state = _make_state(RepoDisposition.HAS_WORK)
    work_ctx = _make_context(work_state)

    ctx = MagicMock()
    ctx.workspace = tmp_path
    ctx.pipeline_mode = PipelineMode.BRANCH
    ctx.progress = Progress()
    ctx.repos = [work_ctx]
    ctx.steps_executed = []
    ctx.done = lambda s: False
    ctx.last_output = {}

    result = run_loop([step], ctx)
    assert result.get("ACTION") == "review"


def test_engine_filters_by_repo_type(tmp_path):
    from orchestrator_engine import run_loop
    from progress import Progress

    step = StepDef(
        "review", "review", "judgment",
        dispositions=frozenset({RepoDisposition.HAS_WORK}),
        disposition_repo_types=frozenset({RepoType.PROJECT}),
    )

    # HAS_WORK but wrong repo type (WORKSPACE, not PROJECT)
    ws_state = _make_state(RepoDisposition.HAS_WORK, RepoType.WORKSPACE, "ws")
    ws_ctx = _make_context(ws_state)

    ctx = MagicMock()
    ctx.workspace = tmp_path
    ctx.pipeline_mode = PipelineMode.BRANCH
    ctx.progress = Progress()
    ctx.repos = [ws_ctx]
    ctx.steps_executed = []
    ctx.done = lambda s: False
    ctx.last_output = {}

    result = run_loop([step], ctx)
    assert result.get("ACTION") == "complete"


def test_default_dispositions_never_filter(tmp_path):
    from orchestrator_engine import run_loop
    from progress import Progress

    step = StepDef("promote", "promote", "judgment")
    # Default dispositions = ALL — should never skip

    empty_state = _make_state(RepoDisposition.EMPTY_BRANCH)
    empty_ctx = _make_context(empty_state)

    ctx = MagicMock()
    ctx.workspace = tmp_path
    ctx.pipeline_mode = PipelineMode.BRANCH
    ctx.progress = Progress()
    ctx.repos = [empty_ctx]
    ctx.steps_executed = []
    ctx.done = lambda s: False
    ctx.last_output = {}

    result = run_loop([step], ctx)
    assert result.get("ACTION") == "promote"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_disposition_filtering.py -v`
Expected: FAIL — `ImportError: cannot import name 'ALL_DISPOSITIONS'`

- [ ] **Step 3: Add fields to StepDef in shared_steps.py**

Add imports and constants near the top of `project/shared_steps.py`:

```python
import sys as _sys
_repo_dir = str(Path(__file__).resolve().parent)
if _repo_dir not in _sys.path:
    _sys.path.insert(0, _repo_dir)
from repo_state import RepoDisposition, RepoType

ALL_DISPOSITIONS = frozenset(RepoDisposition)
ALL_REPO_TYPES = frozenset(RepoType)
```

Add two fields to `StepDef`:

```python
dispositions: frozenset[RepoDisposition] = field(
    default_factory=lambda: ALL_DISPOSITIONS)
disposition_repo_types: frozenset[RepoType] = field(
    default_factory=lambda: ALL_REPO_TYPES)
```

- [ ] **Step 4: Add disposition evaluation to run_loop in orchestrator_engine.py**

In `project/orchestrator_engine.py`, in `run_loop()`, after the modes filtering (line 228) and before the step iteration (line 233), add disposition evaluation inside the loop. The disposition check goes after the `skip_fn` check logic — insert it between the modes filter and the `for step in steps:` loop body.

Actually, disposition is per-step, so it goes inside the loop. After the modes filter (which is pre-loop), inside the `for step in steps:` loop, before `skip_fn`:

```python
# After line 228 (modes filter), inside the for loop, before skip_fn:
# Disposition filtering (D10: modes → dispositions → skip_fn)
from shared_steps import ALL_DISPOSITIONS
if step.dispositions != ALL_DISPOSITIONS and hasattr(ctx, 'repos') and ctx.repos:
    has_matching = any(
        r.state.disposition in step.dispositions
        for r in ctx.repos
        if r.state.repo_type in step.disposition_repo_types
    )
    if not has_matching:
        _mark_done(ctx, step.name, "skipped")
        ctx.steps_executed.append(f"{step.name}:disposition_skip")
        continue
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_disposition_filtering.py -v`
Expected: all 5 PASS

- [ ] **Step 6: Run existing tests to verify no regressions**

Run: `python3 -m pytest tests/test_shared_steps.py tests/test_work_end_orchestrator.py -v`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add project/shared_steps.py project/orchestrator_engine.py tests/test_disposition_filtering.py
git commit -m "feat(#449): add disposition filtering to StepDef and engine Refs #449"
```

---

## Batch 3: Integration — propagate RepoContext into orchestrator

### Task 4: Propagate RepoContext into work_end_orchestrator.py

**Files:**
- Modify: `work-end/work_end_orchestrator.py` (change classify_repos call to build_repo_contexts, update all `r.disposition` → `r.state.disposition` etc.)
- Test: `tests/test_repo_context_integration.py`

**Interfaces:**
- Consumes: `RepoContext`, `build_repo_contexts` (from Task 2), `worklog.connect()`, `worklog.find_work_item()`
- Produces: `OrchestratorContext.repos` typed as `list[RepoContext]`

- [ ] **Step 1: Write the integration test**

```python
# tests/test_repo_context_integration.py
"""Tests for RepoContext integration in the orchestrator."""
import sys
from pathlib import Path
from unittest.mock import patch

import pytest

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)
_workend_dir = str(Path(__file__).resolve().parent.parent / "work-end")
if _workend_dir not in sys.path:
    sys.path.insert(0, _workend_dir)

from repo_state import RepoContext, RepoState, RepoType, RepoDisposition


def test_orchestrator_context_repos_are_repo_context():
    """After integration, repos should be list[RepoContext]."""
    from work_end_orchestrator import OrchestratorContext
    from progress import Progress

    state = RepoState(
        path=Path("/tmp/test"), name="test", repo_type=RepoType.PROJECT,
        disposition=RepoDisposition.HAS_WORK, branch="feature",
        commits_ahead=1, is_dirty=False, has_remote=False,
        remote_branch_exists=False, landing_sha=None, has_upstream=False,
        push_remote="origin", canonical_path=None, is_slot_clone=False,
    )
    repo_ctx = RepoContext(
        state=state, tree_sha="abc", main_tree_sha="def",
        remote_synced=False, lifecycle_state=None,
    )

    ctx = OrchestratorContext(
        workspace=Path("/tmp/ws"), project=Path("/tmp/proj"),
        branch="feature", base_branch="main", meta_state="active",
        on_main=False, in_slot=False, covers="449",
        issue_repo="Hortora/soredium", progress=Progress(),
        repos=[repo_ctx],
    )
    assert isinstance(ctx.repos[0], RepoContext)
    assert ctx.repos[0].state.disposition == RepoDisposition.HAS_WORK
```

- [ ] **Step 2: Run test to verify it passes (type check only)**

Run: `python3 -m pytest tests/test_repo_context_integration.py -v`
Expected: PASS (OrchestratorContext.repos is untyped `list`, accepts anything)

- [ ] **Step 3: Update the classify_repos call site in run_orchestrator**

In `work-end/work_end_orchestrator.py`, around line 1561, change:

```python
# Before:
from repo_state import classify_repos as _classify_repos
repos = _classify_repos(
    project=project, workspace=workspace,
    branch=branch, base_branch=base_branch,
    slot_path=slot_path if in_slot else None,
)

# After:
from repo_state import classify_repos as _classify_repos, build_repo_contexts as _build_repo_contexts
_raw_repos = _classify_repos(
    project=project, workspace=workspace,
    branch=branch, base_branch=base_branch,
    slot_path=slot_path if in_slot else None,
)

_wl_conn = None
_wl_wid = None
if _wl:
    try:
        _wl_conn = _wl.connect()
        _wl_wid = _wl.find_work_item(_wl_conn, branch, str(project))
    except Exception:
        _wl_conn = None

repos = _build_repo_contexts(_raw_repos, base_branch, _wl_conn, _wl_wid)

if _wl_conn:
    try:
        _wl_conn.close()
    except Exception:
        pass
```

- [ ] **Step 4: Update all RepoState access paths**

Mechanical find-and-replace in `work-end/work_end_orchestrator.py`. Each access through `ctx.repos` items changes from direct field access to going through `.state`:

Changes in `_skip_multi_repo_partial_close` (around line 278):
- `r.disposition` → `r.state.disposition`

Changes in `_skip_no_upstream` (around line 307):
- `r.has_upstream` → `r.state.has_upstream`
- `r.repo_type` → `r.state.repo_type`

Changes in `_skip_empty_branch` (around line 322):
- `r.repo_type` → `r.state.repo_type`
- `r.disposition` → `r.state.disposition`

Changes in `_close_per_repo_mechanical` (around line 1744):
- `item.name` → `item.state.name`
- `item.path` → `item.state.path`
- `item.disposition` → `item.state.disposition`

Changes in `_close_per_repo_judgment` (around line 1867):
- `r.disposition` → `r.state.disposition`
- `r.name` → `r.state.name`

Changes in `_hydrate_landed_shas` and other repo access points — search for any remaining `.disposition`, `.path`, `.name`, `.repo_type` on items from `ctx.repos`.

- [ ] **Step 5: Remove the fallback git-probe path**

In `_close_per_repo_mechanical`, remove the `else:` branch (lines ~1770-1800) that shells out to git when `ctx.repos` is empty. Replace with an assertion:

```python
# In _close_per_repo_mechanical, after the initial check:
repo_items = ctx.repos if ctx.repos else []
if not repo_items:
    return {}  # empty repos handled at pipeline entry (D6)
```

The `repo_names_fallback` variable and the `for item in repo_names_fallback:` path with its git probes are removed entirely.

- [ ] **Step 6: Add empty repos hard error at pipeline entry**

In `run_orchestrator`, after building repos and determining pipeline mode (around line 1584):

```python
if not repos and p_mode != PipelineMode.MAIN:
    print(f"ERROR=empty_repos")
    print(f"DETAIL=classify_repos returned empty for {p_mode.value} pipeline at {slot_path or project}")
    sys.exit(1)
```

- [ ] **Step 7: Run all orchestrator tests**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py tests/test_repo_context_integration.py -v`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add work-end/work_end_orchestrator.py tests/test_repo_context_integration.py
git commit -m "feat(#449): propagate RepoContext into orchestrator, remove git-probe fallback Refs #449"
```

### Task 5: Replace _skip_empty_branch with dispositions declarations

**Files:**
- Modify: `work-end/work_end_orchestrator.py` (add dispositions to review/audit StepDefs, remove `_skip_empty_branch` function)
- Modify: `tests/test_disposition_filtering.py` (add integration test)

**Interfaces:**
- Consumes: `ALL_DISPOSITIONS`, `ALL_REPO_TYPES`, `RepoDisposition`, `RepoType` (from shared_steps.py)
- Produces: Review/audit StepDefs with `dispositions={HAS_WORK}`, `disposition_repo_types={PROJECT}`

- [ ] **Step 1: Write the failing test**

```python
# Add to tests/test_disposition_filtering.py

def test_review_steps_have_disposition_declarations():
    """Verify review/audit steps declare dispositions instead of skip_fn."""
    _workend_dir = str(Path(__file__).resolve().parent.parent / "work-end")
    if _workend_dir not in sys.path:
        sys.path.insert(0, _workend_dir)
    from work_end_orchestrator import STEPS

    review_names = {
        "code_review", "branch_audit_conformance",
        "branch_audit_coherence", "branch_audit_structure",
        "branch_audit_robustness",
    }
    for step in STEPS:
        if step.name in review_names:
            assert RepoDisposition.HAS_WORK in step.dispositions, (
                f"{step.name} should declare dispositions={{HAS_WORK}}"
            )
            assert RepoType.PROJECT in step.disposition_repo_types, (
                f"{step.name} should filter by PROJECT repo type"
            )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_disposition_filtering.py::test_review_steps_have_disposition_declarations -v`
Expected: FAIL — dispositions is ALL_DISPOSITIONS

- [ ] **Step 3: Add dispositions to review/audit StepDefs**

Find each review and audit StepDef in the STEPS list in `work_end_orchestrator.py` and add:

```python
dispositions=frozenset({RepoDisposition.HAS_WORK}),
disposition_repo_types=frozenset({RepoType.PROJECT}),
```

Remove `skip_fn=_skip_empty_branch` from these steps (the engine's disposition evaluation replaces it).

Import at the top of `work_end_orchestrator.py`:
```python
from shared_steps import ALL_DISPOSITIONS, ALL_REPO_TYPES
```

- [ ] **Step 4: Remove _skip_empty_branch function**

Delete the `_skip_empty_branch` function (lines ~315-329). Verify no other steps reference it:
```bash
grep -n "_skip_empty_branch" work-end/work_end_orchestrator.py
```
Expected: no matches after removal.

- [ ] **Step 5: Run all tests**

Run: `python3 -m pytest tests/test_disposition_filtering.py tests/test_work_end_orchestrator.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/work_end_orchestrator.py tests/test_disposition_filtering.py
git commit -m "feat(#449): replace _skip_empty_branch with StepDef dispositions Refs #449"
```

---

## Batch 4: Fan-out fixes (#437, #438)

### Task 6: Fix sweep fan-out active-repo filtering (#437) and archive peer-repo precondition (#438)

**Files:**
- Modify: `work-end/work_end_orchestrator.py` (`_close_per_repo_judgment` and `_skip_multi_repo_partial_close`)
- Test: `tests/test_fanout_fixes.py`

**Interfaces:**
- Consumes: `RepoContext`, `RepoDisposition` (from repo_state.py)
- Produces: Corrected fan-out filtering and archive precondition logic

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_fanout_fixes.py
"""Tests for fan-out filtering fixes (#437, #438)."""
import sys
from pathlib import Path
from unittest.mock import MagicMock

import pytest

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)

from repo_state import RepoContext, RepoState, RepoType, RepoDisposition


def _ctx(state, **kw):
    return RepoContext(
        state=state, tree_sha="", main_tree_sha="",
        remote_synced=False, lifecycle_state=None,
    )


def _state(name, disposition, repo_type=RepoType.CLONE):
    return RepoState(
        path=Path(f"/tmp/{name}"), name=name, repo_type=repo_type,
        disposition=disposition, branch="feature", commits_ahead=1,
        is_dirty=False, has_remote=False, remote_branch_exists=False,
        landing_sha=None, has_upstream=False, push_remote="origin",
        canonical_path=None, is_slot_clone=True,
    )


def test_sweep_fanout_skips_no_work_repos():
    """#437: sweep fan-out should only run for HAS_WORK repos."""
    _workend_dir = str(Path(__file__).resolve().parent.parent / "work-end")
    if _workend_dir not in sys.path:
        sys.path.insert(0, _workend_dir)
    from work_end_orchestrator import _close_per_repo_judgment, PER_REPO_SWEEP_STEPS
    from shared_steps import StepDef
    from progress import Progress

    active = _ctx(_state("engine", RepoDisposition.HAS_WORK))
    idle = _ctx(_state("iot", RepoDisposition.NO_BRANCH))

    ctx = MagicMock()
    ctx.in_slot = True
    ctx.slot_repos = ["engine", "iot"]
    ctx.repos = [active, idle]
    ctx.progress = Progress()
    ctx.workspace = Path("/tmp/ws")

    step = StepDef("protocol", "sweep", "judgment")

    result = _close_per_repo_judgment(step, ctx)

    # iot should have been auto-skipped (NO_BRANCH)
    assert ctx.progress.get("protocol:iot") in ("done", "skipped", "skipped_error")


def test_archive_blocked_when_repo_has_work():
    """#438: archive should be skipped when any repo still has work."""
    _workend_dir = str(Path(__file__).resolve().parent.parent / "work-end")
    if _workend_dir not in sys.path:
        sys.path.insert(0, _workend_dir)
    from work_end_orchestrator import _skip_multi_repo_partial_close

    active = _ctx(_state("engine", RepoDisposition.HAS_WORK))
    landed = _ctx(_state("iot", RepoDisposition.ALREADY_LANDED))

    ctx = MagicMock()
    ctx.in_slot = True
    ctx.repos = [active, landed]
    ctx.progress = MagicMock()
    ctx.progress.get = lambda k, default="": ""
    ctx.slot_path = Path("/tmp/slot")
    ctx.slot_repos = ["engine", "iot"]

    # Should return True (skip archive) because engine still has work
    # and hasn't landed yet
    result = _skip_multi_repo_partial_close(ctx)
    assert result is True
```

- [ ] **Step 2: Run tests to verify correctness or failure**

Run: `python3 -m pytest tests/test_fanout_fixes.py -v`
Expected: may pass or fail depending on current behavior — the tests validate the correct behavior.

- [ ] **Step 3: Fix _close_per_repo_judgment to use ctx.repos exclusively**

In `_close_per_repo_judgment`, ensure the function uses `ctx.repos` (which are now `RepoContext`) for disposition filtering. The existing logic at line 1867-1868 already does this but falls back to `ctx.slot_repos` without disposition data. Ensure no fallback path exists without disposition checks.

- [ ] **Step 4: Verify _skip_multi_repo_partial_close uses ctx.repos**

The function already uses `ctx.repos` at line 277-280. Update access paths:
- `r.disposition` → `r.state.disposition`

The directory-enumeration path (lines 283-296) that checks non-classified repos via git probes should now use `ctx.repos` exclusively — all repos are classified at pipeline entry.

- [ ] **Step 5: Run all tests**

Run: `python3 -m pytest tests/test_fanout_fixes.py tests/test_work_end_orchestrator.py tests/test_disposition_filtering.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/work_end_orchestrator.py tests/test_fanout_fixes.py
git commit -m "fix(#437,#438): sweep fan-out and archive use disposition, not git probes Refs #449"
```

---

## Batch 5: Robustness — classify_repos fallback and OrchestratorContextBase

### Task 7: Add directory-enumeration fallback to classify_repos and update OrchestratorContextBase

**Files:**
- Modify: `project/repo_state.py` (add fallback in `classify_repos` for corrupted .slot)
- Modify: `project/shared_steps.py` (type `OrchestratorContextBase.repos` as `list`)
- Test: `tests/test_classify_repos_fallback.py`

**Interfaces:**
- Consumes: `classify_repo()`, `RepoType` (from repo_state.py), `parse_slot()` (from slot_io.py)
- Produces: Robust `classify_repos()` that enumerates directories when .slot is corrupted

- [ ] **Step 1: Write the failing test**

```python
# tests/test_classify_repos_fallback.py
"""Tests for classify_repos directory-enumeration fallback."""
import subprocess
import sys
from pathlib import Path

import pytest

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)

from repo_state import classify_repos, RepoType


def _make_git_repo(path):
    path.mkdir(parents=True)
    subprocess.run(["git", "init", "-b", "main", str(path)],
                   capture_output=True, check=True)
    (path / "file.txt").write_text("content")
    subprocess.run(["git", "-C", str(path), "add", "."],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "init"],
                   capture_output=True, check=True)


def test_fallback_when_slot_file_missing(tmp_path):
    """When .slot file is missing, discover repos from directory."""
    slot_dir = tmp_path / "slot-1"
    slot_dir.mkdir()

    _make_git_repo(slot_dir / "engine")
    _make_git_repo(slot_dir / "iot")

    # No .slot file exists
    repos = classify_repos(
        project=slot_dir / "engine",
        workspace=slot_dir / "wsp-engine",
        branch="main", base_branch="main",
        slot_path=slot_dir,
    )
    assert len(repos) >= 2
    assert all(r.repo_type == RepoType.CLONE for r in repos)


def test_fallback_excludes_workspace_dirs(tmp_path):
    """Workspace directories (with .workspace marker) are excluded."""
    slot_dir = tmp_path / "slot-1"
    slot_dir.mkdir()

    _make_git_repo(slot_dir / "engine")
    ws_dir = slot_dir / "wsp-engine"
    _make_git_repo(ws_dir)
    (ws_dir / ".workspace").touch()

    repos = classify_repos(
        project=slot_dir / "engine",
        workspace=ws_dir,
        branch="main", base_branch="main",
        slot_path=slot_dir,
    )
    repo_names = {r.name for r in repos}
    assert "engine" in repo_names
    assert "wsp-engine" not in repo_names
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_classify_repos_fallback.py -v`
Expected: FAIL or incorrect behavior (currently returns empty when parse_slot fails)

- [ ] **Step 3: Add directory-enumeration fallback to classify_repos**

In `project/repo_state.py`, in `classify_repos()`, when `parse_slot` returns None:

```python
def classify_repos(
    project: Path,
    workspace: Path,
    branch: str,
    base_branch: str,
    slot_path: Path | None = None,
) -> list[RepoState]:
    if slot_path is not None:
        from slot_io import parse_slot
        info = parse_slot(slot_path)
        if info is None:
            # Fallback: enumerate git repos excluding workspaces
            repos = []
            if slot_path.is_dir():
                for entry in sorted(slot_path.iterdir()):
                    if not entry.is_dir() or not (entry / ".git").exists():
                        continue
                    if (entry / ".workspace").exists():
                        continue
                    state = classify_repo(entry, branch, base_branch, RepoType.CLONE)
                    if state is not None:
                        repos.append(state)
            return repos
        # ... rest of existing code
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_classify_repos_fallback.py tests/test_repo_state.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/repo_state.py tests/test_classify_repos_fallback.py
git commit -m "feat(#449): add directory-enumeration fallback to classify_repos Refs #449"
```

---

## References

- [2026-10-13-classify-once-repo-context-design.md] — design spec this plan implements
- [decisions.md] — 10 validated decisions (D1-D10)
- [project/repo_state.py] — RepoState, classify_repo, classify_repos
- [work-end/work_end_orchestrator.py] — OrchestratorContext, skip predicates, per-repo fan-out
- [project/shared_steps.py:31-46] — StepDef, PipelineMode
- [project/orchestrator_engine.py:197-228] — run_loop, modes evaluation
- [~/.claude/lib/worklog.py:66-78] — events table schema
- [verification/step_postconditions.py] — 12 postcondition functions (Phase 3 scope, not changed here)
- [docs/protocols/externalised-scripts-require-tests.md] — tests required for new code
- [docs/protocols/evidence-before-claims.md] — verify before claiming done
- GitHub #437 — sweep fan-out active-repo filtering
- GitHub #438 — archive peer-repo precondition
- GitHub #443 — parent: robust pipeline architecture
- GitHub #449 — focal issue

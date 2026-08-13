# Soredium TUI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #222 — REPL shell for mechanical lifecycle operations
**Issue group:** #222

**Goal:** Build a Textual-based TUI that exposes the soredium work/slot
lifecycle as a guided, panel-based interface with multi-repo/slot
navigation and tmux session management.

**Architecture:** Four-layer design (TUI → Session SPI → Command Layer →
Script Layer). Command layer is the portable core — orchestrates
refactored scripts, emits typed events. TUI renders. Home View discovers
repos/slots; Project View shows lifecycle actions. Two session providers:
TmuxProvider (default, persistent background sessions) and
SuspendingProvider (inline suspend/resume).

**Tech Stack:** Python 3.11+, Textual (TUI framework), tmux (session
multiplexing), existing soredium Python scripts (refactored for library
import).

## Global Constraints

- Python 3.11+ (for `str | None` union syntax and `dataclass` features)
- `textual` is the only new pip dependency
- All script refactoring must preserve existing CLI entry points
  (backward compatibility with Claude Code skills)
- Every refactored script uses the pattern: library function returns
  typed dataclass result; `main()` calls it and prints KEY=value
- `fcntl.flock()` for concurrent file access (`.pause-stack`, `.meta`)
- Tests use `pytest`; TUI tests use `textual`'s built-in test framework
- No IntelliJ MCP for this project (Python skills repo) — use Read/Edit
  tools for code operations

---

### Task 1: Events Module + Script Refactoring (Start/Next Path)

**Files:**
- Create: `commands/__init__.py`
- Create: `commands/events.py`
- Create: `tests/test_events.py`
- Modify: `work-start/scaffold.py`
- Create: `tests/test_scaffold_api.py`
- Modify: `work-start/branch_create.py`
- Create: `tests/test_branch_create_api.py`
- Modify: `project/stack.py`
- Create: `tests/test_stack_api.py`

**Interfaces:**
- Produces: All event dataclasses from the spec (`StateChanged`,
  `BriefReady`, `BranchCreated`, `PlanAdvanced`, `WorkEnded`,
  `WipCommitted`, `Paused`, `Resumed`, `QuickFixLanded`,
  `Recommendation`, `WhatNextReady`, `StatusReady`, `ContinueReady`,
  `CommandFailed`, `StepProgress`, `HealthCheck`, `RepoSlotInfo`,
  `HomeReady`, `ContextSwitched`, `SessionStarted`, `SessionEnded`,
  `IssueContext`)
- Produces: `scaffold(workspace, issue, branch, covers, **opts) -> ScaffoldResult`
- Produces: `create_branches(project, workspace, branch, issues, **opts) -> CreateResult`
- Produces: `read_entries(path) -> list[StackEntry]`,
  `push_entry(path, entry) -> int`,
  `pop_entry(path, branch) -> tuple[bool, int]`

- [ ] **Step 1: Create `commands/events.py` with all typed event dataclasses**

```python
# commands/events.py
from dataclasses import dataclass, field

@dataclass
class StateChanged:
    old_state: str
    new_state: str
    available_actions: list[str]
    suggested_action: str | None

@dataclass
class HealthCheck:
    check: str
    status: str
    detail: str | None

@dataclass
class BriefReady:
    issue: int | None
    branch: str
    state: str
    queue_position: str | None
    health: list[HealthCheck]
    is_epic: bool
    epic_batch: str | None
    epic_active_issue: int | None

@dataclass
class BranchCreated:
    branch: str
    issues: list[int]
    plan_path: str | None

@dataclass
class PlanAdvanced:
    completed_issue: int
    next_issue: int | None
    next_title: str | None
    position: str
    queue_complete: bool

@dataclass
class WorkEnded:
    branch: str
    issues_closed: list[int]

@dataclass
class WipCommitted:
    repo: str
    message: str

@dataclass
class Paused:
    branch: str
    stack_depth: int

@dataclass
class Resumed:
    branch: str
    rebased: bool

@dataclass
class QuickFixLanded:
    branch: str
    message: str

@dataclass
class Recommendation:
    issue: int
    title: str
    strategic_role: str | None
    readiness: str | None
    reason: str | None

@dataclass
class WhatNextReady:
    recommendations: list[Recommendation]

@dataclass
class StatusReady:
    branch: str
    state: str
    on_main: bool
    in_slot: bool
    has_plan: bool
    plan_position: str | None
    stack_depth: int
    owner_repo: str | None
    base_branch: str

@dataclass
class ContinueReady:
    issue: int | None
    branch: str
    state: str
    handoff_summary: str | None
    done_detected: bool
    suggest_next: bool
    suggest_end: bool

@dataclass
class CommandFailed:
    command: str
    step: str | None
    error: str
    detail: str
    recoverable: bool

@dataclass
class StepProgress:
    command: str
    step: str
    detail: str | None

@dataclass
class IssueContext:
    issue: int
    title: str
    branch: str
    plan_position: str | None
    project_path: str
    workspace_path: str | None

@dataclass
class SessionStarted:
    provider: str
    issue: int | None

@dataclass
class SessionEnded:
    provider: str

@dataclass
class RepoSlotInfo:
    repo: str
    slot: str | None
    branch: str
    state: str
    issue: int | None
    plan_position: str | None
    tmux_session: str | None
    project_path: str
    workspace_path: str | None

@dataclass
class HomeReady:
    repos: list[RepoSlotInfo]

@dataclass
class ContextSwitched:
    repo: str
    slot: str | None
    project_path: str
    workspace_path: str | None
```

- [ ] **Step 2: Create `commands/__init__.py` with empty command registry**

```python
# commands/__init__.py
"""Soredium command layer — portable orchestration over lifecycle scripts."""
```

- [ ] **Step 3: Write event serialisation tests**

```python
# tests/test_events.py
import json
from dataclasses import asdict
from commands.events import (
    StateChanged, BriefReady, BranchCreated, PlanAdvanced,
    CommandFailed, StepProgress, HealthCheck, Recommendation,
    WhatNextReady, HomeReady, RepoSlotInfo,
)

def test_state_changed_roundtrip():
    event = StateChanged("idle", "active", ["brief", "next", "pause"], "next")
    d = asdict(event)
    d["type"] = "StateChanged"
    json_str = json.dumps(d)
    parsed = json.loads(json_str)
    assert parsed["type"] == "StateChanged"
    assert parsed["old_state"] == "idle"
    assert parsed["suggested_action"] == "next"

def test_command_failed_fields():
    event = CommandFailed("end", "promoted", "git_push_failed",
                          "Remote rejected push", True)
    d = asdict(event)
    assert d["recoverable"] is True
    assert d["step"] == "promoted"

def test_brief_ready_with_health():
    health = [HealthCheck("meta_consistency", "ok", None),
              HealthCheck("pause_stack", "warn", "2 stale entries")]
    event = BriefReady(42, "issue-42", "active", "1/3", health,
                       False, None, None)
    d = asdict(event)
    assert len(d["health"]) == 2
    assert d["health"][1]["status"] == "warn"

def test_what_next_typed_recommendations():
    recs = [Recommendation(55, "Refactor auth", "quick-win", "ready",
                           "Low risk, high value")]
    event = WhatNextReady(recs)
    d = asdict(event)
    assert d["recommendations"][0]["strategic_role"] == "quick-win"

def test_home_ready_multiple_repos():
    repos = [
        RepoSlotInfo("casehub/engine", None, "main", "idle", None,
                     None, None, "/path/engine", None),
        RepoSlotInfo("soredium", "slot/7", "issue-222", "active", 222,
                     "1/3", "soredium-slot7-222", "/path/soredium",
                     "/path/workspace"),
    ]
    event = HomeReady(repos)
    d = asdict(event)
    assert len(d["repos"]) == 2
    assert d["repos"][1]["tmux_session"] == "soredium-slot7-222"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m pytest tests/test_events.py -v
```

- [ ] **Step 5: Refactor `project/stack.py` — extract library API**

Add `StackEntry` dataclass and pure data functions that don't print to
stdout. Keep existing `main()` and `cmd_*` functions unchanged.

```python
# Add to project/stack.py (above existing functions)

@dataclass
class StackEntry:
    branch: str
    issue: int
    paused: str
    wip_project: bool
    wip_workspace: bool

def read_entries(stack_path: Path) -> list[StackEntry]:
    """Read pause stack entries. Returns empty list if file missing."""
    if not stack_path.exists():
        return []
    text = stack_path.read_text()
    raw = _parse_entries(text)
    return [StackEntry(
        branch=e.get("branch", ""),
        issue=int(e.get("issue", 0)),
        paused=e.get("paused", ""),
        wip_project=e.get("wip_project", "no") == "yes",
        wip_workspace=e.get("wip_workspace", "no") == "yes",
    ) for e in raw]

def push_entry(stack_path: Path, entry: StackEntry) -> int:
    """Push entry onto stack. Returns new depth."""
    entries = read_entries(stack_path)
    entries.insert(0, entry)
    _write_entries(stack_path, entries)
    return len(entries)

def pop_entry(stack_path: Path, branch: str) -> tuple[bool, int]:
    """Remove entry by branch name. Returns (removed, new_depth)."""
    entries = read_entries(stack_path)
    before = len(entries)
    entries = [e for e in entries if e.branch != branch]
    _write_entries(stack_path, entries)
    return before != len(entries), len(entries)

def _write_entries(stack_path: Path, entries: list[StackEntry]) -> None:
    """Write entries to stack file in YAML format."""
    lines = []
    for e in entries:
        lines.append(f"- branch: {e.branch}")
        lines.append(f"  issue: {e.issue}")
        lines.append(f"  paused: {e.paused}")
        lines.append(f"  wip_project: {'yes' if e.wip_project else 'no'}")
        lines.append(f"  wip_workspace: {'yes' if e.wip_workspace else 'no'}")
    stack_path.parent.mkdir(parents=True, exist_ok=True)
    stack_path.write_text("\n".join(lines) + "\n" if lines else "")
```

- [ ] **Step 6: Write stack API tests**

```python
# tests/test_stack_api.py
import tempfile
from pathlib import Path
from project.stack import StackEntry, read_entries, push_entry, pop_entry

def test_read_empty_stack():
    with tempfile.TemporaryDirectory() as tmp:
        path = Path(tmp) / ".pause-stack"
        assert read_entries(path) == []

def test_push_and_read():
    with tempfile.TemporaryDirectory() as tmp:
        path = Path(tmp) / ".pause-stack"
        entry = StackEntry("issue-42", 42, "2026-08-11T00:00:00Z", True, False)
        depth = push_entry(path, entry)
        assert depth == 1
        entries = read_entries(path)
        assert len(entries) == 1
        assert entries[0].branch == "issue-42"
        assert entries[0].wip_project is True

def test_push_prepends():
    with tempfile.TemporaryDirectory() as tmp:
        path = Path(tmp) / ".pause-stack"
        push_entry(path, StackEntry("issue-1", 1, "t1", True, True))
        push_entry(path, StackEntry("issue-2", 2, "t2", False, False))
        entries = read_entries(path)
        assert entries[0].branch == "issue-2"
        assert entries[1].branch == "issue-1"

def test_pop_entry():
    with tempfile.TemporaryDirectory() as tmp:
        path = Path(tmp) / ".pause-stack"
        push_entry(path, StackEntry("issue-1", 1, "t1", True, True))
        push_entry(path, StackEntry("issue-2", 2, "t2", True, True))
        removed, depth = pop_entry(path, "issue-1")
        assert removed is True
        assert depth == 1
        entries = read_entries(path)
        assert entries[0].branch == "issue-2"

def test_pop_nonexistent():
    with tempfile.TemporaryDirectory() as tmp:
        path = Path(tmp) / ".pause-stack"
        push_entry(path, StackEntry("issue-1", 1, "t1", True, True))
        removed, depth = pop_entry(path, "issue-99")
        assert removed is False
        assert depth == 1
```

- [ ] **Step 7: Run stack tests**

```bash
python3 -m pytest tests/test_stack_api.py -v
```

- [ ] **Step 8: Refactor `work-start/scaffold.py` — extract `scaffold()` function**

Extract the body of `main()` into a library function that returns
`ScaffoldResult`. Keep `main()` as a thin CLI wrapper.

```python
# Add to work-start/scaffold.py

from dataclasses import dataclass

@dataclass
class ScaffoldResult:
    meta_path: str
    journal_path: str
    plan_path: str | None

def scaffold(workspace: Path, issue: int, branch: str,
             covers: list[int], base_branch: str = "main",
             owner_repo: str = "", design_repo_key: str = "project",
             is_epic: bool = False) -> ScaffoldResult:
    """Create .meta and JOURNAL.md in workspace/design/."""
    design_dir = workspace / "design"
    design_dir.mkdir(parents=True, exist_ok=True)

    meta_path = design_dir / ".meta"
    journal_path = design_dir / "JOURNAL.md"

    meta_lines = [
        f"state: scaffolded",
        f"issue: {issue}",
        f"issue-repo: {owner_repo}",
        f"covers: {' '.join(str(c) for c in covers)}",
        f"branch: {branch}",
        f"base-branch: {base_branch}",
        f"design-repo-key: {design_repo_key}",
    ]
    if is_epic:
        meta_lines.append("is-epic: yes")

    meta_path.write_text("\n".join(meta_lines) + "\n")

    if not journal_path.exists():
        journal_path.write_text(f"# JOURNAL — {branch}\n\n")

    return ScaffoldResult(
        meta_path=str(meta_path),
        journal_path=str(journal_path),
        plan_path=None,
    )

# Update main() to call scaffold():
def main() -> int:
    if len(sys.argv) < 3:
        print(__doc__)
        return 1
    workspace = Path(sys.argv[1]).resolve()
    params = parse_args(sys.argv[2:])
    missing = REQUIRED - params.keys()
    if missing:
        print(f"ERROR=Missing required params: {', '.join(sorted(missing))}", file=sys.stderr)
        return 1
    result = scaffold(
        workspace=workspace,
        issue=int(params["issue"]),
        branch=params["branch"],
        covers=[int(x) for x in params.get("covers", "").split()],
        base_branch=params.get("base-branch", "main"),
        owner_repo=params.get("issue-repo", ""),
        design_repo_key=params.get("design-repo-key", "project"),
        is_epic=params.get("is-epic", "no") == "yes",
    )
    print(f"META_PATH={result.meta_path}")
    print(f"JOURNAL_PATH={result.journal_path}")
    if result.plan_path:
        print(f"PLAN_PATH={result.plan_path}")
    return 0
```

- [ ] **Step 9: Write scaffold API tests**

```python
# tests/test_scaffold_api.py
import tempfile
from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "work-start"))
from scaffold import scaffold, ScaffoldResult

def test_scaffold_creates_meta():
    with tempfile.TemporaryDirectory() as tmp:
        ws = Path(tmp)
        result = scaffold(ws, issue=42, branch="issue-42-fix",
                         covers=[42], owner_repo="Hortora/soredium")
        assert isinstance(result, ScaffoldResult)
        meta = Path(result.meta_path)
        assert meta.exists()
        content = meta.read_text()
        assert "state: scaffolded" in content
        assert "issue: 42" in content

def test_scaffold_creates_journal():
    with tempfile.TemporaryDirectory() as tmp:
        ws = Path(tmp)
        result = scaffold(ws, issue=42, branch="issue-42-fix", covers=[42])
        journal = Path(result.journal_path)
        assert journal.exists()
        assert "JOURNAL" in journal.read_text()

def test_scaffold_does_not_overwrite_journal():
    with tempfile.TemporaryDirectory() as tmp:
        ws = Path(tmp)
        (ws / "design").mkdir(parents=True)
        (ws / "design" / "JOURNAL.md").write_text("# existing\n")
        result = scaffold(ws, issue=42, branch="issue-42", covers=[42])
        assert Path(result.journal_path).read_text() == "# existing\n"

def test_scaffold_epic_flag():
    with tempfile.TemporaryDirectory() as tmp:
        ws = Path(tmp)
        result = scaffold(ws, issue=10, branch="issue-10-epic",
                         covers=[10, 11, 12], is_epic=True)
        content = Path(result.meta_path).read_text()
        assert "is-epic: yes" in content
```

- [ ] **Step 10: Run scaffold tests**

```bash
python3 -m pytest tests/test_scaffold_api.py -v
```

- [ ] **Step 11: Refactor `work-start/branch_create.py` — extract `create_branches()`**

Extract the `cmd_create` function body into a typed library function.

```python
# Add to work-start/branch_create.py

from dataclasses import dataclass

@dataclass
class CreateResult:
    branch: str
    project_created: bool
    workspace_created: bool
    error: str | None = None

def create_branches(project: str, workspace: str, branch: str,
                    base_branch: str = "main") -> CreateResult:
    """Create branch in both project and workspace repos."""
    p_ok, p_msg = run_git(project, "checkout", "-b", branch)
    if not p_ok:
        if "already exists" in p_msg:
            run_git(project, "checkout", branch)
            p_ok = True
        else:
            return CreateResult(branch, False, False, p_msg)

    w_ok, w_msg = run_git(workspace, "checkout", "-b", branch)
    if not w_ok:
        if "already exists" in w_msg:
            run_git(workspace, "checkout", branch)
            w_ok = True
        else:
            return CreateResult(branch, p_ok, False, w_msg)

    return CreateResult(branch, p_ok, w_ok)
```

- [ ] **Step 12: Write branch_create API tests**

```python
# tests/test_branch_create_api.py
import tempfile, subprocess
from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "work-start"))
from branch_create import create_branches, CreateResult

def _init_repo(path: Path) -> None:
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty",
                    "-m", "init"], capture_output=True)

def test_create_branches_success():
    with tempfile.TemporaryDirectory() as tmp:
        proj = Path(tmp) / "project"
        ws = Path(tmp) / "workspace"
        proj.mkdir(); ws.mkdir()
        _init_repo(proj); _init_repo(ws)
        result = create_branches(str(proj), str(ws), "issue-42")
        assert isinstance(result, CreateResult)
        assert result.branch == "issue-42"
        assert result.project_created is True
        assert result.workspace_created is True
        assert result.error is None

def test_create_branches_idempotent():
    with tempfile.TemporaryDirectory() as tmp:
        proj = Path(tmp) / "project"
        ws = Path(tmp) / "workspace"
        proj.mkdir(); ws.mkdir()
        _init_repo(proj); _init_repo(ws)
        create_branches(str(proj), str(ws), "issue-42")
        result = create_branches(str(proj), str(ws), "issue-42")
        assert result.error is None
```

- [ ] **Step 13: Run branch_create tests**

```bash
python3 -m pytest tests/test_branch_create_api.py -v
```

- [ ] **Step 14: Commit Task 1**

```bash
git add commands/ tests/test_events.py tests/test_stack_api.py \
    tests/test_scaffold_api.py tests/test_branch_create_api.py \
    project/stack.py work-start/scaffold.py work-start/branch_create.py
git commit -m "feat(#222): events module + script refactoring for start path

Refs #222"
```

---

### Task 2: Script Refactoring (End/Pause/Resume Path)

**Files:**
- Modify: `work-end/work_end_execute.py`
- Modify: `work-end/close_artifacts.py`
- Modify: `work-end/land_branch.py`
- Modify: `work-end/branch_cleanup.py`
- Modify: `work-end/verify_promotion.py`
- Modify: `work-pause/pause_exec.py`
- Modify: `work-resume/resume_exec.py`
- Modify: `quick-fix/quick_fix.py`
- Modify: `scripts/enrichment.py`
- Create: `tests/test_end_path_api.py`
- Create: `tests/test_pause_resume_api.py`
- Create: `tests/test_enrichment_api.py`

**Interfaces:**
- Consumes: `StackEntry`, `read_entries()`, `push_entry()`, `pop_entry()` from Task 1
- Produces: `RebaseResult`, `MergeResult`, `PushResult`, `StampResult`,
  `CloseResult`, `VerifyResult`, `CleanupResult`, `PauseWipResult`,
  `PauseStackResult`, `ResumeCheckoutResult`, `ResumeRebaseResult`,
  `ResumeResetResult`, `QuickFixResult`, `WhatNextResult`

- [ ] **Step 1: Add typed results to `work-pause/pause_exec.py`**

```python
# Add to work-pause/pause_exec.py

from dataclasses import dataclass

@dataclass
class PauseWipResult:
    committed: bool  # True if WIP commit created, False if tree was clean
    repo: str
    message: str | None = None

@dataclass
class PauseStackResult:
    pushed: bool
    stack_depth: int
    branch: str
```

Update `commit_wip()` to return `PauseWipResult` instead of printing
KEY=value. Keep the print statements in `main()` only.

- [ ] **Step 2: Add typed results to `work-resume/resume_exec.py`**

```python
# Add to work-resume/resume_exec.py

from dataclasses import dataclass

@dataclass
class ResumeCheckoutResult:
    success: bool
    branch: str
    error: str | None = None

@dataclass
class ResumeRebaseResult:
    success: bool
    conflicts: bool = False

@dataclass
class ResumeResetResult:
    reset: bool  # True if WIP commit was reset
```

Update `checkout_branches()`, `rebase()`, `reset_wip()` to return these
typed results.

- [ ] **Step 3: Add typed results to end-path scripts**

For `work-end/work_end_execute.py`, decompose `cmd_land` into separate
functions:

```python
# Add to work-end/work_end_execute.py

from dataclasses import dataclass

@dataclass
class RebaseResult:
    success: bool
    branch: str
    base: str
    error: str | None = None

@dataclass
class MergeResult:
    success: bool
    error: str | None = None

@dataclass
class PushResult:
    success: bool
    attempts: int
    error: str | None = None

def rebase_onto_base(project: str, branch: str, base: str) -> RebaseResult:
    """Rebase branch onto base branch."""
    result = git(project, "rebase", base, branch)
    if result.returncode != 0:
        return RebaseResult(False, branch, base, result.stderr.strip())
    return RebaseResult(True, branch, base)

def merge_to_main(project: str, branch: str, base: str) -> MergeResult:
    """FF-only merge branch into local main."""
    git(project, "checkout", base)
    result = git(project, "merge", "--ff-only", branch)
    if result.returncode != 0:
        return MergeResult(False, result.stderr.strip())
    return MergeResult(True)

def push_to_remote(project: str, base: str, max_retries: int = 3) -> PushResult:
    """Push main to remote with retries."""
    for attempt in range(1, max_retries + 1):
        result = git(project, "push", "origin", base)
        if result.returncode == 0:
            return PushResult(True, attempt)
    return PushResult(False, max_retries, "Push failed after retries")
```

For `work-end/land_branch.py`:

```python
@dataclass
class StampResult:
    success: bool
    sha: str | None = None
    error: str | None = None
```

For `work-end/close_artifacts.py`:

```python
@dataclass
class CloseResult:
    promoted: bool
    artifacts: list[str]  # paths promoted
    error: str | None = None
```

For `work-end/verify_promotion.py`:

```python
@dataclass
class VerifyResult:
    verified: bool
    missing: list[str]  # paths that failed verification
```

For `work-end/branch_cleanup.py`:

```python
@dataclass
class CleanupResult:
    cleaned: bool
    returned_to_main: bool
    error: str | None = None
```

- [ ] **Step 4: Add typed result to `quick-fix/quick_fix.py`**

```python
@dataclass
class QuickFixResult:
    success: bool
    branch: str
    message: str
    landed_sha: str | None = None
    error: str | None = None
```

Update `run()` to return `QuickFixResult`.

- [ ] **Step 5: Add `what_next()` library function to `scripts/enrichment.py`**

```python
# Add to scripts/enrichment.py

from dataclasses import dataclass

@dataclass
class WhatNextItem:
    issue_number: int
    title: str
    score: float
    strategic_role: str | None
    readiness: str | None
    reason: str | None

def what_next(repo: str, mode: str = "general",
              limit: int = 5) -> list[WhatNextItem]:
    """Library API for what-next recommendations."""
    raw = _what_next_query(repo, mode, limit=limit)
    return [WhatNextItem(
        issue_number=r["issue_number"],
        title=r["title"],
        score=r["score"],
        strategic_role=r.get("strategic_role"),
        readiness=r.get("readiness"),
        reason=_format_reason(r),
    ) for r in raw]
```

- [ ] **Step 6: Write pause/resume API tests**

```python
# tests/test_pause_resume_api.py
import tempfile, subprocess
from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "work-pause"))
sys.path.insert(0, str(Path(__file__).parent.parent / "work-resume"))
from pause_exec import commit_wip, PauseWipResult
from resume_exec import ResumeCheckoutResult

def _init_repo(path: Path) -> None:
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty",
                    "-m", "init"], capture_output=True)

def test_commit_wip_clean_tree():
    with tempfile.TemporaryDirectory() as tmp:
        repo = Path(tmp) / "repo"
        repo.mkdir()
        _init_repo(repo)
        result = commit_wip(str(repo), "WIP: test")
        assert isinstance(result, PauseWipResult)
        assert result.committed is False

def test_commit_wip_dirty_tree():
    with tempfile.TemporaryDirectory() as tmp:
        repo = Path(tmp) / "repo"
        repo.mkdir()
        _init_repo(repo)
        (repo / "file.txt").write_text("dirty")
        subprocess.run(["git", "-C", str(repo), "add", "file.txt"],
                       capture_output=True)
        result = commit_wip(str(repo), "WIP: test")
        assert isinstance(result, PauseWipResult)
        assert result.committed is True
```

- [ ] **Step 7: Write end-path API tests**

```python
# tests/test_end_path_api.py
import tempfile, subprocess
from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
from work_end_execute import rebase_onto_base, RebaseResult

def _init_repo_with_branch(path: Path, branch: str) -> None:
    subprocess.run(["git", "init", str(path)], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty",
                    "-m", "init"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "checkout", "-b", branch],
                   capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty",
                    "-m", "work"], capture_output=True)

def test_rebase_success():
    with tempfile.TemporaryDirectory() as tmp:
        repo = Path(tmp) / "repo"
        repo.mkdir()
        _init_repo_with_branch(repo, "issue-42")
        result = rebase_onto_base(str(repo), "issue-42", "main")
        assert isinstance(result, RebaseResult)
        assert result.success is True
```

- [ ] **Step 8: Write enrichment API test**

```python
# tests/test_enrichment_api.py
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))

def test_what_next_import():
    """Verify the library function is importable and has correct signature."""
    from enrichment import what_next, WhatNextItem
    # what_next requires a database — just verify the function exists
    # and WhatNextItem is a dataclass
    assert callable(what_next)
    item = WhatNextItem(42, "Test", 0.5, "quick-win", "ready", "Low risk")
    assert item.issue_number == 42
```

- [ ] **Step 9: Run all Task 2 tests**

```bash
python3 -m pytest tests/test_pause_resume_api.py tests/test_end_path_api.py tests/test_enrichment_api.py -v
```

- [ ] **Step 10: Commit Task 2**

```bash
git add work-end/ work-pause/ work-resume/ quick-fix/ scripts/enrichment.py \
    tests/test_end_path_api.py tests/test_pause_resume_api.py \
    tests/test_enrichment_api.py
git commit -m "feat(#222): script refactoring — typed results for end/pause/resume/enrichment

Refs #222"
```

---

### Task 3: Command Layer — Core Commands

**Files:**
- Create: `commands/registry.py`
- Create: `commands/brief.py`
- Create: `commands/start.py`
- Create: `commands/next.py`
- Create: `commands/end.py`
- Create: `commands/pause.py`
- Create: `commands/resume.py`
- Create: `commands/continue_.py`
- Create: `commands/quick_fix.py`
- Create: `commands/what_next.py`
- Create: `commands/status.py`
- Create: `commands/concurrency.py`
- Create: `tests/test_commands.py`

**Interfaces:**
- Consumes: All events from Task 1, all typed results from Task 2,
  `ctx.resolve()`, `lifecycle.transition()`, `lifecycle.commit_transition()`,
  `work_state.detect()`, `topology.resolve()`, `work_health.run_checks()`
- Produces: `CommandRegistry` class with `execute(name, **kwargs) -> list[Event]`,
  `refresh() -> StateChanged`, `available_actions(state) -> list[str]`,
  `suggested_action(state) -> str | None`

- [ ] **Step 1: Create `commands/concurrency.py` — file locking**

```python
# commands/concurrency.py
import fcntl
from contextlib import contextmanager
from pathlib import Path

@contextmanager
def file_lock(path: Path):
    lock_path = path.parent / f".{path.name}.lock"
    lock_path.parent.mkdir(parents=True, exist_ok=True)
    with open(lock_path, 'w') as lock_fd:
        fcntl.flock(lock_fd, fcntl.LOCK_EX)
        try:
            yield
        finally:
            fcntl.flock(lock_fd, fcntl.LOCK_UN)
```

- [ ] **Step 2: Create `commands/registry.py` — command registry + state derivation**

```python
# commands/registry.py
import sys
from pathlib import Path
from dataclasses import dataclass
from typing import Callable

from commands.events import StateChanged, CommandFailed

# Action derivation table — maps lifecycle state to available actions
ACTION_TABLE = {
    "idle": {
        "actions": ["start", "quick-fix", "what-next", "status"],
        "suggested": "start",
        "with_stack": {"actions": ["start", "quick-fix", "resume", "what-next", "status"],
                       "suggested": "resume"},
    },
    "active": {
        "actions": ["continue", "brief", "next", "pause", "end", "session", "status"],
        "suggested": "next",
        "no_queue": {"actions": ["continue", "brief", "pause", "end", "session", "status"],
                     "suggested": "end"},
    },
    "paused": {
        "actions": ["resume", "start", "what-next", "status"],
        "suggested": "resume",
    },
    "closing:review": {"actions": ["abort", "status"], "suggested": "abort"},
    "closing:verified": {"actions": ["abort", "status"], "suggested": "abort"},
    "closing:promoted": {"actions": ["status"], "suggested": None},
    "closing:pushed": {"actions": ["status"], "suggested": None},
    "closing:merged": {"actions": ["status"], "suggested": None},
    "closing:stamped": {"actions": ["status"], "suggested": None},
}

def derive_actions(state: str, stack_depth: int = 0,
                   has_queue: bool = True) -> tuple[list[str], str | None]:
    """Derive available actions and suggested action from lifecycle state."""
    entry = ACTION_TABLE.get(state, {"actions": ["status"], "suggested": None})

    if state == "idle" and stack_depth > 0 and "with_stack" in entry:
        entry = entry["with_stack"]
    elif state == "active" and not has_queue and "no_queue" in entry:
        entry = entry["no_queue"]

    return entry["actions"], entry.get("suggested")


@dataclass
class Context:
    project_path: str
    workspace_path: str | None
    branch: str
    state: str
    on_main: bool
    in_slot: bool
    has_plan: bool
    plan_position: str | None
    stack_depth: int
    owner_repo: str | None
    base_branch: str
    meta_path: str | None
    has_queue: bool


def resolve_context(cwd: str | None = None) -> Context:
    """Resolve full context from topology + work_state + ctx."""
    # Import here to avoid circular deps with skill scripts
    project_dir = Path(__file__).parent.parent / "project"
    sys.path.insert(0, str(project_dir))
    from ctx import resolve as ctx_resolve

    raw = ctx_resolve(cwd=cwd)
    return Context(
        project_path=raw.get("PROJECT", ""),
        workspace_path=raw.get("WORKSPACE") or None,
        branch=raw.get("CURRENT_BRANCH", "main"),
        state=raw.get("META_STATE", "idle") or "idle",
        on_main=raw.get("ON_MAIN") == "yes",
        in_slot=raw.get("IN_SLOT") == "yes",
        has_plan=raw.get("HAS_PLAN") == "yes",
        plan_position=raw.get("PLAN_POSITION") or None,
        stack_depth=int(raw.get("STACK_DEPTH", "0")),
        owner_repo=raw.get("OWNER_REPO") or None,
        base_branch=raw.get("BASE_BRANCH", "main"),
        meta_path=None,  # Derived from workspace if available
        has_queue=raw.get("PLAN_POSITION", "0/0").split("/")[0] !=
                  raw.get("PLAN_POSITION", "0/0").split("/")[1]
                  if raw.get("PLAN_POSITION") else False,
    )


def refresh(cwd: str | None = None) -> StateChanged:
    """Re-detect state and return a StateChanged event."""
    ctx = resolve_context(cwd)
    actions, suggested = derive_actions(ctx.state, ctx.stack_depth, ctx.has_queue)
    return StateChanged(ctx.state, ctx.state, actions, suggested)
```

- [ ] **Step 3: Create `commands/brief.py`**

```python
# commands/brief.py
from commands.events import BriefReady, HealthCheck
from commands.registry import resolve_context

def execute(cwd: str | None = None) -> BriefReady:
    """Run brief — context summary. Read-only, no state change."""
    import sys
    from pathlib import Path
    sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
    from work_health import run_checks

    ctx = resolve_context(cwd)
    raw_checks = run_checks("branch", ctx.project_path,
                            ctx.workspace_path or "", ctx.branch,
                            ctx.owner_repo)
    health = [HealthCheck(c.get("check", ""), c.get("status", "ok"),
                          c.get("detail")) for c in (raw_checks or [])]

    return BriefReady(
        issue=None,  # Parsed from ctx if available
        branch=ctx.branch,
        state=ctx.state,
        queue_position=ctx.plan_position,
        health=health,
        is_epic=False,
        epic_batch=None,
        epic_active_issue=None,
    )
```

- [ ] **Step 4: Create `commands/status.py`**

```python
# commands/status.py
from commands.events import StatusReady
from commands.registry import resolve_context

def execute(cwd: str | None = None) -> StatusReady:
    """Full context dump. Read-only."""
    ctx = resolve_context(cwd)
    return StatusReady(
        branch=ctx.branch,
        state=ctx.state,
        on_main=ctx.on_main,
        in_slot=ctx.in_slot,
        has_plan=ctx.has_plan,
        plan_position=ctx.plan_position,
        stack_depth=ctx.stack_depth,
        owner_repo=ctx.owner_repo,
        base_branch=ctx.base_branch,
    )
```

- [ ] **Step 5: Create stub commands for start, next, end, pause, resume, continue_, quick_fix, what_next**

Each command module follows the same pattern: an `execute()` function
that calls refactored script APIs and returns events. Full implementation
wires through the lifecycle three-phase protocol.

```python
# commands/start.py
from commands.events import BranchCreated, StepProgress, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(issues: list[int], cwd: str | None = None,
            decide_fn=None) -> list:
    """Create branch, scaffold, plan. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... wire through lifecycle.transition(), branch_create, scaffold, plan_manager
    # Each step emits StepProgress, final event is BranchCreated + StateChanged
    return events
```

```python
# commands/next.py
from commands.events import PlanAdvanced, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(cwd: str | None = None) -> list:
    """Advance plan to next issue. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... wire through plan_manager.advance(), lifecycle.transition()
    return events
```

```python
# commands/end.py
from commands.events import WorkEnded, StepProgress, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(cwd: str | None = None, decide_fn=None) -> list:
    """Mechanical close — promote, rebase, push, stamp. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... sequential lifecycle transitions through closing substates
    return events
```

```python
# commands/pause.py
from commands.events import WipCommitted, Paused, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(cwd: str | None = None) -> list:
    """WIP commit + stack push. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... wire through pause_exec.commit_wip(), push_and_stack()
    return events
```

```python
# commands/resume.py
from commands.events import Resumed, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(branch: str | None = None, cwd: str | None = None) -> list:
    """Stack pop + rebase + reset WIP. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... wire through resume_exec functions
    return events
```

```python
# commands/continue_.py
from commands.events import ContinueReady
from commands.registry import resolve_context

def execute(cwd: str | None = None) -> ContinueReady:
    """Show context for continuing work. Read-only."""
    ctx = resolve_context(cwd)
    return ContinueReady(
        issue=None, branch=ctx.branch, state=ctx.state,
        handoff_summary=None, done_detected=False,
        suggest_next=False, suggest_end=False,
    )
```

```python
# commands/quick_fix.py
from commands.events import QuickFixLanded, StateChanged, CommandFailed
from commands.registry import resolve_context, derive_actions

def execute(message: str, cwd: str | None = None) -> list:
    """Ephemeral branch landing. Returns list of events."""
    events = []
    ctx = resolve_context(cwd)
    # ... wire through quick_fix.run()
    return events
```

```python
# commands/what_next.py
from commands.events import WhatNextReady, Recommendation
from commands.registry import resolve_context

def execute(cwd: str | None = None) -> WhatNextReady:
    """Enrichment recommendations. Read-only."""
    ctx = resolve_context(cwd)
    # ... wire through enrichment.what_next()
    return WhatNextReady(recommendations=[])
```

- [ ] **Step 6: Write command layer tests**

```python
# tests/test_commands.py
from commands.registry import derive_actions, ACTION_TABLE

def test_idle_no_stack():
    actions, suggested = derive_actions("idle", stack_depth=0)
    assert "start" in actions
    assert "resume" not in actions
    assert suggested == "start"

def test_idle_with_stack():
    actions, suggested = derive_actions("idle", stack_depth=2)
    assert "resume" in actions
    assert suggested == "resume"

def test_active_with_queue():
    actions, suggested = derive_actions("active", has_queue=True)
    assert "next" in actions
    assert suggested == "next"

def test_active_no_queue():
    actions, suggested = derive_actions("active", has_queue=False)
    assert "next" not in actions
    assert suggested == "end"

def test_closing_review_has_abort():
    actions, suggested = derive_actions("closing:review")
    assert "abort" in actions

def test_closing_promoted_no_abort():
    actions, suggested = derive_actions("closing:promoted")
    assert "abort" not in actions

def test_paused_suggests_resume():
    actions, suggested = derive_actions("paused")
    assert suggested == "resume"

def test_unknown_state_defaults_to_status():
    actions, suggested = derive_actions("unknown_state")
    assert actions == ["status"]
```

- [ ] **Step 7: Run command tests**

```bash
python3 -m pytest tests/test_commands.py -v
```

- [ ] **Step 8: Commit Task 3**

```bash
git add commands/ tests/test_commands.py
git commit -m "feat(#222): command layer — registry, action derivation, core command stubs

Refs #222"
```

---

### Task 4: TUI — Project View

**Files:**
- Create: `tui/__init__.py`
- Create: `tui/python/__init__.py`
- Create: `tui/python/__main__.py`
- Create: `tui/python/app.py`
- Create: `tui/python/ui/__init__.py`
- Create: `tui/python/ui/header.py`
- Create: `tui/python/ui/action_panel.py`
- Create: `tui/python/ui/content.py`
- Create: `tui/python/ui/footer.py`
- Create: `tui/python/ui/modals.py`
- Create: `tui/python/styles/app.tcss`
- Create: `tests/test_tui_project_view.py`

**Interfaces:**
- Consumes: `CommandRegistry`, `resolve_context()`, `refresh()`,
  `derive_actions()`, all event dataclasses
- Produces: `SorediumApp` (Textual App subclass), `HeaderBar`,
  `ActionPanel`, `ContentArea`, `FooterBar` widgets

- [ ] **Step 1: Create Textual CSS**

```css
/* tui/python/styles/app.tcss */
Screen {
    layout: grid;
    grid-size: 1;
    grid-rows: 3 1fr 1;
}

HeaderBar {
    height: 3;
    background: $primary-background;
    color: $text;
    padding: 0 1;
}

#main-container {
    layout: horizontal;
}

ActionPanel {
    width: 20;
    border-right: solid $primary;
    padding: 1;
}

ActionPanel .action-item {
    height: 1;
    padding: 0 1;
}

ActionPanel .action-item.--highlight {
    background: $accent;
    color: $text;
}

ActionPanel .action-item.--selected {
    text-style: bold;
}

ContentArea {
    padding: 1;
}

FooterBar {
    height: 1;
    background: $primary-background;
    color: $text-muted;
    padding: 0 1;
}

.dimmed {
    opacity: 0.4;
}
```

- [ ] **Step 2: Create header widget**

```python
# tui/python/ui/header.py
from textual.widget import Widget
from textual.reactive import reactive

class HeaderBar(Widget):
    branch = reactive("main")
    state = reactive("idle")
    queue_position = reactive("")

    def render(self) -> str:
        parts = ["soredium"]
        if self.branch != "main":
            parts.append(f"[{self.branch}]")
        parts.append(self.state)
        if self.queue_position:
            parts.append(f"Queue: {self.queue_position}")
        return "   ".join(parts)
```

- [ ] **Step 3: Create action panel widget**

```python
# tui/python/ui/action_panel.py
from textual.widget import Widget
from textual.reactive import reactive
from textual.message import Message

class ActionSelected(Message):
    def __init__(self, action: str) -> None:
        self.action = action
        super().__init__()

class ActionPanel(Widget):
    actions = reactive(list, always_update=True)
    suggested = reactive("")
    selected_index = reactive(0)
    enabled = reactive(True)

    def render(self) -> str:
        lines = ["Actions", ""]
        for i, action in enumerate(self.actions):
            prefix = "> " if i == self.selected_index else "  "
            suffix = " *" if action == self.suggested else ""
            lines.append(f"{prefix}{action}{suffix}")
        return "\n".join(lines)

    def key_up(self) -> None:
        if self.enabled and self.selected_index > 0:
            self.selected_index -= 1

    def key_down(self) -> None:
        if self.enabled and self.selected_index < len(self.actions) - 1:
            self.selected_index += 1

    def key_enter(self) -> None:
        if self.enabled and self.actions:
            self.post_message(ActionSelected(self.actions[self.selected_index]))
```

- [ ] **Step 4: Create content area widget**

```python
# tui/python/ui/content.py
from textual.widgets import RichLog

class ContentArea(RichLog):
    def show_brief(self, brief) -> None:
        self.clear()
        self.write(f"Issue: #{brief.issue}" if brief.issue else "No active issue")
        self.write(f"Branch: {brief.branch}")
        self.write(f"State: {brief.state}")
        if brief.queue_position:
            self.write(f"Queue: {brief.queue_position}")
        for h in brief.health:
            icon = "✓" if h.status == "ok" else "⚠" if h.status == "warn" else "✗"
            self.write(f"{icon} {h.check}: {h.detail or h.status}")

    def show_step_progress(self, step) -> None:
        self.write(f"  ✓ {step.step}" + (f" — {step.detail}" if step.detail else ""))

    def show_error(self, failed) -> None:
        self.write(f"  ✗ {failed.command}")
        if failed.step:
            self.write(f"    Step: {failed.step}")
        self.write(f"    Error: {failed.detail}")
        if failed.recoverable:
            self.write("    (retry available)")
```

- [ ] **Step 5: Create footer widget**

```python
# tui/python/ui/footer.py
from textual.widget import Widget
from textual.reactive import reactive

class FooterBar(Widget):
    mode = reactive("normal")

    def render(self) -> str:
        if self.mode == "running":
            return "Running..."
        if self.mode == "home":
            return "↑↓ select  Enter open  s new session  q quit"
        return "↑↓ select  Enter run  ? help  Esc back  q quit"
```

- [ ] **Step 6: Create main app**

```python
# tui/python/app.py
from textual.app import App, ComposeResult
from textual.containers import Horizontal
from textual.worker import Worker

from tui.python.ui.header import HeaderBar
from tui.python.ui.action_panel import ActionPanel, ActionSelected
from tui.python.ui.content import ContentArea
from tui.python.ui.footer import FooterBar
from commands.registry import resolve_context, refresh, derive_actions
from commands import events

class SorediumApp(App):
    CSS_PATH = "styles/app.tcss"
    TITLE = "soredium"

    def __init__(self, cwd: str | None = None):
        super().__init__()
        self.cwd = cwd
        self._running_command = False

    def compose(self) -> ComposeResult:
        yield HeaderBar()
        with Horizontal(id="main-container"):
            yield ActionPanel()
            yield ContentArea()
        yield FooterBar()

    def on_mount(self) -> None:
        state = refresh(self.cwd)
        self._apply_state(state)

    def _apply_state(self, state: events.StateChanged) -> None:
        header = self.query_one(HeaderBar)
        panel = self.query_one(ActionPanel)
        ctx = resolve_context(self.cwd)
        header.branch = ctx.branch
        header.state = state.new_state
        header.queue_position = ctx.plan_position or ""
        panel.actions = list(state.available_actions)
        panel.suggested = state.suggested_action or ""
        panel.selected_index = 0

    def on_action_selected(self, message: ActionSelected) -> None:
        if self._running_command:
            return
        self._run_command(message.action)

    def _run_command(self, action: str) -> None:
        self._running_command = True
        panel = self.query_one(ActionPanel)
        panel.enabled = False
        footer = self.query_one(FooterBar)
        footer.mode = "running"
        self.run_worker(self._execute_command(action), exclusive=True)

    async def _execute_command(self, action: str) -> None:
        content = self.query_one(ContentArea)
        content.clear()
        try:
            # Import and execute the command module
            import importlib
            cmd_name = action.replace("-", "_")
            mod = importlib.import_module(f"commands.{cmd_name}")
            result = mod.execute(cwd=self.cwd)
            # Handle result based on type
            if isinstance(result, list):
                for event in result:
                    self._handle_event(event, content)
            else:
                self._handle_event(result, content)
        except Exception as e:
            content.show_error(events.CommandFailed(
                action, None, "exception", str(e), False))
        finally:
            self._running_command = False
            panel = self.query_one(ActionPanel)
            panel.enabled = True
            footer = self.query_one(FooterBar)
            footer.mode = "normal"

    def _handle_event(self, event, content) -> None:
        if isinstance(event, events.StateChanged):
            self._apply_state(event)
        elif isinstance(event, events.StepProgress):
            content.show_step_progress(event)
        elif isinstance(event, events.CommandFailed):
            content.show_error(event)
        elif isinstance(event, events.BriefReady):
            content.show_brief(event)
        # ... handle other event types
```

- [ ] **Step 7: Create entry point**

```python
# tui/python/__main__.py
from tui.python.app import SorediumApp

def main():
    app = SorediumApp()
    app.run()

if __name__ == "__main__":
    main()
```

- [ ] **Step 8: Write TUI tests using Textual test framework**

```python
# tests/test_tui_project_view.py
import pytest
from tui.python.app import SorediumApp
from tui.python.ui.action_panel import ActionPanel
from tui.python.ui.header import HeaderBar

@pytest.mark.asyncio
async def test_app_renders():
    async with SorediumApp().run_test() as pilot:
        assert pilot.app.query_one(HeaderBar) is not None
        assert pilot.app.query_one(ActionPanel) is not None

@pytest.mark.asyncio
async def test_action_panel_keyboard_nav():
    async with SorediumApp().run_test() as pilot:
        panel = pilot.app.query_one(ActionPanel)
        panel.actions = ["start", "what-next", "status"]
        panel.selected_index = 0
        await pilot.press("down")
        assert panel.selected_index == 1
```

- [ ] **Step 9: Run TUI tests**

```bash
python3 -m pytest tests/test_tui_project_view.py -v
```

- [ ] **Step 10: Commit Task 4**

```bash
git add tui/ tests/test_tui_project_view.py
git commit -m "feat(#222): TUI Project View — Textual app with action panel, content, header

Refs #222"
```

---

### Task 5: Home View + Repo/Slot Discovery

**Files:**
- Create: `tui/python/ui/home.py`
- Create: `commands/discover.py`
- Create: `tests/test_discover.py`
- Create: `tests/test_tui_home_view.py`
- Modify: `tui/python/app.py`

**Interfaces:**
- Consumes: `ctx.resolve()`, `topology.resolve()`, `HomeReady`,
  `ContextSwitched`, `RepoSlotInfo`
- Produces: `discover_repos(scan_paths) -> list[RepoSlotInfo]`,
  `HomeView` widget

- [ ] **Step 1: Create `commands/discover.py` — repo/slot scanner**

```python
# commands/discover.py
import subprocess
from pathlib import Path
from commands.events import RepoSlotInfo, HomeReady

def discover_repos(scan_paths: list[str]) -> HomeReady:
    """Scan directories for repos and slots, resolve context for each."""
    repos = []
    for scan_path in scan_paths:
        root = Path(scan_path).expanduser()
        if not root.is_dir():
            continue
        for candidate in _find_repos(root):
            info = _resolve_repo_info(candidate)
            if info:
                repos.append(info)
        for slot_dir in _find_slots(root):
            info = _resolve_slot_info(slot_dir)
            if info:
                repos.append(info)
    return HomeReady(repos=repos)

def _find_repos(root: Path) -> list[Path]:
    """Find directories containing .git and CLAUDE.md."""
    results = []
    for d in root.iterdir():
        if d.is_dir() and (d / ".git").exists() and (d / "CLAUDE.md").exists():
            results.append(d)
        elif d.is_dir() and d.name != ".git":
            for sub in d.iterdir():
                if sub.is_dir() and (sub / ".git").exists() and (sub / "CLAUDE.md").exists():
                    results.append(sub)
    return results

def _find_slots(root: Path) -> list[Path]:
    """Find slot directories (contain .slot marker)."""
    results = []
    slots_dir = root / "slots"
    if not slots_dir.is_dir():
        return results
    for d in slots_dir.iterdir():
        if d.is_dir() and (d / ".slot").exists():
            results.append(d)
    return results

def _resolve_repo_info(repo_path: Path) -> RepoSlotInfo | None:
    """Resolve context for a single repo."""
    try:
        branch = subprocess.run(
            ["git", "-C", str(repo_path), "branch", "--show-current"],
            capture_output=True, text=True, timeout=5,
        ).stdout.strip() or "main"

        repo_name = f"{repo_path.parent.name}/{repo_path.name}"

        return RepoSlotInfo(
            repo=repo_name,
            slot=None,
            branch=branch,
            state="idle" if branch == "main" else "active",
            issue=None,
            plan_position=None,
            tmux_session=_check_tmux_session(repo_name, None),
            project_path=str(repo_path),
            workspace_path=None,
        )
    except Exception:
        return None

def _resolve_slot_info(slot_path: Path) -> RepoSlotInfo | None:
    """Resolve context for a slot directory."""
    try:
        # Slots contain symlinks to project repos
        project_link = slot_path / "soredium"  # or whatever the project name is
        if not project_link.exists():
            # Scan for any git repo in the slot
            for child in slot_path.iterdir():
                if child.is_dir() and (child / ".git").exists():
                    project_link = child
                    break

        if not project_link.exists():
            return None

        branch = subprocess.run(
            ["git", "-C", str(project_link), "branch", "--show-current"],
            capture_output=True, text=True, timeout=5,
        ).stdout.strip() or "main"

        slot_name = f"slot/{slot_path.name}"
        repo_name = project_link.name

        return RepoSlotInfo(
            repo=repo_name,
            slot=slot_name,
            branch=branch,
            state="idle" if branch == "main" else "active",
            issue=None,
            plan_position=None,
            tmux_session=_check_tmux_session(repo_name, slot_name),
            project_path=str(project_link),
            workspace_path=None,
        )
    except Exception:
        return None

def _check_tmux_session(repo: str, slot: str | None) -> str | None:
    """Check if a tmux session exists for this repo/slot."""
    name = f"soredium-{repo.replace('/', '-')}"
    if slot:
        name += f"-{slot.replace('/', '-')}"
    try:
        result = subprocess.run(
            ["tmux", "has-session", "-t", name],
            capture_output=True, timeout=2,
        )
        return name if result.returncode == 0 else None
    except Exception:
        return None
```

- [ ] **Step 2: Create Home View widget**

```python
# tui/python/ui/home.py
from textual.widget import Widget
from textual.reactive import reactive
from textual.message import Message
from commands.events import RepoSlotInfo

class RepoSelected(Message):
    def __init__(self, info: RepoSlotInfo) -> None:
        self.info = info
        super().__init__()

class HomeView(Widget):
    repos = reactive(list, always_update=True)
    selected_index = reactive(0)

    def render(self) -> str:
        lines = ["Repos & Slots", ""]
        for i, repo in enumerate(self.repos):
            prefix = "> " if i == self.selected_index else "  "
            slot_str = f"  {repo.slot}" if repo.slot else ""
            branch_str = f"[{repo.branch}]" if repo.branch != "main" else "[main]"
            session = "●" if repo.tmux_session else "◐" if repo.state == "paused" else "○"
            lines.append(f"{prefix}{repo.repo}{slot_str}  {branch_str} {repo.state}  {session}")
        lines.append("")
        lines.append("● running  ◐ paused  ○ idle")
        return "\n".join(lines)

    def key_up(self) -> None:
        if self.selected_index > 0:
            self.selected_index -= 1

    def key_down(self) -> None:
        if self.selected_index < len(self.repos) - 1:
            self.selected_index += 1

    def key_enter(self) -> None:
        if self.repos:
            self.post_message(RepoSelected(self.repos[self.selected_index]))
```

- [ ] **Step 3: Update `app.py` to support Home/Project view switching**

Add view switching logic to `SorediumApp`: start in Home View, Enter
navigates to Project View for selected repo/slot, Escape returns to
Home View.

- [ ] **Step 4: Write discovery tests**

```python
# tests/test_discover.py
import tempfile, subprocess
from pathlib import Path
from commands.discover import discover_repos, _find_repos

def test_find_repos_discovers_git_and_claude():
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        repo = root / "org" / "project"
        repo.mkdir(parents=True)
        subprocess.run(["git", "init", str(repo)], capture_output=True)
        (repo / "CLAUDE.md").write_text("# CLAUDE.md\n")
        found = _find_repos(root)
        assert len(found) == 1
        assert found[0].name == "project"

def test_discover_returns_home_ready():
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        repo = root / "myrepo"
        repo.mkdir()
        subprocess.run(["git", "init", str(repo)], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "--allow-empty",
                        "-m", "init"], capture_output=True)
        (repo / "CLAUDE.md").write_text("# CLAUDE.md\n")
        result = discover_repos([str(root)])
        assert len(result.repos) >= 1
```

- [ ] **Step 5: Run discovery and home view tests**

```bash
python3 -m pytest tests/test_discover.py tests/test_tui_home_view.py -v
```

- [ ] **Step 6: Commit Task 5**

```bash
git add tui/python/ui/home.py commands/discover.py \
    tests/test_discover.py tests/test_tui_home_view.py tui/python/app.py
git commit -m "feat(#222): Home View — repo/slot discovery + navigation

Refs #222"
```

---

### Task 6: Session SPI — TmuxProvider + SuspendingProvider

**Files:**
- Create: `tui/python/session/__init__.py`
- Create: `tui/python/session/suspend.py`
- Create: `tui/python/session/tmux.py`
- Create: `tests/test_session_spi.py`
- Modify: `tui/python/app.py`

**Interfaces:**
- Consumes: `IssueContext`, `SessionStarted`, `SessionEnded` events
- Produces: `SessionProvider` protocol, `SuspendingProvider`,
  `TmuxProvider`

- [ ] **Step 1: Create SessionProvider protocol**

```python
# tui/python/session/__init__.py
from typing import Protocol
from commands.events import IssueContext

class SessionProvider(Protocol):
    async def start(self, context: IssueContext) -> None: ...
    def is_active(self) -> bool: ...
    async def stop(self) -> None: ...
```

- [ ] **Step 2: Create SuspendingProvider**

```python
# tui/python/session/suspend.py
import asyncio
from commands.events import IssueContext

class SuspendingProvider:
    def __init__(self, cli_command: str = "claude"):
        self.cli_command = cli_command
        self._active = False

    async def start(self, context: IssueContext) -> None:
        self._active = True
        process = await asyncio.create_subprocess_exec(
            self.cli_command,
            cwd=context.project_path,
        )
        await process.wait()
        self._active = False

    def is_active(self) -> bool:
        return self._active

    async def stop(self) -> None:
        self._active = False
```

- [ ] **Step 3: Create TmuxProvider**

```python
# tui/python/session/tmux.py
import subprocess
from commands.events import IssueContext

class TmuxProvider:
    def __init__(self, cli_command: str = "claude"):
        self.cli_command = cli_command
        self._session_name: str | None = None

    def session_name_for(self, context: IssueContext) -> str:
        repo = context.project_path.rstrip("/").split("/")[-1]
        return f"soredium-{repo}-{context.issue or 'main'}"

    async def start(self, context: IssueContext) -> None:
        self._session_name = self.session_name_for(context)
        subprocess.run([
            "tmux", "new-session", "-d",
            "-s", self._session_name,
            "-c", context.project_path,
            self.cli_command,
        ], check=False)

    def is_active(self) -> bool:
        if not self._session_name:
            return False
        result = subprocess.run(
            ["tmux", "has-session", "-t", self._session_name],
            capture_output=True,
        )
        return result.returncode == 0

    async def stop(self) -> None:
        if self._session_name:
            subprocess.run(
                ["tmux", "kill-session", "-t", self._session_name],
                capture_output=True,
            )
            self._session_name = None

    def attach(self) -> None:
        """Attach to the tmux session. Call with app.suspend()."""
        if self._session_name:
            subprocess.run(["tmux", "attach-session", "-t", self._session_name])
```

- [ ] **Step 4: Write session SPI tests**

```python
# tests/test_session_spi.py
from commands.events import IssueContext
from tui.python.session.tmux import TmuxProvider
from tui.python.session.suspend import SuspendingProvider

def test_tmux_session_name():
    provider = TmuxProvider()
    ctx = IssueContext(42, "Fix scoring", "issue-42", "1/3",
                       "/path/to/soredium", None)
    assert provider.session_name_for(ctx) == "soredium-soredium-42"

def test_tmux_not_active_initially():
    provider = TmuxProvider()
    assert provider.is_active() is False

def test_suspending_not_active_initially():
    provider = SuspendingProvider()
    assert provider.is_active() is False
```

- [ ] **Step 5: Run session tests**

```bash
python3 -m pytest tests/test_session_spi.py -v
```

- [ ] **Step 6: Wire session SPI into app**

Update `SorediumApp` to handle the `session` action: create a
`TmuxProvider` or `SuspendingProvider` based on config, call
`app.suspend()` + `provider.start()` for suspend mode, or
`provider.start()` + `provider.attach()` for tmux mode.

- [ ] **Step 7: Commit Task 6**

```bash
git add tui/python/session/ tests/test_session_spi.py tui/python/app.py
git commit -m "feat(#222): Session SPI — TmuxProvider + SuspendingProvider

Refs #222"
```

---

### Task 7: CLI Mode + Packaging + Config

**Files:**
- Create: `cli/__init__.py`
- Create: `cli/__main__.py`
- Create: `pyproject.toml` (or update existing)
- Create: `soredium.toml.example`
- Create: `tests/test_cli.py`

**Interfaces:**
- Consumes: All command modules, all event dataclasses
- Produces: `soredium` CLI entry point, `soredium-tui` TUI entry point,
  JSON Lines event output

- [ ] **Step 1: Create CLI entry point**

```python
# cli/__main__.py
import json
import sys
from dataclasses import asdict

from commands.registry import resolve_context

COMMANDS = {
    "brief", "continue", "start", "next", "end", "pause", "resume",
    "quick-fix", "what-next", "status", "abort",
}

def emit(event) -> None:
    d = asdict(event)
    d["type"] = type(event).__name__
    print(json.dumps(d), flush=True)

def main() -> int:
    if len(sys.argv) < 2 or sys.argv[1] in ("-h", "--help"):
        print("Usage: soredium <command> [args...]")
        print(f"Commands: {', '.join(sorted(COMMANDS))}")
        return 0 if sys.argv[1:] == ["-h"] or sys.argv[1:] == ["--help"] else 2

    cmd = sys.argv[1]
    if cmd not in COMMANDS:
        print(f"Unknown command: {cmd}", file=sys.stderr)
        return 2

    import importlib
    cmd_name = cmd.replace("-", "_")
    try:
        mod = importlib.import_module(f"commands.{cmd_name}")
    except ImportError:
        print(f"Command module not found: commands.{cmd_name}", file=sys.stderr)
        return 2

    yes_mode = "--yes" in sys.argv or not sys.stdin.isatty()
    decide_fn = None if yes_mode else _interactive_decide

    try:
        kwargs = _parse_args(cmd, sys.argv[2:])
        kwargs["decide_fn"] = decide_fn
        result = mod.execute(**kwargs)

        if isinstance(result, list):
            for event in result:
                emit(event)
        else:
            emit(result)
        return 0
    except Exception as e:
        from commands.events import CommandFailed
        emit(CommandFailed(cmd, None, "exception", str(e), False))
        return 1

def _interactive_decide(prompt: str) -> bool:
    print(prompt, file=sys.stderr, end=" [y/N] ")
    return input().strip().lower() in ("y", "yes")

def _parse_args(cmd: str, args: list[str]) -> dict:
    kwargs: dict = {}
    if cmd == "start":
        kwargs["issues"] = [int(a.lstrip("#")) for a in args if a.lstrip("#").isdigit()]
    elif cmd == "quick-fix":
        kwargs["message"] = " ".join(a for a in args if a != "--yes")
    elif cmd == "resume" and args:
        kwargs["branch"] = args[0]
    return kwargs

if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Create config file example**

```toml
# soredium.toml.example
[workspace]
scan_paths = ["~/claude/"]

[session]
provider = "tmux"    # "tmux" or "suspend"
cli = "claude"
```

- [ ] **Step 3: Create/update pyproject.toml**

```toml
[project]
name = "soredium-tui"
version = "0.1.0"
description = "Terminal UI for the soredium work/slot lifecycle"
requires-python = ">=3.11"
dependencies = ["textual>=0.80.0"]

[project.scripts]
soredium = "cli.__main__:main"
soredium-tui = "tui.python.__main__:main"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

- [ ] **Step 4: Write CLI tests**

```python
# tests/test_cli.py
import json
import subprocess
import sys

def test_cli_help():
    result = subprocess.run(
        [sys.executable, "-m", "cli", "--help"],
        capture_output=True, text=True,
    )
    assert result.returncode == 0
    assert "soredium" in result.stdout.lower() or "usage" in result.stdout.lower()

def test_cli_unknown_command():
    result = subprocess.run(
        [sys.executable, "-m", "cli", "nonexistent"],
        capture_output=True, text=True,
    )
    assert result.returncode == 2

def test_cli_status_outputs_json():
    result = subprocess.run(
        [sys.executable, "-m", "cli", "status"],
        capture_output=True, text=True,
    )
    # May fail if not in a repo — that's fine, we just check format
    if result.returncode == 0:
        line = result.stdout.strip().split("\n")[0]
        parsed = json.loads(line)
        assert parsed["type"] == "StatusReady"
```

- [ ] **Step 5: Run CLI tests**

```bash
python3 -m pytest tests/test_cli.py -v
```

- [ ] **Step 6: Commit Task 7**

```bash
git add cli/ pyproject.toml soredium.toml.example tests/test_cli.py
git commit -m "feat(#222): CLI mode + packaging — soredium and soredium-tui entry points

Refs #222"
```

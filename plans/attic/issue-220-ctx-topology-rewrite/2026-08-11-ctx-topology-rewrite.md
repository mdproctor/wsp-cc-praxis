# ctx.py Topology Rewrite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #220 — refactor: rewrite ctx.py with topology model
**Issue group:** #220

**Goal:** Replace ctx.py's ad-hoc fallback chains with three modules:
topology.py (path resolution), work_state.py (lifecycle detection),
ctx.py (field collection). Eliminate code duplication with work_router.py.

**Architecture:** topology.py resolves a `Topology` dataclass from CWD
(one code path, no fallbacks). ctx.py collects ~40 KEY=VALUE fields
using the topology. work_state.py replaces work_router.py using the
same topology. `_find_design_file()` is the single search function for
.meta/.plan/.epic — all callers use it.

**Tech Stack:** Python 3.14, pytest, dataclasses

## Global Constraints

- CLI output format: `python3 ctx.py` → KEY=VALUE. All existing keys preserved. WorkState fields (ROUTE, ON_MAIN, STACK_DEPTH, HAS_HANDOFF, HANDOFF_PATH, IN_SLOT, SLOT_PATH) added to ctx.py output so work/SKILL.md can read everything from one source.
- work/SKILL.md MUST be updated to read from `python3 ctx.py` instead of `python3 work_router.py` (F1 critical fix)
- brief/brief.py (NOT work/brief.py — F2 path fix) MUST be refactored for the new interface
- lifecycle.py, plan_manager.py, slot_manager.py untouched
- Tests use subprocess to call ctx.py (same pattern as existing test_ctx.py) — NOT sys.path + importlib.reload (F8 fragility fix)
- CLAUDE_OK answers "is the project configured?" — reads from topo.project, not CWD (F5 semantic clarification)
- HANDOFF detection must check `HANDOFF-{project_name}.md` before `HANDOFF.md` (F4 project-specific handoffs)
- find_design_file lives in topology.py, not ctx.py (F7 — avoids circular import)

---

### Task 1: topology.py — path resolution module

**Files:**
- Create: `project/topology.py`
- Test: `tests/test_topology.py`

**Interfaces:**
- Consumes: `slot_manager.parse_slot_md()` for primary repo extraction
- Produces: `Topology` dataclass, `resolve(cwd) -> Topology`, `_resolve_symlink_target(symlink) -> str | None`

- [ ] **Step 1: Write test fixtures — init_repo helper**

```python
# tests/test_topology.py
import subprocess, sys
from pathlib import Path
import pytest

TOPOLOGY_SCRIPT = Path(__file__).parent.parent / "project" / "topology.py"

def init_repo(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init"], cwd=str(path), capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"], capture_output=True)
    subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=str(path), capture_output=True, check=True)
    return path
```

- [ ] **Step 2: Write failing tests for all 9 topology scenarios**

```python
class TestTopologyResolve:

    def test_single_repo_no_symlinks(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(repo))
        assert topo.layout == "single"
        assert topo.project == Path(str(repo)).resolve()
        assert topo.workspace == topo.project
        assert topo.workspace_root == topo.project
        assert topo.slot_dir is None
        assert topo.primary_repo is None

    def test_dual_repo_wksp_to_git_root(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "dual"
        assert topo.workspace == workspace.resolve()
        assert topo.workspace_root == workspace.resolve()

    def test_dual_repo_wksp_to_subdir(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "work")
        subdir = workspace / "engine"
        subdir.mkdir()
        (project / "wksp").symlink_to("../work/engine")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "dual"
        assert topo.workspace == subdir.resolve()
        assert topo.workspace_root == workspace.resolve()

    def test_dual_repo_proj_symlink(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (workspace / "proj").symlink_to(project)
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(workspace))
        assert topo.layout == "dual"
        assert topo.project == project.resolve()
        assert topo.workspace == workspace.resolve()

    def test_slot_detected_via_dot_slot_file(self, tmp_path):
        slot_dir = tmp_path / "slots" / "110"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text(
            "# Slot 110 — test\n\n## Repos\n- engine (primary)\n- platform\n"
        )
        project = init_repo(slot_dir / "engine")
        workspace = init_repo(slot_dir / "work")
        subdir = workspace / "engine"
        subdir.mkdir()
        (project / "wksp").symlink_to("../work/engine")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "slot"
        assert topo.slot_dir == slot_dir
        assert topo.primary_repo == "engine"

    def test_no_slot_without_dot_slot_file(self, tmp_path):
        """Substring /slots/ in path is not enough — .slot file required."""
        slot_dir = tmp_path / "slots" / "110"
        slot_dir.mkdir(parents=True)
        project = init_repo(slot_dir / "engine")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "single"
        assert topo.slot_dir is None

    def test_worktree_resolves_symlinks_from_main(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        wt = tmp_path / "wt" / "feat"
        wt.parent.mkdir(parents=True, exist_ok=True)
        subprocess.run(
            ["git", "-C", str(project), "worktree", "add", str(wt), "-b", "feat"],
            capture_output=True, check=True,
        )
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(wt))
        assert topo.in_worktree is True
        assert topo.workspace == workspace.resolve()

    def test_broken_symlink_outside_git_returns_single(self, tmp_path):
        project = init_repo(tmp_path / "project")
        (project / "wksp").symlink_to("/nonexistent/path")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "single"

    def test_dangling_symlink_walks_up_to_git_root(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to("../workspace/nonexistent")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        assert topo.layout == "dual"
        assert topo.workspace == workspace.resolve()
```

- [ ] **Step 3: Run tests to verify all fail**

Run: `python3 -m pytest tests/test_topology.py -v`
Expected: 9 FAIL (topology module doesn't exist)

- [ ] **Step 4: Implement topology.py**

```python
# project/topology.py
"""
Topology resolver — determines project layout from CWD.

One function, one code path. Returns a Topology dataclass consumed by
ctx.py and work_state.py. No fallback chains — the Topology object
contains all resolved paths.
"""
import os
import subprocess
import sys
from dataclasses import dataclass
from pathlib import Path
from typing import Literal


def _run(*cmd: str, cwd: str | None = None) -> str:
    return subprocess.run(
        list(cmd), capture_output=True, text=True, cwd=cwd
    ).stdout.strip()


def _git_root(path: str | Path) -> str | None:
    result = _run("git", "-C", str(path), "rev-parse", "--show-toplevel")
    return result or None


@dataclass
class Topology:
    layout: Literal["single", "dual", "slot"]
    project: Path
    workspace: Path
    workspace_root: Path
    slot_dir: Path | None
    primary_repo: str | None
    in_worktree: bool
    main_worktree_root: Path | None


def _resolve_symlink_target(symlink: Path) -> str | None:
    """Resolve a symlink to a path inside a git repository.

    Existing target: returns the target path (even subdirectories).
    Dangling target: walks up to the nearest git root.
    Outside any git repo: returns None.
    """
    if symlink.exists():
        resolved = symlink.resolve()
        if (resolved / ".git").exists() or (resolved / ".git").is_file():
            return str(resolved)
        candidate = resolved.parent
        while candidate != candidate.parent:
            if (candidate / ".git").exists() or (candidate / ".git").is_file():
                return str(resolved)
            candidate = candidate.parent
        return None
    if not symlink.is_symlink():
        return None
    raw_target = Path(os.readlink(symlink))
    if not raw_target.is_absolute():
        raw_target = (symlink.parent / raw_target).resolve()
    candidate = raw_target
    while candidate != candidate.parent:
        if candidate.is_dir() and (
            (candidate / ".git").exists() or (candidate / ".git").is_file()
        ):
            return str(candidate)
        candidate = candidate.parent
    return None


def _detect_slot(project: Path) -> tuple[Path | None, str | None]:
    """Structural slot detection — requires .slot file in parent."""
    slot_dir = project.parent
    slot_file = slot_dir / ".slot"
    if not slot_file.exists():
        return None, None
    _slot_dir = Path(__file__).parent.parent / "work-slot"
    if str(_slot_dir) not in sys.path:
        sys.path.insert(0, str(_slot_dir))
    from slot_manager import parse_slot_md
    info = parse_slot_md(slot_dir)
    repos = info.get("repos", [])
    primary = repos[0] if repos else None
    return slot_dir, primary


def resolve(cwd: str | None = None) -> Topology:
    if cwd is None:
        cwd = os.getcwd()

    cwd_root = _run("git", "rev-parse", "--show-toplevel", cwd=cwd)
    if not cwd_root:
        raise RuntimeError("Not in a git repository")

    # Worktree detection
    wt_output = _run("git", "worktree", "list", "--porcelain", cwd=cwd)
    main_wt_root = None
    if wt_output:
        for line in wt_output.splitlines():
            if line.startswith("worktree "):
                main_wt_root = line[len("worktree "):]
                break

    in_worktree = bool(
        main_wt_root
        and Path(main_wt_root).resolve() != Path(cwd_root).resolve()
    )
    main_worktree_path = Path(main_wt_root) if in_worktree and main_wt_root else None

    # Symlink resolution
    symlink_root = main_worktree_path if in_worktree else Path(cwd_root)
    proj_symlink = symlink_root / "proj"
    wksp_symlink = symlink_root / "wksp"

    project_str = cwd_root
    workspace_str = cwd_root
    if proj_symlink.exists() or proj_symlink.is_symlink():
        resolved = _resolve_symlink_target(proj_symlink)
        if resolved:
            workspace_str = cwd_root
            project_str = resolved
    elif wksp_symlink.exists() or wksp_symlink.is_symlink():
        resolved = _resolve_symlink_target(wksp_symlink)
        if resolved:
            project_str = cwd_root
            workspace_str = resolved

    project = Path(project_str).resolve()
    workspace = Path(workspace_str).resolve()

    # Workspace root (git root of workspace repo)
    if workspace == project:
        workspace_root = project
    else:
        ws_root_str = _git_root(workspace)
        workspace_root = Path(ws_root_str).resolve() if ws_root_str else workspace

    # Layout + slot detection
    slot_dir, primary_repo = _detect_slot(project)
    if slot_dir:
        layout = "slot"
    elif workspace != project:
        layout = "dual"
    else:
        layout = "single"

    return Topology(
        layout=layout,
        project=project,
        workspace=workspace,
        workspace_root=workspace_root,
        slot_dir=slot_dir,
        primary_repo=primary_repo,
        in_worktree=in_worktree,
        main_worktree_root=main_worktree_path,
    )
```

- [ ] **Step 5: Run tests to verify all pass**

Run: `python3 -m pytest tests/test_topology.py -v`
Expected: 9 PASS

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add project/topology.py tests/test_topology.py
git -C <PROJECT> commit -m "feat(#220): topology.py — path resolution module with Topology dataclass

Single resolve() function, single code path. Structural slot detection
via .slot file, not substring matching. Symlink resolution handles
subdirs, dangling, and non-git targets.

Refs #220"
```

---

### Task 2: _find_design_file — shared search helper

**Files:**
- Create: helper function in `project/topology.py` (same module — it uses Topology)
- Test: append to `tests/test_topology.py`

**Interfaces:**
- Consumes: `Topology` from Task 1, `_git_root()` from Task 1
- Produces: `find_design_file(name, topo) -> Path | None`

- [ ] **Step 1: Write failing tests for all search locations**

```python
# append to tests/test_topology.py

class TestFindDesignFile:

    def test_finds_in_workspace_design_subdir(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        (workspace / "design").mkdir()
        (workspace / "design" / ".meta").write_text("branch: test\n")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        result = topology.find_design_file(".meta", topo)
        assert result == workspace / "design" / ".meta"

    def test_finds_at_workspace_root(self, tmp_path):
        """Slot root .plan — no design/ subdir."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot\n\n## Repos\n- engine (primary)\n")
        project = init_repo(slot_dir / "engine")
        workspace = init_repo(slot_dir / "work")
        (project / "wksp").symlink_to("../work")
        (slot_dir / ".plan").write_text("# Plan\n")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        result = topology.find_design_file(".plan", topo)
        assert result == slot_dir / ".plan"

    def test_finds_at_workspace_git_root(self, tmp_path):
        """wksp → subdir, file at git root's design/."""
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "work")
        subdir = workspace / "engine"
        subdir.mkdir()
        (workspace / "design").mkdir()
        (workspace / "design" / ".meta").write_text("branch: test\n")
        (project / "wksp").symlink_to("../work/engine")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        result = topology.find_design_file(".meta", topo)
        assert result == workspace / "design" / ".meta"

    def test_finds_via_primary_repo_workspace(self, tmp_path):
        """Secondary repo in slot finds .plan at primary's workspace."""
        slot_dir = tmp_path / "slots" / "110"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot\n\n## Repos\n- parent (primary)\n- engine\n")
        primary = init_repo(slot_dir / "parent")
        secondary = init_repo(slot_dir / "engine")
        ws_primary = init_repo(slot_dir / "work-main")
        ws_secondary = init_repo(slot_dir / "work-eng")
        (primary / "wksp").symlink_to("../work-main")
        (secondary / "wksp").symlink_to("../work-eng")
        (ws_primary / "design").mkdir()
        (ws_primary / "design" / ".plan").write_text("# Plan\n")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(secondary))
        result = topology.find_design_file(".plan", topo)
        assert result == ws_primary / "design" / ".plan"

    def test_returns_none_when_not_found(self, tmp_path):
        project = init_repo(tmp_path / "project")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        result = topology.find_design_file(".plan", topo)
        assert result is None

    def test_design_subdir_takes_precedence(self, tmp_path):
        """design/.meta preferred over bare .meta at same level."""
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        (workspace / "design").mkdir()
        (workspace / "design" / ".meta").write_text("in_design: yes\n")
        (workspace / ".meta").write_text("at_root: yes\n")
        sys.path.insert(0, str(TOPOLOGY_SCRIPT.parent))
        import importlib, topology
        importlib.reload(topology)
        topo = topology.resolve(str(project))
        result = topology.find_design_file(".meta", topo)
        assert result == workspace / "design" / ".meta"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_topology.py::TestFindDesignFile -v`
Expected: 6 FAIL (function doesn't exist)

- [ ] **Step 3: Implement find_design_file in topology.py**

```python
def find_design_file(name: str, topo: Topology) -> Path | None:
    """Search all relevant locations for a design file (.meta, .plan, .epic).

    Order: workspace, workspace_root, slot_dir — checking design/<name>
    then <name> at each level. Falls back to primary repo's workspace
    in multi-repo slots.
    """
    candidates = [topo.workspace, topo.workspace_root]
    if topo.slot_dir:
        candidates.append(topo.slot_dir)

    for base in candidates:
        if base is None:
            continue
        for sub in [base / "design" / name, base / name]:
            if sub.exists():
                return sub

    if topo.slot_dir and topo.primary_repo:
        primary_wksp = topo.slot_dir / topo.primary_repo / "wksp"
        if primary_wksp.is_symlink():
            target = primary_wksp.resolve()
            for path in [target / "design" / name, target / name]:
                if path.exists():
                    return path
            root = _git_root(target)
            if root and str(Path(root).resolve()) != str(target.resolve()):
                root_p = Path(root)
                for path in [root_p / "design" / name, root_p / name]:
                    if path.exists():
                        return path
    return None
```

- [ ] **Step 4: Run tests to verify all pass**

Run: `python3 -m pytest tests/test_topology.py -v`
Expected: 15 PASS (9 topology + 6 find_design_file)

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add project/topology.py tests/test_topology.py
git -C <PROJECT> commit -m "feat(#220): find_design_file — single search helper for .meta/.plan/.epic

Replaces all ad-hoc fallback chains. Checks workspace, workspace_root,
slot_dir, primary repo workspace — one function, one search order.

Refs #220"
```

---

### Task 3: work_state.py — lifecycle and routing (replaces work_router.py)

**Files:**
- Create: `project/work_state.py`
- Test: `tests/test_work_state.py`
- Delete: `work/work_router.py` (after wiring)

**Interfaces:**
- Consumes: `Topology` from Task 1, `find_design_file()` from Task 2, `plan_manager.detect()`, `epic_manager.detect()`, `lifecycle.read_state()`
- Produces: `WorkState` dataclass, `detect(topo) -> WorkState`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_work_state.py
import subprocess, sys
from pathlib import Path
import pytest

def init_repo(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init"], cwd=str(path), capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"], capture_output=True)
    subprocess.run(["git", "commit", "--allow-empty", "-m", "init"], cwd=str(path), capture_output=True, check=True)
    return path

PROJ_DIR = Path(__file__).parent.parent / "project"

class TestWorkStateDetect:

    def _resolve_topo(self, cwd: str):
        sys.path.insert(0, str(PROJ_DIR))
        import importlib, topology
        importlib.reload(topology)
        return topology.resolve(cwd)

    def _detect(self, cwd: str):
        topo = self._resolve_topo(cwd)
        import importlib, work_state
        importlib.reload(work_state)
        return work_state.detect(topo)

    def test_on_main_no_stack_routes_start(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        state = self._detect(str(repo))
        assert state.route == "start"
        assert state.on_main is True
        assert state.stack_depth == 0

    def test_on_main_with_stack_routes_resume_stack(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        (repo / "design").mkdir()
        (repo / "design" / ".pause-stack").write_text("- branch: issue-42-foo\n")
        state = self._detect(str(repo))
        assert state.route == "resume_stack"
        assert state.stack_depth == 1

    def test_on_feature_branch_routes_resume_branch(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "issue-42-feat"], capture_output=True)
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text("branch: issue-42-feat\nissue: 42\n")
        state = self._detect(str(repo))
        assert state.route == "resume_branch"
        assert state.on_main is False

    def test_plan_found_has_plan_true(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        (repo / "design").mkdir()
        (repo / "design" / ".plan").write_text(
            "# Work Plan — test\n\n## Queue\n- [ ] #10 — A ← active\n\n## Session State\nCurrent: #10\nStarted: 2026-01-01\n"
        )
        state = self._detect(str(repo))
        assert state.has_plan is True
        assert state.plan_active_issue == "10"

    def test_no_plan_has_plan_false(self, tmp_path):
        repo = init_repo(tmp_path / "repo")
        state = self._detect(str(repo))
        assert state.has_plan is False

    def test_in_slot_detected(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot\n\n## Repos\n- engine (primary)\n")
        project = init_repo(slot_dir / "engine")
        state = self._detect(str(project))
        assert state.in_slot is True

    def test_plan_at_slot_root_found(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot\n\n## Repos\n- engine (primary)\n")
        (slot_dir / ".plan").write_text(
            "# Work Plan — test\n\n## Queue\n- [ ] #10 — A ← active\n\n## Session State\nCurrent: #10\nStarted: 2026-01-01\n"
        )
        project = init_repo(slot_dir / "engine")
        state = self._detect(str(project))
        assert state.has_plan is True
        assert state.plan_active_issue == "10"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_state.py -v`
Expected: 7 FAIL

- [ ] **Step 3: Implement work_state.py**

```python
# project/work_state.py
"""
Work lifecycle state detection — replaces work/work_router.py.

Single detect() function that takes a Topology and returns a WorkState.
No path resolution — all paths come from the Topology object.
"""
import re
import subprocess
import sys
from dataclasses import dataclass
from pathlib import Path

_project_dir = Path(__file__).parent
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))

_slot_dir = Path(__file__).parent.parent / "work-slot"
if str(_slot_dir) not in sys.path:
    sys.path.insert(0, str(_slot_dir))

from topology import Topology, find_design_file
from lifecycle import read_state as _read_state, is_transient as _is_transient


@dataclass
class WorkState:
    route: str
    on_main: bool
    in_slot: bool
    has_plan: bool
    plan_path: str
    plan_active_issue: str
    plan_position: str
    plan_batch: str
    stack_depth: int
    has_handoff: bool
    handoff_path: str
    meta_state: str
    meta_is_transient: bool
    is_epic: bool
    epic_path: str
    epic_batch: str
    epic_active_issue: str


def _run(*cmd: str, cwd: str | None = None) -> str:
    return subprocess.run(
        list(cmd), capture_output=True, text=True, cwd=cwd
    ).stdout.strip()


def detect(topo: Topology) -> WorkState:
    workspace = str(topo.workspace)
    project = str(topo.project)
    current_branch = _run("git", "-C", workspace, "branch", "--show-current")
    on_main = current_branch == "main"
    in_slot = topo.layout == "slot"

    # Pause stack
    stack_path = find_design_file(".pause-stack", topo)
    stack_depth = 0
    if stack_path and stack_path.exists():
        stack_depth = sum(
            1 for line in stack_path.read_text().splitlines()
            if line.strip().startswith("- branch:")
        )

    # Plan detection via shared search
    from plan_manager import detect as _plan_detect
    has_plan = False
    plan_path = ""
    plan_active_issue = ""
    plan_position = ""
    plan_batch = ""

    plan_file = find_design_file(".plan", topo)
    if plan_file:
        plan_info = _plan_detect(plan_file.parent)
        if plan_info is None and plan_file.name == ".plan":
            # plan_manager.detect checks <path>/design/.plan then <path>/.plan
            # If find_design_file found it at a root, try detect on parent
            plan_info = _plan_detect(plan_file.parent.parent) if plan_file.parent.name == "design" else _plan_detect(plan_file.parent)
        if plan_info:
            has_plan = True
            plan_path = plan_info["plan_path"]
            plan_active_issue = str(plan_info["active_issue"] or "")
            completed = plan_info.get("completed_count", 0)
            total = plan_info.get("total_count", 0)
            plan_position = f"{completed}/{total}" if total else ""
            plan_batch = plan_info.get("current_batch") or ""

    # Epic detection via shared search
    from epic_manager import detect as _epic_detect
    is_epic = False
    epic_path = ""
    epic_batch = ""
    epic_active_issue = ""
    if not has_plan:
        epic_file = find_design_file(".epic", topo)
        if epic_file:
            epic_info = _epic_detect(epic_file.parent)
            if epic_info:
                is_epic = True
                epic_path = str(epic_info["epic_path"])
                current = epic_info.get("current_batch", 0)
                total = len(epic_info.get("batches", []))
                epic_batch = f"{current} of {total}" if total else ""
                epic_active_issue = str(epic_info.get("current_issue", ""))

    # Handoff detection — project-specific first (F4 fix)
    has_handoff = False
    handoff_path = ""
    project_name = topo.project.name
    handoff_file = None
    for name in [f"HANDOFF-{project_name}.md", "HANDOFF.md"]:
        for base in [topo.workspace, topo.workspace_root]:
            candidate = base / name
            if candidate.exists():
                handoff_file = candidate
                break
        if handoff_file is None:
            rc = subprocess.run(
                ["git", "-C", workspace, "cat-file", "-e", f"main:{name}"],
                capture_output=True,
            ).returncode
            if rc == 0:
                handoff_file = topo.workspace / name
        if handoff_file:
            break

    if handoff_file:
        handoff_path = str(handoff_file)
        if on_main:
            has_handoff = True
        else:
            issue_match = re.match(r"issue-(\d+)", current_branch)
            if issue_match and handoff_file.exists():
                has_handoff = bool(
                    re.search(rf'#{issue_match.group(1)}\b', handoff_file.read_text())
                )
            elif not issue_match:
                has_handoff = True

    # Meta state
    meta_file = find_design_file(".meta", topo)
    meta_state = _read_state(meta_file) if meta_file else ""
    meta_state = meta_state or ""
    meta_is_transient = bool(meta_state and _is_transient(meta_state))

    # Routing
    project_branch = _run("git", "-C", project, "branch", "--show-current") if project != workspace else current_branch
    workspace_dirty = (
        not on_main
        and meta_file is None
        and project != workspace
        and project_branch == "main"
    )

    if on_main:
        route = "resume_stack" if stack_depth > 0 else "start"
    elif workspace_dirty:
        route = "workspace_dirty"
    else:
        route = "resume_branch"

    return WorkState(
        route=route,
        on_main=on_main,
        in_slot=in_slot,
        has_plan=has_plan,
        plan_path=plan_path,
        plan_active_issue=plan_active_issue,
        plan_position=plan_position,
        plan_batch=plan_batch,
        stack_depth=stack_depth,
        has_handoff=has_handoff,
        handoff_path=handoff_path,
        meta_state=meta_state,
        meta_is_transient=meta_is_transient,
        is_epic=is_epic,
        epic_path=epic_path,
        epic_batch=epic_batch,
        epic_active_issue=epic_active_issue,
    )
```

- [ ] **Step 4: Run tests to verify all pass**

Run: `python3 -m pytest tests/test_work_state.py -v`
Expected: 7 PASS

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add project/work_state.py tests/test_work_state.py
git -C <PROJECT> commit -m "feat(#220): work_state.py — lifecycle detection using Topology

Replaces work/work_router.py. Uses find_design_file for plan/epic
detection — no separate fallback chain. Same routing logic, single
code path.

Refs #220"
```

---

### Task 4: Rewrite ctx.py + wire callers — field collector using topology + work_state

**Files:**
- Rewrite: `project/ctx.py`
- Rewrite: `tests/test_ctx.py`
- Delete: `work/work_router.py`
- Delete: `tests/test_work_router.py`
- Modify: `brief/brief.py` (F2: correct path, refactor from dict to WorkState dataclass)
- Modify: `work/SKILL.md` (F1: replace `python3 work_router.py` with `python3 ctx.py`)

**Interfaces:**
- Consumes: `topology.resolve()`, `topology.find_design_file()`, `work_state.detect()`
- Produces: `resolve(cwd) -> dict[str, str]` — all existing keys PLUS WorkState fields:
  `ROUTE`, `ON_MAIN`, `IN_SLOT`, `SLOT_PATH`, `STACK_DEPTH`, `HAS_HANDOFF`, `HANDOFF_PATH`

- [ ] **Step 1: Write failing tests for the audit findings**

```python
# Add to tests/test_ctx.py — new tests for audit findings

class TestCtxAuditFixes:

    def test_all_claude_md_fields_from_project_not_cwd(self, tmp_path):
        """Finding #2: OWNER_REPO and PROJECT_TYPE from same CLAUDE.md."""
        project = init_repo(tmp_path / "project", "## Project Type\n\n**Type:** java\n**GitHub repo:** Org/Proj\n")
        workspace = init_repo(tmp_path / "workspace")
        (workspace / "proj").symlink_to(project)
        data = parse(run_ctx(workspace))
        assert data["OWNER_REPO"] == "Org/Proj"
        assert data["PROJECT_TYPE"] == "java"
        assert data["CLAUDE_OK"] == "yes"

    def test_empty_git_branch_no_false_mismatch(self, tmp_path):
        """Finding #6: git failure → empty branch must not trigger BRANCH_MISMATCH."""
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        # Both repos have valid branches — no mismatch expected
        data = parse(run_ctx(project))
        assert data["BRANCH_MISMATCH"] == "no"

    def test_in_slot_field_exposed(self, tmp_path):
        """Finding #14: IN_SLOT field in output."""
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text("# Slot\n\n## Repos\n- engine (primary)\n")
        project = init_repo(slot_dir / "engine")
        data = parse(run_ctx(project))
        assert data.get("IN_SLOT") == "yes"

    def test_file_checks_use_workspace_root(self, tmp_path):
        """Finding #5: blog-routing.yaml at workspace root found."""
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "work")
        subdir = workspace / "engine"
        subdir.mkdir()
        (project / "wksp").symlink_to("../work/engine")
        (workspace / "blog-routing.yaml").write_text("# routing\n")
        data = parse(run_ctx(project))
        assert data["HAS_BLOG_ROUTING"] == "yes"
```

- [ ] **Step 2: Rewrite ctx.py resolve()**

Import topology and work_state. The output dict MUST include these WorkState fields (F3 fix):

```python
# WorkState → ctx.py KEY=VALUE mapping (F3 — every field enumerated)
"ROUTE": state.route,
"ON_MAIN": "yes" if state.on_main else "no",
"IN_SLOT": "yes" if state.in_slot else "no",
"SLOT_PATH": str(topo.slot_dir / ".slot") if topo.slot_dir else "",
"STACK_DEPTH": str(state.stack_depth),
"HAS_HANDOFF": "yes" if state.has_handoff else "no",
"HANDOFF_PATH": state.handoff_path,
# These were already in ctx.py from the old code — now sourced from WorkState:
"HAS_PLAN": "yes" if state.has_plan else "no",
"PLAN_PATH": state.plan_path,
"PLAN_ACTIVE_ISSUE": state.plan_active_issue,
"PLAN_POSITION": state.plan_position,
"PLAN_BATCH": state.plan_batch,
"META_STATE": state.meta_state,
"META_IS_TRANSIENT": "yes" if state.meta_is_transient else "no",
"IS_EPIC": "yes" if state.is_epic else "no",
"EPIC_PATH": state.epic_path,
"EPIC_BATCH": state.epic_batch,
"EPIC_ACTIVE_ISSUE": state.epic_active_issue,
```

CLAUDE.md parsing: ALL fields from `topo.project / "CLAUDE.md"` (F5 fix).
File checks: include `topo.workspace_root` alongside project and workspace (F5).
BRANCH_MISMATCH: guard against empty branch string from git failure (F6).
.meta parsing: handle both `: ` and `:` separators.

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/test_topology.py tests/test_work_state.py tests/test_ctx.py -v`
Expected: ALL PASS

- [ ] **Step 4: Update work/SKILL.md (F1 critical fix)**

Replace the CLI invocation:

```markdown
# OLD — calls deleted file:
python3 ~/.claude/skills/work/work_router.py \
  $CURRENT_BRANCH $PROJECT $WORKSPACE

# NEW — all fields from ctx.py:
python3 ~/.claude/skills/project/ctx.py
# Read ROUTE, ON_MAIN, STACK_DEPTH, HAS_HANDOFF, HANDOFF_PATH,
# IN_SLOT, SLOT_PATH from output (alongside existing fields)
```

Update every reference to work_router output keys in work/SKILL.md to
note they now come from ctx.py. The key names are identical — only the
source script changes.

- [ ] **Step 5: Refactor brief/brief.py (F2 critical fix)**

Path is `brief/brief.py` (NOT `work/brief.py`). The interface changes:

```python
# OLD:
from work_router import detect_state
router = detect_state(current_branch, project, workspace)
has_plan = router.get("HAS_PLAN") == "yes"

# NEW — brief.py should call ctx.resolve() which already includes
# both topology and work_state fields:
sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
from ctx import resolve as ctx_resolve
ctx = ctx_resolve(cwd=cwd)
has_plan = ctx["HAS_PLAN"] == "yes"
route = ctx["ROUTE"]
# All keys available from one dict — no separate router call
```

- [ ] **Step 6: Delete work_router.py + tests**

```bash
git -C <PROJECT> rm work/work_router.py
git -C <PROJECT> rm tests/test_work_router.py
```

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest tests/ -v -k "ctx or topology or work_state or brief"`
Expected: ALL PASS

- [ ] **Step 8: Verify CLI output — all keys present**

```bash
python3 project/ctx.py 2>&1 | sort
# Must include ALL of these keys (existing + new):
# ROUTE, ON_MAIN, IN_SLOT, SLOT_PATH, STACK_DEPTH,
# HAS_HANDOFF, HANDOFF_PATH (new from WorkState)
# Plus all 48 existing keys unchanged
```

- [ ] **Step 9: Commit**

```bash
git -C <PROJECT> add project/ctx.py brief/brief.py work/SKILL.md tests/
git -C <PROJECT> rm work/work_router.py tests/test_work_router.py
git -C <PROJECT> commit -m "feat(#220): rewrite ctx.py + wire callers

ctx.py is now a field collector on topology + work_state. WorkState
fields merged into ctx output (F3). work/SKILL.md reads from ctx.py
not work_router.py (F1). brief/brief.py refactored (F2). HANDOFF
detection checks project-specific files first (F4). work_router.py
deleted.

Closes #220"
```

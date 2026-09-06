# Subfolder-Scoped Workspaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #333 — Subfolder-scoped workspaces for monorepo app containers
**Issue group:** #333

**Goal:** Enable independent Claude workspaces scoped to app subfolders within a monorepo, with a uniform pipeline that handles both single-repo and subfolder cases identically.

**Architecture:** Add `git_root` to Topology, restructure symlink search to walk up from CWD, merge scope + root CLAUDE.md in ctx.py. Normalize at the boundary (topology.py), run one pipeline everywhere.

**Tech Stack:** Python 3.14, pytest, git

## Global Constraints

- Protocol: every new .py script ships with pytest tests in the same commit
- `_resolve_symlink_target()` is unchanged — already preserves subfolder targets
- Layout discriminator `Literal["single", "dual", "slot"]` is unchanged
- Single-repo projects MUST produce identical ctx.py output (regression test)
- Use `is not None` (not `or`) for CLAUDE.md field merging
- Use `tmp_path` pytest fixtures; no hardcoded paths in tests

---

## Batch 1: Topology — git_root field and walk-up symlink search

### Task 1: Add git_root to Topology and computed properties

**Files:**
- Modify: `project/topology.py:32-41` (Topology dataclass)
- Modify: `project/topology.py:92-168` (resolve function)
- Test: `tests/test_topology.py`

**Interfaces:**
- Produces: `Topology.git_root: Path`, `Topology.is_scoped: bool`, `Topology.scope_rel: str`

- [ ] **Step 1: Write failing tests for git_root field**

```python
# tests/test_topology.py — add to TestTopologyResolve

def test_single_repo_git_root_equals_project(self, tmp_path):
    repo = init_repo(tmp_path / "repo")
    topo = _resolve(str(repo))
    assert topo.git_root == repo.resolve()
    assert topo.git_root == topo.project
    assert topo.is_scoped is False
    assert topo.scope_rel == ""

def test_dual_repo_git_root_equals_project(self, tmp_path):
    project = init_repo(tmp_path / "project")
    workspace = init_repo(tmp_path / "workspace")
    (project / "wksp").symlink_to(workspace)
    topo = _resolve(str(project))
    assert topo.git_root == project.resolve()
    assert topo.git_root == topo.project
    assert topo.is_scoped is False

def test_slot_has_git_root(self, tmp_path):
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
    topo = _resolve(str(project))
    assert topo.git_root == project.resolve()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_topology.py::TestTopologyResolve::test_single_repo_git_root_equals_project tests/test_topology.py::TestTopologyResolve::test_dual_repo_git_root_equals_project tests/test_topology.py::TestTopologyResolve::test_slot_has_git_root -v`
Expected: FAIL with "Topology has no attribute git_root"

- [ ] **Step 3: Add git_root field and properties to Topology dataclass**

In `project/topology.py`, modify the Topology dataclass:

```python
@dataclass
class Topology:
    layout: Literal["single", "dual", "slot"]
    project: Path
    git_root: Path
    workspace: Path
    workspace_root: Path
    slot_dir: Path | None
    primary_repo: str | None
    in_worktree: bool
    main_worktree_root: Path | None

    @property
    def is_scoped(self) -> bool:
        return self.project != self.git_root

    @property
    def scope_rel(self) -> str:
        if not self.is_scoped:
            return ""
        return str(self.project.relative_to(self.git_root))
```

Then update the `return Topology(...)` call at the end of `resolve()` to include `git_root=Path(cwd_root).resolve()`. For now, set `git_root` to the same value as the git root derived from `cwd_root` — the walk-up logic comes in Task 2.

```python
git_root = Path(cwd_root).resolve()

return Topology(
    layout=layout,
    project=project,
    git_root=git_root,
    workspace=workspace,
    workspace_root=workspace_root,
    slot_dir=slot_dir,
    primary_repo=primary_repo,
    in_worktree=in_worktree,
    main_worktree_root=main_worktree_path,
)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_topology.py -v`
Expected: ALL PASS (new tests + all existing tests)

- [ ] **Step 5: Commit**

```bash
git add project/topology.py tests/test_topology.py
git commit -m "feat(#333): add git_root field and computed properties to Topology"
```

### Task 2: Walk-up symlink search from CWD

**Files:**
- Modify: `project/topology.py:92-168` (resolve function)
- Test: `tests/test_topology.py`

**Interfaces:**
- Consumes: `Topology.git_root` from Task 1
- Produces: Walk-up symlink search that resolves `project` to app folder when `wksp/` found at a subfolder

- [ ] **Step 1: Write failing tests for subfolder resolution**

```python
# tests/test_topology.py — add to TestTopologyResolve

def test_wksp_at_subfolder_resolves_project_to_subfolder(self, tmp_path):
    """wksp/ in app subfolder → project = subfolder, git_root = repo root."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    app.mkdir(parents=True)
    workspace = init_repo(tmp_path / "quarkmind-foo")
    (app / "wksp").symlink_to(workspace)
    topo = _resolve(str(app))
    assert topo.project == app.resolve()
    assert topo.git_root == repo.resolve()
    assert topo.workspace == workspace.resolve()
    assert topo.layout == "dual"
    assert topo.is_scoped is True
    assert topo.scope_rel == "apps/foo"

def test_walkup_finds_wksp_from_nested_cwd(self, tmp_path):
    """CWD deeper than app root — walk-up finds wksp/ at app level."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    deep = app / "src" / "main"
    deep.mkdir(parents=True)
    workspace = init_repo(tmp_path / "quarkmind-foo")
    (app / "wksp").symlink_to(workspace)
    topo = _resolve(str(deep))
    assert topo.project == app.resolve()
    assert topo.git_root == repo.resolve()
    assert topo.workspace == workspace.resolve()

def test_walkup_stops_at_git_root(self, tmp_path):
    """Walk-up does not go beyond git root."""
    outer = init_repo(tmp_path / "outer")
    inner_dir = outer / "inner"
    inner_dir.mkdir()
    topo = _resolve(str(inner_dir))
    assert topo.project == outer.resolve()
    assert topo.git_root == outer.resolve()
    assert topo.layout == "single"

def test_subfolder_wksp_takes_precedence_over_root_wksp(self, tmp_path):
    """wksp/ at subfolder wins over wksp/ at git root."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    app.mkdir(parents=True)
    root_workspace = init_repo(tmp_path / "root-ws")
    app_workspace = init_repo(tmp_path / "app-ws")
    (repo / "wksp").symlink_to(root_workspace)
    (app / "wksp").symlink_to(app_workspace)
    topo = _resolve(str(app))
    assert topo.project == app.resolve()
    assert topo.workspace == app_workspace.resolve()

def test_proj_symlink_to_subfolder_preserves_scope(self, tmp_path):
    """proj/ in workspace → subfolder target preserved as project scope."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    app.mkdir(parents=True)
    workspace = init_repo(tmp_path / "quarkmind-foo")
    (workspace / "proj").symlink_to(app)
    topo = _resolve(str(workspace))
    assert topo.project == app.resolve()
    assert topo.git_root == repo.resolve()
    assert topo.workspace == workspace.resolve()
    assert topo.is_scoped is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_topology.py::TestTopologyResolve::test_wksp_at_subfolder_resolves_project_to_subfolder tests/test_topology.py::TestTopologyResolve::test_walkup_finds_wksp_from_nested_cwd -v`
Expected: FAIL — current resolve() checks git root, not CWD/walk-up

- [ ] **Step 3: Restructure resolve() with walk-up symlink search**

Replace the symlink detection section of `resolve()` (approximately lines 114-141). The new logic:

```python
def _find_symlink_up(start: Path, git_root: Path, name: str) -> Path | None:
    """Walk from start toward git_root looking for a symlink named 'name'."""
    check = start.resolve()
    root = git_root.resolve()
    while True:
        candidate = check / name
        if candidate.is_symlink():
            return candidate
        if check == root:
            break
        parent = check.parent
        if parent == check:
            break
        check = parent
    return None
```

Then in `resolve()`, replace the symlink detection block:

```python
cwd_path = Path(cwd).resolve()
git_root = Path(cwd_root).resolve()

# Walk-up symlink search: CWD → git root
wksp_sym = _find_symlink_up(cwd_path, git_root, "wksp")
proj_sym = _find_symlink_up(cwd_path, git_root, "proj")

project_str = str(git_root)
workspace_str = str(git_root)

if not in_worktree:
    if wksp_sym:
        resolved = _resolve_symlink_target(wksp_sym)
        if resolved:
            project_str = str(wksp_sym.parent.resolve())
            workspace_str = resolved
    elif proj_sym:
        resolved = _resolve_symlink_target(proj_sym)
        if resolved and Path(resolved).resolve() != cwd_path:
            workspace_str = str(cwd_path)
            project_str = resolved
        elif not proj_sym and not wksp_sym:
            pass  # single repo — defaults are correct
else:
    # Worktree handling — check worktree CWD first, then main
    wt_root = Path(cwd_root)
    cwd_wksp = wt_root / "wksp"
    if cwd_wksp.is_symlink() and cwd_wksp.is_dir():
        symlink_root = wt_root
    else:
        symlink_root = main_worktree_path if main_worktree_path else wt_root

    proj_symlink = symlink_root / "proj"
    wksp_symlink = symlink_root / "wksp"

    if proj_symlink.exists() or proj_symlink.is_symlink():
        resolved = _resolve_symlink_target(proj_symlink)
        if resolved and Path(resolved).resolve() != Path(str(symlink_root)).resolve():
            workspace_str = str(symlink_root)
            project_str = resolved
        elif wksp_symlink.exists() or wksp_symlink.is_symlink():
            resolved = _resolve_symlink_target(wksp_symlink)
            if resolved:
                project_str = str(symlink_root)
                workspace_str = resolved
    elif wksp_symlink.exists() or wksp_symlink.is_symlink():
        resolved = _resolve_symlink_target(wksp_symlink)
        if resolved:
            project_str = str(symlink_root)
            workspace_str = resolved

project = Path(project_str).resolve()
workspace = Path(workspace_str).resolve()

# Derive git_root from the resolved project path
proj_git_root = _git_root(project)
if proj_git_root:
    git_root = Path(proj_git_root).resolve()
```

Note: The worktree branch preserves existing behavior. The non-worktree branch adds walk-up. The `git_root` derivation at the end ensures it's always correct regardless of how project was resolved.

- [ ] **Step 4: Run ALL topology tests**

Run: `python3 -m pytest tests/test_topology.py -v`
Expected: ALL PASS (new subfolder tests + all existing regression tests)

- [ ] **Step 5: Commit**

```bash
git add project/topology.py tests/test_topology.py
git commit -m "feat(#333): walk-up symlink search from CWD, git_root derivation"
```

---

## Batch 2: ctx.py — CLAUDE.md merge and GIT_ROOT emission

### Task 3: CLAUDE.md merge and GIT_ROOT in ctx.py

**Files:**
- Modify: `project/ctx.py:98-101` (CLAUDE.md reading)
- Modify: `project/ctx.py:263-331` (output dict)
- Test: `tests/test_ctx.py`

**Interfaces:**
- Consumes: `Topology.git_root`, `Topology.project` (scope), `Topology.is_scoped` from Tasks 1-2
- Produces: `GIT_ROOT` field in ctx.py output, merged CLAUDE.md fields

- [ ] **Step 1: Write failing tests for CLAUDE.md merge and GIT_ROOT**

```python
# tests/test_ctx.py — add new test class

class TestSubfolderScope:
    """Test subfolder-scoped workspace resolution."""

    def test_git_root_emitted_for_single_repo(self, tmp_path):
        """Single-repo: GIT_ROOT == PROJECT."""
        repo = init_repo(tmp_path / "repo", claude_md=(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** java\n"
        ))
        result = run_ctx(repo)
        data = parse(result)
        assert result.returncode == 0
        assert data["GIT_ROOT"] == data["PROJECT"]

    def test_scope_claude_md_wins_over_root(self, tmp_path):
        """App CLAUDE.md project type wins over root."""
        repo = init_repo(tmp_path / "quarkmind", claude_md=(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** generic\n"
            "\n## Work Tracking\n\nIssue tracking: enabled\n"
            "GitHub repo: Org/quarkmind\n"
        ))
        app = repo / "apps" / "foo"
        app.mkdir(parents=True)
        (app / "CLAUDE.md").write_text(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** java\n"
        )
        workspace = init_repo(tmp_path / "quarkmind-foo")
        (app / "wksp").symlink_to(workspace)
        subprocess.run(["git", "-C", str(repo), "add", "-A"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "add app"],
                       cwd=str(repo), capture_output=True)
        result = run_ctx(app)
        data = parse(result)
        assert data["PROJECT_TYPE"] == "java"
        assert data["GIT_ROOT"] == str(repo)
        assert data["PROJECT"] == str(app.resolve())

    def test_root_fills_gap_when_scope_missing_field(self, tmp_path):
        """App has type, root has GitHub repo — merge fills gap."""
        repo = init_repo(tmp_path / "quarkmind", claude_md=(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** generic\n"
            "\n## Work Tracking\n\nIssue tracking: enabled\n"
            "GitHub repo: Org/quarkmind\n"
        ))
        app = repo / "apps" / "foo"
        app.mkdir(parents=True)
        (app / "CLAUDE.md").write_text(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** java\n"
        )
        workspace = init_repo(tmp_path / "quarkmind-foo")
        (app / "wksp").symlink_to(workspace)
        subprocess.run(["git", "-C", str(repo), "add", "-A"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "add app"],
                       cwd=str(repo), capture_output=True)
        result = run_ctx(app)
        data = parse(result)
        assert data["OWNER_REPO"] == "Org/quarkmind"
        assert data["ISSUES_STATUS"] == "enabled"

    def test_empty_scope_field_not_overridden_by_root(self, tmp_path):
        """Empty field in scope is NOT replaced by root's value."""
        repo = init_repo(tmp_path / "quarkmind", claude_md=(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** generic\n"
            "**Name:** quarkmind\n"
        ))
        app = repo / "apps" / "foo"
        app.mkdir(parents=True)
        (app / "CLAUDE.md").write_text(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** java\n"
            "**Name:** foo\n"
        )
        workspace = init_repo(tmp_path / "quarkmind-foo")
        (app / "wksp").symlink_to(workspace)
        subprocess.run(["git", "-C", str(repo), "add", "-A"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "add app"],
                       cwd=str(repo), capture_output=True)
        result = run_ctx(app)
        data = parse(result)
        assert data["PROJECT_NAME"] == "foo"

    def test_single_repo_regression_output_unchanged(self, tmp_path):
        """Single-repo output must include GIT_ROOT but otherwise be identical."""
        repo = init_repo(tmp_path / "repo", claude_md=(
            "# CLAUDE.md\n\n## Project Type\n\n**Type:** java\n**Stage:** pre-release\n"
            "\n## Work Tracking\n\nIssue tracking: enabled\n"
            "GitHub repo: Org/repo\n"
        ))
        result = run_ctx(repo)
        data = parse(result)
        assert data["GIT_ROOT"] == data["PROJECT"]
        assert data["PROJECT_TYPE"] == "java"
        assert data["OWNER_REPO"] == "Org/repo"
        assert data["ISSUES_STATUS"] == "enabled"
        assert data["CLAUDE_OK"] == "yes"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_ctx.py::TestSubfolderScope -v`
Expected: FAIL — no GIT_ROOT key, ctx.py reads CLAUDE.md from git root only

- [ ] **Step 3: Implement CLAUDE.md merge and GIT_ROOT emission in ctx.py**

In `project/ctx.py`, replace the CLAUDE.md reading section (around lines 98-128):

```python
# CLAUDE.md — scope-wins merge with root fallback
scope_md = topo.project / "CLAUDE.md"
scope_text = scope_md.read_text() if scope_md.exists() else ""
scope_text_clean = scope_text.replace("**", "")

root_md = topo.git_root / "CLAUDE.md"
root_text = root_md.read_text() if (root_md.exists() and root_md != scope_md) else ""
root_text_clean = root_text.replace("**", "")

# Merge helper: scope wins, root fills gaps. Uses is-not-None to
# distinguish "field present but empty" from "field absent."
def _merge_field(pattern: str, scope: str, root: str) -> str:
    m = re.search(pattern, scope, re.MULTILINE)
    if m is not None:
        return m.group(1).strip() if m.group(1) else ""
    m = re.search(pattern, root, re.MULTILINE)
    return m.group(1).strip() if m else ""

# Use scope_text for section presence checks (## headers)
claude_text = scope_text  # backward compat for section presence checks
claude_text_clean = scope_text_clean

m_owner = re.search(r"GitHub repo:\s*(\S+)", scope_text_clean)
if not m_owner:
    m_owner = re.search(r"GitHub repo:\s*(\S+)", root_text_clean)
owner_repo = m_owner.group(1) if m_owner else ""

m_base = re.search(r"Project base branch:\s*`([^`]+)`", scope_text_clean)
if not m_base:
    m_base = re.search(r"Project base branch:\s*`([^`]+)`", root_text_clean)
base_branch = m_base.group(1) if m_base else "main"
```

Then for `claude_ok`, check both scope and root:
```python
claude_ok = "yes" if ("## Project Type" in scope_text or "## Project Type" in root_text) else "no"
```

For `project_type` and `maturity_stage`, search scope first, then root:
```python
project_type = ""
maturity_stage = "pre-release"
for text in [scope_text, root_text]:
    if not project_type and "## Project Type" in text:
        m = re.search(r"(?:^type:\s*|^\*\*Type:\*\*\s*)(.+)", text, re.MULTILINE)
        if m:
            project_type = re.sub(r'\s*,\s*', ',', m.group(1).strip())
    if maturity_stage == "pre-release":
        m = re.search(r"(?:^stage:\s*|^\*\*Stage:\*\*\s*)(\S+)", text, re.MULTILINE)
        if m:
            maturity_stage = m.group(1).lower()
```

For `issues_status`, search scope first, then root:
```python
issues_status = "absent"
for text_clean in [scope_text_clean, root_text_clean]:
    if issues_status == "absent":
        if "Issue tracking: enabled" in text_clean:
            issues_status = "enabled"
        elif "Issue tracking: declined" in text_clean:
            issues_status = "declined"
```

For other fields (`github_project`, `project_name`, `blog_dir`, etc.), apply the same scope-first-root-fallback pattern.

Add `GIT_ROOT` to the output dict:
```python
"GIT_ROOT": str(topo.git_root),
```

Update symlink checks to use `topo.project` instead of only CWD:
```python
scope_path = topo.project
wksp_ok = (scope_path / "wksp").is_symlink() and (scope_path / "wksp").is_dir()
proj_ok = (scope_path / "proj").is_symlink() and (scope_path / "proj").is_dir()
# Also check CWD for backward compat
cwd_path = Path(cwd) if cwd else Path.cwd()
wksp_ok = wksp_ok or ((cwd_path / "wksp").is_symlink() and (cwd_path / "wksp").is_dir())
proj_ok = proj_ok or ((cwd_path / "proj").is_symlink() and (cwd_path / "proj").is_dir())
```

- [ ] **Step 4: Run ALL tests**

Run: `python3 -m pytest tests/test_ctx.py tests/test_topology.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add project/ctx.py tests/test_ctx.py
git commit -m "feat(#333): CLAUDE.md scope-root merge, GIT_ROOT emission in ctx.py"
```

---

## Batch 3: Workspace-init subfolder detection and setup

### Task 4: Subfolder detection and workspace creation in workspace-init

**Files:**
- Modify: `workspace-init/workspace_create.py:65-137` (resolve_workspace, ensure_workspace)
- Test: `tests/test_workspace_init_scripts.py`

**Interfaces:**
- Consumes: `resolve_workspace()`, `ensure_workspace()` from workspace_create.py
- Produces: `detect_subfolder(cwd)` function, updated `ensure_workspace()` that handles app folders

- [ ] **Step 1: Write failing tests for subfolder detection**

```python
# tests/test_workspace_init_scripts.py — add tests

def test_detect_subfolder_when_cwd_not_git_root(self, tmp_path):
    """CWD inside repo but not root → detected as subfolder."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    app.mkdir(parents=True)
    sys.path.insert(0, str(Path(__file__).parent.parent / "workspace-init"))
    import importlib, workspace_create
    importlib.reload(workspace_create)
    result = workspace_create.detect_subfolder(app)
    assert result is not None
    assert result["app_dir"] == app.resolve()
    assert result["git_root"] == repo.resolve()
    assert result["app_name"] == "foo"
    assert result["repo_name"] == "quarkmind"

def test_detect_subfolder_when_cwd_is_git_root(self, tmp_path):
    """CWD is git root → not a subfolder."""
    repo = init_repo(tmp_path / "quarkmind")
    sys.path.insert(0, str(Path(__file__).parent.parent / "workspace-init"))
    import importlib, workspace_create
    importlib.reload(workspace_create)
    result = workspace_create.detect_subfolder(repo)
    assert result is None

def test_ensure_workspace_for_subfolder(self, tmp_path):
    """ensure_workspace with app subfolder creates scoped workspace."""
    repo = init_repo(tmp_path / "quarkmind")
    app = repo / "apps" / "foo"
    app.mkdir(parents=True)
    home = tmp_path / "home"
    home.mkdir()
    sys.path.insert(0, str(Path(__file__).parent.parent / "workspace-init"))
    import importlib, workspace_create
    importlib.reload(workspace_create)
    ws = workspace_create.ensure_workspace(app, home_override=home)
    assert ws.exists()
    assert (ws / "proj").is_symlink()
    assert (ws / "proj").resolve() == app.resolve()
    assert (app / "wksp").is_symlink()
    assert (app / "wksp").resolve() == ws.resolve()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_workspace_init_scripts.py::test_detect_subfolder_when_cwd_not_git_root -v`
Expected: FAIL — `detect_subfolder` does not exist

- [ ] **Step 3: Implement detect_subfolder and update ensure_workspace**

Add to `workspace-init/workspace_create.py`:

```python
def detect_subfolder(cwd: Path) -> dict | None:
    """Check if cwd is inside a git repo but not the root.

    Returns None if cwd IS the git root (normal mode).
    Returns dict with app_dir, git_root, app_name, repo_name if subfolder.
    """
    cwd_resolved = cwd.resolve()
    result = subprocess.run(
        ["git", "-C", str(cwd_resolved), "rev-parse", "--show-toplevel"],
        capture_output=True, text=True,
    )
    if result.returncode != 0:
        return None
    git_root = Path(result.stdout.strip()).resolve()
    if git_root == cwd_resolved:
        return None
    return {
        "app_dir": cwd_resolved,
        "git_root": git_root,
        "app_name": cwd_resolved.name,
        "repo_name": git_root.name,
    }
```

Update `ensure_workspace()` to handle subfolder projects. Add optional `home_override` parameter for testing:

```python
def ensure_workspace(project: Path, home_override: Path | None = None) -> Path:
    existing = resolve_workspace(project)
    if existing:
        marker = existing / ".workspace"
        if not marker.exists():
            write_workspace_marker(existing, project)
        return existing

    home = home_override or Path.home()

    # Check if project is a subfolder of a git repo
    subfolder = detect_subfolder(project)
    if subfolder:
        workspace_name = f"{subfolder['repo_name']}-{subfolder['app_name']}"
        parent_name = subfolder["repo_name"]
    else:
        parent_name = project.resolve().parent.name
        workspace_name = project.resolve().name

    workspace = home / "claude" / "public" / parent_name / workspace_name
    workspace.mkdir(parents=True, exist_ok=True)

    cmd_create_dirs(workspace)
    write_workspace_marker(workspace, project)

    git_dir = workspace / ".git"
    if not git_dir.exists():
        subprocess.run(["git", "init"], cwd=str(workspace), capture_output=True)
        gitignore = workspace / ".gitignore"
        if not gitignore.exists():
            gitignore.write_text(".DS_Store\n*.swp\n")
        subprocess.run(["git", "add", "-A"], cwd=str(workspace), capture_output=True)
        subprocess.run(["git", "commit", "-m", "init: workspace setup"],
                       cwd=str(workspace), capture_output=True)

    # Create bidirectional symlinks
    proj_link = workspace / "proj"
    if not proj_link.exists():
        rel_to_project = os.path.relpath(project.resolve(), workspace.resolve())
        proj_link.symlink_to(rel_to_project)
    wksp_link = project / "wksp"
    if wksp_link.is_symlink():
        wksp_link.unlink()
    rel_to_workspace = os.path.relpath(workspace.resolve(), project.resolve())
    wksp_link.symlink_to(rel_to_workspace)

    # For subfolders, add wksp to .git/info/exclude
    if subfolder:
        git_root = subfolder["git_root"]
        exclude_file = git_root / ".git" / "info" / "exclude"
        exclude_file.parent.mkdir(parents=True, exist_ok=True)
        rel_wksp = str(project.resolve().relative_to(git_root)) + "/wksp"
        existing_excludes = exclude_file.read_text() if exclude_file.exists() else ""
        if rel_wksp not in existing_excludes:
            with open(exclude_file, "a") as f:
                f.write(f"\n{rel_wksp}\n")

    return workspace
```

- [ ] **Step 4: Run ALL workspace-init tests**

Run: `python3 -m pytest tests/test_workspace_init_scripts.py -v`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite to verify no regressions**

Run: `python3 -m pytest tests/test_topology.py tests/test_ctx.py tests/test_workspace_init_scripts.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add workspace-init/workspace_create.py tests/test_workspace_init_scripts.py
git commit -m "feat(#333): subfolder detection and scoped workspace creation"
```

---

## References

- [2026-09-05-subfolder-scoped-workspaces-design.md] — design spec this plan implements
- [project/topology.py] — current path resolution
- [project/ctx.py] — current CLAUDE.md reading and field emission
- [workspace-init/workspace_create.py] — workspace creation logic
- [tests/test_topology.py] — existing topology tests (pattern reference)
- [tests/test_ctx.py] — existing ctx tests (pattern reference)
- [issue-220 decisions] — D3 flat dataclass pattern
- [GE-20260529-182916] — ctx.py CWD subdirectory false negatives
- [PP-20260609-df21ed] — externalised scripts require tests protocol
- [GitHub #333] — focal issue

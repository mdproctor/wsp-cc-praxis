# Slot Workspace Convergence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** To be filed before implementation
**Issue group:** Covers workspace layout, detection, migration, work-end convergence, session orientation

**Goal:** Make slots use per-repo workspace clones so skills see identical structure inside and outside slots, eliminating 12+ divergence points.

**Architecture:** Fix `resolve_workspace_source()` to use `git rev-parse --show-toplevel` instead of parent-first `.git` check. Introduce `.workspace` marker for structural detection. Migrate existing slots with backwards-compatible symlinks. Converge work-end to a shared parameterized flow with topology-specific transport adapters.

**Tech Stack:** Python 3.14, pytest, git CLI, slot_manager.py (~1700 lines), work_end_execute.py (~600 lines)

## Global Constraints

- All changes in `work-slot/slot_manager.py` unless otherwise noted
- Existing tests in `tests/test_slot_manager.py` (~3600 lines) must continue to pass
- No breaking changes to `.slot` file format
- No changes to `.plan` location or format
- `git clone --shared` for all clones (disk efficiency)
- `.workspace` marker is a plain empty file (`touch`)

---

## Batch 1: Detection Bootstrap

What's working after this batch: `.workspace` marker is the primary
detection signal. `is_workspace_clone()` and `is_project_repo()` use
structural detection. All existing workspace clones across all slots
have markers. Existing tests pass. No layout changes yet.

### Task 1: `.workspace` marker detection

**Files:**
- Modify: `work-slot/slot_manager.py:880-894` (`is_workspace_clone`)
- Modify: `work-slot/slot_manager.py:872-877` (`is_project_repo`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `is_workspace_clone(repo_path: Path) -> bool` — checks `.workspace` marker as primary signal, falls back to name-based for transition period
- Produces: `is_project_repo(name: str, repo_path: Path | None = None) -> bool` — adds optional path parameter to check `.workspace` marker

- [ ] **Step 1: Write failing test for marker-based detection**

```python
class TestWorkspaceMarkerDetection:
    def test_workspace_marker_detected(self, tmp_path):
        ws = tmp_path / "wsp-casehub-connectors"
        ws.mkdir()
        (ws / ".git").mkdir()
        (ws / ".workspace").touch()
        assert is_workspace_clone(ws) is True

    def test_no_marker_no_naming_convention_is_project(self, tmp_path):
        repo = tmp_path / "my-custom-workspace"
        repo.mkdir()
        (repo / ".git").mkdir()
        (repo / ".workspace").touch()
        assert is_workspace_clone(repo) is True

    def test_marker_takes_precedence_over_name(self, tmp_path):
        repo = tmp_path / "connectors"
        repo.mkdir()
        (repo / ".git").mkdir()
        (repo / ".workspace").touch()
        assert is_workspace_clone(repo) is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestWorkspaceMarkerDetection -v`
Expected: `test_no_marker_no_naming_convention_is_project` FAILS (name doesn't match `work-*`), `test_marker_takes_precedence_over_name` FAILS (name `connectors` passes `is_project_repo`)

- [ ] **Step 3: Update `is_workspace_clone` to check marker first**

```python
def is_workspace_clone(repo_path: Path) -> bool:
    if not repo_path.is_dir():
        return False
    if (repo_path / ".workspace").exists():
        return True
    # Transition fallback — remove after all workspaces have markers
    if (repo_path / "proj").is_symlink():
        return True
    return not is_project_repo(repo_path.name)
```

No functional change yet — marker check was already first. The key change is documenting the transition plan in the comment.

- [ ] **Step 4: Run all existing slot tests to verify nothing breaks**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "feat: .workspace marker detection with transition fallback Refs #<N>"
```

### Task 2: Marker placement script

**Files:**
- Create: `scripts/place_workspace_markers.py`
- Test: `tests/test_place_workspace_markers.py`

**Interfaces:**
- Produces: `place_markers(family_root: Path, include_attic: bool = True) -> dict` — returns `{"placed": int, "already_marked": int, "skipped": int}`
- Consumes: `is_workspace_clone(repo_path)` from Task 1

- [ ] **Step 1: Write failing test**

```python
import pytest
from pathlib import Path

class TestPlaceWorkspaceMarkers:
    def _make_slot(self, tmp_path, slot_num, repos, workspaces):
        slot_dir = tmp_path / "slots" / str(slot_num)
        slot_dir.mkdir(parents=True)
        for r in repos:
            d = slot_dir / r
            d.mkdir()
            (d / ".git").mkdir()
        for w in workspaces:
            d = slot_dir / w
            d.mkdir()
            (d / ".git").mkdir()
        return slot_dir

    def test_places_markers_on_work_prefixed_dirs(self, tmp_path):
        self._make_slot(tmp_path, 1, ["engine"], ["work-casehub"])
        from scripts.place_workspace_markers import place_markers
        result = place_markers(tmp_path)
        assert (tmp_path / "slots" / "1" / "work-casehub" / ".workspace").exists()
        assert result["placed"] == 1

    def test_skips_project_repos(self, tmp_path):
        self._make_slot(tmp_path, 1, ["engine"], ["work-casehub"])
        from scripts.place_workspace_markers import place_markers
        place_markers(tmp_path)
        assert not (tmp_path / "slots" / "1" / "engine" / ".workspace").exists()

    def test_idempotent(self, tmp_path):
        self._make_slot(tmp_path, 1, ["engine"], ["work-casehub"])
        from scripts.place_workspace_markers import place_markers
        place_markers(tmp_path)
        result = place_markers(tmp_path)
        assert result["placed"] == 0
        assert result["already_marked"] == 1

    def test_includes_attic(self, tmp_path):
        attic_dir = tmp_path / "slots" / "attic" / "5"
        attic_dir.mkdir(parents=True)
        ws = attic_dir / "work-casehub"
        ws.mkdir()
        (ws / ".git").mkdir()
        from scripts.place_workspace_markers import place_markers
        result = place_markers(tmp_path)
        assert (ws / ".workspace").exists()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_place_workspace_markers.py -v`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement marker placement script**

```python
#!/usr/bin/env python3
"""Place .workspace markers on all workspace clones across all slots."""
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
from slot_manager import is_workspace_clone, is_project_repo


def place_markers(family_root: Path, include_attic: bool = True) -> dict:
    result = {"placed": 0, "already_marked": 0, "skipped": 0}
    slots_dir = family_root / "slots"
    if not slots_dir.is_dir():
        return result

    slot_dirs = []
    for entry in sorted(slots_dir.iterdir()):
        if entry.name == "attic":
            if include_attic and entry.is_dir():
                for archived in sorted(entry.iterdir()):
                    if archived.is_dir():
                        slot_dirs.append(archived)
            continue
        if entry.is_dir():
            slot_dirs.append(entry)

    for slot_dir in slot_dirs:
        for child in sorted(slot_dir.iterdir()):
            if not child.is_dir() or not (child / ".git").exists():
                continue
            if child.name in (".m2", "attic"):
                continue
            marker = child / ".workspace"
            if marker.exists():
                result["already_marked"] += 1
                continue
            # Use current detection (name-based) to identify workspaces
            if not is_project_repo(child.name) or (child / "proj").is_symlink():
                marker.touch()
                result["placed"] += 1
            else:
                result["skipped"] += 1

    return result


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: place_workspace_markers.py <family-root>")
        sys.exit(1)
    root = Path(sys.argv[1])
    res = place_markers(root)
    print(f"Placed: {res['placed']}, Already marked: {res['already_marked']}, Skipped: {res['skipped']}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_place_workspace_markers.py -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/place_workspace_markers.py tests/test_place_workspace_markers.py
git commit -m "feat: marker placement script for .workspace bootstrap Refs #<N>"
```

### Task 2b: workspace-init marker placement

**Files:**
- Modify: `workspace-init/SKILL.md`

**Interfaces:**
- Produces: Updated skill text that places `.workspace` marker during workspace repo creation and catches up existing repos

- [ ] **Step 1: Add `.workspace` marker to workspace-init repo creation**

In workspace-init/SKILL.md, find the workspace repo creation step (after
`git init` or `git clone`). Add `touch .workspace` and commit as part of
initial setup.

- [ ] **Step 2: Add idempotent catch-up**

In the symlink setup section, add: if workspace repo exists but lacks
`.workspace`, place the marker. This catches repos created before this
change.

- [ ] **Step 3: Commit**

```bash
git add workspace-init/SKILL.md
git commit -m "feat: workspace-init places .workspace marker Refs #<N>"
```

---

## Batch 2: Fix Slot Creation

What's working after this batch: New slots use per-repo workspace clones
with correct names derived from git remote URLs. `resolve_workspace_source`
uses `git rev-parse --show-toplevel`. `create_slot` places `.workspace`
markers. Old slots still work via transition fallback.

### Task 3: Fix `resolve_workspace_source`

**Files:**
- Modify: `work-slot/slot_manager.py:309-320` (`resolve_workspace_source`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `resolve_workspace_source(repo_path: Path) -> tuple[Path, str] | None` — returns `(workspace_repo_root, slot_name)` where `slot_name` is derived from git remote URL (e.g., `wsp-casehub-connectors`)
- Consumes: `run_cmd` (existing), `_get_clone_origin` (existing)

- [ ] **Step 1: Write failing test for parent-first bug**

```python
class TestResolveWorkspaceSourceGitToplevel:
    def test_resolves_to_child_not_parent(self, tmp_path):
        """When child workspace is nested inside a parent git repo,
        resolve to the child (the actual workspace repo)."""
        parent = tmp_path / "public" / "casehub"
        parent.mkdir(parents=True)
        (parent / ".git").mkdir()  # parent is also a git repo

        child = parent / "connectors"
        child.mkdir()
        (child / ".git").mkdir()  # child is its own git repo

        project = tmp_path / "casehub" / "connectors"
        project.mkdir(parents=True)
        wksp = project / "wksp"
        wksp.symlink_to(child)

        source = resolve_workspace_source(project)
        assert source is not None
        ws_path, ws_name = source
        assert ws_path == child  # NOT parent

    def test_name_from_remote_url(self, tmp_path):
        """Slot name derived from workspace repo's origin remote URL."""
        ws_repo = tmp_path / "workspace"
        ws_repo.mkdir()
        (ws_repo / ".git").mkdir()

        project = tmp_path / "project"
        project.mkdir()
        wksp = project / "wksp"
        wksp.symlink_to(ws_repo)

        # Mock git commands
        # ... (will need to mock run_cmd for git rev-parse and git remote)

    def test_fallback_name_when_no_remote(self, tmp_path):
        """When workspace repo has no remote, construct name from path."""
        ws_repo = tmp_path / "public" / "casehub" / "connectors"
        ws_repo.mkdir(parents=True)
        (ws_repo / ".git").mkdir()

        project = tmp_path / "project"
        project.mkdir()
        wksp = project / "wksp"
        wksp.symlink_to(ws_repo)

        # With no remote, should produce wsp-casehub-connectors
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestResolveWorkspaceSourceGitToplevel -v`
Expected: FAIL — `ws_path == parent` instead of `child`

- [ ] **Step 3: Implement fix**

Replace `resolve_workspace_source`:

```python
def resolve_workspace_source(repo_path: Path) -> tuple[Path, str] | None:
    wksp = repo_path / "wksp"
    if not wksp.is_symlink():
        return None
    target = wksp.resolve()
    if not target.is_dir():
        return None

    # Use git rev-parse to find the actual repo root (not parent)
    rc, stdout, _ = run_cmd(["git", "-C", str(target), "rev-parse", "--show-toplevel"])
    if rc != 0:
        return None
    ws_root = Path(stdout.strip())

    # Derive slot name from remote URL (unique, no collisions)
    rc, url_out, _ = run_cmd(["git", "-C", str(ws_root), "remote", "get-url", "origin"])
    if rc == 0 and url_out.strip():
        url = url_out.strip()
        # Extract repo name from URL: "https://.../<name>.git" or "git@...:<name>.git"
        name = Path(url.rstrip("/")).stem  # strips .git extension
        return ws_root, name

    # Fallback: construct from path — wsp-<parent>-<name>
    parent_name = ws_root.parent.name
    ws_name = f"wsp-{parent_name}-{ws_root.name}"
    return ws_root, ws_name
```

- [ ] **Step 4: Update existing `resolve_workspace_source` tests**

The existing `TestResolveWorkspaceSource` tests (lines 83-110) test the old parent-first behavior. Update them to expect the new git-toplevel behavior. Some tests may need git init to make `git rev-parse` work in tmp_path.

- [ ] **Step 5: Run all slot tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "fix: resolve_workspace_source uses git-toplevel, not parent-first Refs #<N>"
```

### Task 4: Update `create_slot` for per-repo workspaces

**Files:**
- Modify: `work-slot/slot_manager.py:547-711` (`create_slot`)
- Modify: `work-slot/slot_manager.py:431-444` (`repoint_wksp`, `create_proj_symlink`)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `resolve_workspace_source(repo_path)` from Task 3
- Consumes: `is_workspace_clone(repo_path)` from Task 1
- Produces: Updated `create_slot` that creates per-repo workspace clones with `.workspace` markers

- [ ] **Step 1: Write failing test for per-repo workspace layout**

```python
class TestCreateSlotPerRepoWorkspace:
    def test_creates_workspace_clone_per_repo(self, mock_cmd, tmp_path):
        """Each repo gets its own workspace clone, not a shared family clone."""
        # Setup: two repos with different workspace repos
        # ... (mock resolve_workspace_source to return per-repo paths)
        # Assert: slot has wsp-casehub-connectors/ and wsp-casehub-pages/
        # Assert: no work-casehub/ directory exists

    def test_workspace_marker_placed(self, mock_cmd, tmp_path):
        """Each workspace clone has .workspace marker."""
        # Assert: (slot_dir / "wsp-casehub-connectors" / ".workspace").exists()

    def test_wksp_symlink_points_to_sibling(self, mock_cmd, tmp_path):
        """wksp symlink points to sibling workspace dir, not subdirectory."""
        # Assert: connectors/wksp resolves to ../wsp-casehub-connectors/

    def test_proj_symlink_points_back(self, mock_cmd, tmp_path):
        """proj symlink in workspace points back to project clone."""
        # Assert: wsp-casehub-connectors/proj resolves to ../connectors/

    def test_no_ws_created_dedup(self, mock_cmd, tmp_path):
        """With correct resolution, each project resolves to its own
        workspace — no deduplication needed."""
        # Two repos, each with distinct workspace repo
        # Both get cloned independently
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotPerRepoWorkspace -v`
Expected: FAIL

- [ ] **Step 3: Update `create_slot`**

Key changes to the workspace cloning section (lines 599-642):
- Remove `ws_created` deduplication dict
- Clone each workspace independently using the name from `resolve_workspace_source`
- Place `.workspace` marker after cloning
- `repoint_wksp` to point to sibling directory (not subdirectory)
- `create_proj_symlink` at workspace root (not subdirectory)
- Remove `_unignore_subdir` call (no subdirectory nesting)
- Add collision detection: if `slot_workspace_name` already exists as a directory, error with diagnostic

- [ ] **Step 4: Update `validate_slot_wksp` for flat layout**

The validation function needs to check that `wksp` points to a sibling directory, not a subdirectory of a family workspace.

- [ ] **Step 5: Run all slot tests**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All PASS (update existing `create_slot` tests for new layout)

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "feat: create_slot uses per-repo workspace clones Refs #<N>"
```

---

## Batch 3: Active Slot Migration

What's working after this batch: A migration script can restructure
active slots from family workspace clone to per-repo workspace clones
with backwards-compatible symlinks. Tested on real slot structures.

### Task 5: Migration script

**Files:**
- Create: `scripts/migrate_slot_workspace.py`
- Test: `tests/test_migrate_slot_workspace.py`

**Interfaces:**
- Produces: `migrate_slot(slot_dir: Path, family_root: Path, dry_run: bool = False) -> dict` — returns `{"status": "migrated"|"already_migrated"|"skipped", "repos_migrated": list[str], "errors": list[str]}`
- Consumes: `resolve_workspace_source` from Task 3, `is_workspace_clone` from Task 1

- [ ] **Step 1: Write failing tests for migration**

```python
class TestMigrateSlotWorkspace:
    def test_migrates_family_workspace_to_per_repo(self, tmp_path):
        """Creates per-repo workspace clones, replaces family subdirs
        with symlinks, places .workspace markers."""

    def test_replays_branch_commits(self, tmp_path):
        """Commits on the feature branch in work-casehub/connectors/
        are replayed into wsp-casehub-connectors/ via format-patch."""

    def test_symlink_bridge_for_backwards_compat(self, tmp_path):
        """work-casehub/connectors/ becomes a symlink to
        ../wsp-casehub-connectors/ after migration."""

    def test_repoints_wksp_symlink(self, tmp_path):
        """connectors/wksp points to ../wsp-casehub-connectors/
        after migration, not ../work-casehub/connectors/."""

    def test_idempotent(self, tmp_path):
        """Running migration twice produces same result."""

    def test_resumes_from_progress_file(self, tmp_path):
        """If .migration-progress exists with partial completion,
        only migrates remaining repos."""

    def test_commits_symlinks_to_family_clone(self, tmp_path):
        """The family workspace clone gets a commit recording the
        symlink replacements to keep its working tree clean."""

    def test_no_patches_when_no_branch_commits(self, tmp_path):
        """Fresh clone on feature branch with no commits beyond main
        doesn't need format-patch replay."""

    def test_dry_run_reports_without_changing(self, tmp_path):
        """dry_run=True reports what would change without touching disk."""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_migrate_slot_workspace.py -v`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement migration script**

Core algorithm:
1. Detect old structure: slot has `work-*` or `work` directory that is a git repo
2. Read `.migration-progress` if exists (resume support)
3. For each project repo in slot (via `get_slot_repos`):
   a. Resolve original workspace repo via original project's `wksp` symlink
   b. `git clone --shared` from original workspace repo
   c. Check out feature branch
   d. `git format-patch --relative=<subdir>/ main..feature-branch -- <subdir>/`
   e. `git am *.patch` in new clone
   f. Place `.workspace` marker
   g. Repoint `wksp` symlink
   h. Replace family workspace subdirectory with symlink
   i. Record progress
4. Commit symlink replacements to family workspace clone

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_migrate_slot_workspace.py -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/migrate_slot_workspace.py tests/test_migrate_slot_workspace.py
git commit -m "feat: slot workspace migration script with replay and resume Refs #<N>"
```

---

## Batch 4: Session Orientation

What's working after this batch: Slot sessions are told their full repo
scope on resume. work-start surfaces all repos and workspace paths.
work-slot documents the scope rule.

### Task 6: Slot orientation in work-start and work-slot

**Files:**
- Modify: `work-start/SKILL.md` (slot resume path)
- Modify: `work-slot/SKILL.md` (add Slot Scope section)

**Interfaces:**
- Consumes: `.slot` file format (repo list)
- Produces: Updated skill text that communicates full slot scope

- [ ] **Step 1: Update work-start slot resume path**

In work-start/SKILL.md, find the slot resume path (where `IN_SLOT=yes`).
Add orientation output:

```markdown
When `IN_SLOT=yes`, read `.slot` to enumerate all repos and their
workspace pairs. Present to the session:

> **Slot scope:** This session has write access to all repos in slot <N>:
> - connectors/ (primary) ← workspace: wsp-casehub-connectors/
> - pages/ ← workspace: wsp-casehub-pages/
> - examples/ ← workspace: wsp-casehub-examples/
>
> Write to the repo tagged in `.plan` for each issue. Artifacts go in
> that repo's workspace. Cross-cutting artifacts go in the primary
> workspace.
```

- [ ] **Step 2: Add Slot Scope section to work-slot/SKILL.md**

After the "How slots work" section, add:

```markdown
### Slot Scope

A session in a slot owns ALL repos in that slot. The session can and
should write to any cloned repo — that's the point of slots.

- **Issue routing:** `.plan` entries are tagged with their target repo
  (e.g., `[pages]`). Work goes in the tagged repo, regardless of which
  repo the session was started in.
- **Artifact routing:** Each repo's workspace artifacts go in that repo's
  workspace clone. Cross-cutting artifacts go in the primary workspace,
  tagged for scope.
- **CWD is not scope:** Starting in `slots/138/connectors/` does not
  limit the session to connectors. It simply means connectors is the
  shell's working directory.
```

- [ ] **Step 3: Commit**

```bash
git add work-start/SKILL.md work-slot/SKILL.md
git commit -m "feat: slot session orientation — document full repo scope Refs #<N>"
```

---

## Batch 5: Work-end Convergence (Phase 3)

What's working after this batch: Work-end uses a shared parameterized
flow for both slot and branch modes. `merge_slot` is decomposed into
shared flow + transport helpers. Workspace repos are merged, pushed,
and stamped in slot mode (fixing the current gap).

**Note:** This is the largest and riskiest batch. It should only be
started after Batches 1-4 are stable and all active slots have been
migrated (or have symlink bridges in place). The 14-day sunset timer
starts when this batch lands.

### Task 7: Repo descriptor and shared flow

**Files:**
- Create: `work-end/land_flow.py`
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Produces: `RepoDescriptor` dataclass with fields: `repo_path`, `original_path`, `push_target`, `base_branch`, `is_workspace`, `transport`
- Produces: `land_batch(descriptors: list[RepoDescriptor], progress_file: Path) -> LandResult`
- Produces: `LandResult` dataclass with per-repo status

- [ ] **Step 1: Write failing test for repo descriptor construction**

```python
from dataclasses import dataclass
from pathlib import Path

class TestRepoDescriptor:
    def test_slot_adapter_builds_descriptors(self):
        """Slot adapter creates descriptors with two-hop transport."""

    def test_branch_adapter_builds_descriptors(self):
        """Branch adapter creates descriptors with direct transport."""

    def test_project_repos_ordered_before_workspace(self):
        """In the batch, project repos come before workspace repos."""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_flow.py -v`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement repo descriptor and shared flow skeleton**

```python
from dataclasses import dataclass
from pathlib import Path
from enum import Enum

class Transport(Enum):
    DIRECT = "direct"
    TWO_HOP = "two-hop"

@dataclass
class RepoDescriptor:
    repo_path: Path
    original_path: Path
    push_target: str
    base_branch: str
    is_workspace: bool
    transport: Transport

@dataclass
class RepoStatus:
    repo_path: Path
    merged: bool = False
    pushed: bool = False
    stamped: bool = False
    error: str = ""

@dataclass
class LandResult:
    repos: list[RepoStatus]
    success: bool = True
```

- [ ] **Step 4: Implement shared flow steps**

The 5 steps: preflight, rebase, merge+push, stamp, record.
Each step operates on the batch of descriptors. Progress is tracked
in `.execute-progress` for crash recovery.

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_flow.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/land_flow.py tests/test_land_flow.py
git commit -m "feat: shared land flow with repo descriptors Refs #<N>"
```

### Task 8: Slot and branch adapters

**Files:**
- Modify: `work-end/land_flow.py`
- Modify: `work-slot/slot_manager.py` (extract transport helpers from `merge_slot`)
- Modify: `work-end/work_end_execute.py` (`cmd_land` becomes thin adapter)
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Produces: `build_slot_batch(slot_dir: Path, base_branch: str) -> list[RepoDescriptor]`
- Produces: `build_branch_batch(project_path: Path, workspace_path: Path | None, base_branch: str) -> list[RepoDescriptor]`
- Consumes: `land_batch` from Task 7, `get_all_slot_repos` and `is_workspace_clone` from slot_manager

- [ ] **Step 1: Write failing tests for adapters**

```python
class TestSlotAdapter:
    def test_includes_workspace_repos(self):
        """Slot adapter includes both project and workspace repos."""

    def test_two_hop_transport(self):
        """All slot descriptors use TWO_HOP transport."""

    def test_project_repos_before_workspace(self):
        """Project repos ordered before workspace repos."""

class TestBranchAdapter:
    def test_direct_transport(self):
        """Branch descriptors use DIRECT transport."""

    def test_includes_workspace_when_present(self):
        """Branch adapter includes workspace if provided."""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_flow.py -v`
Expected: FAIL

- [ ] **Step 3: Implement adapters**

Extract transport helpers from `merge_slot` (preflight sync, two-hop push, SHA verification) into standalone functions. Wire them into the shared flow via the transport field on descriptors.

Refactor `cmd_land` in `work_end_execute.py` to build a branch batch and call `land_batch`.

- [ ] **Step 4: Run all work-end and slot tests**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_verify_slot_close.py -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/land_flow.py work-end/work_end_execute.py work-slot/slot_manager.py tests/test_land_flow.py
git commit -m "feat: slot and branch adapters for shared land flow Refs #<N>"
```

### Task 9: Update work-end/SKILL.md

**Files:**
- Modify: `work-end/SKILL.md`

**Interfaces:**
- Consumes: `land_flow.py` from Tasks 7-8

- [ ] **Step 1: Fix `get_all_slot_repos` claim at line 402-403**

The SKILL.md incorrectly claims `merge_slot` stamps workspace repos via
`get_all_slot_repos()`. Update to reflect the shared flow model.

- [ ] **Step 2: Converge slot-specific steps to shared flow**

Update Phase C (land) to reference the shared flow. Remove separate
slot-specific orchestration instructions. Keep `.phase-a-complete`
as a SKILL.md pre-condition for slot mode.

- [ ] **Step 3: Commit**

```bash
git add work-end/SKILL.md
git commit -m "feat: work-end SKILL.md converged to shared land flow Refs #<N>"
```

---

## Batch 6: Cleanup + Sunset

What's working after this batch: All transition code removed. Detection
is purely structural. Dead code eliminated. This batch lands after the
14-day sunset period.

### Task 10: Remove transition code

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: confirmation that 14-day sunset has passed

- [ ] **Step 1: Remove `_unignore_subdir` function**

Dead code — no subdirectory nesting in new layout.

- [ ] **Step 2: Remove name-based fallback from `is_workspace_clone`**

```python
def is_workspace_clone(repo_path: Path) -> bool:
    if not repo_path.is_dir():
        return False
    return (repo_path / ".workspace").exists()
```

- [ ] **Step 3: Simplify `is_project_repo`**

```python
def is_project_repo(name: str, repo_path: Path | None = None) -> bool:
    if name in (".m2", "attic"):
        return False
    if repo_path and is_workspace_clone(repo_path):
        return False
    return True
```

- [ ] **Step 4: Remove `ensure_clone_layout` legacy migration**

- [ ] **Step 5: Remove `ws_created` dedup remnants if any remain**

- [ ] **Step 6: Update tests — remove tests for removed functions**

- [ ] **Step 7: Run full test suite**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "chore: remove transition code after 14-day sunset Refs #<N>"
```

---

## References

- [2026-08-19-slot-workspace-convergence-design.md] — design spec this plan implements
- [decisions.md] — D1-D7 validated decisions
- `work-slot/slot_manager.py:309-320` — `resolve_workspace_source` (root cause)
- `work-slot/slot_manager.py:547-711` — `create_slot`
- `work-slot/slot_manager.py:872-912` — detection functions
- `work-slot/slot_manager.py:1163-1458` — `merge_slot`
- `work-end/work_end_execute.py:226-414` — `cmd_land`
- `tests/test_slot_manager.py` — existing test suite (~3600 lines)

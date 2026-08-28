# slot_manager.py Decomposition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #306 — refactor: slot_manager.py stability audit and decomposition
**Issue group:** #306

**Goal:** Decompose `work-slot/slot_manager.py` (2,291 lines, 67 functions) into 11 focused modules, preserving 298+ tests and the CLI contract.

**Architecture:** Incremental extraction following dependency depth — leaf modules first, orchestrators last. Each extraction is one commit with the new module, its tests, and all consumer import updates. `slot_manager.py` becomes a 3-line shim at the end.

**Tech Stack:** Python 3.12+, pytest, no new dependencies

## Global Constraints

- Full test suite must pass after every extraction: `python3 -m pytest tests/test_slot_manager.py -v`
- CLI contract (KEY=VALUE output on stdout) must not change
- All 11 external Python API import sites must be updated in the same commit as the extraction
- No re-export facades — consumers import directly from new modules
- Protocol: `archive_slot()` must preserve `.artifacts-promoted` stamp check
- All new modules live in `work-slot/`; all new test files live in `tests/`
- Shared test helpers extracted to `tests/slot_test_helpers.py` in the first batch

---

## Batch 0: Test infrastructure — shared helpers

### Task 0: Extract shared test helpers

6 helper functions are used across many test classes and must be available before tests start moving.

**Files:**
- Create: `tests/slot_test_helpers.py`
- Modify: `tests/test_slot_manager.py` (import helpers instead of defining them inline)

- [ ] **Step 1: Create `tests/slot_test_helpers.py`**

Extract these shared helpers from `test_slot_manager.py`:

```python
import subprocess
from pathlib import Path


def init_repo(path):
    """Initialize a bare git repo with an initial commit."""
    subprocess.run(["git", "init", str(path)], capture_output=True)
    (path / "README.md").write_text("init")
    subprocess.run(["git", "-C", str(path), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "init"], capture_output=True)


def init_repo_with_workspace(path):
    """Initialize a git repo with a wksp symlink pointing to a workspace."""
    # ... copy exact body from test_slot_manager.py lines 27-33
```

Also extract: `_init_repo_with_remote`, `_create_merge_test_repos`, `_create_worktree_test_repos`, `_create_clone_test_repos`.

- [ ] **Step 2: Update `test_slot_manager.py`**

Replace inline helper definitions with:
```python
from slot_test_helpers import (
    init_repo, init_repo_with_workspace, _init_repo_with_remote,
    _create_merge_test_repos, _create_worktree_test_repos,
    _create_clone_test_repos,
)
```

- [ ] **Step 3: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All 298+ tests pass unchanged

- [ ] **Step 4: Commit**

```bash
git add tests/slot_test_helpers.py tests/test_slot_manager.py
git commit -m "refactor: extract shared test helpers for slot_manager decomposition Refs #306"
```

---

## Batch 1: Foundation — slot_core.py

### Task 1: Extract slot_core.py

Extract the 16 shared utilities, constants, and the exception class that every other module depends on.

**Files:**
- Create: `work-slot/slot_core.py`
- Create: `tests/test_slot_core.py`
- Modify: `work-slot/slot_manager.py` (remove extracted functions, add `from slot_core import ...`)
- Modify: `work-pause/pause_exec.py` (update `from slot_manager import is_slot_path` → `from slot_core import is_slot_path`)
- Modify: `scripts/place_workspace_markers.py` (update `from slot_manager import is_project_repo` → `from slot_core import is_project_repo`)
- Modify: `scripts/migrate_slot_workspace.py` (update `from slot_manager import is_workspace_clone, get_slot_repos` → `from slot_core import is_workspace_clone, get_slot_repos`)

**Interfaces:**
- Produces: `SLOT_DIR_NAME`, `LEGACY_SLOT_DIR_NAME`, `_IDE_ARTIFACTS`, `_REGENERABLE_DIRS`, `SlotCreationError`, `run_cmd(args, cwd=None) -> tuple[int, str, str]`, `_resolve_slots_dir(family_root: Path) -> Path`, `_resolve_slot_dir_for_number(family_root: Path, slot_num: int) -> Path`, `_get_family_repo_names(family_root: Path) -> set[str]`, `is_slot_path(path: str) -> bool`, `is_project_repo(name: str) -> bool`, `is_workspace_clone(repo_path: Path) -> bool`, `is_worktree(repo_path: Path) -> bool`, `resolve_original_repo(repo_path: Path) -> Path`, `_get_clone_origin(clone_path: Path) -> str | None`, `get_slot_repos(slot_dir: Path) -> list[str]`, `get_all_slot_repos(slot_dir: Path) -> list[str]`, `_cleanup_remnant_dir(path: Path) -> bool`, `_escape_slot_cwd(slot_dir: Path, escape_to: Path) -> tuple[bool, Path | None]`, `_has_unmerged_content(slot_dir: Path) -> list[str]`

- [ ] **Step 1: Create `work-slot/slot_core.py`**

Copy these functions and constants from `slot_manager.py` into `slot_core.py`:

Constants and imports:
```python
import os
import subprocess
import sys
from pathlib import Path

_IDE_ARTIFACTS = {".idea", ".run", ".settings", ".project", ".classpath", ".vscode"}

_REGENERABLE_DIRS = {
    "node_modules", ".gradle", "build", "dist", "target", "out",
    ".next", ".nuxt", ".cache", ".parcel-cache", ".turbo",
    *_IDE_ARTIFACTS,
}

SLOT_DIR_NAME = "slots"
LEGACY_SLOT_DIR_NAME = "worktrees"


class SlotCreationError(Exception):
    pass
```

Then copy all 15 functions listed in Produces above, preserving their exact signatures and bodies.

- [ ] **Step 2: Update `slot_manager.py`**

Remove all extracted functions, constants, and the exception class. Add at the top (after the existing imports):

```python
from slot_core import (
    SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME, _IDE_ARTIFACTS, _REGENERABLE_DIRS,
    SlotCreationError, run_cmd,
    _resolve_slots_dir, _resolve_slot_dir_for_number, _get_family_repo_names,
    is_slot_path, is_project_repo, is_workspace_clone, is_worktree,
    resolve_original_repo, _get_clone_origin,
    get_slot_repos, get_all_slot_repos,
    _cleanup_remnant_dir, _escape_slot_cwd, _has_unmerged_content,
)
```

- [ ] **Step 3: Update external consumers**

Update import statements in these files:

`work-pause/pause_exec.py`: Change `from slot_manager import is_slot_path as _is_slot_path` → `from slot_core import is_slot_path as _is_slot_path`

`scripts/place_workspace_markers.py`: Change `from slot_manager import is_project_repo` → `from slot_core import is_project_repo`

`scripts/migrate_slot_workspace.py`: Change the `from slot_manager import (...)` block — move `get_slot_repos` and `is_workspace_clone` to a separate `from slot_core import get_slot_repos, is_workspace_clone` line. Keep the remaining slot_manager imports.

- [ ] **Step 4: Create `tests/test_slot_core.py`**

Move all tests that primarily exercise the extracted functions from `test_slot_manager.py` into `test_slot_core.py`. Update imports in the moved tests from `slot_manager` to `slot_core`. Copy any shared fixtures needed by the moved tests.

- [ ] **Step 5: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py -v`
Expected: All 298+ tests pass (split across both files)

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_core.py tests/test_slot_core.py work-slot/slot_manager.py work-pause/pause_exec.py scripts/place_workspace_markers.py scripts/migrate_slot_workspace.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_core.py from slot_manager Refs #306"
```

---

## Batch 2: Data layer — slot_metadata.py

### Task 2: Extract slot_metadata.py

Extract the pure data layer: .slot file parsing/writing, promotion stamps, landed markers.

**Files:**
- Create: `work-slot/slot_metadata.py`
- Create: `tests/test_slot_metadata.py`
- Modify: `work-slot/slot_manager.py` (remove functions, add import)
- Modify: `work-end/work_end_context.py` (update `from slot_manager import parse_slot_md` → `from slot_metadata import parse_slot_md`)
- Modify: `project/topology.py` (update `from slot_manager import parse_slot_md` → `from slot_metadata import parse_slot_md`)

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_core.get_slot_repos`
- Produces: `parse_slot_md(slot_dir: Path) -> dict`, `write_slot_md(slot_dir: Path, slot_number: int, repos: list[str], ...) -> None`, `_read_promotion_stamp(slot_dir: Path) -> tuple[list[str], list[str], str]`, `is_slot_landed(slot_dir: Path) -> bool`, `verify_landed_shas(slot_dir: Path, family_root: Path) -> tuple[bool, list[str]]`, `_fix_stale_checkboxes(slot_path: Path, issues_to_tick: list[int]) -> int`

- [ ] **Step 1: Create `work-slot/slot_metadata.py`**

Copy `parse_slot_md`, `write_slot_md`, `_read_promotion_stamp`, `is_slot_landed`, `verify_landed_shas`, `_fix_stale_checkboxes` from `slot_manager.py`. Add imports:

```python
from pathlib import Path
from slot_core import run_cmd, get_slot_repos
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions. Add:
```python
from slot_metadata import (
    parse_slot_md, write_slot_md, _read_promotion_stamp,
    is_slot_landed, verify_landed_shas, _fix_stale_checkboxes,
)
```

- [ ] **Step 3: Update external consumers**

`work-end/work_end_context.py`: Change `from slot_manager import parse_slot_md, get_slot_repos` → `from slot_metadata import parse_slot_md` and `from slot_core import get_slot_repos`

`project/topology.py`: Change `from slot_manager import parse_slot_md` → `from slot_metadata import parse_slot_md`

- [ ] **Step 4: Move tests to `tests/test_slot_metadata.py`**

Move tests exercising the 6 extracted functions. Update imports.

- [ ] **Step 5: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py -v`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_metadata.py tests/test_slot_metadata.py work-slot/slot_manager.py work-end/work_end_context.py project/topology.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_metadata.py from slot_manager Refs #306"
```

---

## Batch 3: Build tooling — slot_maven.py

### Task 3: Extract slot_maven.py

Extract Maven settings generation and slot repo setup. Self-contained, no external consumers.

**Files:**
- Create: `work-slot/slot_maven.py`
- Create: `tests/test_slot_maven.py`
- Modify: `work-slot/slot_manager.py`

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_core._REGENERABLE_DIRS`
- Produces: `_write_slot_settings(slot_dir: Path) -> Path`, `setup_slot_repo(repo_worktree: Path, m2_path: Path) -> bool`

- [ ] **Step 1: Create `work-slot/slot_maven.py`**

Copy `_write_slot_settings` and `setup_slot_repo`. Add imports:
```python
from pathlib import Path
from slot_core import run_cmd, _REGENERABLE_DIRS
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions. Add:
```python
from slot_maven import _write_slot_settings, setup_slot_repo
```

- [ ] **Step 3: Move tests to `tests/test_slot_maven.py`**

- [ ] **Step 4: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_maven.py tests/test_slot_maven.py work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_maven.py from slot_manager Refs #306"
```

---

## Batch 4: ISX isolation — slot_isx.py

### Task 4: Extract slot_isx.py

Extract ISX instance lifecycle management.

**Files:**
- Create: `work-slot/slot_isx.py`
- Create: `tests/test_slot_isx.py`
- Modify: `work-slot/slot_manager.py`

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_metadata.parse_slot_md`, `slot_core.get_slot_repos`
- Produces: `_check_isx_available() -> bool`, `_truncate_instance_name(name: str, max_len: int = 63) -> str`, `_teardown_isx(slot_dir: Path) -> None`, `_wire_isx_remotes(slot_dir: Path, repos: list[str], instance: str) -> None`, `sync_isx(slot_dir: Path) -> int`

- [ ] **Step 1: Create `work-slot/slot_isx.py`**

Copy the 5 ISX functions. Add imports:
```python
from pathlib import Path
from slot_core import run_cmd, get_slot_repos
from slot_metadata import parse_slot_md
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions. Add:
```python
from slot_isx import (
    _check_isx_available, _truncate_instance_name,
    _teardown_isx, _wire_isx_remotes, sync_isx,
)
```

- [ ] **Step 3: Move tests to `tests/test_slot_isx.py`**

- [ ] **Step 4: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_isx.py tests/test_slot_isx.py work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_isx.py from slot_manager Refs #306"
```

---

## Batch 5: Claude project dirs — slot_claude.py

### Task 5: Extract slot_claude.py

Extract Claude Code project directory management.

**Files:**
- Create: `work-slot/slot_claude.py`
- Create: `tests/test_slot_claude.py`
- Modify: `work-slot/slot_manager.py`
- Modify: `scripts/reconcile_slots.py` (update imports)

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_core._resolve_slots_dir`, `slot_metadata.parse_slot_md`, `slot_metadata._read_promotion_stamp`
- Produces: `_claude_project_matches(proj_name: str, slot_path_encoded: str) -> bool`, `relocate_claude_projects(slot_dir: Path, dest_dir: Path) -> int`, `sweep_orphaned_claude_projects(family_root: Path) -> int`, `remove_claude_projects(slot_dir: Path) -> int`

Note: `sweep_orphaned_claude_projects` contains an inline worklog recording block. Keep it inline for now — the worklog wrapper extraction happens in Batch 9.

- [ ] **Step 1: Create `work-slot/slot_claude.py`**

Copy the 4 Claude project functions. Add imports:
```python
import os
import sys
from pathlib import Path
from slot_core import run_cmd, _resolve_slots_dir, SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME
from slot_metadata import parse_slot_md, _read_promotion_stamp
```

Also copy the `_wl` (worklog) import block since `sweep_orphaned_claude_projects` uses it:
```python
_lib = Path.home() / ".claude" / "lib"
if _lib.exists():
    sys.path.insert(0, str(_lib))
try:
    import worklog as _wl
except ImportError:
    _wl = None
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions. Add:
```python
from slot_claude import (
    _claude_project_matches, relocate_claude_projects,
    sweep_orphaned_claude_projects, remove_claude_projects,
)
```

- [ ] **Step 3: Update external consumers**

`scripts/reconcile_slots.py`: Change `from slot_manager import relocate_claude_projects, remove_claude_projects` → `from slot_claude import relocate_claude_projects, remove_claude_projects`

- [ ] **Step 4: Move tests to `tests/test_slot_claude.py`**

- [ ] **Step 5: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py -v`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_claude.py tests/test_slot_claude.py work-slot/slot_manager.py scripts/reconcile_slots.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_claude.py from slot_manager Refs #306"
```

---

## Batch 6: Git infrastructure — slot_git.py

### Task 6: Extract slot_git.py

Extract git clone infrastructure. The `_detect_topology` optional import moves here.

**Files:**
- Create: `work-slot/slot_git.py`
- Create: `tests/test_slot_git.py`
- Modify: `work-slot/slot_manager.py` (remove functions and `_detect_topology` import)

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_core.is_worktree`, `slot_core.resolve_original_repo`, `slot_core._REGENERABLE_DIRS`, `slot_core._IDE_ARTIFACTS`, `slot_core.get_all_slot_repos`
- Produces: `configure_slot_remotes(clone_path: Path, original_path: Path) -> dict[str, str]`, `configure_update_instead(original_path: Path) -> None`, `install_post_commit_hook(clone_path: Path) -> None`, `sync_main(repo_path: str) -> None`, `_symlink_gitignored_assets(source_repo: Path, clone_dest: Path) -> list[str]`, `_exclude_symlinks(clone_path: Path) -> None`, `_repack_broken_alternates(slot_dir: Path, family_root: Path) -> int`, `_migrate_worktree_to_clone(worktree_path: Path) -> bool`, `ensure_clone_layout(slot_dir: Path) -> int`

- [ ] **Step 1: Create `work-slot/slot_git.py`**

Copy the 9 git functions. Move the `_detect_topology` import block here:
```python
import os
import shutil
import subprocess
import sys
import tempfile
from pathlib import Path
from slot_core import (
    run_cmd, is_worktree, resolve_original_repo,
    _REGENERABLE_DIRS, _IDE_ARTIFACTS, get_all_slot_repos,
)

_work_end = Path(__file__).parent.parent / "work-end"
if _work_end.exists():
    sys.path.insert(0, str(_work_end))
try:
    from common import detect_topology as _detect_topology
except ImportError:
    _detect_topology = None
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions and the `_detect_topology` import block. Add:
```python
from slot_git import (
    configure_slot_remotes, configure_update_instead,
    install_post_commit_hook, sync_main,
    _symlink_gitignored_assets, _exclude_symlinks,
    _repack_broken_alternates, _migrate_worktree_to_clone, ensure_clone_layout,
)
```

- [ ] **Step 3: Move tests to `tests/test_slot_git.py`**

- [ ] **Step 4: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py tests/test_slot_git.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_git.py tests/test_slot_git.py work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_git.py from slot_manager Refs #306"
```

---

## Batch 7: Workspace wiring — slot_workspace.py

### Task 7: Extract slot_workspace.py

Extract workspace discovery, symlink management, and CLAUDE.md replication.

**Files:**
- Create: `work-slot/slot_workspace.py`
- Create: `tests/test_slot_workspace.py`
- Modify: `work-slot/slot_manager.py`
- Modify: `scripts/migrate_slot_workspace.py` (update imports)
- Modify: `scripts/fix_active_slots.py` (update imports)

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_core.is_workspace_clone`
- Produces: `resolve_workspace_source(repo_path: Path) -> tuple[Path, str] | None`, `discover_workspace(repo_path: Path) -> tuple[Path, str] | None`, `repoint_wksp(repo_worktree: Path, ws_subdir: Path) -> None`, `create_proj_symlink(ws_subdir: Path, repo_worktree: Path) -> None`, `_unignore_subdir(ws_clone: Path, subdir_name: str) -> None`, `replicate_claude_md(repo_path: Path, ws_subdir: Path, repo_worktree: Path) -> None`, `validate_slot_wksp(slot_dir: Path, repo_names: list[str] | None = None) -> list[str]`

- [ ] **Step 1: Create `work-slot/slot_workspace.py`**

Copy the 7 workspace functions. Add imports:
```python
import os
import shutil
from pathlib import Path
from slot_core import run_cmd, is_workspace_clone
```

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions. Add:
```python
from slot_workspace import (
    resolve_workspace_source, discover_workspace,
    repoint_wksp, create_proj_symlink, _unignore_subdir,
    replicate_claude_md, validate_slot_wksp,
)
```

- [ ] **Step 3: Update external consumers**

`scripts/migrate_slot_workspace.py`: Update the import block — move `resolve_workspace_source`, `repoint_wksp`, `create_proj_symlink` to `from slot_workspace import ...`. The remaining `get_slot_repos` and `is_workspace_clone` already moved to `slot_core` in Batch 1.

`scripts/fix_active_slots.py`: Change `from slot_manager import _unignore_subdir` → `from slot_workspace import _unignore_subdir`

- [ ] **Step 4: Move tests to `tests/test_slot_workspace.py`**

- [ ] **Step 5: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py tests/test_slot_git.py tests/test_slot_workspace.py -v`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_workspace.py tests/test_slot_workspace.py work-slot/slot_manager.py scripts/migrate_slot_workspace.py scripts/fix_active_slots.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_workspace.py from slot_manager Refs #306"
```

---

## Batch 8: Queries — slot_query.py

### Task 8: Extract slot_query.py

Extract read-only query functions.

**Files:**
- Create: `work-slot/slot_query.py`
- Create: `tests/test_slot_query.py`
- Modify: `work-slot/slot_manager.py`

**Interfaces:**
- Consumes: `slot_core._resolve_slots_dir`, `slot_core._resolve_slot_dir_for_number`, `slot_core.run_cmd`, `slot_core.get_slot_repos`, `slot_core.get_all_slot_repos`, `slot_metadata.parse_slot_md`, `slot_workspace.validate_slot_wksp`
- Produces: `list_slots(family_root: Path, include_archived: bool = False) -> list[dict]`, `scan_ready(family_root: Path) -> list[dict]`, `check_cross_deps(family_root: Path, slot_num: int) -> int`, `find_slot_by_branch(family_root: Path, branch: str) -> tuple[int, bool] | None`, `get_repo_stats(repo_path: Path) -> dict`

- [ ] **Step 1: Create `work-slot/slot_query.py`**

Copy the 5 query functions. Add imports:
```python
import os
import sys
from pathlib import Path
from slot_core import (
    _resolve_slots_dir, _resolve_slot_dir_for_number,
    run_cmd, get_slot_repos, get_all_slot_repos,
    SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME,
)
from slot_metadata import parse_slot_md
from slot_workspace import validate_slot_wksp
```

Note: `list_slots` has an inline worklog block for drift checking and `_check_drift`/`_map_db_to_disk_state` calls. Keep `_check_drift` and `_map_db_to_disk_state` inline in `slot_query.py` for now rather than importing from `slot_observability.py` (which doesn't exist yet). They'll be moved to `slot_observability.py` in Batch 9.

Also copy the `_wl` import block since `list_slots` uses it for drift checking.

- [ ] **Step 2: Update `slot_manager.py`**

Remove extracted functions (including `_check_drift`, `_map_db_to_disk_state`). Add:
```python
from slot_query import (
    list_slots, scan_ready, check_cross_deps,
    find_slot_by_branch, get_repo_stats,
)
```

- [ ] **Step 3: Move tests to `tests/test_slot_query.py`**

- [ ] **Step 4: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py tests/test_slot_git.py tests/test_slot_workspace.py tests/test_slot_query.py -v`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_query.py tests/test_slot_query.py work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_query.py from slot_manager Refs #306"
```

---

## Batch 9: Observability — slot_observability.py

### Task 9: Extract slot_observability.py

Extract the worklog recording wrapper and drift detection. Create a `record_worklog()` helper to replace the 7 inline try/connect/record/close blocks.

**Files:**
- Create: `work-slot/slot_observability.py`
- Create: `tests/test_slot_observability.py`
- Modify: `work-slot/slot_manager.py` (remove drift functions, replace inline worklog blocks)
- Modify: `work-slot/slot_query.py` (move `_check_drift`, `_map_db_to_disk_state` here; replace inline worklog block with `record_worklog`)
- Modify: `work-slot/slot_claude.py` (replace inline worklog block with `record_worklog`)

**Interfaces:**
- Consumes: `slot_core.run_cmd`, `slot_metadata.parse_slot_md`
- Produces: `record_worklog(event: str, **kwargs) -> None`, `_check_drift(family_root: Path, slots: list[dict], ...) -> None`, `_map_db_to_disk_state(db_state: str) -> str`

- [ ] **Step 1: Create `work-slot/slot_observability.py`**

Create the module with the worklog wrapper and drift functions:

```python
import sys
from pathlib import Path

_lib = Path.home() / ".claude" / "lib"
if _lib.exists():
    sys.path.insert(0, str(_lib))
try:
    import worklog as _wl
except ImportError:
    _wl = None


def record_worklog(event: str, **kwargs) -> None:
    """Record a worklog event. Silently no-ops if worklog unavailable."""
    if _wl is None:
        return
    try:
        conn = _wl.connect()
        _wl.record(conn, event, **kwargs)
        conn.close()
    except Exception:
        pass
```

Move `_check_drift` and `_map_db_to_disk_state` from `slot_query.py`.

- [ ] **Step 2: Update `slot_query.py`**

Remove `_check_drift`, `_map_db_to_disk_state`, and the `_wl` import block. Add:
```python
from slot_observability import _check_drift, _map_db_to_disk_state
```

- [ ] **Step 3: Update `slot_claude.py`**

Remove the `_wl` import block. Replace inline worklog try/connect/record/close blocks with:
```python
from slot_observability import record_worklog
```

- [ ] **Step 4: Update `slot_manager.py`**

Replace remaining inline worklog try/connect/record/close blocks with calls to `record_worklog()`. Remove the `_wl` import block from `slot_manager.py`.

- [ ] **Step 5: Move tests to `tests/test_slot_observability.py`**

- [ ] **Step 6: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py tests/test_slot_git.py tests/test_slot_workspace.py tests/test_slot_query.py tests/test_slot_observability.py -v`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add work-slot/slot_observability.py tests/test_slot_observability.py work-slot/slot_manager.py work-slot/slot_query.py work-slot/slot_claude.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_observability.py, centralize worklog recording Refs #306"
```

---

## Batch 10: Orchestrators — slot_lifecycle.py

### Task 10: Extract slot_lifecycle.py and deduplicate create_slot/add_repo

Extract all orchestrator functions. Factor the shared 12-step repo-setup pipeline into `_setup_repo_in_slot()`.

**Files:**
- Create: `work-slot/slot_lifecycle.py`
- Create: `tests/test_slot_lifecycle.py`
- Modify: `work-slot/slot_manager.py` (becomes near-empty)

**Interfaces:**
- Consumes: All leaf modules
- Produces: `create_slot(...)`, `add_repo(...)`, `remove_repo(...)`, `_update_slot_repos(...)`, `merge_slot(...)`, `archive_slot(...)`, `remove_slot(...)`, `restore_slot(...)`, `_build_epic_plan(...)`, `allocate_slot_number(...)`, `migrate_remotes(...)`, `_setup_repo_in_slot(...)`

- [ ] **Step 1: Create `work-slot/slot_lifecycle.py`**

Copy all 11 orchestrator functions. Add imports from all leaf modules:

```python
import json
import os
import shutil
import subprocess
import sys
from pathlib import Path

from slot_core import (
    SlotCreationError, run_cmd,
    _resolve_slots_dir, _resolve_slot_dir_for_number,
    _get_family_repo_names,
    get_slot_repos, get_all_slot_repos,
    _cleanup_remnant_dir, _escape_slot_cwd, _has_unmerged_content,
    SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME,
    is_project_repo, resolve_original_repo,
)
from slot_metadata import (
    parse_slot_md, write_slot_md, _read_promotion_stamp,
    is_slot_landed, verify_landed_shas, _fix_stale_checkboxes,
)
from slot_maven import _write_slot_settings, setup_slot_repo
from slot_isx import (
    _check_isx_available, _truncate_instance_name,
    _teardown_isx, _wire_isx_remotes,
)
from slot_claude import (
    relocate_claude_projects, sweep_orphaned_claude_projects,
    remove_claude_projects,
)
from slot_git import (
    configure_slot_remotes, configure_update_instead,
    install_post_commit_hook, sync_main,
    _symlink_gitignored_assets, _exclude_symlinks,
    _repack_broken_alternates, ensure_clone_layout,
)
from slot_workspace import (
    resolve_workspace_source, discover_workspace,
    repoint_wksp, create_proj_symlink, _unignore_subdir,
    replicate_claude_md,
)
from slot_observability import record_worklog
from slot_query import find_slot_by_branch
```

- [ ] **Step 2: Extract `_setup_repo_in_slot()` from create_slot/add_repo**

Identify the shared 12-step sequence in both `create_slot` and `add_repo`:
1. `sync_main`
2. `git clone --shared`
3. `git checkout -b branch`
4. `_exclude_symlinks`
5. `_symlink_gitignored_assets`
6. `configure_slot_remotes`
7. `configure_update_instead`
8. `install_post_commit_hook`
9. `setup_slot_repo`
10. workspace discovery (`resolve_workspace_source` / `discover_workspace`)
11. workspace cloning
12. workspace wiring (`repoint_wksp`, `create_proj_symlink`, `replicate_claude_md`)

Factor into:
```python
def _setup_repo_in_slot(
    slot_dir: Path,
    repo_name: str,
    repo_path: Path,
    branch: str,
    m2_path: Path,
    family_root: Path,
) -> dict:
    """Set up a single repo clone in a slot. Returns workspace info dict."""
    # ... shared pipeline
```

Update `create_slot` and `add_repo` to call `_setup_repo_in_slot` for each repo.

- [ ] **Step 3: Update `slot_manager.py`**

Remove all orchestrator functions. `slot_manager.py` should now contain only:
- The CLI dispatch (`main`, `parse_args`) — these move in the next batch
- Import re-exports from all modules

- [ ] **Step 4: Move tests to `tests/test_slot_lifecycle.py`**

Move all tests for orchestrator functions. These are the largest test group.

- [ ] **Step 5: Run test suite**

Run: `python3 -m pytest tests/test_slot_manager.py tests/test_slot_core.py tests/test_slot_metadata.py tests/test_slot_maven.py tests/test_slot_isx.py tests/test_slot_claude.py tests/test_slot_git.py tests/test_slot_workspace.py tests/test_slot_query.py tests/test_slot_observability.py tests/test_slot_lifecycle.py -v`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_lifecycle.py tests/test_slot_lifecycle.py work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_lifecycle.py, deduplicate create_slot/add_repo Refs #306"
```

---

## Batch 11: CLI dispatch — slot_cli.py and shim

### Task 11: Extract slot_cli.py and convert slot_manager.py to shim

Final extraction. Move CLI dispatch to `slot_cli.py`. Convert `slot_manager.py` into a 3-line entry point shim.

**Files:**
- Create: `work-slot/slot_cli.py`
- Create: `tests/test_slot_cli.py`
- Modify: `work-slot/slot_manager.py` (becomes 3-line shim)

**Interfaces:**
- Consumes: `slot_lifecycle.*`, `slot_query.*`, `slot_isx.sync_isx`, `slot_git.ensure_clone_layout`, `slot_claude.sweep_orphaned_claude_projects`, `slot_core._resolve_slot_dir_for_number`
- Produces: `parse_args() -> dict[str, str]`, `main() -> None`

- [ ] **Step 1: Create `work-slot/slot_cli.py`**

Copy `parse_args` and `main`. Add imports:

```python
import json
import sys
from pathlib import Path

from slot_core import SlotCreationError, _resolve_slot_dir_for_number
from slot_lifecycle import (
    create_slot, remove_slot, merge_slot,
    archive_slot, restore_slot, add_repo, migrate_remotes,
)
from slot_query import list_slots, scan_ready, check_cross_deps
from slot_isx import sync_isx
from slot_git import ensure_clone_layout
from slot_claude import sweep_orphaned_claude_projects
```

- [ ] **Step 2: Convert `slot_manager.py` to shim**

Replace the entire contents of `slot_manager.py` with:

```python
#!/usr/bin/env python3
"""Slot manager — see slot_cli.py for implementation."""
from slot_cli import main

if __name__ == "__main__":
    main()
```

This preserves the `python3 work-slot/slot_manager.py` entry point for subprocess callers.

- [ ] **Step 3: Move tests to `tests/test_slot_cli.py`**

Move tests for `parse_args` and `main` dispatch. Rename `tests/test_slot_manager.py` to `tests/test_slot_lifecycle.py` if any lifecycle tests remain, otherwise delete it.

- [ ] **Step 4: Run full test suite**

Run: `python3 -m pytest tests/ -v -k slot`
Expected: All 298+ slot tests pass across 11 test files

- [ ] **Step 5: CLI smoke test**

Verify the shim works:
```bash
python3 work-slot/slot_manager.py --help 2>&1 | head -5
```
Expected: Usage/help output unchanged

- [ ] **Step 6: Commit**

```bash
git add work-slot/slot_cli.py work-slot/slot_manager.py tests/test_slot_cli.py tests/test_slot_manager.py
git commit -m "refactor: extract slot_cli.py, slot_manager.py becomes entry-point shim Refs #306"
```

---

## Batch 12: Verification and cleanup

### Task 12: Final verification and cleanup

Verify the complete decomposition is correct, clean, and documented.

**Files:**
- Verify: all 11 new modules in `work-slot/`
- Verify: all 11 new test files in `tests/`
- Verify: `work-slot/slot_manager.py` is a 3-line shim

- [ ] **Step 1: Run full test suite**

Run: `python3 -m pytest tests/ -v`
Expected: All tests pass (298+ slot tests + all other project tests)

- [ ] **Step 2: Verify module line counts**

```bash
wc -l work-slot/slot_*.py
```
Expected: No single module exceeds ~200 lines (except `slot_lifecycle.py` at ~450). Total should be close to the original 2,291 lines.

- [ ] **Step 3: Verify no stale imports in slot_manager.py**

The shim should have no `import os`, `import shutil`, etc. — just `from slot_cli import main`.

- [ ] **Step 4: Verify CLI contract**

Compare output format of `list-slots` and `create-slot --help` against pre-decomposition behavior. KEY=VALUE format unchanged.

- [ ] **Step 5: Commit cleanup if needed**

Only if verification found issues. Otherwise, decomposition is complete.

```bash
git commit -m "refactor: slot_manager decomposition complete — 11 modules Closes #306"
```

---

## References

- [2026-08-28-slot-manager-decomposition-design.md] — design spec
- [decisions.md] — D1 (module boundaries), D2 (extraction order), D3 (test organization), D4 (dedup strategy)
- [work-slot/slot_manager.py] — source file (2,291 lines, 67 functions)
- [tests/test_slot_manager.py] — test file (4,315 lines, 298+ tests)
- Protocol PP-20260609-df21ed — externalised scripts require tests
- Protocol PP-20260801-a1b2c3 — archive requires promotion verification
- [GitHub #306] — focal issue with full audit

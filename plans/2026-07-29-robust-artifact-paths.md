# Robust Artifact Path Resolution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #112 — fix: close_artifacts.py spec scanning broken — wrong paths for both workspace and slot layouts
**Issue group:** #112

**Goal:** Replace fragile inline path logic in close_artifacts.py with a tested, central artifact resolver that works for both original workspaces and worktree slots.

**Architecture:** Extract a single `workspace_artifacts.py` module that owns all artifact path knowledge. It takes a workspace root `Path` and returns a dict of category → list of relative paths. No branch parameter (artifacts aren't organized by branch), no repo-name parameter (the `wksp` symlink already resolves to the right root). Bug 2 from #112 (per-repo spec paths) is a non-issue: `workspace-init` creates per-repo child workspaces, so the `wksp` symlink always resolves to the per-repo root where specs are flat at `specs/`. Everything else — `close_artifacts.py`, `artifact_promote.py` — calls through it. Tests create temp directory trees shaped like real workspaces, never reference absolute machine paths.

**Scope:** This spec addresses #112 §3 category 1 (scan finds files in correct location) and removes the broken cleanup-specs step. Categories 2–6 (promotion destinations, routing, blog publishing, ADR INDEX.md updates, plan archival) remain in #112. All commit messages use `Refs #112`, not `Closes #112`.

**Tech Stack:** Python 3, pytest, pathlib, tempfile

## Global Constraints

- No hardcoded absolute paths — all resolution relative to a workspace root `Path`
- Artifact categories: `specs`, `adr`, `blog`, `plans`, `snapshots`
- INDEX.md is never a promotable artifact (excluded from all scans)
- `plans/attic/` is not scanned (archived plans)
- Specs persist in workspace after promotion — no cleanup step
- Return values are paths relative to workspace root (strings), matching the contract `artifact_promote.py` expects for `git checkout <branch> -- <path>`

---

### Task 1: Create `workspace_artifacts.py` — the central resolver

**Files:**
- Create: `work-end/workspace_artifacts.py`
- Test: `tests/test_workspace_artifacts.py`

**Interfaces:**
- Consumes: nothing (leaf module)
- Produces: `scan(workspace: Path) -> dict[str, list[str]]` — returns `{"specs": ["specs/design.md", ...], "adr": [...], ...}` with paths relative to workspace root

- [ ] **Step 1: Write failing tests for all fixture layouts**

The test file creates 4 fixture builders (functions that populate a `tmp_path`), then tests `scan()` against each. These fixtures represent real workspace shapes observed on disk.

```python
"""Tests for work-end/workspace_artifacts.py"""

import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))

from workspace_artifacts import scan


# ── Fixture builders ──────────────────────────────────────────────
# Each creates a realistic directory tree in tmp_path and returns it.
# No absolute paths — everything relative to the tmp root.

def build_single_repo_workspace(root: Path) -> Path:
    """Flat workspace (e.g. cc-praxis for soredium).

    Layout:
        root/
          specs/        ← flat, files directly here
            2026-06-17-design.md
            2026-07-28-epic-design.md
          adr/
            0001-doc-completeness.md
            INDEX.md     ← excluded from scan
          blog/
            2026-07-29-entry.md
            INDEX.md
          plans/
            2026-07-29-plan.md
            INDEX.md
            attic/       ← excluded from scan
              issue-87/
                archived-plan.md
          snapshots/
            2026-07-01-snapshot.md
            INDEX.md
          design/
            .meta
            JOURNAL.md
    """
    (root / "specs").mkdir()
    (root / "specs" / "2026-06-17-design.md").write_text("# Design\n")
    (root / "specs" / "2026-07-28-epic-design.md").write_text("# Epic\n")

    (root / "adr").mkdir()
    (root / "adr" / "0001-doc-completeness.md").write_text("# ADR\n")
    (root / "adr" / "INDEX.md").write_text("# Index\n")

    (root / "blog").mkdir()
    (root / "blog" / "2026-07-29-entry.md").write_text("# Blog\n")
    (root / "blog" / "INDEX.md").write_text("# Index\n")

    (root / "plans").mkdir()
    (root / "plans" / "2026-07-29-plan.md").write_text("# Plan\n")
    (root / "plans" / "INDEX.md").write_text("# Index\n")
    (root / "plans" / "attic" / "issue-87").mkdir(parents=True)
    (root / "plans" / "attic" / "issue-87" / "archived-plan.md").write_text("old")

    (root / "snapshots").mkdir()
    (root / "snapshots" / "2026-07-01-snapshot.md").write_text("# Snap\n")
    (root / "snapshots" / "INDEX.md").write_text("# Index\n")

    (root / "design").mkdir()
    (root / "design" / ".meta").write_text("issue: 112\n")
    (root / "design" / "JOURNAL.md").write_text("# Journal\n")

    return root


def build_multi_repo_workspace_subdir(root: Path) -> Path:
    """Per-repo subdirectory of a multi-repo workspace (e.g. casehub/blocks).

    The wksp symlink resolves to this subdirectory, so scan() sees it
    as a flat workspace — same shape as single-repo from scan()'s POV.

    Layout:
        root/           ← this IS the per-repo subdir (e.g. casehub/blocks/)
          specs/
            2026-05-14-spi-design.md
          adr/
            0001-decision.md
            INDEX.md
          blog/
            2026-05-10-entry.md
          plans/
            2026-05-09-plan.md
          snapshots/
            2026-05-12-snapshot.md
          design/
            .meta
    """
    (root / "specs").mkdir()
    (root / "specs" / "2026-05-14-spi-design.md").write_text("# Spec\n")

    (root / "adr").mkdir()
    (root / "adr" / "0001-decision.md").write_text("# ADR\n")
    (root / "adr" / "INDEX.md").write_text("# Index\n")

    (root / "blog").mkdir()
    (root / "blog" / "2026-05-10-entry.md").write_text("# Blog\n")

    (root / "plans").mkdir()
    (root / "plans" / "2026-05-09-plan.md").write_text("# Plan\n")

    (root / "snapshots").mkdir()
    (root / "snapshots" / "2026-05-12-snapshot.md").write_text("# Snap\n")

    (root / "design").mkdir()
    (root / "design" / ".meta").write_text("issue: 50\n")

    return root


def build_slot_workspace(root: Path) -> Path:
    """Workspace worktree inside a slot (e.g. worktrees/1/cc-praxis).

    Same internal structure — scan() doesn't care that the root is
    inside a worktrees/ directory. The structure is identical to
    single-repo because wksp was repointed by slot_manager.

    Layout:
        root/           ← the workspace worktree root
          specs/
            2026-07-20-slot-spec.md
          adr/
            0002-slot-decision.md
          blog/
            2026-07-20-slot-entry.md
          plans/
            (empty — no plans in this slot session)
          snapshots/
            (none)
          design/
            .meta
    """
    (root / "specs").mkdir()
    (root / "specs" / "2026-07-20-slot-spec.md").write_text("# Slot Spec\n")

    (root / "adr").mkdir()
    (root / "adr" / "0002-slot-decision.md").write_text("# ADR\n")

    (root / "blog").mkdir()
    (root / "blog" / "2026-07-20-slot-entry.md").write_text("# Blog\n")

    (root / "plans").mkdir()

    (root / "design").mkdir()
    (root / "design" / ".meta").write_text("issue: 99\n")

    return root


def build_empty_workspace(root: Path) -> Path:
    """Workspace with no artifact directories at all."""
    return root


# ── Tests ─────────────────────────────────────────────────────────

class TestScanSingleRepo:
    def test_finds_all_artifact_types(self, tmp_path):
        ws = build_single_repo_workspace(tmp_path)
        result = scan(ws)

        assert result["specs"] == [
            "specs/2026-06-17-design.md",
            "specs/2026-07-28-epic-design.md",
        ]
        assert result["adr"] == ["adr/0001-doc-completeness.md"]
        assert result["blog"] == ["blog/2026-07-29-entry.md"]
        assert result["plans"] == ["plans/2026-07-29-plan.md"]
        assert result["snapshots"] == ["snapshots/2026-07-01-snapshot.md"]

    def test_excludes_index_md_from_all_categories(self, tmp_path):
        ws = build_single_repo_workspace(tmp_path)
        result = scan(ws)
        for category, paths in result.items():
            assert not any("INDEX.md" in p for p in paths), \
                f"INDEX.md found in {category}: {paths}"

    def test_excludes_plans_attic(self, tmp_path):
        ws = build_single_repo_workspace(tmp_path)
        result = scan(ws)
        assert not any("attic" in p for p in result["plans"]), \
            f"attic content found in plans: {result['plans']}"


class TestScanMultiRepoSubdir:
    def test_finds_artifacts_in_per_repo_subdir(self, tmp_path):
        ws = build_multi_repo_workspace_subdir(tmp_path)
        result = scan(ws)

        assert result["specs"] == ["specs/2026-05-14-spi-design.md"]
        assert result["adr"] == ["adr/0001-decision.md"]
        assert result["blog"] == ["blog/2026-05-10-entry.md"]
        assert result["plans"] == ["plans/2026-05-09-plan.md"]
        assert result["snapshots"] == ["snapshots/2026-05-12-snapshot.md"]


class TestScanSlotWorkspace:
    def test_finds_artifacts_in_worktree(self, tmp_path):
        ws = build_slot_workspace(tmp_path)
        result = scan(ws)

        assert result["specs"] == ["specs/2026-07-20-slot-spec.md"]
        assert result["adr"] == ["adr/0002-slot-decision.md"]
        assert result["blog"] == ["blog/2026-07-20-slot-entry.md"]
        assert result["plans"] == []
        assert result["snapshots"] == []

    def test_worktree_nested_in_deep_path(self, tmp_path):
        """Scan works regardless of how deep the workspace root is."""
        deep = tmp_path / "family" / "worktrees" / "1" / "cc-praxis"
        deep.mkdir(parents=True)
        ws = build_slot_workspace(deep)
        result = scan(ws)
        assert result["specs"] == ["specs/2026-07-20-slot-spec.md"]


class TestScanEmptyWorkspace:
    def test_returns_empty_lists(self, tmp_path):
        ws = build_empty_workspace(tmp_path)
        result = scan(ws)
        assert result == {
            "specs": [],
            "adr": [],
            "blog": [],
            "plans": [],
            "snapshots": [],
        }


class TestScanEdgeCases:
    def test_non_md_files_in_specs_ignored(self, tmp_path):
        (tmp_path / "specs").mkdir()
        (tmp_path / "specs" / "design.md").write_text("spec")
        (tmp_path / "specs" / "diagram.png").write_bytes(b"\x89PNG")
        (tmp_path / "specs" / ".DS_Store").write_bytes(b"x")

        result = scan(tmp_path)
        assert result["specs"] == ["specs/design.md"]

    def test_non_md_snapshots_included(self, tmp_path):
        """Snapshots can be any file type (diagrams, exports)."""
        (tmp_path / "snapshots").mkdir()
        (tmp_path / "snapshots" / "arch.md").write_text("snap")
        (tmp_path / "snapshots" / "diagram.png").write_bytes(b"\x89PNG")

        result = scan(tmp_path)
        assert len(result["snapshots"]) == 2

    def test_subdirectories_in_artifact_dirs_ignored(self, tmp_path):
        """Only top-level files are scanned, not nested subdirs."""
        (tmp_path / "specs").mkdir()
        (tmp_path / "specs" / "design.md").write_text("spec")
        (tmp_path / "specs" / "subfolder").mkdir()
        (tmp_path / "specs" / "subfolder" / "nested.md").write_text("nested")

        result = scan(tmp_path)
        assert result["specs"] == ["specs/design.md"]

    def test_design_dir_not_scanned_as_artifact(self, tmp_path):
        """design/ contains .meta and JOURNAL.md — not an artifact category."""
        (tmp_path / "design").mkdir()
        (tmp_path / "design" / ".meta").write_text("issue: 1")
        (tmp_path / "design" / "JOURNAL.md").write_text("journal")

        result = scan(tmp_path)
        assert "design" not in result

    def test_scan_returns_sorted_paths(self, tmp_path):
        """Paths within each category are sorted for determinism."""
        (tmp_path / "specs").mkdir()
        (tmp_path / "specs" / "z-spec.md").write_text("z")
        (tmp_path / "specs" / "a-spec.md").write_text("a")
        (tmp_path / "specs" / "m-spec.md").write_text("m")

        result = scan(tmp_path)
        assert result["specs"] == [
            "specs/a-spec.md",
            "specs/m-spec.md",
            "specs/z-spec.md",
        ]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`
Expected: ImportError — `workspace_artifacts` module doesn't exist yet

- [ ] **Step 3: Write minimal implementation**

```python
#!/usr/bin/env python3
"""
Central artifact path resolver for soredium workspaces.

Given a workspace root Path, returns all promotable artifacts grouped
by category. Works identically for original workspaces, per-repo
subdirectories of multi-repo workspaces, and worktree slots — the
wksp symlink already resolves to the correct root before this module
is called.

No branch parameter: artifacts are not organized by branch.
No repo-name parameter: wksp symlink handles per-repo resolution.
"""

from pathlib import Path

CATEGORIES: dict[str, dict] = {
    "specs":     {"ext": ".md",  "exclude_names": {"INDEX.md"}},
    "adr":       {"ext": ".md",  "exclude_names": {"INDEX.md"}},
    "blog":      {"ext": ".md",  "exclude_names": {"INDEX.md"}},
    "plans":     {"ext": ".md",  "exclude_names": {"INDEX.md"}},
    "snapshots": {"ext": None,   "exclude_names": {"INDEX.md"}},
}


def scan(workspace: Path) -> dict[str, list[str]]:
    """Scan workspace for promotable artifacts.

    Returns category -> sorted list of paths relative to workspace root.
    """
    found: dict[str, list[str]] = {}

    for category, cfg in CATEGORIES.items():
        cat_dir = workspace / category
        if not cat_dir.is_dir():
            found[category] = []
            continue

        ext = cfg["ext"]
        exclude_names = cfg["exclude_names"]

        entries = []
        for f in cat_dir.iterdir():
            if f.name in exclude_names:
                continue
            if f.is_dir():
                continue
            if ext is not None and f.suffix != ext:
                continue
            entries.append(str(f.relative_to(workspace)))

        found[category] = sorted(entries)

    return found
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/workspace_artifacts.py tests/test_workspace_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#112): add workspace_artifacts.py — central artifact path resolver

Extracts artifact scanning into a tested module with fixture-based
tests for single-repo, multi-repo, and slot workspace layouts.
Refs #112"
```

---

### Task 2: Rewire `close_artifacts.py` to use the new resolver

**Files:**
- Modify: `work-end/close_artifacts.py`
- Modify: `tests/test_close_artifacts.py`

**Interfaces:**
- Consumes: `workspace_artifacts.scan(workspace: Path) -> dict[str, list[str]]` from Task 1
- Produces: same CLI interface as before (KEY=VALUE output), minus `SPECS_CLEANED`

- [ ] **Step 1: Write failing test — scan uses workspace_artifacts.scan()**

Update `tests/test_close_artifacts.py` — replace the broken `TestScanArtifacts` class that tests `specs/<branch>/` layout with tests against the correct flat layout.

```python
class TestScanArtifacts:
    """scan_artifacts now delegates to workspace_artifacts.scan()."""

    def test_finds_specs_flat(self, tmp_path):
        """Specs are flat at workspace/specs/, not workspace/specs/<branch>/."""
        specs = tmp_path / "specs"
        specs.mkdir()
        (specs / "design.md").write_text("spec")
        (specs / "notes.md").write_text("notes")

        result = scan_artifacts(tmp_path)
        assert len(result["specs"]) == 2
        assert "specs/design.md" in result["specs"]

    def test_does_not_use_branch_subdirectory(self, tmp_path):
        """The old branch-based spec path is dead."""
        (tmp_path / "specs" / "issue-42-feat").mkdir(parents=True)
        (tmp_path / "specs" / "issue-42-feat" / "spec.md").write_text("x")
        (tmp_path / "specs").mkdir(exist_ok=True)
        (tmp_path / "specs" / "top-level.md").write_text("y")

        result = scan_artifacts(tmp_path)
        # Only the top-level file, not the nested one
        assert result["specs"] == ["specs/top-level.md"]

    def test_finds_all_artifact_types(self, tmp_path):
        for cat in ("specs", "adr", "blog", "plans", "snapshots"):
            d = tmp_path / cat
            d.mkdir()
            (d / f"test-{cat}.md").write_text(f"# {cat}\n")
        result = scan_artifacts(tmp_path)
        for cat in ("specs", "adr", "blog", "plans", "snapshots"):
            assert len(result[cat]) == 1, f"{cat} should have 1 entry"

    def test_empty_workspace(self, tmp_path):
        result = scan_artifacts(tmp_path)
        assert all(len(v) == 0 for v in result.values())
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestScanArtifacts -v`
Expected: FAIL — `scan_artifacts()` still takes `branch` parameter

- [ ] **Step 3: Update `scan_artifacts()` to delegate to `workspace_artifacts.scan()`**

Replace the entire function body. Remove `branch` parameter.

```python
from workspace_artifacts import scan as _scan_workspace

def scan_artifacts(workspace: Path) -> dict[str, list[str]]:
    """Scan workspace for promotable artifacts. Returns category -> list of relative paths."""
    return _scan_workspace(workspace)
```

Update `main()` to call `scan_artifacts(scan_source)` without `branch`:

```python
    artifacts = scan_artifacts(scan_source)
```

Remove the `specs_cleaned` tracking from `main()` — delete the entire cleanup-specs block (lines 232-239) and the `results["specs_cleaned"] = "0"` fallback.

Remove `specs_cleaned` from `write_stamp()`.

Replace the independent blog scan with a check against the already-scanned data:

```python
# Replace lines 266-267:
blog_dir = workspace / "blog"
blog_entries = [f for f in blog_dir.glob("*.md") if f.name != "INDEX.md"] if blog_dir.is_dir() else []
if blog_entries:
# With:
if artifacts["blog"]:
    blog_dir = scan_source / "blog"
```

This eliminates the duplicate blog scan and fixes a latent slot-mode bug where `blog_dir` used `workspace` (original, on main) instead of `scan_source` (slot workspace, with branch artifacts).

- [ ] **Step 4: Update the remaining tests in `test_close_artifacts.py`**

Fix `TestScanWorkspaceParameter` tests — they create `specs/<branch>/` directories which are now wrong. Update to flat layout:

```python
class TestScanWorkspaceParameter:
    # ...
    def test_scan_workspace_reads_from_alternate_path(self, tmp_path):
        workspace = tmp_path / "original-workspace"
        slot_workspace = tmp_path / "slot-workspace"
        project = tmp_path / "project"
        self._init_git(workspace)
        self._init_git(slot_workspace)
        self._init_git(project)
        (workspace / "design").mkdir()

        (slot_workspace / "specs").mkdir()
        (slot_workspace / "specs" / "spec.md").write_text("# Spec\n")

        result = subprocess.run(
            [sys.executable, str(self.SCRIPT),
             str(workspace), str(project), "any-branch",
             f"scan-workspace={slot_workspace}"],
            capture_output=True, text=True,
        )
        assert result.returncode != 1

    def test_scan_workspace_unit_scan_artifacts(self, tmp_path):
        slot = tmp_path / "slot"
        slot.mkdir()
        (slot / "specs").mkdir()
        (slot / "specs" / "design.md").write_text("spec")
        (slot / "blog").mkdir()
        (slot / "blog" / "entry.md").write_text("blog")

        result = scan_artifacts(slot)
        assert len(result["specs"]) == 1
        assert len(result["blog"]) == 1
```

Fix `TestWriteStamp` — remove `specs_cleaned` from the results dict.

- [ ] **Step 5: Run all close_artifacts tests**

Run: `python3 -m pytest tests/test_close_artifacts.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_artifacts.py tests/test_close_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix(#112): rewire close_artifacts to use workspace_artifacts resolver

Replaces broken spec scanning (specs/<branch>/ path that never existed)
with delegation to workspace_artifacts.scan(). Removes specs cleanup
step — specs persist in workspace as source of truth.
Refs #112"
```

---

### Task 3: Remove `cleanup_specs()` from `artifact_promote.py`

**Files:**
- Modify: `work-end/artifact_promote.py`

**Interfaces:**
- Consumes: nothing new
- Produces: same CLI interface minus the `cleanup-specs` subcommand

- [ ] **Step 1: Write failing test — cleanup-specs subcommand rejected**

```python
def test_cleanup_specs_subcommand_removed(self, tmp_path):
    """cleanup-specs is no longer a valid subcommand."""
    script = Path(__file__).parent.parent / "work-end" / "artifact_promote.py"
    result = subprocess.run(
        [sys.executable, str(script), "cleanup-specs", str(tmp_path), "branch=test"],
        capture_output=True, text=True,
    )
    assert result.returncode == 1
```

- [ ] **Step 2: Run to verify it fails (cleanup-specs still exists)**

Run: `python3 -m pytest tests/test_close_artifacts.py::test_cleanup_specs_subcommand_removed -v`
Expected: FAIL — cleanup-specs still returns 0

- [ ] **Step 3: Delete `cleanup_specs()` and remove from SUBCOMMANDS**

Remove the `cleanup_specs` function (lines 187-227) and its entry in the `SUBCOMMANDS` dict.

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/test_close_artifacts.py tests/test_workspace_artifacts.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/artifact_promote.py tests/test_close_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "refactor(#112): remove cleanup_specs — specs persist in workspace

Specs are source of truth in workspace. Promotion copies them to the
project repo. No deletion needed.
Refs #112"
```

---

### Task 4: Fix SKILL.md paths and remove residual SPECS_CLEANED references

**Files:**
- Modify: `work-end/SKILL.md` (fix scan paths, remove cleanup-specs references)
- Modify: `work-end/close_report.py` (remove dead specs_cleaned handling)

- [ ] **Step 1: Fix SKILL.md Step 4 — broken spec inventory path**

Replace the `specs/$BRANCH_NAME/` path with flat `specs/`:

```
# In Step 4 artifact inventory, replace:
ls "$WORKSPACE/specs/$BRANCH_NAME/" 2>/dev/null
# With:
ls "$WORKSPACE/specs/" 2>/dev/null | grep -v INDEX.md
```

- [ ] **Step 2: Fix SKILL.md Step 6 — broken spec selection path**

```
# Replace:
If tracking enabled: list `$WORKSPACE/specs/$BRANCH_NAME/`, ask which to post
# With:
If tracking enabled: list `$WORKSPACE/specs/`, ask which to post
```

- [ ] **Step 3: Remove SPECS_CLEANED from SKILL.md Step 8a**

Three specific removals in the Step 8a section:
1. Remove `- \`SPECS_CLEANED=N\` — spec files cleaned from workspace branch` from the "Read output KEY=value lines" list
2. Remove `specs_cleaned=$SPECS_CLEANED \` from the `close_report.py record` call
3. Remove `SPECS_CLEANED=N` from the output documentation block at the top of Step 8a

- [ ] **Step 4: Remove specs_cleaned handling from close_report.py**

In `_format_detail()` for the "artifacts" step, remove the dead `specs_cleaned` lines:

```python
# Remove these lines from the "artifacts" case:
sc = d.get("specs_cleaned", "0")
# ...
if int(sc) > 0:
    parts.append(f"{sc} specs cleaned")
```

Without this, passing `specs_cleaned=` (empty string from an unset shell variable) would cause `int("")` → `ValueError`.

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest tests/test_workspace_artifacts.py tests/test_close_artifacts.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/SKILL.md work-end/close_report.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix(#112): fix SKILL.md scan paths and remove SPECS_CLEANED references

Steps 4 and 6 used specs/<branch>/ which never existed. Step 8a
referenced SPECS_CLEANED which is no longer emitted. close_report.py
had dead specs_cleaned handling that would ValueError on empty input.
Refs #112"
```

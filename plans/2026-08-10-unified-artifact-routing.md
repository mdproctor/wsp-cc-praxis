# Unified Artifact Routing — Phase 1: Skill Changes

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before starting
**Issue group:** TBD

**Goal:** Unify artifact routing so ADRs, specs, and blogs all promote
to `docs/<type>/` in project repos, replace the blog-specific resolver
with a unified artifact resolver, and add blog to the docs-prefix
promotion path.

**Architecture:** Three changes: (1) replace `resolve_blog_dir.py` with
`resolve_artifact_dir.py` that handles all artifact types, (2) add blog
to `_docs_categories` in `close_artifacts.py` so it gets the `docs/`
prefix on project promotion, (3) update all consuming skills to use the
unified resolver.

**Tech Stack:** Python 3.14, pytest, soredium skill markdown

## Global Constraints

- All artifact types (adr, specs, blog, plans) live at root in
  workspaces and under `docs/` in project repos
- Workspace authoring paths: `$WORKSPACE/<type>/`
- Project promotion paths: `$PROJECT/docs/<type>/`
- Slot-escape detection must be preserved
- Backward compatibility with `**Blog directory:**` CLAUDE.md field
- No breaking changes to `routing.py` (3-layer cascade is correct)

## Scope

This plan covers **soredium skill and script changes only** (Phase 1).
Subsequent plans cover migration of existing entries, historical
backfill, CLAUDE.md routing table updates, and hygiene sweeps.

---

### Task 1: Unified Artifact Resolver

Replace `write-content/resolve_blog_dir.py` with
`write-content/resolve_artifact_dir.py` that resolves the authoring
directory for any artifact type.

**Files:**
- Create: `write-content/resolve_artifact_dir.py`
- Delete: `write-content/resolve_blog_dir.py` (after all references updated)
- Create: `tests/test_resolve_artifact_dir.py`
- Delete: `tests/test_resolve_blog_dir.py` (after new tests cover all cases)

**Interfaces:**
- Produces: `resolve(artifact_type, workspace, claude_text, slot_root=None) -> str`
- Produces: `resolve_with_warning(artifact_type, workspace, claude_text, slot_root=None) -> tuple[str, str]`
- Produces: CLI: `python3 resolve_artifact_dir.py <type> <workspace> <claude_md_path> [slot_root=<path>]`
  Output: `ARTIFACT_DIR=<path>`

- [ ] **Step 1: Write failing tests for the unified resolver**

Port all tests from `test_resolve_blog_dir.py` to
`test_resolve_artifact_dir.py`, parameterized across artifact types.
Add new tests for ADR, specs, and plans resolution.

```python
"""Tests for write-content/resolve_artifact_dir.py"""

import sys
from pathlib import Path

import pytest

skill_dir = Path(__file__).parent.parent / "write-content"
sys.path.insert(0, str(skill_dir))

import resolve_artifact_dir


ARTIFACT_TYPES = ["blog", "adr", "specs", "plans"]


class TestDefaultResolution:
    """Without CLAUDE.md overrides, all types default to $WORKSPACE/<type>/."""

    @pytest.mark.parametrize("artifact_type", ARTIFACT_TYPES)
    def test_defaults_to_workspace_subdir(self, tmp_path, artifact_type):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        result = resolve_artifact_dir.resolve(artifact_type, str(workspace), "")
        assert result == str(workspace / artifact_type)

    @pytest.mark.parametrize("artifact_type", ARTIFACT_TYPES)
    def test_empty_claude_text(self, tmp_path, artifact_type):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        result = resolve_artifact_dir.resolve(artifact_type, str(workspace), "")
        assert result == str(workspace / artifact_type)


class TestCustomDirectoryOverride:
    """CLAUDE.md **<Type> directory:** field overrides default."""

    def test_blog_directory_override(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        claude_text = '**Blog directory:** `custom-blog/`'
        result = resolve_artifact_dir.resolve("blog", str(workspace), claude_text)
        assert result == str(workspace / "custom-blog")

    def test_adr_directory_override(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        claude_text = '**ADR directory:** `custom-adr/`'
        result = resolve_artifact_dir.resolve("adr", str(workspace), claude_text)
        assert result == str(workspace / "custom-adr")

    def test_absolute_path_non_slot(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        blog_path = tmp_path / "external" / "blog"
        claude_text = f'**Blog directory:** `{blog_path}/`'
        result = resolve_artifact_dir.resolve("blog", str(workspace), claude_text)
        assert result == str(blog_path)

    def test_unrecognized_type_uses_default(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        result = resolve_artifact_dir.resolve("snapshots", str(workspace), "")
        assert result == str(workspace / "snapshots")


class TestSlotEscapeDetection:
    """Absolute paths escaping slot boundary fall back to default."""

    @pytest.mark.parametrize("artifact_type", ARTIFACT_TYPES)
    def test_absolute_path_in_slot_detected(self, tmp_path, artifact_type):
        slot_workspace = tmp_path / "slots" / "1" / "work"
        slot_workspace.mkdir(parents=True)
        field_name = _field_name(artifact_type)
        claude_text = f'**{field_name} directory:** `/external/path/`'
        result = resolve_artifact_dir.resolve(
            artifact_type, str(slot_workspace), claude_text,
            slot_root=str(tmp_path / "slots" / "1"),
        )
        assert result == str(slot_workspace / artifact_type)

    @pytest.mark.parametrize("artifact_type", ARTIFACT_TYPES)
    def test_relative_path_in_slot_ok(self, tmp_path, artifact_type):
        slot_workspace = tmp_path / "slots" / "1" / "work"
        slot_workspace.mkdir(parents=True)
        field_name = _field_name(artifact_type)
        claude_text = f'**{field_name} directory:** `{artifact_type}/`'
        result = resolve_artifact_dir.resolve(
            artifact_type, str(slot_workspace), claude_text,
            slot_root=str(tmp_path / "slots" / "1"),
        )
        assert result == str(slot_workspace / artifact_type)

    def test_escape_returns_warning(self, tmp_path):
        slot_workspace = tmp_path / "slots" / "1" / "work"
        slot_workspace.mkdir(parents=True)
        claude_text = '**Blog directory:** `/external/blog/`'
        result, warning = resolve_artifact_dir.resolve_with_warning(
            "blog", str(slot_workspace), claude_text,
            slot_root=str(tmp_path / "slots" / "1"),
        )
        assert result == str(slot_workspace / "blog")
        assert "escapes slot boundary" in warning


def _field_name(artifact_type: str) -> str:
    """Map artifact type to CLAUDE.md field name."""
    return {"adr": "ADR", "blog": "Blog", "specs": "Specs", "plans": "Plans"}.get(
        artifact_type, artifact_type.title()
    )
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_resolve_artifact_dir.py -v`
Expected: FAIL — `resolve_artifact_dir` module not found

- [ ] **Step 3: Implement resolve_artifact_dir.py**

```python
#!/usr/bin/env python3
"""
resolve_artifact_dir.py — Unified artifact directory resolution with slot-escape detection.

Resolves the authoring directory for any artifact type (blog, adr, specs, plans)
from CLAUDE.md content. When running inside a slot, detects absolute paths that
escape the slot boundary and falls back to $WORKSPACE/<type>/.

Replaces the blog-specific resolve_blog_dir.py.

Usage:
    python3 resolve_artifact_dir.py <type> <workspace> <claude_md_path> [slot_root=<path>]

Output:
    ARTIFACT_DIR=<resolved path>
    WARNING=<message if escape detected, empty otherwise>
"""

import os
import re
import sys
from pathlib import Path

FIELD_NAMES = {
    "adr": "ADR",
    "blog": "Blog",
    "specs": "Specs",
    "plans": "Plans",
}


def _parse_custom_dir(artifact_type: str, claude_text: str) -> str | None:
    field = FIELD_NAMES.get(artifact_type, artifact_type.title())
    m = re.search(rf"\*\*{field} directory:\*\*\s*`([^`]+)`", claude_text)
    if m:
        return m.group(1).rstrip("/")
    return None


def _resolve_path(raw: str, workspace: str) -> str:
    expanded = os.path.expanduser(raw)
    p = Path(expanded)
    if p.is_absolute():
        return str(p)
    return str(Path(workspace) / expanded)


def _is_inside(path: str, root: str) -> bool:
    try:
        Path(path).relative_to(root)
        return True
    except ValueError:
        return False


def resolve_with_warning(
    artifact_type: str,
    workspace: str,
    claude_text: str,
    slot_root: str | None = None,
) -> tuple[str, str]:
    default = str(Path(workspace) / artifact_type)
    raw = _parse_custom_dir(artifact_type, claude_text)

    if raw is None:
        return default, ""

    resolved = _resolve_path(raw, workspace)

    if slot_root is not None:
        if not _is_inside(resolved, slot_root):
            warning = (
                f"Artifact directory '{resolved}' escapes slot boundary '{slot_root}'. "
                f"Falling back to {default}."
            )
            return default, warning

    return resolved, ""


def resolve(
    artifact_type: str,
    workspace: str,
    claude_text: str,
    slot_root: str | None = None,
) -> str:
    path, _ = resolve_with_warning(artifact_type, workspace, claude_text, slot_root)
    return path


def main() -> int:
    if len(sys.argv) < 4:
        print("Usage: resolve_artifact_dir.py <type> <workspace> <claude_md_path> [slot_root=<path>]")
        return 1

    artifact_type = sys.argv[1]
    workspace = sys.argv[2]
    claude_md_path = sys.argv[3]

    kv = {}
    for arg in sys.argv[4:]:
        if "=" in arg:
            k, _, v = arg.partition("=")
            kv[k] = v

    slot_root = kv.get("slot_root")

    try:
        claude_text = Path(claude_md_path).read_text()
    except FileNotFoundError:
        claude_text = ""

    path, warning = resolve_with_warning(artifact_type, workspace, claude_text, slot_root)
    print(f"ARTIFACT_DIR={path}")
    if warning:
        print(f"WARNING={warning}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_resolve_artifact_dir.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add write-content/resolve_artifact_dir.py tests/test_resolve_artifact_dir.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: unified artifact directory resolver — replaces blog-specific resolve_blog_dir.py

Refs #TBD"
```

---

### Task 2: Add blog to docs-prefix promotion in close_artifacts.py

The core routing fix: blog entries promoted to project repos must land
at `docs/blog/`, not `blog/`.

**Files:**
- Modify: `work-end/close_artifacts.py:157` — add `"blog"` to `_docs_categories`
- Modify: `tests/test_close_artifacts.py` — add test for blog→docs/blog/ promotion

**Interfaces:**
- Consumes: existing `close_artifacts.py` API (no change)
- Produces: blog artifacts now promoted to `$PROJECT/docs/blog/` instead of `$PROJECT/blog/`

- [ ] **Step 1: Write failing test for blog docs-prefix promotion**

```python
class TestBlogDocsPrefix:
    """Blog entries must land at project/docs/blog/, not project/blog/."""

    SCRIPT = Path(__file__).parent.parent / "work-end" / "artifact_promote.py"

    def _init_git(self, path):
        path.mkdir(parents=True, exist_ok=True)
        subprocess.run(["git", "init", str(path)], capture_output=True)
        subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"], capture_output=True)
        subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"], capture_output=True)
        subprocess.run(["git", "-C", str(path), "commit", "--allow-empty", "-m", "init"], capture_output=True)

    def test_blog_promoted_to_docs_blog(self, tmp_path):
        """Blog promoted to project lands at docs/blog/, not blog/."""
        workspace = tmp_path / "workspace"
        project = tmp_path / "project"
        self._init_git(workspace)
        self._init_git(project)
        (workspace / "design").mkdir()
        (workspace / "blog").mkdir()
        (workspace / "blog" / "entry.md").write_text("# Blog\n")

        subprocess.run(
            ["git", "-C", str(workspace), "checkout", "-b", "issue-42-test"],
            capture_output=True, check=True,
        )
        subprocess.run(["git", "-C", str(workspace), "add", "-A"], capture_output=True)
        subprocess.run(
            ["git", "-C", str(workspace), "commit", "-m", "add blog"],
            capture_output=True, check=True,
        )

        # Default routing: blog → project
        result = subprocess.run(
            [sys.executable, str(Path(__file__).parent.parent / "work-end" / "close_artifacts.py"),
             str(workspace), str(project), "issue-42-test"],
            capture_output=True, text=True,
        )

        assert (project / "docs" / "blog" / "entry.md").is_file(), (
            f"Blog should be at docs/blog/, not blog/.\n"
            f"stdout: {result.stdout}\nstderr: {result.stderr}"
        )
        assert not (project / "blog" / "entry.md").exists(), (
            "Blog should NOT be at root blog/"
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestBlogDocsPrefix -v`
Expected: FAIL — blog lands at `blog/` not `docs/blog/`

- [ ] **Step 3: Fix close_artifacts.py**

Change line 157:
```python
# Before:
_docs_categories = {"specs", "adr"}

# After:
_docs_categories = {"specs", "adr", "blog"}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestBlogDocsPrefix -v`
Expected: PASS

- [ ] **Step 5: Run full close_artifacts test suite for regressions**

Run: `python3 -m pytest tests/test_close_artifacts.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_artifacts.py tests/test_close_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: promote blog to docs/blog/ in project repos, not root blog/

Blog was excluded from _docs_categories, causing it to land at
project root instead of docs/blog/ alongside specs and adr.

Refs #TBD"
```

---

### Task 3: Update diary form to use unified resolver

**Files:**
- Modify: `write-content/forms/diary.md:80-83` — replace `resolve_blog_dir.py` with `resolve_artifact_dir.py`

**Interfaces:**
- Consumes: `resolve_artifact_dir.resolve("blog", ...)` from Task 1

- [ ] **Step 1: Update diary.md Step 1**

Change lines 80-83 from:
```markdown
**Resolve blog directory:**

\`\`\`bash
python3 ~/.claude/skills/write-content/resolve_blog_dir.py <WORKSPACE> <CLAUDE_MD_PATH> [slot_root=<SLOT_ROOT>]
\`\`\`

Read `BLOG_DIR` from output. Resolve to an absolute path.
```

To:
```markdown
**Resolve blog directory:**

\`\`\`bash
python3 ~/.claude/skills/write-content/resolve_artifact_dir.py blog <WORKSPACE> <CLAUDE_MD_PATH> [slot_root=<SLOT_ROOT>]
\`\`\`

Read `ARTIFACT_DIR` from output. Resolve to an absolute path.
```

- [ ] **Step 2: Search for other references to resolve_blog_dir**

```bash
grep -rn "resolve_blog_dir" ~/claude/hortora/soredium/ --include="*.md" --include="*.py" | grep -v __pycache__ | grep -v test_resolve_blog
```

Update any remaining references.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add write-content/forms/diary.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "refactor: diary form uses unified resolve_artifact_dir.py

Refs #TBD"
```

---

### Task 4: Update ADR skill to use unified resolver

The ADR skill currently has its own routing cascade in Step 1 that
duplicates the logic in `routing.py` and `resolve_artifact_dir.py`.
Simplify to use the unified resolver.

**Files:**
- Modify: `adr/SKILL.md:28-54` — replace custom routing with unified resolver call

**Interfaces:**
- Consumes: `resolve_artifact_dir.py adr` from Task 1

- [ ] **Step 1: Update adr/SKILL.md Step 1**

Replace the three-layer routing cascade (lines 30-54) with:

```markdown
### Step 1 — Resolve write destination

Resolve the ADR directory using the unified artifact resolver:

\`\`\`bash
python3 ~/.claude/skills/project/ctx.py
\`\`\`

Use `WORKSPACE` and `PROJECT` from the output as concrete strings.

\`\`\`bash
python3 ~/.claude/skills/write-content/resolve_artifact_dir.py adr <WORKSPACE> <WORKSPACE>/CLAUDE.md [slot_root=<SLOT_ROOT>]
\`\`\`

Read `ARTIFACT_DIR` from output.

| Context | Write to | git -C path |
|---------|----------|-------------|
| Workspace | `$WORKSPACE/adr/` | `$WORKSPACE` |
| Project (default) | `$PROJECT/docs/adr/` | `$PROJECT` |

Use `git -C <resolved-path>` for all git operations — never bare `git add/commit`.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add adr/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "refactor: ADR skill uses unified resolve_artifact_dir.py

Removes duplicated 3-layer routing cascade in favour of the shared resolver.

Refs #TBD"
```

---

### Task 5: Delete resolve_blog_dir.py and its tests

Now that all consumers use `resolve_artifact_dir.py`, remove the
blog-specific resolver.

**Files:**
- Delete: `write-content/resolve_blog_dir.py`
- Delete: `tests/test_resolve_blog_dir.py`

**Interfaces:**
- Consumes: confirmation that no references remain (from Task 3 Step 2)

- [ ] **Step 1: Verify no remaining references**

```bash
grep -rn "resolve_blog_dir" ~/claude/hortora/soredium/ --include="*.md" --include="*.py" | grep -v __pycache__
```

Expected: no results (or only the files being deleted)

- [ ] **Step 2: Delete files**

```bash
git -C /Users/mdproctor/claude/hortora/soredium rm write-content/resolve_blog_dir.py tests/test_resolve_blog_dir.py
```

- [ ] **Step 3: Run full test suite to verify nothing breaks**

Run: `python3 -m pytest tests/ -v --timeout=30`
Expected: ALL PASS (no imports of resolve_blog_dir remain)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add -A
git -C /Users/mdproctor/claude/hortora/soredium commit -m "chore: remove resolve_blog_dir.py — replaced by resolve_artifact_dir.py

Refs #TBD"
```

---

### Task 6: Update workspace_artifacts.py to scan docs/blog/ as alt_path

The scanner already has `alt_paths` for ADR (`docs/adr`). Add the same
for blog so that blog entries authored directly in `docs/blog/` (which
can happen in project repos) are also discovered.

**Files:**
- Modify: `work-end/workspace_artifacts.py:23` — add `alt_paths` for blog
- Modify: `tests/test_workspace_artifacts.py` — add test

- [ ] **Step 1: Write failing test**

```python
def test_finds_blog_in_docs_blog(self, tmp_path):
    """Blog entries in docs/blog/ are discovered via alt_paths."""
    (tmp_path / "docs" / "blog").mkdir(parents=True)
    (tmp_path / "docs" / "blog" / "entry.md").write_text("# Blog\n")
    result = scan(tmp_path)
    assert "docs/blog/entry.md" in result["blog"]
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `docs/blog/entry.md` not in results

- [ ] **Step 3: Add alt_paths for blog**

```python
# Line 23 — before:
"blog":      {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set()},

# After:
"blog":      {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set(),
              "alt_paths": ["docs/blog"]},
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Run full workspace_artifacts test suite**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/workspace_artifacts.py tests/test_workspace_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: scan docs/blog/ as alt_path for blog artifacts

Matches existing alt_paths pattern for adr (docs/adr/).

Refs #TBD"
```

---

### Task 7: Update SOURCES.md and CLAUDE.md references

Update documentation to reflect the unified artifact layout.

**Files:**
- Modify: `SOURCES.md` — update Blog row
- Modify: `CLAUDE.md` — update Blog section and Project Artifacts table

- [ ] **Step 1: Update SOURCES.md**

The Blog row currently says `hortora.github.io/_posts/`. This is the
publish destination, not the project location. Update or add a row:

```markdown
| Blog | docs/blog/ | Project diary entries (promoted from workspace) |
```

- [ ] **Step 2: Update CLAUDE.md Project Artifacts table**

Add `docs/blog/` to the table:

```markdown
| `docs/blog/` | Project diary / blog entries |
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add SOURCES.md CLAUDE.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "docs: update SOURCES.md and CLAUDE.md for unified artifact layout

Refs #TBD"
```

---

## Subsequent Plans (Not This Session)

### Phase 2: Migration of existing entries
- Move root-level `blog/`, `adr/`, `specs/` to `docs/` in all canonical
  project repos
- Merge `docs/superpowers/{specs,plans}/` into `docs/{specs,plans}/`
- Delete `docs/superpowers/` directories
- One commit per project repo

### Phase 3: Historical backfill
- Copy entries from `public/casehub/<project>/blog/` into
  `casehub/<project>/docs/blog/` where missing
- Attribute 57 unattributed entries in `casehub/work/blog/` to primary
  projects with cross-project labels
- Deduplicate across project/workspace/public

### Phase 4: CLAUDE.md routing table updates
- Update routing tables in all casehub project CLAUDE.md files
- Add `| blog | project |` row where missing
- Update `**Blog directory:**` fields

### Phase 5: Hygiene sweep
- Scan all repos, slots, worktrees, attic for orphaned artifacts
- Cross-reference against canonical project `docs/` directories
- Report any entries that exist only in slots/worktrees (never promoted)
- Script this as a reusable validator

### Phase 6: Validation
- Add commit-tier validator to flag artifacts in wrong locations
- Flag root-level `blog/`, `adr/`, `specs/` in project repos
- Flag `docs/superpowers/` directories

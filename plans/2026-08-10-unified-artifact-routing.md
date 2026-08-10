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
with a unified artifact resolver, and fix the blog publish pipeline to
work with the new layout.

**Architecture:** Four concerns addressed: (1) replace
`resolve_blog_dir.py` with `resolve_artifact_dir.py` that handles all
artifact types for authoring path resolution, (2) add blog to
`_docs_categories` in `close_artifacts.py` so it gets the `docs/`
prefix on project promotion, (3) fix the blog publish pipeline to read
from the correct directory regardless of routing, (4) update all
consuming skills to use the unified resolver. `routing.py` (3-layer
routing cascade) is untouched — it solves a different problem (workspace
vs project destination).

**Tech Stack:** Python 3.14, pytest, soredium skill markdown

## Global Constraints

- All artifact types (adr, specs, blog) live at root in workspaces and
  under `docs/` in project repos
- Plans are archived (not promoted) — they stay in the workspace and
  move to `plans/attic/` at work-end. They are NOT in `_docs_categories`
- Workspace authoring paths: `$WORKSPACE/<type>/`
- Project promotion paths: `$PROJECT/docs/<type>/`
- Slot-escape detection must be preserved
- Backward compatibility with `**Blog directory:**` CLAUDE.md field
- No breaking changes to `routing.py` (3-layer cascade is correct)
- `routing.py` resolves WHERE an artifact goes (workspace vs project) —
  a routing decision
- `resolve_artifact_dir.py` resolves the authoring path on disk (which
  directory to write to) — a path resolution concern
- These are two separate concerns; do not conflate them

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
- Create: `tests/test_resolve_artifact_dir.py`

**Interfaces:**
- Produces: `resolve(artifact_type, workspace, claude_text, slot_root=None) -> str`
- Produces: `resolve_with_warning(artifact_type, workspace, claude_text, slot_root=None) -> tuple[str, str]`
- Produces: CLI: `python3 resolve_artifact_dir.py <type> <workspace> <claude_md_path> [slot_root=<path>]`
  Output: `ARTIFACT_DIR=<path>`

Design note: This resolver handles path resolution — "what directory
should I write this artifact to?" It parses optional `**<Type> directory:**`
fields from CLAUDE.md (currently only `**Blog directory:**` exists in
practice; the others will be used as projects adopt the convention).
It does NOT handle routing decisions (workspace vs project) — that
remains in `routing.py`.

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
        field_name = resolve_artifact_dir.FIELD_NAMES.get(
            artifact_type, artifact_type.title()
        )
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
        field_name = resolve_artifact_dir.FIELD_NAMES.get(
            artifact_type, artifact_type.title()
        )
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

    def test_no_escape_returns_empty_warning(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        claude_text = '**Blog directory:** `blog/`'
        result, warning = resolve_artifact_dir.resolve_with_warning(
            "blog", str(workspace), claude_text,
        )
        assert result == str(workspace / "blog")
        assert warning == ""


class TestEdgeCases:
    def test_trailing_slash_stripped(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        claude_text = '**Blog directory:** `blog/`'
        result = resolve_artifact_dir.resolve("blog", str(workspace), claude_text)
        assert not result.endswith("/")

    def test_tilde_expansion(self, tmp_path):
        slot_workspace = tmp_path / "slots" / "1" / "work"
        slot_workspace.mkdir(parents=True)
        claude_text = '**Blog directory:** `~/claude/public/casehub/blog/`'
        result = resolve_artifact_dir.resolve(
            "blog", str(slot_workspace), claude_text,
            slot_root=str(tmp_path / "slots" / "1"),
        )
        # ~ expands to home dir, which is outside the slot
        assert result == str(slot_workspace / "blog")
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

This is a PATH RESOLVER, not a ROUTING RESOLVER. It answers "what directory
should I write to?" — not "should this go to workspace or project?" (that's
routing.py's job).

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
                f"Artifact directory '{resolved}' escapes slot boundary "
                f"'{slot_root}'. Falling back to {default}."
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

### Task 2: Add blog to docs-prefix promotion + fix blog publish path

Two changes in `close_artifacts.py`:
1. Add `"blog"` to `_docs_categories` so blog promotes to `docs/blog/`
2. Fix the blog publish section to find blog entries regardless of
   whether they're at `blog/` or `docs/blog/` in the scan source

The blog publish pipeline (lines 261-311) is a separate concern from
promotion — it copies entries to an external git repo (e.g.
`hortora.github.io`). It should run whenever blog artifacts exist,
regardless of routing. But the blog directory path on line 263
(`scan_source / "blog"`) must handle the `docs/blog/` alt_path case.

**Files:**
- Modify: `work-end/close_artifacts.py:157` — add `"blog"` to `_docs_categories`
- Modify: `work-end/close_artifacts.py:262-263` — resolve blog dir from scan results, not hardcoded path
- Modify: `tests/test_close_artifacts.py` — add tests

**Interfaces:**
- Consumes: existing `close_artifacts.py` API (no signature change)
- Produces: blog artifacts promoted to `$PROJECT/docs/blog/`
- Produces: blog publish reads from whichever directory contains the entries

- [ ] **Step 1: Write failing test for blog docs-prefix promotion**

Add to `tests/test_close_artifacts.py`:

```python
class TestBlogDocsPrefix:
    """Blog entries must land at project/docs/blog/, not project/blog/."""

    SCRIPT = Path(__file__).parent.parent / "work-end" / "close_artifacts.py"

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

        # Create branch with blog committed
        subprocess.run(
            ["git", "-C", str(workspace), "checkout", "-b", "issue-42-test"],
            capture_output=True, check=True,
        )
        subprocess.run(["git", "-C", str(workspace), "add", "-A"], capture_output=True)
        subprocess.run(
            ["git", "-C", str(workspace), "commit", "-m", "add blog"],
            capture_output=True, check=True,
        )

        # Default routing: blog -> project (no CLAUDE.md override)
        result = subprocess.run(
            [sys.executable, str(self.SCRIPT),
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

- [ ] **Step 3: Fix close_artifacts.py — add blog to _docs_categories**

Change line 157:
```python
# Before:
_docs_categories = {"specs", "adr"}

# After:
_docs_categories = {"specs", "adr", "blog"}
```

- [ ] **Step 4: Fix blog publish section to use scan results for path**

The blog publish section (line 262-263) hardcodes `scan_source / "blog"`.
After adding `alt_paths` for blog (Task 6), the scanner may find entries
at `docs/blog/` instead. The publish step should derive the blog
directory from the artifact paths found by the scanner.

Change lines 262-263:
```python
# Before:
    if artifacts["blog"]:
        blog_dir = scan_source / "blog"

# After:
    if artifacts["blog"]:
        # Derive blog dir from first scanned entry (handles blog/ and docs/blog/)
        first_blog = artifacts["blog"][0]
        blog_dir = scan_source / Path(first_blog).parent
```

- [ ] **Step 5: Run test to verify it passes**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestBlogDocsPrefix -v`
Expected: PASS

- [ ] **Step 6: Run full close_artifacts test suite for regressions**

Run: `python3 -m pytest tests/test_close_artifacts.py -v`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_artifacts.py tests/test_close_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: promote blog to docs/blog/ in project repos, fix publish path

Blog was excluded from _docs_categories, causing it to land at project
root instead of docs/blog/ alongside specs and adr. Also fixed blog
publish to derive the source directory from scan results instead of
hardcoding 'blog/'.

Refs #TBD"
```

---

### Task 3: Add alt_paths for blog and specs in workspace_artifacts.py

The scanner has `alt_paths` for ADR (`docs/adr`) but not for blog or
specs. All three docs-prefix types need `alt_paths` so the scanner
discovers entries authored directly in `docs/<type>/`.

**Files:**
- Modify: `work-end/workspace_artifacts.py:20,23` — add `alt_paths` for specs and blog
- Modify: `tests/test_workspace_artifacts.py` — add tests

**Interfaces:**
- Consumes: existing scanner API (no change)
- Produces: scanner finds entries in both `<type>/` and `docs/<type>/`

- [ ] **Step 1: Write failing tests**

```python
def test_finds_blog_in_docs_blog(self, tmp_path):
    """Blog entries in docs/blog/ are discovered via alt_paths."""
    (tmp_path / "docs" / "blog").mkdir(parents=True)
    (tmp_path / "docs" / "blog" / "entry.md").write_text("# Blog\n")
    result = scan(tmp_path)
    assert "docs/blog/entry.md" in result["blog"]

def test_finds_specs_in_docs_specs(self, tmp_path):
    """Spec entries in docs/specs/ are discovered via alt_paths."""
    (tmp_path / "docs" / "specs").mkdir(parents=True)
    (tmp_path / "docs" / "specs" / "design.md").write_text("# Spec\n")
    result = scan(tmp_path)
    assert "docs/specs/design.md" in result["specs"]
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAIL — `docs/blog/entry.md` and `docs/specs/design.md` not found

- [ ] **Step 3: Add alt_paths**

```python
# Line 20 — before:
"specs":     {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set()},

# After:
"specs":     {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set(),
              "alt_paths": ["docs/specs"]},

# Line 23 — before:
"blog":      {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set()},

# After:
"blog":      {"ext": ".md",  "exclude_names": {"INDEX.md"}, "exclude_dirs": set(),
              "alt_paths": ["docs/blog"]},
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Run full workspace_artifacts test suite**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/workspace_artifacts.py tests/test_workspace_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: scan docs/blog/ and docs/specs/ as alt_paths

Matches existing alt_paths pattern for adr (docs/adr/). All three
docs-prefix artifact types now discovered in both locations.

Refs #TBD"
```

---

### Task 4: Update diary form to use unified resolver

**Files:**
- Modify: `write-content/forms/diary.md:80-86` — replace `resolve_blog_dir.py` with `resolve_artifact_dir.py`

**Interfaces:**
- Consumes: `resolve_artifact_dir.py blog` from Task 1

- [ ] **Step 1: Update diary.md Step 1**

Change lines 80-86 from:
```markdown
**Resolve blog directory:**

` `` bash
python3 ~/.claude/skills/write-content/resolve_blog_dir.py <WORKSPACE> <CLAUDE_MD_PATH> [slot_root=<SLOT_ROOT>]
` ``

Read `BLOG_DIR` from output. Resolve to an absolute path.
```

To:
```markdown
**Resolve blog directory:**

` `` bash
python3 ~/.claude/skills/write-content/resolve_artifact_dir.py blog <WORKSPACE> <CLAUDE_MD_PATH> [slot_root=<SLOT_ROOT>]
` ``

Read `ARTIFACT_DIR` from output. Resolve to an absolute path.
```

- [ ] **Step 2: Search for other references to resolve_blog_dir**

```bash
grep -rn "resolve_blog_dir" ~/claude/hortora/soredium/ --include="*.md" --include="*.py" | grep -v __pycache__ | grep -v test_resolve_blog
```

Update any remaining references (e.g. diary-retrospective.md if it
references the script).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add write-content/forms/diary.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "refactor: diary form uses unified resolve_artifact_dir.py

Refs #TBD"
```

---

### Task 5: Update ADR skill routing — keep routing.py, use unified path resolver

The ADR skill currently has its own 3-layer routing cascade in Step 1
that duplicates routing logic. Simplify, but preserve the two-concern
separation:
- `routing.py` (via `ctx.py`) determines workspace vs project
- `resolve_artifact_dir.py` determines the authoring directory path

Do NOT replace the routing decision with the path resolver — they solve
different problems.

**Files:**
- Modify: `adr/SKILL.md:28-54` — simplify routing step

**Interfaces:**
- Consumes: `ctx.py` for workspace/project paths
- Consumes: `routing.py` for routing decision (workspace vs project)
- Consumes: `resolve_artifact_dir.py adr` for path resolution

- [ ] **Step 1: Update adr/SKILL.md Step 1**

Replace the three-layer routing cascade (lines 30-54) with:

```markdown
### Step 1 — Resolve write destination

Resolve paths and routing:

` `` bash
python3 ~/.claude/skills/project/ctx.py
` ``

Use `WORKSPACE` and `PROJECT` from the output as concrete strings.

Resolve routing (workspace vs project):

` `` bash
python3 ~/.claude/skills/project/routing.py ~/.claude/CLAUDE.md <WORKSPACE>/CLAUDE.md adr
` ``

Read `DESTINATION` from output (`workspace` or `project`).

Resolve authoring directory:

` `` bash
python3 ~/.claude/skills/write-content/resolve_artifact_dir.py adr <WORKSPACE> <WORKSPACE>/CLAUDE.md [slot_root=<SLOT_ROOT>]
` ``

Read `ARTIFACT_DIR` from output.

| Routing destination | Write to | git -C path |
|---------------------|----------|-------------|
| `workspace` | `$WORKSPACE/adr/` | `$WORKSPACE` |
| `project` (default) | `$PROJECT/docs/adr/` | `$PROJECT` |

Use `git -C <resolved-path>` for all git operations — never bare `git add/commit`.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add adr/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "refactor: ADR skill uses ctx.py + routing.py + resolve_artifact_dir.py

Replaces duplicated 3-layer routing cascade with the canonical tools.
Routing decision (routing.py) and path resolution (resolve_artifact_dir.py)
are kept as separate concerns.

Refs #TBD"
```

---

### Task 6: Delete resolve_blog_dir.py and its tests

Now that all consumers use `resolve_artifact_dir.py`, remove the
blog-specific resolver.

**Files:**
- Delete: `write-content/resolve_blog_dir.py`
- Delete: `tests/test_resolve_blog_dir.py`

- [ ] **Step 1: Verify no remaining references**

```bash
grep -rn "resolve_blog_dir" ~/claude/hortora/soredium/ --include="*.md" --include="*.py" | grep -v __pycache__
```

Expected: only the two files being deleted (or nothing if already
removed from all consumers).

- [ ] **Step 2: Delete files**

```bash
git -C /Users/mdproctor/claude/hortora/soredium rm write-content/resolve_blog_dir.py tests/test_resolve_blog_dir.py
```

- [ ] **Step 3: Run full test suite to verify nothing breaks**

Run: `python3 -m pytest tests/ -v --timeout=30`
Expected: ALL PASS (no imports of resolve_blog_dir remain)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add write-content/resolve_blog_dir.py tests/test_resolve_blog_dir.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "chore: remove resolve_blog_dir.py — replaced by resolve_artifact_dir.py

Refs #TBD"
```

---

### Task 7: Verify update_blog_index.py is path-safe

`update_blog_index.py` takes a file path argument and writes INDEX.md
to the same directory as the blog file. It does NOT hardcode `blog/`.
Verify this is true and add a regression test.

**Files:**
- Modify: `tests/test_resolve_artifact_dir.py` — add integration test
  (or a new test file if appropriate)

- [ ] **Step 1: Read update_blog_index.py and verify**

Confirm that `update_blog_index.py`:
- Takes `<blog-file>` as argument (line 83)
- Writes INDEX.md to `blog_file.parent / "INDEX.md"` (line 66)
- Does NOT hardcode any path

Expected: confirmed safe — uses `blog_file.parent`, not a hardcoded path.

- [ ] **Step 2: Write a test confirming it works in docs/blog/**

```python
def test_update_index_in_docs_blog(self, tmp_path):
    """update_blog_index works in docs/blog/ not just blog/."""
    blog_dir = tmp_path / "docs" / "blog"
    blog_dir.mkdir(parents=True)
    entry = blog_dir / "2026-08-10-test.md"
    entry.write_text("---\ntitle: Test\ndate: 2026-08-10\n---\n# Test\n")

    update_index(entry, summary="Test entry")

    index = blog_dir / "INDEX.md"
    assert index.exists()
    assert "2026-08-10-test.md" in index.read_text()
```

- [ ] **Step 3: Run test and commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "test: verify update_blog_index works in docs/blog/ path

Regression guard for unified artifact routing.

Refs #TBD"
```

---

### Task 8: Update SOURCES.md and CLAUDE.md references

Update documentation to reflect the unified artifact layout.

**Files:**
- Modify: `SOURCES.md` — add Blog row for project location
- Modify: `CLAUDE.md` — add `docs/blog/` to Project Artifacts table

- [ ] **Step 1: Update SOURCES.md**

Add a row:
```markdown
| Blog | docs/blog/ | Project diary entries (promoted from workspace) |
```

- [ ] **Step 2: Update CLAUDE.md Project Artifacts table**

Add row:
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

### Task 9: Explicitly document brainstorming and writing-plans scope

The `brainstorming` and `writing-plans` skills write to
`$WORKSPACE/specs/` and `$WORKSPACE/plans/` respectively. They do NOT
use `resolve_blog_dir.py` or any resolver — they hardcode the workspace
path. This is correct: during authoring, artifacts always go to the
workspace. Promotion to project `docs/` happens at work-end.

No code changes needed. This task exists to document the decision and
verify no hidden references exist.

**Files:** none

- [ ] **Step 1: Verify no resolver references in brainstorming or writing-plans**

```bash
grep -rn "resolve_blog_dir\|resolve_artifact_dir" ~/claude/hortora/soredium/brainstorming/ ~/claude/hortora/soredium/writing-plans/ 2>/dev/null
```

Expected: no results. These skills use `$WORKSPACE/specs/` and
`$WORKSPACE/plans/` directly, which is correct.

- [ ] **Step 2: No commit needed — verification only**

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
- Attribute unattributed entries in `casehub/work/blog/` to primary
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

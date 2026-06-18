# Externalise Bash Blocks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract ~175 mechanical bash blocks from SKILL.md files into callable Python scripts, eliminating permission prompts and reducing token consumption.

**Architecture:** Two-layer extraction — ctx.py absorbs ~85 cheap data lookups (Phase 1), per-skill scripts handle ~84 write operations (Phase 2). Skills retain only decision logic, content generation instructions, and user messages.

**Tech Stack:** Python 3.14, pytest, subprocess, pathlib. Scripts output KEY=VALUE (or JSON for commit_gather.py). Tests use tmp_path fixtures.

**Spec:** `docs/specs/2026-06-17-externalise-bash-blocks-design.md`

---

## File Structure

### Phase 1 — ctx.py expansion (1 commit)

| Action | Path | Responsibility |
|--------|------|----------------|
| Modify | `project-init/ctx.py` | Add 14 new fields, rename ISSUES_OK → ISSUES_STATUS |
| Create | `tests/test_ctx.py` | Tests for all ctx.py fields (existing + new) |
| Modify | 10+ SKILL.md files | Replace inline grep/ls with ctx.py field references |

### Phase 2 — per-skill scripts (1 commit per skill, 11 skills)

| Priority | Skill | Scripts created | Test file |
|----------|-------|----------------|-----------|
| 1 | workspace-init | `workspace_create.py`, `artifact_migrate.py`, `hook_install.py` | `tests/test_workspace_init_scripts.py` |
| 2 | work-end | `artifact_promote.py`, `branch_cleanup.py` | `tests/test_work_end_scripts.py` |
| 3 | git-squash | `commit_gather.py`, `rebase_exec.py`, `branch_swap.py` | `tests/test_git_squash_scripts.py` |
| 4 | work-pause | `pause_exec.py` | `tests/test_pause_exec.py` |
| 5 | work-resume | `resume_exec.py` | `tests/test_resume_exec.py` |
| 6 | issue-workflow | `issue_setup.py` | `tests/test_issue_setup.py` |
| 7 | retro-issues | `retro_create.py` | `tests/test_retro_create.py` |
| 8 | publish-blog | `blog_publish.py` | `tests/test_blog_publish.py` |
| 9 | git-commit | `commit_exec.py` | `tests/test_commit_exec.py` |
| 10 | handover | `handover_commit.py` | `tests/test_handover_commit.py` |
| 11 | work-start | `branch_create.py` | `tests/test_branch_create.py` |

---

## Task 1: Create test_ctx.py — baseline tests for existing fields

**Files:**
- Create: `tests/test_ctx.py`

Tests validate the EXISTING ctx.py output before any changes. This is the safety net — if Phase 1 breaks existing fields, these tests catch it.

- [ ] **Step 1: Write tests for existing ctx.py fields**

```python
#!/usr/bin/env python3
"""Tests for project-init/ctx.py — all fields (existing + new)."""

import subprocess
import sys
from pathlib import Path
import pytest

SCRIPT = Path(__file__).parent.parent / "project-init" / "ctx.py"


def run_ctx(cwd: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(SCRIPT)],
        capture_output=True, text=True, cwd=str(cwd),
    )


def parse(result: subprocess.CompletedProcess) -> dict:
    return dict(
        line.split("=", 1)
        for line in result.stdout.strip().splitlines()
        if "=" in line
    )


def init_repo(path: Path, claude_md: str = "") -> Path:
    """Create a git repo with optional CLAUDE.md content."""
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init"], cwd=str(path), capture_output=True)
    subprocess.run(
        ["git", "commit", "--allow-empty", "-m", "init"],
        cwd=str(path), capture_output=True,
    )
    if claude_md:
        (path / "CLAUDE.md").write_text(claude_md)
    return path


# ---------------------------------------------------------------------------
# Existing fields — baseline
# ---------------------------------------------------------------------------

class TestPathResolution:

    def test_single_repo_mode(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["WORKSPACE"] == str(repo)
        assert out["PROJECT"] == str(repo)
        assert out["SINGLE_REPO"] == "yes"

    def test_workspace_with_proj_symlink(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (workspace / "proj").symlink_to(project)
        out = parse(run_ctx(workspace))
        assert out["WORKSPACE"] == str(workspace)
        assert out["PROJECT"] == str(project)
        assert out["SINGLE_REPO"] == "no"

    def test_project_with_wksp_symlink(self, tmp_path):
        project = init_repo(tmp_path / "project")
        workspace = init_repo(tmp_path / "workspace")
        (project / "wksp").symlink_to(workspace)
        out = parse(run_ctx(project))
        assert out["WORKSPACE"] == str(workspace)
        assert out["PROJECT"] == str(project)
        assert out["SINGLE_REPO"] == "no"


class TestClaudeMdParsing:

    def test_owner_repo_extracted(self, tmp_path):
        repo = init_repo(tmp_path / "project", "GitHub repo: mdproctor/cc-praxis\n")
        out = parse(run_ctx(repo))
        assert out["OWNER_REPO"] == "mdproctor/cc-praxis"

    def test_owner_repo_empty_when_missing(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No repo field\n")
        out = parse(run_ctx(repo))
        assert out["OWNER_REPO"] == ""

    def test_base_branch_extracted(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "**Project base branch:** `develop`\n")
        out = parse(run_ctx(repo))
        assert out["BASE_BRANCH"] == "develop"

    def test_base_branch_defaults_to_main(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No base branch\n")
        out = parse(run_ctx(repo))
        assert out["BASE_BRANCH"] == "main"


class TestMetaParsing:

    def test_meta_fields_extracted(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        design = repo / "design"
        design.mkdir()
        (design / ".meta").write_text(
            "branch: issue-42-auth\n"
            "project-sha: deadbeef\n"
            "issue: 42\n"
            "issue-repo: mdproctor/cc-praxis\n"
            "covers: 42,43\n"
        )
        out = parse(run_ctx(repo))
        assert out["BRANCH_NAME"] == "issue-42-auth"
        assert out["PROJECT_SHA"] == "deadbeef"
        assert out["ISSUE_N"] == "42"
        assert out["ISSUE_REPO"] == "mdproctor/cc-praxis"
        assert out["COVERS"] == "42,43"

    def test_meta_missing_returns_empty(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["BRANCH_NAME"] == ""
        assert out["PROJECT_SHA"] == ""
        assert out["ISSUE_N"] == ""
        assert out["COVERS"] == ""


class TestFastPathChecks:

    def test_claude_ok_yes_when_project_type_present(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\ntype: java\n")
        out = parse(run_ctx(repo))
        assert out["CLAUDE_OK"] == "yes"

    def test_claude_ok_no_when_project_type_missing(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# Just a heading\n")
        out = parse(run_ctx(repo))
        assert out["CLAUDE_OK"] == "no"
```

- [ ] **Step 2: Run tests to verify they pass against current ctx.py**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: ALL PASS (these test existing behaviour)

- [ ] **Step 3: Commit baseline tests**

```bash
git -C /Users/mdproctor/claude/cc-praxis add tests/test_ctx.py
git -C /Users/mdproctor/claude/cc-praxis commit -m "test: add baseline tests for ctx.py existing fields

Refs #123"
```

---

## Task 2: Add new ctx.py fields + ISSUES_STATUS rename

**Files:**
- Modify: `project-init/ctx.py`
- Modify: `tests/test_ctx.py`

- [ ] **Step 1: Add tests for all 14 new fields to test_ctx.py**

Append these test classes to `tests/test_ctx.py`:

```python
# ---------------------------------------------------------------------------
# New fields — Phase 1
# ---------------------------------------------------------------------------

class TestProjectType:

    def test_type_colon_format(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\ntype: java\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_TYPE"] == "java"

    def test_bold_type_format(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\n**Type:** skills\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_TYPE"] == "skills"

    def test_empty_when_no_project_type_section(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No type\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_TYPE"] == ""

    def test_empty_when_section_exists_but_no_type_line(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\nSome text but no type line\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_TYPE"] == ""


class TestIssuesStatus:

    def test_enabled(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Work Tracking\n\nIssue tracking: enabled\n")
        out = parse(run_ctx(repo))
        assert out["ISSUES_STATUS"] == "enabled"

    def test_declined(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Work Tracking\n\nIssue tracking: declined\n")
        out = parse(run_ctx(repo))
        assert out["ISSUES_STATUS"] == "declined"

    def test_absent(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No tracking\n")
        out = parse(run_ctx(repo))
        assert out["ISSUES_STATUS"] == "absent"

    def test_issues_ok_removed(self, tmp_path):
        """ISSUES_OK should no longer appear in output."""
        repo = init_repo(tmp_path / "project",
            "## Work Tracking\n\nIssue tracking: enabled\n")
        out = parse(run_ctx(repo))
        assert "ISSUES_OK" not in out


class TestHasMeta:

    def test_yes_when_meta_exists(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text("branch: test\n")
        out = parse(run_ctx(repo))
        assert out["HAS_META"] == "yes"

    def test_no_when_meta_missing(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["HAS_META"] == "no"


class TestDesignRepoKey:

    def test_from_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text(
            "branch: test\ndesign-repo: workspace\n")
        out = parse(run_ctx(repo))
        assert out["DESIGN_REPO_KEY"] == "workspace"

    def test_empty_when_no_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["DESIGN_REPO_KEY"] == ""


class TestFileExistenceChecks:

    def test_has_arc42stories_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "ARC42STORIES.MD").write_text("# Stories\n")
        out = parse(run_ctx(repo))
        assert out["HAS_ARC42STORIES"] == "yes"

    def test_has_arc42stories_no(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["HAS_ARC42STORIES"] == "no"

    def test_has_project_artifacts_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Artifacts\n\n| Path | What |\n")
        out = parse(run_ctx(repo))
        assert out["HAS_PROJECT_ARTIFACTS"] == "yes"

    def test_has_project_artifacts_no(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No artifacts\n")
        out = parse(run_ctx(repo))
        assert out["HAS_PROJECT_ARTIFACTS"] == "no"

    def test_workspace_declined_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\ntype: java\nworkspace: declined\n")
        out = parse(run_ctx(repo))
        assert out["WORKSPACE_DECLINED"] == "yes"

    def test_workspace_declined_no(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "## Project Type\n\ntype: java\n")
        out = parse(run_ctx(repo))
        assert out["WORKSPACE_DECLINED"] == "no"

    def test_has_platform_doc_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "docs").mkdir()
        (repo / "docs" / "PLATFORM.md").write_text("# Platform\n")
        out = parse(run_ctx(repo))
        assert out["HAS_PLATFORM_DOC"] == "yes"

    def test_has_platform_doc_no(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["HAS_PLATFORM_DOC"] == "no"

    def test_has_protocols_dir_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "docs" / "protocols").mkdir(parents=True)
        out = parse(run_ctx(repo))
        assert out["HAS_PROTOCOLS_DIR"] == "yes"

    def test_has_protocols_dir_no(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["HAS_PROTOCOLS_DIR"] == "no"

    def test_has_blog_routing_global(self, tmp_path, monkeypatch):
        repo = init_repo(tmp_path / "project")
        home = tmp_path / "home"
        home.mkdir()
        (home / ".claude").mkdir()
        (home / ".claude" / "blog-routing.yaml").write_text("routes:\n")
        monkeypatch.setenv("HOME", str(home))
        out = parse(run_ctx(repo))
        assert out["HAS_BLOG_ROUTING"] == "yes"

    def test_has_blog_routing_no(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["HAS_BLOG_ROUTING"] == "no"

    def test_has_writing_style_ref_yes(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "Writing style: ~/styles/blog-technical.md\n")
        out = parse(run_ctx(repo))
        assert out["HAS_WRITING_STYLE_REF"] == "yes"

    def test_has_writing_style_ref_no(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No style ref\n")
        out = parse(run_ctx(repo))
        assert out["HAS_WRITING_STYLE_REF"] == "no"


class TestClaudeMdFields:

    def test_blog_dir_extracted(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "**Blog directory:** `docs/_posts/`\n")
        out = parse(run_ctx(repo))
        assert out["BLOG_DIR"] == "docs/_posts/"

    def test_blog_dir_empty_when_missing(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No blog dir\n")
        out = parse(run_ctx(repo))
        assert out["BLOG_DIR"] == ""

    def test_project_name_extracted(self, tmp_path):
        repo = init_repo(tmp_path / "project",
            "**Name:** cc-praxis\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_NAME"] == "cc-praxis"

    def test_project_name_empty_when_missing(self, tmp_path):
        repo = init_repo(tmp_path / "project", "# No name\n")
        out = parse(run_ctx(repo))
        assert out["PROJECT_NAME"] == ""


class TestMetaNewFields:

    def test_flyway_next_v_from_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text(
            "branch: test\nflyway-next-v: 17\n")
        out = parse(run_ctx(repo))
        assert out["FLYWAY_NEXT_V"] == "17"

    def test_flyway_next_v_empty_no_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["FLYWAY_NEXT_V"] == ""

    def test_meta_section_hashes_from_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text(
            "branch: test\ndesign-section-hashes: abc:## Foo|def:## Bar\n")
        out = parse(run_ctx(repo))
        assert out["META_SECTION_HASHES"] == "abc:## Foo|def:## Bar"

    def test_meta_section_hashes_empty_no_meta(self, tmp_path):
        repo = init_repo(tmp_path / "project")
        out = parse(run_ctx(repo))
        assert out["META_SECTION_HASHES"] == ""
```

- [ ] **Step 2: Run tests — new field tests should FAIL**

Run: `python3 -m pytest tests/test_ctx.py -v --tb=line 2>&1 | tail -30`
Expected: TestProjectType, TestIssuesStatus, TestHasMeta, etc. all FAIL. Baseline tests still PASS.

- [ ] **Step 3: Implement new fields in ctx.py**

Replace the entire ctx.py with the expanded version. Key changes:
- Add PROJECT_TYPE parsing (handles both `type: X` and `**Type:** X`)
- Add ISSUES_STATUS (three-state: enabled/declined/absent) — replaces ISSUES_OK
- Add .meta field extraction for design-repo, flyway-next-v, design-section-hashes
- Add file existence checks (ARC42STORIES.MD, docs/PLATFORM.md, docs/protocols/)
- Add CLAUDE.md field extraction (blog dir, project name, writing style ref)
- Add HAS_BLOG_ROUTING (checks global + project locations)
- Remove ISSUES_OK

The implementation adds ~60 lines to ctx.py (mostly `print()` statements and regex/existence checks). All new fields are cheap local operations.

Key implementation patterns:
```python
# PROJECT_TYPE — parse after "## Project Type" section
project_type = ""
if "## Project Type" in cwd_text:
    m = re.search(r"(?:^type:\s*|^\*\*Type:\*\*\s*)(\S+)", cwd_text, re.MULTILINE)
    if m:
        project_type = m.group(1)

# ISSUES_STATUS — three states
if "Issue tracking: enabled" in cwd_text:
    issues_status = "enabled"
elif "Issue tracking: declined" in cwd_text:
    issues_status = "declined"
else:
    issues_status = "absent"

# File existence checks — use project path for project files
has_arc42 = "yes" if (Path(project) / "ARC42STORIES.MD").exists() else "no"

# HAS_PLATFORM_DOC — check 4 locations (project docs/, workspace root, workspace docs/, casehub parent)
has_platform = "no"
for candidate in [
    Path(project) / "docs" / "PLATFORM.md",
    Path(workspace) / "PLATFORM.md",
    Path(workspace) / "docs" / "PLATFORM.md",
]:
    if candidate.exists():
        has_platform = "yes"
        break

# HAS_BLOG_ROUTING — check global + project
has_blog_routing = "no"
for candidate in [
    Path.home() / ".claude" / "blog-routing.yaml",
    Path(project) / "blog-routing.yaml",
    Path(workspace) / "blog-routing.yaml",
]:
    if candidate.exists():
        has_blog_routing = "yes"
        break

# .meta new fields
design_repo_key = meta.get("design-repo", "")
flyway_next_v = meta.get("flyway-next-v", "")
meta_section_hashes = meta.get("design-section-hashes", "")
```

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: ALL PASS (baseline + new fields)

- [ ] **Step 5: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/cc-praxis add project-init/ctx.py tests/test_ctx.py
git -C /Users/mdproctor/claude/cc-praxis commit -m "feat: expand ctx.py with 14 new fields, rename ISSUES_OK → ISSUES_STATUS

Add PROJECT_TYPE, ISSUES_STATUS, HAS_META, DESIGN_REPO_KEY,
HAS_ARC42STORIES, HAS_PROJECT_ARTIFACTS, WORKSPACE_DECLINED,
HAS_PLATFORM_DOC, HAS_PROTOCOLS_DIR, BLOG_DIR, HAS_BLOG_ROUTING,
PROJECT_NAME, HAS_WRITING_STYLE_REF, FLYWAY_NEXT_V, META_SECTION_HASHES.

ISSUES_OK removed — replaced by ISSUES_STATUS (enabled/declined/absent).

Refs #123"
```

---

## Task 3: Update SKILL.md files to consume ctx.py fields

**Files:**
- Modify: 12+ SKILL.md files (remove inline bash, add ctx.py field references)

This task replaces inline grep/ls/cat bash blocks with references to ctx.py output. The ctx.py call already exists in most skills (Path Resolution step). Skills that don't call ctx.py yet need the call added.

- [ ] **Step 1: Update project-type detection in 10 skills**

In each of these SKILL.md files, find the bash block that greps CLAUDE.md for project type and replace it with a reference to ctx.py's `PROJECT_TYPE` field:

| Skill | Current pattern | Replacement |
|-------|----------------|-------------|
| code-review/SKILL.md | `grep -A 2 "## Project Type" CLAUDE.md` | `Read PROJECT_TYPE from ctx.py output.` |
| dependency-update/SKILL.md | same | same |
| git-commit/SKILL.md | same (Step 0) | same |
| security-audit/SKILL.md | same | same |
| update-design/SKILL.md | same | same |
| project-health/SKILL.md | same | same |
| project-refine/SKILL.md | same (Step 0) | same |
| work-end/SKILL.md | `grep -i "^type:" CLAUDE.md` (Step 8k) | same |
| workspace-init/SKILL.md | `grep -E "^\*\*Type:\*\*\|^type:" CLAUDE.md` | same |
| project-init/SKILL.md | already uses CLAUDE_OK (section presence only) | No change needed |

For skills that don't already call ctx.py (code-review, dependency-update, security-audit, update-design): add `python3 ~/.claude/skills/project-init/ctx.py` at the start with "Read PROJECT_TYPE from output."

- [ ] **Step 2: Update issue tracking detection — atomic ISSUES_OK → ISSUES_STATUS rename**

In each of these SKILL.md files, replace every reference to `ISSUES_OK` with `ISSUES_STATUS` and update the conditional logic:

| Skill | What to change |
|-------|---------------|
| git-commit/SKILL.md | `ISSUES_OK` → `ISSUES_STATUS`, check `== "enabled"` not `== "yes"` |
| issue-workflow/SKILL.md | same |
| update-claude-md/SKILL.md | same |
| project-init/SKILL.md | Replace three-grep detection with `ISSUES_STATUS` (enabled/declined/absent) |
| workspace-init/SKILL.md | same pattern |
| work-start/SKILL.md | same pattern |

Also update any bash blocks that independently grep for "Issue tracking: enabled" — replace with "Read ISSUES_STATUS from ctx.py output."

- [ ] **Step 3: Update remaining field references**

For each new ctx.py field, find the SKILL.md files that grep/ls for the same data and replace:

| ctx.py field | Skills to update | Current bash pattern |
|-------------|-----------------|---------------------|
| HAS_META | work-pause, handover, work-start | `ls .meta`, `cat .meta` |
| DESIGN_REPO_KEY | work-end | `grep design-repo .meta` |
| HAS_ARC42STORIES | handover, update-claude-md, update-design, work-end | `[ -f ARC42STORIES.MD ]` |
| HAS_PROJECT_ARTIFACTS | git-squash, workspace-init | `grep "## Project Artifacts" CLAUDE.md` |
| WORKSPACE_DECLINED | project-init | `grep "workspace: declined" CLAUDE.md` |
| HAS_PLATFORM_DOC | work-start | `ls docs/PLATFORM.md` (4 locations) |
| HAS_PROTOCOLS_DIR | work-start | `ls protocols/` (4 locations) |
| BLOG_DIR | publish-blog | `grep "blog directory" CLAUDE.md` |
| HAS_BLOG_ROUTING | publish-blog, git-squash, workspace-init | `ls blog-routing.yaml` |
| PROJECT_NAME | update-claude-md | `grep "**Name:**" CLAUDE.md` |
| HAS_WRITING_STYLE_REF | update-claude-md | `grep "writing style" CLAUDE.md` |

- [ ] **Step 4: Fix work/SKILL.md anti-pattern**

In `work/SKILL.md`, replace the CLAUDE.md grep for `**Workspace:**` with a ctx.py call:

Before:
```bash
WORKSPACE=$(grep "^\*\*Workspace:\*\*" CLAUDE.md | head -1 | sed "s/.*\`\(.*\)\`.*/\1/")
```

After:
```
Run `python3 ~/.claude/skills/project-init/ctx.py`.
Read WORKSPACE from output.
```

- [ ] **Step 5: Validate all modified SKILL.md files**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 6: Commit Phase 1 SKILL.md updates**

```bash
git -C /Users/mdproctor/claude/cc-praxis add -A
git -C /Users/mdproctor/claude/cc-praxis commit -m "refactor: replace inline grep/ls in SKILL.md files with ctx.py fields

Update 12+ skills to consume PROJECT_TYPE, ISSUES_STATUS, HAS_META,
DESIGN_REPO_KEY, HAS_ARC42STORIES, and other ctx.py fields instead of
inline bash commands. Fix work/SKILL.md CLAUDE.md grep anti-pattern.

Refs #123"
```

---

## Task 4: workspace-init scripts (Phase 2, Priority 1)

**Files:**
- Create: `workspace-init/workspace_create.py`
- Create: `workspace-init/artifact_migrate.py`
- Create: `workspace-init/hook_install.py`
- Create: `tests/test_workspace_init_scripts.py`
- Modify: `workspace-init/SKILL.md`

workspace-init has 44 blocks total: 12 instructional (templates), 13 DATA (→ ctx.py in Task 3), 1 already externalised (create_symlinks.py), 18 OPERATION.

### workspace_create.py

Creates workspace directories, INDEX.md files, and stub files (HANDOFF.md, IDEAS.md).

**Subcommands:**

| Subcommand | Args | Output | What it does |
|------------|------|--------|-------------|
| `create-dirs` | `<workspace>` | `CREATED=yes/no` | mkdir -p for plans/, specs/, snapshots/, adr/, blog/, design/ |
| `create-indexes` | `<workspace>` | `CREATED=<count>` | Write INDEX.md for snapshots/, adr/, blog/ (skip if exists) |
| `create-stubs` | `<workspace>` | `CREATED=<count>` | Write HANDOFF.md and IDEAS.md stubs (skip if exists) |
| `init-repo` | `<workspace> name=<name> visibility=<public/private>` | `REPO_URL=<url>` or `ERROR=<code>` | git init, git add, git commit, gh repo create, git remote add, git push |

### artifact_migrate.py

Migrates existing methodology artifacts from project repo to workspace.

**Subcommands:**

| Subcommand | Args | Output | What it does |
|------------|------|--------|-------------|
| `scan` | `<project>` | JSON: `{"found": ["HANDOFF.md", "blog/", ...]}` | Check for artifacts at project root and docs/ |
| `migrate` | `<project> <workspace> paths=<comma-sep>` | `MIGRATED=<count>` | cp -r to workspace, git rm from project, git commit both |

### hook_install.py

Installs git hooks from skill directories into project repo.

**Subcommands:**

| Subcommand | Args | Output | What it does |
|------------|------|--------|-------------|
| `install` | `<project> hook-src=<path> hook-name=<name>` | `INSTALLED=yes/skipped` | cp, chmod +x, git add, git commit |
| `configure` | `<project>` | `CONFIGURED=yes` | git config core.hooksPath |

- [ ] **Step 1: Write tests for workspace_create.py**

Test all subcommands: create-dirs (happy path, idempotent), create-indexes (content, skip existing), create-stubs (content, skip existing), init-repo (success, already initialised).

- [ ] **Step 2: Implement workspace_create.py**

Follow the scaffold.py pattern: argparse with subcommands, subprocess for git/gh, KEY=VALUE output, ERROR= on failure.

- [ ] **Step 3: Write tests for artifact_migrate.py**

Test: scan finds artifacts, scan empty project, migrate moves files, migrate git operations.

- [ ] **Step 4: Implement artifact_migrate.py**

- [ ] **Step 5: Write tests for hook_install.py**

Test: install copies and chmods, install skips existing, configure sets hooksPath.

- [ ] **Step 6: Implement hook_install.py**

- [ ] **Step 7: Run all new tests**

Run: `python3 -m pytest tests/test_workspace_init_scripts.py -v`
Expected: ALL PASS

- [ ] **Step 8: Update workspace-init/SKILL.md**

Replace the 18 OPERATION bash blocks with script calls. Instructional blocks (templates) stay unchanged.

- [ ] **Step 9: Validate**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/cc-praxis add workspace-init/ tests/test_workspace_init_scripts.py
git -C /Users/mdproctor/claude/cc-praxis commit -m "feat(workspace-init): externalise 18 operation blocks to 3 scripts

workspace_create.py: dir creation, INDEX.md, stubs, git init + gh repo create
artifact_migrate.py: scan + migrate existing artifacts from project to workspace
hook_install.py: install git hooks, configure core.hooksPath

Refs #123"
```

---

## Task 5: work-end scripts (Phase 2, Priority 2)

**Files:**
- Create: `work-end/artifact_promote.py`
- Create: `work-end/branch_cleanup.py`
- Create: `tests/test_work_end_scripts.py`
- Modify: `work-end/SKILL.md`

work-end has 31 blocks: 3 already externalised, 12 DATA (→ ctx.py), 16 OPERATION.

### artifact_promote.py

Promotes workspace artifacts to main or project repo.

| Subcommand | Args | Output |
|------------|------|--------|
| `to-workspace-main` | `<workspace> branch=<name> artifacts=<comma-sep-paths>` | `PROMOTED=<count>` |
| `to-project` | `<project> <workspace> artifacts=<comma-sep-paths>` | `PROMOTED=<count>` |
| `cleanup-specs` | `<workspace> branch=<name>` | `CLEANED=<count>` |
| `merge-journal` | `<workspace> <project> branch=<name> baseline-sha=<sha> design-repo=<key>` | `MERGED=yes/no` |
| `close-issues` | `<repo> covers=<comma-sep-issues>` | `CLOSED=<count>` |
| `publish-blog` | `<workspace> <project>` | `PUBLISHED=<count>` (delegates to blog_dest.py) |

### branch_cleanup.py

Handles branch closing operations.

| Subcommand | Args | Output |
|------------|------|--------|
| `create-epic-closed` | `<workspace> branch=<name> single-repo=<yes/no> date=<YYYY-MM-DD> issues=<csv>` | `CREATED=yes/no` |
| `cleanup-scaffold` | `<workspace> single-repo=<yes/no>` | `CLEANED=yes/no` |
| `cleanup-stack` | `<workspace> branch=<name>` | `REMOVED=yes/no` |
| `checkout-main` | `<project> <workspace>` | `SWITCHED=yes` |

- [ ] **Steps 1-6: TDD cycle** (same pattern as Task 4 — write tests, implement, verify)
- [ ] **Step 7: Update work-end/SKILL.md** — replace 16 OPERATION blocks
- [ ] **Step 8: Validate and commit**

---

## Task 6: git-squash scripts (Phase 2, Priority 3)

**Files:**
- Create: `git-squash/commit_gather.py`
- Create: `git-squash/rebase_exec.py`
- Create: `git-squash/branch_swap.py`
- Create: `tests/test_git_squash_scripts.py`
- Modify: `git-squash/SKILL.md`

git-squash has 36 blocks: 3 already externalised, 2 instructional, 16 DATA, 15 OPERATION.

### commit_gather.py (JSON output — exception to KEY=VALUE protocol)

Collects classification data for Steps 3a-3i. Called AFTER filter-repo (SHAs stable).

**Args:** `<project> range=<base..head>`
**Output:** JSON (see spec for full schema)

Gathers per-commit: sha, subject, body, author, date, files, insertions, deletions, issue_refs, patch_id. Plus repo-level: is_conventional, PR data (if `gh` available).

### rebase_exec.py

Executes the squash plan non-interactively.

| Subcommand | Args | Output |
|------------|------|--------|
| `single` | `<project>` | `SQUASHED=yes` | git reset --soft HEAD~1, commit --amend |
| `multi` | `<project> base=<sha> todo-file=<path>` | `REBASED=yes, COMMITS_BEFORE=N, COMMITS_AFTER=M` |
| `amend-message` | `<project> message=<msg>` | `AMENDED=yes` |

### branch_swap.py

Handles Step 8 branch rename and force push.

**Args:** `<project> orig=<branch> work=<branch>`
**Output:** `SWAPPED=yes, BACKUP=<backup-branch-name>`

Performs: branch rename (orig → backup), branch rename (work → orig), set upstream, force-push with lease. Includes post-swap verification (git status, unpushed check).

### git-squash/ctx.py update

Add `base-branch=<value>` argument support. When provided, skip CLAUDE.md parsing for BASE_BRANCH and use the argument directly. The SKILL.md passes the value from project-init/ctx.py output.

- [ ] **Steps 1-8: TDD cycle per script**
- [ ] **Step 9: Update git-squash/SKILL.md** — replace 15 OPERATION blocks, update ctx.py call to pass base-branch
- [ ] **Step 10: Validate and commit**

---

## Tasks 7-11: Remaining full scripts

Each follows the identical pattern: write tests → implement script → update SKILL.md → validate → commit.

### Task 7: work-pause — `pause_exec.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `commit-wip` | `<repo> message=<msg>` | `COMMITTED=yes/clean` (clean if nothing to commit) |
| `push-and-stack` | `<workspace> <project> branch=<name> issue=<N> base-branch=<base>` | `STACKED=yes` |

7 OPERATION blocks eliminated.

### Task 8: work-resume — `resume_exec.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `checkout-branches` | `<project> <workspace> branch=<name>` | `CHECKED_OUT=yes` |
| `rebase` | `<project> <workspace> base-branch=<base>` | `REBASED=yes/skipped` |
| `reset-wip` | `<project> <workspace>` | `RESET=yes/no` (no if HEAD isn't a WIP commit) |

5 OPERATION blocks eliminated.

### Task 9: issue-workflow — `issue_setup.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `create-labels` | `<repo>` | `CREATED=<count>` |
| `install-hooks` | `<project>` | `INSTALLED=yes` |
| `create-epic` | `<repo> title=<t> body-file=<path>` | `ISSUE_NUMBER=<N>` |
| `create-issue` | `<repo> title=<t> body-file=<path> labels=<csv>` | `ISSUE_NUMBER=<N>` |
| `update-scope` | `<repo> epic=<N> body-file=<path>` | `UPDATED=yes` |

5 OPERATION blocks eliminated.

### Task 10: retro-issues — `retro_create.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `create-epic` | `<repo> title=<t> body-file=<path>` | `ISSUE_NUMBER=<N>` |
| `create-issue` | `<repo> title=<t> body-file=<path> labels=<csv> close=<yes/no>` | `ISSUE_NUMBER=<N>` |
| `close-issue` | `<repo> issue=<N>` | `CLOSED=yes` |
| `commit-mapping` | `<project> file=<path>` | `COMMITTED=yes` |

4 OPERATION blocks eliminated.

### Task 11: publish-blog — `blog_publish.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `copy-entry` | `<source-path> <dest-dir>` | `COPIED=yes` |
| `commit-destination` | `<dest-repo> files=<csv> message=<msg>` | `COMMITTED=yes` |
| `remove-source` | `<source-repo> files=<csv>` | `REMOVED=<count>` |

5 OPERATION blocks eliminated.

---

## Tasks 12-14: Thin scripts

### Task 12: git-commit — `commit_exec.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `commit` | `<project> message=<msg> files=<csv>` | `COMMITTED=yes, SHA=<sha>` |
| `squash` | `<project>` | `SQUASHED=yes` (git reset --soft HEAD~1) |
| `stage-docs` | `<project> files=<csv>` | `STAGED=<count>` |

4 OPERATION blocks eliminated. Thin because most git-commit blocks are DATA (→ ctx.py).

### Task 13: handover — `handover_commit.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `commit-to-main` | `<workspace> branch=<current>` | `COMMITTED=yes` |

Handles the full stash → checkout main → pull → add → commit → push → checkout back → stash pop sequence. 3 OPERATION blocks eliminated.

### Task 14: work-start — `branch_create.py`

| Subcommand | Args | Output |
|------------|------|--------|
| `create-branches` | `<project> <workspace> branch=<name> base=<base>` | `CREATED=yes` |
| `commit-scaffold` | `<workspace> branch=<name>` | `COMMITTED=yes` |

2 OPERATION blocks eliminated. work-start is already well-externalised (scaffold.py, flyway_scan.py, ctx.py, routing.py, section_hashes.py). This script handles the remaining git operations.

---

## Validation Checklist

After all tasks complete:

- [ ] `python3 -m pytest tests/ -v` — full test suite passes (existing + new)
- [ ] `python3 scripts/validate_all.py --tier commit` — no SKILL.md regressions
- [ ] No SKILL.md contains `ISSUES_OK` (replaced by ISSUES_STATUS)
- [ ] work/SKILL.md no longer greps CLAUDE.md for `**Workspace:**`
- [ ] Each new script has a corresponding test file
- [ ] Each test file covers: happy path, 2+ edge cases, bad argument handling
- [ ] No hardcoded paths in scripts or tests

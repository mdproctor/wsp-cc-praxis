# Workspace Rename Script — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #595 — Finish the work lifecycle
**Issue group:** #595

**Goal:** Build a script that safely renames a workspace repo across all
locations that reference it: GitHub, local directories, git internals,
worklog database, Claude Code session history, and documentation files.

**Architecture:** Two Python modules — `rename_topo.py` (shared topology
discovery, reusable by a future project-rename script) and
`rename_workspace.py` (workspace-specific orchestration with
audit/plan/apply phases). Tests use scratch git layouts with real repos.

**Tech Stack:** Python 3, subprocess (git), sqlite3, pathlib, pytest

## Global Constraints

- Python scripts live in `scripts/`, tests in `tests/`
- Follow `fix_absolute_symlinks.py` pattern: dry-run default, `--apply` to execute
- Use `slot_test_helpers.init_repo` patterns for test fixtures
- Never touch historical content (blog, specs, retro-issues)
- Idempotent: re-running `--apply` is safe (each step checks before acting)
- Worklog backup with sqlite3 backup API before any DB write

---

## Batch 1: Topology Discovery Layer

### Task 1: Discovery and finding types + workspace dir scanner

**Files:**
- Create: `scripts/rename_topo.py`
- Create: `tests/test_rename_topo.py`

**Interfaces:**
- Produces: `Finding(location, current, proposed, category)` dataclass
- Produces: `WorkspaceDir(path, layout, active, rename_dir)` dataclass
- Produces: `discover_workspace_dirs(family_root, old_name) -> list[WorkspaceDir]`

- [ ] **Step 1: Write tests for data types and discovery**

```python
# tests/test_rename_topo.py
import subprocess
import sys
from pathlib import Path

import pytest

scripts_dir = Path(__file__).parent.parent / "scripts"
sys.path.insert(0, str(scripts_dir))


def _init_repo(path: Path) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init"], cwd=str(path), capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"],
                   capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"],
                   capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty", "-m", "init"],
                   capture_output=True, check=True)
    return path


@pytest.fixture
def family(tmp_path):
    """Minimal family layout with canonical, clone, slot and attic."""
    root = tmp_path / "hortora"
    root.mkdir()

    # Canonical workspace
    public = tmp_path / "public"
    canonical = public / "old-name"
    _init_repo(canonical)

    # Project repo
    project = root / "myproject"
    _init_repo(project)

    # Clone workspace
    clone_root = root / "clone" / "myproject"
    clone_root.mkdir(parents=True)
    clone_ws = clone_root / "wsp-old-name"
    _init_repo(clone_ws)
    # Set up alternates + remote
    alt_dir = clone_ws / ".git" / "objects" / "info"
    alt_dir.mkdir(parents=True, exist_ok=True)
    (alt_dir / "alternates").write_text(
        f"{canonical / '.git' / 'objects'}\n")
    subprocess.run(["git", "-C", str(clone_ws), "remote", "add", "local",
                    str(canonical)], capture_output=True)

    # Active slot
    slot9 = root / "slots" / "9"
    slot9.mkdir(parents=True)
    slot_ws = slot9 / "work-old-name"
    _init_repo(slot_ws)
    alt_dir9 = slot_ws / ".git" / "objects" / "info"
    alt_dir9.mkdir(parents=True, exist_ok=True)
    (alt_dir9 / "alternates").write_text(
        f"{canonical / '.git' / 'objects'}\n")
    subprocess.run(["git", "-C", str(slot_ws), "remote", "add", "local",
                    str(canonical)], capture_output=True)

    # Attic slot
    attic3 = root / "slots" / "attic" / "3"
    attic3.mkdir(parents=True)
    attic_ws = attic3 / "work-old-name"
    _init_repo(attic_ws)
    alt_dir3 = attic_ws / ".git" / "objects" / "info"
    alt_dir3.mkdir(parents=True, exist_ok=True)
    (alt_dir3 / "alternates").write_text(
        f"{canonical / '.git' / 'objects'}\n")

    return {
        "root": root,
        "public": public,
        "canonical": canonical,
        "project": project,
        "clone_root": clone_root,
        "clone_ws": clone_ws,
        "slot_ws": slot_ws,
        "attic_ws": attic_ws,
    }


def test_discover_finds_canonical(family):
    from rename_topo import discover_workspace_dirs
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    canonical = [d for d in dirs if d.layout == "canonical"]
    assert len(canonical) == 1
    assert canonical[0].path == family["canonical"]
    assert canonical[0].rename_dir is True


def test_discover_finds_clone(family):
    from rename_topo import discover_workspace_dirs
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    clones = [d for d in dirs if d.layout == "clone"]
    assert len(clones) == 1
    assert clones[0].path == family["clone_ws"]
    assert clones[0].rename_dir is True


def test_discover_finds_active_slot(family):
    from rename_topo import discover_workspace_dirs
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    slots = [d for d in dirs if d.layout == "slot"]
    assert len(slots) == 1
    assert slots[0].active is True
    assert slots[0].rename_dir is False


def test_discover_finds_attic_slot(family):
    from rename_topo import discover_workspace_dirs
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    attics = [d for d in dirs if d.layout == "attic"]
    assert len(attics) == 1
    assert attics[0].active is False
    assert attics[0].rename_dir is False
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_rename_topo.py -v
```

Expected: `ModuleNotFoundError: No module named 'rename_topo'`

- [ ] **Step 3: Implement data types and discovery**

```python
# scripts/rename_topo.py
"""Topology discovery for workspace and project renames."""

from __future__ import annotations

import os
import re
import sqlite3
import subprocess
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class Finding:
    location: str
    current: str
    proposed: str
    category: str  # "mechanical" | "historical" | "advisory"


@dataclass
class WorkspaceDir:
    path: Path
    layout: str  # "canonical" | "clone" | "slot" | "attic" | "quarantine"
    active: bool
    rename_dir: bool


def discover_workspace_dirs(
    family_root: Path,
    old_name: str,
    public_dir: Path | None = None,
) -> list[WorkspaceDir]:
    """Find every workspace directory matching old_name in the family."""
    if public_dir is None:
        public_dir = family_root.parent / "public"

    dirs: list[WorkspaceDir] = []

    # Canonical workspace
    canonical = public_dir / old_name
    if canonical.is_dir():
        dirs.append(WorkspaceDir(
            path=canonical, layout="canonical", active=True, rename_dir=True))

    # Clone workspaces
    clone_base = family_root / "clone"
    if clone_base.is_dir():
        for project_dir in sorted(clone_base.iterdir()):
            if not project_dir.is_dir():
                continue
            for ws_name in (f"wsp-{old_name}", f"work-{old_name}"):
                ws = project_dir / ws_name
                if ws.is_dir():
                    dirs.append(WorkspaceDir(
                        path=ws, layout="clone", active=True,
                        rename_dir=ws_name.startswith("wsp-")))

    # Slots (active, attic, quarantine)
    for sub, layout, active in [
        ("slots", "slot", True),
        (os.path.join("slots", "attic"), "attic", False),
        (os.path.join("slots", "quarantine"), "quarantine", False),
    ]:
        base = family_root / sub
        if not base.is_dir():
            continue
        for slot_dir in sorted(base.iterdir()):
            if not slot_dir.is_dir():
                continue
            for ws_name in (f"wsp-{old_name}", f"work-{old_name}"):
                ws = slot_dir / ws_name
                if ws.is_dir():
                    dirs.append(WorkspaceDir(
                        path=ws, layout=layout, active=active,
                        rename_dir=False))

    return dirs
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
python3 -m pytest tests/test_rename_topo.py -v
```

Expected: 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/rename_topo.py tests/test_rename_topo.py
git -C ~/claude/hortora/soredium commit -m "feat: topology discovery for workspace rename — types and dir scanner

Refs Hortora/soredium#595"
```

### Task 2: Finders — alternates, remotes, symlinks, files, worklog, Claude projects

**Files:**
- Modify: `scripts/rename_topo.py`
- Modify: `tests/test_rename_topo.py`

**Interfaces:**
- Consumes: `WorkspaceDir`, `Finding` from Task 1
- Produces: `find_alternates(dirs, old_path) -> list[Finding]`
- Produces: `find_remotes(dirs, old_path) -> list[Finding]`
- Produces: `find_symlinks(family_root, old_path) -> list[Finding]`
- Produces: `find_worklog_rows(old_path, db_path) -> list[Finding]`
- Produces: `find_claude_projects(old_path, new_path) -> list[Finding]`
- Produces: `find_file_references(file_path, old_patterns, new_patterns) -> list[Finding]`
- Produces: `mangle_path(path) -> str` (Claude project dir encoding)

- [ ] **Step 1: Write tests for all finders**

```python
# append to tests/test_rename_topo.py

def test_find_alternates(family):
    from rename_topo import discover_workspace_dirs, find_alternates
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    old_path = str(family["canonical"])
    findings = find_alternates(dirs, old_path)
    assert len(findings) == 3  # clone + slot + attic
    assert all(f.category == "mechanical" for f in findings)
    new_path = str(family["public"] / "new-name")
    for f in findings:
        assert new_path in f.proposed


def test_find_remotes(family):
    from rename_topo import discover_workspace_dirs, find_remotes
    dirs = discover_workspace_dirs(
        family["root"], "old-name", public_dir=family["public"])
    old_path = str(family["canonical"])
    findings = find_remotes(dirs, old_path)
    assert len(findings) == 2  # clone + slot (attic has no remote)
    assert all(f.category == "mechanical" for f in findings)


def test_find_symlinks(family):
    from rename_topo import find_symlinks
    # Create a wksp symlink pointing at canonical
    wksp = family["project"] / "wksp"
    wksp.symlink_to(family["canonical"])
    findings = find_symlinks(family["root"], str(family["canonical"]),
                             str(family["public"] / "new-name"))
    assert len(findings) == 1
    assert "new-name" in findings[0].proposed


def test_find_file_references(family):
    from rename_topo import find_file_references
    claude_md = family["canonical"] / "CLAUDE.md"
    claude_md.write_text(
        "workspace: ~/claude/public/old-name\n"
        "history: old-name was renamed from something\n"
    )
    findings = find_file_references(
        claude_md,
        ["~/claude/public/old-name"],
        ["~/claude/public/new-name"],
    )
    assert len(findings) == 1
    assert "new-name" in findings[0].proposed


def test_find_worklog_rows(tmp_path):
    from rename_topo import find_worklog_rows
    db = tmp_path / "worklog.db"
    conn = sqlite3.connect(str(db))
    conn.execute("CREATE TABLE repos (id INTEGER PRIMARY KEY, path TEXT, a, b, c, d)")
    conn.execute("INSERT INTO repos VALUES (1, '/old/path', NULL, NULL, NULL, NULL)")
    conn.execute("INSERT INTO repos VALUES (2, '/other', NULL, NULL, NULL, NULL)")
    conn.commit()
    conn.close()
    findings = find_worklog_rows("/old/path", "/new/path", db_path=db)
    assert len(findings) == 1
    assert findings[0].current == "/old/path"
    assert findings[0].proposed == "/new/path"


def test_find_claude_projects(tmp_path):
    from rename_topo import find_claude_projects, mangle_path
    projects_dir = tmp_path / ".claude" / "projects"
    old_mangled = mangle_path("/Users/me/claude/public/old-name")
    (projects_dir / old_mangled).mkdir(parents=True)
    findings = find_claude_projects(
        "/Users/me/claude/public/old-name",
        "/Users/me/claude/public/new-name",
        projects_dir=projects_dir,
    )
    assert len(findings) == 1
    assert "new-name" in findings[0].proposed


def test_mangle_path():
    from rename_topo import mangle_path
    assert mangle_path("/Users/me/claude/public/foo") == "-Users-me-claude-public-foo"
```

- [ ] **Step 2: Run tests — verify new tests fail**

```bash
python3 -m pytest tests/test_rename_topo.py -v -k "find_ or mangle"
```

Expected: `ImportError` or `AttributeError` for missing functions

- [ ] **Step 3: Implement all finders**

Add to `scripts/rename_topo.py`:

```python
def find_alternates(
    dirs: list[WorkspaceDir],
    old_path: str,
    new_path: str | None = None,
) -> list[Finding]:
    """Find .git/objects/info/alternates referencing old_path."""
    findings: list[Finding] = []
    for d in dirs:
        alt_file = d.path / ".git" / "objects" / "info" / "alternates"
        if not alt_file.exists():
            continue
        content = alt_file.read_text()
        if old_path in content:
            proposed = content.replace(old_path, new_path) if new_path else ""
            findings.append(Finding(
                location=str(alt_file),
                current=content.strip(),
                proposed=proposed.strip(),
                category="mechanical",
            ))
    return findings


def find_remotes(
    dirs: list[WorkspaceDir],
    old_path: str,
    new_path: str | None = None,
) -> list[Finding]:
    """Find git remotes referencing old_path."""
    findings: list[Finding] = []
    for d in dirs:
        result = subprocess.run(
            ["git", "-C", str(d.path), "remote", "-v"],
            capture_output=True, text=True)
        if result.returncode != 0:
            continue
        for line in result.stdout.splitlines():
            if old_path in line and "(fetch)" in line:
                parts = line.split()
                remote_name = parts[0]
                remote_url = parts[1]
                proposed_url = remote_url.replace(old_path, new_path) if new_path else ""
                findings.append(Finding(
                    location=f"{d.path}:{remote_name}",
                    current=remote_url,
                    proposed=proposed_url,
                    category="mechanical",
                ))
    return findings


def find_symlinks(
    family_root: Path,
    old_path: str,
    new_path: str,
) -> list[Finding]:
    """Find wksp/proj symlinks pointing at old_path."""
    findings: list[Finding] = []
    search_dirs = [family_root]
    clone_base = family_root / "clone"
    if clone_base.is_dir():
        for project_dir in sorted(clone_base.iterdir()):
            if project_dir.is_dir():
                for sub in sorted(project_dir.iterdir()):
                    if sub.is_dir() and (sub / ".git").exists():
                        search_dirs.append(sub)

    for base in search_dirs:
        for child in sorted(base.iterdir()):
            if not child.is_dir() or not (child / ".git").exists():
                continue
            for name in ("wksp", "proj"):
                link = child / name
                if link.is_symlink():
                    target = os.readlink(link)
                    if old_path in target:
                        findings.append(Finding(
                            location=str(link),
                            current=target,
                            proposed=target.replace(old_path, new_path),
                            category="mechanical",
                        ))
    return findings


def find_file_references(
    file_path: Path,
    old_patterns: list[str],
    new_patterns: list[str],
) -> list[Finding]:
    """Find lines in a file containing old patterns (not in historical sections)."""
    if not file_path.exists():
        return []
    findings: list[Finding] = []
    lines = file_path.read_text().splitlines()
    in_historical = False
    for i, line in enumerate(lines, 1):
        heading = line.strip().lstrip("#").strip().lower()
        if heading and any(w in heading for w in ("history", "migration", "retro", "legacy")):
            in_historical = True
        elif line.startswith("#"):
            in_historical = False
        if in_historical:
            continue
        for old_p, new_p in zip(old_patterns, new_patterns):
            if old_p in line:
                findings.append(Finding(
                    location=f"{file_path}:{i}",
                    current=line.strip(),
                    proposed=line.replace(old_p, new_p).strip(),
                    category="mechanical",
                ))
                break
    return findings


def find_worklog_rows(
    old_path: str,
    new_path: str,
    db_path: Path | None = None,
) -> list[Finding]:
    """Find worklog repos table rows with old_path."""
    if db_path is None:
        db_path = Path.home() / ".hortora" / "worklog.db"
    if not db_path.exists():
        return []
    findings: list[Finding] = []
    conn = sqlite3.connect(str(db_path))
    try:
        for row in conn.execute(
            "SELECT id, path FROM repos WHERE path LIKE ?",
            (f"%{old_path}%",),
        ):
            findings.append(Finding(
                location=f"worklog:repos:id={row[0]}",
                current=row[1],
                proposed=row[1].replace(old_path, new_path),
                category="mechanical",
            ))
    finally:
        conn.close()
    return findings


def mangle_path(path: str) -> str:
    """Encode a path the way Claude Code does for project directories."""
    return path.replace("/", "-")


def find_claude_projects(
    old_path: str,
    new_path: str,
    projects_dir: Path | None = None,
) -> list[Finding]:
    """Find .claude/projects/ directories keyed by old_path."""
    if projects_dir is None:
        projects_dir = Path.home() / ".claude" / "projects"
    if not projects_dir.is_dir():
        return []
    old_mangled = mangle_path(old_path)
    new_mangled = mangle_path(new_path)
    findings: list[Finding] = []
    for d in sorted(projects_dir.iterdir()):
        if d.is_dir() and d.name == old_mangled:
            findings.append(Finding(
                location=str(d),
                current=d.name,
                proposed=new_mangled,
                category="mechanical",
            ))
    return findings
```

- [ ] **Step 4: Run all tests — verify they pass**

```bash
python3 -m pytest tests/test_rename_topo.py -v
```

Expected: all 11 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/rename_topo.py tests/test_rename_topo.py
git -C ~/claude/hortora/soredium commit -m "feat: topology finders — alternates, remotes, symlinks, worklog, Claude projects

Refs Hortora/soredium#595"
```

---

## Batch 2: Workspace Rename Script

### Task 3: Audit, plan, and apply phases

**Files:**
- Create: `scripts/rename_workspace.py`
- Create: `tests/test_rename_workspace.py`

**Interfaces:**
- Consumes: all types and functions from `rename_topo.py`
- Produces: CLI `python3 scripts/rename_workspace.py <family> old=X new=Y [plan] [--apply]`

- [ ] **Step 1: Write integration test with full fixture**

```python
# tests/test_rename_workspace.py
import os
import sqlite3
import subprocess
import sys
from pathlib import Path
from unittest.mock import patch

import pytest

scripts_dir = Path(__file__).parent.parent / "scripts"
sys.path.insert(0, str(scripts_dir))


def _init_repo(path: Path, files: dict[str, str] | None = None) -> Path:
    path.mkdir(parents=True, exist_ok=True)
    subprocess.run(["git", "init"], cwd=str(path), capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "Test"],
                   capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"],
                   capture_output=True)
    if files:
        for name, content in files.items():
            (path / name).parent.mkdir(parents=True, exist_ok=True)
            (path / name).write_text(content)
        subprocess.run(["git", "-C", str(path), "add", "."],
                       capture_output=True, check=True)
    subprocess.run(["git", "-C", str(path), "commit", "--allow-empty", "-m", "init"],
                   capture_output=True, check=True)
    return path


@pytest.fixture
def workspace_family(tmp_path):
    """Full family layout for rename testing."""
    root = tmp_path / "hortora"
    root.mkdir()
    public = tmp_path / "public"

    claude_md_content = (
        "# CLAUDE.md\n\n"
        "- **workspace:** `~/claude/public/old-ws`\n"
        "- GitHub: `testuser/wsp-old-ws`\n"
    )
    handoff_content = (
        "# Handover\n\n"
        "**Where to work:** clone at `~/path/wsp-old-ws/`\n"
        "## History\n"
        "old-ws was fine in the past\n"
    )

    # Canonical workspace
    canonical = _init_repo(public / "old-ws", {
        "CLAUDE.md": claude_md_content,
        "HANDOFF.md": handoff_content,
    })
    # Set up origin remote (bare repo for GitHub stand-in)
    bare = tmp_path / "bare" / "wsp-old-ws.git"
    bare.mkdir(parents=True)
    subprocess.run(["git", "init", "--bare", str(bare)], capture_output=True)
    subprocess.run(["git", "-C", str(canonical), "remote", "add", "origin",
                    f"https://github.com/testuser/wsp-old-ws.git"],
                   capture_output=True)

    # Project repo with wksp symlink
    project = _init_repo(root / "myproject")
    (project / "wksp").symlink_to(canonical)

    # Clone
    clone_root = root / "clone" / "myproject"
    clone_root.mkdir(parents=True)
    clone_ws = _init_repo(clone_root / "wsp-old-ws", {
        "CLAUDE.md": claude_md_content,
        "HANDOFF.md": handoff_content,
    })
    alt_dir = clone_ws / ".git" / "objects" / "info"
    alt_dir.mkdir(parents=True, exist_ok=True)
    (alt_dir / "alternates").write_text(f"{canonical / '.git' / 'objects'}\n")
    subprocess.run(["git", "-C", str(clone_ws), "remote", "add", "local",
                    str(canonical)], capture_output=True)
    # .queue-root
    (clone_root / ".queue-root").write_text(
        f"since: 2026-01-01\nadopted-from: {clone_ws}/.queue\n")

    # Attic slot
    attic = root / "slots" / "attic" / "3" / "work-old-ws"
    _init_repo(attic)
    alt_dir3 = attic / ".git" / "objects" / "info"
    alt_dir3.mkdir(parents=True, exist_ok=True)
    (alt_dir3 / "alternates").write_text(f"{canonical / '.git' / 'objects'}\n")

    # Worklog
    db = tmp_path / "worklog.db"
    conn = sqlite3.connect(str(db))
    conn.execute("CREATE TABLE repos (id INTEGER PRIMARY KEY, path TEXT, a, b, c, d)")
    conn.execute("INSERT INTO repos VALUES (1, ?, NULL, NULL, NULL, NULL)",
                 (str(canonical),))
    conn.execute("INSERT INTO repos VALUES (2, ?, NULL, NULL, NULL, NULL)",
                 (str(clone_ws),))
    conn.execute("INSERT INTO repos VALUES (3, '/other/repo', NULL, NULL, NULL, NULL)")
    conn.commit()
    conn.close()

    # Claude projects dir
    claude_projects = tmp_path / ".claude" / "projects"
    from rename_topo import mangle_path
    old_mangled = mangle_path(str(canonical))
    (claude_projects / old_mangled).mkdir(parents=True)
    (claude_projects / old_mangled / "memory").mkdir()

    return {
        "root": root,
        "public": public,
        "canonical": canonical,
        "project": project,
        "clone_root": clone_root,
        "clone_ws": clone_ws,
        "attic_ws": attic,
        "db": db,
        "claude_projects": claude_projects,
        "tmp": tmp_path,
    }


def test_audit_finds_all_layers(workspace_family):
    from rename_workspace import audit
    findings = audit(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    categories = {f.location.split(":")[0] for f in findings
                  if f.category == "mechanical"}
    assert "alternates" in str(findings) or any("alternates" in f.location for f in findings)
    assert len(findings) >= 8  # alternates, remote, symlink, worklog, claude, CLAUDE.md, HANDOFF.md


def test_apply_renames_directories(workspace_family):
    from rename_workspace import apply
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    assert (workspace_family["public"] / "new-ws").is_dir()
    assert not (workspace_family["public"] / "old-ws").exists()
    assert (workspace_family["clone_root"] / "wsp-new-ws").is_dir()
    assert not (workspace_family["clone_root"] / "wsp-old-ws").exists()
    # Attic dir name preserved
    assert (workspace_family["root"] / "slots" / "attic" / "3" / "work-old-ws").is_dir()


def test_apply_fixes_alternates(workspace_family):
    from rename_workspace import apply
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    new_canonical = workspace_family["public"] / "new-ws"
    expected = str(new_canonical / ".git" / "objects")
    # Check clone alternates
    clone_alt = (workspace_family["clone_root"] / "wsp-new-ws" /
                 ".git" / "objects" / "info" / "alternates")
    assert expected in clone_alt.read_text()
    # Check attic alternates
    attic_alt = (workspace_family["root"] / "slots" / "attic" / "3" /
                 "work-old-ws" / ".git" / "objects" / "info" / "alternates")
    assert expected in attic_alt.read_text()


def test_apply_fixes_symlinks(workspace_family):
    from rename_workspace import apply
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    wksp = workspace_family["project"] / "wksp"
    target = os.readlink(wksp)
    assert "new-ws" in target
    assert "old-ws" not in target


def test_apply_updates_worklog(workspace_family):
    from rename_workspace import apply
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    # Backup exists
    backup = workspace_family["db"].parent / (
        workspace_family["db"].name + ".before-rename-" +
        __import__("datetime").date.today().strftime("%Y%m%d"))
    assert backup.exists()
    # Rows updated
    conn = sqlite3.connect(str(workspace_family["db"]))
    rows = conn.execute("SELECT path FROM repos WHERE id IN (1, 2)").fetchall()
    conn.close()
    assert all("new-ws" in r[0] for r in rows)
    assert all("old-ws" not in r[0] for r in rows)


def test_apply_renames_claude_projects(workspace_family):
    from rename_workspace import apply
    from rename_topo import mangle_path
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    new_canonical = workspace_family["public"] / "new-ws"
    new_mangled = mangle_path(str(new_canonical))
    assert (workspace_family["claude_projects"] / new_mangled).is_dir()


def test_apply_rewrites_handoff_where_to_work(workspace_family):
    from rename_workspace import apply
    apply(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    handoff = (workspace_family["public"] / "new-ws" / "HANDOFF.md").read_text()
    assert "wsp-new-ws" in handoff
    # Historical section preserved
    assert "old-ws was fine in the past" in handoff


def test_idempotent_rerun(workspace_family):
    from rename_workspace import apply
    kwargs = dict(
        family_root=workspace_family["root"],
        old_name="old-ws",
        new_name="new-ws",
        public_dir=workspace_family["public"],
        db_path=workspace_family["db"],
        projects_dir=workspace_family["claude_projects"],
        skip_github=True,
    )
    apply(**kwargs)
    # Second run should be all no-ops
    result = apply(**kwargs)
    assert result == 0  # 0 actions taken
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_rename_workspace.py -v
```

Expected: `ModuleNotFoundError: No module named 'rename_workspace'`

- [ ] **Step 3: Implement rename_workspace.py**

```python
# scripts/rename_workspace.py
#!/usr/bin/env python3
"""
Safely rename a workspace repo across all locations.

Usage:
    python3 scripts/rename_workspace.py <family_root> old=<name> new=<name>          # audit
    python3 scripts/rename_workspace.py <family_root> old=<name> new=<name> plan     # plan
    python3 scripts/rename_workspace.py <family_root> old=<name> new=<name> --apply  # apply
"""

from __future__ import annotations

import datetime
import os
import shutil
import sqlite3
import subprocess
import sys
from pathlib import Path

scripts_dir = Path(__file__).resolve().parent
sys.path.insert(0, str(scripts_dir))

from rename_topo import (
    Finding, WorkspaceDir, discover_workspace_dirs,
    find_alternates, find_remotes, find_symlinks,
    find_worklog_rows, find_claude_projects, find_file_references,
    mangle_path,
)


def _github_user(canonical: Path) -> str | None:
    """Extract GitHub username from origin remote URL."""
    result = subprocess.run(
        ["git", "-C", str(canonical), "remote", "get-url", "origin"],
        capture_output=True, text=True)
    if result.returncode != 0:
        return None
    url = result.stdout.strip()
    # https://github.com/USER/REPO.git or git@github.com:USER/REPO.git
    for prefix in ("https://github.com/", "git@github.com:"):
        if url.startswith(prefix):
            return url[len(prefix):].split("/")[0]
    return None


def audit(
    family_root: Path,
    old_name: str,
    new_name: str,
    public_dir: Path | None = None,
    db_path: Path | None = None,
    projects_dir: Path | None = None,
    skip_github: bool = False,
) -> list[Finding]:
    """Read-only scan for everything that needs changing."""
    if public_dir is None:
        public_dir = family_root.parent / "public"

    old_canonical = public_dir / old_name
    new_canonical = public_dir / new_name
    old_path = str(old_canonical)
    new_path = str(new_canonical)

    findings: list[Finding] = []

    # Discover all workspace dirs
    dirs = discover_workspace_dirs(family_root, old_name, public_dir=public_dir)

    # GitHub
    if not skip_github and old_canonical.is_dir():
        user = _github_user(old_canonical)
        if user:
            findings.append(Finding(
                location="github",
                current=f"{user}/wsp-{old_name}",
                proposed=f"{user}/wsp-{new_name}",
                category="mechanical",
            ))

    # Directory renames
    for d in dirs:
        if d.rename_dir:
            if d.layout == "canonical":
                new_dir = public_dir / new_name
            else:
                new_dir = d.path.parent / d.path.name.replace(old_name, new_name)
            findings.append(Finding(
                location=f"dir:{d.layout}",
                current=str(d.path),
                proposed=str(new_dir),
                category="mechanical",
            ))

    # Alternates
    findings.extend(find_alternates(dirs, old_path, new_path))

    # Remotes
    findings.extend(find_remotes(dirs, old_path, new_path))

    # Symlinks
    findings.extend(find_symlinks(family_root, old_path, new_path))

    # File references (CLAUDE.md, HANDOFF.md in canonical + clone)
    old_patterns = [f"~/claude/public/{old_name}", f"wsp-{old_name}",
                    f"work-{old_name}/"]  # only in directory refs, not renames
    new_patterns = [f"~/claude/public/{new_name}", f"wsp-{new_name}",
                    f"work-{old_name}/"]  # keep work-X directory names
    # Actually for HANDOFF where-to-work: match wsp-old -> wsp-new
    for d in dirs:
        if not d.path.is_dir():
            continue
        for doc in ("CLAUDE.md", "HANDOFF.md"):
            doc_path = d.path / doc
            if doc_path.exists():
                findings.extend(find_file_references(
                    doc_path,
                    [f"~/claude/public/{old_name}", f"wsp-{old_name}"],
                    [f"~/claude/public/{new_name}", f"wsp-{new_name}"],
                ))
    # Also check canonical project CLAUDE.md
    for child in sorted(family_root.iterdir()):
        if child.is_dir() and (child / ".git").exists():
            cm = child / "CLAUDE.md"
            if cm.exists():
                findings.extend(find_file_references(
                    cm,
                    [f"~/claude/public/{old_name}", f"wsp-{old_name}"],
                    [f"~/claude/public/{new_name}", f"wsp-{new_name}"],
                ))

    # .queue-root
    clone_base = family_root / "clone"
    if clone_base.is_dir():
        for project_dir in sorted(clone_base.iterdir()):
            qr = project_dir / ".queue-root"
            if qr.exists() and old_name in qr.read_text():
                findings.extend(find_file_references(
                    qr,
                    [f"wsp-{old_name}", f"work-{old_name}"],
                    [f"wsp-{new_name}", f"work-{old_name}"],
                ))

    # Worklog
    findings.extend(find_worklog_rows(old_path, new_path, db_path=db_path))
    # Also check for clone workspace path
    for d in dirs:
        if d.layout in ("clone", "slot") and str(d.path) != old_path:
            findings.extend(find_worklog_rows(
                str(d.path),
                str(d.path).replace(old_name, new_name),
                db_path=db_path,
            ))

    # Claude projects
    findings.extend(find_claude_projects(old_path, new_path,
                                         projects_dir=projects_dir))

    return findings


def apply(
    family_root: Path,
    old_name: str,
    new_name: str,
    public_dir: Path | None = None,
    db_path: Path | None = None,
    projects_dir: Path | None = None,
    skip_github: bool = False,
) -> int:
    """Execute the rename. Returns count of actions taken."""
    if public_dir is None:
        public_dir = family_root.parent / "public"

    old_canonical = public_dir / old_name
    new_canonical = public_dir / new_name
    old_path = str(old_canonical)
    new_path = str(new_canonical)
    actions = 0

    # 1. GitHub rename
    if not skip_github and old_canonical.is_dir():
        user = _github_user(old_canonical)
        if user:
            result = subprocess.run(
                ["gh", "repo", "rename", f"wsp-{new_name}",
                 "--repo", f"{user}/wsp-{old_name}", "--yes"],
                capture_output=True, text=True)
            if result.returncode == 0:
                print(f"OK: GitHub {user}/wsp-{old_name} → wsp-{new_name}")
                actions += 1
            else:
                print(f"FAIL: GitHub rename: {result.stderr.strip()}")
                return -1

    # 2. Canonical directory rename
    if old_canonical.is_dir():
        old_canonical.rename(new_canonical)
        print(f"OK: {old_canonical} → {new_canonical}")
        actions += 1

    # Update old_path references to reflect the rename
    old_path_resolved = str(new_canonical)  # canonical is now at new path

    # 3. Clone/slot directory renames
    dirs = discover_workspace_dirs(family_root, old_name, public_dir=public_dir)
    for d in dirs:
        if d.rename_dir and d.path.is_dir():
            new_dir = d.path.parent / d.path.name.replace(old_name, new_name)
            if not new_dir.exists():
                d.path.rename(new_dir)
                print(f"OK: {d.path} → {new_dir}")
                actions += 1

    # Re-discover with new names in place
    all_dirs = discover_workspace_dirs(family_root, new_name, public_dir=public_dir)
    # Also add non-renamed dirs (attic/slot with work-old-name)
    all_dirs.extend(discover_workspace_dirs(family_root, old_name, public_dir=public_dir))
    # Deduplicate
    seen = set()
    deduped = []
    for d in all_dirs:
        if str(d.path) not in seen:
            seen.add(str(d.path))
            deduped.append(d)
    all_dirs = deduped

    # 4. Alternates
    for f in find_alternates(all_dirs, old_path, new_path):
        alt_file = Path(f.location)
        if alt_file.exists():
            content = alt_file.read_text()
            alt_file.write_text(content.replace(old_path, new_path))
            print(f"OK: alternates {alt_file}")
            actions += 1

    # 5. Remotes
    for f in find_remotes(all_dirs, old_path, new_path):
        ws_path, remote_name = f.location.rsplit(":", 1)
        subprocess.run(
            ["git", "-C", ws_path, "remote", "set-url", remote_name, f.proposed],
            capture_output=True, check=True)
        print(f"OK: remote {ws_path}:{remote_name} → {f.proposed}")
        actions += 1

    # GitHub remote on canonical
    if new_canonical.is_dir() and not skip_github:
        result = subprocess.run(
            ["git", "-C", str(new_canonical), "remote", "get-url", "origin"],
            capture_output=True, text=True)
        if result.returncode == 0 and old_name in result.stdout:
            new_url = result.stdout.strip().replace(
                f"wsp-{old_name}", f"wsp-{new_name}")
            subprocess.run(
                ["git", "-C", str(new_canonical), "remote", "set-url",
                 "origin", new_url],
                capture_output=True, check=True)
            print(f"OK: canonical origin → {new_url}")
            actions += 1

    # 6. Symlinks
    for f in find_symlinks(family_root, old_path, new_path):
        link = Path(f.location)
        if link.is_symlink():
            link.unlink()
            os.symlink(f.proposed, link)
            print(f"OK: symlink {link} → {f.proposed}")
            actions += 1

    # 7. Worklog
    if db_path is None:
        db_path = Path.home() / ".hortora" / "worklog.db"
    if db_path.exists():
        worklog_findings = find_worklog_rows(old_path, new_path, db_path=db_path)
        # Also check clone paths
        for d in all_dirs:
            if str(d.path) != new_path:
                old_d = str(d.path).replace(new_name, old_name)
                worklog_findings.extend(
                    find_worklog_rows(old_d, str(d.path), db_path=db_path))
        if worklog_findings:
            today = datetime.date.today().strftime("%Y%m%d")
            backup = db_path.parent / f"{db_path.name}.before-rename-{today}"
            if not backup.exists():
                src = sqlite3.connect(str(db_path))
                dst = sqlite3.connect(str(backup))
                src.backup(dst)
                dst.close()
                src.close()
                print(f"OK: worklog backup → {backup}")
            conn = sqlite3.connect(str(db_path))
            for f in worklog_findings:
                _id = int(f.location.split("id=")[1])
                conn.execute("UPDATE repos SET path = ? WHERE id = ?",
                             (f.proposed, _id))
            conn.commit()
            conn.close()
            actions += len(worklog_findings)
            print(f"OK: worklog {len(worklog_findings)} rows updated")

    # 8. CLAUDE.md references
    old_pats = [f"~/claude/public/{old_name}", f"wsp-{old_name}"]
    new_pats = [f"~/claude/public/{new_name}", f"wsp-{new_name}"]
    for d in all_dirs:
        if not d.path.is_dir():
            continue
        for doc in ("CLAUDE.md", "HANDOFF.md"):
            doc_path = d.path / doc
            refs = find_file_references(doc_path, old_pats, new_pats)
            if refs:
                content = doc_path.read_text()
                for old_p, new_p in zip(old_pats, new_pats):
                    content = content.replace(old_p, new_p)
                doc_path.write_text(content)
                print(f"OK: {doc_path} ({len(refs)} lines)")
                actions += 1
    # Project CLAUDE.md
    for child in sorted(family_root.iterdir()):
        if child.is_dir() and (child / ".git").exists():
            cm = child / "CLAUDE.md"
            refs = find_file_references(cm, old_pats, new_pats)
            if refs:
                content = cm.read_text()
                for old_p, new_p in zip(old_pats, new_pats):
                    content = content.replace(old_p, new_p)
                cm.write_text(content)
                print(f"OK: {cm} ({len(refs)} lines)")
                actions += 1

    # 9. .queue-root
    clone_base = family_root / "clone"
    if clone_base.is_dir():
        for project_dir in sorted(clone_base.iterdir()):
            qr = project_dir / ".queue-root"
            if qr.exists() and old_name in qr.read_text():
                content = qr.read_text()
                content = content.replace(f"wsp-{old_name}", f"wsp-{new_name}")
                qr.write_text(content)
                print(f"OK: {qr}")
                actions += 1

    # 10. Claude projects
    if projects_dir is None:
        projects_dir = Path.home() / ".claude" / "projects"
    for f in find_claude_projects(old_path, new_path,
                                   projects_dir=projects_dir):
        old_dir = Path(f.location)
        new_dir = old_dir.parent / f.proposed
        if old_dir.is_dir() and not new_dir.exists():
            old_dir.rename(new_dir)
            print(f"OK: Claude projects {old_dir.name} → {f.proposed}")
            actions += 1

    return actions


def main() -> int:
    args = sys.argv[1:]
    if len(args) < 3:
        print(__doc__.strip())
        return 1

    family_root = Path(args[0]).expanduser().resolve()
    old_name = new_name = ""
    mode = "audit"
    for arg in args[1:]:
        if arg.startswith("old="):
            old_name = arg[4:]
        elif arg.startswith("new="):
            new_name = arg[4:]
        elif arg == "plan":
            mode = "plan"
        elif arg == "--apply":
            mode = "apply"

    if not old_name or not new_name:
        print("Required: old=<name> new=<name>")
        return 1

    if mode == "apply":
        count = apply(family_root, old_name, new_name)
        if count < 0:
            return 1
        print(f"\n{count} actions applied.")
        return 0

    findings = audit(family_root, old_name, new_name)
    mechanical = [f for f in findings if f.category == "mechanical"]
    historical = [f for f in findings if f.category == "historical"]
    advisory = [f for f in findings if f.category == "advisory"]

    if not findings:
        print("Nothing to do.")
        return 0

    print(f"{'LAYER':<15} {'LOCATION':<55} {'ACTION'}")
    print("-" * 100)
    for f in mechanical:
        loc = f.location if len(f.location) <= 55 else "..." + f.location[-52:]
        print(f"{'mechanical':<15} {loc:<55} {f.current[:30]} → {f.proposed[:30]}")

    if historical:
        print(f"\n{len(historical)} historical references (no change)")
    if advisory:
        print(f"\n{len(advisory)} advisory items")

    if mode == "plan":
        print(f"\nRun with --apply to execute {len(mechanical)} actions.")

    return 1 if mechanical else 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
python3 -m pytest tests/test_rename_workspace.py -v
```

Expected: all tests PASS

- [ ] **Step 5: Run full test suite**

```bash
python3 -m pytest tests/test_rename_topo.py tests/test_rename_workspace.py -v
```

Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git -C ~/claude/hortora/soredium add scripts/rename_workspace.py tests/test_rename_workspace.py
git -C ~/claude/hortora/soredium commit -m "feat: workspace rename script — audit, plan, apply phases

Three-mode script that discovers workspace references across all slots,
clones, and attic, then rewrites alternates, remotes, symlinks, worklog,
Claude project dirs, and documentation references.

Refs Hortora/soredium#595"
```

### Task 4: CLAUDE.md and README updates

**Files:**
- Modify: `CLAUDE.md` (Developer Workflow section)

- [ ] **Step 1: Add rename_workspace.py to Developer Workflow**

Add to the scripts section in CLAUDE.md:

```markdown
# Workspace rename
python3 scripts/rename_workspace.py <family_root> old=<name> new=<name>          # audit
python3 scripts/rename_workspace.py <family_root> old=<name> new=<name> plan     # plan
python3 scripts/rename_workspace.py <family_root> old=<name> new=<name> --apply  # apply
```

- [ ] **Step 2: Commit**

```bash
git -C ~/claude/hortora/soredium add CLAUDE.md
git -C ~/claude/hortora/soredium commit -m "docs: add rename_workspace.py to developer workflow

Refs Hortora/soredium#595"
```

---

## References

- `specs/2026-11-20-workspace-rename-design.md` — design spec
- `scripts/fix_absolute_symlinks.py` — dry-run/apply pattern
- `scripts/reconcile_slots.py` — 3-phase audit/strategy/execute pattern
- `work-slot/slot_claude.py` — Claude project directory path mangling
- `tests/slot_test_helpers.py` — test fixture patterns for git repos
- Hortora/soredium#595 — parent epic

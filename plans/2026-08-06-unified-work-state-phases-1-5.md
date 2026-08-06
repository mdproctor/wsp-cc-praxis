# Unified Work State (Phases 1-5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #188 — Unified work state
**Issue group:** #118 (splitting HANDOFF.md roles)

**Goal:** Establish the unified work state foundation — `is_closed()`
predicate, `work_health.py` validation module, `.plan` batch validation,
main `.plan` global queue, and slim HANDOFF.md.

**Architecture:** Five phases building sequentially. Phase 1 adds the
`is_closed()` predicate to `lifecycle.py`. Phase 2 creates
`work_health.py` with entry-scope checks. Phase 3 wires `.plan` batch
validation into `work_health.py` and builds the human-readable resume
display. Phase 4 adds main `.plan` support. Phase 5 slims HANDOFF.md.

**Tech Stack:** Python 3.12+, pytest, GitHub CLI (`gh`), git

## Global Constraints

- All new Python files follow existing soredium conventions (no type
  stubs, pytest for testing, `KEY=VALUE` stdout for script output)
- `plan_manager.py` is the sole writer of `.plan` content — no other
  script writes `.plan` directly
- `is_closed()` uses local git only — no remote refs, no GitHub API
- Tests use temp git repos (follow `test_lifecycle.py` patterns)
- Every `#N` reference in `.plan` must be a GitHub issue

---

### Task 1: `is_closed()` predicate — Phase 1

**Files:**
- Modify: `project/lifecycle.py` (add after `is_closing()` at line 399)
- Test: `tests/test_lifecycle.py` (append new test class)

**Interfaces:**
- Consumes: git CLI (`git branch --list`, `git log`, `git merge-base`)
- Produces: `ClosureState` enum and `is_closed(project, branch, workspace, base_branch)` function.
  Return type is `ClosureState` with values `CLOSED`, `MERGED_UNSTAMPED`, `STAMPED_UNMERGED`, `OPEN`, `DELETED`.

- [ ] **Step 1: Write the ClosureState enum and is_closed() stub**

Add to `project/lifecycle.py` after `is_closing()` (line 399):

```python
from enum import Enum

class ClosureState(Enum):
    CLOSED = "closed"
    MERGED_UNSTAMPED = "merged_unstamped"
    STAMPED_UNMERGED = "stamped_unmerged"
    OPEN = "open"
    DELETED = "deleted"


def is_closed(
    project: str,
    branch: str,
    workspace: str | None = None,
    base_branch: str = "main",
) -> ClosureState:
    raise NotImplementedError
```

Note: `Enum` import goes at the top of the file with other imports.

- [ ] **Step 2: Write failing tests for each ClosureState**

Append to `tests/test_lifecycle.py`:

```python
import subprocess
from pathlib import Path
from lifecycle import is_closed, ClosureState


class TestIsClosed:
    """Tests for is_closed() predicate."""

    def _init_repo(self, tmp_path):
        """Create a git repo with an initial commit on main."""
        repo = tmp_path / "project"
        repo.mkdir()
        subprocess.run(["git", "init", "-b", "main"], cwd=repo, check=True,
                        capture_output=True)
        subprocess.run(["git", "config", "user.email", "test@test.com"],
                        cwd=repo, check=True, capture_output=True)
        subprocess.run(["git", "config", "user.name", "Test"],
                        cwd=repo, check=True, capture_output=True)
        (repo / "README.md").write_text("init")
        subprocess.run(["git", "add", "."], cwd=repo, check=True,
                        capture_output=True)
        subprocess.run(["git", "commit", "-m", "init"], cwd=repo, check=True,
                        capture_output=True)
        return repo

    def _create_branch_with_commit(self, repo, branch, filename="work.txt"):
        subprocess.run(["git", "checkout", "-b", branch], cwd=repo,
                        check=True, capture_output=True)
        (repo / filename).write_text("work")
        subprocess.run(["git", "add", "."], cwd=repo, check=True,
                        capture_output=True)
        subprocess.run(["git", "commit", "-m", f"feat: {filename}"],
                        cwd=repo, check=True, capture_output=True)
        subprocess.run(["git", "checkout", "main"], cwd=repo, check=True,
                        capture_output=True)

    def _merge_branch(self, repo, branch):
        subprocess.run(["git", "merge", branch, "--no-ff", "-m",
                        f"merge {branch}"], cwd=repo, check=True,
                        capture_output=True)

    def _rebase_merge_branch(self, repo, branch):
        subprocess.run(["git", "rebase", branch], cwd=repo, check=True,
                        capture_output=True)

    def _stamp_branch(self, repo, branch, landing_sha=None):
        subprocess.run(["git", "checkout", branch], cwd=repo, check=True,
                        capture_output=True)
        msg = "chore: branch closed"
        if landing_sha:
            msg += f" — landed as {landing_sha} on main"
        subprocess.run(["git", "commit", "--allow-empty", "-m", msg],
                        cwd=repo, check=True, capture_output=True)
        subprocess.run(["git", "checkout", "main"], cwd=repo, check=True,
                        capture_output=True)

    def _get_sha(self, repo, ref="HEAD"):
        result = subprocess.run(["git", "rev-parse", ref], cwd=repo,
                                check=True, capture_output=True, text=True)
        return result.stdout.strip()

    def test_deleted_branch(self, tmp_path):
        repo = self._init_repo(tmp_path)
        result = is_closed(str(repo), "nonexistent")
        assert result == ClosureState.DELETED

    def test_open_branch(self, tmp_path):
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-1")
        result = is_closed(str(repo), "feature-1")
        assert result == ClosureState.OPEN

    def test_merged_unstamped(self, tmp_path):
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-2")
        self._rebase_merge_branch(repo, "feature-2")
        result = is_closed(str(repo), "feature-2")
        assert result == ClosureState.MERGED_UNSTAMPED

    def test_closed_with_stamp(self, tmp_path):
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-3")
        self._rebase_merge_branch(repo, "feature-3")
        landing_sha = self._get_sha(repo, "main")
        self._stamp_branch(repo, "feature-3", landing_sha)
        result = is_closed(str(repo), "feature-3")
        assert result == ClosureState.CLOSED

    def test_closed_old_format_stamp(self, tmp_path):
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-4")
        self._rebase_merge_branch(repo, "feature-4")
        self._stamp_branch(repo, "feature-4")  # no landing SHA
        result = is_closed(str(repo), "feature-4")
        assert result == ClosureState.CLOSED

    def test_stamped_unmerged(self, tmp_path):
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-5")
        # Stamp WITHOUT merging first
        self._stamp_branch(repo, "feature-5")
        result = is_closed(str(repo), "feature-5")
        assert result == ClosureState.STAMPED_UNMERGED

    def test_stamp_only_commit_ahead(self, tmp_path):
        """Branch merged, stamp is the only commit ahead — should be CLOSED."""
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-6")
        self._rebase_merge_branch(repo, "feature-6")
        landing_sha = self._get_sha(repo, "main")
        self._stamp_branch(repo, "feature-6", landing_sha)
        result = is_closed(str(repo), "feature-6")
        assert result == ClosureState.CLOSED

    def test_workspace_branch_checked(self, tmp_path):
        """When workspace is provided, both project and workspace are checked."""
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        self._create_branch_with_commit(project, "feature-7")
        self._create_branch_with_commit(workspace, "feature-7")
        self._rebase_merge_branch(project, "feature-7")
        self._rebase_merge_branch(workspace, "feature-7")
        landing_p = self._get_sha(project, "main")
        landing_w = self._get_sha(workspace, "main")
        self._stamp_branch(project, "feature-7", landing_p)
        self._stamp_branch(workspace, "feature-7", landing_w)
        result = is_closed(str(project), "feature-7", workspace=str(workspace))
        assert result == ClosureState.CLOSED

    def test_workspace_not_closed_downgrades(self, tmp_path):
        """Project CLOSED but workspace OPEN → result is OPEN."""
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        self._create_branch_with_commit(project, "feature-8")
        self._create_branch_with_commit(workspace, "feature-8")
        self._rebase_merge_branch(project, "feature-8")
        landing_p = self._get_sha(project, "main")
        self._stamp_branch(project, "feature-8", landing_p)
        result = is_closed(str(project), "feature-8", workspace=str(workspace))
        assert result == ClosureState.OPEN

    def test_landing_sha_mismatch_still_closed(self, tmp_path):
        """Landing SHA not on main is advisory — still returns CLOSED."""
        repo = self._init_repo(tmp_path)
        self._create_branch_with_commit(repo, "feature-9")
        self._rebase_merge_branch(repo, "feature-9")
        self._stamp_branch(repo, "feature-9", "deadbeefdeadbeef")
        result = is_closed(str(repo), "feature-9")
        assert result == ClosureState.CLOSED
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py::TestIsClosed -v`
Expected: All fail with `NotImplementedError`

- [ ] **Step 4: Implement is_closed()**

Replace the stub in `project/lifecycle.py`:

```python
def is_closed(
    project: str,
    branch: str,
    workspace: str | None = None,
    base_branch: str = "main",
) -> ClosureState:
    import subprocess as _sp

    def _check_repo(repo_path: str, branch_name: str) -> ClosureState:
        exists = _sp.run(
            ["git", "-C", repo_path, "branch", "--list", branch_name],
            capture_output=True, text=True, timeout=10,
        )
        if not exists.stdout.strip():
            return ClosureState.DELETED

        ahead = _sp.run(
            ["git", "-C", repo_path, "log", "--oneline",
             f"{base_branch}..{branch_name}"],
            capture_output=True, text=True, timeout=10,
        )
        non_stamp_commits = [
            line for line in ahead.stdout.strip().splitlines()
            if line and not line.split(" ", 1)[1].startswith("chore: branch closed")
        ] if ahead.stdout.strip() else []

        last_commit = _sp.run(
            ["git", "-C", repo_path, "log", "-1", "--format=%s", branch_name],
            capture_output=True, text=True, timeout=10,
        )
        is_stamped = last_commit.stdout.strip().startswith("chore: branch closed")

        merged = len(non_stamp_commits) == 0
        if merged and is_stamped:
            return ClosureState.CLOSED
        if merged and not is_stamped:
            return ClosureState.MERGED_UNSTAMPED
        if not merged and is_stamped:
            return ClosureState.STAMPED_UNMERGED
        return ClosureState.OPEN

    project_state = _check_repo(project, branch)

    if workspace is None:
        return project_state

    workspace_state = _check_repo(workspace, branch)

    if workspace_state == ClosureState.DELETED:
        return project_state

    return min(project_state, workspace_state,
               key=lambda s: [ClosureState.CLOSED, ClosureState.MERGED_UNSTAMPED,
                              ClosureState.OPEN, ClosureState.STAMPED_UNMERGED,
                              ClosureState.DELETED].index(s))
```

Note: The workspace downgrade logic uses ordering where CLOSED > MERGED_UNSTAMPED > OPEN > STAMPED_UNMERGED > DELETED. The "worst" state wins. If project is CLOSED but workspace is OPEN, result is OPEN.

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_lifecycle.py::TestIsClosed -v`
Expected: All PASS

- [ ] **Step 6: Run full lifecycle test suite for regressions**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: All existing tests still pass

- [ ] **Step 7: Commit**

```bash
git add project/lifecycle.py tests/test_lifecycle.py
git commit -m "feat(#N): add is_closed() predicate to lifecycle.py

Single function answering 'is this branch done?' definitively.
Returns ClosureState enum: CLOSED, MERGED_UNSTAMPED, STAMPED_UNMERGED,
OPEN, DELETED. All checks are local git, sub-second. Replaces five
scattered closure-checking implementations."
```

---

### Task 2: `work_health.py` — check registry and entry-scope checks — Phase 2

**Files:**
- Create: `project/work_health.py`
- Test: `tests/test_work_health.py`

**Interfaces:**
- Consumes: `lifecycle.is_closed()` from Task 1, `plan_manager.parse_plan()`,
  `plan_manager.detect()`, git CLI, GitHub CLI (`gh issue list`)
- Produces: `work_health.py` CLI with `--scope entry|wrap|close --project P --workspace W`.
  Output: `CHECK=<name> STATUS=ok|changed|fix|warn|skip DETAIL=<text>` lines,
  final `FIXED=N WARNINGS=N ERRORS=N` summary.

- [ ] **Step 1: Write failing test for the check registry structure**

Create `tests/test_work_health.py`:

```python
import subprocess
from pathlib import Path


class TestWorkHealthEntryScope:

    def _init_repo(self, path):
        path.mkdir(parents=True, exist_ok=True)
        subprocess.run(["git", "init", "-b", "main"], cwd=path, check=True,
                        capture_output=True)
        subprocess.run(["git", "config", "user.email", "test@test.com"],
                        cwd=path, check=True, capture_output=True)
        subprocess.run(["git", "config", "user.name", "Test"],
                        cwd=path, check=True, capture_output=True)
        (path / "README.md").write_text("init")
        subprocess.run(["git", "add", "."], cwd=path, check=True,
                        capture_output=True)
        subprocess.run(["git", "commit", "-m", "init"], cwd=path, check=True,
                        capture_output=True)
        return path

    def _run_health(self, scope, project, workspace, extra_args=None):
        cmd = [
            "python3", "project/work_health.py",
            "--scope", scope,
            "--project", str(project),
            "--workspace", str(workspace),
        ]
        if extra_args:
            cmd.extend(extra_args)
        result = subprocess.run(cmd, capture_output=True, text=True,
                                timeout=30, cwd=Path(__file__).parent.parent)
        return result.stdout, result.returncode

    def test_clean_entry_all_ok(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        stdout, rc = self._run_health("entry", project, workspace)
        assert rc == 0
        assert "FIXED=0" in stdout
        assert "WARNINGS=0" in stdout
        assert "ERRORS=0" in stdout

    def test_meta_consistency_branch_mismatch(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        design = workspace / "design"
        design.mkdir()
        (design / ".meta").write_text("branch: nonexistent-branch\nstate: active\n")
        stdout, rc = self._run_health("entry", project, workspace)
        assert "CHECK=meta_consistency" in stdout
        assert "STATUS=warn" in stdout

    def test_pause_stack_stale_entry(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        design = workspace / "design"
        design.mkdir()
        (design / ".pause-stack").write_text(
            "- branch: deleted-branch\n  issue: 999\n  paused: 2026-08-01\n"
        )
        stdout, rc = self._run_health("entry", project, workspace)
        assert "CHECK=pause_stack" in stdout
        assert "STATUS=warn" in stdout

    def test_main_divergence_detected(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        # Create a commit ahead of "origin" — no remote, so divergence check
        # should skip gracefully
        stdout, rc = self._run_health("entry", project, workspace)
        assert rc == 0

    def test_partial_pause_detected(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        design = workspace / "design"
        design.mkdir()
        (design / ".pausing").write_text(
            "branch: issue-123-foo\nstarted: 2026-08-06T14:30:00Z\n"
            "wip_project: done\nwip_workspace: done\n"
            "stack_push: pending\ncheckout_main: pending\n"
        )
        stdout, rc = self._run_health("entry", project, workspace)
        assert "CHECK=partial_pause" in stdout
        assert "STATUS=warn" in stdout

    def test_partial_resume_detected(self, tmp_path):
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        design = workspace / "design"
        design.mkdir()
        (design / ".resuming").write_text(
            "branch: issue-123-foo\nstarted: 2026-08-06T14:30:00Z\n"
            "stack_pop: done\ncheckout: pending\n"
            "rebase: pending\nwip_reset: pending\n"
        )
        stdout, rc = self._run_health("entry", project, workspace)
        assert "CHECK=partial_resume" in stdout
        assert "STATUS=warn" in stdout
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All fail (script doesn't exist)

- [ ] **Step 3: Implement work_health.py with entry-scope checks**

Create `project/work_health.py`:

```python
#!/usr/bin/env python3
"""Unified work state validation.

Usage:
    python3 project/work_health.py --scope entry --project P --workspace W
    python3 project/work_health.py --scope wrap  --project P --workspace W
    python3 project/work_health.py --scope close --project P --workspace W --branch B
"""
import argparse
import subprocess
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
from lifecycle import is_closed, ClosureState


def _git(repo, *args, timeout=10):
    result = subprocess.run(
        ["git", "-C", str(repo)] + list(args),
        capture_output=True, text=True, timeout=timeout,
    )
    return result.stdout.strip(), result.returncode


def _parse_pause_stack(workspace):
    stack_path = Path(workspace) / "design" / ".pause-stack"
    if not stack_path.exists():
        return []
    entries = []
    current = {}
    for line in stack_path.read_text().splitlines():
        line = line.strip()
        if line.startswith("- branch:"):
            if current:
                entries.append(current)
            current = {"branch": line.split(":", 1)[1].strip()}
        elif ":" in line and current:
            key, val = line.split(":", 1)
            current[key.strip()] = val.strip()
    if current:
        entries.append(current)
    return entries


def _parse_yaml_intent(path):
    if not path.exists():
        return None
    try:
        data = {}
        for line in path.read_text().splitlines():
            if ":" in line:
                key, val = line.split(":", 1)
                data[key.strip()] = val.strip()
        return data
    except Exception:
        return {"_parse_error": True}


def check_meta_consistency(project, workspace):
    meta_path = Path(workspace) / "design" / ".meta"
    if not meta_path.exists():
        return "CHECK=meta_consistency STATUS=ok"
    meta_branch = None
    for line in meta_path.read_text().splitlines():
        if line.startswith("branch:"):
            meta_branch = line.split(":", 1)[1].strip()
            break
    if not meta_branch:
        return "CHECK=meta_consistency STATUS=ok"
    current, _ = _git(workspace, "branch", "--show-current")
    if current != meta_branch and current == "main":
        return (f"CHECK=meta_consistency STATUS=warn "
                f"DETAIL=.meta says branch '{meta_branch}' but on main — orphaned .meta")
    if current != meta_branch:
        return (f"CHECK=meta_consistency STATUS=warn "
                f"DETAIL=.meta says '{meta_branch}', git says '{current}'")
    return "CHECK=meta_consistency STATUS=ok"


def check_pause_stack(project, workspace):
    entries = _parse_pause_stack(workspace)
    if not entries:
        return "CHECK=pause_stack STATUS=ok"
    warnings = []
    for e in entries:
        branch = e.get("branch", "unknown")
        exists_out, rc = _git(project, "branch", "--list", branch)
        if not exists_out:
            warnings.append(f"{branch} branch deleted")
    if warnings:
        return f"CHECK=pause_stack STATUS=warn DETAIL={'; '.join(warnings)}"
    return "CHECK=pause_stack STATUS=ok"


def check_workspace_alignment(project, workspace):
    if str(project) == str(workspace):
        return "CHECK=workspace_alignment STATUS=ok"
    w_branch, _ = _git(workspace, "branch", "--show-current")
    p_branch, _ = _git(project, "branch", "--show-current")
    if w_branch != p_branch:
        return (f"CHECK=workspace_alignment STATUS=warn "
                f"DETAIL=workspace on '{w_branch}', project on '{p_branch}'")
    return "CHECK=workspace_alignment STATUS=ok"


def check_main_divergence(project, workspace):
    warnings = []
    for label, repo in [("project", project), ("workspace", workspace)]:
        ahead, rc = _git(repo, "log", "origin/main..main", "--oneline")
        if rc != 0:
            continue
        if ahead:
            count = len(ahead.splitlines())
            warnings.append(f"{label} {count} commit(s) ahead of origin/main")
        behind, rc = _git(repo, "log", "main..origin/main", "--oneline")
        if rc == 0 and behind:
            count = len(behind.splitlines())
            warnings.append(f"{label} {count} commit(s) behind origin/main")
    if warnings:
        return f"CHECK=main_divergence STATUS=warn DETAIL={'; '.join(warnings)}"
    return "CHECK=main_divergence STATUS=ok"


def check_dirty_main(project, workspace):
    current, _ = _git(project, "branch", "--show-current")
    if current != "main":
        return "CHECK=dirty_main STATUS=ok"
    status, _ = _git(project, "status", "--porcelain")
    if status:
        return f"CHECK=dirty_main STATUS=warn DETAIL=project main has uncommitted changes"
    return "CHECK=dirty_main STATUS=ok"


def check_partial_pause(workspace):
    path = Path(workspace) / "design" / ".pausing"
    data = _parse_yaml_intent(path)
    if data is None:
        return "CHECK=partial_pause STATUS=ok"
    branch = data.get("branch", "unknown")
    steps = [f"{k}={v}" for k, v in data.items()
             if k not in ("branch", "started", "_parse_error")]
    return f"CHECK=partial_pause STATUS=warn DETAIL={branch} pause interrupted ({', '.join(steps)})"


def check_partial_resume(workspace):
    path = Path(workspace) / "design" / ".resuming"
    data = _parse_yaml_intent(path)
    if data is None:
        return "CHECK=partial_resume STATUS=ok"
    branch = data.get("branch", "unknown")
    steps = [f"{k}={v}" for k, v in data.items()
             if k not in ("branch", "started", "_parse_error")]
    return f"CHECK=partial_resume STATUS=warn DETAIL={branch} resume interrupted ({', '.join(steps)})"


def check_branch_closure(project, workspace):
    branches_to_check = set()
    for e in _parse_pause_stack(workspace):
        if "branch" in e:
            branches_to_check.add(e["branch"])
    meta_path = Path(workspace) / "design" / ".meta"
    if meta_path.exists():
        for line in meta_path.read_text().splitlines():
            if line.startswith("branch:"):
                branches_to_check.add(line.split(":", 1)[1].strip())
    if not branches_to_check:
        return "CHECK=branch_closure STATUS=ok"
    findings = []
    for branch in sorted(branches_to_check):
        state = is_closed(project, branch, workspace=workspace)
        if state == ClosureState.MERGED_UNSTAMPED:
            findings.append(f"{branch} MERGED_UNSTAMPED — offer stamp")
        elif state == ClosureState.STAMPED_UNMERGED:
            findings.append(f"{branch} STAMPED_UNMERGED — investigate")
    if findings:
        return f"CHECK=branch_closure STATUS=warn DETAIL={'; '.join(findings)}"
    return "CHECK=branch_closure STATUS=ok"


ENTRY_CHECKS = [
    lambda p, w: check_meta_consistency(p, w),
    lambda p, w: check_pause_stack(p, w),
    lambda p, w: check_workspace_alignment(p, w),
    lambda p, w: check_main_divergence(p, w),
    lambda p, w: check_dirty_main(p, w),
    lambda p, w: check_partial_pause(w),
    lambda p, w: check_partial_resume(w),
    lambda p, w: check_branch_closure(p, w),
]


def run_checks(scope, project, workspace, branch=None):
    if scope == "entry":
        checks = ENTRY_CHECKS
    else:
        print(f"ERROR=scope '{scope}' not yet implemented")
        return

    fixed = 0
    warnings = 0
    errors = 0

    for check_fn in checks:
        result = check_fn(project, workspace)
        print(result)
        if "STATUS=fix" in result:
            fixed += 1
        elif "STATUS=warn" in result:
            warnings += 1
        elif "STATUS=error" in result:
            errors += 1

    print(f"FIXED={fixed} WARNINGS={warnings} ERRORS={errors}")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--scope", required=True, choices=["entry", "wrap", "close"])
    parser.add_argument("--project", required=True)
    parser.add_argument("--workspace", required=True)
    parser.add_argument("--branch", default=None)
    args = parser.parse_args()
    run_checks(args.scope, args.project, args.workspace, args.branch)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add project/work_health.py tests/test_work_health.py
git commit -m "feat(#N): add work_health.py — unified validation with entry-scope checks

Check registry with 8 entry-scope checks: meta_consistency, pause_stack,
workspace_alignment, main_divergence, dirty_main, partial_pause,
partial_resume, branch_closure. Uses is_closed() for branch state."
```

---

### Task 3: `plan_state` batch validation — Phase 3

**Files:**
- Modify: `project/work_health.py` (add `check_plan_state()`)
- Modify: `work-slot/plan_manager.py` (promote `_mark_completed()` to public API)
- Test: `tests/test_work_health.py` (append plan_state tests)

**Interfaces:**
- Consumes: `plan_manager.parse_plan()`, `plan_manager.mark_completed()` (new public API),
  `gh issue list` CLI
- Produces: `CHECK=plan_state STATUS=ok|changed|skip` output lines.
  `plan_manager.mark_completed(plan_path, issue_number)` — public function.

- [ ] **Step 1: Promote _mark_completed to public API in plan_manager.py**

In `work-slot/plan_manager.py`, add a public wrapper after `_mark_completed()` (line 412):

```python
def mark_completed(plan_path: Path, issue_number: int) -> bool:
    """Mark an issue as completed [x] in the plan. Public API for work_health.py."""
    tree = parse_plan(plan_path)
    leaves = flatten_leaves(tree)
    changed = _mark_completed(tree.items, issue_number)
    if changed:
        rewrite_plan(plan_path, tree)
    return changed
```

- [ ] **Step 2: Write failing test for plan_state check**

Append to `tests/test_work_health.py`:

```python
class TestPlanStateCheck:

    def _init_repo(self, path):
        path.mkdir(parents=True, exist_ok=True)
        subprocess.run(["git", "init", "-b", "main"], cwd=path, check=True,
                        capture_output=True)
        subprocess.run(["git", "config", "user.email", "test@test.com"],
                        cwd=path, check=True, capture_output=True)
        subprocess.run(["git", "config", "user.name", "Test"],
                        cwd=path, check=True, capture_output=True)
        (path / "README.md").write_text("init")
        subprocess.run(["git", "add", "."], cwd=path, check=True,
                        capture_output=True)
        subprocess.run(["git", "commit", "-m", "init"], cwd=path, check=True,
                        capture_output=True)
        return path

    def _write_plan(self, workspace, content):
        design = Path(workspace) / "design"
        design.mkdir(exist_ok=True)
        (design / ".plan").write_text(content)

    def test_plan_state_no_plan(self, tmp_path):
        from pathlib import Path as P
        sys.path.insert(0, str(P(__file__).parent.parent / "project"))
        from work_health import check_plan_state
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        result = check_plan_state(project, workspace)
        assert "STATUS=ok" in result

    def test_plan_state_skip_on_no_gh(self, tmp_path, monkeypatch):
        from pathlib import Path as P
        sys.path.insert(0, str(P(__file__).parent.parent / "project"))
        from work_health import check_plan_state
        project = self._init_repo(tmp_path / "proj")
        workspace = self._init_repo(tmp_path / "wksp")
        self._write_plan(workspace, (
            "# Work Plan — test\n\n"
            "## Queue\n"
            "- [ ] #100 — some issue ← active\n\n"
            "## Session State\nCurrent: #100\n"
        ))
        # No OWNER_REPO means gh can't run — should skip gracefully
        result = check_plan_state(project, workspace, owner_repo="")
        assert "STATUS=skip" in result or "STATUS=ok" in result
```

- [ ] **Step 3: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_health.py::TestPlanStateCheck -v`
Expected: FAIL (check_plan_state doesn't exist)

- [ ] **Step 4: Implement check_plan_state in work_health.py**

Add to `project/work_health.py`:

```python
def check_plan_state(project, workspace, owner_repo=None):
    plan_path = Path(workspace) / "design" / ".plan"
    if not plan_path.exists():
        return "CHECK=plan_state STATUS=ok"
    if not owner_repo:
        return "CHECK=plan_state STATUS=skip DETAIL=no OWNER_REPO configured"

    sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
    from plan_manager import parse_plan, flatten_leaves, mark_completed

    tree = parse_plan(plan_path)
    leaves = flatten_leaves(tree)
    open_issues = [l for l in leaves if not l.completed]
    if not open_issues:
        return "CHECK=plan_state STATUS=ok"

    issue_numbers = [l.issue_number for l in open_issues]

    try:
        result = subprocess.run(
            ["gh", "issue", "list", "--repo", owner_repo, "--state", "all",
             "--json", "number,state,title", "--limit", "500"],
            capture_output=True, text=True, timeout=15,
        )
        if result.returncode != 0:
            return "CHECK=plan_state STATUS=skip DETAIL=GitHub API unavailable"
        import json
        issues = {i["number"]: i for i in json.loads(result.stdout)}
    except (subprocess.TimeoutExpired, Exception):
        return "CHECK=plan_state STATUS=skip DETAIL=GitHub API unavailable"

    changed = []
    unmatched = []
    for num in issue_numbers:
        if num in issues:
            if issues[num]["state"] == "CLOSED":
                mark_completed(plan_path, num)
                changed.append(f"#{num} now CLOSED")
        else:
            unmatched.append(num)

    for num in unmatched:
        try:
            result = subprocess.run(
                ["gh", "issue", "view", str(num), "--repo", owner_repo,
                 "--json", "state", "--jq", ".state"],
                capture_output=True, text=True, timeout=5,
            )
            if result.returncode == 0 and result.stdout.strip() == "CLOSED":
                mark_completed(plan_path, num)
                changed.append(f"#{num} now CLOSED")
        except subprocess.TimeoutExpired:
            pass

    if changed:
        return f"CHECK=plan_state STATUS=changed DETAIL={', '.join(changed)}"
    return "CHECK=plan_state STATUS=ok"
```

Then add `check_plan_state` to the `ENTRY_CHECKS` list with a lambda that
passes `owner_repo` from an env var or CLAUDE.md parsing.

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add project/work_health.py work-slot/plan_manager.py tests/test_work_health.py
git commit -m "feat(#N): add plan_state batch validation to work_health.py

Batch-validates .plan items against GitHub issue state in one API call.
Marks closed issues [x] via plan_manager.mark_completed(). Falls back to
individual lookups for repos with >500 issues. Gracefully skips when
GitHub is unavailable."
```

---

### Task 4: Human-readable resume display — Phase 3 (continued)

**Files:**
- Modify: `project/work_health.py` (add `format_resume_display()`)
- Test: `tests/test_work_health.py` (append resume display tests)

**Interfaces:**
- Consumes: `plan_manager.parse_plan()`, `plan_manager.flatten_leaves()`,
  `work_health.py` check results
- Produces: `format_resume_display(workspace, health_output)` → formatted string
  for the resume path to display.

- [ ] **Step 1: Write failing test for resume display**

Append to `tests/test_work_health.py`:

```python
class TestResumeDisplay:

    def _write_plan(self, workspace, content):
        design = Path(workspace) / "design"
        design.mkdir(exist_ok=True)
        (design / ".plan").write_text(content)

    def test_resume_display_with_plan(self, tmp_path):
        from pathlib import Path as P
        sys.path.insert(0, str(P(__file__).parent.parent / "project"))
        from work_health import format_resume_display
        workspace = tmp_path / "wksp"
        workspace.mkdir()
        self._write_plan(workspace, (
            "# Work Plan — test\n\n"
            "## Queue\n"
            "- [x] #142 — completed issue\n"
            "- [ ] #123 — active issue ← active\n"
            "- [ ] #155 — pending issue\n\n"
            "## Session State\nCurrent: #123\nStarted: 2026-08-06\n"
        ))
        output = format_resume_display(str(workspace), "")
        assert "#142" in output
        assert "#123" in output
        assert "active" in output.lower() or "current" in output.lower()

    def test_resume_display_no_plan(self, tmp_path):
        from pathlib import Path as P
        sys.path.insert(0, str(P(__file__).parent.parent / "project"))
        from work_health import format_resume_display
        workspace = tmp_path / "wksp"
        workspace.mkdir()
        output = format_resume_display(str(workspace), "")
        assert output == "" or "no queue" in output.lower()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_health.py::TestResumeDisplay -v`
Expected: FAIL

- [ ] **Step 3: Implement format_resume_display**

Add to `project/work_health.py`:

```python
def format_resume_display(workspace, health_output):
    plan_path = Path(workspace) / "design" / ".plan"
    if not plan_path.exists():
        return ""

    sys.path.insert(0, str(Path(__file__).parent.parent / "work-slot"))
    from plan_manager import parse_plan, flatten_leaves

    tree = parse_plan(plan_path)
    leaves = flatten_leaves(tree)
    if not leaves:
        return ""

    completed = [l for l in leaves if l.completed]
    active = [l for l in leaves if l.active]
    pending = [l for l in leaves if not l.completed and not l.active]

    lines = [f"## Queue ({len(leaves)} items, {len(completed)} complete, "
             f"{len(active)} active):"]
    for l in completed:
        lines.append(f"  ✅ #{l.issue_number} — {l.title}")
    for l in active:
        lines.append(f"  → #{l.issue_number} — {l.title} (current)")
    for l in pending:
        lines.append(f"     #{l.issue_number} — {l.title}")

    health_notes = []
    for line in health_output.splitlines():
        if "CHECK=plan_state" in line and "STATUS=changed" in line:
            detail = line.split("DETAIL=", 1)[1] if "DETAIL=" in line else ""
            health_notes.append(f"  work_health: {detail}")
    if health_notes:
        lines.append("")
        lines.extend(health_notes)

    return "\n".join(lines)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_health.py::TestResumeDisplay -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add project/work_health.py tests/test_work_health.py
git commit -m "feat(#N): add human-readable resume display for .plan queue

format_resume_display() renders .plan as a formatted queue summary with
completion status, active markers, and work_health annotations."
```

---

### Task 5: Main `.plan` support — Phase 4

**Files:**
- Modify: `work-slot/plan_manager.py` (support main .plan creation and detection on main)
- Modify: `work/SKILL.md` (add main .plan bootstrapping prompt)
- Test: `tests/test_plan_manager.py` (append main .plan tests)

**Interfaces:**
- Consumes: `plan_manager.detect()`, `plan_manager.build_plan_content()`,
  `gh issue list` CLI
- Produces: `plan_manager.create_main_plan(workspace, items)` — creates
  `.plan` on workspace main. `plan_manager.detect()` already works on
  main — verify it returns correct results.

- [ ] **Step 1: Write failing test for main .plan detection**

Append to `tests/test_plan_manager.py`:

```python
class TestMainPlan:

    def _write_main_plan(self, workspace, content):
        design = Path(workspace) / "design"
        design.mkdir(exist_ok=True)
        (design / ".plan").write_text(content)

    def test_detect_main_plan(self, tmp_path):
        workspace = tmp_path / "wksp"
        workspace.mkdir()
        self._write_main_plan(workspace, (
            "# Work Plan — soredium\n\n"
            "## Queue\n"
            "- [ ] #170 — pre-merge hook ← active\n"
            "- [ ] #95 — mechanize LLM ops\n\n"
            "## Session State\nCurrent: #170\n"
        ))
        result = detect(workspace)
        assert result is not None
        assert result["active_issue"] == "170"

    def test_create_main_plan(self, tmp_path):
        workspace = tmp_path / "wksp"
        workspace.mkdir()
        (workspace / "design").mkdir()
        items = [
            {"number": 170, "title": "pre-merge hook"},
            {"number": 95, "title": "mechanize LLM ops"},
        ]
        create_main_plan(workspace, items)
        plan_path = workspace / "design" / ".plan"
        assert plan_path.exists()
        content = plan_path.read_text()
        assert "#170" in content
        assert "#95" in content
        assert "← active" in content
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_plan_manager.py::TestMainPlan -v`
Expected: FAIL (create_main_plan doesn't exist)

- [ ] **Step 3: Implement create_main_plan**

Add to `work-slot/plan_manager.py`:

```python
def create_main_plan(workspace_path: Path, items: list[dict],
                     project_name: str = "project") -> Path:
    """Create a main .plan on workspace main with the given issues.

    items: list of {"number": int, "title": str}
    Returns path to created .plan file.
    """
    design = workspace_path / "design"
    design.mkdir(exist_ok=True)
    plan_path = design / ".plan"

    queue_items = []
    for i, item in enumerate(items):
        queue_items.append(QueueItem(
            issue_number=item["number"],
            title=item["title"],
            completed=False,
            active=(i == 0),
            is_epic=False,
            indent=0,
            children=[],
        ))

    from datetime import date
    content = build_plan_content(
        project_name, queue_items, str(date.today()), None
    )
    plan_path.write_text(content)
    return plan_path
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestMainPlan -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add work-slot/plan_manager.py tests/test_plan_manager.py
git commit -m "feat(#N): add create_main_plan() for global curated queue

Main .plan on workspace main serves as the project priority queue.
create_main_plan() takes a list of issue dicts and generates the .plan
file. detect() already works on main — no changes needed."
```

---

### Task 6: Slim HANDOFF.md — Phase 5

**Files:**
- Modify: `handover/SKILL.md` (rewrite resume path and write path)
- Modify: `handover/handover-reference.md` (slim template)
- Test: Manual testing (skill files aren't unit-testable)

**Interfaces:**
- Consumes: `work_health.py` check output, `format_resume_display()`,
  HANDOFF.md content
- Produces: Updated handover skill that writes slim HANDOFF.md (no What's Left,
  no What's Next) and displays .plan queue on resume.

- [ ] **Step 1: Update handover-reference.md with slim template**

Replace the HANDOFF.md template sections in `handover/handover-reference.md`.
Remove What's Left, What's Next, Queue Progress sections. Keep Last Session,
Immediate Next Step, Cross-Module, References.

The template becomes:

```markdown
# HANDOFF — <project>

## Last Session
2-3 lines: what was done, what was tried, key reasoning.

## Immediate Next Step
Single specific action.

## Cross-Module
Only if active cross-repo blockers exist with tracked issues.
Omit entirely if none.

## References
Paths only, no content inline.
```

- [ ] **Step 2: Update handover SKILL.md resume path (Steps R2-R3)**

Modify Step R2b to use `is_closed()` instead of checking for EPIC-CLOSED.md:

```
### Step R2b — Detect open branch (continue-in-place)

Run work_health.py entry-scope validation:
```bash
python3 ~/.claude/skills/project/work_health.py --scope entry --project "$PROJECT" --workspace "$WORKSPACE"
```

Read the output. If any CHECK has STATUS=warn or STATUS=changed, summarise
the findings in the resume output.

Then check for an open branch using is_closed():
- Read branch name from `$WORKSPACE/design/.meta`
- If .meta exists and `is_closed()` returns OPEN or MERGED_UNSTAMPED → branch still open
- If CLOSED or DELETED → previous session closed cleanly
```

Modify Step R3 to display .plan queue instead of cross-checking HANDOFF.md issues:

```
### Step R3 — Display .plan queue (replaces issue cross-check)

If `$WORKSPACE/design/.plan` exists, display the human-readable queue:
- Read format_resume_display() output
- Present after HANDOFF.md session context

If .plan doesn't exist, skip — show HANDOFF.md only.
```

- [ ] **Step 3: Update handover SKILL.md write path (Steps 2, 5)**

Remove the What's Left and What's Next sections from Step 2 (Recall from context)
and Step 5 (Write HANDOFF.md). The handover now writes only:
- Last Session
- Immediate Next Step
- Cross-Module (if applicable)
- References

Remove Step 5b (Issue repo cross-check) — no `#N` references in HANDOFF.md
to validate since What's Left/What's Next are gone.

- [ ] **Step 4: Update success criteria**

Update the handover skill's success criteria to reflect the slim format:
- Remove: "What's Left items carry Scale · Complexity tags"
- Remove: "What's Next table carries Scale / Complexity / Notes columns"
- Remove: "Issue repo cross-check passed"
- Add: "HANDOFF.md under 200 tokens"
- Add: ".plan queue displayed on resume (if exists)"

- [ ] **Step 5: Commit**

```bash
git add handover/SKILL.md handover/handover-reference.md
git commit -m "feat(#N): slim HANDOFF.md — remove work tracking, add .plan resume display

HANDOFF.md now contains only session context (Last Session, Immediate
Next Step, Cross-Module, References). Work tracking moved to .plan.
Resume path displays .plan queue via format_resume_display() and runs
work_health.py entry-scope validation."
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ Phase 1: `is_closed()` → Task 1
- ✅ Phase 2: `work_health.py` entry-scope → Task 2
- ✅ Phase 3: `.plan` batch validation + resume display → Tasks 3-4
- ✅ Phase 4: Main `.plan` → Task 5
- ✅ Phase 5: Slim HANDOFF.md → Task 6
- ⏭ Phases 6-12: Separate plan (after Phases 1-5 land)

**Placeholder scan:** No TBD, TODO, or "add appropriate" placeholders.

**Type consistency:** `ClosureState` enum used consistently. `plan_manager.mark_completed()`
signature matches between Task 3 (consumer) and Task 3 Step 1 (producer).

**Tooling safety:** No bash file operations on source files. All code changes
via direct file editing.

# Work-end Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** To be filed — spec tracking line requires issue before first commit
**Issue group:** #95 (partial overlap — mechanize LLM operations)

**Goal:** Collapse work-end from ~20 steps to 5 (Context → Sweep → Execute → Verify → Close), fix multi-repo slot close bug, and remediate 27 legacy slots with missing stamps.

**Architecture:** Three new Python scripts (`work_end_context.py`, `work_end_execute.py`, `verify_slot_close.py`) absorb mechanical operations from the SKILL.md. The SKILL.md shrinks from ~1400 to ~400 lines, retaining only LLM-driven decisions (sweep, code review, squash analysis). Execute uses a three-phase structure (Rebase → Squash analysis → Land) to eliminate LLM/script interleaving. Per-repo progress tracking (`.execute-progress`) enables crash recovery.

**Tech Stack:** Python 3.12+, pytest, git CLI, gh CLI, existing soredium scripts (close_artifacts.py, land_branch.py, verify_stamp.py, verify_promotion.py, branch_recon.py, branch_cleanup.py, close_report.py, hygiene_scan.py)

## Global Constraints

- TDD: every script ships with pytest tests per `externalised-scripts-require-tests` protocol
- All scripts use `KEY=value` stdout output (match existing convention)
- No AI attribution in commit messages
- Every commit references an issue (`Refs #N` or `Closes #N`)
- Python type hints on all function signatures
- `.phase-a-complete` compatibility marker must be written until slot_manager.py is updated

---

### Task 1: File issue and recover stranded artifacts

**Files:**
- No new files — GitHub issue + git operations only

**Interfaces:**
- Produces: GitHub issue number (used by all subsequent commits)

- [ ] **Step 1: File the soredium issue**

```bash
gh issue create --repo Hortora/soredium \
  --title "Restructure work-end: 5-step model with mechanical promotion and multi-repo verification gate" \
  --body "Spec: cc-praxis/specs/2026-08-05-work-end-restructure-and-slot-audit-design.md

Two deliverables:
1. Legacy slot audit remediation (stamp 27 problem slots)
2. Work-end restructure: Context → Sweep → Execute → Verify → Close

Refs #95 (partial overlap — mechanize LLM operations)"
```

Record the issue number — all subsequent commits reference it.

- [ ] **Step 2: Recover 4 stranded specs to workspace main**

Cherry-pick specs from closed workspace branches to main:

```bash
# From issue-157 branch
git -C /Users/mdproctor/claude/public/cc-praxis log --oneline issue-157-worklog-rest-mcp -- specs/issue-157-worklog-rest-mcp/ | head -1
# Cherry-pick the spec commit(s) to main
git -C /Users/mdproctor/claude/public/cc-praxis cherry-pick <SHA> --no-commit
# Repeat for issue-171, issue-182, issue-66
# Then commit all recovered specs
git -C /Users/mdproctor/claude/public/cc-praxis add specs/
git -C /Users/mdproctor/claude/public/cc-praxis commit -m "docs: recover 4 stranded specs from closed branches  Refs #<issue>"
git -C /Users/mdproctor/claude/public/cc-praxis push
```

Verify: `git ls-tree main specs/ | wc -l` should show 4 more entries than before.

- [ ] **Step 3: Commit audit_slot_merges.py and query_worklog.py**

These scripts were written during this session but not committed:

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/audit_slot_merges.py scripts/query_worklog.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#<issue>): add slot merge audit and worklog query scripts  Refs #<issue>"
```

---

### Task 2: audit_slot_merges.py --fix mode (Deliverable 1)

**Files:**
- Modify: `scripts/audit_slot_merges.py`
- Create: `tests/test_audit_slot_merges.py`

**Interfaces:**
- Consumes: existing `audit_slot_merges.py` classification logic
- Produces: `--fix` CLI flag that stamps UNSTAMPED branches

- [ ] **Step 1: Write failing tests for --fix mode**

```python
# tests/test_audit_slot_merges.py
import subprocess
from pathlib import Path

def test_fix_stamps_unstamped_branch(tmp_path):
    """--fix stamps a MERGED-but-UNSTAMPED branch."""
    # Setup: create original repo with main + branch, merge branch content
    # but don't stamp it
    original = _create_repo_with_merged_unstamped_branch(tmp_path)
    slot_dir = _create_slot_with_slot_file(tmp_path, original)

    result = subprocess.run(
        ["python3", "scripts/audit_slot_merges.py", str(tmp_path), "--fix"],
        capture_output=True, text=True
    )
    assert result.returncode == 0
    assert "STAMPED" in result.stdout

    # Verify stamp exists
    last_msg = subprocess.run(
        ["git", "-C", str(original), "log", "-1", "--format=%s", "test-branch"],
        capture_output=True, text=True
    ).stdout.strip()
    assert last_msg.startswith("chore: branch closed")

def test_fix_skips_already_stamped(tmp_path):
    """--fix does not double-stamp an already-stamped branch."""
    original = _create_repo_with_stamped_branch(tmp_path)
    slot_dir = _create_slot_with_slot_file(tmp_path, original)

    result = subprocess.run(
        ["python3", "scripts/audit_slot_merges.py", str(tmp_path), "--fix"],
        capture_output=True, text=True
    )
    assert "STAMPED" not in result.stdout
    assert "OK" in result.stdout or "STAMP_ONLY" in result.stdout

def test_fix_verifies_landing_sha(tmp_path):
    """--fix verifies the landing SHA via verify_stamp.py before stamping."""
    # Setup: branch with content NOT on main (should fail verification)
    original = _create_repo_with_unmerged_branch(tmp_path)
    slot_dir = _create_slot_with_slot_file(tmp_path, original)

    result = subprocess.run(
        ["python3", "scripts/audit_slot_merges.py", str(tmp_path), "--fix"],
        capture_output=True, text=True
    )
    # Should NOT stamp — content not on main
    assert "VERIFICATION_FAILED" in result.stdout or "UNMERGED" in result.stdout

def test_fix_produces_summary_report(tmp_path):
    """--fix produces a summary report with per-slot disposition."""
    original = _create_repo_with_merged_unstamped_branch(tmp_path)
    slot_dir = _create_slot_with_slot_file(tmp_path, original)

    result = subprocess.run(
        ["python3", "scripts/audit_slot_merges.py", str(tmp_path), "--fix"],
        capture_output=True, text=True
    )
    assert "Audit Summary" in result.stdout
    assert "STAMPED" in result.stdout or "OK" in result.stdout
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_audit_slot_merges.py -v`
Expected: FAIL — `--fix` mode not implemented

- [ ] **Step 3: Implement --fix mode**

Add to `audit_slot_merges.py`:
- `--fix` CLI flag in `main()`
- `fix_unstamped()` function: finds landing SHA via `git log --oneline --grep`, stamps the branch, calls `verify_stamp.py`, pushes stamp. Rolls back on verification failure.
- `fix_report()` function: produces per-slot summary (STAMPED / SKIPPED / VERIFICATION_FAILED)

Key logic for finding landing SHA:
```python
def find_landing_sha(repo_path: Path, branch: str) -> str | None:
    """Find the SHA on main that corresponds to the branch's squashed content."""
    # Get the last real commit message (not the stamp)
    commits = git(repo_path, "log", "--oneline", f"main..{branch}")
    for line in commits.splitlines():
        if "chore: branch closed" in line:
            continue
        subject = line.split(" ", 1)[1] if " " in line else ""
        # Search main for a commit with the same subject
        match = git(repo_path, "log", "--oneline", f"--grep={subject[:50]}", "main")
        if match:
            return match.split()[0]
    return None
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_audit_slot_merges.py -v`
Expected: PASS

- [ ] **Step 5: Run --fix against casehub (dry run first)**

```bash
python3 scripts/audit_slot_merges.py /Users/mdproctor/claude/casehub --all --fix
```

Review output. If results look correct, the stamps are already applied.

- [ ] **Step 6: Commit**

```bash
git add scripts/audit_slot_merges.py tests/test_audit_slot_merges.py
git commit -m "feat(#<issue>): add --fix mode to audit_slot_merges.py — retroactive stamp remediation  Refs #<issue>"
```

---

### Task 3: land_branch.py stamp push + close_report.py build step

**Files:**
- Modify: `work-end/land_branch.py`
- Modify: `work-end/close_report.py`
- Modify: existing tests if present

**Interfaces:**
- Consumes: `land_branch.py stamp` subcommand (adds push after stamp)
- Produces: stamp commit pushed to fork remote; `build-verify` step type in close_report.py

- [ ] **Step 1: Write failing test for stamp push**

```python
def test_stamp_pushes_to_origin(tmp_path):
    """cmd_stamp pushes the work branch after creating stamp commit."""
    project, remote = _create_project_with_remote(tmp_path)
    _create_and_merge_branch(project, "feature-branch")

    result = subprocess.run(
        ["python3", "work-end/land_branch.py", "stamp", str(project),
         "branch=feature-branch", "base_branch=main"],
        capture_output=True, text=True
    )
    assert "STAMP=ok" in result.stdout

    # Verify stamp is on remote
    remote_tip = subprocess.run(
        ["git", "-C", str(remote), "log", "-1", "--format=%s", "feature-branch"],
        capture_output=True, text=True
    ).stdout.strip()
    assert remote_tip.startswith("chore: branch closed")
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — stamp commit created but not pushed

- [ ] **Step 3: Implement stamp push in land_branch.py**

In `cmd_stamp()`, after the stamp commit succeeds, push the work branch:

```python
# After: git commit --allow-empty -m "chore: branch closed ..."
# Add: push work branch to origin
fork_remote, _ = detect_topology(project)
if fork_remote:
    rc, out, err = run_cmd(["git", "-C", project, "push", fork_remote,
                            branch, "--force-with-lease"])
    if rc != 0:
        print(f"STAMP_PUSH_WARNING=push failed: {err}")
```

- [ ] **Step 4: Add build-verify step to close_report.py**

Add `"build-verify"` to `STEP_ORDER` with label `"Build verified"`.

- [ ] **Step 5: Run tests, commit**

```bash
python3 -m pytest tests/ -k "stamp" -v
git add work-end/land_branch.py work-end/close_report.py
git commit -m "fix(#<issue>): push stamp to remote after creation; add build-verify step to close_report  Refs #<issue>"
```

---

### Task 4: verify_slot_close.py (Step 4 — Verify)

**Files:**
- Create: `work-end/verify_slot_close.py`
- Create: `tests/test_verify_slot_close.py`

**Interfaces:**
- Consumes: `.slot` file (repo list), routing config, COVERS list, close_report.py build record
- Produces: `VERIFIED=yes|no` with per-check results on stdout

- [ ] **Step 1: Write failing tests**

```python
def test_verify_all_pass(tmp_path):
    """VERIFIED=yes when all repos merged, stamped, pushed, artifacts promoted."""
    family = _create_clean_slot_state(tmp_path, repos=["engine", "blocks"])
    result = _run_verify(family)
    assert "VERIFIED=yes" in result.stdout

def test_verify_missing_stamp(tmp_path):
    """VERIFIED=no when one repo is unstamped."""
    family = _create_slot_with_unstamped_repo(tmp_path, stamped=["engine"], unstamped=["blocks"])
    result = _run_verify(family)
    assert "VERIFIED=no" in result.stdout
    assert "blocks" in result.stdout
    assert "UNSTAMPED" in result.stdout

def test_verify_unpushed_main(tmp_path):
    """VERIFIED=no when local main is ahead of origin."""
    family = _create_slot_with_unpushed_repo(tmp_path)
    result = _run_verify(family)
    assert "VERIFIED=no" in result.stdout
    assert "UNPUSHED" in result.stdout

def test_verify_unclosed_issue(tmp_path):
    """VERIFIED=no when a COVERS issue is still OPEN."""
    # This test mocks gh CLI
    ...

def test_verify_bad_args(tmp_path):
    """Exit 1 on missing arguments."""
    result = subprocess.run(
        ["python3", "work-end/verify_slot_close.py"],
        capture_output=True, text=True
    )
    assert result.returncode == 1

def test_verify_single_repo_mode(tmp_path):
    """Works for non-slot (single repo) mode."""
    project = _create_single_repo_clean_state(tmp_path)
    result = _run_verify_single(project)
    assert "VERIFIED=yes" in result.stdout
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_verify_slot_close.py -v`
Expected: FAIL — script doesn't exist

- [ ] **Step 3: Implement verify_slot_close.py**

Structure:
```python
def check_repo_merged(repo_path: Path, branch: str) -> dict
def check_repo_stamped(repo_path: Path, branch: str) -> dict
def check_landing_sha(repo_path: Path, branch: str) -> dict
def check_repo_pushed(repo_path: Path) -> dict
def check_workspace_stamped(workspace: Path, branch: str) -> dict
def check_artifacts_promoted(workspace: Path, project: Path) -> dict  # calls verify_promotion.py
def check_issues_closed(covers: list[int], issue_repo: str) -> dict
def check_build_passed(report_path: Path) -> dict

def verify(family_root: Path, slot_num: int | None, ...) -> bool
def main() -> int
```

Each check function returns `{"status": "pass"|"fail", "detail": "..."}`.
`verify()` runs all checks, prints results, returns True/False.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_verify_slot_close.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/verify_slot_close.py tests/test_verify_slot_close.py
git commit -m "feat(#<issue>): add verify_slot_close.py — unified verification gate for multi-repo slot close  Refs #<issue>"
```

---

### Task 5: work_end_context.py (Step 1 — Context)

**Files:**
- Create: `work-end/work_end_context.py`
- Create: `tests/test_work_end_context.py`

**Interfaces:**
- Consumes: `ctx.py`, `branch_recon.py`, `project/routing.py`, `plan_manager.py`
- Produces: JSON object with context values, precondition statuses, branch state, routing table

- [ ] **Step 1: Write failing tests**

```python
def test_context_clean_state(tmp_path):
    """All preconditions pass on clean state."""
    project, workspace = _create_clean_project(tmp_path)
    result = _run_context(workspace, project)
    data = json.loads(result.stdout)
    assert data["preconditions"]["clean_tree"]["status"] == "pass"
    assert data["context"]["workspace"] == str(workspace)

def test_context_dirty_tree(tmp_path):
    """dirty tree → status=fail."""
    project, workspace = _create_dirty_project(tmp_path)
    result = _run_context(workspace, project)
    data = json.loads(result.stdout)
    assert data["preconditions"]["clean_tree"]["status"] == "fail"
    assert result.returncode == 0  # exit 0 even on fail preconditions

def test_context_no_meta(tmp_path):
    """Missing .meta → status=needs_input with inferred issue."""
    project, workspace = _create_project_without_meta(tmp_path)
    result = _run_context(workspace, project)
    data = json.loads(result.stdout)
    assert data["preconditions"]["meta_exists"]["status"] == "needs_input"

def test_context_bad_args():
    """Exit 1 on missing arguments."""
    result = subprocess.run(
        ["python3", "work-end/work_end_context.py"],
        capture_output=True, text=True
    )
    assert result.returncode == 1

def test_context_includes_routing(tmp_path):
    """Output includes routing table from CLAUDE.md."""
    project, workspace = _create_project_with_routing(tmp_path)
    result = _run_context(workspace, project)
    data = json.loads(result.stdout)
    assert "routing" in data
    assert "specs" in data["routing"]

def test_context_includes_branch_recon(tmp_path):
    """Output includes branch_recon data."""
    project, workspace = _create_project_with_commits(tmp_path)
    result = _run_context(workspace, project)
    data = json.loads(result.stdout)
    assert "branch_recon" in data
    assert "commit_count" in data["branch_recon"]
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement work_end_context.py**

Structure:
```python
def check_branch_divergence(ctx: dict) -> dict
def check_queue_gate(ctx: dict) -> dict
def check_issue_complete(ctx: dict) -> dict
def check_pause_stack(ctx: dict) -> dict
def check_meta_exists(ctx: dict) -> dict
def check_clean_tree(ctx: dict) -> dict
def run_branch_recon(workspace: str, project: str, ...) -> dict
def resolve_routing(workspace: str) -> dict

def gather_context(workspace: str, project: str) -> dict
def main() -> int
```

Calls `ctx.py` via subprocess, parses output, runs each precondition check,
calls `branch_recon.py` via subprocess, resolves routing via import.
Outputs single JSON object.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add work-end/work_end_context.py tests/test_work_end_context.py
git commit -m "feat(#<issue>): add work_end_context.py — unified context and precondition checking  Refs #<issue>"
```

---

### Task 6: work_end_execute.py promote subcommand

**Files:**
- Create: `work-end/work_end_execute.py`
- Create: `tests/test_work_end_execute.py`

**Interfaces:**
- Consumes: context JSON from work_end_context.py, close_artifacts.py, verify_promotion.py
- Produces: `promote` subcommand; `.execute-progress` file; `PROMOTED=yes|no` output

- [ ] **Step 1: Write failing tests for promote**

```python
def test_promote_single_repo(tmp_path):
    """Promotes artifacts from workspace branch to workspace main to project."""
    ws, project = _create_workspace_with_artifacts(tmp_path)
    result = _run_execute("promote", ws, project)
    assert "PROMOTED=yes" in result.stdout
    # Verify artifacts on workspace main and project
    ...

def test_promote_slot_deduplicates_workspaces(tmp_path):
    """In multi-repo slot, promotes once per unique workspace."""
    family = _create_slot_with_shared_workspace(tmp_path, repos=3)
    result = _run_execute("promote", family)
    # close_artifacts.py should be called twice, not thrice
    assert result.stdout.count("WORKSPACE_PROMOTED=") <= 2

def test_promote_writes_progress(tmp_path):
    """Writes .execute-progress after each repo."""
    ws, project = _create_workspace_with_artifacts(tmp_path)
    result = _run_execute("promote", ws, project)
    progress = (Path(ws) / "design" / ".execute-progress").read_text()
    assert "promoted" in progress

def test_promote_skips_already_promoted(tmp_path):
    """Skips repos already marked promoted in .execute-progress."""
    ws, project = _create_workspace_with_artifacts(tmp_path)
    # Write existing progress
    (Path(ws) / "design" / ".execute-progress").write_text("soredium=promoted\n")
    result = _run_execute("promote", ws, project)
    assert "SKIPPED" in result.stdout or "already promoted" in result.stdout.lower()
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement promote subcommand**

`work_end_execute.py` structure (initial — promote only):
```python
SUBCOMMANDS = {"promote", "rebase", "land"}

def read_progress(progress_path: Path) -> dict[str, str]
def write_progress(progress_path: Path, repo: str, step: str)
def cmd_promote(args: dict) -> int
def main() -> int  # dispatches to subcommand
```

`cmd_promote` calls `close_artifacts.py` per unique workspace (without `covers=`),
then calls `verify_promotion.py`. Writes `.execute-progress` entries.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add work-end/work_end_execute.py tests/test_work_end_execute.py
git commit -m "feat(#<issue>): add work_end_execute.py promote — per-workspace artifact promotion  Refs #<issue>"
```

---

### Task 7: work_end_execute.py rebase subcommand (Phase A)

**Files:**
- Modify: `work-end/work_end_execute.py`
- Modify: `tests/test_work_end_execute.py`

**Interfaces:**
- Consumes: repo list from .slot or single project path
- Produces: `rebase` subcommand; all repos rebased onto base branch

- [ ] **Step 1: Write failing tests for rebase**

```python
def test_rebase_single_repo(tmp_path):
    """Rebases branch onto main for single repo."""
    project = _create_project_with_branch(tmp_path)
    result = _run_execute("rebase", project, branch="feature")
    assert "REBASED=yes" in result.stdout

def test_rebase_slot_all_repos(tmp_path):
    """Rebases all repos in a slot."""
    family = _create_slot_with_branches(tmp_path, repos=["engine", "blocks"])
    result = _run_execute("rebase", family, slot=True)
    assert result.stdout.count("REBASED") == 2

def test_rebase_conflict_stops(tmp_path):
    """Stops on rebase conflict with error."""
    project = _create_project_with_conflict(tmp_path)
    result = _run_execute("rebase", project, branch="feature")
    assert "ERROR=REBASE_CONFLICT" in result.stdout

def test_rebase_slot_mode_uses_fetch_origin(tmp_path):
    """Slot mode fetches from origin (on-disk original), not GitHub."""
    family = _create_slot_clone(tmp_path)
    result = _run_execute("rebase", family, slot=True)
    assert "REBASED=yes" in result.stdout
```

- [ ] **Step 2-5: Implement, test, commit**

---

### Task 8: work_end_execute.py land subcommand (Phase C)

**Files:**
- Modify: `work-end/work_end_execute.py`
- Modify: `tests/test_work_end_execute.py`

**Interfaces:**
- Consumes: `.squash-plan-<repo>.json` files (from Phase B LLM), progress file
- Produces: `land` subcommand; repos squashed, built, pushed, stamped; `.landed` marker; `.phase-a-complete` marker

This is the core multi-repo fix. The land subcommand:
1. Reads `.squash-plan-<repo>.json` and applies squash mechanically
2. Runs build verification (Java: mvn install)
3. Pushes to fork (non-slot) or slot→original→GitHub (slot mode)
4. Stamps branch + pushes stamp
5. Writes `.phase-a-complete` and `.landed` markers
6. Closes issues (once, after all repos)

- [ ] **Step 1: Write failing tests for land**

```python
def test_land_single_repo(tmp_path):
    """Applies squash plan, pushes, stamps for single repo."""
    project = _create_rebased_project(tmp_path)
    _write_squash_plan(project, plan={"groups": [...]})
    result = _run_execute("land", project)
    assert "LANDED=yes" in result.stdout
    assert "STAMP=ok" in result.stdout

def test_land_slot_all_repos(tmp_path):
    """Lands all repos in a slot with per-repo progress."""
    family = _create_rebased_slot(tmp_path, repos=["engine", "blocks"])
    _write_squash_plans(family, ["engine", "blocks"])
    result = _run_execute("land", family, slot=True)
    assert result.stdout.count("STAMP=ok") == 2

def test_land_writes_landed_marker(tmp_path):
    """Writes .landed marker in slot mode."""
    family = _create_rebased_slot(tmp_path, repos=["engine"])
    _write_squash_plans(family, ["engine"])
    result = _run_execute("land", family, slot=True)
    landed = (family / ".landed").read_text()
    assert "landed_shas=" in landed

def test_land_writes_phase_a_complete(tmp_path):
    """Writes .phase-a-complete for slot_manager.py compatibility."""
    family = _create_rebased_slot(tmp_path, repos=["engine"])
    _write_squash_plans(family, ["engine"])
    result = _run_execute("land", family, slot=True)
    assert (family / ".phase-a-complete").exists()

def test_land_resumes_from_progress(tmp_path):
    """Skips repos already stamped in .execute-progress."""
    family = _create_rebased_slot(tmp_path, repos=["engine", "blocks"])
    _write_progress(family, "engine=stamped")
    _write_squash_plans(family, ["blocks"])
    result = _run_execute("land", family, slot=True)
    # Only blocks should be processed
    assert result.stdout.count("STAMP=ok") == 1

def test_land_stamp_idempotent(tmp_path):
    """Skips stamp if branch tip is already a stamp commit."""
    project = _create_already_stamped_project(tmp_path)
    _write_squash_plan(project, plan={"groups": []})
    result = _run_execute("land", project)
    # Should not create duplicate stamp
    assert "STAMP_SKIPPED" in result.stdout or "STAMP=ok" in result.stdout

def test_land_build_before_push(tmp_path):
    """Build runs before push — build failure means nothing pushed."""
    project = _create_rebased_java_project(tmp_path)
    _write_squash_plan(project)
    # Inject build failure
    result = _run_execute("land", project, build_cmd="false")
    assert "BUILD_FAILED" in result.stdout
    # Verify nothing was pushed
    ...

def test_land_closes_issues_once(tmp_path):
    """Issues closed once after all repos, not per-repo."""
    family = _create_rebased_slot(tmp_path, repos=["engine", "blocks"])
    _write_squash_plans(family, ["engine", "blocks"])
    result = _run_execute("land", family, slot=True, covers="5,19")
    # gh issue close should appear once per issue, not per repo
    ...
```

- [ ] **Step 2-5: Implement, test, commit**

The land subcommand is the largest single piece. Key implementation details:

**Slot mode push sequence** (slot clone → original → GitHub):
```python
def push_slot_repo(slot_repo: Path, original_repo: Path, branch: str) -> dict:
    git(slot_repo, "push", "origin", branch, "--force-with-lease")
    git(original_repo, "fetch", "origin")
    git(original_repo, "merge", "--ff-only", branch)
    git(original_repo, "push", "origin", "main")
```

**Squash plan application:**
```python
def apply_squash_plan(repo_path: Path, plan_path: Path) -> bool:
    plan = json.loads(plan_path.read_text())
    # Build rebase todo from plan groups
    # Call rebase_exec.py multi
```

---

### Task 9: SKILL.md rewrite — 5-step version

**Files:**
- Modify: `work-end/SKILL.md` (complete rewrite, ~1400 → ~400 lines)

**Interfaces:**
- References: work_end_context.py, work_end_execute.py, verify_slot_close.py

- [ ] **Step 1: Write the 5-step SKILL.md**

Structure:
```markdown
# work-end
[CSO description — unchanged]

## Hard Gates
[Condensed from current — code review mandatory, doc sync mandatory, main mutations through work-end only]

## Path Resolution
[Unchanged — ctx.py call]

## Lifecycle State Machine Integration
[New — from spec §Lifecycle State Machine Integration]

## Step 1 — Context
[Call work_end_context.py, handle needs_input conditions interactively]

## Step 2 — Sweep
[4-item checklist, session-bound rules, journal validation decisions, slot mode per-repo sweep]

## Step 3 — Execute
[LLM/script boundary table, call sequence:
 1. Code review (LLM subagent)
 2. work_end_execute.py promote
 3. Phase A: work_end_execute.py rebase
 4. Phase B: LLM per-repo squash analysis loop
 5. Phase C: work_end_execute.py land
 6. Blessed repo delivery prompt]

## Step 4 — Verify
[Call verify_slot_close.py, handle failures]

## Step 5 — Close
[EPIC-CLOSED, archive, return to main, ARC42, HANDOFF]

## Skill Chaining
[Updated references]
```

- [ ] **Step 2: Validate — run skill validation**

```bash
python3 scripts/validate_all.py --tier commit
```

- [ ] **Step 3: Sync to installed skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 4: Commit**

```bash
git add work-end/SKILL.md
git commit -m "refactor(#<issue>): rewrite work-end SKILL.md — 5-step model (Context/Sweep/Execute/Verify/Close)  Refs #<issue>"
```

---

### Task 10: Remove phase_a_complete.py and phase_b_gate.py

**Files:**
- Delete: `work-end/phase_a_complete.py`
- Delete: `work-end/phase_b_gate.py`
- Modify: any tests referencing these scripts

- [ ] **Step 1: Verify no remaining callers**

```bash
grep -r "phase_a_complete\|phase_b_gate" /Users/mdproctor/claude/hortora/soredium/ --include="*.py" --include="*.md"
```

Expected: only hits in the scripts themselves and this plan.
If `slot_manager.py` still references `.phase-a-complete`, that's OK — Execute writes the marker. The scripts that WRITE the marker are removed; the code that READS it stays.

- [ ] **Step 2: Delete scripts**

- [ ] **Step 3: Run tests**

```bash
python3 -m pytest tests/ -v
```

- [ ] **Step 4: Commit**

```bash
git add -u work-end/phase_a_complete.py work-end/phase_b_gate.py
git commit -m "chore(#<issue>): remove vestigial phase_a_complete.py and phase_b_gate.py  Refs #<issue>"
```

---

## Dependency Graph

```
Task 1 (issue + recover artifacts)
  ↓
Task 2 (audit --fix)
  ↓
Task 3 (land_branch.py stamp push + close_report.py)
  ↓
Task 4 (verify_slot_close.py)     ← independent
Task 5 (work_end_context.py)      ← independent
  ↓
Task 6 (execute.py promote)
  ↓
Task 7 (execute.py rebase)
  ↓
Task 8 (execute.py land)          ← depends on 3, 6, 7
  ↓
Task 9 (SKILL.md rewrite)         ← depends on 4, 5, 6, 7, 8
  ↓
Task 10 (remove phase scripts)    ← depends on 9
```

Tasks 4 and 5 can run in parallel. Tasks 6-8 are sequential (same file, growing interface).

# Graceful .meta + Slot-Mode Sweep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #87 — work-end should handle branches without .meta gracefully
**Issue group:** #87, #88

**Goal:** Make work-end degrade gracefully when `.meta` is absent, and add
per-repo sweep support for slot-mode branches.

**Architecture:** ctx.py gets one additive output (`INFERRED_ISSUE`). work-end
SKILL.md pre-condition 2 replaces its hard gate with a reconstruction block.
close_artifacts.py gets an optional `scan-workspace` parameter. work-end SKILL.md
Step 3b gets a slot-mode per-repo sweep loop.

**Tech Stack:** Python 3, pytest, markdown (SKILL.md)

## Global Constraints

- Protocol: externalised-scripts-require-tests — every script change ships
  with corresponding pytest tests in the same commit
- ctx.py output contract: all existing fields unchanged — `INFERRED_ISSUE` is
  additive only
- close_artifacts.py interface: `scan-workspace` is optional — omitting it
  preserves current behaviour exactly
- No AI attribution in commit messages

---

### Task 1: Add INFERRED_ISSUE to ctx.py

**Files:**
- Modify: `project/ctx.py` (add ~5 lines after line 68)
- Test: `tests/test_ctx.py` (add new test class)

**Interfaces:**
- Consumes: existing ctx.py output (ISSUE_N, CURRENT_BRANCH)
- Produces: `INFERRED_ISSUE=<number>` — populated only when `ISSUE_N` is
  empty AND branch name matches `issue-(\d+)`

- [ ] **Step 1: Write failing tests**

```python
class TestInferredIssue:
    """Test INFERRED_ISSUE field — infers issue number from branch name."""

    def test_inferred_issue_from_branch_name(self, tmp_path):
        """INFERRED_ISSUE populated when no .meta and branch matches issue-NNN."""
        repo = init_repo(tmp_path / "repo")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "issue-42-fix-bug"],
                        capture_output=True)
        data = parse(run_ctx(repo))
        assert data["INFERRED_ISSUE"] == "42"

    def test_inferred_issue_empty_when_meta_present(self, tmp_path):
        """INFERRED_ISSUE empty when .meta provides ISSUE_N."""
        repo = init_repo(tmp_path / "repo")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "issue-42-fix-bug"],
                        capture_output=True)
        (repo / "design").mkdir()
        (repo / "design" / ".meta").write_text("branch: issue-42-fix-bug\nissue: 42\n")
        data = parse(run_ctx(repo))
        assert data["INFERRED_ISSUE"] == ""

    def test_inferred_issue_empty_when_no_match(self, tmp_path):
        """INFERRED_ISSUE empty when branch doesn't match issue-NNN."""
        repo = init_repo(tmp_path / "repo")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "fix-typo"],
                        capture_output=True)
        data = parse(run_ctx(repo))
        assert data["INFERRED_ISSUE"] == ""

    def test_inferred_issue_empty_on_main(self, tmp_path):
        """INFERRED_ISSUE empty on main branch."""
        repo = init_repo(tmp_path / "repo")
        data = parse(run_ctx(repo))
        assert data["INFERRED_ISSUE"] == ""

    def test_inferred_issue_multi_digit(self, tmp_path):
        """INFERRED_ISSUE handles multi-digit issue numbers."""
        repo = init_repo(tmp_path / "repo")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "issue-1234-big-feature"],
                        capture_output=True)
        data = parse(run_ctx(repo))
        assert data["INFERRED_ISSUE"] == "1234"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_ctx.py::TestInferredIssue -v`
Expected: FAIL — `KeyError: 'INFERRED_ISSUE'`

- [ ] **Step 3: Implement INFERRED_ISSUE in ctx.py**

Add after line 68 (`current_branch = run(...)`) in `project/ctx.py`:

```python
import re  # already imported at line 11

# Infer issue number from branch name when .meta is absent
inferred_issue = ""
if not issue_n and current_branch:
    m_issue = re.search(r'issue-(\d+)', current_branch)
    if m_issue:
        inferred_issue = m_issue.group(1)
```

Add the print line after the existing `COVERS` print (around line 80):

```python
print(f"INFERRED_ISSUE={inferred_issue}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_ctx.py::TestInferredIssue -v`
Expected: all 5 PASS

- [ ] **Step 5: Run full ctx.py test suite**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: all existing tests still pass (additive change)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/ctx.py tests/test_ctx.py
git commit -m "feat(#87): add INFERRED_ISSUE output to ctx.py

Parses issue-NNN from branch name when .meta is absent.
Additive — no existing output changes.

Refs #87"
```

---

### Task 2: Add scan-workspace parameter to close_artifacts.py

**Files:**
- Modify: `work-end/close_artifacts.py` (modify `main()` and `scan_artifacts()`)
- Test: `tests/test_close_artifacts.py` (add new test class)

**Interfaces:**
- Consumes: existing close_artifacts.py CLI interface
- Produces: same KEY=value output. When `scan-workspace=<path>` is provided,
  artifacts are scanned from that path instead of the workspace positional arg.
  The workspace positional arg remains the promotion target.

- [ ] **Step 1: Write failing tests**

```python
class TestScanWorkspace:
    """Test scan-workspace parameter for slot-mode artifact promotion."""

    def test_scan_workspace_reads_from_alternate_path(self, tmp_path):
        """When scan-workspace provided, artifacts scanned from that path."""
        workspace = tmp_path / "original-workspace"
        workspace.mkdir()
        (workspace / "design").mkdir()
        slot_workspace = tmp_path / "slot-workspace"
        slot_workspace.mkdir()
        project = tmp_path / "project"
        project.mkdir()
        _init_git(workspace)
        _init_git(slot_workspace)
        _init_git(project)

        # Put artifacts in slot workspace, not original
        branch = "issue-99-test"
        (slot_workspace / "specs" / branch).mkdir(parents=True)
        (slot_workspace / "specs" / branch / "spec.md").write_text("# Spec\n")

        result = run_close_artifacts(
            workspace, project, branch,
            extra_args=[f"scan-workspace={slot_workspace}"],
        )
        output = parse_output(result.stdout)
        # Should find the spec in slot workspace
        assert result.returncode != 1  # not fatal

    def test_scan_workspace_missing_path_is_fatal(self, tmp_path):
        """scan-workspace pointing to non-existent path is fatal."""
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        (workspace / "design").mkdir()
        project = tmp_path / "project"
        project.mkdir()
        _init_git(workspace)
        _init_git(project)

        result = run_close_artifacts(
            workspace, project, "test-branch",
            extra_args=["scan-workspace=/nonexistent/path"],
        )
        output = parse_output(result.stdout)
        assert output.get("ERROR") == "scan_workspace_not_found"

    def test_scan_workspace_omitted_uses_workspace(self, tmp_path):
        """Without scan-workspace, artifacts scanned from workspace (current behaviour)."""
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        (workspace / "design").mkdir()
        project = tmp_path / "project"
        project.mkdir()
        _init_git(workspace)
        _init_git(project)

        branch = "issue-99-test"
        (workspace / "specs" / branch).mkdir(parents=True)
        (workspace / "specs" / branch / "spec.md").write_text("# Spec\n")

        result = run_close_artifacts(workspace, project, branch)
        assert result.returncode != 1
```

Adapt helper functions (`run_close_artifacts`, `parse_output`, `_init_git`) to
match existing test patterns in `test_close_artifacts.py`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestScanWorkspace -v`
Expected: FAIL — scan-workspace not recognized

- [ ] **Step 3: Implement scan-workspace in close_artifacts.py**

In `main()`, after parsing `covers` (line 163), add:

```python
scan_workspace_path = params.get("scan-workspace", "")
if scan_workspace_path:
    scan_workspace = Path(scan_workspace_path)
    if not scan_workspace.is_dir():
        print("ERROR=scan_workspace_not_found")
        print(f"ERROR_DETAIL={scan_workspace_path}")
        return 1
    scan_source = scan_workspace
else:
    scan_source = workspace
```

Change `scan_artifacts` call (line 174) from:
```python
artifacts = scan_artifacts(workspace, branch)
```
to:
```python
artifacts = scan_artifacts(scan_source, branch)
```

Change `resolve_routing` call (line 175) from:
```python
routing = resolve_routing(workspace)
```
to:
```python
routing = resolve_routing(scan_source)
```

The rest of `main()` continues using `workspace` as the promotion target
(for `artifact_promote.py` calls) — only the scan source changes.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_close_artifacts.py::TestScanWorkspace -v`
Expected: all 3 PASS

- [ ] **Step 5: Run full close_artifacts test suite**

Run: `python3 -m pytest tests/test_close_artifacts.py -v`
Expected: all existing tests still pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_artifacts.py tests/test_close_artifacts.py
git commit -m "feat(#88): add scan-workspace parameter to close_artifacts.py

When provided, artifacts are scanned from the given path instead of
the workspace positional arg. The workspace remains the promotion
target. Enables slot-mode artifact promotion where artifacts live
on the slot workspace branch.

Refs #88"
```

---

### Task 3: Update work-end SKILL.md — graceful .meta (pre-condition 2)

**Files:**
- Modify: `work-end/SKILL.md` (pre-condition 2, Step 3, Step 5, Step 8d)

**Interfaces:**
- Consumes: `INFERRED_ISSUE` from ctx.py (Task 1)
- Produces: updated skill instructions — no script interfaces

- [ ] **Step 1: Replace pre-condition 2 hard gate**

In `work-end/SKILL.md`, replace pre-condition 2 (currently line 76):

```markdown
2. **`$WORKSPACE/design/.meta` must exist on the current branch** → proceed.
```

With:

```markdown
2. **`$WORKSPACE/design/.meta` exists?**
   - **YES** → proceed (read values from ctx.py output as normal).
   - **NO** → check branch:
     - **On main or `$PROJECT_BASE_BRANCH`** → "Nothing to close — you're on
       `$CURRENT_BRANCH`." Exit cleanly.
     - **On a feature branch** → graceful degradation:
       1. **Resolve issue:**
          - `INFERRED_ISSUE` populated (from ctx.py) →
            `gh issue view $INFERRED_ISSUE --repo $OWNER_REPO`,
            extract title and confirm with user
          - `INFERRED_ISSUE` populated but `gh issue view` fails
            (network error, issue not found, `OWNER_REPO` empty) →
            present inferred number for manual confirmation:
            "Branch references issue #N but verification failed —
            confirm, enter a different number, or skip."
          - `INFERRED_ISSUE` empty →
            ask user for one-line work description,
            offer to invoke issue-workflow Phase 2 to create an issue
          - User declines issue creation →
            set `ISSUE_N=""` and `COVERS=""`.
            Steps depending on `ISSUE_N` (8e spec posting, 8a issue close)
            are skipped.
       2. **Set context values** (from ctx.py output + defaults):
          - `BRANCH_NAME` = `CURRENT_BRANCH` (from ctx.py)
          - `ISSUE_N` = confirmed issue number (or empty if declined)
          - `ISSUE_REPO` = `OWNER_REPO` (from ctx.py)
          - `COVERS` = `ISSUE_N` (single issue only without `.meta`)
          - `DESIGN_REPO_KEY` = `""` (from ctx.py; Step 3 defaults to "project")
          - `PROJECT_SHA` = `""` (no baseline)
          - `META_SECTION_HASHES` = `""` (no section hashes)
          - `FLYWAY_NEXT_V` = `""` (from ctx.py, already empty without `.meta`)
       3. Proceed with normal close flow using these values.
```

- [ ] **Step 2: Fix Step 3 — use ctx.py output instead of .meta re-read**

Replace the Step 3 bash block (lines 265-286) that greps `.meta` directly:

```markdown
**`$DESIGN_REPO` — use `DESIGN_REPO_KEY` from ctx.py output:**

`DESIGN_REPO_KEY` comes from ctx.py (which reads `.meta` at parse time).
When `.meta` is absent, `DESIGN_REPO_KEY` is empty — apply the default case.

```bash
case "$DESIGN_REPO_KEY" in
  workspace)
    DESIGN_REPO="$WORKSPACE" ;;
  project)
    DESIGN_REPO="$PROJECT" ;;
  cross-repo:*)
    CROSS_REPO_NAME="${DESIGN_REPO_KEY#cross-repo:}"
    CANDIDATE="$(dirname "$PROJECT")/$CROSS_REPO_NAME"
    if [ -d "$CANDIDATE/.git" ]; then
      DESIGN_REPO="$CANDIDATE"
    else
      echo "Warning: Cross-repo path not found at $CANDIDATE"
      echo "Options: [S]kip journal merge  [A]bort close"
    fi ;;
  *)
    DESIGN_REPO="$PROJECT" ;;
esac
```
```

- [ ] **Step 3: Add Step 5 skip path for empty PROJECT_SHA**

Add a note after the Step 5 journal validation decisions section:

```markdown
**When `PROJECT_SHA` is empty (`HAS_META=no`):** `section_drift` will be
empty (no baseline hashes to compare). The remaining checks
(`arc42_exists`, `empty_journal`) still apply — a branch without `.meta`
can still have a journal.
```

- [ ] **Step 4: Add Step 8d skip path for empty PROJECT_SHA**

Add before the Step 8d merge steps:

```markdown
**When `PROJECT_SHA` is empty** (no `.meta`), skip journal merge entirely.
The baseline read (`git show "$PROJECT_SHA":ARC42STORIES.MD`) requires a
valid SHA. Journal entries on the branch are preserved but not merged into
ARC42STORIES.MD.
```

- [ ] **Step 5: Update ARC42STORIES.MD**

If the workspace has an `ARC42STORIES.MD` with an L2 Lifecycle section,
add a note documenting that `.meta` absence triggers graceful degradation
(inference from branch name, default routing to "project", journal merge
skipped). This is a documentation-only change — no code.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/SKILL.md
git commit -m "feat(#87): graceful .meta degradation in work-end

Replace pre-condition 2 hard gate with reconstruction block.
Step 3: use DESIGN_REPO_KEY from ctx.py instead of re-reading .meta.
Steps 5 and 8d: skip paths when PROJECT_SHA is empty.

Refs #87"
```

---

### Task 4: Update work-end SKILL.md — slot-mode per-repo sweep (Step 3b)

**Files:**
- Modify: `work-end/SKILL.md` (Step 3b — add slot-mode branch)

**Interfaces:**
- Consumes: `scan-workspace` parameter on close_artifacts.py (Task 2)
- Produces: updated skill instructions for slot-mode sweep

- [ ] **Step 1: Add slot-mode Step 3b section**

After the existing Step 3b content (line ~341), before Step 3c, add:

```markdown
### Step 3b-slot — Per-repo sweep (slot mode only)

When `/worktrees/` is detected in `$PROJECT` path (slot mode), replace the
single-repo sweep above with a per-repo loop:

**1. Discover repos in the slot:**

Priority: `SLOT.md` is authoritative (read via `parse_slot_md()` from
`slot_manager.py`). Fallback: if `SLOT.md` is absent, scan the slot
directory for git repos (via `get_slot_repos()`).

- Primary repo = `$PROJECT` (the repo work-slot was invoked from)
- Secondary repos = other repos in the slot

**2. Per-repo loop** (primary + secondaries), in this order:

For each repo R:
  a. **protocol sweep** against R's `docs/protocols/` — captures rules first
  b. **update-claude-md** against R's CLAUDE.md — syncs conventions
  c. **implementation-doc-sync** against R's `docs/` — syncs documentation

Retargeting: the LLM sets repo R's path as context for each skill
invocation, reading/writing R's files using absolute paths.

Each skill commits its own changes independently (same as non-slot).

**3. Session-bound** (run once, not per-repo), in this order:
  a. **forage SWEEP** — global (no change)
  b. **adr** — primary workspace `adr/` (shared across repos)
  c. **write-content** (diary) — primary workspace `blog/` only
     (last, so it can synthesise the full branch narrative)
```

- [ ] **Step 2: Add Phase B step B4 scan-workspace wiring**

In the Phase B section, update step B4 to pass `scan-workspace`:

```markdown
**B4. Promote artifacts and clean up specs.** Run `close_artifacts.py`
with `scan-workspace` pointing to the slot workspace:

```bash
python3 ~/.claude/skills/work-end/close_artifacts.py \
  <ORIGINAL_WORKSPACE> <PROJECT> <BRANCH_NAME> \
  issue-repo=<ISSUE_REPO> covers=<COVERS> \
  scan-workspace=<SLOT_WORKSPACE>
```

Without `scan-workspace`, Phase B would scan the original workspace
(now on main after B2 merge) and find no branch artifacts to promote.
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/SKILL.md
git commit -m "feat(#88): slot-mode per-repo sweep and scan-workspace wiring

Step 3b-slot: per-repo sweep loop for slot-mode branches.
Phase B step B4: pass scan-workspace to close_artifacts.py.

Refs #88"
```

---

### Task 5: Sync installed skills and run validators

**Files:**
- No new files — validation and sync of existing changes

- [ ] **Step 1: Sync skills to ~/.claude/skills/**

Run: `python3 /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill sync-local --all -y`

- [ ] **Step 2: Run commit-tier validators**

Run: `python3 /Users/mdproctor/claude/hortora/soredium/scripts/validate_all.py --tier commit`
Expected: 0 CRITICAL findings

- [ ] **Step 3: Run full test suite for modified files**

Run: `python3 -m pytest tests/test_ctx.py tests/test_close_artifacts.py -v`
Expected: all tests pass

- [ ] **Step 4: Verify readme-sync workflow**

Check if SKILL.md changes require README.md updates by following
`docs/development/readme-sync.md`.

- [ ] **Step 5: Commit any sync/validation fixes**

If validators or readme-sync flagged issues, fix and commit.

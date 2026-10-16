# Lifecycle Gaps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #459 — Three lifecycle gaps: publish parity, zombie slots, dangling plans
**Issue group:** #459

**Goal:** Close three independent lifecycle gaps — prevent dangling plans after queue advance, prevent zombie slots after close, and verify publish parity across destinations.

**Architecture:** Each gap gets two fixes (prevention + detection). Prevention fixes are one-liners in existing functions. Detection fixes follow the established corruption check and subcommand patterns.

**Tech Stack:** Python 3.14, pytest, existing orchestrator/corruption/plan_manager infrastructure

## Global Constraints

- Protocol: `externalised-scripts-require-tests.md` — every new function gets tests
- Protocol: `evidence-before-claims.md` — run tests before claiming done
- All changes to soredium source (not installed skills) — sync-local after
- No new files except tests — all code goes in existing modules

---

## Batch 1: Dangling Plans (Gap 3)

### Task 1: Clear branch field in advance() + S15 corruption check

**Files:**
- Modify: `work-slot/plan_manager.py:569-570` — add `set_state` call after `rewrite_plan`
- Modify: `project/corruption.py:473-485` — add S15 check after S5/S7
- Test: `tests/test_plan_manager.py` — test branch clearing
- Test: `tests/test_corruption.py` — test S15 detection

**Interfaces:**
- Consumes: `plan_manager.set_state(plan_path, key, value)` (existing)
- Consumes: `plan_io.read_field(plan_path, field)` (existing)
- Produces: `corruption.check_plan_branch_stale(plan_path, workspace, current_branch)` → `Finding | None`

- [ ] **Step 1: Write failing test for advance() clearing branch**

```python
# In tests/test_plan_manager.py — add to existing test class
def test_advance_clears_branch_field(self, tmp_path):
    """advance() should clear the branch field so the next session creates a new branch."""
    from plan_io import read_field
    plan = tmp_path / ".plan"
    plan.write_text(
        "# Work Plan\n\n## State\nbranch: issue-100-old\nstate: active\n"
        "issue-repo: Org/repo\ncovers: 100\n\n## Queue\n"
        "- [ ] org/repo#100 — Old issue ← active\n"
        "- [ ] org/repo#101 — Next issue\n"
    )
    from plan_manager import advance
    advance(plan)
    assert read_field(plan, "branch") == ""
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_plan_manager.py -k "advance_clears_branch" -v`
Expected: FAIL — branch field still contains `issue-100-old`

- [ ] **Step 3: Implement branch clearing in advance()**

In `work-slot/plan_manager.py`, after line 570 (`rewrite_plan(plan_path, tree)`), add:

```python
    if next_leaf:
        set_state(plan_path, "branch", "")
```

This clears the stale branch reference when advancing to the next issue. When `next_leaf` is None (epic complete), the branch field doesn't matter.

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_plan_manager.py -k "advance_clears_branch" -v`
Expected: PASS

- [ ] **Step 5: Write failing test for S15 corruption check**

```python
# In tests/test_corruption.py — add new test class
class TestPlanBranchStale:
    def test_detects_stale_branch_after_advance(self, tmp_path):
        """S15: plan has active state but branch doesn't match active issue."""
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\nbranch: issue-100-old\nstate: active\n"
            "issue-repo: Org/repo\ncovers: 100\n\n## Queue\n"
            "- [x] org/repo#100 — Old issue\n"
            "- [ ] org/repo#101 — Next issue ← active\n"
        )
        from corruption import check_plan_branch_stale
        finding = check_plan_branch_stale(plan, tmp_path, "main")
        assert finding is not None
        assert finding.scenario == "S15_PLAN_BRANCH_STALE"
        assert "clear_branch" in finding.actions

    def test_no_finding_when_branch_matches(self, tmp_path):
        """No S15 when branch matches active issue."""
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\nbranch: issue-101-next\nstate: active\n"
            "issue-repo: Org/repo\ncovers: 101\n\n## Queue\n"
            "- [ ] org/repo#101 — Next issue ← active\n"
        )
        from corruption import check_plan_branch_stale
        finding = check_plan_branch_stale(plan, tmp_path, "main")
        assert finding is None

    def test_no_finding_when_branch_empty(self, tmp_path):
        """No S15 when branch is already empty (advance() cleared it)."""
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan\n\n## State\nbranch: \nstate: active\n"
            "issue-repo: Org/repo\ncovers: 101\n\n## Queue\n"
            "- [ ] org/repo#101 — Next issue ← active\n"
        )
        from corruption import check_plan_branch_stale
        finding = check_plan_branch_stale(plan, tmp_path, "main")
        assert finding is None
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_corruption.py -k "PlanBranchStale" -v`
Expected: FAIL — `check_plan_branch_stale` not defined

- [ ] **Step 7: Implement S15 check in corruption.py**

Add after `check_stale_plan_on_main` (around line 138):

```python
def check_plan_branch_stale(
    plan_path: Path, workspace: Path, current_branch: str,
) -> Optional[Finding]:
    if not plan_path.exists():
        return None
    plan_state = read_field(plan_path, "state")
    if plan_state != "active":
        return None
    plan_branch = read_field(plan_path, "branch")
    if not plan_branch:
        return None
    # Extract issue number from branch name
    import re
    branch_match = re.search(r'issue-(\d+)', plan_branch)
    if not branch_match:
        return None
    branch_issue = int(branch_match.group(1))
    # Find active issue in queue
    try:
        tree = read_plan(plan_path)
    except Exception:
        return None
    active_issue = None
    for item in tree.get("queue", []):
        text = item if isinstance(item, str) else str(item)
        if "← active" in text:
            issue_match = re.search(r'#(\d+)', text)
            if issue_match:
                active_issue = int(issue_match.group(1))
            break
    if active_issue is None or active_issue == branch_issue:
        return None
    return Finding(
        scenario="S15_PLAN_BRANCH_STALE",
        severity="warning",
        detail=(f".plan branch '{plan_branch}' references issue #{branch_issue} "
                f"but active queue issue is #{active_issue}"),
        actions=["clear_branch", "switch_to_main"],
    )
```

Wire it into `diagnose()` after S5/S7 (around line 482):

```python
        s15 = check_plan_branch_stale(plan_path, workspace, current_branch)
        if s15:
            findings.append(s15)
```

- [ ] **Step 8: Run all corruption tests**

Run: `python3 -m pytest tests/test_corruption.py -v`
Expected: All pass including new S15 tests

- [ ] **Step 9: Commit**

```bash
git add work-slot/plan_manager.py project/corruption.py tests/test_plan_manager.py tests/test_corruption.py
git commit -m "feat(#459): clear branch on advance + S15 stale branch detection

advance() now clears the branch field when advancing to the next
queue issue. S15_PLAN_BRANCH_STALE detects plans where the branch
references a different issue than the active queue entry.

Refs #459"
```

---

## Batch 2: Zombie Slots (Gap 2)

### Task 2: Relax skip condition + S16 zombie detection

**Files:**
- Modify: `work-end/work_end_orchestrator.py:276-283` — exclude current workspace from .plan check
- Modify: `project/corruption.py` — add S16 check
- Test: `tests/test_work_end_orchestrator.py` — test relaxed skip
- Test: `tests/test_corruption.py` — test S16 detection

**Interfaces:**
- Consumes: `ctx.workspace` (existing on OrchestratorContext)
- Produces: `corruption.check_zombie_slot(slot_path, project, base_branch)` → `Finding | None`

- [ ] **Step 1: Write failing test for relaxed skip condition**

```python
# In tests/test_work_end_orchestrator.py — add new test class
class TestSkipMultiRepoRelaxed:
    def test_current_workspace_plan_does_not_block_archival(self, tmp_path):
        """The current workspace's .plan shouldn't prevent archival."""
        from work_end_orchestrator import _skip_multi_repo_partial_close, OrchestratorContext
        from progress import Progress
        # Create slot dir with one repo on main and one workspace with .plan
        slot = tmp_path / "slot"
        slot.mkdir()
        repo = slot / "engine"
        repo.mkdir()
        (repo / ".git").mkdir()
        ws = slot / "workspace"
        ws.mkdir()
        (ws / ".git").mkdir()
        (ws / ".workspace").touch()
        (ws / ".plan").write_text("state: active\n")
        # Simulate: git rev-parse returns "main" for the repo
        ctx = OrchestratorContext(
            workspace=ws, project=repo,
            branch="issue-42-test", base_branch="main",
            meta_state="closing:stamped",
            on_main=False, in_slot=True,
            covers="42", issue_repo="Org/repo",
            progress=Progress(),
            slot_path=slot,
        )
        ctx.repos = []
        result = _skip_multi_repo_partial_close(ctx)
        assert result is False, "Current workspace .plan should not block archival"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -k "current_workspace_plan_does_not_block" -v`
Expected: FAIL — returns True (blocks archival)

- [ ] **Step 3: Fix _skip_multi_repo_partial_close**

In `work-end/work_end_orchestrator.py`, change the workspace .plan check (lines 281-283):

```python
            if (entry / ".workspace").exists():
                if (entry / ".plan").exists() and entry != ctx.workspace:
                    return True
                continue
```

Add `and entry != ctx.workspace` to the condition. The current workspace's .plan is about to be cleaned up — only other workspaces' plans indicate real ongoing work.

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -k "current_workspace_plan_does_not_block" -v`
Expected: PASS

- [ ] **Step 5: Write failing test for S16 zombie slot detection**

```python
# In tests/test_corruption.py — add new test class
class TestZombieSlot:
    def test_detects_zombie_slot(self, tmp_path):
        """S16: slot is active but all branches are stamped."""
        slot = tmp_path / "slot"
        slot.mkdir()
        slot_file = slot / ".slot"
        slot_file.write_text("repos:\n  - engine\n")
        repo = slot / "engine"
        repo.mkdir()
        (repo / ".git").mkdir()
        from corruption import check_zombie_slot
        # Mock: all repos on main, stamp commit exists
        finding = check_zombie_slot(
            slot, tmp_path / "project", "main",
            all_stamped=True,
        )
        assert finding is not None
        assert finding.scenario == "S16_ZOMBIE_SLOT"
        assert "archive_slot" in finding.actions

    def test_no_finding_when_branch_active(self, tmp_path):
        """No S16 when a repo still has an active branch."""
        slot = tmp_path / "slot"
        slot.mkdir()
        from corruption import check_zombie_slot
        finding = check_zombie_slot(
            slot, tmp_path / "project", "main",
            all_stamped=False,
        )
        assert finding is None
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_corruption.py -k "ZombieSlot" -v`
Expected: FAIL — `check_zombie_slot` not defined

- [ ] **Step 7: Implement S16 check in corruption.py**

```python
def check_zombie_slot(
    slot_path: Path, project: Path, base_branch: str,
    all_stamped: bool | None = None,
) -> Optional[Finding]:
    if not slot_path or not slot_path.is_dir():
        return None
    slot_file = slot_path / ".slot"
    if not slot_file.exists():
        return None
    if all_stamped is None:
        # Check each repo in the slot
        all_stamped = True
        for entry in sorted(slot_path.iterdir()):
            if not entry.is_dir() or not (entry / ".git").exists():
                continue
            if (entry / ".workspace").exists():
                continue
            branch_out, _ = _git(entry, "rev-parse", "--abbrev-ref", "HEAD")
            if branch_out.strip() != base_branch:
                all_stamped = False
                break
    if not all_stamped:
        return None
    return Finding(
        scenario="S16_ZOMBIE_SLOT",
        severity="warning",
        detail="slot is active but all repos are on main — likely missed archival",
        actions=["archive_slot", "ignore"],
    )
```

- [ ] **Step 8: Run all tests**

Run: `python3 -m pytest tests/test_corruption.py tests/test_work_end_orchestrator.py -k "ZombieSlot or current_workspace_plan" -v`
Expected: All pass

- [ ] **Step 9: Commit**

```bash
git add work-end/work_end_orchestrator.py project/corruption.py tests/test_work_end_orchestrator.py tests/test_corruption.py
git commit -m "feat(#459): relax slot skip condition + S16 zombie slot detection

_skip_multi_repo_partial_close now excludes the current workspace's
.plan from the 'has remaining work' check. S16_ZOMBIE_SLOT detects
slots that are active but have all repos on main.

Refs #459"
```

---

## Batch 3: Publish Parity (Gap 1)

### Task 3: Add verify-parity and check-present subcommands

**Files:**
- Modify: `publish-blog/blog_publish.py` — add two new subcommands
- Modify: `publish-blog/SKILL.md` — add verification step, pre-copy check
- Test: `tests/test_blog_publish.py` — tests for both subcommands

**Interfaces:**
- Consumes: `blog-routing.yaml` structure (destinations dict, rules list with match/destinations)
- Produces: `verify_parity(routing_yaml, dest_configs)` → prints KEY=VALUE parity report
- Produces: `check_present(entry_path, dest_dir)` → prints `PRESENT=yes|no`

- [ ] **Step 1: Write failing test for check_present**

```python
# In tests/test_blog_publish.py — add new test class
class TestCheckPresent:
    def test_present_when_file_exists(self, tmp_path):
        from blog_publish import check_present_fn
        dest = tmp_path / "dest"
        dest.mkdir()
        (dest / "entry.md").write_text("content")
        assert check_present_fn(str(tmp_path / "entry.md"), str(dest)) is True

    def test_absent_when_file_missing(self, tmp_path):
        from blog_publish import check_present_fn
        dest = tmp_path / "dest"
        dest.mkdir()
        assert check_present_fn(str(tmp_path / "entry.md"), str(dest)) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_blog_publish.py -k "CheckPresent" -v`
Expected: FAIL — `check_present_fn` not defined

- [ ] **Step 3: Implement check_present**

In `publish-blog/blog_publish.py`, add:

```python
def check_present_fn(entry_path: str, dest_dir: str) -> bool:
    src = Path(entry_path)
    dest = Path(dest_dir) / src.name
    return dest.exists()


def check_present(entry_path: str, dest_dir: str) -> None:
    present = check_present_fn(entry_path, dest_dir)
    print(f"PRESENT={'yes' if present else 'no'}", flush=True)
```

Add to `main()` dispatch:

```python
    elif subcommand == "check-present":
        if len(sys.argv) != 4:
            print("Usage: check-present <entry-path> <dest-dir>", file=sys.stderr)
            sys.exit(1)
        check_present(sys.argv[2], sys.argv[3])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_blog_publish.py -k "CheckPresent" -v`
Expected: PASS

- [ ] **Step 5: Write failing test for verify_parity**

```python
class TestVerifyParity:
    def test_detects_missing_entry_at_destination(self, tmp_path):
        import yaml
        from blog_publish import verify_parity_fn
        # Set up two destinations
        dest_a = tmp_path / "dest-a"
        dest_a.mkdir()
        dest_b = tmp_path / "dest-b"
        dest_b.mkdir()
        # Entry exists in A but not B
        (dest_a / "entry.md").write_text("content")
        routing = {
            "destinations": {
                "a": {"path": str(dest_a)},
                "b": {"path": str(dest_b)},
            },
            "defaults": {"destinations": ["a", "b"]},
            "rules": [],
        }
        routing_path = tmp_path / "routing.yaml"
        routing_path.write_text(yaml.dump(routing))
        missing = verify_parity_fn(str(routing_path))
        assert len(missing) == 1
        assert missing[0]["entry"] == "entry.md"
        assert missing[0]["missing_dest"] == "b"

    def test_parity_ok_when_all_present(self, tmp_path):
        import yaml
        from blog_publish import verify_parity_fn
        dest_a = tmp_path / "dest-a"
        dest_a.mkdir()
        dest_b = tmp_path / "dest-b"
        dest_b.mkdir()
        (dest_a / "entry.md").write_text("content")
        (dest_b / "entry.md").write_text("content")
        routing = {
            "destinations": {
                "a": {"path": str(dest_a)},
                "b": {"path": str(dest_b)},
            },
            "defaults": {"destinations": ["a", "b"]},
            "rules": [],
        }
        routing_path = tmp_path / "routing.yaml"
        routing_path.write_text(yaml.dump(routing))
        missing = verify_parity_fn(str(routing_path))
        assert missing == []
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_blog_publish.py -k "VerifyParity" -v`
Expected: FAIL — `verify_parity_fn` not defined

- [ ] **Step 7: Implement verify_parity**

In `publish-blog/blog_publish.py`, add:

```python
def verify_parity_fn(routing_yaml: str) -> list[dict]:
    import yaml
    config = yaml.safe_load(Path(routing_yaml).read_text())
    destinations = config.get("destinations", {})
    defaults = config.get("defaults", {}).get("destinations", [])
    if len(defaults) < 2:
        return []
    # Collect all entries across default destinations
    dest_entries: dict[str, set[str]] = {}
    for name in defaults:
        dest_config = destinations.get(name, {})
        dest_path = Path(dest_config.get("path", "")).expanduser()
        subdir = dest_config.get("subdir", "")
        full_path = dest_path / subdir if subdir else dest_path
        if full_path.is_dir():
            dest_entries[name] = {f.name for f in full_path.iterdir() if f.is_file() and f.suffix == ".md"}
        else:
            dest_entries[name] = set()
    # Find the union of all entries
    all_entries = set()
    for entries in dest_entries.values():
        all_entries |= entries
    # Check parity
    missing = []
    for entry in sorted(all_entries):
        for name in defaults:
            if entry not in dest_entries.get(name, set()):
                missing.append({"entry": entry, "missing_dest": name})
    return missing


def verify_parity(routing_yaml: str) -> None:
    missing = verify_parity_fn(routing_yaml)
    if not missing:
        print("PARITY=yes", flush=True)
        print("MISSING_COUNT=0", flush=True)
        return
    print("PARITY=no", flush=True)
    print(f"MISSING_COUNT={len(missing)}", flush=True)
    for i, m in enumerate(missing, 1):
        print(f"MISSING_{i}={m['entry']} dest={m['missing_dest']}", flush=True)
```

Add to `main()` dispatch:

```python
    elif subcommand == "verify-parity":
        if len(sys.argv) != 3:
            print("Usage: verify-parity <routing-yaml>", file=sys.stderr)
            sys.exit(1)
        verify_parity(sys.argv[2])
```

- [ ] **Step 8: Run test to verify it passes**

Run: `python3 -m pytest tests/test_blog_publish.py -k "VerifyParity" -v`
Expected: PASS

- [ ] **Step 9: Update publish-blog SKILL.md**

Add pre-copy check to Step 6 and verification step after Step 7. (Exact SKILL.md edits are documentation — read the current SKILL.md and add the steps inline.)

- [ ] **Step 10: Run all blog_publish tests**

Run: `python3 -m pytest tests/test_blog_publish.py -v`
Expected: All pass

- [ ] **Step 11: Commit**

```bash
git add publish-blog/blog_publish.py publish-blog/SKILL.md tests/test_blog_publish.py
git commit -m "feat(#459): add verify-parity and check-present to blog_publish

verify-parity scans routing config and reports entries that exist
at some destinations but not all. check-present is a pre-copy check
that prevents redundant copies during backfill.

Refs #459"
```

---

## References

- [2026-10-16-lifecycle-gaps-design.md] — design spec this plan implements
- `work-slot/plan_manager.py:529-570` — advance() function
- `project/corruption.py:93-165` — S5, S6, S7 checks (pattern for S15/S16)
- `work-end/work_end_orchestrator.py:260-290` — _skip_multi_repo_partial_close
- `publish-blog/blog_publish.py` — existing subcommand pattern
- GE-20260813-556494 — authoritative reads with best-effort writes
- GE-20260821-e9c59e — worklog.db slot lifecycle audit
- `docs/protocols/externalised-scripts-require-tests.md`
- GitHub #452 — multi-destination verification
- GitHub #402 — S14_MISSING_PLAN

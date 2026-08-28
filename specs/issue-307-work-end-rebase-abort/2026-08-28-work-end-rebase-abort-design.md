# Work-end: Prevent silent branch content abandonment after rebase abort

## Problem

Work-end can silently abandon a feature branch's content when a rebase fails in slot-mode landing. Observed in slot 140 (casehub): 849 commits on branch `issue-421-worker-reasoning-traces`, rebase failed and was aborted, but the pipeline continued — merged main, wrote `.landed`, reported success, and archived the slot. 377 source files remained unmerged.

Two independent safety nets failed:
1. `land_flow.py` — no postcondition check after merge+push; stamp proceeded despite content gap
2. `verify_slot_close.py` — the orchestrator's `verify` step completed with `done` despite 377 files differing

## Design

Three changes, each independently preventing the failure mode:

### 1. Land postcondition in `land_flow.py` (D1)

After `_merge_and_push_two_hop` or `_merge_and_push_direct` succeeds for each repo, verify the branch content actually reached the base branch before proceeding to stamp.

**Location:** `land_flow.py`, between merge+push and stamp (between lines 655 and 659 in current code).

**Check:** For each repo that successfully merged+pushed, run a source-file diff between the branch and the base branch. Reuse the same extension list and logic from `verify_stamp.py`:

```python
def _verify_content_landed(desc: RepoDescriptor, branch: str) -> str | None:
    """Return error string if branch content is not on base_branch, else None."""
    result = _git(desc.repo_path, "diff", "--name-only",
                  f"{desc.base_branch}...{branch}")
    if result.returncode != 0:
        return f"diff_failed repo={desc.repo_path.name}"
    
    source_exts = (".java", ".kt", ".py", ".ts", ".tsx", ".js", ".jsx",
                   ".xml", ".yaml", ".yml", ".json", ".sql", ".css",
                   ".scss", ".html", ".properties")
    source_files = [f for f in result.stdout.strip().split("\n")
                    if f and any(f.endswith(ext) for ext in source_exts)]
    if not source_files:
        return None
    
    diff = _git(desc.repo_path, "diff", desc.base_branch, branch,
                "--", *source_files)
    if diff.returncode == 0 and diff.stdout.strip():
        count = len([f for f in _git(desc.repo_path, "diff", "--name-only",
                     desc.base_branch, branch, "--", *source_files)
                     .stdout.strip().split("\n") if f])
        return f"content_not_landed repo={desc.repo_path.name} files={count}"
    return None
```

**Integration into `land_batch`:** After the merge+push loop (Step 3) succeeds and before the stamp loop (Step 4), iterate over successfully pushed repos and call `_verify_content_landed`. If any repo fails, set `result.success = False`, append a `RepoStatus` with `error="content_not_landed"`, and return early — do not proceed to stamp.

**Why before stamp, not after:** The stamp writes `chore: branch closed` which signals to all downstream consumers (archive, cleanup, lifecycle) that the branch is done. Once stamped, recovery requires manual intervention. Catching the gap before stamp means the branch stays open and landable.

### 2. Pre-archive hard gate in `archive_slot()` (D2)

A last-resort safety net: `archive_slot()` refuses to archive any slot that has unmerged branch content, regardless of `.landed` markers or `--force` flags.

**Location:** `slot_manager.py:archive_slot()`, early — before the existing `_repack_broken_alternates` call and `shutil.move`.

**Check:** For each git repo in the slot directory, determine the branch (from `git branch --show-current` or from `.slot` metadata), then run the same source-file diff against main:

```python
def _has_unmerged_content(slot_dir: Path) -> list[str]:
    """Return list of repo names with unmerged branch content."""
    unmerged = []
    for repo_dir in sorted(slot_dir.iterdir()):
        if not repo_dir.is_dir() or not (repo_dir / ".git").exists():
            continue
        branch_r = run_cmd(["git", "branch", "--show-current"], cwd=str(repo_dir))
        if branch_r[0] != 0:
            continue
        branch = branch_r[1].strip()
        if not branch or branch == "main":
            continue
        diff_r = run_cmd(
            ["git", "diff", "--stat", f"main...{branch}"],
            cwd=str(repo_dir),
        )
        if diff_r[0] == 0 and diff_r[1].strip():
            unmerged.append(repo_dir.name)
    return unmerged
```

**Gate behaviour:** If `_has_unmerged_content` returns any repos, print an error and exit. No `--force` override — this is a data loss prevention gate:

```
ERROR=unmerged_content slot=<N>
ERROR_DETAIL=repos with unmerged branch content: <repo1>, <repo2>
HINT=land the branch content first, or manually verify it's already on main under different SHAs
```

The hint is important: in some cases content IS on main under different SHAs (squash-merged). The user needs to verify manually and, if satisfied, can checkout main in the affected clones before re-running archive.

### 3. Rebase failure: hard stop + worklog recording (D3, D4)

On rebase failure, work-end stops and reports to the user for guidance rather than falling through.

**Current behaviour (already correct in land_flow.py):** `land_batch()` returns `success=False` on rebase failure after 3 retries (line 615-622). The orchestrator receives `ERROR=REBASE_CONFLICT` from `cmd_rebase` and yields to the LLM.

**What needs fixing:** The orchestrator's rebase step can fail, but the land step (which has its own internal rebase in slot-mode via `land_flow.land_batch()`) may attempt a different rebase strategy. The key is that D1's postcondition now catches any scenario where content doesn't land regardless of the rebase strategy used.

**Worklog recording:** On rebase failure, record a `rebase_failed` event in the worklog DB with:
- `repo_path`: which repo failed
- `branch`: the branch being rebased
- `commit_count`: number of commits on the branch
- `main_ahead`: how far main has moved since the branch point
- `error_detail`: the git stderr output (truncated to 500 chars)

This enables fleet queries like "which slots have rebase failures?" and "how long have they been stuck?"

**Location:** In `land_flow.py:land_batch()` at the rebase failure return (line 618-622), add a worklog recording call before returning. In `work_end_execute.py:cmd_rebase()` at the error return (lines 228 and 235), add the same.

## Testing

### Land postcondition (D1)
- Test: merge succeeds but branch has files not on base → `_verify_content_landed` returns error string
- Test: merge succeeds and all branch content on base → returns None
- Test: branch has no source files (docs only) → returns None (no false positives)
- Test: `land_batch` returns `success=False` when postcondition fails, stamp is not attempted

### Pre-archive gate (D2)
- Test: slot with repo on feature branch with unmerged content → archive refused
- Test: slot with repo on main → archive proceeds
- Test: slot with repo on feature branch, all content on main → archive proceeds
- Test: multiple repos, one has unmerged content → archive refused, error names the repo

### Worklog recording (D4)
- Test: rebase failure records event with correct fields
- Test: worklog unavailable → warning printed, no crash

## References

- work-end/land_flow.py:548-683 — `land_batch` pipeline (preflight → rebase → merge+push → stamp)
- work-end/land_flow.py:343-358 — `_rebase_repo` (rebase with abort on failure)
- work-end/work_end_execute.py:198-240 — `cmd_rebase` (orchestrator rebase step)
- work-end/work_end_execute.py:243-316 — `cmd_land` (orchestrator land step, delegates to land_flow)
- work-end/work_end_orchestrator.py:818-840 — step sequence (rebase → squash → land → lifecycle transitions)
- work-end/verify_stamp.py — existing content verification (reused pattern for D1)
- work-slot/slot_manager.py:1636-1741 — `archive_slot` (D2 gate location)
- Slot 140 incident — 849 commits, 377 unmerged files, verify=done, archive=done

# Duplicate Commit Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #312 — fix: detect and prevent merge-after-rebase duplicate commits on main
**Issue group:** #312

**Goal:** Detect duplicate commits (same content, different SHAs) before merging in sync-main, reconcile automatically by dropping duplicates, and provide an audit script for periodic scanning.

**Architecture:** Two-stage detection (message pre-filter + patch-id confirmation) in `branch_create.py`. Pre-merge gate in `_sync_repo()` reconciles by dropping duplicate commits via `GIT_SEQUENCE_EDITOR`. Standalone audit script for on-demand repo scanning.

**Tech Stack:** Python 3, git CLI (patch-id, log, rebase)

## Global Constraints

- All functions use `run_git()` from `branch_create.py` for git operations
- KEY=VALUE stdout protocol for all script output
- Non-fatal — sync-main never fails the overall work-start flow
- Tests use `_init_repo_with_bare()` and `_make_fork_of()` helpers from existing test suite

---

## Batch 1: Detection core

### Task 1: `_get_patch_ids()` and `_find_duplicate_commits()`

**Files:**
- Modify: `work-start/branch_create.py` (add after `_has_shared_fork_commits` at line 157)
- Test: `tests/test_branch_create.py`

**Interfaces:**
- Consumes: `run_git(repo, *args)` → `(bool, str)`
- Produces:
  - `_get_patch_ids(repo: str, rev_range: str) -> dict[str, str]` — returns `{patch_id: sha}`
  - `_find_duplicate_commits(repo: str, local_range: str, remote_range: str) -> list[tuple[str, str, str]]` — returns `[(local_sha, remote_sha, subject)]`

- [ ] **Step 1: Write failing test — `_get_patch_ids` returns patch-id mapping**

```python
def test_get_patch_ids_returns_mapping(tmp_path):
    project, bare = _init_repo_with_bare(tmp_path, "project")
    (project / "file.txt").write_text("content")
    subprocess.run(["git", "-C", str(project), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(project), "commit", "-m", "add file"],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(project), "push", "origin", "main"],
                   capture_output=True)
    from branch_create import _get_patch_ids
    result = _get_patch_ids(str(project), "HEAD~1..HEAD")
    assert len(result) == 1
    sha = list(result.values())[0]
    assert len(sha) >= 7
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_branch_create.py::test_get_patch_ids_returns_mapping -v`
Expected: ImportError — `_get_patch_ids` does not exist

- [ ] **Step 3: Implement `_get_patch_ids`**

Add to `branch_create.py` after line 157:

```python
def _get_patch_ids(repo: str, rev_range: str) -> dict[str, str]:
    """Map patch-id -> commit SHA for commits in the given range.

    Uses git patch-id to compute content-based identity for each commit.
    """
    ok, log_out = run_git(repo, "log", "--format=%H", rev_range)
    if not ok or not log_out.strip():
        return {}
    result = {}
    for sha in log_out.strip().splitlines():
        sha = sha.strip()
        if not sha:
            continue
        diff_proc = subprocess.run(
            ["git", "-C", repo, "diff-tree", "-p", sha],
            capture_output=True, text=True,
        )
        if diff_proc.returncode != 0 or not diff_proc.stdout.strip():
            continue
        pid_proc = subprocess.run(
            ["git", "patch-id", "--stable"],
            input=diff_proc.stdout,
            capture_output=True, text=True,
        )
        if pid_proc.returncode == 0 and pid_proc.stdout.strip():
            parts = pid_proc.stdout.strip().split()
            if len(parts) >= 1:
                result[parts[0]] = sha
    return result
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_branch_create.py::test_get_patch_ids_returns_mapping -v`
Expected: PASS

- [ ] **Step 5: Write failing test — `_find_duplicate_commits` detects cherry-picked commits**

```python
def test_find_duplicate_commits_detects_cherry_pick(tmp_path):
    """Cherry-picked commit appears on both sides with different SHA but same patch-id."""
    blessed, blessed_bare = _init_repo_with_bare(tmp_path, "blessed")
    fork, fork_bare = _make_fork_of(tmp_path, blessed_bare)

    # Create a commit on fork
    (fork / "feature.txt").write_text("feature work")
    subprocess.run(["git", "-C", str(fork), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(fork), "commit", "-m", "feat: add feature"],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(fork), "push", "origin", "main"],
                   capture_output=True, check=True)

    # Cherry-pick the same commit into blessed (simulating upstream merge)
    fork_sha = subprocess.run(
        ["git", "-C", str(fork), "rev-parse", "HEAD"],
        capture_output=True, text=True,
    ).stdout.strip()
    subprocess.run(["git", "-C", str(blessed), "fetch", str(fork_bare)],
                   capture_output=True)
    subprocess.run(["git", "-C", str(blessed), "cherry-pick", fork_sha],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(blessed), "push", "origin", "main"],
                   capture_output=True, check=True)

    # Fetch so fork sees blessed/main
    subprocess.run(["git", "-C", str(fork), "fetch", "upstream"],
                   capture_output=True)

    from branch_create import _find_duplicate_commits
    dupes = _find_duplicate_commits(
        str(fork), "upstream/main..main", "main..upstream/main"
    )
    assert len(dupes) == 1
    assert dupes[0][2] == "feat: add feature"
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest tests/test_branch_create.py::test_find_duplicate_commits_detects_cherry_pick -v`
Expected: ImportError — `_find_duplicate_commits` does not exist

- [ ] **Step 7: Implement `_find_duplicate_commits`**

```python
def _find_duplicate_commits(repo: str, local_range: str,
                            remote_range: str) -> list[tuple[str, str, str]]:
    """Find commits with identical content on both sides of a merge.

    Two-stage: message pre-filter (fast) then patch-id confirmation (precise).
    Returns [(local_sha, remote_sha, subject)] for confirmed duplicates.
    """
    ok, local_log = run_git(repo, "log", "--format=%H %s", local_range)
    if not ok or not local_log.strip():
        return []
    ok, remote_log = run_git(repo, "log", "--format=%H %s", remote_range)
    if not ok or not remote_log.strip():
        return []

    def parse_log(raw: str) -> list[tuple[str, str]]:
        entries = []
        for line in raw.strip().splitlines():
            parts = line.split(" ", 1)
            if len(parts) == 2:
                entries.append((parts[0], parts[1]))
        return entries

    local_commits = parse_log(local_log)
    remote_commits = parse_log(remote_log)

    remote_by_subject: dict[str, list[str]] = {}
    for sha, subj in remote_commits:
        remote_by_subject.setdefault(subj, []).append(sha)

    candidates = []
    for local_sha, subj in local_commits:
        if subj in remote_by_subject:
            for remote_sha in remote_by_subject[subj]:
                candidates.append((local_sha, remote_sha, subj))

    if not candidates:
        return []

    local_patches = _get_patch_ids(repo, local_range)
    remote_patches = _get_patch_ids(repo, remote_range)

    local_sha_to_pid = {sha: pid for pid, sha in local_patches.items()}
    remote_sha_to_pid = {sha: pid for pid, sha in remote_patches.items()}

    confirmed = []
    for local_sha, remote_sha, subj in candidates:
        local_pid = local_sha_to_pid.get(local_sha)
        remote_pid = remote_sha_to_pid.get(remote_sha)
        if local_pid and remote_pid and local_pid == remote_pid:
            confirmed.append((local_sha, remote_sha, subj))

    return confirmed
```

- [ ] **Step 8: Run test to verify it passes**

Run: `python3 -m pytest tests/test_branch_create.py::test_find_duplicate_commits_detects_cherry_pick -v`
Expected: PASS

- [ ] **Step 9: Write failing test — same message but different content NOT flagged**

```python
def test_find_duplicate_commits_ignores_different_content(tmp_path):
    """Same commit message but different file content — not a duplicate."""
    blessed, blessed_bare = _init_repo_with_bare(tmp_path, "blessed")
    fork, fork_bare = _make_fork_of(tmp_path, blessed_bare)

    (fork / "file.txt").write_text("fork version")
    subprocess.run(["git", "-C", str(fork), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(fork), "commit", "-m", "fix: update file"],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(fork), "push", "origin", "main"],
                   capture_output=True)

    (blessed / "file.txt").write_text("blessed version")
    subprocess.run(["git", "-C", str(blessed), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(blessed), "commit", "-m", "fix: update file"],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(blessed), "push", "origin", "main"],
                   capture_output=True)

    subprocess.run(["git", "-C", str(fork), "fetch", "upstream"], capture_output=True)

    from branch_create import _find_duplicate_commits
    dupes = _find_duplicate_commits(
        str(fork), "upstream/main..main", "main..upstream/main"
    )
    assert len(dupes) == 0
```

- [ ] **Step 10: Run test — should pass immediately (negative case)**

Run: `python3 -m pytest tests/test_branch_create.py::test_find_duplicate_commits_ignores_different_content -v`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-start/branch_create.py tests/test_branch_create.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: _get_patch_ids and _find_duplicate_commits for duplicate detection Refs #312"
```

## Batch 2: Pre-merge gate and reconciliation

### Task 2: Wire gate into `_sync_repo()` with reconciliation

**Files:**
- Modify: `work-start/branch_create.py` — `_sync_repo()` function and new `_rebase_dropping()` helper
- Test: `tests/test_branch_create.py`

**Interfaces:**
- Consumes: `_find_duplicate_commits()`, `_get_patch_ids()`, `run_git()`
- Produces:
  - `_rebase_dropping(repo: str, blessed_remote: str, base: str, drop_shas: set[str]) -> bool` — returns True if rebase succeeded
  - Modified `_sync_repo()` emits `DUPLICATE_COMMITS=N`, `RECONCILED=N`, `STRATEGY=reconcile`

- [ ] **Step 1: Write failing test — sync-main detects and reconciles duplicates**

```python
def test_sync_main_reconciles_duplicate_commits(tmp_path):
    """When duplicate commits exist on both sides, reconcile by dropping dupes."""
    blessed, blessed_bare = _init_repo_with_bare(tmp_path, "blessed")
    fork, fork_bare = _make_fork_of(tmp_path, blessed_bare)

    # Create commit on fork and push
    (fork / "feature.txt").write_text("the feature")
    subprocess.run(["git", "-C", str(fork), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(fork), "commit", "-m", "feat: the feature"],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(fork), "push", "origin", "main"],
                   capture_output=True)

    # Cherry-pick same commit into blessed (simulates upstream accepting the work)
    fork_sha = subprocess.run(
        ["git", "-C", str(fork), "rev-parse", "HEAD"],
        capture_output=True, text=True,
    ).stdout.strip()
    subprocess.run(["git", "-C", str(blessed), "fetch", str(fork_bare)],
                   capture_output=True)
    subprocess.run(["git", "-C", str(blessed), "cherry-pick", fork_sha],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(blessed), "push", "origin", "main"],
                   capture_output=True)

    workspace, _ = _init_repo_with_bare(tmp_path, "workspace")

    result = subprocess.run(
        ["python3", str(BRANCH_CREATE), "sync-main",
         str(fork), str(workspace), "base=main"],
        capture_output=True, text=True,
    )
    assert result.returncode == 0
    assert "DUPLICATE_COMMITS=1" in result.stdout
    assert "STRATEGY=reconcile" in result.stdout

    # Verify the feature exists exactly once in history
    log = subprocess.run(
        ["git", "-C", str(fork), "log", "--oneline", "--all", "--grep=feat: the feature"],
        capture_output=True, text=True,
    )
    lines = [l for l in log.stdout.strip().splitlines() if l.strip()]
    assert len(lines) == 1, f"Expected 1 commit, got {len(lines)}: {log.stdout}"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_branch_create.py::test_sync_main_reconciles_duplicate_commits -v`
Expected: FAIL — no `DUPLICATE_COMMITS` in output (old code merges without checking)

- [ ] **Step 3: Implement `_rebase_dropping` helper**

Add to `branch_create.py`:

```python
def _rebase_dropping(repo: str, blessed_remote: str, base: str,
                     drop_shas: set[str]) -> bool:
    """Rebase onto blessed, dropping commits whose SHAs are in drop_shas.

    Uses GIT_SEQUENCE_EDITOR to remove 'pick <sha>' lines for duplicate
    commits from the rebase todo before execution begins.
    """
    import os
    import tempfile

    if not drop_shas:
        ok, _ = run_git(repo, "rebase", f"{blessed_remote}/{base}")
        return ok

    short_shas = set()
    for sha in drop_shas:
        short_shas.add(sha[:7])
        short_shas.add(sha)

    script = tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False)
    script.write("import sys\n")
    script.write(f"drops = {short_shas!r}\n")
    script.write("path = sys.argv[1]\n")
    script.write("with open(path) as f: lines = f.readlines()\n")
    script.write("kept = []\n")
    script.write("dropped = 0\n")
    script.write("for line in lines:\n")
    script.write("    parts = line.split()\n")
    script.write("    if len(parts) >= 2 and parts[0] == 'pick' and any(parts[1].startswith(s) for s in drops):\n")
    script.write("        dropped += 1\n")
    script.write("        continue\n")
    script.write("    kept.append(line)\n")
    script.write("with open(path, 'w') as f: f.writelines(kept)\n")
    script.write(f"print(f'DROPPED={{dropped}}')\n")
    script.name_path = script.name
    script.close()

    env = dict(os.environ)
    env["GIT_SEQUENCE_EDITOR"] = f"python3 {script.name}"
    try:
        proc = subprocess.run(
            ["git", "-C", repo, "rebase", "-i", f"{blessed_remote}/{base}"],
            capture_output=True, text=True, env=env,
        )
        return proc.returncode == 0
    finally:
        os.unlink(script.name)
```

- [ ] **Step 4: Wire gate into `_sync_repo`**

Modify `_sync_repo` — add duplicate detection before the merge/rebase decision:

```python
def _sync_repo(repo, blessed_remote, origin_remote, base, warnings, label):
    ok, _ = run_git(repo, "fetch", blessed_remote)
    if not ok:
        warnings.append(f"fetch_{label}_failed")
        return "skipped"

    if origin_remote != blessed_remote:
        run_git(repo, "fetch", origin_remote)

    # --- Duplicate detection gate ---
    local_range = f"{blessed_remote}/{base}..{base}"
    remote_range = f"{base}..{blessed_remote}/{base}"
    duplicates = _find_duplicate_commits(repo, local_range, remote_range)

    if duplicates:
        for local_sha, remote_sha, subj in duplicates:
            print(f"DUPLICATE_DETAIL={local_sha[:12]}:{remote_sha[:12]}:{subj[:60]}")
        print(f"DUPLICATE_COMMITS={len(duplicates)} repo={Path(repo).name}")

        blessed_patches = _get_patch_ids(repo, remote_range)
        local_commits_with_pids = _get_patch_ids(repo, local_range)
        drop_shas = {sha for pid, sha in local_commits_with_pids.items()
                     if pid in blessed_patches}

        ok = _rebase_dropping(repo, blessed_remote, base, drop_shas)
        if not ok:
            run_git(repo, "rebase", "--abort")
            warnings.append(f"reconcile_{label}_failed")
            return "skipped"

        if origin_remote != blessed_remote:
            run_git(repo, "push", origin_remote, base, "--force-with-lease", "--no-verify")

        print(f"RECONCILED={len(drop_shas)} repo={Path(repo).name}")
        print(f"STRATEGY=reconcile repo={Path(repo).name}")
        return "reconciled"

    # --- Existing merge/rebase logic (unchanged) ---
    if origin_remote != blessed_remote:
        shared = _has_shared_fork_commits(repo, blessed_remote, origin_remote, base)
        # ... rest unchanged
```

- [ ] **Step 5: Run test to verify it passes**

Run: `python3 -m pytest tests/test_branch_create.py::test_sync_main_reconciles_duplicate_commits -v`
Expected: PASS

- [ ] **Step 6: Run full sync-main test suite for regressions**

Run: `python3 -m pytest tests/test_branch_create.py -v -k sync`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-start/branch_create.py tests/test_branch_create.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: pre-merge gate detects and reconciles duplicate commits in sync-main Refs #312"
```

## Batch 3: Audit script

### Task 3: Standalone `audit_duplicate_commits.py`

**Files:**
- Create: `scripts/audit_duplicate_commits.py`
- Test: `tests/test_audit_duplicate_commits.py`

**Interfaces:**
- Consumes: `git` CLI directly (standalone script, does not import branch_create)
- Produces:
  - `audit_repo(repo_path: str) -> list[tuple[str, str, str]]` — returns duplicate pairs
  - CLI output: `REPO=`, `PAIRS=`, `PAIR=`, `TOTAL_PAIRS=`, `STATUS=`
  - Exit code: 0 = clean, 1 = duplicates found

- [ ] **Step 1: Write failing test — audit detects duplicates in a repo**

```python
def test_audit_detects_duplicates(tmp_path):
    """Audit script finds duplicate commits in a single repo."""
    # Create repo with a duplicate commit pair
    repo = tmp_path / "repo"
    repo.mkdir()
    subprocess.run(["git", "init", str(repo)], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "config", "user.name", "Test"], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "config", "user.email", "t@t"], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "checkout", "-b", "main"], capture_output=True)
    (repo / "init.txt").write_text("init")
    subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "commit", "-m", "init"], capture_output=True)

    # Create a branch, add a commit, come back to main and cherry-pick
    subprocess.run(["git", "-C", str(repo), "checkout", "-b", "feature"], capture_output=True)
    (repo / "feat.txt").write_text("feature work")
    subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat: add feature"],
                   capture_output=True, check=True)
    feat_sha = subprocess.run(
        ["git", "-C", str(repo), "rev-parse", "HEAD"],
        capture_output=True, text=True,
    ).stdout.strip()
    subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "cherry-pick", feat_sha],
                   capture_output=True, check=True)
    subprocess.run(["git", "-C", str(repo), "merge", "feature", "--no-edit"],
                   capture_output=True)

    result = subprocess.run(
        [sys.executable, AUDIT_SCRIPT, str(repo)],
        capture_output=True, text=True,
    )
    assert result.returncode == 1
    assert "PAIRS=1" in result.stdout
    assert "STATUS=dirty" in result.stdout
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_audit_duplicate_commits.py::test_audit_detects_duplicates -v`
Expected: FileNotFoundError — script doesn't exist

- [ ] **Step 3: Implement `audit_duplicate_commits.py`**

```python
#!/usr/bin/env python3
"""
audit_duplicate_commits.py — Scan repos for duplicate commits.

Usage:
    python3 scripts/audit_duplicate_commits.py <repo_path>
    python3 scripts/audit_duplicate_commits.py <family_root> --all-repos

Output: KEY=VALUE lines. Exit 0 = clean, 1 = duplicates found.
"""

import subprocess
import sys
from pathlib import Path


def _git(repo, *args):
    result = subprocess.run(
        ["git", "-C", str(repo)] + list(args),
        capture_output=True, text=True,
    )
    return result.returncode == 0, result.stdout.strip()


def _patch_id_for_sha(repo, sha):
    diff = subprocess.run(
        ["git", "-C", str(repo), "diff-tree", "-p", sha],
        capture_output=True, text=True,
    )
    if diff.returncode != 0 or not diff.stdout.strip():
        return None
    pid = subprocess.run(
        ["git", "patch-id", "--stable"],
        input=diff.stdout, capture_output=True, text=True,
    )
    if pid.returncode == 0 and pid.stdout.strip():
        return pid.stdout.strip().split()[0]
    return None


def audit_repo(repo_path: str) -> list[tuple[str, str, str]]:
    ok, log = _git(repo_path, "log", "main", "--format=%H %s")
    if not ok or not log:
        return []

    by_subject: dict[str, list[str]] = {}
    for line in log.splitlines():
        parts = line.split(" ", 1)
        if len(parts) == 2:
            by_subject.setdefault(parts[1], []).append(parts[0])

    pairs = []
    for subj, shas in by_subject.items():
        if len(shas) < 2:
            continue
        pid_map: dict[str, list[str]] = {}
        for sha in shas:
            pid = _patch_id_for_sha(repo_path, sha)
            if pid:
                pid_map.setdefault(pid, []).append(sha)
        for pid, matching in pid_map.items():
            if len(matching) >= 2:
                for i in range(len(matching) - 1):
                    pairs.append((matching[i], matching[i + 1], subj))
    return pairs


def main():
    if len(sys.argv) < 2:
        print(__doc__.strip(), file=sys.stderr)
        sys.exit(1)

    target = Path(sys.argv[1])
    all_repos = "--all-repos" in sys.argv

    if all_repos:
        repos = sorted(p.parent for p in target.rglob(".git") if p.is_dir())
    else:
        repos = [target]

    total_pairs = 0
    has_dupes = False

    for repo in repos:
        pairs = audit_repo(str(repo))
        name = repo.name
        print(f"REPO={name}")
        print(f"PAIRS={len(pairs)}")
        for local, remote, subj in pairs:
            print(f"PAIR={local[:12]}:{remote[:12]}:{subj[:60]}")
        total_pairs += len(pairs)
        if pairs:
            has_dupes = True

    print(f"TOTAL_PAIRS={total_pairs}")
    print(f"STATUS={'dirty' if has_dupes else 'clean'}")
    sys.exit(1 if has_dupes else 0)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_audit_duplicate_commits.py::test_audit_detects_duplicates -v`
Expected: PASS

- [ ] **Step 5: Write failing test — clean repo returns exit 0**

```python
def test_audit_clean_repo(tmp_path):
    repo = tmp_path / "repo"
    repo.mkdir()
    subprocess.run(["git", "init", str(repo)], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "config", "user.name", "Test"], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "config", "user.email", "t@t"], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "checkout", "-b", "main"], capture_output=True)
    (repo / "init.txt").write_text("init")
    subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(repo), "commit", "-m", "init"], capture_output=True)

    result = subprocess.run(
        [sys.executable, AUDIT_SCRIPT, str(repo)],
        capture_output=True, text=True,
    )
    assert result.returncode == 0
    assert "STATUS=clean" in result.stdout
```

- [ ] **Step 6: Run test — should pass immediately**

Run: `python3 -m pytest tests/test_audit_duplicate_commits.py::test_audit_clean_repo -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/audit_duplicate_commits.py tests/test_audit_duplicate_commits.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: standalone audit script for duplicate commit detection Refs #312"
```

## References

- [2026-08-30-duplicate-commit-detection-design.md] — design spec
- [work-start/branch_create.py:149-199] — existing sync-main logic
- [tests/test_branch_create.py:275-289] — test helper patterns
- [git-patch-id(1)] — content-based commit identity
- [GitHub #312] — root cause chain and evidence
- [GitHub #301] — merge-instead-of-rebase prevention

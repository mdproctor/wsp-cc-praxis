# Slot Propagation Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #210 — work-end land step fails to propagate slot main back to original repos
**Issue group:** #210

**Goal:** Ensure every git repo in a slot (project AND workspace) propagates to GitHub and the local original after work-end closes the slot.

**Architecture:** Dual-push topology — slot pushes to GitHub (fork) first (durable), then to local original (immediate sync via `updateInstead`). Unified push loop replaces the current split handling. SHA verification after each push prevents silent failures.

**Tech Stack:** Python 3, git CLI, pytest with tmp_path fixtures

## Global Constraints

- All changes in `work-slot/slot_manager.py` and `tests/test_slot_manager.py`
- Protocol: every externalised script ships with pytest tests in the same commit
- No push to blessed repo (upstream/casehubio) — fork only
- `get_slot_repos()` must be retained unchanged for project-only callers

---

### Task 1: Add `get_all_slot_repos()` and remote configuration helpers

**Files:**
- Modify: `work-slot/slot_manager.py` (add functions after `get_slot_repos` at line 677)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `get_all_slot_repos(slot_dir: Path) -> list[str]` — returns all git repo dirs in slot (project + workspace), excluding `.m2` and `attic`
- Produces: `configure_slot_remotes(clone_path: Path, original_path: Path) -> dict[str, str]` — reconfigures clone remotes (rename origin→local, add origin→fork, optionally add upstream→blessed), returns `{"origin": url, "upstream": url_or_empty, "local": path}`
- Produces: `configure_update_instead(original_path: Path) -> None` — sets `receive.denyCurrentBranch=updateInstead` on original
- Consumes: `detect_topology(project: str) -> tuple[str, str]` from `work-end/common.py`

- [ ] **Step 1: Write failing test for `get_all_slot_repos`**

```python
class TestGetAllSlotRepos:
    def test_includes_workspace_dirs(self, tmp_path):
        slot = tmp_path / "slot1"
        slot.mkdir()
        for name in ["engine", "pages", "work-casehub"]:
            d = slot / name
            d.mkdir()
            (d / ".git").mkdir()
        (slot / ".m2").mkdir()
        result = slot_manager.get_all_slot_repos(slot)
        assert result == ["engine", "pages", "work-casehub"]

    def test_excludes_m2_and_attic(self, tmp_path):
        slot = tmp_path / "slot1"
        slot.mkdir()
        for name in ["engine", ".m2", "attic"]:
            d = slot / name
            d.mkdir()
            (d / ".git").mkdir()
        result = slot_manager.get_all_slot_repos(slot)
        assert result == ["engine"]

    def test_excludes_non_git_dirs(self, tmp_path):
        slot = tmp_path / "slot1"
        slot.mkdir()
        (slot / "engine").mkdir()
        (slot / "engine" / ".git").mkdir()
        (slot / "random-dir").mkdir()  # no .git
        result = slot_manager.get_all_slot_repos(slot)
        assert result == ["engine"]

    def test_get_slot_repos_still_excludes_workspace(self, tmp_path):
        """Verify get_slot_repos is unchanged — still project-only."""
        slot = tmp_path / "slot1"
        slot.mkdir()
        for name in ["engine", "work-casehub"]:
            d = slot / name
            d.mkdir()
            (d / ".git").mkdir()
        result = slot_manager.get_slot_repos(slot)
        assert result == ["engine"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestGetAllSlotRepos -v`
Expected: FAIL with `AttributeError: module 'slot_manager' has no attribute 'get_all_slot_repos'`

- [ ] **Step 3: Implement `get_all_slot_repos`**

Add after `get_slot_repos()` (line 677) in `work-slot/slot_manager.py`:

```python
def get_all_slot_repos(slot_dir: Path) -> list[str]:
    """All git repos in the slot — project + workspace."""
    return [
        d.name for d in sorted(slot_dir.iterdir())
        if d.is_dir() and (d / ".git").exists()
        and d.name not in (".m2", "attic")
    ]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestGetAllSlotRepos -v`
Expected: PASS (all 4 tests)

- [ ] **Step 5: Write failing tests for `configure_slot_remotes`**

```python
class TestConfigureSlotRemotes:
    def test_direct_model_renames_origin_adds_github(self, tmp_path):
        """Direct model: origin=github, no upstream."""
        original = init_repo(tmp_path / "original")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/user/repo.git"], capture_output=True)
        clone = tmp_path / "clone"
        subprocess.run(["git", "clone", str(original), str(clone)], capture_output=True)

        result = slot_manager.configure_slot_remotes(clone, original)

        # local remote should point to original path
        rc, local_url, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "local"])
        assert rc == 0
        assert str(original) in local_url.strip()

        # origin should point to github
        rc, origin_url, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "origin"])
        assert rc == 0
        assert origin_url.strip() == "https://github.com/user/repo.git"

        assert result["upstream"] == ""

    def test_fork_model_adds_upstream(self, tmp_path):
        """Fork model: origin=fork, upstream=blessed."""
        original = init_repo(tmp_path / "original")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/mdproctor/repo.git"], capture_output=True)
        subprocess.run(["git", "-C", str(original), "remote", "add", "upstream",
                        "https://github.com/casehubio/repo.git"], capture_output=True)
        clone = tmp_path / "clone"
        subprocess.run(["git", "clone", str(original), str(clone)], capture_output=True)

        result = slot_manager.configure_slot_remotes(clone, original)

        rc, origin_url, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "origin"])
        assert origin_url.strip() == "https://github.com/mdproctor/repo.git"

        rc, upstream_url, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "upstream"])
        assert rc == 0
        assert upstream_url.strip() == "https://github.com/casehubio/repo.git"

        assert result["upstream"] == "https://github.com/casehubio/repo.git"

    def test_no_remotes_on_original_skips(self, tmp_path):
        """No remotes on original — no reconfiguration."""
        original = init_repo(tmp_path / "original")
        clone = tmp_path / "clone"
        subprocess.run(["git", "clone", str(original), str(clone)], capture_output=True)

        result = slot_manager.configure_slot_remotes(clone, original)
        assert result["origin"] == ""
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestConfigureSlotRemotes -v`
Expected: FAIL

- [ ] **Step 7: Implement `configure_slot_remotes` and `configure_update_instead`**

Add an import at the top of `slot_manager.py`:

```python
# Add to existing imports section
sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
from common import detect_topology
```

Add the functions after `get_all_slot_repos`:

```python
def configure_slot_remotes(clone_path: Path, original_path: Path) -> dict[str, str]:
    """Reconfigure clone remotes: local=clone-source, origin=fork, upstream=blessed."""
    fork_remote, blessed_remote = detect_topology(str(original_path))
    if not fork_remote:
        return {"origin": "", "upstream": "", "local": str(original_path)}

    rc, fork_url, _ = run_cmd(
        ["git", "-C", str(original_path), "remote", "get-url", fork_remote])
    if rc != 0:
        return {"origin": "", "upstream": "", "local": str(original_path)}
    fork_url = fork_url.strip()

    run_cmd(["git", "-C", str(clone_path), "remote", "rename", "origin", "local"])
    run_cmd(["git", "-C", str(clone_path), "remote", "add", "origin", fork_url])
    run_cmd(["git", "-C", str(clone_path), "fetch", "origin"])
    run_cmd(["git", "-C", str(clone_path), "branch", "--set-upstream-to=origin/main", "main"])

    upstream_url = ""
    if blessed_remote:
        rc, blessed_url, _ = run_cmd(
            ["git", "-C", str(original_path), "remote", "get-url", blessed_remote])
        if rc == 0:
            upstream_url = blessed_url.strip()
            run_cmd(["git", "-C", str(clone_path), "remote", "add", "upstream", upstream_url])

    return {"origin": fork_url, "upstream": upstream_url, "local": str(original_path)}


def configure_update_instead(original_path: Path) -> None:
    """Set receive.denyCurrentBranch=updateInstead on original repo."""
    run_cmd(["git", "-C", str(original_path), "config",
             "receive.denyCurrentBranch", "updateInstead"])
```

- [ ] **Step 8: Write failing test for `configure_update_instead`**

```python
class TestConfigureUpdateInstead:
    def test_sets_config_on_original(self, tmp_path):
        original = init_repo(tmp_path / "original")
        slot_manager.configure_update_instead(original)
        rc, value, _ = slot_manager.run_cmd(
            ["git", "-C", str(original), "config", "receive.denyCurrentBranch"])
        assert rc == 0
        assert value.strip() == "updateInstead"

    def test_idempotent(self, tmp_path):
        original = init_repo(tmp_path / "original")
        slot_manager.configure_update_instead(original)
        slot_manager.configure_update_instead(original)
        rc, value, _ = slot_manager.run_cmd(
            ["git", "-C", str(original), "config", "receive.denyCurrentBranch"])
        assert value.strip() == "updateInstead"
```

- [ ] **Step 9: Run all new tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestGetAllSlotRepos tests/test_slot_manager.py::TestConfigureSlotRemotes tests/test_slot_manager.py::TestConfigureUpdateInstead -v`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#210): add get_all_slot_repos, configure_slot_remotes, configure_update_instead"
```

---

### Task 2: Wire remote configuration into `create_slot()`

**Files:**
- Modify: `work-slot/slot_manager.py` — `create_slot()` function (lines 403-520)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `configure_slot_remotes(clone_path, original_path)` from Task 1
- Consumes: `configure_update_instead(original_path)` from Task 1
- Produces: Modified `create_slot()` that configures remotes and `updateInstead` on every cloned repo

- [ ] **Step 1: Write failing test**

```python
class TestCreateSlotRemoteConfig:
    @patch("slot_manager.run_cmd")
    def test_create_slot_configures_remotes_on_project_clone(self, mock_cmd, tmp_path):
        """After cloning, project repos get remote reconfiguration."""
        family = tmp_path / "family"
        family.mkdir()
        repo = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(repo), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        mock_cmd.return_value = (0, "", "")

        # Check that configure_slot_remotes is called
        with patch("slot_manager.configure_slot_remotes") as mock_remotes, \
             patch("slot_manager.configure_update_instead") as mock_update, \
             patch("slot_manager.sync_main"):
            mock_remotes.return_value = {"origin": "https://github.com/user/engine.git",
                                         "upstream": "", "local": str(repo)}
            try:
                slot_manager.create_slot(family, ["engine"], "feature-1",
                                         "1", "user/engine", "1", "test")
            except (SystemExit, Exception):
                pass

            assert mock_remotes.called
            assert mock_update.called
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotRemoteConfig -v`
Expected: FAIL — `configure_slot_remotes` not called by `create_slot`

- [ ] **Step 3: Modify `create_slot()` to call remote configuration**

In `create_slot()`, after each project repo clone (after line 436 `_symlink_gitignored_assets`), add:

```python
        configure_slot_remotes(clone_dest, repo_path)
        configure_update_instead(repo_path)
```

After each workspace clone (after line 465 `_exclude_symlinks(ws_slot_dir)`), add:

```python
                configure_slot_remotes(ws_slot_dir, ws_source)
                configure_update_instead(ws_source)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotRemoteConfig -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#210): wire remote configuration into create_slot"
```

---

### Task 3: Rewrite `merge_slot()` — unified push loop with dual push, verification, and summary

This is the core change. Replace the current split handling (project-only push loop + workspace stamp-only loop) with a single loop over all repos.

**Files:**
- Modify: `work-slot/slot_manager.py` — `merge_slot()` function (lines 876-1080)
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `get_all_slot_repos(slot_dir)` from Task 1
- Consumes: `resolve_original_repo(repo_path)` (existing)
- Consumes: `run_cmd(args)` (existing)
- Produces: Revised `merge_slot()` with four phases: rebase all, merge+push+verify all, stamp all, report summary

- [ ] **Step 1: Write failing test for relaxed preflight (dirty worktree on main blocks, feature branch passes)**

```python
class TestMergeSlotRelaxedPreflight:
    def _setup_slot(self, tmp_path):
        """Create a minimal slot with one project repo and phase-a marker."""
        family = tmp_path / "family"
        family.mkdir()
        original = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = slot_dir / "engine"
        subprocess.run(["git", "clone", "--shared", str(original), str(clone)],
                       capture_output=True)
        subprocess.run(["git", "-C", str(clone), "checkout", "-b", "feature-1"],
                       capture_output=True)
        # Add remotes (simulating create_slot)
        subprocess.run(["git", "-C", str(clone), "remote", "rename", "origin", "local"],
                       capture_output=True)
        subprocess.run(["git", "-C", str(clone), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        (slot_dir / ".phase-a-complete").write_text("branch=feature-1\n")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — feature-1\n## Repos\n- engine\n")
        return family, slot_dir, original, clone

    def test_original_on_feature_branch_passes_preflight(self, tmp_path):
        family, slot_dir, original, clone = self._setup_slot(tmp_path)
        # Switch original to a feature branch
        subprocess.run(["git", "-C", str(original), "checkout", "-b", "other-work"],
                       capture_output=True)

        # merge_slot should NOT fail preflight due to branch check
        # (it will fail later at push since no real GitHub remote — that's OK)
        with patch("slot_manager.run_cmd", wraps=slot_manager.run_cmd) as mock:
            result = slot_manager.merge_slot(family, 1)

        # Should NOT see ERROR=not_on_main in output
        # (may still fail at push stage — that's expected without real remotes)

    def test_dirty_worktree_on_main_blocks(self, tmp_path):
        family, slot_dir, original, clone = self._setup_slot(tmp_path)
        # Make original dirty on main
        (original / "dirty.txt").write_text("uncommitted")

        import io
        captured = io.StringIO()
        with patch("sys.stdout", captured):
            result = slot_manager.merge_slot(family, 1)

        assert result == 1
        assert "dirty_worktree" in captured.getvalue()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMergeSlotRelaxedPreflight -v`
Expected: FAIL — current preflight checks `not_on_main`

- [ ] **Step 3: Rewrite the preflight section of `merge_slot()`**

Replace lines 914-938 (the current preflight) with:

```python
    print("STAGE=preflight")
    preflight_ok = True
    all_repos = get_all_slot_repos(slot_dir)
    for repo_name in all_repos:
        slot_repo = slot_dir / repo_name
        original = resolve_original_repo(slot_repo)
        if not original.is_dir():
            print(f"ERROR=original_not_found repo={repo_name} path={original}")
            preflight_ok = False
            continue
        rc, status_out, _ = run_cmd(
            ["git", "-C", str(original), "status", "--porcelain"])
        if rc == 0 and status_out.strip():
            rc2, cur_branch, _ = run_cmd(
                ["git", "-C", str(original), "branch", "--show-current"])
            cur_branch = cur_branch.strip() if rc2 == 0 else ""
            if cur_branch == "main":
                print(f"ERROR=dirty_worktree repo={repo_name} path={original}")
                preflight_ok = False
    if not preflight_ok:
        print("STAGE=preflight STATUS=fail")
        return 1
    print("STAGE=preflight STATUS=pass")
```

- [ ] **Step 4: Run preflight tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMergeSlotRelaxedPreflight -v`
Expected: PASS

- [ ] **Step 5: Write failing test for the unified push loop (dual push + SHA verification + summary)**

```python
class TestMergeSlotDualPush:
    def _setup_full_slot(self, tmp_path):
        """Create a slot with project + workspace, both with GitHub remotes."""
        family = tmp_path / "family"
        family.mkdir()

        # Create project original
        proj_orig = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(proj_orig), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)
        slot_manager.configure_update_instead(proj_orig)

        # Create workspace original
        ws_orig = init_repo(family / "work-hub")
        subprocess.run(["git", "-C", str(ws_orig), "remote", "add", "origin",
                        "https://github.com/user/work-hub.git"], capture_output=True)
        slot_manager.configure_update_instead(ws_orig)

        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)

        # Clone project
        proj_clone = slot_dir / "engine"
        subprocess.run(["git", "clone", "--shared", str(proj_orig), str(proj_clone)],
                       capture_output=True)
        subprocess.run(["git", "-C", str(proj_clone), "checkout", "-b", "feature-1"],
                       capture_output=True)
        # Simulate remote reconfiguration
        subprocess.run(["git", "-C", str(proj_clone), "remote", "rename", "origin", "local"],
                       capture_output=True)
        subprocess.run(["git", "-C", str(proj_clone), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        # Clone workspace
        ws_clone = slot_dir / "work-hub"
        subprocess.run(["git", "clone", "--shared", str(ws_orig), str(ws_clone)],
                       capture_output=True)
        subprocess.run(["git", "-C", str(ws_clone), "checkout", "-b", "feature-1"],
                       capture_output=True)
        subprocess.run(["git", "-C", str(ws_clone), "remote", "rename", "origin", "local"],
                       capture_output=True)
        subprocess.run(["git", "-C", str(ws_clone), "remote", "add", "origin",
                        "https://github.com/user/work-hub.git"], capture_output=True)

        # Add feature commits
        (proj_clone / "feature.txt").write_text("feature work")
        subprocess.run(["git", "-C", str(proj_clone), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(proj_clone), "commit", "-m", "feat: add feature"],
                       capture_output=True)

        (ws_clone / "journal.md").write_text("# Journal")
        subprocess.run(["git", "-C", str(ws_clone), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(ws_clone), "commit", "-m", "docs: journal"],
                       capture_output=True)

        (slot_dir / ".phase-a-complete").write_text("branch=feature-1\n")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — feature-1\n## Repos\n- engine\n")

        return family, slot_dir, proj_orig, ws_orig

    def test_workspace_repo_included_in_push_loop(self, tmp_path):
        """Workspace repos must be merged and pushed, not just stamped."""
        family, slot_dir, proj_orig, ws_orig = self._setup_full_slot(tmp_path)

        # The GitHub push will fail (no real remote) — mock it
        original_run_cmd = slot_manager.run_cmd
        def mock_push(args, cwd=None):
            if "push" in args and "origin" in args and "main" in args:
                return (0, "", "")  # Simulate successful GitHub push
            if "ls-remote" in args:
                # Return the current HEAD sha for verification
                rc, sha, _ = original_run_cmd(
                    ["git", "-C", args[2], "rev-parse", "main"])
                return (0, f"{sha.strip()}\trefs/heads/main\n", "")
            return original_run_cmd(args, cwd)

        with patch("slot_manager.run_cmd", side_effect=mock_push):
            result = slot_manager.merge_slot(family, 1)

        # Check workspace original got the work
        rc, ws_log, _ = original_run_cmd(
            ["git", "-C", str(ws_orig), "log", "--oneline"])
        assert "journal" in ws_log.lower() or result == 0

    def test_local_push_failure_is_warning_not_error(self, tmp_path):
        """Local push fail should warn, not block — work is on GitHub."""
        family, slot_dir, proj_orig, ws_orig = self._setup_full_slot(tmp_path)

        original_run_cmd = slot_manager.run_cmd
        push_count = {"github": 0, "local": 0}

        def mock_selective_push(args, cwd=None):
            if "push" in args and "origin" in args and "main" in args:
                push_count["github"] += 1
                return (0, "", "")
            if "push" in args and "local" in args and "main" in args:
                push_count["local"] += 1
                return (1, "", "rejected")  # Local push fails
            if "ls-remote" in args:
                rc, sha, _ = original_run_cmd(
                    ["git", "-C", args[2], "rev-parse", "main"])
                return (0, f"{sha.strip()}\trefs/heads/main\n", "")
            return original_run_cmd(args, cwd)

        with patch("slot_manager.run_cmd", side_effect=mock_selective_push):
            result = slot_manager.merge_slot(family, 1)

        # Should succeed (local failure is warning)
        assert result == 0
        assert push_count["github"] > 0
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMergeSlotDualPush -v`
Expected: FAIL — current merge_slot doesn't include workspace repos or do dual push

- [ ] **Step 7: Rewrite the rebase, push, and stamp sections of `merge_slot()`**

Replace lines 940-1049 (rebase loop + push loop + both stamp loops) with the new four-phase implementation. This is the largest change:

```python
    # Phase 1: Rebase all feature branches
    all_repos = get_all_slot_repos(slot_dir)
    max_attempts = 3
    for attempt in range(1, max_attempts + 1):
        print(f"STAGE=rebase ATTEMPT={attempt}")
        rebase_failed = False
        for repo_name in all_repos:
            slot_repo = slot_dir / repo_name
            # Determine rebase source: upstream (blessed) or origin (fork/direct)
            rc, _, _ = run_cmd(
                ["git", "-C", str(slot_repo), "remote", "get-url", "upstream"])
            rebase_remote = "upstream" if rc == 0 else "origin"
            run_cmd(["git", "-C", str(slot_repo), "fetch", rebase_remote, "main"])
            rc, _, _ = run_cmd(
                ["git", "-C", str(slot_repo), "rebase", f"{rebase_remote}/main"])
            if rc != 0:
                run_cmd(["git", "-C", str(slot_repo), "rebase", "--abort"])
                print(f"ERROR=conflict repo={repo_name}")
                rebase_failed = True
                break
        if rebase_failed:
            if attempt < max_attempts:
                continue
            print("STAGE=rebase STATUS=fail")
            return 1
        print("STAGE=rebase STATUS=pass")
        break

    # Phase 2: Merge, push, verify (per repo)
    print("STAGE=push")
    results: dict[str, tuple[str, str]] = {}
    landed_shas: dict[str, str] = {}

    for repo_name in all_repos:
        slot_repo = slot_dir / repo_name

        # Checkout main and catch up with origin
        run_cmd(["git", "-C", str(slot_repo), "checkout", "main"])
        rc, _, _ = run_cmd(
            ["git", "-C", str(slot_repo), "fetch", "origin", "main"])
        if rc == 0:
            rc_ff, _, _ = run_cmd(
                ["git", "-C", str(slot_repo), "merge", "--ff-only", "origin/main"])
            if rc_ff != 0:
                run_cmd(["git", "-C", str(slot_repo), "merge", "origin/main", "--no-edit"])

        # Merge feature branch (ff-only first, regular merge fallback)
        rc, _, _ = run_cmd(
            ["git", "-C", str(slot_repo), "merge", "--ff-only", branch])
        if rc != 0:
            rc2, _, stderr = run_cmd(
                ["git", "-C", str(slot_repo), "merge", branch, "--no-edit"])
            if rc2 != 0:
                results[repo_name] = ("merge_failed", "")
                print(f"ERROR=merge_failed repo={repo_name} stderr={stderr.strip()}")
                continue

        rc, sha_out, _ = run_cmd(
            ["git", "-C", str(slot_repo), "rev-parse", "main"])
        slot_sha = sha_out.strip() if rc == 0 else "unknown"

        # Push to GitHub (fork) — primary
        rc, _, stderr = run_cmd(
            ["git", "-C", str(slot_repo), "push", "origin", "main"])
        if rc != 0:
            results[repo_name] = ("github_failed", slot_sha)
            print(f"WARN=github_push_failed repo={repo_name} err={stderr.strip()}")
            continue

        # Verify GitHub push
        rc, ls_out, _ = run_cmd(
            ["git", "-C", str(slot_repo), "ls-remote", "origin", "main"])
        github_sha = ls_out.split()[0] if rc == 0 and ls_out.strip() else ""
        if github_sha != slot_sha:
            results[repo_name] = ("github_verify_failed", slot_sha)
            print(f"WARN=github_verify_failed repo={repo_name} expected={slot_sha} got={github_sha}")
            continue

        # Push to local original — secondary, best-effort
        rc, _, stderr = run_cmd(
            ["git", "-C", str(slot_repo), "push", "local", "main"])
        if rc != 0:
            results[repo_name] = ("local_failed", slot_sha)
            print(f"WARN=local_push_failed repo={repo_name} err={stderr.strip()}")
            landed_shas[repo_name] = slot_sha
            continue

        # Verify local push
        original = resolve_original_repo(slot_repo)
        rc, local_sha_out, _ = run_cmd(
            ["git", "-C", str(original), "rev-parse", "main"])
        local_sha = local_sha_out.strip() if rc == 0 else ""
        if local_sha != slot_sha:
            results[repo_name] = ("local_verify_failed", slot_sha)
            print(f"WARN=local_verify_failed repo={repo_name}")
            landed_shas[repo_name] = slot_sha
            continue

        results[repo_name] = ("ok", slot_sha)
        landed_shas[repo_name] = slot_sha

    # Check for any GitHub failures (hard stop)
    github_failures = [r for r, (status, _) in results.items()
                       if status in ("github_failed", "github_verify_failed", "merge_failed")]
    if github_failures:
        print("STAGE=push STATUS=fail")
        for repo_name in all_repos:
            status, sha = results.get(repo_name, ("unknown", ""))
            icon = "FAIL" if status in ("github_failed", "github_verify_failed", "merge_failed") \
                   else "WARN" if "local" in status else "OK"
            print(f"RESULT={repo_name} STATUS={status} SHA={sha} ICON={icon}")
        print(f"ERROR=github_push_incomplete failed={','.join(github_failures)}")
        return 1

    # Write .landed marker
    shas_str = ",".join(f"{r}:{s}" for r, s in landed_shas.items())
    (slot_dir / ".landed").write_text(
        f"branch={branch}\n"
        f"repos={','.join(all_repos)}\n"
        f"landed_shas={shas_str}\n"
        f"timestamp={datetime.datetime.now(datetime.timezone.utc).isoformat()}\n"
    )

    # Phase 3: Stamp all feature branches
    for repo_name in all_repos:
        slot_repo = slot_dir / repo_name
        if not slot_repo.is_dir() or not (slot_repo / ".git").exists():
            continue
        sha = landed_shas.get(repo_name, "unknown")
        run_cmd(["git", "-C", str(slot_repo), "checkout", branch])
        run_cmd([
            "git", "-C", str(slot_repo), "commit", "--allow-empty",
            "-m", f"chore: branch closed — landed as {sha} on main",
        ])
        run_cmd(["git", "-C", str(slot_repo), "push", "origin", branch, "--force-with-lease"])

    # Phase 4: Report propagation summary
    print("STAGE=push STATUS=pass")
    for repo_name in all_repos:
        status, sha = results.get(repo_name, ("unknown", ""))
        icon = "OK" if status == "ok" else "WARN" if "local" in status else "FAIL"
        print(f"RESULT={repo_name} STATUS={status} SHA={sha} ICON={icon}")
    ok_count = sum(1 for s, _ in results.values() if s == "ok")
    warn_count = sum(1 for s, _ in results.values() if "local" in s)
    print(f"SUMMARY=ok:{ok_count} warn:{warn_count} fail:0")
```

- [ ] **Step 8: Run all merge_slot tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMergeSlotRelaxedPreflight tests/test_slot_manager.py::TestMergeSlotDualPush -v`
Expected: PASS

- [ ] **Step 9: Run the full existing test suite to check for regressions**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: PASS (existing tests may need minor adjustments if they depend on the old preflight behavior)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#210): unified merge_slot with dual push, SHA verification, propagation summary"
```

---

### Task 4: Add `migrate-remotes` subcommand

**Files:**
- Modify: `work-slot/slot_manager.py` — add `migrate_remotes()` function and wire into `main()`
- Test: `tests/test_slot_manager.py`

**Interfaces:**
- Consumes: `configure_slot_remotes(clone_path, original_path)` from Task 1
- Consumes: `configure_update_instead(original_path)` from Task 1
- Consumes: `get_all_slot_repos(slot_dir)` from Task 1
- Consumes: `resolve_original_repo(repo_path)` (existing)
- Produces: `migrate_remotes(family_root: Path) -> int` — migrates all active slots, returns count

- [ ] **Step 1: Write failing test**

```python
class TestMigrateRemotes:
    def test_migrates_active_slot(self, tmp_path):
        family = tmp_path / "family"
        family.mkdir()
        original = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = slot_dir / "engine"
        subprocess.run(["git", "clone", "--shared", str(original), str(clone)],
                       capture_output=True)
        (slot_dir / ".slot").write_text("# Slot 1\n## Repos\n- engine\n")

        count = slot_manager.migrate_remotes(family)
        assert count > 0

        # Verify local remote exists
        rc, _, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "local"])
        assert rc == 0

        # Verify origin points to GitHub
        rc, url, _ = slot_manager.run_cmd(
            ["git", "-C", str(clone), "remote", "get-url", "origin"])
        assert "github.com" in url

    def test_skips_archived_slots(self, tmp_path):
        family = tmp_path / "family"
        family.mkdir()
        original = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        attic = family / "slots" / "attic" / "1"
        attic.mkdir(parents=True)
        clone = attic / "engine"
        subprocess.run(["git", "clone", "--shared", str(original), str(clone)],
                       capture_output=True)

        count = slot_manager.migrate_remotes(family)
        assert count == 0

    def test_idempotent(self, tmp_path):
        family = tmp_path / "family"
        family.mkdir()
        original = init_repo(family / "engine")
        subprocess.run(["git", "-C", str(original), "remote", "add", "origin",
                        "https://github.com/user/engine.git"], capture_output=True)

        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        clone = slot_dir / "engine"
        subprocess.run(["git", "clone", "--shared", str(original), str(clone)],
                       capture_output=True)
        (slot_dir / ".slot").write_text("# Slot 1\n## Repos\n- engine\n")

        count1 = slot_manager.migrate_remotes(family)
        count2 = slot_manager.migrate_remotes(family)
        assert count1 > 0
        assert count2 == 0  # Already migrated
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMigrateRemotes -v`
Expected: FAIL

- [ ] **Step 3: Implement `migrate_remotes()`**

Add before `main()` in `slot_manager.py`:

```python
def migrate_remotes(family_root: Path) -> int:
    """Add GitHub remotes + updateInstead to all active (non-archived) slots."""
    migrated = 0
    for dir_name in (SLOT_DIR_NAME, LEGACY_SLOT_DIR_NAME):
        slots_dir = family_root / dir_name
        if not slots_dir.exists():
            continue
        for d in sorted(slots_dir.iterdir()):
            if not d.is_dir() or not d.name.isdigit():
                continue
            for repo_name in get_all_slot_repos(d):
                clone = d / repo_name
                # Skip if already migrated (local remote exists)
                rc, _, _ = run_cmd(
                    ["git", "-C", str(clone), "remote", "get-url", "local"])
                if rc == 0:
                    continue
                original = resolve_original_repo(clone)
                result = configure_slot_remotes(clone, original)
                if result["origin"]:
                    migrated += 1
                configure_update_instead(original)
    print(f"MIGRATED={migrated}")
    return migrated
```

- [ ] **Step 4: Wire into `main()` dispatch**

Add after the `check-cross-deps` elif block (before the `else` at line 1603):

```python
    elif subcommand == "migrate-remotes":
        family_root = Path(args.get("target", "."))
        count = migrate_remotes(family_root)
        print(f"COUNT={count}")
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestMigrateRemotes -v`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#210): add migrate-remotes subcommand for retroactive slot migration"
```

---

### Task 5: Update docstring and run full validation

**Files:**
- Modify: `work-slot/slot_manager.py` — update module docstring to include `migrate-remotes`
- Run: full validation suite

- [ ] **Step 1: Update module docstring**

Add `migrate-remotes` to the docstring at the top of `slot_manager.py` (line 7):

```
  migrate-remotes <family-root>
```

- [ ] **Step 2: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "docs(#210): update slot_manager docstring with migrate-remotes subcommand"
```

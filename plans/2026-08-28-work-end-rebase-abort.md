# Work-end Rebase Abort Prevention — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #307 — fix: work-end silently abandons branch content after rebase abort
**Issue group:** #307

**Goal:** Prevent work-end from silently abandoning branch content when rebase fails, with three independent safety layers.

**Architecture:** Add a content-landed postcondition check in `land_flow.py` (after merge+push, before stamp), a hard pre-archive gate in `slot_manager.py` (refuses to archive slots with unmerged content), and worklog DB recording for rebase failures (visibility across fleet).

**Tech Stack:** Python 3, git subprocess, pytest, worklog DB (SQLite)

## Global Constraints

- All source extension lists must match `verify_stamp.py:SOURCE_EXTENSIONS` — single source of truth
- No `--force` override on the pre-archive gate — this is data loss prevention
- Worklog recording failures are non-fatal (warn and continue)
- All new functions need tests committed alongside them (protocol: externalised-scripts-require-tests)

---

## Batch 1: Land postcondition check (D1)

### Task 1: `_verify_content_landed` in land_flow.py

**Files:**
- Modify: `work-end/land_flow.py` — add `_verify_content_landed` function, add `SOURCE_EXTENSIONS` constant, integrate into `land_batch`
- Modify: `tests/test_land_flow.py` — add `TestVerifyContentLanded` class

**Interfaces:**
- Consumes: `_git(repo, *args)` (existing helper in land_flow.py)
- Produces: `_verify_content_landed(desc: RepoDescriptor, branch: str) -> str | None` — returns error string if branch content not on base_branch, else None

- [ ] **Step 1: Write the failing tests**

Add to `tests/test_land_flow.py`:

```python
class TestVerifyContentLanded:
    def test_returns_none_when_content_landed(self, tmp_path):
        """Branch content on main → None (safe to stamp)."""
        from land_flow import RepoDescriptor, Transport, _verify_content_landed
        repo = _init_repo(tmp_path / "repos" / "engine")
        _add_feature(repo, "feat-1", "feature.py")
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", "feat-1"], capture_output=True)
        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        result = _verify_content_landed(desc, "feat-1")
        assert result is None

    def test_returns_error_when_content_not_landed(self, tmp_path):
        """Branch has source files not on main → error string."""
        from land_flow import RepoDescriptor, Transport, _verify_content_landed
        repo = _init_repo(tmp_path / "repos" / "engine")
        _add_feature(repo, "feat-1", "feature.py")
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        # Do NOT merge — branch content is not on main
        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        result = _verify_content_landed(desc, "feat-1")
        assert result is not None
        assert "content_not_landed" in result

    def test_returns_none_for_docs_only_branch(self, tmp_path):
        """Branch with only non-source files → None (no false positives)."""
        from land_flow import RepoDescriptor, Transport, _verify_content_landed
        repo = _init_repo(tmp_path / "repos" / "engine")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "docs-only"], capture_output=True)
        (repo / "docs.md").write_text("# docs\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "docs"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        result = _verify_content_landed(desc, "docs-only")
        assert result is None

    def test_land_batch_fails_when_content_not_landed(self, tmp_path):
        """land_batch returns success=False when postcondition fails."""
        from land_flow import build_branch_batch, land_batch
        repo = _init_repo(tmp_path / "repos" / "engine")
        _add_feature(repo, "feat-1", "feature.py")
        # Simulate merge that doesn't include branch content:
        # checkout main, do NOT merge feat-1, but push main
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        progress_file = tmp_path / ".execute-progress"
        descs = build_branch_batch(repo, None, "feat-1", "main")
        result = land_batch(descs, "feat-1", progress_file)
        # The merge step will either fail (no ff) or the postcondition will catch it
        assert not result.success or any(s.error for s in result.repos)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_flow.py::TestVerifyContentLanded -v`
Expected: ImportError for `_verify_content_landed` — function doesn't exist yet

- [ ] **Step 3: Add SOURCE_EXTENSIONS and implement `_verify_content_landed`**

Add to `work-end/land_flow.py` after the `_git` helper (around line 77):

```python
SOURCE_EXTENSIONS = (
    ".java", ".kt", ".xml", ".yaml", ".yml", ".json",
    ".properties", ".sql", ".py", ".ts", ".tsx", ".js",
    ".jsx", ".css", ".scss", ".html",
)


def _verify_content_landed(desc: RepoDescriptor, branch: str) -> str | None:
    """Return error string if branch source content is not on base_branch."""
    result = _git(desc.repo_path, "diff", "--name-only",
                  f"{desc.base_branch}...{branch}")
    if result.returncode != 0:
        return f"diff_failed repo={desc.repo_path.name}"
    source_files = [f for f in result.stdout.strip().split("\n")
                    if f and any(f.endswith(ext) for ext in SOURCE_EXTENSIONS)]
    if not source_files:
        return None
    diff = _git(desc.repo_path, "diff", desc.base_branch, branch,
                "--", *source_files)
    if diff.returncode == 0 and diff.stdout.strip():
        missing = [f for f in _git(desc.repo_path, "diff", "--name-only",
                   desc.base_branch, branch, "--", *source_files)
                   .stdout.strip().split("\n") if f]
        return f"content_not_landed repo={desc.repo_path.name} files={len(missing)}"
    return None
```

- [ ] **Step 4: Integrate into `land_batch` — add postcondition check after merge+push, before stamp**

In `land_flow.py:land_batch()`, after the merge+push loop (after line 655 `print("STAGE=push STATUS=pass")`), add:

```python
    # Step 3b: Verify content landed (postcondition)
    print("STAGE=verify_content")
    for desc in active:
        if desc.is_workspace:
            continue
        if desc.repo_path.name in failed_repos:
            continue
        err = _verify_content_landed(desc, branch)
        if err:
            result.repos.append(RepoStatus(
                repo_path=desc.repo_path, error="content_not_landed",
            ))
            result.success = False
            print(f"CONTENT_NOT_LANDED={desc.repo_path.name}")
    if not result.success:
        print("STAGE=verify_content STATUS=fail")
        return result
    print("STAGE=verify_content STATUS=pass")
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_flow.py::TestVerifyContentLanded -v`
Expected: All 4 tests PASS

- [ ] **Step 6: Run full land_flow test suite for regressions**

Run: `python3 -m pytest tests/test_land_flow.py -v`
Expected: All 31+ tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/land_flow.py tests/test_land_flow.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: land postcondition — verify branch content before stamp Refs #307"
```

---

## Batch 2: Pre-archive gate (D2)

### Task 2: `_has_unmerged_content` in slot_manager.py

**Files:**
- Modify: `work-slot/slot_manager.py` — add `_has_unmerged_content` function, integrate into `archive_slot`
- Modify: `tests/test_slot_manager.py` — add `TestHasUnmergedContent` class

**Interfaces:**
- Consumes: `run_cmd(args, cwd)` (existing helper in slot_manager.py)
- Produces: `_has_unmerged_content(slot_dir: Path) -> list[str]` — returns list of repo names with unmerged branch content

- [ ] **Step 1: Write the failing tests**

Add to `tests/test_slot_manager.py` (after the `TestRepackBrokenAlternates` class):

```python
class TestHasUnmergedContent:
    """_has_unmerged_content detects repos with branch content not on main."""

    def test_detects_unmerged_branch_content(self, tmp_path):
        slots = tmp_path / "slots"
        slot = slots / "10"
        repo = init_repo(slot / "engine")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "feat-1"],
                        capture_output=True, check=True)
        (repo / "feature.py").write_text("# feature\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat"],
                        capture_output=True, check=True)

        result = slot_manager._has_unmerged_content(slot)

        assert result == ["engine"]

    def test_returns_empty_when_on_main(self, tmp_path):
        slots = tmp_path / "slots"
        slot = slots / "10"
        init_repo(slot / "engine")

        result = slot_manager._has_unmerged_content(slot)

        assert result == []

    def test_returns_empty_when_branch_content_merged(self, tmp_path):
        slots = tmp_path / "slots"
        slot = slots / "10"
        repo = init_repo(slot / "engine")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "feat-1"],
                        capture_output=True, check=True)
        (repo / "feature.py").write_text("# feature\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat"],
                        capture_output=True, check=True)
        subprocess.run(["git", "-C", str(repo), "checkout", "main"],
                        capture_output=True, check=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", "feat-1"],
                        capture_output=True, check=True)
        subprocess.run(["git", "-C", str(repo), "checkout", "feat-1"],
                        capture_output=True, check=True)

        result = slot_manager._has_unmerged_content(slot)

        assert result == []

    def test_reports_multiple_repos_with_unmerged(self, tmp_path):
        slots = tmp_path / "slots"
        slot = slots / "10"
        for name in ["engine", "worker"]:
            repo = init_repo(slot / name)
            subprocess.run(["git", "-C", str(repo), "checkout", "-b", "feat-1"],
                            capture_output=True, check=True)
            (repo / "feature.py").write_text(f"# {name}\n")
            subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
            subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat"],
                            capture_output=True, check=True)

        result = slot_manager._has_unmerged_content(slot)

        assert sorted(result) == ["engine", "worker"]

    def test_archive_slot_refuses_unmerged_content(self, tmp_path, capsys):
        slots = tmp_path / "slots"
        slot = slots / "10"
        repo = init_repo(slot / "engine")
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "feat-1"],
                        capture_output=True, check=True)
        (repo / "feature.py").write_text("# feature\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat"],
                        capture_output=True, check=True)
        (slot / ".slot").write_text("## State\nbranch: feat-1\n")
        (slot / ".landed").write_text("landed_shas=engine:abc123\n")

        with pytest.raises(SystemExit):
            slot_manager.archive_slot(tmp_path, 10, force=True)

        out = capsys.readouterr().out
        assert "ERROR=unmerged_content" in out
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestHasUnmergedContent -v`
Expected: ImportError for `_has_unmerged_content`

- [ ] **Step 3: Implement `_has_unmerged_content`**

Add to `work-slot/slot_manager.py` after `_repack_broken_alternates` (after line 296):

```python
def _has_unmerged_content(slot_dir: Path) -> list[str]:
    """Return list of repo names with unmerged branch content vs main."""
    unmerged = []
    for repo_dir in sorted(slot_dir.iterdir()):
        if not repo_dir.is_dir() or not (repo_dir / ".git").exists():
            continue
        rc, stdout, _ = run_cmd(
            ["git", "branch", "--show-current"], cwd=str(repo_dir))
        if rc != 0:
            continue
        branch = stdout.strip()
        if not branch or branch == "main":
            continue
        rc, stdout, _ = run_cmd(
            ["git", "diff", "--stat", f"main...{branch}"], cwd=str(repo_dir))
        if rc == 0 and stdout.strip():
            unmerged.append(repo_dir.name)
    return unmerged
```

- [ ] **Step 4: Integrate into `archive_slot` — add hard gate**

In `slot_manager.py:archive_slot()`, after the `ensure_clone_layout(slot_dir)` call (line 1642) and before the `.landed` check, add:

```python
    unmerged = _has_unmerged_content(slot_dir)
    if unmerged:
        print(f"ERROR=unmerged_content slot={slot_num}")
        print(f"ERROR_DETAIL=repos with unmerged branch content: {', '.join(unmerged)}")
        print("HINT=land the branch content first, or manually verify it's already on main under different SHAs")
        sys.exit(1)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestHasUnmergedContent -v`
Expected: All 5 tests PASS

- [ ] **Step 6: Run full slot_manager archive test suite for regressions**

Run: `python3 -m pytest tests/test_slot_manager.py::TestArchiveSlot tests/test_slot_manager.py::TestArchiveSlotDoubleArchive tests/test_slot_manager.py::TestArchiveSlotCleanup tests/test_slot_manager.py::TestArchiveSlotPromotionGate -v`
Expected: All existing tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: pre-archive gate — refuse unmerged branch content Refs #307"
```

---

## Batch 3: Worklog recording for rebase failures (D4)

### Task 3: Record rebase failures in worklog DB

**Files:**
- Modify: `work-end/land_flow.py` — add worklog recording at rebase failure return
- Modify: `work-end/work_end_execute.py` — add worklog recording in `cmd_rebase`
- Modify: `tests/test_land_flow.py` — add test for worklog recording
- Modify: `tests/test_work_end_execute.py` — add test for worklog recording

**Interfaces:**
- Consumes: `_wl.record_close_event()` or a new `_wl.record_rebase_failure()` (worklog API)
- Produces: Worklog event with `repo_path`, `branch`, `commit_count`, `main_ahead`, `error_detail`

- [ ] **Step 1: Write the failing tests**

Add to `tests/test_land_flow.py`:

```python
class TestRebaseFailureWorklog:
    def test_rebase_failure_records_worklog_event(self, tmp_path, monkeypatch):
        """Rebase failure records event in worklog DB."""
        from land_flow import land_batch, build_branch_batch
        recorded = []

        class FakeWl:
            @staticmethod
            def connect():
                return FakeConn()
            @staticmethod
            def record_rebase_failure(conn, **kwargs):
                recorded.append(kwargs)

        class FakeConn:
            def close(self):
                pass

        import land_flow
        monkeypatch.setattr(land_flow, "_wl", FakeWl())

        repo = _init_repo(tmp_path / "repos" / "engine")
        _add_feature(repo, "feat-1", "feature.py")
        # Create a conflict on main
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        (repo / "feature.py").write_text("# conflict\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "conflict"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "checkout", "feat-1"], capture_output=True)

        descs = build_branch_batch(repo, None, "feat-1", "main")
        progress_file = tmp_path / ".execute-progress"
        result = land_batch(descs, "feat-1", progress_file)

        assert not result.success
        assert len(recorded) == 1
        assert recorded[0]["branch"] == "feat-1"
        assert "repo_path" in recorded[0]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_land_flow.py::TestRebaseFailureWorklog -v`
Expected: FAIL — `record_rebase_failure` not called yet

- [ ] **Step 3: Add worklog recording to `land_flow.py:land_batch()`**

In `land_flow.py`, add a helper after `_record_worklog_end` (around line 36):

```python
def _record_rebase_failure(repo_path: str, branch: str,
                           commit_count: int, main_ahead: int,
                           error_detail: str) -> None:
    if not _wl:
        return
    try:
        conn = _wl.connect()
        _wl.record_rebase_failure(
            conn, repo_path=repo_path, branch=branch,
            commit_count=commit_count, main_ahead=main_ahead,
            error_detail=error_detail[:500],
        )
        conn.close()
    except Exception:
        pass
```

In `land_batch()`, at the rebase failure return (line 618-622), before `return result`:

```python
            # Record rebase failure for fleet visibility
            branch_count = _git(failed_desc.repo_path, "rev-list", "--count",
                                f"{failed_desc.base_branch}..{branch}")
            main_count = _git(failed_desc.repo_path, "rev-list", "--count",
                              f"{branch}..{failed_desc.base_branch}")
            _record_rebase_failure(
                repo_path=str(failed_desc.repo_path),
                branch=branch,
                commit_count=int(branch_count.stdout.strip()) if branch_count.returncode == 0 else 0,
                main_ahead=int(main_count.stdout.strip()) if main_count.returncode == 0 else 0,
                error_detail="rebase_conflict_after_3_retries",
            )
```

- [ ] **Step 4: Add worklog recording to `work_end_execute.py:cmd_rebase()`**

In `work_end_execute.py`, add the import at the top (after existing imports):

```python
_lib = Path.home() / ".claude" / "lib"
if _lib.exists():
    sys.path.insert(0, str(_lib))
try:
    import worklog as _wl
except ImportError:
    _wl = None
```

In `cmd_rebase()`, at the rebase conflict return (line 228 and 235), before `return 1`:

```python
        if _wl:
            try:
                conn = _wl.connect()
                branch_count = git(project, "rev-list", "--count", f"{base_branch}..{branch}")
                main_count = git(project, "rev-list", "--count", f"{branch}..{base_branch}")
                _wl.record_rebase_failure(
                    conn, repo_path=project, branch=branch,
                    commit_count=int(branch_count.stdout.strip()) if branch_count.returncode == 0 else 0,
                    main_ahead=int(main_count.stdout.strip()) if main_count.returncode == 0 else 0,
                    error_detail=result.stderr.strip()[:500],
                )
                conn.close()
            except Exception:
                pass
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_flow.py::TestRebaseFailureWorklog -v`
Expected: PASS

- [ ] **Step 6: Run full test suites for regressions**

Run: `python3 -m pytest tests/test_land_flow.py tests/test_work_end_execute.py -v`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/land_flow.py work-end/work_end_execute.py tests/test_land_flow.py tests/test_work_end_execute.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: record rebase failures in worklog DB for fleet visibility Refs #307"
```

---

## References

- [2026-08-28-work-end-rebase-abort-design.md] — design spec this plan implements
- [work-end/land_flow.py:548-683] — land_batch pipeline
- [work-end/work_end_execute.py:198-240] — cmd_rebase
- [work-end/verify_stamp.py:30-35] — SOURCE_EXTENSIONS (canonical list)
- [work-slot/slot_manager.py:1636-1741] — archive_slot
- [work-slot/slot_manager.py:257-296] — _repack_broken_alternates (sibling pattern)
- [tests/test_land_flow.py] — existing test patterns
- [tests/test_slot_manager.py:3826-3911] — TestRepackBrokenAlternates (sibling test pattern)
- [GitHub #307] — focal issue
- [protocol: externalised-scripts-require-tests] — scripts must ship with tests
- [protocol: evidence-before-claims] — verify before claiming done

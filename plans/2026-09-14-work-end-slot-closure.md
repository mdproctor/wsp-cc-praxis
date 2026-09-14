# Work-End Slot Closure Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #368 — work-end stalls on multi-repo main-based slots — incomplete closure
**Issue group:** #368

**Goal:** Fix the two root causes that leave multi-repo slots in a half-closed state: mechanical step failures creating a dead end, and verify checking all repos instead of covered repos.

**Architecture:** Two independent fixes. Fix 1 changes the orchestrator engine's mechanical step error handling to auto-skip after MAX retries (implementing the original spec D10 policy). Fix 2 scopes `verify_slot_close.py` to only check repos listed in the `.slot` file's `Covers:` line.

**Tech Stack:** Python 3.14, pytest, work-end orchestrator scripts

## Global Constraints

- All modified scripts require tests (protocol: `externalised-scripts-require-tests`)
- Evidence before claims at every completion boundary (protocol: `evidence-before-claims`)
- `skipped_error` is a new terminal state alongside `done` and `skipped`
- No changes to lifecycle state machine or individual execution scripts

---

## Batch 1: Auto-skip mechanical failures

### Task 1: Add MAX_MECHANICAL_RETRIES and update done() to recognize skipped_error

**Files:**
- Modify: `work-end/shared_steps.py:58` (add constant)
- Modify: `work-end/shared_steps.py:50-51` (update `OrchestratorContextBase.done()`)
- Modify: `work-end/work_end_orchestrator.py:182-183` (update `OrchestratorContext.done()`)
- Modify: `work-end/work_end_orchestrator.py:185-188` (update `per_repo_done()`)
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Produces: `MAX_MECHANICAL_RETRIES = 3` constant; `done()` and `per_repo_done()` recognizing `"skipped_error"` as terminal

- [ ] **Step 1: Write the failing tests**

```python
class TestSkippedErrorRecognition:
    """skipped_error is a terminal state like done and skipped."""

    def test_done_recognizes_skipped_error(self, tmp_path, monkeypatch):
        monkeypatch.setattr("work_end_orchestrator._run_script", lambda cmd, ws, **kw: {})
        from work_end_orchestrator import OrchestratorContext
        ctx = OrchestratorContext(
            workspace=tmp_path, project=tmp_path / "project",
            branch="test", base_branch="main", meta_state="closing:review",
            on_main=False, in_slot=False, covers="", issue_repo="",
            progress={"promote": "skipped_error"},
        )
        assert ctx.done("promote") is True

    def test_per_repo_done_with_mixed_skipped_error(self, tmp_path, monkeypatch):
        monkeypatch.setattr("work_end_orchestrator._run_script", lambda cmd, ws, **kw: {})
        from work_end_orchestrator import OrchestratorContext
        ctx = OrchestratorContext(
            workspace=tmp_path, project=tmp_path / "project",
            branch="test", base_branch="main", meta_state="closing:promoted",
            on_main=False, in_slot=True, covers="", issue_repo="",
            progress={
                "land:engine": "done",
                "land:work": "skipped_error",
            },
            slot_repos=["engine", "work"],
        )
        assert ctx.per_repo_done("land") is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestSkippedErrorRecognition -v`
Expected: FAIL — `skipped_error` not in `("done", "skipped")`

- [ ] **Step 3: Implement the changes**

In `work-end/shared_steps.py`, add constant after line 58 and update `done()`:

```python
MAX_JUDGMENT_RETRIES = 3
MAX_MECHANICAL_RETRIES = 3

# ...

class OrchestratorContextBase:
    # ...
    def done(self, step: str) -> bool:
        return self.progress.get(step) in ("done", "skipped", "skipped_error")
```

In `work-end/work_end_orchestrator.py`, update `OrchestratorContext.done()`:

```python
def done(self, step: str) -> bool:
    return self.progress.get(step) in ("done", "skipped", "skipped_error")
```

And update `per_repo_done()`:

```python
def per_repo_done(self, step: str) -> bool:
    if not self.in_slot or not self.slot_repos:
        return self.done(step)
    return all(
        self.progress.get(f"{step}:{repo}") in ("done", "skipped", "skipped_error")
        for repo in self.slot_repos
    )
```

Import `MAX_MECHANICAL_RETRIES` in `work_end_orchestrator.py`:

```python
from shared_steps import StepDef, MAX_JUDGMENT_RETRIES, MAX_MECHANICAL_RETRIES
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestSkippedErrorRecognition -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/shared_steps.py work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat(#368): add MAX_MECHANICAL_RETRIES, recognize skipped_error as terminal state  Refs #368"
```

---

### Task 2: Auto-skip in orchestrator_engine.py run_loop()

**Files:**
- Modify: `work-end/orchestrator_engine.py:19` (import MAX_MECHANICAL_RETRIES)
- Modify: `work-end/orchestrator_engine.py:224-244` (auto-skip logic in run_loop)
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes: `MAX_MECHANICAL_RETRIES` from shared_steps; `"skipped_error"` state from Task 1
- Produces: `run_loop()` auto-skips mechanical steps after MAX_MECHANICAL_RETRIES, continues loop

- [ ] **Step 1: Write the failing test**

```python
class TestMechanicalAutoSkip:
    """Mechanical steps auto-skip after MAX retries (spec D10)."""

    def test_mechanical_step_auto_skips_after_max_retries(self, tmp_path, monkeypatch):
        """After MAX_MECHANICAL_RETRIES, mechanical step marked skipped_error, loop continues."""
        from close_progress import update_close_progress
        # Set attempt counter to MAX - 1 (next failure triggers auto-skip)
        update_close_progress(tmp_path, "promote_mechanical_attempt", "2")
        # promote not done yet
        update_close_progress(tmp_path, "report_init", "done")
        # Skip review steps
        for sub in ["code_review", "branch_audit_conformance", "branch_audit_coherence",
                     "branch_audit_structure", "branch_audit_robustness",
                     "loose_ends", "forcing_function", "sweep_config",
                     "forage", "protocol", "update_claude_md",
                     "impl_doc_sync", "doc_freshness_gate", "adr", "write_content",
                     "review_pass"]:
            update_close_progress(tmp_path, sub, "done")
        update_close_progress(tmp_path, "sweep_selected", "")

        # Make promote fail
        def failing_script(cmd, ws, **kw):
            return {"ERROR": "worktree_failed", "ERROR_DETAIL": "test failure"}

        monkeypatch.setattr("work_end_orchestrator._run_script", failing_script)
        from work_end_orchestrator import run_orchestrator
        result = run_orchestrator({
            "workspace": str(tmp_path),
            "project": str(tmp_path / "project"),
            "branch": "issue-368-test",
            "base_branch": "main",
            "meta_state": "closing:verified",
        })
        # Should NOT be user_input/step_failed — should have auto-skipped
        # and continued to next step (report_promote or trajectory)
        assert result.get("ACTION") != "user_input" or result.get("CONTEXT") != "step_failed"

        # Verify promote marked as skipped_error in progress
        from close_progress import read_close_progress
        progress = read_close_progress(tmp_path)
        assert progress.get("promote") == "skipped_error"

    def test_judgment_step_still_escalates_to_user(self, tmp_path, monkeypatch):
        """Judgment steps still yield step_failed — auto-skip is mechanical only."""
        from close_progress import update_close_progress
        update_close_progress(tmp_path, "report_init", "done")
        update_close_progress(tmp_path, "code_review_attempt", "3")

        monkeypatch.setattr("work_end_orchestrator._run_script", lambda cmd, ws, **kw: {})
        from work_end_orchestrator import run_orchestrator
        result = run_orchestrator({
            "workspace": str(tmp_path),
            "project": str(tmp_path / "project"),
            "branch": "issue-368-test",
            "base_branch": "main",
            "meta_state": "closing:review",
        })
        assert result["ACTION"] == "user_input"
        assert result["CONTEXT"] == "step_failed"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestMechanicalAutoSkip -v`
Expected: FAIL — `test_mechanical_step_auto_skips_after_max_retries` fails because promote returns `user_input step_failed`

- [ ] **Step 3: Implement auto-skip in orchestrator_engine.py**

In `work-end/orchestrator_engine.py`, update the import:

```python
from shared_steps import StepDef, MAX_JUDGMENT_RETRIES, MAX_MECHANICAL_RETRIES
```

Replace the mechanical step error handling in `run_loop()` (lines ~234-244):

```python
            if result and "ERROR" in result:
                if on_mechanical_error:
                    override = on_mechanical_error(step, ctx, result)
                    if override is not None:
                        ctx.steps_executed.append(f"{step.name}:ERROR:classified")
                        return override
                attempt += 1
                update_close_progress(ctx.workspace, attempt_key, str(attempt))
                if attempt >= MAX_MECHANICAL_RETRIES:
                    update_close_progress(ctx.workspace, step.name, "skipped_error")
                    ctx.steps_executed.append(f"{step.name}:SKIPPED_ERROR")
                    continue
                ctx.steps_executed.append(f"{step.name}:ERROR:{attempt}")
                return _make_error_result(step.name, attempt, result)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestMechanicalAutoSkip -v`
Expected: PASS

- [ ] **Step 5: Run full orchestrator test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git add work-end/orchestrator_engine.py tests/test_work_end_orchestrator.py
git commit -m "feat(#368): auto-skip mechanical steps after MAX retries (spec D10)  Refs #368"
```

---

### Task 3: Auto-skip for per-repo mechanical steps in slot mode

**Files:**
- Modify: `work-end/work_end_orchestrator.py:1276-1310` (`_close_per_repo_mechanical`)
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes: `MAX_MECHANICAL_RETRIES` from Task 1; auto-skip pattern from Task 2
- Produces: per-repo mechanical steps auto-skip individual repos after MAX retries

- [ ] **Step 1: Write the failing test**

```python
class TestPerRepoMechanicalAutoSkip:
    """Per-repo mechanical steps auto-skip individual repos after MAX retries."""

    def test_per_repo_auto_skip_on_max_retries(self, tmp_path, monkeypatch):
        from close_progress import update_close_progress
        # Set up: engine's land has failed 2 times already
        update_close_progress(tmp_path, "land:engine_mechanical_attempt", "2")
        # work's land is done
        update_close_progress(tmp_path, "land:work", "done")
        # Skip everything before land
        for step in ["report_init", "code_review", "branch_audit_conformance",
                     "branch_audit_coherence", "branch_audit_structure",
                     "branch_audit_robustness", "loose_ends", "forcing_function",
                     "sweep_config", "review_pass", "promote", "report_promote",
                     "promote_pass", "trajectory", "rebase:engine", "rebase:work",
                     "report_rebase", "squash", "report_squash", "write_marker"]:
            update_close_progress(tmp_path, step, "done")
        update_close_progress(tmp_path, "sweep_selected", "")

        slot_path = tmp_path / "slot"
        slot_path.mkdir()
        for repo in ["engine", "work"]:
            repo_dir = slot_path / repo
            repo_dir.mkdir()
            (repo_dir / ".git").mkdir()

        (slot_path / ".slot").write_text(
            "# Slot\n## Repos\n- engine (primary)\n- work\n## Status\nstatus: active\n"
        )

        call_count = {"engine": 0}
        def failing_land(cmd, ws, **kw):
            if "land" in str(cmd) and "engine" in str(cmd):
                call_count["engine"] += 1
                return {"ERROR": "push_failed", "ERROR_DETAIL": "test"}
            return {"LANDED_SHA": "abc123"}

        monkeypatch.setattr("work_end_orchestrator._run_script", failing_land)

        from work_end_orchestrator import run_orchestrator
        result = run_orchestrator({
            "workspace": str(tmp_path),
            "project": str(slot_path / "engine"),
            "branch": "issue-368-test",
            "base_branch": "main",
            "meta_state": "closing:promoted",
            "in_slot": "yes",
            "slot_path": str(slot_path),
        })

        # engine's land should be auto-skipped, not blocking
        from close_progress import read_close_progress
        progress = read_close_progress(tmp_path)
        assert progress.get("land:engine") == "skipped_error"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestPerRepoMechanicalAutoSkip -v`
Expected: FAIL

- [ ] **Step 3: Implement per-repo auto-skip**

In `work-end/work_end_orchestrator.py`, update `_close_per_repo_mechanical()` — replace the retryable failure handling block (around line 1307-1310):

```python
    if retryable_failure:
        step_key, attempt, result = retryable_failure
        if attempt >= MAX_MECHANICAL_RETRIES:
            update_close_progress(ctx.workspace, step_key, "skipped_error")
            ctx.steps_executed.append(f"{step_key}:SKIPPED_ERROR")
        else:
            from orchestrator_engine import _make_error_result
            return _make_error_result(step_key, attempt, result)
```

Also ensure `MAX_MECHANICAL_RETRIES` is imported at the top of the file (should already be done from Task 1).

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestPerRepoMechanicalAutoSkip -v`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v`
Expected: All pass

- [ ] **Step 6: Commit**

```bash
git add work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat(#368): per-repo mechanical auto-skip in slot mode  Refs #368"
```

---

### Task 4: Surface skipped_error in progress summary

**Files:**
- Modify: `work-end/progress_summary.py:269-283` (`_build_rows` — add skipped_error case)
- Test: `tests/test_work_end_orchestrator.py` (or existing progress_summary tests if they exist)

**Interfaces:**
- Consumes: `skipped_error` state from Tasks 1-3
- Produces: Close report shows "error-skipped" with retry detail for auto-skipped steps

- [ ] **Step 1: Write the failing test**

```python
class TestProgressSummarySkippedError:
    def test_skipped_error_shows_in_summary(self):
        from progress_summary import format_summary
        progress = {
            "code_review": "done",
            "code_review_produced": "0",
            "promote": "skipped_error",
            "promote_mechanical_attempt": "3",
            "land": "done",
        }
        summary = format_summary(progress, "close")
        assert "error-skipped" in summary or "skipped_error" in summary.lower()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestProgressSummarySkippedError -v`
Expected: FAIL — `skipped_error` not handled in `_build_rows`

- [ ] **Step 3: Implement skipped_error rendering**

In `work-end/progress_summary.py`, add the `skipped_error` case in `_build_rows()` after the `skipped` case (around line 279):

```python
        elif status == "skipped":
            status_text = "skipped"
            detail = _sweep_detail(step_name, progress, sweep_key, sweep_steps) or ""
        elif status == "skipped_error":
            status_text = "error-skipped"
            detail = _retry_detail(step_name, progress) or "mechanical failure"
        else:
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestProgressSummarySkippedError -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/progress_summary.py tests/test_work_end_orchestrator.py
git commit -m "feat(#368): surface skipped_error in close report  Refs #368"
```

---

## Batch 2: Scope verify to covered repos

### Task 5: Add _parse_covers_repos and scope _resolve_original_repos

**Files:**
- Modify: `work-end/verify_slot_close.py:191-248` (add `_parse_covers_repos`, modify `_resolve_original_repos`)
- Modify: `work-end/verify_slot_close.py:286-298` (modify `check_landed_completeness`)
- Modify: `work-end/verify_slot_close.py:397-469` (modify `verify()` and `main()`)
- Test: `tests/test_verify_slot_close.py`

**Interfaces:**
- Produces: `_parse_covers_repos()` returning set of repo names; scoped `_resolve_original_repos()` and `check_landed_completeness()`

- [ ] **Step 1: Write the failing tests**

```python
class TestParseCoverRepos:
    def test_parses_covers_line(self, tmp_path):
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        from verify_slot_close import _parse_covers_repos

        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text(
            "# Slot 181\n"
            "slug: test-slot\n"
            "\n"
            "## Issue\n"
            "casehubio/parent#469\n"
            "Covers: platform:276,engine:1049,work:394\n"
            "\n"
            "## Repos\n"
            "- platform (primary)\n"
            "- engine\n"
            "- work\n"
            "- aml\n"
            "- clinical\n"
        )
        result = _parse_covers_repos(str(slot_dir))
        assert result == {"platform", "engine", "work"}

    def test_missing_covers_returns_empty(self, tmp_path):
        from verify_slot_close import _parse_covers_repos
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text("# Slot\n## Repos\n- engine\n")
        result = _parse_covers_repos(str(slot_dir))
        assert result == set()

    def test_no_slot_file_returns_empty(self, tmp_path):
        from verify_slot_close import _parse_covers_repos
        result = _parse_covers_repos(str(tmp_path / "nonexistent"))
        assert result == set()


class TestResolveOriginalReposScoped:
    def _make_slot_with_repos(self, tmp_path, repos, covers_repos=None):
        slot_dir = tmp_path / "slot"
        slot_dir.mkdir(exist_ok=True)
        for name in repos:
            repo = slot_dir / name
            repo.mkdir(exist_ok=True)
            git_dir = repo / ".git"
            git_dir.mkdir(exist_ok=True)
            # Mock local remote by creating config
            subprocess.run(
                ["git", "init", str(repo)],
                capture_output=True,
            )
            subprocess.run(
                ["git", "-C", str(repo), "remote", "add", "local", str(tmp_path / "original" / name)],
                capture_output=True,
            )
            orig = tmp_path / "original" / name
            orig.mkdir(parents=True, exist_ok=True)
        return slot_dir

    def test_scoped_to_covers(self, tmp_path):
        from verify_slot_close import _resolve_original_repos
        slot_dir = self._make_slot_with_repos(tmp_path, ["engine", "work", "aml"])
        result = _resolve_original_repos(str(slot_dir), covers_repos={"engine", "work"})
        assert "engine" in result
        assert "work" in result
        assert "aml" not in result

    def test_no_covers_returns_all(self, tmp_path):
        from verify_slot_close import _resolve_original_repos
        slot_dir = self._make_slot_with_repos(tmp_path, ["engine", "work", "aml"])
        result = _resolve_original_repos(str(slot_dir))
        assert "engine" in result
        assert "work" in result
        assert "aml" in result


class TestLandedCompletenessScoped:
    def test_scoped_completeness(self, tmp_path):
        from verify_slot_close import check_landed_completeness

        slot_dir = tmp_path / "slot"
        slot_dir.mkdir()
        (slot_dir / ".slot").write_text("## Repos\n- engine\n- work\n- aml\n")
        (slot_dir / ".landed").write_text("landed_shas=engine:abc,work:def\n")

        # Without scoping: aml is missing → fail
        result_unscoped = check_landed_completeness(str(slot_dir))
        assert result_unscoped["status"] == "fail"

        # With scoping: only engine and work expected → pass
        result_scoped = check_landed_completeness(str(slot_dir), covers_repos={"engine", "work"})
        assert result_scoped["status"] == "pass"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestParseCoverRepos tests/test_verify_slot_close.py::TestResolveOriginalReposScoped tests/test_verify_slot_close.py::TestLandedCompletenessScoped -v`
Expected: FAIL — functions don't exist or don't accept `covers_repos` parameter

- [ ] **Step 3: Implement the changes**

In `work-end/verify_slot_close.py`:

Add `_parse_covers_repos()` after `_parse_landed_issues()` (around line 220):

```python
def _parse_covers_repos(slot_dir: str) -> set[str]:
    """Extract repo names from the Covers: line in .slot."""
    slot_file = Path(slot_dir) / ".slot"
    if not slot_file.exists():
        return set()
    for line in slot_file.read_text().splitlines():
        if line.startswith("Covers:"):
            parts = line.split(":", 1)[1].strip()
            return {
                entry.split(":")[0].strip()
                for entry in parts.split(",")
                if ":" in entry
            }
    return set()
```

Update `_resolve_original_repos()` signature (around line 381):

```python
def _resolve_original_repos(
    slot_dir: str,
    covers_repos: set[str] | None = None,
) -> dict[str, str]:
    result = {}
    slot_path = Path(slot_dir)
    for sub in sorted(slot_path.iterdir()):
        if not sub.is_dir() or not (sub / ".git").exists():
            continue
        if sub.name in (".m2", "attic"):
            continue
        if covers_repos and sub.name not in covers_repos:
            continue
        local_url = git(str(sub), "remote", "get-url", "local")
        if local_url.returncode == 0 and local_url.stdout.strip():
            orig_path = local_url.stdout.strip()
            if Path(orig_path).is_dir():
                result[sub.name] = orig_path
    return result
```

Update `check_landed_completeness()` signature (around line 286):

```python
def check_landed_completeness(
    slot_dir: str,
    covers_repos: set[str] | None = None,
) -> dict:
    expected_repos = covers_repos if covers_repos else _parse_slot_repos(slot_dir)
    if not expected_repos:
        return {"status": "pass", "detail": "no repos to check"}
    landed_repos = _parse_landed_repos(slot_dir)
    missing = expected_repos - landed_repos
    if missing:
        return {"status": "fail", "detail": f"repos not in .landed: {', '.join(sorted(missing))}"}
    extra = landed_repos - expected_repos
    if extra:
        return {"status": "warn", "detail": f"extra repos in .landed: {', '.join(sorted(extra))}"}
    return {"status": "pass", "detail": f"{len(landed_repos)}/{len(expected_repos)} repos landed"}
```

Update `verify()` signature and the `check_landed_completeness` call:

```python
def verify(
    project: str, branch: str, workspace: str,
    base: str = "main", covers: list[int] | None = None,
    issue_repo: str = "",
    slot_dir: str = "", original_repos: dict[str, str] | None = None,
    on_main: bool = False,
    covers_repos: set[str] | None = None,
) -> bool:
    # ... existing code ...
    if slot_dir:
        checks.append(("landed_marker", check_landed_marker(slot_dir)))
        checks.append(("landed_shas_populated", check_landed_shas_populated(slot_dir)))
        checks.append(("landed_completeness",
                        check_landed_completeness(slot_dir, covers_repos=covers_repos)))
    # ... rest unchanged ...
```

Update `main()` to parse covers_repos and pass through:

```python
    covers_repos = None
    if slot_dir:
        covers_repos = _parse_covers_repos(slot_dir)
        original_repos = _resolve_original_repos(slot_dir, covers_repos=covers_repos)

    verify(project, branch, workspace, base, covers,
           issue_repo=issue_repo,
           slot_dir=slot_dir, original_repos=original_repos,
           on_main=on_main, covers_repos=covers_repos)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestParseCoverRepos tests/test_verify_slot_close.py::TestResolveOriginalReposScoped tests/test_verify_slot_close.py::TestLandedCompletenessScoped -v`
Expected: PASS

- [ ] **Step 5: Run full verify test suite**

Run: `python3 -m pytest tests/test_verify_slot_close.py -v`
Expected: All pass

- [ ] **Step 6: Commit**

```bash
git add work-end/verify_slot_close.py tests/test_verify_slot_close.py
git commit -m "feat(#368): scope verify_slot_close to covered repos only  Refs #368"
```

---

## Batch 3: Integration verification

### Task 6: End-to-end orchestrator test — promote fails, landing proceeds

**Files:**
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes: All changes from Tasks 1-5
- Produces: Integration test proving the full fix works end-to-end

- [ ] **Step 1: Write the integration test**

```python
class TestPromoteFailureLandingProceeds:
    """End-to-end: promote fails → auto-skip → lifecycle advances → land runs."""

    def test_promote_auto_skip_unblocks_landing(self, tmp_path, monkeypatch):
        from close_progress import update_close_progress

        # Complete all review + sweep steps
        for step in ["report_init", "code_review", "branch_audit_conformance",
                     "branch_audit_coherence", "branch_audit_structure",
                     "branch_audit_robustness", "loose_ends", "forcing_function",
                     "sweep_config", "forage", "protocol", "update_claude_md",
                     "impl_doc_sync", "doc_freshness_gate", "adr", "write_content",
                     "review_pass"]:
            update_close_progress(tmp_path, step, "done")
        update_close_progress(tmp_path, "sweep_selected", "")

        # Pre-set promote at attempt 2 (one more failure triggers auto-skip)
        update_close_progress(tmp_path, "promote_mechanical_attempt", "2")

        landed = {"called": False}
        def selective_script(cmd, ws, **kw):
            cmd_str = str(cmd)
            if "promote" in cmd_str and "report" not in cmd_str:
                return {"ERROR": "worktree_failed"}
            if "land" in cmd_str and "report" not in cmd_str:
                landed["called"] = True
                return {"LANDED_SHA": "abc123", "LANDED": "yes"}
            return {}

        monkeypatch.setattr("work_end_orchestrator._run_script", selective_script)
        from work_end_orchestrator import run_orchestrator

        # Run orchestrator multiple times until land is called or we hit a dead end
        result = run_orchestrator({
            "workspace": str(tmp_path),
            "project": str(tmp_path / "project"),
            "branch": "issue-368-test",
            "base_branch": "main",
            "meta_state": "closing:verified",
            "on_main": "yes",
        })

        from close_progress import read_close_progress
        progress = read_close_progress(tmp_path)
        assert progress.get("promote") == "skipped_error"

        # The orchestrator should have continued past promote.
        # It may yield trajectory or continue to land depending on step sequence.
        # Re-invoke if needed to advance.
        max_iterations = 10
        for _ in range(max_iterations):
            if result.get("ACTION") == "complete":
                break
            step_done = result.get("ACTION", "")
            if step_done and step_done not in ("error", "user_input", "complete", "verify_recover"):
                result = run_orchestrator({
                    "workspace": str(tmp_path),
                    "project": str(tmp_path / "project"),
                    "branch": "issue-368-test",
                    "base_branch": "main",
                    "meta_state": "closing:verified",
                    "on_main": "yes",
                    "step_done": step_done,
                    "produced": "0",
                })
            else:
                break

        assert landed["called"], "Land step was never called — promote failure still blocks"
```

- [ ] **Step 2: Run test**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::TestPromoteFailureLandingProceeds -v`
Expected: PASS (all earlier tasks' changes are in place)

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py tests/test_verify_slot_close.py -v`
Expected: All pass

- [ ] **Step 4: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: No CRITICAL findings

- [ ] **Step 5: Commit**

```bash
git add tests/test_work_end_orchestrator.py
git commit -m "test(#368): integration test — promote failure unblocks landing  Refs #368"
```

---

## References

- [2026-09-14-work-end-slot-closure-design.md] — design spec this plan implements
- [2026-08-24-mechanise-work-end-close-design.md] — original orchestrator design (section D10)
- [2026-08-12-work-end-slot-landing-design.md] — slot landing design
- `work-end/orchestrator_engine.py:224-244` — current mechanical error handling
- `work-end/work_end_orchestrator.py:1245-1312` — per-repo mechanical fan-out
- `work-end/verify_slot_close.py:381-394` — `_resolve_original_repos` (unscoped)
- `work-end/shared_steps.py:50-58` — `done()`, `MAX_JUDGMENT_RETRIES`
- `work-end/progress_summary.py:255-284` — `_build_rows` status rendering
- `docs/protocols/evidence-before-claims.md`
- `docs/protocols/externalised-scripts-require-tests.md`
- GitHub #368

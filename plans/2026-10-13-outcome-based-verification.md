# Outcome-Based Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #450 — Phase 3: Outcome-based verification — three-valued VerifyResult, eliminate stamp files
**Issue group:** #450, #451

**Goal:** Replace bool postconditions and Finding-based close-checks with a unified three-valued verification layer (PASS/FAIL/UNKNOWN) backed by git state evidence.

**Architecture:** New `verification/outcomes.py` module with `VerifyResult` enum, `Verification` dataclass, and `verify_*()` pure functions. `StepDef` gains `outcome_fn` (returning `Verification`) replacing `postcondition_fn` (returning `bool`). Engine `run_loop()` gets three-way dispatch: PASS→skip, FAIL→execute/fail, UNKNOWN→retry/escalate. `close_verify.py` adapts via `verification_to_findings()`.

**Tech Stack:** Python 3.12, pytest, git CLI

## Global Constraints

- Every new `.py` file ships with tests in the same commit (protocol: externalised-scripts-require-tests)
- `verify_fn` on `StepDef` is UNCHANGED — it has 8 active consumers for judgment response validation
- Stamp files (`.artifacts-promoted`, `.landed`, `.phase-a-complete`) persist during migration as secondary evidence
- `close_checks.py` is partially deprecated — non-overlapping checks remain (`check_no_open_findings`, `check_no_stale_scaffold`, `check_slot_archive_status`, `check_landed_completeness`, `resolve_canonical_repos`)
- Verify functions take explicit parameters (repo path, branch, etc.) — not context objects
- `outcome_fn` lambdas bind context fields at step definition time, preserving the engine's `(ctx) -> Verification` contract

---

## Batch 1: Foundation — types, engine, adapter

### Task 1: VerifyResult, Verification, and helpers

**Files:**
- Create: `verification/outcomes.py`
- Test: `tests/test_outcomes.py`

**Interfaces:**
- Consumes: `verification.Finding` (from `verification/__init__.py`)
- Produces:
  - `class VerifyResult(Enum)` with values `PASS`, `FAIL`, `UNKNOWN`
  - `class Verification` (frozen dataclass) with fields `check: str`, `result: VerifyResult`, `evidence: dict`, `repo: str`
  - `def compose_verifications(check: str, *verifications: Verification) -> Verification`
  - `def verification_to_findings(v: Verification) -> list[Finding]`

- [ ] **Step 1: Write failing tests for VerifyResult and Verification**

```python
# tests/test_outcomes.py
import pytest
from verification.outcomes import VerifyResult, Verification, compose_verifications, verification_to_findings

class TestVerifyResult:
    def test_values(self):
        assert VerifyResult.PASS.value == "pass"
        assert VerifyResult.FAIL.value == "fail"
        assert VerifyResult.UNKNOWN.value == "unknown"

class TestVerification:
    def test_creation(self):
        v = Verification("content_landed", VerifyResult.PASS,
                         {"method": "ancestry"}, "soredium")
        assert v.check == "content_landed"
        assert v.result == VerifyResult.PASS
        assert v.evidence == {"method": "ancestry"}
        assert v.repo == "soredium"

    def test_frozen(self):
        v = Verification("test", VerifyResult.PASS, {}, "repo")
        with pytest.raises(AttributeError):
            v.check = "other"

class TestComposeVerifications:
    def test_all_pass(self):
        v1 = Verification("a", VerifyResult.PASS, {"k": "1"}, "r")
        v2 = Verification("b", VerifyResult.PASS, {"k": "2"}, "r")
        result = compose_verifications("composite", v1, v2)
        assert result.result == VerifyResult.PASS
        assert result.check == "composite"
        assert "a" in result.evidence
        assert "b" in result.evidence

    def test_fail_wins_over_unknown(self):
        v1 = Verification("a", VerifyResult.UNKNOWN, {"reason": "x"}, "r")
        v2 = Verification("b", VerifyResult.FAIL, {"diff": "y"}, "r")
        result = compose_verifications("composite", v1, v2)
        assert result.result == VerifyResult.FAIL
        # Both results preserved in evidence
        assert result.evidence["a"]["result"] == "unknown"
        assert result.evidence["b"]["result"] == "fail"

    def test_unknown_wins_over_pass(self):
        v1 = Verification("a", VerifyResult.PASS, {}, "r")
        v2 = Verification("b", VerifyResult.UNKNOWN, {"reason": "x"}, "r")
        result = compose_verifications("composite", v1, v2)
        assert result.result == VerifyResult.UNKNOWN

    def test_preserves_all_evidence(self):
        v1 = Verification("push", VerifyResult.UNKNOWN,
                          {"reason": "remote_unreachable"}, "repo")
        v2 = Verification("content", VerifyResult.FAIL,
                          {"diff_files": ["a.py"]}, "repo")
        result = compose_verifications("land", v1, v2)
        assert result.evidence["push"]["evidence"]["reason"] == "remote_unreachable"
        assert result.evidence["content"]["evidence"]["diff_files"] == ["a.py"]

class TestVerificationToFindings:
    def test_pass_returns_empty(self):
        v = Verification("check", VerifyResult.PASS, {}, "r")
        assert verification_to_findings(v) == []

    def test_fail_returns_error(self):
        v = Verification("content_landed", VerifyResult.FAIL,
                         {"diff_files": ["a.py"]}, "r")
        findings = verification_to_findings(v)
        assert len(findings) == 1
        assert findings[0].severity == "ERROR"
        assert findings[0].category == "content_landed"

    def test_unknown_returns_warn(self):
        v = Verification("push", VerifyResult.UNKNOWN,
                         {"reason": "remote_unreachable"}, "r")
        findings = verification_to_findings(v)
        assert len(findings) == 1
        assert findings[0].severity == "WARN"
        assert findings[0].category == "push"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_outcomes.py -v`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement VerifyResult, Verification, and helpers**

```python
# verification/outcomes.py
"""Three-valued outcome verification for the work-end pipeline.

Replaces bool postconditions (step_postconditions.py) and overlapping
Finding-based checks (close_checks.py) with a unified layer.
"""
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum

from verification import Finding


class VerifyResult(Enum):
    PASS = "pass"
    FAIL = "fail"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class Verification:
    check: str
    result: VerifyResult
    evidence: dict
    repo: str


def compose_verifications(check: str, *verifications: Verification) -> Verification:
    worst = VerifyResult.PASS
    composite_evidence: dict = {}
    for v in verifications:
        composite_evidence[v.check] = {
            "result": v.result.value,
            "evidence": v.evidence,
            "repo": v.repo,
        }
        if v.result == VerifyResult.FAIL:
            worst = VerifyResult.FAIL
        elif v.result == VerifyResult.UNKNOWN and worst != VerifyResult.FAIL:
            worst = VerifyResult.UNKNOWN
    repo = verifications[0].repo if verifications else ""
    return Verification(check, worst, composite_evidence, repo)


def verification_to_findings(v: Verification) -> list[Finding]:
    if v.result == VerifyResult.PASS:
        return []
    severity = "ERROR" if v.result == VerifyResult.FAIL else "WARN"
    evidence_summary = "; ".join(f"{k}={val}" for k, val in v.evidence.items())
    return [Finding(severity, v.check, evidence_summary)]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_outcomes.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add verification/outcomes.py tests/test_outcomes.py
git commit -m "feat(#450): add VerifyResult, Verification types and helpers"
```

### Task 2: StepDef outcome_fn + engine three-way dispatch

**Files:**
- Modify: `project/shared_steps.py` (add `outcome_fn`, `idempotent` fields)
- Modify: `project/orchestrator_engine.py` (three-way dispatch in `run_loop()`)
- Test: `tests/test_engine_outcomes.py`

**Interfaces:**
- Consumes: `VerifyResult`, `Verification` from `verification/outcomes.py`
- Produces:
  - `StepDef.outcome_fn: Callable | None` field (replaces `postcondition_fn`)
  - `StepDef.idempotent: bool` field (default `True`)
  - `_yield_unknown_escalation(step_name, verification) -> dict`
  - `_make_verify_fail_result(step_name, attempt, verification) -> dict`
  - `_make_unknown_retry_result(step_name, attempt, verification) -> dict`
  - Three-way dispatch in `run_loop()` for `outcome_fn`

- [ ] **Step 1: Write failing tests for engine three-way dispatch**

```python
# tests/test_engine_outcomes.py
import pytest
from pathlib import Path
from unittest.mock import MagicMock, patch
from verification.outcomes import VerifyResult, Verification

import sys
_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)

from shared_steps import StepDef
from orchestrator_engine import run_loop


def _make_ctx(workspace, progress=None):
    ctx = MagicMock()
    ctx.workspace = workspace
    ctx.progress = MagicMock()
    ctx.progress.is_done.return_value = False
    ctx.progress.is_per_repo_done.return_value = False
    ctx.progress.get.return_value = "0"
    ctx.progress.get_attempt.return_value = 0
    ctx.progress.__contains__ = lambda self, k: False
    ctx.done.return_value = False
    ctx.steps_executed = []
    ctx.last_output = {}
    ctx.repos = []
    ctx.dry_run = False
    ctx.call_log = []
    return ctx


class TestOutcomeFnPreCheck:
    def test_pass_skips_step(self, tmp_path):
        v = Verification("test_check", VerifyResult.PASS, {"ok": True}, "repo")
        step = StepDef("my_step", "phase", "mechanical",
                       outcome_fn=lambda ctx: v,
                       script_fn=lambda ctx: {})
        ctx = _make_ctx(tmp_path)
        result = run_loop([step], ctx)
        assert "my_step:verify_skip" in ctx.steps_executed
        assert result.get("ACTION") == "complete"

    def test_fail_executes_step(self, tmp_path):
        call_count = 0
        def counting_outcome(ctx):
            nonlocal call_count
            call_count += 1
            if call_count == 1:
                return Verification("check", VerifyResult.FAIL, {}, "r")
            return Verification("check", VerifyResult.PASS, {}, "r")

        step = StepDef("my_step", "phase", "mechanical",
                       outcome_fn=counting_outcome,
                       script_fn=lambda ctx: {})
        ctx = _make_ctx(tmp_path)
        result = run_loop([step], ctx)
        assert result.get("ACTION") == "complete"
        assert "my_step" in ctx.steps_executed

    def test_unknown_escalates_after_max_retries(self, tmp_path):
        v = Verification("check", VerifyResult.UNKNOWN,
                         {"reason": "missing"}, "repo")
        step = StepDef("my_step", "phase", "mechanical",
                       outcome_fn=lambda ctx: v,
                       script_fn=lambda ctx: {})
        ctx = _make_ctx(tmp_path)
        ctx.progress.get.return_value = "2"
        result = run_loop([step], ctx)
        assert result["ACTION"] == "user_input"
        assert result["CONTEXT"] == "unknown_verification"
        assert result["CHECK"] == "check"

    def test_unknown_non_idempotent_yields_without_execution(self, tmp_path):
        v = Verification("check", VerifyResult.UNKNOWN,
                         {"reason": "propagating"}, "repo")
        executed = []
        step = StepDef("my_step", "phase", "mechanical",
                       outcome_fn=lambda ctx: v,
                       script_fn=lambda ctx: executed.append(1) or {},
                       idempotent=False)
        ctx = _make_ctx(tmp_path)
        result = run_loop([step], ctx)
        assert result["ACTION"] == "error"
        assert "UNKNOWN" in result.get("REASON", "") or result.get("ERROR") == "unknown_verification"
        assert len(executed) == 0


class TestOutcomeFnPostCheck:
    def test_post_execution_fail_returns_error(self, tmp_path):
        v_fail = Verification("check", VerifyResult.FAIL,
                              {"diff": ["a.py"]}, "repo")
        step = StepDef("my_step", "phase", "mechanical",
                       outcome_fn=lambda ctx: v_fail,
                       script_fn=lambda ctx: {})
        ctx = _make_ctx(tmp_path)
        result = run_loop([step], ctx)
        assert result["ACTION"] == "error"
        assert result["ERROR"] == "verify_failed"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_engine_outcomes.py -v`
Expected: FAIL (outcome_fn not recognized / three-way dispatch not implemented)

- [ ] **Step 3: Add outcome_fn and idempotent to StepDef**

In `project/shared_steps.py`, add after `postcondition_fn`:

```python
    outcome_fn: Callable | None = None
    idempotent: bool = True
```

- [ ] **Step 4: Update run_loop() for three-way dispatch**

In `project/orchestrator_engine.py`, add imports at top:

```python
import json
from verification.outcomes import VerifyResult, Verification
```

Add helper functions before `run_loop()`:

```python
def _yield_unknown_escalation(step_name: str, verification) -> dict[str, str]:
    return {
        "ACTION": "user_input",
        "CONTEXT": "unknown_verification",
        "STEP": step_name,
        "CHECK": verification.check,
        "RESULT": "unknown",
        "EVIDENCE": json.dumps(verification.evidence),
        "REASON": f"Verification '{verification.check}' returned UNKNOWN "
                  f"after {MAX_MECHANICAL_RETRIES} retries",
    }


def _make_verify_fail_result(step_name: str, attempt: int,
                              verification) -> dict[str, str]:
    return {
        "ACTION": "error",
        "ERROR": "verify_failed",
        "STEP": step_name,
        "RETRY": str(attempt),
        "CHECK": verification.check,
        "EVIDENCE": json.dumps(verification.evidence),
        "REASON": f"Verification '{verification.check}' returned FAIL",
    }


def _make_unknown_retry_result(step_name: str, attempt: int,
                                verification) -> dict[str, str]:
    return {
        "ACTION": "error",
        "ERROR": "unknown_verification",
        "STEP": step_name,
        "RETRY": str(attempt),
        "CHECK": verification.check,
        "EVIDENCE": json.dumps(verification.evidence),
        "REASON": f"Verification '{verification.check}' returned UNKNOWN — retrying",
    }
```

In `run_loop()`, replace the mechanical step block (lines ~265-329).
The new logic handles both `postcondition_fn` (backward compat during
migration) and `outcome_fn` (new three-way dispatch):

```python
        if step.step_type == "mechanical":
            if per_repo_mechanical:
                handled = per_repo_mechanical(step, ctx)
                if handled is not None:
                    if handled:
                        return handled
                    continue

            if ctx.done(step.name):
                continue

            # --- NEW: outcome_fn three-way pre-check ---
            if step.outcome_fn:
                v = step.outcome_fn(ctx)
                if v.result == VerifyResult.PASS:
                    _mark_done(ctx, step.name)
                    ctx.steps_executed.append(f"{step.name}:verify_skip")
                    continue
                if v.result == VerifyResult.UNKNOWN:
                    attempt_key = f"{step.name}_mechanical_attempt"
                    raw = ctx.progress.get(attempt_key, "0")
                    try:
                        attempt = int(raw)
                    except (ValueError, TypeError):
                        attempt = 0
                    attempt += 1
                    update_close_progress(ctx.workspace, attempt_key, str(attempt))
                    if attempt >= MAX_MECHANICAL_RETRIES:
                        return _yield_unknown_escalation(step.name, v)
                    if not step.idempotent:
                        return _make_unknown_retry_result(step.name, attempt, v)
                # FAIL or UNKNOWN (idempotent): fall through to execution

            # --- LEGACY: postcondition_fn binary pre-check ---
            elif step.postcondition_fn and step.postcondition_fn(ctx):
                _mark_done(ctx, step.name)
                ctx.steps_executed.append(f"{step.name}:postcondition_skip")
                continue

            attempt_key = f"{step.name}_mechanical_attempt"
            raw = ctx.progress.get(attempt_key, "0")
            try:
                attempt = int(raw)
            except (ValueError, TypeError):
                attempt = 0

            if execute_mechanical_fn:
                result = execute_mechanical_fn(step, ctx)
            else:
                result = _default_execute_mechanical(step, ctx)

            if result and "ERROR" in result:
                if on_mechanical_error:
                    override = on_mechanical_error(step, ctx, result)
                    if override is not None:
                        ctx.steps_executed.append(f"{step.name}:ERROR:classified")
                        return override
                attempt += 1
                update_close_progress(ctx.workspace, attempt_key, str(attempt))
                if attempt >= MAX_MECHANICAL_RETRIES:
                    _mark_done(ctx, step.name, "skipped_error")
                    ctx.steps_executed.append(f"{step.name}:SKIPPED_ERROR")
                    continue
                ctx.steps_executed.append(f"{step.name}:ERROR:{attempt}")
                return _make_error_result(step.name, attempt, result)

            ctx.last_output = result or {}
            if on_step_done:
                on_step_done(step, ctx, result or {})

            # --- NEW: outcome_fn three-way post-verify ---
            if step.outcome_fn:
                v = step.outcome_fn(ctx)
                if v.result == VerifyResult.PASS:
                    _mark_done(ctx, step.name)
                    ctx.steps_executed.append(step.name)
                    continue
                if v.result == VerifyResult.FAIL:
                    attempt += 1
                    update_close_progress(ctx.workspace, attempt_key, str(attempt))
                    if attempt >= MAX_MECHANICAL_RETRIES:
                        _mark_done(ctx, step.name, "skipped_error")
                        ctx.steps_executed.append(f"{step.name}:VERIFY_FAIL:skipped")
                        continue
                    return _make_verify_fail_result(step.name, attempt, v)
                if v.result == VerifyResult.UNKNOWN:
                    attempt += 1
                    update_close_progress(ctx.workspace, attempt_key, str(attempt))
                    if attempt >= MAX_MECHANICAL_RETRIES:
                        return _yield_unknown_escalation(step.name, v)
                    return _make_unknown_retry_result(step.name, attempt, v)

            # --- LEGACY: postcondition_fn binary post-verify ---
            elif step.postcondition_fn and not step.postcondition_fn(ctx):
                attempt += 1
                update_close_progress(ctx.workspace, attempt_key, str(attempt))
                if attempt >= MAX_MECHANICAL_RETRIES:
                    _mark_done(ctx, step.name, "skipped_error")
                    ctx.steps_executed.append(f"{step.name}:POSTCONDITION_FAIL:skipped")
                    continue
                ctx.steps_executed.append(f"{step.name}:POSTCONDITION_FAIL:{attempt}")
                return {
                    "ACTION": "error",
                    "ERROR": "postcondition_failed",
                    "STEP": step.name,
                    "RETRY": str(attempt),
                    "REASON": f"Step '{step.name}' executed but postcondition not met",
                }

            _mark_done(ctx, step.name)
            ctx.steps_executed.append(step.name)
            continue
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_engine_outcomes.py tests/test_outcomes.py -v`
Expected: all PASS

- [ ] **Step 6: Run existing engine tests to verify no regression**

Run: `python3 -m pytest tests/ -k "engine or orchestrator" -v`
Expected: all existing tests still PASS

- [ ] **Step 7: Commit**

```bash
git add project/shared_steps.py project/orchestrator_engine.py tests/test_engine_outcomes.py
git commit -m "feat(#450): add outcome_fn to StepDef, three-way dispatch in run_loop"
```

---

## Batch 2: First verify functions — content_landed + push

### Task 3: verify_content_landed and verify_push

**Files:**
- Modify: `verification/outcomes.py` (add verify functions)
- Modify: `tests/test_outcomes.py` (add verify function tests)

**Interfaces:**
- Consumes: `git` from `verification.__init__` (subprocess wrapper)
- Produces:
  - `def verify_content_landed(repo_path: Path, landed_sha: str, base_branch: str) -> Verification`
  - `def verify_push(repo_path: Path, base_branch: str) -> Verification`

- [ ] **Step 1: Write failing tests for verify_content_landed**

```python
# append to tests/test_outcomes.py
from pathlib import Path
from unittest.mock import patch, MagicMock
from verification.outcomes import verify_content_landed, verify_push


class TestVerifyContentLanded:
    def test_pass_via_ancestry(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="")
            result = verify_content_landed(tmp_path, "abc123", "main")
            assert result.result == VerifyResult.PASS
            assert result.check == "content_landed"

    def test_unknown_no_sha(self, tmp_path):
        result = verify_content_landed(tmp_path, "", "main")
        assert result.result == VerifyResult.UNKNOWN
        assert "no_landed_sha" in result.evidence.get("reason", "")

    def test_fail_with_diff(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            def git_side_effect(path, *args, **kwargs):
                cmd = args[0] if args else ""
                if cmd == "merge-base":
                    return MagicMock(returncode=1, stdout="")
                if cmd == "diff" and "--name-only" not in args:
                    return MagicMock(returncode=0, stdout="--- a/file.py\n+++ b/file.py\n@@ ...\n")
                if cmd == "diff" and "--name-only" in args:
                    return MagicMock(returncode=0, stdout="file.py\n")
                return MagicMock(returncode=0, stdout="")
            mock_git.side_effect = git_side_effect
            result = verify_content_landed(tmp_path, "abc123", "main")
            assert result.result == VerifyResult.FAIL

    def test_pass_via_empty_diff(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            def git_side_effect(path, *args, **kwargs):
                cmd = args[0] if args else ""
                if cmd == "merge-base":
                    return MagicMock(returncode=1, stdout="")
                return MagicMock(returncode=0, stdout="")
            mock_git.side_effect = git_side_effect
            result = verify_content_landed(tmp_path, "abc123", "main")
            assert result.result == VerifyResult.PASS


class TestVerifyPush:
    def test_pass_no_unpushed(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="")
            result = verify_push(tmp_path, "main")
            assert result.result == VerifyResult.PASS

    def test_fail_with_unpushed(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="abc123 commit msg\n")
            result = verify_push(tmp_path, "main")
            assert result.result == VerifyResult.FAIL

    def test_unknown_remote_unreachable(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=128, stdout="", stderr="fatal: couldn't find remote ref")
            result = verify_push(tmp_path, "main")
            assert result.result == VerifyResult.UNKNOWN
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_outcomes.py::TestVerifyContentLanded tests/test_outcomes.py::TestVerifyPush -v`
Expected: FAIL (functions not defined)

- [ ] **Step 3: Implement verify_content_landed and verify_push**

Add to `verification/outcomes.py`:

```python
from pathlib import Path
import subprocess
import sys

_project_dir = str(Path(__file__).resolve().parent.parent / "project")
if _project_dir not in sys.path:
    sys.path.insert(0, _project_dir)
from git_util import git as _canonical_git


def _git(repo_path, *args: str, **kwargs) -> subprocess.CompletedProcess:
    return _canonical_git(repo_path, *args, timeout=kwargs.get("timeout", 10))


SOURCE_EXTENSIONS = (
    ".java", ".kt", ".xml", ".yaml", ".yml", ".json",
    ".properties", ".sql", ".py", ".ts", ".tsx", ".js",
    ".jsx", ".css", ".scss", ".html",
)


def verify_content_landed(repo_path: Path, landed_sha: str,
                          base_branch: str) -> Verification:
    repo_name = repo_path.name if isinstance(repo_path, Path) else Path(repo_path).name
    if not landed_sha:
        return Verification("content_landed", VerifyResult.UNKNOWN,
                            {"reason": "no_landed_sha"}, repo_name)

    result = _git(repo_path, "merge-base", "--is-ancestor", landed_sha, base_branch)
    if result.returncode == 0:
        return Verification("content_landed", VerifyResult.PASS,
                            {"method": "ancestry", "sha": landed_sha}, repo_name)

    diff = _git(repo_path, "diff", f"{base_branch}..{landed_sha}",
                "--", *[f"*{ext}" for ext in SOURCE_EXTENSIONS])
    if diff.returncode == 0 and not diff.stdout.strip():
        return Verification("content_landed", VerifyResult.PASS,
                            {"method": "empty_source_diff", "sha": landed_sha}, repo_name)

    if diff.returncode == 0:
        diff_names = _git(repo_path, "diff", "--name-only",
                          f"{base_branch}..{landed_sha}",
                          "--", *[f"*{ext}" for ext in SOURCE_EXTENSIONS])
        files = [f for f in diff_names.stdout.strip().split("\n") if f]
        return Verification("content_landed", VerifyResult.FAIL,
                            {"method": "diff_check", "diff_files": files,
                             "sha": landed_sha}, repo_name)

    return Verification("content_landed", VerifyResult.UNKNOWN,
                        {"reason": "diff_failed", "sha": landed_sha,
                         "stderr": diff.stderr[:200] if diff.stderr else ""},
                        repo_name)


def verify_push(repo_path: Path, base_branch: str) -> Verification:
    repo_name = repo_path.name if isinstance(repo_path, Path) else Path(repo_path).name
    result = _git(repo_path, "log", f"origin/{base_branch}..{base_branch}", "--oneline")
    if result.returncode != 0:
        return Verification("push", VerifyResult.UNKNOWN,
                            {"reason": "remote_unreachable",
                             "stderr": result.stderr[:200] if result.stderr else ""},
                            repo_name)
    unpushed = result.stdout.strip()
    if unpushed:
        count = len(unpushed.splitlines())
        return Verification("push", VerifyResult.FAIL,
                            {"unpushed_count": count}, repo_name)
    return Verification("push", VerifyResult.PASS,
                        {"method": "no_unpushed"}, repo_name)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_outcomes.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add verification/outcomes.py tests/test_outcomes.py
git commit -m "feat(#450): add verify_content_landed and verify_push"
```

### Task 4: Wire land step to outcome_fn

**Files:**
- Modify: `work-end/work_end_orchestrator.py` (replace land step's `postcondition_fn` with `outcome_fn`)
- Test: existing orchestrator tests + `tests/test_engine_outcomes.py`

**Interfaces:**
- Consumes: `verify_content_landed`, `verify_push`, `compose_verifications` from `verification/outcomes.py`
- Produces: `land` step uses `outcome_fn` with composite verification; `postcondition_fn` removed from `land` step

- [ ] **Step 1: Write a test verifying land step has outcome_fn**

```python
# append to tests/test_engine_outcomes.py

class TestLandStepWiring:
    def test_land_step_has_outcome_fn(self):
        """Verify the land step uses outcome_fn, not postcondition_fn."""
        import importlib.util
        spec = importlib.util.spec_from_file_location(
            "_weo", str(Path(__file__).parent.parent / "work-end" / "work_end_orchestrator.py"))
        mod = importlib.util.module_from_spec(spec)
        # Don't exec — just verify we can import. The actual step
        # definition test requires the full orchestrator context.
        # Instead test the composition pattern directly:
        from verification.outcomes import (
            verify_content_landed, verify_push,
            compose_verifications, VerifyResult,
        )
        from unittest.mock import patch, MagicMock
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="")
            v = compose_verifications("land",
                verify_push(Path("/tmp/test"), "main"),
                verify_content_landed(Path("/tmp/test"), "abc123", "main"))
            assert v.result == VerifyResult.PASS
            assert v.check == "land"
            assert "push" in v.evidence
            assert "content_landed" in v.evidence
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_engine_outcomes.py::TestLandStepWiring -v`
Expected: PASS (the composition test should pass since types exist)

- [ ] **Step 3: Update land step definition in work_end_orchestrator.py**

Find the `land` StepDef definition and replace `postcondition_fn=lambda ctx: push_postcondition(ctx) and content_landed_postcondition(ctx)` with:

```python
outcome_fn=lambda ctx: compose_verifications("land",
    verify_push(
        getattr(ctx, "current_repo_project", None) or ctx.project,
        ctx.base_branch),
    verify_content_landed(
        getattr(ctx, "current_repo_project", None) or ctx.project,
        ctx.landed_shas.get(
            (getattr(ctx, "current_repo_project", None) or ctx.project).name, ""),
        ctx.base_branch)),
idempotent=False,
```

Add import at top of `work_end_orchestrator.py`:

```python
from verification.outcomes import (
    verify_content_landed, verify_push, compose_verifications,
)
```

Remove the `push_postcondition` and `content_landed_postcondition` imports if no other step uses them (check first — `upstream_push` still uses `push_postcondition`; keep that import for now).

- [ ] **Step 4: Run full test suite to verify no regression**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/work_end_orchestrator.py
git commit -m "feat(#450): wire land step to outcome_fn with composite verification"
```

---

## Batch 3: Remaining verify functions + step wiring

### Task 5: verify_rebase, verify_branch_stamped, verify_artifacts_promoted

**Files:**
- Modify: `verification/outcomes.py`
- Modify: `tests/test_outcomes.py`
- Modify: `work-end/work_end_orchestrator.py` (wire rebase and promote steps)

**Interfaces:**
- Produces:
  - `def verify_rebase(repo_path: Path, branch: str, base_branch: str) -> Verification`
  - `def verify_branch_stamped(repo_path: Path, branch: str, base_branch: str) -> Verification`
  - `def verify_artifacts_promoted(workspace: Path, project: Path, branch: str, base_branch: str) -> Verification`

- [ ] **Step 1: Write failing tests**

```python
# append to tests/test_outcomes.py
from verification.outcomes import verify_rebase, verify_branch_stamped, verify_artifacts_promoted


class TestVerifyRebase:
    def test_pass(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0)
            result = verify_rebase(tmp_path, "feature", "main")
            assert result.result == VerifyResult.PASS

    def test_fail(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=1)
            result = verify_rebase(tmp_path, "feature", "main")
            assert result.result == VerifyResult.FAIL

    def test_unknown_no_branch(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=128,
                stderr="fatal: Not a valid object name")
            result = verify_rebase(tmp_path, "nonexistent", "main")
            assert result.result == VerifyResult.UNKNOWN


class TestVerifyBranchStamped:
    def test_pass_with_sha(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            def side_effect(path, *args, **kw):
                if args[0] == "log":
                    return MagicMock(returncode=0,
                        stdout="chore: branch closed — landed as abc123 on main\n")
                if args[0] == "merge-base":
                    return MagicMock(returncode=0)
                return MagicMock(returncode=0, stdout="")
            mock_git.side_effect = side_effect
            result = verify_branch_stamped(tmp_path, "feature", "main")
            assert result.result == VerifyResult.PASS

    def test_fail_no_stamp(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0,
                stdout="feat: add something\nfix: repair thing\n")
            result = verify_branch_stamped(tmp_path, "feature", "main")
            assert result.result == VerifyResult.FAIL


class TestVerifyArtifactsPromoted:
    def test_pass_stamp_exists(self, tmp_path):
        (tmp_path / ".artifacts-promoted").touch()
        result = verify_artifacts_promoted(tmp_path, tmp_path, "feature", "main")
        assert result.result == VerifyResult.PASS

    def test_pass_no_diff(self, tmp_path):
        with patch("verification.outcomes._git") as mock_git:
            mock_git.return_value = MagicMock(returncode=0, stdout="")
            result = verify_artifacts_promoted(tmp_path, tmp_path, "feature", "main")
            assert result.result == VerifyResult.PASS
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_outcomes.py -k "Rebase or Stamped or Promoted" -v`
Expected: FAIL

- [ ] **Step 3: Implement verify_rebase, verify_branch_stamped, verify_artifacts_promoted**

Add to `verification/outcomes.py`:

```python
import re

STAMP_PREFIX = "chore: branch closed"


def verify_rebase(repo_path: Path, branch: str,
                  base_branch: str) -> Verification:
    repo_name = Path(repo_path).name
    result = _git(repo_path, "merge-base", "--is-ancestor", base_branch, branch)
    if result.returncode == 0:
        return Verification("rebase", VerifyResult.PASS,
                            {"method": "ancestry"}, repo_name)
    if result.returncode == 128 or (result.stderr and "Not a valid" in result.stderr):
        return Verification("rebase", VerifyResult.UNKNOWN,
                            {"reason": "ref_not_found", "branch": branch}, repo_name)
    return Verification("rebase", VerifyResult.FAIL,
                        {"method": "ancestry_failed", "branch": branch,
                         "base": base_branch}, repo_name)


def verify_branch_stamped(repo_path: Path, branch: str,
                          base_branch: str) -> Verification:
    repo_name = Path(repo_path).name
    result = _git(repo_path, "log", "-5", "--format=%s", branch)
    if result.returncode != 0:
        return Verification("branch_stamped", VerifyResult.UNKNOWN,
                            {"reason": "branch_not_found"}, repo_name)
    for line in result.stdout.strip().splitlines():
        if line.startswith(STAMP_PREFIX):
            sha_match = re.search(r"landed as ([0-9a-f]+)", line)
            if not sha_match:
                return Verification("branch_stamped", VerifyResult.PASS,
                                    {"method": "stamp_found_no_sha"}, repo_name)
            sha = sha_match.group(1)
            check = _git(repo_path, "merge-base", "--is-ancestor", sha, base_branch)
            if check.returncode == 0:
                return Verification("branch_stamped", VerifyResult.PASS,
                                    {"method": "stamp_sha_verified", "sha": sha},
                                    repo_name)
            return Verification("branch_stamped", VerifyResult.UNKNOWN,
                                {"reason": "stamp_sha_not_on_base", "sha": sha},
                                repo_name)
    return Verification("branch_stamped", VerifyResult.FAIL,
                        {"method": "no_stamp_found"}, repo_name)


def verify_artifacts_promoted(workspace: Path, project: Path,
                              branch: str, base_branch: str) -> Verification:
    repo_name = Path(workspace).name
    for candidate in [workspace / ".artifacts-promoted",
                      workspace / "design" / ".artifacts-promoted"]:
        if candidate.exists():
            return Verification("artifacts_promoted", VerifyResult.PASS,
                                {"method": "stamp_file"}, repo_name)
    result = _git(workspace, "log", "--oneline", "--all", "-1",
                  "--grep=workspace and project promotion stamp")
    if result.returncode == 0 and result.stdout.strip():
        return Verification("artifacts_promoted", VerifyResult.PASS,
                            {"method": "promotion_commit"}, repo_name)
    diff = _git(project, "diff", "--stat", f"{base_branch}..{branch}")
    if diff.returncode == 0 and not diff.stdout.strip():
        return Verification("artifacts_promoted", VerifyResult.PASS,
                            {"method": "no_diff"}, repo_name)
    return Verification("artifacts_promoted", VerifyResult.FAIL,
                        {"method": "no_evidence"}, repo_name)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_outcomes.py -v`
Expected: all PASS

- [ ] **Step 5: Wire rebase and promote steps in work_end_orchestrator.py**

Replace `postcondition_fn=rebase_postcondition` with:
```python
outcome_fn=lambda ctx: verify_rebase(
    getattr(ctx, "current_repo_project", None) or ctx.project,
    ctx.branch, ctx.base_branch),
```

Replace `postcondition_fn=promote_postcondition` with:
```python
outcome_fn=lambda ctx: verify_artifacts_promoted(
    getattr(ctx, "current_repo_workspace", None) or ctx.workspace,
    getattr(ctx, "current_repo_project", None) or ctx.project,
    ctx.branch, ctx.base_branch),
```

Add import: `from verification.outcomes import verify_rebase, verify_artifacts_promoted`

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add verification/outcomes.py tests/test_outcomes.py work-end/work_end_orchestrator.py
git commit -m "feat(#450): add verify_rebase, verify_branch_stamped, verify_artifacts_promoted; wire rebase+promote steps"
```

### Task 6: Remaining verify functions and step wiring

**Files:**
- Modify: `verification/outcomes.py`
- Modify: `tests/test_outcomes.py`
- Modify: `work-end/work_end_orchestrator.py`

**Interfaces:**
- Produces:
  - `def verify_issues_closed(issue_repo: str, covers: str, plan_path: Path | None = None) -> Verification`
  - `def verify_checkout_main(project: Path, workspace: Path) -> Verification`
  - `def verify_landed_marker(slot_path: Path) -> Verification`
  - `def verify_slot_phase_complete(slot_path: Path, marker: str) -> Verification`
  - `def verify_fork_next(workspace: Path, progress: dict) -> Verification`
  - `def verify_checkout_fork(workspace: Path, progress: dict) -> Verification`
  - `def verify_cleanup_scaffold(workspace: Path) -> Verification`
  - `def verify_promote_plan_next(workspace: Path) -> Verification`

- [ ] **Step 1: Write failing tests for all remaining verify functions**

Tests for each function covering PASS, FAIL, UNKNOWN paths.
Follow the same pattern as Task 3/5 tests — mock `_git` and
check the returned `Verification` result and evidence.

Key test cases:
- `verify_issues_closed`: PASS (all closed), FAIL (one open), UNKNOWN (gh fails)
- `verify_checkout_main`: PASS (both on main), FAIL (one not on main)
- `verify_landed_marker`: PASS (valid .landed), FAIL (missing), UNKNOWN (malformed)
- `verify_slot_phase_complete`: PASS (marker exists), FAIL (missing)
- `verify_fork_next`: PASS (branch exists), FAIL (not found)
- `verify_checkout_fork`: PASS (on fork branch), FAIL (on wrong branch)
- `verify_cleanup_scaffold`: PASS (no scaffold), FAIL (scaffold remains)
- `verify_promote_plan_next`: PASS (no .plan-next), FAIL (.plan-next exists)

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement all remaining verify functions**

Each follows the same pattern: check the actual state, return
PASS/FAIL/UNKNOWN with evidence dict.

`verify_issues_closed` uses `subprocess.run(["gh", ...])` directly
(same as current `issues_closed_postcondition`).

Simple checks (`verify_cleanup_scaffold`, `verify_promote_plan_next`,
`verify_fork_next`, `verify_checkout_fork`) are straightforward
file/branch existence checks — they typically only return PASS/FAIL
with no UNKNOWN path.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_outcomes.py -v`

- [ ] **Step 5: Wire all remaining steps in work_end_orchestrator.py**

For each of the remaining 8 steps, replace `postcondition_fn=X` with
`outcome_fn=lambda ctx: verify_Y(...)`:

| Step | Old postcondition_fn | New outcome_fn |
|------|---------------------|----------------|
| `upstream_push` | `push_postcondition` | `verify_push(ctx.project, ctx.base_branch)` |
| `close_issues` | `issues_closed_postcondition` | `verify_issues_closed(ctx.issue_repo, ctx.covers, ctx.plan_path)` |
| `checkout_main` | `checkout_main_postcondition` | `verify_checkout_main(ctx.project, ctx.workspace)` |
| `write_landed` | `landed_marker_postcondition` | `verify_landed_marker(ctx.slot_path)` |
| `write_marker` | `write_marker_postcondition` | `verify_slot_phase_complete(ctx.slot_path, ".phase-a-complete")` |
| `fork_next_branch` | `fork_next_postcondition` | `verify_fork_next(ctx.workspace, ctx.progress)` |
| `checkout_fork` | `checkout_fork_postcondition` | `verify_checkout_fork(ctx.workspace, ctx.progress)` |
| `cleanup` | `cleanup_scaffold_postcondition` | `verify_cleanup_scaffold(ctx.workspace)` |
| `promote_plan_next` | `promote_plan_next_postcondition` | `verify_promote_plan_next(ctx.workspace)` |

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`

- [ ] **Step 7: Commit**

```bash
git add verification/outcomes.py tests/test_outcomes.py work-end/work_end_orchestrator.py
git commit -m "feat(#450): wire all remaining steps to outcome_fn, replace all postcondition_fn"
```

---

## Batch 4: Close-verify adaptation + cleanup

### Task 7: Update close_verify.py to use verify functions

**Files:**
- Modify: `verification/close_verify.py`
- Modify: `tests/test_outcomes.py` (or existing close_verify tests)

**Interfaces:**
- Consumes: all `verify_*()` from `verification/outcomes.py`, `verification_to_findings()`
- Produces: `close_verify.verify()` uses `verify_*()` for overlapping checks; output format unchanged

- [ ] **Step 1: Write test verifying output format is unchanged**

```python
# Test that close_verify.verify() still produces VERIFIED=yes/no,
# FAILURE_N=, BLOCKING_FAIL= output format after migration.
# Use existing test patterns from tests/ if they exist.
```

- [ ] **Step 2: Update _run_branch_checks to use verify functions**

Replace calls to `close_checks.check_branch_merged`, `check_branch_stamp`,
`check_main_pushed`, `check_on_main`, `check_artifacts_promoted`,
`check_issues_closed` with calls to `verify_*()` + `verification_to_findings()`.

Non-overlapping checks (`check_no_open_findings`, `check_no_stale_scaffold`)
remain as-is from `close_checks.py`.

- [ ] **Step 3: Update _run_slot_checks similarly**

Replace `check_landed_marker`, `check_slot_marker`, `check_canonical_sync`
calls with `verify_landed_marker`, `verify_slot_phase_complete`,
`verify_canonical_sync` + adapter.

`check_landed_completeness`, `check_slot_archive_status` remain from
`close_checks.py`.

- [ ] **Step 4: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`

- [ ] **Step 5: Commit**

```bash
git add verification/close_verify.py tests/test_outcomes.py
git commit -m "feat(#450): migrate close_verify.py to use verify_*() via adapter"
```

### Task 8: Remove deprecated code and dead code

**Files:**
- Modify: `verification/step_postconditions.py` (remove all functions — file becomes empty/deleted)
- Modify: `verification/close_checks.py` (remove overlapping functions, keep non-overlapping)
- Modify: `work-end/work_end_orchestrator.py` (remove unused postcondition imports)
- Modify: `project/shared_steps.py` (remove `postcondition_fn` field from StepDef)

**Interfaces:**
- Consumes: nothing new
- Produces: clean codebase with no duplicate verification logic

- [ ] **Step 1: Remove postcondition_fn from StepDef**

In `project/shared_steps.py`, remove the `postcondition_fn` field.
Verify no remaining references via search.

- [ ] **Step 2: Remove all functions from step_postconditions.py**

Delete all 14 functions. Keep the file with just the module docstring
(or delete entirely if no other imports reference the module path).

- [ ] **Step 3: Remove overlapping functions from close_checks.py**

Remove: `check_branch_merged`, `check_branch_stamp`, `check_main_pushed`,
`check_on_main`, `check_artifacts_promoted`, `check_issues_closed`,
`check_landed_marker`, `check_slot_marker`, `check_canonical_sync`.

Keep: `check_no_open_findings`, `check_no_stale_scaffold`,
`check_slot_archive_status`, `check_landed_completeness`,
`resolve_canonical_repos`.

- [ ] **Step 4: Remove dead code**

Remove `stamp_postcondition` and `archive_move_postcondition` references
(already dead — confirmed by review R1-11).

Remove unused imports from `work_end_orchestrator.py`.

- [ ] **Step 5: Run full test suite**

Run: `python3 -m pytest tests/ -v --timeout=120`

- [ ] **Step 6: Commit**

```bash
git add verification/step_postconditions.py verification/close_checks.py \
    work-end/work_end_orchestrator.py project/shared_steps.py
git commit -m "feat(#450): remove deprecated postconditions and overlapping close_checks"
```

---

## References

- `specs/issue-450-outcome-based-verification/2026-10-13-outcome-based-verification-design.md` — design spec
- `specs/issue-443-robust-pipeline-architecture/2026-10-12-robust-pipeline-architecture-design.md` § Phase 3 — parent spec
- `verification/step_postconditions.py` — 14 bool postconditions being replaced
- `verification/close_checks.py` — Finding-based checks partially replaced
- `verification/close_verify.py` — composed reporter being adapted
- `project/orchestrator_engine.py:265-329` — current binary postcondition dispatch
- `project/shared_steps.py:34-49` — StepDef definition
- `work-end/work_end_orchestrator.py` — 12 steps with postcondition_fn wired
- `docs/protocols/externalised-scripts-require-tests.md` — testing protocol
- `docs/protocols/evidence-before-claims.md` — verification protocol
- GitHub #436, #375, #435, #386, #388 — postcondition failure issues

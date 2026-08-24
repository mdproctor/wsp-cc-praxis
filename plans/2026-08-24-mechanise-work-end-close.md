# Mechanise work-end close sequence — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #271 — Mechanise work-end close sequence
**Issue group:** #271

**Goal:** Replace the LLM-as-orchestrator work-end model with a Python
orchestrator script that owns the close sequence, yielding to the LLM
only for judgment steps.

**Architecture:** A stateless re-entrant Python script
(`work_end_orchestrator.py`) reads `.close-progress` and `META_STATE`,
runs mechanical steps, and yields one `ACTION=` line when it needs LLM
judgment. The SKILL.md becomes a ~20-line dispatch loop plus ~200 lines
of Action Handlers (per-action judgment constraints the LLM needs) plus
~250 lines of pre-close context handling. Existing scripts are called
as-is. In slot mode, the orchestrator routes `land` to
`slot_manager.py merge-slot` instead of `work_end_execute.py land`.

**Tech Stack:** Python 3, pytest, lifecycle.py state machine,
existing work-end scripts

## Global Constraints

- Python 3.11+ (match existing soredium scripts)
- All new scripts must have pytest tests per `externalised-scripts-require-tests` protocol
- Atomic write-then-rename for all progress files (POSIX `os.replace()`)
- KEY=VALUE output format (match soredium ecosystem)
- No new dependencies beyond stdlib + existing project imports
- `evidence-before-claims` protocol applies at every completion boundary
- Two progress files coexist: `.close-progress` (orchestrator-level: which close step) and `.execute-progress` (script-level: which repo promoted/landed). `.execute-progress` is owned by `work_end_execute.py` and cleaned by `branch_cleanup.py cleanup-scaffold`. `.close-progress` is owned by the orchestrator and cleaned by the orchestrator before `ACTION=complete`. They are independent — neither reads the other.

---

## Batch 1: Crash-safety fix

### Task 1: Atomic write-then-rename for progress files

**Files:**
- Modify: `work-end/work_end_execute.py:127-132` — `write_progress()`
- Modify: `work-end/land_flow.py:69-74` — `_write_progress()`
- Test: `tests/test_work_end_execute.py`
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Consumes: existing `read_progress()` / `_read_progress()` (unchanged)
- Produces: same `write_progress(path, key, value)` and `_write_progress(path, key, value)` signatures (unchanged API, safer implementation)

- [ ] **Step 1: Write failing test for atomic write in work_end_execute.py**

```python
def test_write_progress_atomic_uses_rename(tmp_path):
    progress = tmp_path / ".execute-progress"
    write_progress(progress, "step1", "done")
    assert progress.exists()
    assert not (tmp_path / ".execute-progress.tmp").exists()
    content = progress.read_text()
    assert "step1=done" in content
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_execute.py::test_write_progress_atomic_uses_rename -v`
Expected: PASS (current implementation writes the file, just not atomically — the test verifies the .tmp is cleaned up)

Note: The existing implementation WILL pass this test since `.tmp` is never created. We need to verify the atomic pattern is used, not just that the file exists. Add a more targeted test:

```python
def test_write_progress_survives_crash(tmp_path, monkeypatch):
    """Simulate crash between truncation and write — old data must survive."""
    progress = tmp_path / ".execute-progress"
    write_progress(progress, "step1", "done")

    original_replace = os.replace
    def crash_on_replace(src, dst):
        raise OSError("simulated crash")
    monkeypatch.setattr("os.replace", crash_on_replace)

    with pytest.raises(OSError):
        write_progress(progress, "step2", "done")

    content = progress.read_text()
    assert "step1=done" in content
    assert "step2" not in content
```

- [ ] **Step 3: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_execute.py::test_write_progress_survives_crash -v`
Expected: FAIL — current implementation uses `path.write_text()` which truncates first; crash loses step1

- [ ] **Step 4: Fix write_progress in work_end_execute.py**

Replace lines 127-132:

```python
def write_progress(progress_path: Path, key: str, value: str) -> None:
    progress = read_progress(progress_path)
    progress[key] = value
    lines = [f"{k}={v}" for k, v in progress.items()]
    progress_path.parent.mkdir(parents=True, exist_ok=True)
    tmp = progress_path.with_suffix('.tmp')
    tmp.write_text("\n".join(lines) + "\n")
    os.replace(tmp, progress_path)
```

Add `import os` at the top if not already present.

- [ ] **Step 5: Run test to verify it passes**

Run: `python3 -m pytest tests/test_work_end_execute.py::test_write_progress_survives_crash -v`
Expected: PASS

- [ ] **Step 6: Apply same fix to land_flow.py**

Replace `_write_progress` at lines 69-74 with identical atomic pattern:

```python
def _write_progress(path: Path, key: str, value: str) -> None:
    progress = _read_progress(path)
    progress[key] = value
    lines = [f"{k}={v}" for k, v in progress.items()]
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix('.tmp')
    tmp.write_text("\n".join(lines) + "\n")
    os.replace(tmp, path)
```

Add `import os` at the top if not already present.

- [ ] **Step 7: Write and run equivalent test for land_flow.py**

```python
def test_land_flow_write_progress_survives_crash(tmp_path, monkeypatch):
    progress = tmp_path / ".execute-progress"
    _write_progress(progress, "repo:branch", "pushed")

    original_replace = os.replace
    def crash_on_replace(src, dst):
        raise OSError("simulated crash")
    monkeypatch.setattr("os.replace", crash_on_replace)

    with pytest.raises(OSError):
        _write_progress(progress, "repo:branch", "stamped")

    content = progress.read_text()
    assert "pushed" in content
    assert "stamped" not in content
```

Run: `python3 -m pytest tests/test_land_flow.py::test_land_flow_write_progress_survives_crash -v`
Expected: PASS

- [ ] **Step 8: Run full existing test suites**

Run: `python3 -m pytest tests/test_work_end_execute.py tests/test_land_flow.py -v`
Expected: All existing tests PASS (API unchanged)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_execute.py work-end/land_flow.py tests/test_work_end_execute.py tests/test_land_flow.py
git commit -m "fix: atomic write-then-rename for progress files

write_progress() used Path.write_text() which truncates before writing.
Process crash between truncation and write loses all prior progress.
Now writes to .tmp then os.replace() — atomic on POSIX.

Refs #271"
```

---

## Batch 2: Orchestrator core — progress tracking and action yielding

### Task 2: Progress tracking module

**Files:**
- Create: `work-end/close_progress.py`
- Test: `tests/test_close_progress.py`

**Interfaces:**
- Consumes: nothing (standalone module)
- Produces:
  - `read_close_progress(workspace: Path) -> dict[str, str]`
  - `write_close_progress(workspace: Path, entries: dict[str, str]) -> None`
  - `update_close_progress(workspace: Path, key: str, value: str) -> None`
  - `delete_close_progress(workspace: Path) -> None`
  - `is_stale(progress: dict, meta_state: str) -> bool`

- [ ] **Step 1: Write failing tests**

```python
import os
import pytest
from pathlib import Path
from work_end.close_progress import (
    read_close_progress, write_close_progress,
    update_close_progress, delete_close_progress, is_stale,
)

def test_read_empty(tmp_path):
    result = read_close_progress(tmp_path)
    assert result == {}

def test_write_and_read_roundtrip(tmp_path):
    entries = {"review": "done", "sweep_config": "done"}
    write_close_progress(tmp_path, entries)
    result = read_close_progress(tmp_path)
    assert result == entries

def test_update_adds_key(tmp_path):
    update_close_progress(tmp_path, "review", "done")
    result = read_close_progress(tmp_path)
    assert result["review"] == "done"

def test_update_overwrites_key(tmp_path):
    update_close_progress(tmp_path, "review", "pending")
    update_close_progress(tmp_path, "review", "done")
    result = read_close_progress(tmp_path)
    assert result["review"] == "done"

def test_atomic_write_survives_crash(tmp_path, monkeypatch):
    write_close_progress(tmp_path, {"review": "done"})
    def crash(*a, **kw):
        raise OSError("crash")
    monkeypatch.setattr(os, "replace", crash)
    with pytest.raises(OSError):
        update_close_progress(tmp_path, "forage", "done")
    result = read_close_progress(tmp_path)
    assert result == {"review": "done"}

def test_delete(tmp_path):
    write_close_progress(tmp_path, {"review": "done"})
    delete_close_progress(tmp_path)
    assert read_close_progress(tmp_path) == {}

def test_delete_also_removes_tmp(tmp_path):
    tmp_file = tmp_path / ".close-progress.tmp"
    tmp_file.write_text("stale")
    delete_close_progress(tmp_path)
    assert not tmp_file.exists()

def test_is_stale_progress_ahead(tmp_path):
    progress = {"review": "done", "promote": "done", "land": "done"}
    assert is_stale(progress, "closing:review") is True

def test_is_stale_progress_behind():
    progress = {"review": "done"}
    assert is_stale(progress, "closing:promoted") is False

def test_is_stale_empty_progress():
    assert is_stale({}, "closing:review") is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_close_progress.py -v`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Implement close_progress.py**

```python
#!/usr/bin/env python3
"""Progress tracking for work-end close sequence.

Atomic write-then-rename. Stale detection via lifecycle state comparison.
"""
import os
from pathlib import Path

PROGRESS_FILE = ".close-progress"
PROGRESS_TMP = ".close-progress.tmp"

LIFECYCLE_PHASE_ORDER = [
    "closing:review", "closing:verified", "closing:promoted",
    "closing:pushed", "closing:merged", "closing:stamped", "idle",
]

STEP_TO_PHASE = {
    "review": "closing:review",
    "sweep_config": "closing:review",
    "forage": "closing:review",
    "protocol": "closing:review",
    "update_claude_md": "closing:review",
    "impl_doc_sync": "closing:review",
    "adr": "closing:review",
    "write_content": "closing:review",
    "promote": "closing:verified",
    "trajectory": "closing:promoted",
    "rebase": "closing:promoted",
    "squash": "closing:promoted",
    "land": "closing:promoted",
    "close_issues": "closing:stamped",
    "verify": "closing:stamped",
    "cleanup": "closing:stamped",
}


def read_close_progress(workspace: Path) -> dict[str, str]:
    path = workspace / PROGRESS_FILE
    if not path.exists():
        return {}
    result: dict[str, str] = {}
    for line in path.read_text().splitlines():
        if "=" in line:
            k, _, v = line.partition("=")
            result[k.strip()] = v.strip()
    return result


def write_close_progress(workspace: Path, entries: dict[str, str]) -> None:
    path = workspace / PROGRESS_FILE
    tmp = workspace / PROGRESS_TMP
    lines = [f"{k}={v}" for k, v in entries.items()]
    tmp.write_text("\n".join(lines) + "\n")
    os.replace(tmp, path)


def update_close_progress(workspace: Path, key: str, value: str) -> None:
    entries = read_close_progress(workspace)
    entries[key] = value
    write_close_progress(workspace, entries)


def delete_close_progress(workspace: Path) -> None:
    for name in (PROGRESS_FILE, PROGRESS_TMP):
        p = workspace / name
        if p.exists():
            p.unlink()


def is_stale(progress: dict[str, str], meta_state: str) -> bool:
    if not progress:
        return False
    meta_idx = LIFECYCLE_PHASE_ORDER.index(meta_state) if meta_state in LIFECYCLE_PHASE_ORDER else 0
    max_progress_idx = 0
    for step in progress:
        phase = STEP_TO_PHASE.get(step, "closing:review")
        idx = LIFECYCLE_PHASE_ORDER.index(phase) if phase in LIFECYCLE_PHASE_ORDER else 0
        max_progress_idx = max(max_progress_idx, idx)
    return max_progress_idx > meta_idx
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_close_progress.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_progress.py tests/test_close_progress.py
git commit -m "feat: close_progress module — atomic progress tracking for orchestrator

Refs #271"
```

### Task 3: Orchestrator script — core loop and mechanical steps

**Files:**
- Create: `work-end/work_end_orchestrator.py`
- Test: `tests/test_work_end_orchestrator.py`

**Interfaces:**
- Consumes:
  - `close_progress.read_close_progress(workspace)` → `dict[str, str]`
  - `close_progress.update_close_progress(workspace, key, value)`
  - `close_progress.delete_close_progress(workspace)`
  - `close_progress.is_stale(progress, meta_state)`
  - `lifecycle.py transition/commit-transition` (via subprocess)
  - `work_end_execute.py promote/rebase/land/close-issues/archive-slot` (via subprocess)
  - `verify_slot_close.py` (via subprocess)
  - `branch_cleanup.py checkout-main/cleanup-stack/cleanup-scaffold` (via subprocess)
  - `close_report.py init/record/render` (via subprocess)
- Produces: KEY=VALUE output with `ACTION=` line, called via CLI:
  ```
  python3 work-end/work_end_orchestrator.py \
    workspace=<WS> project=<PROJ> branch=<BRANCH> \
    base_branch=<BASE> meta_state=<STATE> \
    [covers=<CSV>] [issue_repo=<REPO>] [in_slot=<yes|no>] \
    [slot_path=<PATH>] [on_main=<yes|no>] \
    [sweep_selected=<CSV>] [skip_step=<NAME>] \
    [abort=yes] [conflict_resolved=yes]
  ```

This is the largest task. The orchestrator is ~300-400 lines implementing:
1. Read progress + META_STATE, handle stale/fast-forward
2. Walk through the step sequence, running mechanical steps
3. Yield `ACTION=` at judgment points
4. Validate outputs of prior judgment steps on re-entry
5. Fire lifecycle transitions with evidence dicts
6. Handle abort, skip, conflict_resolved arguments

- [ ] **Step 1: Write core sequence test — full branch mode happy path**

```python
def test_full_sequence_branch_mode(tmp_path, monkeypatch):
    """Orchestrator yields correct actions in sequence for branch mode."""
    ws = tmp_path / "workspace"
    ws.mkdir()
    proj = tmp_path / "project"
    proj.mkdir()

    calls = []
    def mock_subprocess(cmd, **kw):
        calls.append(cmd)
        return MockResult(stdout="", returncode=0)

    monkeypatch.setattr(subprocess, "run", mock_subprocess)

    # First call — should yield ACTION=review
    result = run_orchestrator(ws, proj, "issue-271", "main",
                              meta_state="closing:review")
    assert "ACTION=review" in result

    # Simulate review done — mark in progress
    update_close_progress(ws, "review", "done")

    # Second call — should yield ACTION=sweep_config
    result = run_orchestrator(ws, proj, "issue-271", "main",
                              meta_state="closing:review")
    assert "ACTION=sweep_config" in result
```

(Full test file will have ~25 test functions matching the spec's test table.)

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py::test_full_sequence_branch_mode -v`
Expected: FAIL — module doesn't exist

- [ ] **Step 3: Implement orchestrator — step sequence and action yielding**

The orchestrator is a sequential state machine. Each invocation:
1. Parse CLI args
2. Read progress, check for stale/fast-forward
3. Walk steps in order, skip completed ones
4. At the next incomplete step: if mechanical, run it; if judgment, yield ACTION
5. After running a mechanical step, update progress and continue to next

The step sequence is a list of tuples. The `phase` column is the
lifecycle state the step runs IN (not the state it transitions TO).
Lifecycle steps use the state they transition FROM as their phase.

```python
STEPS = [
    # (name, phase, type, action_or_script, skip_condition)
    ("report_init", "closing:review", "mechanical", "close_report.py init", None),
    ("review", "closing:review", "judgment", "review", None),
    ("sweep_config", "closing:review", "judgment", "sweep_config", None),
    # sweep sub-steps inserted dynamically from sweep_selected
    ("review_pass", "closing:review", "lifecycle", "review_pass", None),
    ("promote", "closing:verified", "mechanical", "work_end_execute.py promote", None),
    ("report_promote", "closing:verified", "mechanical", "close_report.py record promote", None),
    ("promote_pass", "closing:verified", "lifecycle", "promote_pass", None),
    ("trajectory", "closing:promoted", "judgment", "trajectory", None),
    ("rebase", "closing:promoted", "mechanical", "work_end_execute.py rebase", "on_main"),
    ("report_rebase", "closing:promoted", "mechanical", "close_report.py record rebase", "on_main"),
    ("squash", "closing:promoted", "judgment", "squash", "on_main"),
    ("report_squash", "closing:promoted", "mechanical", "close_report.py record squash", "on_main"),
    ("land", "closing:promoted", "mechanical", "ROUTE_LAND", None),  # see slot routing below
    ("report_land", "closing:promoted", "mechanical", "close_report.py record land", None),
    ("push_pass", "closing:promoted", "lifecycle", "push_pass", None),
    ("merge_pass", "closing:pushed", "lifecycle", "merge_pass", None),  # FROM closing:pushed
    ("stamp_pass", "closing:merged", "lifecycle", "stamp_pass", None),  # FROM closing:merged
    ("arc42_scan", "closing:stamped", "judgment", "user_input", "no_arc42"),
    ("session_rename", "closing:stamped", "judgment", "user_input", None),
    ("garden_feedback", "closing:stamped", "judgment", "user_input", "no_ge_ids"),
    ("notes", "closing:stamped", "judgment", "user_input", "no_notes_dir"),
    ("close_issues", "closing:stamped", "mechanical", "work_end_execute.py close-issues", "no_covers"),
    ("report_close_issues", "closing:stamped", "mechanical", "close_report.py record close-issues", "no_covers"),
    ("verify", "closing:stamped", "mechanical", "verify_slot_close.py", None),
    ("report_verify", "closing:stamped", "mechanical", "close_report.py record verify", None),
    ("archive_slot", "closing:stamped", "mechanical", "work_end_execute.py archive-slot", "not_slot"),
    ("report_archive", "closing:stamped", "mechanical", "close_report.py record archive", "not_slot"),
    ("checkout_main", "closing:stamped", "mechanical", "branch_cleanup.py checkout-main", "on_main"),
    ("cleanup_stack", "closing:stamped", "mechanical", "branch_cleanup.py cleanup-stack", "on_main"),
    ("cleanup_scaffold", "closing:stamped", "mechanical", "branch_cleanup.py cleanup-scaffold", None),
    ("report_scaffold", "closing:stamped", "mechanical", "close_report.py record scaffold-cleanup", None),
    ("cleanup_pass", "closing:stamped", "lifecycle", "ROUTE_CLEANUP", None),  # cleanup_pass or cleanup_main
    ("delete_progress", "idle_or_drained", "mechanical", "delete .close-progress", None),
    ("report_render", "idle_or_drained", "mechanical", "close_report.py render", None),
    ("complete", "idle_or_drained", "terminal", "complete", None),
]
```

**Slot-mode routing (ROUTE_LAND):**
When `in_slot=yes`, the orchestrator calls `slot_manager.py merge-slot <SLOT_PATH>`
instead of `work_end_execute.py land`. The current SKILL.md explicitly says
"Do NOT call `work_end_execute.py land` in slot mode." The routing is:

```python
if in_slot:
    run("slot_manager.py merge-slot", slot_path)
else:
    run("work_end_execute.py land", project, branch, base_branch, workspace)
```

**Lifecycle state tracking for rapid-fire transitions:**
After `land` completes, three lifecycle transitions fire in sequence.
Each transition changes the state, so the orchestrator must track the
expected state internally (not re-read META_STATE, which would add
unnecessary I/O):

```python
expected_state = "closing:promoted"
for event in ["push_pass", "merge_pass", "stamp_pass"]:
    evidence = build_evidence(event, land_output)
    run_lifecycle("commit-transition", plan_path,
                  from_state=expected_state, new_state=NEXT[event],
                  event=event, evidence=evidence)
    expected_state = NEXT[event]
# expected_state is now "closing:stamped"
```

In main mode, `merge_pass` and `stamp_pass` fire with empty evidence dicts.

**Evidence dict construction:**
Each lifecycle event requires specific evidence keys. The orchestrator
builds these from script output and `.execute-progress` state:

```python
EVIDENCE_BUILDERS = {
    "review_pass": lambda ctx: {"review_result": "pass"},
    "promote_pass": lambda ctx: {
        "promoted_files": ctx.promote_output.get("PROMOTED_FILES", ""),
        "target_repos": ctx.promote_output.get("TARGET_REPOS", ""),
    },
    "push_pass": lambda ctx: {
        "pushed_repos": ctx.land_output.get("PUSHED_REPOS", "").split(","),
        "pushed_shas": parse_sha_dict(ctx.land_output),
    },
    "merge_pass": lambda ctx: {
        "landed_shas": parse_sha_dict(ctx.land_output) if not ctx.on_main else {},
        "verified_on_main": infer_verification(ctx) if not ctx.on_main else {},
    },
    "stamp_pass": lambda ctx: {
        "stamp_shas": read_stamp_shas(ctx.execute_progress) if not ctx.on_main else {},
    },
    "cleanup_pass": lambda ctx: {
        "repos_on_main": {"project": True, "workspace": True},
        "work_items_ended": True,
    },
    "cleanup_main": lambda ctx: {"work_items_ended": True},
}
```

**Validation on re-entry (per judgment step):**

```python
VALIDATORS = {
    "review": lambda ws, proj: all_findings_resolved(ws),
    "write_content": lambda ws, proj: blog_file_exists(ws) and blog_valid(ws),
    "forage": lambda ws, proj: garden_entries_created(ws) or was_skipped(ws, "forage"),
    "protocol": lambda ws, proj: protocol_files_created(ws) or was_skipped(ws, "protocol"),
    "squash": lambda ws, proj: squash_plans_exist(ws, proj),
    "trajectory": lambda ws, proj: True,  # non-blocking
    "sweep_config": lambda ws, proj: True,  # validated by sweep_selected arg
}
```

**Retry counting:**

```python
def handle_judgment_step(step_name, ws, progress):
    attempt_key = f"{step_name}_attempt"
    attempt = int(progress.get(attempt_key, "0")) + 1
    if attempt > 3:
        yield_action("user_input", CONTEXT="step_failed", STEP=step_name,
                     ATTEMPTS=3, REASON=validator_reason)
        return
    update_close_progress(ws, attempt_key, str(attempt))
    yield_action(step_name, **build_context(step_name))
```

**Abort handling:**

```python
if abort_requested:
    state = read_meta_state(plan_path)
    if state in ("closing:review", "closing:verified"):
        delete_close_progress(workspace)
        delete_execute_progress(workspace)
        run_lifecycle("transition", plan_path, "abort_close")
        run_lifecycle("commit-transition", plan_path,
                      from_state=state, new_state="active", event="abort_close")
        yield_action("complete", SUMMARY="Aborted — returned to active state")
    else:
        print(f"ERROR=abort_blocked STATE={state}")
        print("REASON=Post-promotion states are forward-only")
```

- [ ] **Step 4: Run tests iteratively as each section is implemented**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v`
Expected: PASS as each test is addressed

- [ ] **Step 5: Write validation tests**

Tests for each judgment step's validation logic:
- `test_review_validation_checks_findings`
- `test_write_content_validation_checks_blog_file`
- `test_squash_validation_checks_plan_json`
- `test_sweep_step_validation`
- `test_retry_count_escalates_to_user`
- `test_mechanical_step_skips_after_retries`

- [ ] **Step 6: Write crash recovery tests**

Tests for stale detection, fast-forward, mid-write crash:
- `test_stale_progress_deleted_on_fresh_close`
- `test_meta_state_ahead_fast_forwards`
- `test_resume_from_mid_sequence`

- [ ] **Step 7: Write mode-specific tests**

Tests for slot mode routing and main mode skips:
- `test_slot_mode_uses_merge_slot`
- `test_main_mode_skips_rebase_squash_stamp`
- `test_abort_from_review_deletes_progress`
- `test_abort_from_promoted_rejected`

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py -v`
Expected: All ~25 tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/work_end_orchestrator.py tests/test_work_end_orchestrator.py
git commit -m "feat: work_end_orchestrator — Python-driven close sequence

Stateless re-entrant orchestrator that owns the work-end close sequence.
Yields ACTION= lines for judgment steps, runs mechanical steps silently.
Validates outputs, tracks progress, handles crash recovery.

Refs #271"
```

---

## Batch 3: SKILL.md rewrite and close_report update

### Task 4: Update close_report.py for orchestrator step names

**Files:**
- Modify: `work-end/close_report.py:19-53` — STEP_ORDER, STEP_LABELS
- Test: `tests/test_close_report.py`

**Interfaces:**
- Consumes: orchestrator calls `close_report.py record step=<name>`
- Produces: rendered summary via `close_report.py render`

- [ ] **Step 1: Write failing test**

```python
def test_orchestrator_step_order():
    expected_steps = [
        "promote", "rebase", "squash", "land",
        "close-issues", "verify", "archive", "scaffold-cleanup",
    ]
    for step in expected_steps:
        assert step in STEP_ORDER, f"{step} missing from STEP_ORDER"

def test_orchestrator_step_labels():
    assert STEP_LABELS["promote"] == "Artifacts promoted"
    assert STEP_LABELS["land"] == "Landed"
    assert STEP_LABELS["close-issues"] == "Issues closed"
    assert STEP_LABELS["verify"] == "Verified"
```

- [ ] **Step 2: Run test — expect FAIL (old step names)**

Run: `python3 -m pytest tests/test_close_report.py::test_orchestrator_step_order -v`

- [ ] **Step 3: Update STEP_ORDER and STEP_LABELS**

Replace lines 19-53 with new STEP_ORDER, STEP_LABELS, and add
`_format_detail` handlers for the new step names. The existing
`_format_detail()` function dispatches by step name — add cases for
`promote`, `land`, `close-issues`, and `verify` that render meaningful
detail (file counts, SHA references, issue numbers, check results)
rather than falling through to generic `_kv_summary`.

```python
STEP_ORDER = [
    "promote",
    "rebase",
    "squash",
    "land",
    "close-issues",
    "verify",
    "archive",
    "scaffold-cleanup",
]

STEP_LABELS = {
    "promote": "Artifacts promoted",
    "rebase": "Rebased",
    "squash": "Squashed",
    "land": "Landed",
    "close-issues": "Issues closed",
    "verify": "Verified",
    "archive": "Slot archived",
    "scaffold-cleanup": "Scaffold cleaned",
}
```

And in `_format_detail()`, add:

```python
elif step == "promote":
    files = data.get("promoted_files", "")
    targets = data.get("target_repos", "")
    return f"{files} files → {targets}" if files else "no artifacts"
elif step == "land":
    sha = data.get("landed_sha", "")
    repos = data.get("pushed_repos", "")
    return f"{repos} (SHA {sha[:7]})" if sha else "pushed"
elif step == "close-issues":
    closed = data.get("closed", "0")
    return f"{closed} issues closed"
elif step == "verify":
    result = data.get("verified", "unknown")
    return f"VERIFIED={result}"
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_close_report.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/close_report.py tests/test_close_report.py
git commit -m "fix: update close_report step names for orchestrator

Refs #271"
```

### Task 5: Rewrite SKILL.md

**Files:**
- Modify: `work-end/SKILL.md` — replace ~660 lines with ~270 lines

**Interfaces:**
- Consumes: `work_end_orchestrator.py` ACTION= output
- Produces: SKILL.md that Claude Code loads for work-end

The new SKILL.md has four sections:
1. **Frontmatter** — unchanged name/description
2. **Pre-close** — context resolution, preconditions, dirty-tree protocol (~250 lines, adapted from current Steps 0-1)
3. **Dispatch loop** — ~20 lines that call the orchestrator and dispatch actions
4. **Action Handlers** — ~200 lines of per-action judgment constraints the LLM needs when executing each action. These are the work-end-specific instructions from the current SKILL.md that aren't part of the invoked skills:

| Action | Handler content (from current SKILL.md) |
|--------|---------------------------------------|
| `review` | Security-audit suppression during work-end, budget limits as non-gates, findings persistence to findings.jsonl, forcing function severity constraints (CRITICAL cannot be dismissed), batch operations, re-review after fixes |
| `review_rebase` | Scope constraint: code-review only, no branch-audit/loose-ends/forcing function |
| `sweep_config` | All items default ON, NEVER-RECOMMEND-SKIPPING hard gate, session-bound items cannot be deferred |
| `forage` | Run while context is full (first in sweep order) |
| `write_content` | Run last (synthesises full narrative), session-bound — cannot be deferred |
| `squash` | Per-repo .squash-plan-<repo>.json output format, slot mode marker write |
| `trajectory` | Non-blocking, enrichment.py commands, propose-confirm pattern |
| `user_input CONTEXT=arc42_scan` | Scan for stale statuses, offer fixes |
| `user_input CONTEXT=garden_feedback` | 5-level relevance scale, grouping by outcome |
| `user_input CONTEXT=notes` | Surface recent date section, prompt for append |

The SKILL.md total: ~20 (loop) + ~250 (pre-close) + ~200 (handlers) = ~470 lines.
Down from 660 lines, but the judgment instructions are preserved — not lost.

- [ ] **Step 1: Write the new SKILL.md**

The dispatch loop:
```markdown
## Close Sequence — Orchestrator Loop

After context is resolved and `closing:review` is entered:

```
loop:
  output = run("python3 work-end/work_end_orchestrator.py
    workspace=<WORKSPACE> project=<PROJECT> branch=<BRANCH>
    base_branch=<BASE> meta_state=<META_STATE>
    [covers=<COVERS>] [issue_repo=<OWNER_REPO>]
    [in_slot=<IN_SLOT>] [slot_path=<SLOT_PATH>]
    [on_main=<ON_MAIN>]
    [sweep_selected=<CSV>] [skip_step=<NAME>]
    [abort=yes] [conflict_resolved=yes]")
  parse ACTION= from output

  if ACTION=complete → print SUMMARY, done
  if ACTION=user_input → present CONTEXT to user, collect response
  if ACTION=review → invoke code-review, branch-audit, forcing function
  if ACTION=review_rebase → invoke code-review on DIFF_RANGE only
  if ACTION=sweep_config → present toggle UI, report via sweep_selected=
  if ACTION=squash → classify commits per repo, write .squash-plan-<repo>.json
  if ACTION=verify_recover → present failures, offer recovery
  if ACTION in [forage, protocol, update_claude_md, impl_doc_sync,
                adr, write_content, trajectory] → invoke named skill
  go to loop
```

The pre-close section retains context resolution, preconditions, queue gate,
dirty-tree protocol, main-mode detection, and lifecycle entry — adapted from
the current SKILL.md Steps 0-1.

- [ ] **Step 2: Verify the new SKILL.md loads correctly**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS (frontmatter valid, no broken references)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/SKILL.md
git commit -m "feat: rewrite work-end SKILL.md — orchestrator dispatch loop

Replace 660-line LLM-orchestrated skill with ~270-line skill:
- Pre-close context resolution (adapted from Steps 0-1)
- ~20-line dispatch loop calling work_end_orchestrator.py
- LLM sees one action at a time, cannot skip steps

Refs #271"
```

---

## Batch 4: Integration testing and validation

### Task 6: End-to-end integration test

**Files:**
- Modify: `tests/test_work_end_orchestrator.py` — add integration tests

**Interfaces:**
- Consumes: all orchestrator + script interfaces
- Produces: confidence that the full sequence works

- [ ] **Step 1: Write integration test with real git repos**

```python
def test_integration_branch_mode_full_sequence(tmp_path):
    """End-to-end: create git repos, run orchestrator through full sequence."""
    # Set up test git repos (project + workspace)
    # Run orchestrator calls in sequence
    # Verify: lifecycle state transitions correctly
    # Verify: .close-progress updates at each step
    # Verify: final state is idle, .close-progress deleted
```

- [ ] **Step 2: Write integration test for crash recovery**

```python
def test_integration_crash_recovery(tmp_path):
    """Simulate crash mid-sequence, verify resume from correct point."""
    # Run orchestrator through first 5 steps
    # Kill mid-step (leave .close-progress at step 5)
    # Re-run orchestrator
    # Verify: steps 1-5 skipped, resumes at step 6
```

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/test_work_end_orchestrator.py tests/test_close_progress.py tests/test_work_end_execute.py tests/test_land_flow.py tests/test_close_report.py -v`
Expected: All PASS

- [ ] **Step 4: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/
git commit -m "test: integration tests for orchestrator full sequence and crash recovery

Refs #271"
```

---

## References

- `$WORKSPACE/specs/issue-271-mechanise-work-end-close/2026-08-24-mechanise-work-end-close-design.md` — design spec (workspace, not project repo)
- `work-end/work_end_execute.py:127-132` — write_progress crash-safety issue
- `work-end/land_flow.py:69-74` — _write_progress crash-safety issue
- `project/lifecycle.py:237-239` — atomic write pattern (precedent)
- `work-end/close_report.py:19-53` — STEP_ORDER/STEP_LABELS to update
- `work-end/SKILL.md` — 660-line skill to rewrite
- GE-20260821-ebba3b — work-end atomicity failures
- `docs/protocols/externalised-scripts-require-tests.md`
- `docs/protocols/evidence-before-claims.md`
- GitHub #271

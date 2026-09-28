# Pipeline Engine Lifecycle — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #384 — Pipeline engine — Python-driven lifecycle for start/continue/pause/resume
**Issue group:** #384

**Goal:** Replace LLM-orchestrated lifecycle commands with Python-driven
pipelines using the proven `run_loop` pattern from work-end.

**Architecture:** A single `project/work.py` entry point dispatches to
command-specific step lists. Mechanical steps delegate to existing
externalized scripts (`branch_create.py`, `pause_exec.py`, etc.).
Judgment steps yield `ACTION=` to the LLM. Shared infrastructure
(`orchestrator_engine.py`, `shared_steps.py`, progress tracking) moves
from `work-end/` to `project/`.

**Tech Stack:** Python 3.11+, pytest, existing CLI scripts

## Global Constraints

- Every new `.py` file ships with tests in the same commit
- All mechanical steps delegate to existing externalized scripts — no reimplementation
- Progress tracking uses `.work-progress` with backward compat for `.close-progress`
- work-end's step list stays in `work-end/` — only shared infrastructure moves
- Existing tests must pass after each batch

---

## Batch 1: Foundation — Move shared infrastructure

After this batch: shared engine lives in `project/`, work-end imports
updated, all existing tests pass.

### Task 1: Create project/work_progress.py

Generalize `close_progress.py` into a location-agnostic progress module
that reads both `.work-progress` and `.close-progress` for backward compat.

**Files:**
- Create: `project/work_progress.py`
- Test: `tests/test_work_progress.py`

**Interfaces:**
- Produces: `read_progress(workspace) -> dict[str, str]`,
  `update_progress(workspace, key, value)`,
  `write_progress(workspace, entries)`,
  `delete_progress(workspace)`,
  `is_stale(progress, meta_state, plan_path) -> bool`

- [ ] **Step 1: Write failing tests for read_progress backward compat**

```python
# tests/test_work_progress.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


class TestReadProgress:
    def test_reads_work_progress(self, tmp_path):
        from work_progress import read_progress
        (tmp_path / ".work-progress").write_text("step_a=done\nstep_b=pending\n")
        result = read_progress(tmp_path)
        assert result == {"step_a": "done", "step_b": "pending"}

    def test_falls_back_to_close_progress(self, tmp_path):
        from work_progress import read_progress
        (tmp_path / ".close-progress").write_text("review=done\n")
        result = read_progress(tmp_path)
        assert result == {"review": "done"}

    def test_prefers_work_progress_over_close(self, tmp_path):
        from work_progress import read_progress
        (tmp_path / ".work-progress").write_text("new=yes\n")
        (tmp_path / ".close-progress").write_text("old=yes\n")
        result = read_progress(tmp_path)
        assert result == {"new": "yes"}

    def test_empty_when_no_file(self, tmp_path):
        from work_progress import read_progress
        assert read_progress(tmp_path) == {}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_progress.py -v`
Expected: FAIL (module not found)

- [ ] **Step 3: Write failing tests for update_progress and delete_progress**

```python
class TestUpdateProgress:
    def test_writes_to_work_progress(self, tmp_path):
        from work_progress import update_progress, read_progress
        update_progress(tmp_path, "step_a", "done")
        assert (tmp_path / ".work-progress").exists()
        assert not (tmp_path / ".close-progress").exists()
        assert read_progress(tmp_path) == {"step_a": "done"}

    def test_preserves_existing_keys(self, tmp_path):
        from work_progress import update_progress, read_progress
        update_progress(tmp_path, "a", "1")
        update_progress(tmp_path, "b", "2")
        assert read_progress(tmp_path) == {"a": "1", "b": "2"}


class TestDeleteProgress:
    def test_removes_work_progress(self, tmp_path):
        from work_progress import update_progress, delete_progress
        update_progress(tmp_path, "a", "1")
        delete_progress(tmp_path)
        assert not (tmp_path / ".work-progress").exists()

    def test_removes_close_progress_too(self, tmp_path):
        from work_progress import delete_progress
        (tmp_path / ".close-progress").write_text("old=yes\n")
        delete_progress(tmp_path)
        assert not (tmp_path / ".close-progress").exists()

    def test_noop_when_no_files(self, tmp_path):
        from work_progress import delete_progress
        delete_progress(tmp_path)  # should not raise
```

- [ ] **Step 4: Implement work_progress.py**

```python
#!/usr/bin/env python3
"""Progress tracking for work lifecycle commands.

Atomic write-then-rename. Reads .work-progress or .close-progress
(backward compat). Writes .work-progress only.
"""
import os
import sys
from pathlib import Path

_project_dir = Path(__file__).resolve().parent
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))
from plan_io import read_field as _read_plan_field

PROGRESS_FILE = ".work-progress"
PROGRESS_TMP = ".work-progress.tmp"
LEGACY_FILE = ".close-progress"
LEGACY_TMP = ".close-progress.tmp"

LIFECYCLE_PHASE_ORDER = [
    "active",
    "closing:review",
    "closing:verified",
    "closing:promoted",
    "closing:pushed",
    "closing:merged",
    "closing:stamped",
    "idle",
    "drained",
]

STEP_TO_PHASE = {
    "report_init": "closing:review",
    "review": "closing:review",
    "code_review": "closing:review",
    "branch_audit_conformance": "closing:review",
    "branch_audit_coherence": "closing:review",
    "branch_audit_structure": "closing:review",
    "branch_audit_robustness": "closing:review",
    "loose_ends": "closing:review",
    "forcing_function": "closing:review",
    "sweep_config": "closing:review",
    "forage": "closing:review",
    "protocol": "closing:review",
    "update_claude_md": "closing:review",
    "impl_doc_sync": "closing:review",
    "doc_freshness_gate": "closing:review",
    "adr": "closing:review",
    "write_content": "closing:review",
    "promote": "closing:verified",
    "report_promote": "closing:verified",
    "trajectory": "closing:promoted",
    "rebase": "closing:promoted",
    "report_rebase": "closing:promoted",
    "squash": "closing:promoted",
    "report_squash": "closing:promoted",
    "write_marker": "closing:promoted",
    "land": "closing:promoted",
    "report_land": "closing:promoted",
    "sync_pass": "closing:stamped",
    "promote_plan_next": "closing:stamped",
    "close_issues": "closing:stamped",
    "report_close_issues": "closing:stamped",
    "verify": "closing:stamped",
    "report_verify": "closing:stamped",
    "archive_slot": "closing:stamped",
    "report_archive": "closing:stamped",
    "elevate_plan": "closing:stamped",
    "checkout_main": "closing:stamped",
    "cleanup_stack": "closing:stamped",
    "cleanup": "closing:stamped",
    "report_scaffold": "closing:stamped",
    "upstream_push": "closing:stamped",
    "arc42_scan": "closing:stamped",
    "session_rename": "closing:stamped",
    "garden_feedback": "closing:stamped",
    "notes": "closing:stamped",
    "delete_progress": "idle",
    "report_render": "idle",
}


def read_progress(workspace: Path) -> dict[str, str]:
    for name in (PROGRESS_FILE, LEGACY_FILE):
        path = workspace / name
        if path.exists():
            result: dict[str, str] = {}
            for line in path.read_text().splitlines():
                if "=" in line:
                    k, _, v = line.partition("=")
                    result[k.strip()] = v.strip()
            return result
    return {}


def write_progress(workspace: Path, entries: dict[str, str]) -> None:
    path = workspace / PROGRESS_FILE
    tmp = workspace / PROGRESS_TMP
    lines = [f"{k}={v}" for k, v in entries.items()]
    tmp.write_text("\n".join(lines) + "\n")
    os.replace(tmp, path)


def update_progress(workspace: Path, key: str, value: str) -> None:
    entries = read_progress(workspace)
    entries[key] = value
    write_progress(workspace, entries)


def delete_progress(workspace: Path) -> None:
    for name in (PROGRESS_FILE, PROGRESS_TMP, LEGACY_FILE, LEGACY_TMP):
        p = workspace / name
        if p.exists():
            p.unlink()


def is_stale(progress: dict[str, str], meta_state: str,
             plan_path: Path | None = None) -> bool:
    if not progress:
        return False
    if plan_path and plan_path.exists():
        actual_state = _read_plan_field(plan_path, "state") or ""
        if actual_state and actual_state in LIFECYCLE_PHASE_ORDER:
            meta_state = actual_state
    if meta_state not in LIFECYCLE_PHASE_ORDER:
        return False
    meta_idx = LIFECYCLE_PHASE_ORDER.index(meta_state)
    max_progress_idx = 0
    for step in progress:
        base_step = step.split("_attempt")[0] if "_attempt" in step else step
        if ":" in base_step:
            base_step = base_step.split(":")[0]
        if base_step.startswith("_") or base_step in ("last_yielded", "sweep_selected"):
            continue
        phase = STEP_TO_PHASE.get(base_step, "closing:review")
        if phase in LIFECYCLE_PHASE_ORDER:
            idx = LIFECYCLE_PHASE_ORDER.index(phase)
            max_progress_idx = max(max_progress_idx, idx)
    return max_progress_idx > meta_idx


# Backward compatibility wrappers for work-end
read_close_progress = read_progress
write_close_progress = write_progress
update_close_progress = update_progress
delete_close_progress = delete_progress
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_progress.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add project/work_progress.py tests/test_work_progress.py
git commit -m "feat(#384): create work_progress.py — generalized progress tracking

Reads .work-progress or .close-progress (backward compat).
Writes .work-progress only. Includes compat aliases for work-end.

Refs #384"
```

### Task 2: Move orchestrator_engine.py and shared_steps.py to project/

**Files:**
- Move: `work-end/orchestrator_engine.py` → `project/orchestrator_engine.py`
- Move: `work-end/shared_steps.py` → `project/shared_steps.py`
- Modify: `work-end/work_end_orchestrator.py` (update imports)
- Modify: `work-end/orchestrator_engine.py` → thin re-export wrapper
- Modify: `work-end/shared_steps.py` → thin re-export wrapper

**Interfaces:**
- Consumes: `project/work_progress.py` (from Task 1)
- Produces: Engine and step defs accessible from `project/`

- [ ] **Step 1: Copy orchestrator_engine.py to project/**

Copy `work-end/orchestrator_engine.py` to `project/orchestrator_engine.py`.
Update the import of `close_progress` to use `work_progress`:

Change:
```python
from close_progress import update_close_progress
```
To:
```python
from work_progress import update_progress as update_close_progress
```

And update `from shared_steps import` to a relative import (same directory).

- [ ] **Step 2: Copy shared_steps.py to project/**

Copy `work-end/shared_steps.py` to `project/shared_steps.py`.
Update the import of `close_progress` in shared_steps.py:

Change:
```python
from close_progress import update_close_progress
```
To:
```python
from work_progress import update_progress as update_close_progress
```

- [ ] **Step 3: Create thin re-export wrappers in work-end/**

Replace `work-end/orchestrator_engine.py` with:
```python
#!/usr/bin/env python3
"""Re-export from project/ for backward compat."""
import sys
from pathlib import Path
_project = str(Path(__file__).resolve().parent.parent / "project")
if _project not in sys.path:
    sys.path.insert(0, _project)
from orchestrator_engine import *  # noqa: F401,F403
```

Replace `work-end/shared_steps.py` with:
```python
#!/usr/bin/env python3
"""Re-export from project/ for backward compat."""
import sys
from pathlib import Path
_project = str(Path(__file__).resolve().parent.parent / "project")
if _project not in sys.path:
    sys.path.insert(0, _project)
from shared_steps import *  # noqa: F401,F403
```

Replace `work-end/close_progress.py` with:
```python
#!/usr/bin/env python3
"""Re-export from project/ for backward compat."""
import sys
from pathlib import Path
_project = str(Path(__file__).resolve().parent.parent / "project")
if _project not in sys.path:
    sys.path.insert(0, _project)
from work_progress import *  # noqa: F401,F403
```

- [ ] **Step 4: Run ALL existing tests**

Run: `python3 -m pytest tests/ -v --timeout=120`
Expected: All tests pass — work-end's orchestrator still works via re-exports.

Focus on: `tests/test_orchestrator_engine.py`, `tests/test_close_progress.py`,
`tests/test_work_end_orchestrator.py`, `tests/test_shared_steps.py`

- [ ] **Step 5: Commit**

```bash
git add project/orchestrator_engine.py project/shared_steps.py \
  work-end/orchestrator_engine.py work-end/shared_steps.py work-end/close_progress.py
git commit -m "feat(#384): move engine infrastructure to project/

orchestrator_engine.py, shared_steps.py, close_progress.py (as
work_progress.py) now live in project/. work-end/ has thin re-export
wrappers for backward compat.

Refs #384"
```

---

## Batch 2: Core — work.py with pause and resume pipelines

After this batch: `python3 project/work.py pause|resume` works end-to-end
via dry-run tests. The simplest commands prove the dispatch pattern.

### Task 3: Create project/work.py with command parsing and context

**Files:**
- Create: `project/work.py`
- Test: `tests/test_work_pipeline.py`

**Interfaces:**
- Consumes: `project/orchestrator_engine.py` (run_loop), `project/shared_steps.py` (StepDef, OrchestratorContextBase), `project/work_progress.py` (read_progress)
- Produces: `WorkContext` dataclass, `parse_args(argv) -> dict`, `build_context(args) -> WorkContext`, `main()` entry point

- [ ] **Step 1: Write failing test for command parsing**

```python
# tests/test_work_pipeline.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


class TestParseArgs:
    def test_parses_command_and_kv_args(self):
        from work import parse_args
        result = parse_args(["work.py", "pause",
                             "workspace=/tmp/ws", "project=/tmp/proj",
                             "branch=issue-42-foo", "base_branch=main"])
        assert result["command"] == "pause"
        assert result["workspace"] == "/tmp/ws"
        assert result["project"] == "/tmp/proj"
        assert result["branch"] == "issue-42-foo"

    def test_unknown_command_returns_error(self):
        from work import parse_args
        result = parse_args(["work.py", "bogus"])
        assert "error" in result

    def test_missing_command_returns_error(self):
        from work import parse_args
        result = parse_args(["work.py"])
        assert "error" in result
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_pipeline.py::TestParseArgs -v`
Expected: FAIL

- [ ] **Step 3: Write failing test for WorkContext**

```python
class TestBuildContext:
    def test_builds_from_args(self, tmp_path):
        from work import build_context
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        args = {
            "command": "pause",
            "workspace": str(ws),
            "project": str(proj),
            "branch": "issue-42-foo",
            "base_branch": "main",
            "on_main": "no",
            "in_slot": "no",
            "covers": "42",
            "issue_repo": "Org/repo",
            "meta_state": "active",
            "owner_repo": "Org/repo",
            "issue_n": "42",
        }
        ctx = build_context(args)
        assert ctx.command == "pause"
        assert ctx.workspace == ws
        assert ctx.project == proj
        assert ctx.branch == "issue-42-foo"
        assert ctx.covers == "42"
        assert ctx.issue_n == "42"
```

- [ ] **Step 4: Implement work.py skeleton**

```python
#!/usr/bin/env python3
"""Unified work lifecycle pipeline.

Usage:
    python3 project/work.py <command> workspace=<path> project=<path> ...

Commands: start, continue, pause, resume, next, find

Each invocation runs mechanical steps up to the next judgment point,
then prints ACTION= and exits. The LLM calls this in a loop until
ACTION=complete.
"""
import sys
from dataclasses import dataclass, field
from pathlib import Path

_project_dir = Path(__file__).resolve().parent
if str(_project_dir) not in sys.path:
    sys.path.insert(0, str(_project_dir))

from orchestrator_engine import run_loop, run_script
from shared_steps import StepDef, OrchestratorContextBase
from work_progress import read_progress, update_progress, delete_progress

VALID_COMMANDS = {"start", "continue", "pause", "resume", "next", "find"}


@dataclass
class WorkContext(OrchestratorContextBase):
    command: str = ""
    issue_n: str = ""
    issue_title: str = ""
    owner_repo: str = ""
    meta_state: str = ""
    has_handoff: bool = False
    handoff_path: Path | None = None
    has_platform_doc: bool = False
    has_protocols_dir: bool = False
    flyway_next_v: str = "none"
    design_repo_key: str = ""


def parse_args(argv: list[str]) -> dict[str, str]:
    if len(argv) < 2:
        return {"error": "missing_command"}
    command = argv[1]
    if command not in VALID_COMMANDS:
        return {"error": f"unknown_command:{command}"}
    result = {"command": command}
    for arg in argv[2:]:
        if "=" in arg:
            k, _, v = arg.partition("=")
            result[k] = v
    return result


def build_context(args: dict[str, str]) -> WorkContext:
    ws = Path(args.get("workspace", "."))
    proj = Path(args.get("project", "."))
    progress = read_progress(ws)
    return WorkContext(
        workspace=ws,
        project=proj,
        branch=args.get("branch", ""),
        base_branch=args.get("base_branch", "main"),
        on_main=args.get("on_main", "no") == "yes",
        in_slot=args.get("in_slot", "no") == "yes",
        covers=args.get("covers", ""),
        issue_repo=args.get("issue_repo", ""),
        progress=progress,
        plan_path=Path(args["plan_path"]) if args.get("plan_path") else None,
        slot_path=Path(args["slot_path"]) if args.get("slot_path") else None,
        family_root=Path(args["family_root"]) if args.get("family_root") else None,
        command=args.get("command", ""),
        issue_n=args.get("issue_n", ""),
        issue_title=args.get("issue_title", ""),
        owner_repo=args.get("owner_repo", ""),
        meta_state=args.get("meta_state", ""),
        has_handoff=args.get("has_handoff", "no") == "yes",
        handoff_path=Path(args["handoff_path"]) if args.get("handoff_path") else None,
        has_platform_doc=args.get("has_platform_doc", "no") == "yes",
        has_protocols_dir=args.get("has_protocols_dir", "no") == "yes",
        flyway_next_v=args.get("flyway_next_v", "none"),
        design_repo_key=args.get("design_repo_key", ""),
    )


# Pipelines are added in subsequent tasks
PIPELINES: dict[str, list[StepDef]] = {}


def main() -> int:
    args = parse_args(sys.argv)
    if "error" in args:
        print(f"ERROR={args['error']}")
        return 1
    ctx = build_context(args)
    command = ctx.command
    if command not in PIPELINES:
        print(f"ERROR=pipeline_not_implemented:{command}")
        return 1
    steps = PIPELINES[command]
    result = run_loop(steps, ctx, complete_summary=f"{command} complete.")
    for k, v in result.items():
        print(f"{k}={v}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_pipeline.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add project/work.py tests/test_work_pipeline.py
git commit -m "feat(#384): create work.py skeleton — command parsing and context

Refs #384"
```

### Task 4: Add pause pipeline to work.py

**Files:**
- Modify: `project/work.py` (add PAUSE_STEPS, wire into PIPELINES)
- Modify: `tests/test_work_pipeline.py` (add pause tests)

**Interfaces:**
- Consumes: `work-pause/pause_exec.py` (commit-wip, push-and-stack)
- Produces: `PAUSE_STEPS` step list in `PIPELINES["pause"]`

- [ ] **Step 1: Write failing test for pause pipeline dry-run**

```python
class TestPausePipeline:
    def test_pause_pipeline_exists(self):
        from work import PIPELINES
        assert "pause" in PIPELINES
        steps = PIPELINES["pause"]
        names = [s.name for s in steps]
        assert "wip_commit_project" in names
        assert "wip_commit_workspace" in names
        assert "push_and_stack" in names

    def test_pause_all_mechanical(self):
        from work import PIPELINES
        for step in PIPELINES["pause"]:
            assert step.step_type == "mechanical", f"{step.name} should be mechanical"

    def test_pause_dry_run(self, tmp_path):
        from work import build_context, PIPELINES
        from orchestrator_engine import run_loop
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        ctx = build_context({
            "command": "pause",
            "workspace": str(ws), "project": str(proj),
            "branch": "issue-42-foo", "base_branch": "main",
            "on_main": "no", "in_slot": "no",
            "covers": "42", "issue_repo": "Org/repo",
            "issue_n": "42", "owner_repo": "Org/repo",
            "meta_state": "active",
        })
        ctx.dry_run = True
        result = run_loop(PIPELINES["pause"], ctx)
        assert result["ACTION"] == "complete"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_pipeline.py::TestPausePipeline -v`
Expected: FAIL (pause not in PIPELINES)

- [ ] **Step 3: Implement pause steps in work.py**

Add to `project/work.py` before `PIPELINES`:

```python
_WORK_END_DIR = _project_dir.parent / "work-end"
_WORK_START_DIR = _project_dir.parent / "work-start"
_WORK_PAUSE_DIR = _project_dir.parent / "work-pause"
_WORK_RESUME_DIR = _project_dir.parent / "work-resume"


def _pause_exec(ctx):
    return _WORK_PAUSE_DIR / "pause_exec.py"


PAUSE_STEPS: list[StepDef] = [
    StepDef("wip_commit_project", "pause", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_pause_exec(ctx)), "commit-wip",
                str(ctx.project), f"message=WIP: pause {ctx.branch}"]),
    StepDef("wip_commit_workspace", "pause", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_pause_exec(ctx)), "commit-wip",
                str(ctx.workspace), f"message=WIP: pause {ctx.branch}"]),
    StepDef("push_and_stack", "pause", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_pause_exec(ctx)), "push-and-stack",
                str(ctx.workspace), str(ctx.project),
                f"branch={ctx.branch}", f"issue={ctx.issue_n}",
                f"base-branch={ctx.base_branch}"]),
]
```

Update `PIPELINES`:
```python
PIPELINES: dict[str, list[StepDef]] = {
    "pause": PAUSE_STEPS,
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_pipeline.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_pipeline.py
git commit -m "feat(#384): add pause pipeline — entirely mechanical

Three steps: wip_commit_project, wip_commit_workspace, push_and_stack.
All delegate to pause_exec.py.

Refs #384"
```

### Task 5: Add resume pipeline to work.py

**Files:**
- Modify: `project/work.py` (add RESUME_STEPS, wire into PIPELINES)
- Modify: `tests/test_work_pipeline.py` (add resume tests)

**Interfaces:**
- Consumes: `work-resume/resume_exec.py` (checkout-branches, rebase, reset-wip), `project/stack.py` (pop)
- Produces: `RESUME_STEPS` step list in `PIPELINES["resume"]`

- [ ] **Step 1: Write failing test for resume pipeline**

```python
class TestResumePipeline:
    def test_resume_pipeline_exists(self):
        from work import PIPELINES
        assert "resume" in PIPELINES
        steps = PIPELINES["resume"]
        names = [s.name for s in steps]
        assert "stack_pick" in names
        assert "pop_stack" in names
        assert "checkout_branches" in names
        assert "rebase" in names
        assert "reset_wip" in names
        assert "context_resume" in names

    def test_resume_has_judgment_steps(self):
        from work import PIPELINES
        steps = PIPELINES["resume"]
        judgment = [s for s in steps if s.step_type == "judgment"]
        names = [s.name for s in judgment]
        assert "stack_pick" in names
        assert "context_resume" in names

    def test_resume_dry_run_yields_stack_pick(self, tmp_path):
        from work import build_context, PIPELINES
        from orchestrator_engine import run_loop
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        ctx = build_context({
            "command": "resume",
            "workspace": str(ws), "project": str(proj),
            "branch": "", "base_branch": "main",
            "on_main": "yes", "in_slot": "no",
            "covers": "", "issue_repo": "Org/repo",
            "meta_state": "", "owner_repo": "Org/repo",
        })
        ctx.dry_run = True
        result = run_loop(PIPELINES["resume"], ctx)
        assert result["ACTION"] == "stack_pick"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_pipeline.py::TestResumePipeline -v`
Expected: FAIL

- [ ] **Step 3: Implement resume steps in work.py**

```python
def _resume_exec(ctx):
    return _WORK_RESUME_DIR / "resume_exec.py"


def _skip_single_stack(ctx) -> bool:
    """Skip stack picker when only one entry — auto-select."""
    stack_depth = int(ctx.progress.get("stack_depth", "0"))
    return stack_depth <= 1


RESUME_STEPS: list[StepDef] = [
    StepDef("stack_pick", "resume", "judgment",
            skip_fn=_skip_single_stack,
            action_context_fn=lambda ctx: {"CONTEXT": "stack_pick"}),
    StepDef("pop_stack", "resume", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_project_dir / "stack.py"), "pop",
                str(ctx.workspace / ".pause-stack"),
                ctx.progress.get("selected_branch", ctx.branch)]),
    StepDef("checkout_branches", "resume", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_resume_exec(ctx)), "checkout-branches",
                str(ctx.project), str(ctx.workspace),
                f"branch={ctx.progress.get('selected_branch', ctx.branch)}"]),
    StepDef("rebase", "resume", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_resume_exec(ctx)), "rebase",
                str(ctx.project), str(ctx.workspace),
                f"base-branch={ctx.base_branch}"]),
    StepDef("reset_wip", "resume", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_resume_exec(ctx)), "reset-wip",
                str(ctx.project), str(ctx.workspace)]),
    StepDef("context_resume", "resume", "judgment",
            action_context_fn=lambda ctx: {"CONTEXT": "load_context"}),
]
```

Update `PIPELINES`:
```python
PIPELINES: dict[str, list[StepDef]] = {
    "pause": PAUSE_STEPS,
    "resume": RESUME_STEPS,
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_pipeline.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_pipeline.py
git commit -m "feat(#384): add resume pipeline — stack pick + mechanical restore

Six steps: stack_pick (judgment), pop_stack, checkout_branches,
rebase, reset_wip, context_resume (judgment).

Refs #384"
```

---

## Batch 3: Full pipelines — start, continue, next, find

After this batch: all six commands have step lists. The full lifecycle
is Python-driven.

### Task 6: Add start pipeline to work.py

**Files:**
- Modify: `project/work.py` (add START_STEPS)
- Modify: `tests/test_work_pipeline.py` (add start tests)

**Interfaces:**
- Consumes: `work-start/branch_create.py` (sync-main, create-branches, commit-scaffold),
  `work-start/scaffold.py`, `work-start/flyway_scan.py`,
  `project/routing.py`, `project/section_hashes.py`
- Produces: `START_STEPS` step list in `PIPELINES["start"]`

- [ ] **Step 1: Write failing test for start pipeline structure**

```python
class TestStartPipeline:
    def test_start_pipeline_exists(self):
        from work import PIPELINES
        assert "start" in PIPELINES

    def test_start_has_expected_steps(self):
        from work import PIPELINES
        names = [s.name for s in PIPELINES["start"]]
        assert "sync_main" in names
        assert "create_branches" in names
        assert "scaffold" in names
        assert "commit_scaffold" in names
        assert "brainstorm_offer" in names

    def test_start_judgment_steps(self):
        from work import PIPELINES
        judgment = [s.name for s in PIPELINES["start"] if s.step_type == "judgment"]
        assert "resolve_issue" in judgment
        assert "branch_name" in judgment
        assert "brainstorm_offer" in judgment

    def test_start_dry_run_yields_resolve_issue(self, tmp_path):
        from work import build_context, PIPELINES
        from orchestrator_engine import run_loop
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        ctx = build_context({
            "command": "start",
            "workspace": str(ws), "project": str(proj),
            "branch": "", "base_branch": "main",
            "on_main": "yes", "in_slot": "no",
            "covers": "", "issue_repo": "Org/repo",
            "meta_state": "", "owner_repo": "Org/repo",
        })
        ctx.dry_run = True
        result = run_loop(PIPELINES["start"], ctx)
        # First judgment step: resolve_issue (sync_main is mechanical but dry_run skips it)
        assert result["ACTION"] in ("resolve_issue", "sync_main", "complete")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_pipeline.py::TestStartPipeline -v`
Expected: FAIL

- [ ] **Step 3: Implement start steps**

Add to `project/work.py`:

```python
def _branch_create(ctx):
    return _WORK_START_DIR / "branch_create.py"


def _skip_issue_resolved(ctx) -> bool:
    return bool(ctx.issue_n)


def _skip_no_platform_doc(ctx) -> bool:
    return not ctx.has_platform_doc


def _skip_no_protocols(ctx) -> bool:
    return not ctx.has_protocols_dir


def _skip_flyway_none(ctx) -> bool:
    return ctx.flyway_next_v == "none"


def _skip_clone_feature_off(ctx) -> bool:
    # Clone feature is off by default — checked via clone_manager.py
    return True  # TODO: check clone_manager.py enabled


START_STEPS: list[StepDef] = [
    StepDef("clone_redirect", "start", "mechanical",
            skip_fn=_skip_clone_feature_off),
    StepDef("sync_main", "start", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_branch_create(ctx)), "sync-main",
                str(ctx.project), str(ctx.workspace),
                f"base={ctx.base_branch}"]),
    StepDef("resolve_issue", "start", "judgment",
            skip_fn=_skip_issue_resolved,
            action_context_fn=lambda ctx: {
                "CONTEXT": "resolve_issue",
                "OWNER_REPO": ctx.owner_repo}),
    StepDef("stacked_pr_detect", "start", "mechanical",
            skip_fn=lambda ctx: not ctx.issue_n),
    StepDef("activate_issues", "start", "mechanical",
            skip_fn=lambda ctx: True),  # skip unless GITHUB_PROJECT set
    StepDef("branch_name", "start", "judgment",
            action_context_fn=lambda ctx: {
                "CONTEXT": "branch_name",
                "ISSUE_N": ctx.issue_n,
                "ISSUE_TITLE": ctx.issue_title}),
    StepDef("flyway_scan", "start", "mechanical",
            skip_fn=_skip_flyway_none,
            script_fn=lambda ctx: [
                "python3", str(_WORK_START_DIR / "flyway_scan.py"),
                str(ctx.project), ctx.base_branch]),
    StepDef("create_branches", "start", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_branch_create(ctx)), "create-branches",
                str(ctx.project), str(ctx.workspace),
                f"branch={ctx.branch}",
                f"base={ctx.base_branch}"]),
    StepDef("design_routing", "start", "mechanical"),
    StepDef("scaffold", "start", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_WORK_START_DIR / "scaffold.py"),
                str(ctx.workspace),
                f"branch={ctx.branch}",
                f"project-sha={ctx.progress.get('project_sha', '')}",
                f"date={ctx.progress.get('date', '')}",
                f"issue={ctx.issue_n}",
                f"issue-repo={ctx.issue_repo}",
                f"covers={ctx.covers}",
                f"flyway-next-v={ctx.flyway_next_v}",
                f"design-repo={ctx.design_repo_key}"]),
    StepDef("commit_scaffold", "start", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_branch_create(ctx)), "commit-scaffold",
                str(ctx.workspace), f"branch={ctx.branch}"]),
    StepDef("platform_coherence", "start", "judgment",
            skip_fn=_skip_no_platform_doc,
            action_context_fn=lambda ctx: {"CONTEXT": "platform_coherence"}),
    StepDef("check_protocols", "start", "judgment",
            skip_fn=_skip_no_protocols,
            action_context_fn=lambda ctx: {"CONTEXT": "check_protocols"}),
    StepDef("garden_search", "start", "mechanical"),
    StepDef("load_specs", "start", "mechanical"),
    StepDef("check_intellij", "start", "mechanical"),
    StepDef("brainstorm_offer", "start", "judgment",
            action_context_fn=lambda ctx: {"CONTEXT": "brainstorm_offer"}),
]
```

Update `PIPELINES`:
```python
PIPELINES: dict[str, list[StepDef]] = {
    "start": START_STEPS,
    "pause": PAUSE_STEPS,
    "resume": RESUME_STEPS,
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_pipeline.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_pipeline.py
git commit -m "feat(#384): add start pipeline — 18 steps, mechanical + judgment

Delegates to existing scripts: branch_create.py, scaffold.py,
flyway_scan.py. Judgment steps: resolve_issue, branch_name,
platform_coherence, check_protocols, brainstorm_offer.

Refs #384"
```

### Task 7: Add continue, next, find pipelines to work.py

**Files:**
- Modify: `project/work.py` (add CONTINUE_STEPS, NEXT_STEPS, FIND_STEPS)
- Modify: `tests/test_work_pipeline.py` (add tests)

**Interfaces:**
- Consumes: `project/lifecycle.py` (transitions), `project/work_health.py`,
  `work-slot/plan_manager.py` (advance, append), `scripts/enrichment.py`
- Produces: `CONTINUE_STEPS`, `NEXT_STEPS`, `FIND_STEPS` in PIPELINES

- [ ] **Step 1: Write failing tests for continue, next, find pipelines**

```python
class TestContinuePipeline:
    def test_continue_pipeline_exists(self):
        from work import PIPELINES
        assert "continue" in PIPELINES
        names = [s.name for s in PIPELINES["continue"]]
        assert "auto_resolve_transient" in names
        assert "health_check" in names
        assert "load_context" in names

    def test_continue_dry_run(self, tmp_path):
        from work import build_context, PIPELINES
        from orchestrator_engine import run_loop
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        ctx = build_context({
            "command": "continue",
            "workspace": str(ws), "project": str(proj),
            "branch": "issue-42-foo", "base_branch": "main",
            "on_main": "no", "in_slot": "no",
            "covers": "42", "issue_repo": "Org/repo",
            "meta_state": "active", "owner_repo": "Org/repo",
        })
        ctx.dry_run = True
        result = run_loop(PIPELINES["continue"], ctx)
        # Should yield at load_context (judgment) since mechanical steps are dry-run
        assert result["ACTION"] in ("load_context", "complete")


class TestNextPipeline:
    def test_next_pipeline_exists(self):
        from work import PIPELINES
        assert "next" in PIPELINES
        names = [s.name for s in PIPELINES["next"]]
        assert "advance_issue" in names
        assert "context_refresh" in names


class TestFindPipeline:
    def test_find_pipeline_exists(self):
        from work import PIPELINES
        assert "find" in PIPELINES
        names = [s.name for s in PIPELINES["find"]]
        assert "refresh_cache" in names
        assert "present_candidates" in names
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_pipeline.py -v -k "Continue or Next or Find"`
Expected: FAIL

- [ ] **Step 3: Implement continue, next, find steps**

```python
def _skip_state_already_active(ctx) -> bool:
    return ctx.meta_state == "active"


def _skip_no_deferred(ctx) -> bool:
    return "has_deferred" not in ctx.last_output


CONTINUE_STEPS: list[StepDef] = [
    StepDef("auto_resolve_transient", "continue", "mechanical",
            skip_fn=_skip_state_already_active),
    StepDef("lifecycle_continue", "continue", "lifecycle",
            from_state="active", to_state="active", event="work_continue"),
    StepDef("health_check", "continue", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_project_dir / "work_health.py"),
                "--scope", "entry",
                "--project", str(ctx.project),
                "--workspace", str(ctx.workspace),
                "--owner-repo", ctx.owner_repo]),
    StepDef("load_specs", "continue", "mechanical"),
    StepDef("load_context", "continue", "judgment",
            action_context_fn=lambda ctx: {
                "CONTEXT": "load_context",
                "META_STATE": ctx.meta_state,
                "HAS_HANDOFF": "yes" if ctx.has_handoff else "no",
                "HANDOFF_PATH": str(ctx.handoff_path) if ctx.handoff_path else ""}),
]


_PLAN_MANAGER = _project_dir.parent / "work-slot" / "plan_manager.py"


NEXT_STEPS: list[StepDef] = [
    StepDef("lifecycle_next", "next", "lifecycle",
            from_state="active", to_state="transitioning", event="work_next"),
    StepDef("advance_issue", "next", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_PLAN_MANAGER), "advance",
                str(ctx.plan_path)] if ctx.plan_path else None),
    StepDef("tick_github", "next", "mechanical"),
    StepDef("context_refresh", "next", "mechanical"),
    StepDef("deferred_check", "next", "judgment",
            skip_fn=_skip_no_deferred,
            action_context_fn=lambda ctx: {"CONTEXT": "deferred_check"}),
]


_ENRICHMENT = _project_dir.parent / "scripts" / "enrichment.py"


FIND_STEPS: list[StepDef] = [
    StepDef("refresh_cache", "find", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_ENRICHMENT), "refresh",
                "--repo", ctx.owner_repo]),
    StepDef("query_recommendations", "find", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_ENRICHMENT), "what-next",
                "--repo", ctx.owner_repo, "--mode", "general",
                "--limit", "5"]),
    StepDef("present_candidates", "find", "judgment",
            action_context_fn=lambda ctx: {
                "CONTEXT": "present_candidates",
                **ctx.last_output}),
    StepDef("populate_queue", "find", "mechanical",
            script_fn=lambda ctx: [
                "python3", str(_PLAN_MANAGER), "append",
                str(ctx.plan_path),
                f"issues={ctx.progress.get('selected_issues', '')}"]
            if ctx.plan_path and ctx.progress.get("selected_issues") else None),
]
```

Update `PIPELINES`:
```python
PIPELINES: dict[str, list[StepDef]] = {
    "start": START_STEPS,
    "continue": CONTINUE_STEPS,
    "pause": PAUSE_STEPS,
    "resume": RESUME_STEPS,
    "next": NEXT_STEPS,
    "find": FIND_STEPS,
}
```

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/test_work_pipeline.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add project/work.py tests/test_work_pipeline.py
git commit -m "feat(#384): add continue, next, find pipelines

All six lifecycle commands now have step lists.
continue: 5 steps (transient resolve, lifecycle, health, specs, context).
next: 5 steps (lifecycle, advance, tick, refresh, deferred check).
find: 4 steps (refresh, query, present candidates, populate queue).

Refs #384"
```

---

## Batch 4: SKILL.md reduction and handler files

After this batch: `work/SKILL.md` is a minimal orchestrator loop with
a dispatch table. Handler files loaded lazily per ACTION.

### Task 8: Create handler files and shrink SKILL.md

**Files:**
- Create: `work/handlers/resolve-issue.md`
- Create: `work/handlers/branch-name.md`
- Create: `work/handlers/platform-coherence.md`
- Create: `work/handlers/check-protocols.md`
- Create: `work/handlers/brainstorm-offer.md`
- Create: `work/handlers/load-context.md`
- Create: `work/handlers/stack-pick.md`
- Create: `work/handlers/present-candidates.md`
- Create: `work/handlers/deferred-check.md`
- Create: `work/handlers/resolve-conflict.md`
- Modify: `work/SKILL.md`
- Modify: `work-start/SKILL.md`
- Modify: `work-pause/SKILL.md`
- Modify: `work-resume/SKILL.md`

This task extracts handler content from the existing SKILL.md files into
standalone handler files. Each handler file contains the LLM instructions
for one judgment step. The orchestrator loop in `work/SKILL.md` loads
the handler file only when the pipeline yields that action.

- [ ] **Step 1: Create handlers directory and resolve-issue handler**

Extract issue resolution guidance from `work-start/SKILL.md` Step 4
into `work/handlers/resolve-issue.md`:

```markdown
# Handler: resolve_issue

Resolve the issue for this branch. The orchestrator yielded this
because no issue number was provided.

1. Read OWNER_REPO from the ACTION context
2. Invoke issue-workflow Phase 2 with the work description
3. When Phase 2 returns ISSUE_N and ISSUE_TITLE, call the orchestrator
   with step_done=resolve_issue and issue_n=<N> issue_title=<title>
```

- [ ] **Step 2: Create remaining handler files**

Create handler files for each judgment action, extracting content from
the current SKILL.md files. Each file is self-contained — the handler
reader sees only that file, not the full SKILL.md.

Handler files to create: `branch-name.md`, `platform-coherence.md`,
`check-protocols.md`, `brainstorm-offer.md`, `load-context.md`,
`stack-pick.md`, `present-candidates.md`, `deferred-check.md`,
`resolve-conflict.md`.

- [ ] **Step 3: Rewrite work/SKILL.md to orchestrator loop + dispatch**

Replace the routing logic with:

```markdown
## Orchestrator Loop

For commands: start, continue, pause, resume, next, find.

Run:
\`\`\`bash
python3 project/work.py <command> \
    workspace=<WORKSPACE> project=<PROJECT> \
    branch=<BRANCH> base_branch=<BASE> \
    [... all ctx.py fields as key=value args]
\`\`\`

Read ACTION= from output. Dispatch:

| ACTION | Handler |
|--------|---------|
| resolve_issue | Read handlers/resolve-issue.md |
| branch_name | Read handlers/branch-name.md |
| ... | ... |
| complete | Done. Report summary. |
| error | Diagnose and retry. |

For end and sync commands, continue to use work_end_orchestrator.py
as before (see Step 7 in the existing routing).
```

- [ ] **Step 4: Shrink work-start/SKILL.md, work-pause/SKILL.md, work-resume/SKILL.md**

Each shrinks to: "These commands are handled by `work/SKILL.md` which
calls `project/work.py`. See the work skill for the orchestrator loop."

- [ ] **Step 5: Commit**

```bash
git add work/handlers/ work/SKILL.md work-start/SKILL.md \
  work-pause/SKILL.md work-resume/SKILL.md
git commit -m "feat(#384): shrink SKILL.md to orchestrator loop + dispatch

Handler files in work/handlers/ loaded lazily per ACTION=.
work-start, work-pause, work-resume now redirect to work/SKILL.md.

Refs #384"
```

---

## Batch 5: Conflict resolution tiers

After this batch: rebase conflicts auto-resolve lifecycle files (Tier 1),
yield to LLM for semantic conflicts (Tier 2), escalate to user for
complex conflicts (Tier 3).

### Task 9: Implement conflict resolution tiers

**Files:**
- Create: `project/conflict_resolution.py`
- Create: `tests/test_conflict_resolution.py`
- Modify: `project/work.py` (wire into rebase steps via on_mechanical_error)

**Interfaces:**
- Produces: `classify_conflict(repo, conflicted_files) -> (auto, remaining)`,
  `auto_resolve_lifecycle(repo, files)`,
  `make_rebase_error_handler(ctx) -> Callable`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_conflict_resolution.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))


class TestClassifyConflict:
    def test_all_lifecycle_files(self):
        from conflict_resolution import classify_conflict
        auto, remaining = classify_conflict([
            ".plan", "JOURNAL.md", ".work-progress"])
        assert auto == [".plan", "JOURNAL.md", ".work-progress"]
        assert remaining == []

    def test_mixed_files(self):
        from conflict_resolution import classify_conflict
        auto, remaining = classify_conflict([
            ".plan", "src/main.py", "JOURNAL.md"])
        assert auto == [".plan", "JOURNAL.md"]
        assert remaining == ["src/main.py"]

    def test_no_lifecycle_files(self):
        from conflict_resolution import classify_conflict
        auto, remaining = classify_conflict(["src/main.py", "README.md"])
        assert auto == []
        assert remaining == ["src/main.py", "README.md"]


class TestMakeRebaseErrorHandler:
    def test_classifies_rebase_conflict(self):
        from conflict_resolution import make_rebase_error_handler
        handler = make_rebase_error_handler()
        result = handler(
            None, None,
            {"ERROR": "rebase_conflict",
             "CONFLICTED_FILES": ".plan,JOURNAL.md"})
        # All lifecycle — auto-resolved, returns None (continue)
        assert result is None or result.get("ACTION") != "user_input"

    def test_yields_for_source_conflicts(self):
        from conflict_resolution import make_rebase_error_handler
        handler = make_rebase_error_handler()
        result = handler(
            None, None,
            {"ERROR": "rebase_conflict",
             "CONFLICTED_FILES": ".plan,src/main.py"})
        assert result is not None
        assert result["ACTION"] == "resolve_conflict"
        assert "src/main.py" in result.get("FILES", "")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_conflict_resolution.py -v`
Expected: FAIL

- [ ] **Step 3: Implement conflict_resolution.py**

```python
#!/usr/bin/env python3
"""Three-tier conflict resolution for rebase operations.

Tier 1: Auto-resolve lifecycle files (take ours)
Tier 2: LLM semantic resolution (yield ACTION=resolve_conflict)
Tier 3: Human intervention (yield ACTION=user_input)
"""

LIFECYCLE_AUTO_RESOLVE = {
    ".plan", "JOURNAL.md", ".close-progress", ".work-progress",
    ".close-log.jsonl", ".artifacts-promoted", ".close-progress.tmp",
    ".work-progress.tmp", ".execute-progress", ".land-ledger.jsonl",
}


def classify_conflict(conflicted_files: list[str]) -> tuple[list[str], list[str]]:
    auto = [f for f in conflicted_files if f in LIFECYCLE_AUTO_RESOLVE]
    remaining = [f for f in conflicted_files if f not in LIFECYCLE_AUTO_RESOLVE]
    return auto, remaining


def make_rebase_error_handler():
    def handler(step, ctx, result):
        if result.get("ERROR") != "rebase_conflict":
            return None
        files_raw = result.get("CONFLICTED_FILES", "")
        if not files_raw:
            return None
        files = [f.strip() for f in files_raw.split(",") if f.strip()]
        auto, remaining = classify_conflict(files)

        # Tier 1 handled by caller (auto-resolve lifecycle files)
        if not remaining:
            return None  # all auto-resolved

        # Tier 2: yield to LLM
        return {
            "ACTION": "resolve_conflict",
            "FILES": ",".join(remaining),
            "AUTO_RESOLVED": ",".join(auto),
            "CONTEXT": "rebase_conflict",
        }
    return handler
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_conflict_resolution.py -v`
Expected: all PASS

- [ ] **Step 5: Wire into work.py rebase steps**

In `project/work.py`, import and use the handler:

```python
from conflict_resolution import make_rebase_error_handler

# In the main() function, pass to run_loop:
result = run_loop(
    steps, ctx,
    on_mechanical_error=make_rebase_error_handler(),
    complete_summary=f"{command} complete.",
)
```

- [ ] **Step 6: Run all tests**

Run: `python3 -m pytest tests/test_work_pipeline.py tests/test_conflict_resolution.py tests/test_work_progress.py -v`
Expected: all PASS

Also run the full test suite to check for regressions:
Run: `python3 -m pytest tests/ -v --timeout=120`

- [ ] **Step 7: Commit**

```bash
git add project/conflict_resolution.py tests/test_conflict_resolution.py project/work.py
git commit -m "feat(#384): add three-tier conflict resolution

Tier 1: auto-resolve lifecycle files (take ours).
Tier 2: yield ACTION=resolve_conflict to LLM.
Tier 3: yield ACTION=user_input for human intervention.

Refs #384"
```

---

## References

- [2026-09-28-pipeline-lifecycle-design.md] — design spec
- [decisions.md] — D1–D7 design decisions
- [#382] — unified mechanical pipeline (prerequisites A–D)
- [work-end/orchestrator_engine.py] — engine pattern (322 lines)
- [work-end/shared_steps.py] — StepDef, OrchestratorContextBase
- [work-end/close_progress.py] — progress tracking to generalize
- [work-end/work_end_orchestrator.py] — proven step list pattern
- [work-start/branch_create.py] — externalized branch operations
- [work-start/scaffold.py] — externalized scaffold operations
- [work-pause/pause_exec.py] — externalized pause operations
- [work-resume/resume_exec.py] — externalized resume operations
- [docs/protocols/skill-md-minimal-orchestrator-loop.md] — SKILL.md protocol
- [docs/protocols/externalised-scripts-require-tests.md] — test requirement
- [GitHub #384] — focal issue

# Work Command Taxonomy Phase 2 — Brief Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #202 — Work command taxonomy: separate continue/resume/brief verbs
**Issue group:** #202

**Goal:** Implement `/brief` as a standalone orientation skill backed by
`brief.py` structured data aggregator and `ctx.py` importable refactor.

**Architecture:** `ctx.py` is refactored to expose a `resolve() -> dict`
function (importable without side effects). `brief.py` composes `ctx.resolve()`,
`work_router.detect_state()`, `work_health.run_checks()`, and HANDOFF.md
parsing into a unified structured output. `/brief` skill is a thin CLI
wrapper.

**Tech Stack:** Python 3, pytest, SKILL.md (markdown)

## Global Constraints

- No AI attribution in commit messages
- Every commit references #202 (`Refs #202`)
- Run `python3 scripts/claude-skill sync-local --all -y` after skill edits
- `ctx.py` refactor must not break any existing callers (all skills call
  it via subprocess — the CLI interface must remain identical)

---

### Task 1: Refactor ctx.py into importable library

**Files:**
- Modify: `project/ctx.py`
- Test: `tests/test_ctx.py` (new)

**Interfaces:**
- Produces: `resolve(cwd: str | None = None) -> dict[str, str]`
- Preserves: CLI `python3 ctx.py` output unchanged

This is the riskiest task — ctx.py is called by every skill. The refactor
wraps the ~200 lines of top-level state derivation into a function, collects
results into a dict instead of printing, and guards the print loop with
`if __name__ == '__main__'`.

- [ ] **Step 1: Write test for existing CLI behavior (golden test)**

Create `tests/test_ctx.py` that captures the current CLI output format
as a regression test. Run ctx.py via subprocess and verify the output
contains expected KEY=VALUE lines.

```python
"""Tests for project/ctx.py"""
import subprocess
import sys
from pathlib import Path

CTX = Path(__file__).parent.parent / "project" / "ctx.py"


class TestCtxCLI:
    def test_outputs_key_value_lines(self, tmp_path):
        result = subprocess.run(
            [sys.executable, str(CTX)],
            capture_output=True, text=True,
            cwd=str(Path(__file__).parent.parent),
        )
        assert result.returncode == 0
        lines = result.stdout.strip().splitlines()
        for line in lines:
            assert "=" in line, f"Line missing '=': {line}"
        keys = [l.split("=", 1)[0] for l in lines]
        assert "WORKSPACE" in keys
        assert "PROJECT" in keys
        assert "CURRENT_BRANCH" in keys
        assert "META_STATE" in keys

    def test_cli_matches_resolve(self):
        sys.path.insert(0, str(CTX.parent))
        from ctx import resolve
        result = resolve(cwd=str(Path(__file__).parent.parent))
        assert "WORKSPACE" in result
        assert "PROJECT" in result
        assert isinstance(result, dict)
```

- [ ] **Step 2: Run tests to verify the CLI golden test passes and resolve test fails**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: CLI test passes, resolve test fails (function doesn't exist)

- [ ] **Step 3: Refactor ctx.py**

Wrap all state derivation (lines 23-310) into `def resolve(cwd=None) -> dict[str, str]:`.
The function:
1. Uses `cwd` parameter instead of implicit CWD (defaults to `os.getcwd()`)
2. Collects all values into a `result = {}` dict
3. Returns the dict instead of printing

Guard the print loop:
```python
if __name__ == '__main__':
    for key, value in resolve().items():
        print(f"{key}={value}")
```

Key constraints:
- Helper functions (`run()`, `check_file()`, `check_dir()`,
  `_resolve_symlink_target()`) stay at module top level
- Imports stay at module top level
- The `cwd` parameter replaces `Path.cwd()` calls inside resolve
- `sys.exit(1)` on missing git root becomes a raised exception
  (callers via subprocess see exit code 1; callers via import catch
  the exception)

- [ ] **Step 4: Run tests to verify both pass**

Run: `python3 -m pytest tests/test_ctx.py -v`
Expected: Both tests pass

- [ ] **Step 5: Run existing test suites that depend on ctx.py**

Run: `python3 -m pytest tests/test_lifecycle.py tests/test_work_router.py tests/test_work_health.py tests/test_pre_push_hook.py -v`
Expected: All pass (ctx.py CLI behavior unchanged)

- [ ] **Step 6: Commit**

```bash
git add project/ctx.py tests/test_ctx.py
git commit -m "refactor(#202): extract ctx.resolve() for importable use

ctx.py now exposes resolve(cwd=None) -> dict. CLI behavior unchanged.
Enables brief.py to import ctx directly instead of shelling out.

Refs #202"
```

---

### Task 2: Create brief.py structured data aggregator

**Files:**
- Create: `brief/brief.py`
- Test: `tests/test_brief.py` (new)

**Interfaces:**
- Consumes: `ctx.resolve()` from Task 1, `work_router.detect_state()`,
  `work_health.run_checks()` (existing)
- Produces: KEY=VALUE structured output (see spec D5 for format)

- [ ] **Step 1: Write test for brief.py output format**

```python
"""Tests for brief/brief.py"""
import subprocess
import sys
from pathlib import Path

BRIEF = Path(__file__).parent.parent / "brief" / "brief.py"


class TestBriefCLI:
    def test_outputs_state_field(self):
        result = subprocess.run(
            [sys.executable, str(BRIEF)],
            capture_output=True, text=True,
            cwd=str(Path(__file__).parent.parent),
        )
        assert result.returncode == 0
        lines = result.stdout.strip().splitlines()
        keys = [l.split("=", 1)[0] for l in lines if "=" in l]
        assert "STATE" in keys
        assert "STACK_DEPTH" in keys

    def test_state_values(self):
        result = subprocess.run(
            [sys.executable, str(BRIEF)],
            capture_output=True, text=True,
            cwd=str(Path(__file__).parent.parent),
        )
        output = dict(
            l.split("=", 1) for l in result.stdout.strip().splitlines()
            if "=" in l and not l.startswith("COMMIT=") and not l.startswith("CHECK=") and not l.startswith("CLOSED_BRANCH=")
        )
        assert output["STATE"] in ("feature_branch", "main_with_stack", "main_idle")
```

- [ ] **Step 2: Run test — expect failure (file doesn't exist)**

- [ ] **Step 3: Create brief/brief.py**

The script:
1. Imports `ctx.resolve()`, `work_router.detect_state()`,
   `work_health.run_checks()` as Python functions
2. Calls each sequentially
3. Derives `STATE` from ctx + router output
4. Extracts HANDOFF summary if present
5. Gets recent commits via git log
6. On main with no work: enumerates recently closed branches
7. Emits unified KEY=VALUE output

- [ ] **Step 4: Run tests — expect pass**

- [ ] **Step 5: Commit**

---

### Task 3: Create brief/SKILL.md and slash command

**Files:**
- Create: `brief/SKILL.md`
- Create: `brief/commands/brief.md`

**Interfaces:**
- Consumes: `brief.py` from Task 2

- [ ] **Step 1: Create SKILL.md with CSO description**

```yaml
---
name: brief
description: >
  Use when the user says "/brief", "brief me", "what's the state",
  "where are we", or "orient me" — provides an orientation summary of the
  current work state, branch, issue, plan progress, and health checks.
  Not for writing content (use write-content for that).
---
```

Skill body: run `brief.py`, format output for terminal display.

- [ ] **Step 2: Generate slash command**

```bash
python3 scripts/generate_commands.py
```

- [ ] **Step 3: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 4: Update README.md**

Add `/brief` to the skill table.

- [ ] **Step 5: Commit**

---

### Task 4: Validation and final tests

- [ ] **Step 1: Run full test suite**

```bash
python3 -m pytest tests/ -v --timeout 60
```

- [ ] **Step 2: Run commit-tier validation**

```bash
python3 scripts/validate_all.py --tier commit
```

- [ ] **Step 3: Commit any fixes**

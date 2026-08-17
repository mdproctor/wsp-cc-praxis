# Issue Transition Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #158 — Worklog: track individual issue transitions within branches
**Issue group:** #158

**Goal:** Wire existing `record_issue_activate()` and `record_issue_complete()` worklog functions into the three call sites where issue transitions occur.

**Architecture:** Add a shared `_emit_issue_events()` helper to `plan_manager.py` that imports worklog via `sys.path` (same pattern as `lifecycle.py`). Call it from `advance()` (work-next), `complete_active_issue()` (work-end), and add `record_issue_activate()` to `scaffold.py` (work-start).

**Tech Stack:** Python 3, SQLite (worklog.db), pytest

## Global Constraints

- Worklog errors are advisory — never gate on them. Use try/except with `WARN=worklog_error`.
- Import worklog via `sys.path.insert(0, scripts_dir)` — same pattern as `lifecycle.py:_emit_to_worklog()`.
- Worklog recording happens AFTER file writes in `advance()`.
- All new Python code needs tests in the same commit (protocol: `externalised-scripts-require-tests`).

---

### Task 1: Add `_read_meta_fields()` and `_emit_issue_events()` helpers to plan_manager.py

**Files:**
- Modify: `work-slot/plan_manager.py`
- Modify: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `worklog.record_issue_complete()`, `worklog.record_issue_activate()`, `worklog.connect()` from `scripts/worklog.py`
- Produces: `_read_meta_fields(meta_path: Path) -> dict[str, str]`, `_emit_issue_events(meta_path: Path, repo_path: str, completed: int, next_issue: int | None) -> None`

- [ ] **Step 1: Write failing tests for `_read_meta_fields`**

```python
class TestReadMetaFields:
    def test_reads_branch_and_issue_repo(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-fix\nstate: active\nissue: 42\nissue-repo: Hortora/soredium\n")
        fields = plan_manager._read_meta_fields(meta)
        assert fields["branch"] == "issue-42-fix"
        assert fields["issue-repo"] == "Hortora/soredium"

    def test_handles_empty_values(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: test\nissue:\nissue-repo:\n")
        fields = plan_manager._read_meta_fields(meta)
        assert fields["branch"] == "test"
        assert fields["issue"] == ""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestReadMetaFields -v`
Expected: FAIL with `AttributeError: module 'plan_manager' has no attribute '_read_meta_fields'`

- [ ] **Step 3: Implement `_read_meta_fields`**

Add to `work-slot/plan_manager.py` after the existing imports:

```python
def _read_meta_fields(meta_path: Path) -> dict[str, str]:
    fields: dict[str, str] = {}
    for line in meta_path.read_text().splitlines():
        if ':' in line:
            k, _, v = line.partition(':')
            fields[k.strip()] = v.strip()
    return fields
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestReadMetaFields -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for `_emit_issue_events`**

```python
class TestEmitIssueEvents:
    def _setup_meta(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-42-fix\nissue: 42\nissue-repo: Hortora/soredium\ncovers: 42\n")
        return meta

    def test_emits_complete_and_activate(self, tmp_path):
        meta = self._setup_meta(tmp_path)
        with unittest.mock.patch.dict('sys.modules', {'worklog': unittest.mock.MagicMock()}) as modules:
            mock_wl = modules['worklog']
            mock_conn = unittest.mock.MagicMock()
            mock_wl.connect.return_value = mock_conn
            plan_manager._emit_issue_events(meta, "/repo", 42, 43)
            mock_wl.record_issue_complete.assert_called_once_with(
                mock_conn, "issue-42-fix", "/repo", 42, "Hortora/soredium")
            mock_wl.record_issue_activate.assert_called_once_with(
                mock_conn, "issue-42-fix", "/repo", 43, "Hortora/soredium")

    def test_emits_only_complete_when_no_next(self, tmp_path):
        meta = self._setup_meta(tmp_path)
        with unittest.mock.patch.dict('sys.modules', {'worklog': unittest.mock.MagicMock()}) as modules:
            mock_wl = modules['worklog']
            mock_conn = unittest.mock.MagicMock()
            mock_wl.connect.return_value = mock_conn
            plan_manager._emit_issue_events(meta, "/repo", 42, None)
            mock_wl.record_issue_complete.assert_called_once()
            mock_wl.record_issue_activate.assert_not_called()

    def test_swallows_errors(self, tmp_path):
        meta = self._setup_meta(tmp_path)
        with unittest.mock.patch.dict('sys.modules', {'worklog': unittest.mock.MagicMock()}) as modules:
            mock_wl = modules['worklog']
            mock_wl.connect.side_effect = Exception("db locked")
            plan_manager._emit_issue_events(meta, "/repo", 42, 43)
            # Should not raise — error is swallowed
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestEmitIssueEvents -v`
Expected: FAIL with `AttributeError: module 'plan_manager' has no attribute '_emit_issue_events'`

- [ ] **Step 7: Implement `_emit_issue_events`**

Add to `work-slot/plan_manager.py` after `_read_meta_fields`:

```python
import sys as _sys

def _emit_issue_events(meta_path: Path, repo_path: str,
                       completed: int, next_issue: int | None) -> None:
    try:
        _scripts_dir = str(Path(__file__).resolve().parent.parent / "scripts")
        if _scripts_dir not in _sys.path:
            _sys.path.insert(0, _scripts_dir)
        import worklog

        fields = _read_meta_fields(meta_path)
        branch = fields.get("branch", "")
        issue_repo = fields.get("issue-repo", "")
        if not branch:
            return

        conn = worklog.connect()
        worklog.record_issue_complete(conn, branch, repo_path, completed, issue_repo)
        if next_issue is not None:
            worklog.record_issue_activate(conn, branch, repo_path, next_issue, issue_repo)
        conn.close()
    except Exception as e:
        print(f"WARN=worklog_error detail={e}")
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestEmitIssueEvents -v`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#158): add _emit_issue_events helper to plan_manager"
```

---

### Task 2: Wire `advance()` and `advance_issue()` to emit worklog events

**Files:**
- Modify: `work-slot/plan_manager.py`
- Modify: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `_emit_issue_events()` from Task 1
- Produces: `advance(plan_path, meta_path, repo_path=None)` — updated signature; `advance_issue(plan_path, epic_path, meta_path, repo_path=None)` — updated signature

- [ ] **Step 1: Write failing test for advance with worklog emission**

```python
class TestAdvanceWorklog:
    def _setup(self, tmp_path, plan_content, covers="42"):
        design = tmp_path / "design"
        design.mkdir(exist_ok=True)
        plan_file = design / ".plan"
        plan_file.write_text(plan_content)
        meta = design / ".meta"
        meta.write_text(f"branch: test-branch\nissue: 42\nissue-repo: Org/repo\ncovers: {covers}\n")
        return plan_file, meta

    def test_advance_emits_events_when_repo_path_provided(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.advance(plan_file, meta, repo_path="/project")
            mock_emit.assert_called_once_with(meta, "/project", 42, 43)

    def test_advance_skips_worklog_when_no_repo_path(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.advance(plan_file, meta)
            mock_emit.assert_not_called()

    def test_advance_emits_none_next_on_queue_exhausted(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [x] #42 — A\n- [ ] #43 — B ← active\n\n## Session State\nCurrent: #43 — B\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.advance(plan_file, meta, repo_path="/project")
            mock_emit.assert_called_once_with(meta, "/project", 43, None)

    def test_advance_worklog_error_does_not_break_advance(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events', side_effect=Exception("boom")):
            result = plan_manager.advance(plan_file, meta, repo_path="/project")
            assert result.completed == 42
            assert result.next_issue == 43
            tree = plan_manager.parse_plan(plan_file)
            assert tree.queue[0].completed is True
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAdvanceWorklog -v`
Expected: FAIL — `_emit_issue_events` is never called (signature doesn't accept `repo_path` yet)

- [ ] **Step 3: Add `repo_path` parameter to `advance()` and wire emission**

In `work-slot/plan_manager.py`, modify the `advance` function signature and add the emission call after `rewrite_plan`:

Change signature from:
```python
def advance(plan_path: Path, meta_path: Path) -> AdvanceResult:
```
to:
```python
def advance(plan_path: Path, meta_path: Path,
            repo_path: str | None = None) -> AdvanceResult:
```

Add after `rewrite_plan(plan_path, tree)` (line 329), before the `return`:
```python
    if repo_path:
        try:
            _emit_issue_events(
                meta_path, repo_path,
                completed_leaf.issue_number,
                next_leaf.issue_number if next_leaf else None,
            )
        except Exception as e:
            print(f"WARN=worklog_error detail={e}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAdvanceWorklog -v`
Expected: PASS

- [ ] **Step 5: Add `repo_path` to `advance_issue()` dispatch**

Change signature from:
```python
def advance_issue(plan_path: Path | None, epic_path: Path | None,
                  meta_path: Path) -> AdvanceResult:
```
to:
```python
def advance_issue(plan_path: Path | None, epic_path: Path | None,
                  meta_path: Path, repo_path: str | None = None) -> AdvanceResult:
```

Change the `.plan` dispatch call from:
```python
        return advance(plan_path, meta_path)
```
to:
```python
        return advance(plan_path, meta_path, repo_path=repo_path)
```

The epic_manager fallback path is unchanged — no `repo_path` propagation (deprecated path).

- [ ] **Step 6: Write test for advance_issue dispatch**

```python
class TestAdvanceIssueWorklog:
    def test_passes_repo_path_to_plan_advance(self, tmp_path):
        design = tmp_path / "design"
        design.mkdir()
        plan_file = design / ".plan"
        plan_file.write_text("# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n")
        meta = design / ".meta"
        meta.write_text("branch: test\nissue: 42\nissue-repo: Org/repo\ncovers: 42\n")
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.advance_issue(plan_file, None, meta, repo_path="/project")
            mock_emit.assert_called_once_with(meta, "/project", 42, 43)
```

- [ ] **Step 7: Run all advance tests**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAdvanceWorklog tests/test_plan_manager.py::TestAdvanceIssueWorklog tests/test_plan_manager.py::TestAdvance -v`
Expected: all PASS (existing tests still pass without `repo_path`)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#158): wire advance() and advance_issue() to emit worklog events"
```

---

### Task 3: Add `complete_active_issue()` for work-end

**Files:**
- Modify: `work-slot/plan_manager.py`
- Modify: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `_emit_issue_events()`, `parse_plan()`, `_find_active_leaf()` from plan_manager
- Produces: `complete_active_issue(plan_path: Path, meta_path: Path, repo_path: str) -> int | None`

- [ ] **Step 1: Write failing tests**

```python
class TestCompleteActiveIssue:
    def _setup(self, tmp_path, plan_content):
        design = tmp_path / "design"
        design.mkdir(exist_ok=True)
        plan_file = design / ".plan"
        plan_file.write_text(plan_content)
        meta = design / ".meta"
        meta.write_text("branch: test-branch\nissue: 42\nissue-repo: Org/repo\ncovers: 42\n")
        return plan_file, meta

    def test_emits_complete_for_active_issue(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.complete_active_issue(plan_file, meta, "/project")
            assert result == 42
            mock_emit.assert_called_once_with(meta, "/project", 42, None)

    def test_returns_none_when_no_active_issue(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [x] #42 — A\n- [x] #43 — B\n\n## Session State\nCurrent: none\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events') as mock_emit:
            result = plan_manager.complete_active_issue(plan_file, meta, "/project")
            assert result is None
            mock_emit.assert_not_called()

    def test_does_not_modify_plan_file(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nCurrent: #42 — A\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        original_content = plan_file.read_text()
        with unittest.mock.patch.object(plan_manager, '_emit_issue_events'):
            plan_manager.complete_active_issue(plan_file, meta, "/project")
        assert plan_file.read_text() == original_content
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestCompleteActiveIssue -v`
Expected: FAIL with `AttributeError: module 'plan_manager' has no attribute 'complete_active_issue'`

- [ ] **Step 3: Implement `complete_active_issue`**

Add to `work-slot/plan_manager.py` after the `advance_issue` function:

```python
def complete_active_issue(plan_path: Path, meta_path: Path,
                          repo_path: str) -> int | None:
    tree = parse_plan(plan_path)
    active = _find_active_leaf(tree.queue)
    if not active:
        return None
    _emit_issue_events(meta_path, repo_path, active.issue_number, next_issue=None)
    return active.issue_number
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestCompleteActiveIssue -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#158): add complete_active_issue() for work-end implicit completion"
```

---

### Task 4: Wire `scaffold.py` to emit `issue-activate` at work-start

**Files:**
- Modify: `work-start/scaffold.py`
- Modify: `tests/test_scaffold.py`

**Interfaces:**
- Consumes: `worklog.record_issue_activate()` from `scripts/worklog.py`
- Produces: scaffold emits `issue-activate` event after `work-start` event

- [ ] **Step 1: Write failing tests**

Existing scaffold tests use subprocess (can't mock `_wl`). Test worklog integration
via a real in-memory DB using `WORKLOG_DB` env var, matching the subprocess pattern:

```python
class TestScaffoldIssueActivate:
    def test_emits_issue_activate_after_work_start(self, tmp_path):
        ws = tmp_path / "workspace"
        ws.mkdir()
        db_path = str(tmp_path / "test-worklog.db")
        env = {**os.environ, "WORKLOG_DB": db_path}
        result = subprocess.run(
            [sys.executable, str(SCRIPT), str(ws),
             "branch=test-branch", "project-sha=abc123",
             "date=2026-08-04", "issue=42",
             "issue-repo=Org/repo", "covers=42"],
            capture_output=True, text=True, env=env,
        )
        assert result.returncode == 0
        # Verify worklog DB has the issue-activate event
        sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
        import worklog
        conn = worklog.connect(db_path)
        events = worklog.event_log(conn, event_type="issue-activate")
        assert len(events) == 1
        meta = json.loads(events[0]["metadata"])
        assert meta["issue_number"] == 42
        assert meta["issue_repo"] == "Org/repo"
        conn.close()

    def test_skips_issue_activate_when_no_issue(self, tmp_path):
        ws = tmp_path / "workspace"
        ws.mkdir()
        db_path = str(tmp_path / "test-worklog.db")
        env = {**os.environ, "WORKLOG_DB": db_path}
        result = subprocess.run(
            [sys.executable, str(SCRIPT), str(ws),
             "branch=test-branch", "project-sha=abc123",
             "date=2026-08-04"],
            capture_output=True, text=True, env=env,
        )
        assert result.returncode == 0
        sys.path.insert(0, str(Path(__file__).parent.parent / "scripts"))
        import worklog
        conn = worklog.connect(db_path)
        events = worklog.event_log(conn, event_type="issue-activate")
        assert len(events) == 0
        conn.close()

    def test_scaffold_succeeds_without_worklog(self, tmp_path):
        ws = tmp_path / "workspace"
        ws.mkdir()
        # No WORKLOG_DB env — _wl import uses default path which may not exist
        # but scaffold should still succeed (try/except catches errors)
        result = run(ws, *required_args(issue="42", **{"issue-repo": "Org/repo"}))
        assert result.returncode == 0
        out = parse(result)
        assert out["CREATED"] == "yes"
```

Add `import json, os` to the test file imports if not already present.

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_scaffold.py::TestScaffoldIssueActivate -v`
Expected: FAIL — no `issue-activate` events in the DB (scaffold doesn't call `record_issue_activate` yet)

- [ ] **Step 4: Add `record_issue_activate` call to scaffold.py**

In `work-start/scaffold.py`, modify the worklog block (starting at line 133). Change from:

```python
    if _wl:
        try:
            _conn = _wl.connect()
            _wl.record_work_start(
                _conn, branch, str(workspace),
                issue_number=int(params.get("issue", "0") or "0"),
                issue_repo=params.get("issue-repo", ""),
                covers=params.get("covers", ""),
            )
            _conn.close()
        except Exception:
            pass
```

to:

```python
    if _wl:
        try:
            import os as _os
            _db_path = _os.environ.get("WORKLOG_DB")
            _conn = _wl.connect(_db_path) if _db_path else _wl.connect()
            issue_num = int(params.get("issue", "0") or "0")
            issue_repo = params.get("issue-repo", "")
            _wl.record_work_start(
                _conn, branch, str(workspace),
                issue_number=issue_num,
                issue_repo=issue_repo,
                covers=params.get("covers", ""),
            )
            if issue_num > 0:
                _wl.record_issue_activate(
                    _conn, branch, str(workspace),
                    issue_number=issue_num,
                    issue_repo=issue_repo,
                )
            _conn.close()
        except Exception:
            pass
```

Note: also adds `WORKLOG_DB` env var support (matching lifecycle.py's pattern)
for testability via subprocess.

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_scaffold.py -v`
Expected: all PASS (new and existing)

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/test_plan_manager.py tests/test_scaffold.py tests/test_worklog.py tests/test_lifecycle.py -v`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-start/scaffold.py tests/test_scaffold.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#158): wire scaffold.py to emit issue-activate at work-start"
```

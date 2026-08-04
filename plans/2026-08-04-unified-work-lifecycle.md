# Unified Work Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #180 — Unified work lifecycle
**Issue group:** #180

**Goal:** Replace fragmented work lifecycle routes with a unified model:
`.plan` file as universal issue queue, auto-epic detection, worklog
issue-level events for trellis, dynamic slot repos, and collapsed
Phase A/B into single `work-end`.

**Architecture:** New `plan_manager.py` module handles `.plan` tree
parsing, writing, flattening, and advancing. It sits alongside
`epic_manager.py` with a dispatch function for backward compat.
Existing skills are updated to read `.plan` instead of `.epic`.
worklog.py gains two new event types. ctx.py gains `PLAN_*` output
variables. Trellis integration is a separate plan (different repo).

**Tech Stack:** Python 3.12, pytest, existing soredium infrastructure

## Global Constraints

- All existing work-start, work-end, slot, pause/resume steps preserved (spec §Scope Principle)
- `.plan` always created, even for single issues (spec §2.8)
- `.slot` stays as slot identity file; `.plan` alongside it (spec §7.4)
- New archives to `slots/attic/` only, never `worktrees/attic/` (spec §Attic Directory Policy)
- `work-slot merge` removed — `work-end` handles full close (spec §11.3)
- `work_epic` and `slot_epic` lifecycle events removed (spec §5.1)
- Backward compat: `.epic` without `.plan` still works (spec §17.1)
- No AI attribution in commits

## Scope Note — Trellis Integration

Phase 8 (trellis sidecar: `WorkspaceScanner.scanPlanFile()`, `WorklogReader`,
`WorkPlan`/`QueueItem` data model, `LifecycleManager` updates) is a separate
plan in the trellis repo. This plan covers soredium Phases 1-7 only.

---

### Task 1: `.plan` Tree Parser and Writer (`plan_manager.py`)

**Files:**
- Create: `work-slot/plan_manager.py`
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Produces: `parse_plan(Path) -> PlanTree`, `flatten_leaves(PlanTree) -> list[LeafItem]`,
  `write_plan(Path, PlanTree)`, `rewrite_plan(Path, PlanTree)`,
  `build_plan_content(branch, items, date) -> str`,
  `append_to_queue(Path, list[QueueItem])`,
  `detect(workspace_path) -> dict | None`

**Data structures:**

```python
@dataclass
class QueueItem:
    issue_number: int
    title: str
    completed: bool = False
    active: bool = False
    is_epic: bool = False
    children: list['QueueItem'] = field(default_factory=list)
    batch: str | None = None  # "Batch 1 — Data model"

@dataclass
class PlanTree:
    heading: str        # "Work Plan — issue-42-batch-work"
    queue: list[QueueItem]
    current_issue: int | None
    started: str        # ISO date
    last_wrap: str | None

@dataclass
class LeafItem:
    issue_number: int
    title: str
    completed: bool
    active: bool
    parent_epic: int | None
    batch: str | None

@dataclass
class AdvanceResult:
    completed: int            # issue just finished
    next_issue: int | None    # next active issue (None = queue exhausted)
    next_title: str | None
    batch_complete: bool
    epic_complete: bool       # entire queue done
    safe_exit: bool           # at a batch boundary
```

- [ ] **Step 1: Write parser tests — single issue**

```python
# tests/test_plan_manager.py
SINGLE_ISSUE_PLAN = """\
# Work Plan — issue-42-fix-login

## Queue
- [ ] #42 — Fix login validation ← active

## Session State
Current: #42 — Fix login validation
Started: 2026-08-04
"""

class TestParsePlan:
    def test_single_issue(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(SINGLE_ISSUE_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        assert len(tree.queue) == 1
        assert tree.queue[0].issue_number == 42
        assert tree.queue[0].active is True
        assert tree.queue[0].is_epic is False
        assert tree.current_issue == 42
```

- [ ] **Step 2: Write parser tests — multi-issue with epic**

```python
MULTI_ISSUE_PLAN = """\
# Work Plan — issue-42-batch-work

## Queue
- [x] #42 — Fix login validation
- [ ] #50 — Weighted profiles (epic)
  - [x] #108 — Add weight field
  - [ ] #109 — Update scoring ← active
  - [ ] #110 — Migration script
- [ ] #32 — Update API docs

## Session State
Current: #109 — Update scoring
Started: 2026-08-04
"""

    def test_multi_issue_with_epic(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(MULTI_ISSUE_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        assert len(tree.queue) == 3  # top-level items
        assert tree.queue[0].completed is True
        assert tree.queue[1].is_epic is True
        assert len(tree.queue[1].children) == 3
        assert tree.queue[1].children[1].active is True
        assert tree.current_issue == 109
```

- [ ] **Step 3: Write parser tests — nested epics**

```python
NESTED_EPIC_PLAN = """\
# Work Plan — issue-42-nested

## Queue
- [ ] #42 — Fix login ← active
- [ ] #50 — Weighted profiles (epic)
  - [ ] #51 — Add weight field
  - [ ] #52 — Scoring subsystem (epic)
    - [ ] #60 — Score calculator
    - [ ] #61 — Score migration
  - [ ] #53 — API endpoints
- [ ] #32 — Update API docs

## Session State
Current: #42 — Fix login
Started: 2026-08-04
"""

    def test_nested_epics(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(NESTED_EPIC_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        epic50 = tree.queue[1]
        assert epic50.children[1].is_epic is True
        assert len(epic50.children[1].children) == 2
        assert epic50.children[1].children[0].issue_number == 60
```

- [ ] **Step 4: Write parser tests — batch planning within epic**

```python
BATCH_PLAN = """\
# Work Plan — issue-50-weighted

## Queue
- [ ] #50 — Weighted profiles (epic)
  ### Batch 1 — Data model ← current
  - [ ] #108 — Add weight field ← active
  - [ ] #109 — Migration script
  ### Batch 2 — Scoring logic
  - [ ] #110 — Update scoring algorithm
  - [ ] #111 — Recalculate existing scores

## Session State
Current: #108 — Add weight field
Started: 2026-08-04
"""

    def test_batch_planning(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(BATCH_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        epic = tree.queue[0]
        assert len(epic.children) == 4
        assert epic.children[0].batch == "Batch 1 — Data model"
        assert epic.children[2].batch == "Batch 2 — Scoring logic"
```

- [ ] **Step 5: Write parser tests — empty queue (free text mode)**

```python
EMPTY_PLAN = """\
# Work Plan — improve-scoring-engine

## Queue
(empty — issues created during design)

## Session State
Started: 2026-08-04
"""

    def test_empty_queue(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(EMPTY_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        assert len(tree.queue) == 0
        assert tree.current_issue is None
```

- [ ] **Step 6: Write flatten_leaves tests**

```python
class TestFlattenLeaves:
    def test_flat_list(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(NESTED_EPIC_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        leaves = plan_manager.flatten_leaves(tree)
        assert [l.issue_number for l in leaves] == [42, 51, 60, 61, 53, 32]

    def test_single_issue(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(SINGLE_ISSUE_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        leaves = plan_manager.flatten_leaves(tree)
        assert len(leaves) == 1
        assert leaves[0].issue_number == 42

    def test_empty_queue(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(EMPTY_PLAN)
        tree = plan_manager.parse_plan(plan_file)
        leaves = plan_manager.flatten_leaves(tree)
        assert leaves == []
```

- [ ] **Step 7: Write round-trip test (parse → write → parse)**

```python
class TestWritePlan:
    @pytest.mark.parametrize("content", [
        SINGLE_ISSUE_PLAN, MULTI_ISSUE_PLAN, NESTED_EPIC_PLAN, BATCH_PLAN, EMPTY_PLAN
    ])
    def test_round_trip(self, tmp_path, content):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(content)
        tree = plan_manager.parse_plan(plan_file)
        plan_manager.rewrite_plan(plan_file, tree)
        tree2 = plan_manager.parse_plan(plan_file)
        assert plan_manager.flatten_leaves(tree) == plan_manager.flatten_leaves(tree2)
```

- [ ] **Step 8: Write build_plan_content test**

```python
class TestBuildPlanContent:
    def test_builds_single_issue(self):
        items = [QueueItem(42, "Fix login", active=True)]
        content = plan_manager.build_plan_content("issue-42-fix-login", items, "2026-08-04")
        assert "# Work Plan — issue-42-fix-login" in content
        assert "- [ ] #42 — Fix login ← active" in content

    def test_builds_epic_with_children(self):
        children = [QueueItem(108, "Add weight"), QueueItem(109, "Update scoring")]
        items = [QueueItem(50, "Weighted profiles", is_epic=True, children=children)]
        content = plan_manager.build_plan_content("issue-50-weighted", items, "2026-08-04")
        assert "(epic)" in content
        assert "  - [ ] #108" in content  # indented
```

- [ ] **Step 9: Write append_to_queue test**

```python
class TestAppendToQueue:
    def test_appends_to_existing(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(SINGLE_ISSUE_PLAN)
        plan_manager.append_to_queue(plan_file, [QueueItem(99, "New issue")])
        tree = plan_manager.parse_plan(plan_file)
        assert len(tree.queue) == 2
        assert tree.queue[1].issue_number == 99

    def test_appends_to_empty(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(EMPTY_PLAN)
        plan_manager.append_to_queue(plan_file, [QueueItem(42, "First issue", active=True)])
        tree = plan_manager.parse_plan(plan_file)
        assert len(tree.queue) == 1
        assert tree.queue[0].active is True
```

- [ ] **Step 10: Write detect() test**

```python
class TestDetect:
    def test_detects_plan(self, tmp_path):
        design = tmp_path / "design"
        design.mkdir()
        (design / ".plan").write_text(SINGLE_ISSUE_PLAN)
        result = plan_manager.detect(tmp_path)
        assert result is not None
        assert result["has_plan"] is True
        assert result["active_issue"] == 42
        assert result["plan_path"] == str(design / ".plan")

    def test_no_plan(self, tmp_path):
        result = plan_manager.detect(tmp_path)
        assert result is None
```

- [ ] **Step 11: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: ImportError — `plan_manager` module doesn't exist

- [ ] **Step 12: Implement plan_manager.py**

Implement all functions: `parse_plan`, `flatten_leaves`, `write_plan`,
`rewrite_plan`, `build_plan_content`, `append_to_queue`, `detect`.

Key implementation detail for the parser: indentation-based nesting.
Each `- [ ]` or `- [x]` line's indentation level (in 2-space increments)
determines its depth in the tree. `### Batch N` headers within an epic's
indentation scope set the `batch` field on subsequent items.

- [ ] **Step 13: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: All PASS

- [ ] **Step 14: Commit**

```bash
git add work-slot/plan_manager.py tests/test_plan_manager.py
git commit -m "feat(#180): plan_manager — .plan tree parser, writer, flatten, detect

Refs #180"
```

---

### Task 2: `.plan` Advance Logic

**Files:**
- Modify: `work-slot/plan_manager.py`
- Modify: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `parse_plan`, `flatten_leaves`, `rewrite_plan` from Task 1
- Produces: `advance(plan_path, meta_path) -> AdvanceResult`,
  `advance_issue(plan_path, epic_path, meta_path) -> AdvanceResult` (dispatch)

- [ ] **Step 1: Write advance tests — linear queue**

```python
class TestAdvance:
    def _setup(self, tmp_path, plan_content, covers="42"):
        plan_file = tmp_path / ".plan"
        plan_file.write_text(plan_content)
        meta = tmp_path / ".meta"
        meta.write_text(f"branch: test\nissue: 42\ncovers: {covers}\n")
        return plan_file, meta

    def test_linear_advance(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        result = plan_manager.advance(plan_file, meta)
        assert result.completed == 42
        assert result.next_issue == 43
        assert result.epic_complete is False
        # Verify .plan rewritten
        tree = plan_manager.parse_plan(plan_file)
        assert tree.queue[0].completed is True
        assert tree.queue[1].active is True
        # Verify covers updated
        assert "42,43" not in meta.read_text()  # 43 not added yet — only completed issues
        assert "42" in meta.read_text()
```

- [ ] **Step 2: Write advance tests — epic boundary**

```python
    def test_epic_child_advance(self, tmp_path):
        plan_file, meta = self._setup(tmp_path, MULTI_ISSUE_PLAN, covers="42")
        result = plan_manager.advance(plan_file, meta)
        assert result.completed == 109
        assert result.next_issue == 110
        assert result.batch_complete is False

    def test_epic_last_child(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #50 — Epic (epic)\n  - [x] #108 — A\n  - [ ] #109 — B ← active\n- [ ] #32 — C\n\n## Session State\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        result = plan_manager.advance(plan_file, meta)
        assert result.completed == 109
        assert result.next_issue == 32
        # Epic parent should be marked complete
        tree = plan_manager.parse_plan(plan_file)
        assert tree.queue[0].completed is True
```

- [ ] **Step 3: Write advance tests — nested epic boundary**

```python
    def test_nested_epic_completes(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #50 — Epic (epic)\n  - [ ] #52 — Nested (epic)\n    - [x] #60 — A\n    - [ ] #61 — B ← active\n  - [ ] #53 — C\n\n## Session State\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        result = plan_manager.advance(plan_file, meta)
        assert result.completed == 61
        assert result.next_issue == 53
        tree = plan_manager.parse_plan(plan_file)
        assert tree.queue[0].children[0].completed is True  # nested epic marked done
```

- [ ] **Step 4: Write advance tests — queue exhausted**

```python
    def test_queue_exhausted(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [x] #42 — A\n- [ ] #43 — B ← active\n\n## Session State\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan)
        result = plan_manager.advance(plan_file, meta)
        assert result.completed == 43
        assert result.next_issue is None
        assert result.epic_complete is True
```

- [ ] **Step 5: Write advance tests — batch boundary safe_exit**

```python
    def test_batch_boundary_safe_exit(self, tmp_path):
        plan_file, meta = self._setup(tmp_path, BATCH_PLAN)
        # Advance past #108
        result1 = plan_manager.advance(plan_file, meta)
        assert result1.next_issue == 109
        # Advance past #109 (last in Batch 1)
        result2 = plan_manager.advance(plan_file, meta)
        assert result2.batch_complete is True
        assert result2.safe_exit is True
        assert result2.next_issue == 110
```

- [ ] **Step 6: Write advance tests — covers deduplication**

```python
    def test_covers_deduplication(self, tmp_path):
        plan = "# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n\n## Session State\nStarted: 2026-08-04\n"
        plan_file, meta = self._setup(tmp_path, plan, covers="42")
        result = plan_manager.advance(plan_file, meta)
        content = meta.read_text()
        # 42 should appear once, not twice
        assert content.count("42") == content.count("42")  # just verify no dups
        covers_line = [l for l in content.splitlines() if l.startswith("covers:")][0]
        nums = covers_line.split(":")[1].strip().split(",")
        assert len(nums) == len(set(nums))
```

- [ ] **Step 7: Write advance_issue dispatch test**

```python
class TestAdvanceIssueDispatch:
    def test_dispatches_to_plan(self, tmp_path):
        plan_file = tmp_path / ".plan"
        plan_file.write_text("# Work Plan — test\n\n## Queue\n- [ ] #42 — A ← active\n- [ ] #43 — B\n\n## Session State\nStarted: 2026-08-04\n")
        meta = tmp_path / ".meta"
        meta.write_text("branch: test\nissue: 42\ncovers: 42\n")
        result = plan_manager.advance_issue(plan_file, None, meta)
        assert result.completed == 42

    def test_falls_back_to_epic(self, tmp_path):
        # No .plan, only .epic — should delegate to epic_manager
        epic_file = tmp_path / ".slot"
        # ... (epic format content)
        # This test validates the dispatch, not epic_manager itself
```

- [ ] **Step 8: Run tests, verify fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestAdvance -v`

- [ ] **Step 9: Implement advance() and advance_issue()**

Key logic: find `← active` in flattened list, mark `[x]`, find next leaf,
mark `← active`, update `.meta` covers (with dedup), rewrite `.plan`,
compute `safe_exit`/`batch_complete`/`epic_complete` flags.

- [ ] **Step 10: Run tests, verify pass**

- [ ] **Step 11: Commit**

```bash
git commit -m "feat(#180): plan_manager advance — queue iteration with tree rewrite

Refs #180"
```

---

### Task 3: Worklog Issue Events

**Files:**
- Modify: `scripts/worklog.py`
- Modify: `tests/test_worklog.py` (or create if absent)

**Interfaces:**
- Produces: `record_issue_activate(conn, branch, repo_path, issue_number, issue_repo)`,
  `record_issue_complete(conn, branch, repo_path, issue_number, issue_repo)`

- [ ] **Step 1: Write tests**

```python
class TestIssueEvents:
    def test_record_issue_activate(self, tmp_path):
        db = str(tmp_path / "test.db")
        conn = worklog.connect(db)
        worklog.ensure_repo(conn, "/tmp/repo")
        worklog.record_work_start(conn, "issue-42", "/tmp/repo", 42, "Org/repo")
        worklog.record_issue_activate(conn, "issue-42", "/tmp/repo", 42, "Org/repo")
        events = worklog.event_log(conn, event_type="issue-activate")
        assert len(events) == 1
        assert '"issue_number": 42' in events[0]["metadata"]

    def test_record_issue_complete_updates_work_item_issues(self, tmp_path):
        db = str(tmp_path / "test.db")
        conn = worklog.connect(db)
        worklog.ensure_repo(conn, "/tmp/repo")
        worklog.record_work_start(conn, "issue-42", "/tmp/repo", 42, "Org/repo")
        worklog.record_issue_complete(conn, "issue-42", "/tmp/repo", 108, "Org/repo")
        # Check work_item_issues table has 108
        rows = conn.execute(
            "SELECT * FROM work_item_issues WHERE issue_number=108"
        ).fetchall()
        assert len(rows) == 1
        assert rows[0]["is_primary"] == 0
```

- [ ] **Step 2: Run tests, verify fail**

- [ ] **Step 3: Implement functions in worklog.py**

Add `record_issue_activate` and `record_issue_complete` with `@safe` decorator.
`record_issue_complete` also inserts into `work_item_issues` via `INSERT OR IGNORE`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#180): worklog issue-activate/issue-complete events

Refs #180"
```

---

### Task 4: ctx.py `PLAN_*` Output Variables

**Files:**
- Modify: `project/ctx.py`
- Modify: `tests/test_ctx.py` (or create)

**Interfaces:**
- Consumes: `plan_manager.detect()` from Task 1
- Produces: `HAS_PLAN`, `PLAN_PATH`, `PLAN_ACTIVE_ISSUE`, `PLAN_POSITION`, `PLAN_BATCH`

- [ ] **Step 1: Write tests**

```python
class TestCtxPlanOutput:
    def test_outputs_plan_variables(self, tmp_path):
        # Set up workspace with .plan
        design = tmp_path / "design"
        design.mkdir()
        (design / ".plan").write_text(MULTI_ISSUE_PLAN)
        (design / ".meta").write_text("branch: test\nissue: 42\ncovers: 42\nplan: yes\n")
        output = run_ctx(tmp_path)  # helper that runs ctx.py and parses KEY=VALUE
        assert output["HAS_PLAN"] == "yes"
        assert "PLAN_ACTIVE_ISSUE" in output
        assert output["PLAN_ACTIVE_ISSUE"] == "109"

    def test_no_plan(self, tmp_path):
        design = tmp_path / "design"
        design.mkdir()
        (design / ".meta").write_text("branch: test\nissue: 42\ncovers: 42\n")
        output = run_ctx(tmp_path)
        assert output["HAS_PLAN"] == "no"
```

- [ ] **Step 2: Run tests, verify fail**

- [ ] **Step 3: Implement — add plan detection to ctx.py**

Import `plan_manager.detect()`. When `.plan` exists, output `HAS_PLAN=yes`,
`PLAN_PATH`, `PLAN_ACTIVE_ISSUE`, `PLAN_POSITION`, `PLAN_BATCH`. When not,
output `HAS_PLAN=no`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#180): ctx.py PLAN_* output variables

Refs #180"
```

---

### Task 5: work_router.py `.plan` Detection

**Files:**
- Modify: `work/work_router.py`
- Modify: `tests/test_work_router.py` (if exists)

**Interfaces:**
- Consumes: `plan_manager.detect()` from Task 1, `PLAN_*` from ctx.py Task 4

- [ ] **Step 1: Write test for .plan detection in router**

- [ ] **Step 2: Run test, verify fail**

- [ ] **Step 3: Implement — import plan_manager.detect(), use for queue-aware routing**

Replace duplicate `epic_manager.detect()` call with `plan_manager.detect()` fallback
to `epic_manager.detect()`. Set `HAS_PLAN`, `PLAN_ACTIVE_ISSUE` etc. in router output.

- [ ] **Step 4: Run test, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#180): work_router.py .plan detection with epic fallback

Refs #180"
```

---

### Task 6: Lifecycle State Machine Updates

**Files:**
- Modify: `project/lifecycle.py`
- Modify: `tests/test_lifecycle.py`

**Interfaces:**
- Consumes: existing transition table
- Produces: updated table without `work_epic`/`slot_epic`, with `build_plan` effect,
  `write_plan_closed` replacing `write_epic_closed`

- [ ] **Step 1: Write tests for removed events**

```python
def test_work_epic_removed():
    with pytest.raises(InvalidTransition):
        transition(meta_path, "work_epic")  # from idle — should fail now

def test_slot_epic_removed():
    with pytest.raises(InvalidTransition):
        transition(meta_path, "slot_epic")
```

- [ ] **Step 2: Write tests for build_plan in work/slot_create effects**

```python
def test_work_includes_build_plan():
    result = transition(meta_path, "work")  # from idle
    assert "build_plan" in result.effects

def test_slot_create_includes_build_plan():
    result = transition(meta_path, "slot_create")
    assert "build_plan" in result.effects
```

- [ ] **Step 3: Write test for write_plan_closed**

```python
def test_cleanup_has_write_plan_closed():
    result = transition(meta_path, "cleanup_pass")  # from closing:stamped
    assert "write_plan_closed" in result.effects
    assert "write_epic_closed" not in result.effects
```

- [ ] **Step 4: Run tests, verify fail**

- [ ] **Step 5: Implement changes to lifecycle.py**

Remove `work_epic` and `slot_epic` rows from `TRANSITION_TABLE`. Remove from
`INVALID_MESSAGES`. Add `build_plan` to `work` and `slot_create` effects.
Replace `write_epic_closed` with `write_plan_closed` in `cleanup_pass`.

- [ ] **Step 6: Run full lifecycle test suite**

Run: `python3 -m pytest tests/test_lifecycle.py -v`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#180): lifecycle — remove work_epic/slot_epic, add build_plan effect

Refs #180"
```

---

### Task 7: Pause Stack `.plan` Fields

**Files:**
- Modify: `project/stack.py`
- Modify: `tests/test_stack.py` (if exists)

**Interfaces:**
- Produces: `plan_active_issue` and `plan_position` fields in stack entries

- [ ] **Step 1: Write tests**

```python
def test_push_with_plan_fields(tmp_path):
    stack_file = tmp_path / ".pause-stack"
    stack.push(stack_file, branch="issue-42-test", issue="42",
               plan_active_issue="109", plan_position="2/5")
    entries = stack.list_entries(stack_file)
    assert entries[0]["plan_active_issue"] == "109"
    assert entries[0]["plan_position"] == "2/5"
```

- [ ] **Step 2: Run test, verify fail**

- [ ] **Step 3: Implement — add fields to known_order and serialization**

Add `plan_active_issue` and `plan_position` to the `known_order` tuple in
`_entries_to_text()`. These are optional fields — absent when `.plan` doesn't exist.

- [ ] **Step 4: Run test, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#180): pause stack plan_active_issue/plan_position fields

Refs #180"
```

---

### Task 8: Skill Updates (work-start, work-end, work-slot, handover)

**Files:**
- Modify: `work-start/SKILL.md` — Step 3d (epic overlay reads `.plan`),
  Step 4 (issue resolution builds queue), Step 9 (scaffold adds `.plan`),
  Step 10 (commit stages `.plan`)
- Modify: `work-end/SKILL.md` — Pre-condition 0b (reads `.plan`),
  slot close (collapsed Phase A/B), remove `work-slot merge` references
- Modify: `work-slot/SKILL.md` — merge `create`/`epic` subcommands,
  add `add-repo`/`remove-repo`, remove `merge` subcommand
- Modify: `work/SKILL.md` — routing table update, remove `work epic` route
- Modify: `handover/SKILL.md` — Queue Progress section from `.plan`
- Modify: `handover/handover-reference.md` — template update

**Interfaces:**
- Consumes: `plan_manager` (Task 1-2), `PLAN_*` ctx.py variables (Task 4),
  lifecycle changes (Task 6), stack fields (Task 7)

This task is skill documentation — no Python tests. Validation is by
the existing SKILL.md validation suite (`python3 scripts/validate_all.py --tier commit`).

- [ ] **Step 1: Update work-start/SKILL.md**

Key changes:
- Step 3d: read `.plan` via `HAS_PLAN`/`PLAN_*` from ctx.py, fall back to `.epic`
- Step 4: after issue resolution, build `.plan` via `build_plan` effect (issue mode),
  or create empty `.plan` (free text mode)
- Step 9: scaffold.py invocation adds `plan=yes`
- Step 10: `commit-scaffold` stages `.plan`
- Detection table: add `HAS_PLAN` awareness

- [ ] **Step 2: Update work-end/SKILL.md**

Key changes:
- Pre-condition 0b: read `.plan` for queue completion status, `safe_exit` flag
- Slot close: collapse Phase A/B into single sequence. Remove "Phase A" / "Phase B"
  language. `work-end` in a slot runs: review, promote, squash, push, merge to
  original, stamp, archive — all in one sequence.
- Remove all references to `work-slot merge` as a separate step

- [ ] **Step 3: Update work-slot/SKILL.md**

Key changes:
- Merge `create` and `epic` subcommands into single `work-slot`
- Add `add-repo` and `remove-repo` subcommands (section 7.1/7.2 from spec)
- Remove `merge` subcommand (section 11.3)
- Update `next` to delegate to unified `work-next` (reads `.plan`)
- Update `status` to read `.plan` for progress

- [ ] **Step 4: Update work/SKILL.md**

Key changes:
- Remove `work epic #N` route from routing table
- `work #42 #50 #32` routes to work-start with multi-issue (auto-epic detection)
- `work next` reads `.plan` via `advance_issue` dispatch

- [ ] **Step 5: Update handover/SKILL.md and handover-reference.md**

Key changes:
- When `HAS_PLAN=yes`: read `.plan` for Queue Progress section
- Template: completed/active/pending counts from `.plan` tree
- Fall back to `IS_EPIC` codepath when no `.plan`

- [ ] **Step 6: Run skill validation**

Run: `python3 scripts/validate_all.py --tier commit`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#180): skill updates — unified work-start, work-end, work-slot, handover

Refs #180"
```

---

### Task 9: scaffold.py and branch_create.py Updates

**Files:**
- Modify: `work-start/scaffold.py`
- Modify: `work-start/branch_create.py`
- Modify: `tests/test_scaffold.py` (if exists)

**Interfaces:**
- Consumes: `plan_manager.build_plan_content()` from Task 1

- [ ] **Step 1: Write test — scaffold creates .plan**

```python
def test_scaffold_creates_plan(tmp_path):
    # Run scaffold.py with plan=yes
    result = run_scaffold(tmp_path, branch="issue-42-test", issue="42",
                         plan="yes", plan_content="...")
    plan_path = tmp_path / "design" / ".plan"
    assert plan_path.exists()
```

- [ ] **Step 2: Write test — commit-scaffold stages .plan**

```python
def test_commit_scaffold_stages_plan(tmp_path):
    # Setup git repo, run scaffold, run commit-scaffold
    # Verify .plan is in the committed tree
    result = subprocess.run(["git", "-C", str(tmp_path), "show", "HEAD:design/.plan"],
                          capture_output=True)
    assert result.returncode == 0
```

- [ ] **Step 3: Implement scaffold.py changes**

Accept `plan=yes` and `plan-content=<content>` parameters. Write `.plan` alongside
`.meta` and `JOURNAL.md`.

- [ ] **Step 4: Implement branch_create.py changes**

`commit-scaffold` stages `design/.plan` alongside `design/.meta` and `design/JOURNAL.md`.

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#180): scaffold creates .plan, commit-scaffold stages it

Refs #180"
```

---

### Task 10: slot_manager.py `add-repo` and `remove-repo`

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `add_repo(family_root, slot_number, repo_name, branch)`,
  `remove_repo(family_root, slot_number, repo_name)`

- [ ] **Step 1: Write add_repo tests**

```python
class TestAddRepo:
    def test_adds_repo_to_slot(self, tmp_path):
        family, originals, slot, branch = _create_slot_test_repos(tmp_path, ["engine"])
        slot_manager.add_repo(family, 1, "trellis", branch)
        assert (slot / "trellis").exists()
        assert (slot / "trellis" / ".git").exists()
        # .slot file updated
        info = slot_manager.parse_slot_md(slot / ".slot")
        assert "trellis" in info["repos"]

    def test_add_repo_creates_branch(self, tmp_path):
        family, originals, slot, branch = _create_slot_test_repos(tmp_path, ["engine"])
        slot_manager.add_repo(family, 1, "trellis", branch)
        current = subprocess.run(["git", "-C", str(slot / "trellis"), "branch", "--show-current"],
                                capture_output=True, text=True).stdout.strip()
        assert current == branch
```

- [ ] **Step 2: Write remove_repo tests**

```python
class TestRemoveRepo:
    def test_removes_repo_from_slot(self, tmp_path):
        family, originals, slot, branch = _create_slot_test_repos(tmp_path, ["engine", "trellis"])
        slot_manager.remove_repo(family, 1, "trellis")
        assert not (slot / "trellis").exists()
        info = slot_manager.parse_slot_md(slot / ".slot")
        assert "trellis" not in info["repos"]

    def test_refuses_to_remove_primary(self, tmp_path):
        family, originals, slot, branch = _create_slot_test_repos(tmp_path, ["engine"])
        with pytest.raises(ValueError, match="primary"):
            slot_manager.remove_repo(family, 1, "engine")
```

- [ ] **Step 3: Run tests, verify fail**

- [ ] **Step 4: Implement add_repo and remove_repo**

`add_repo`: mirrors `create_slot` logic for a single repo — `sync_main`, `git clone --shared`,
create branch, setup `.m2` config, resolve workspace, wire symlinks, update `.slot`.

`remove_repo`: guard checks (not primary, clean working tree), `relocate_claude_projects`,
remove directory, update `.slot`.

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#180): slot add-repo and remove-repo subcommands

Refs #180"
```

---

### Task 11: Epic Auto-Detection (`detect_epic` and `build_queue`)

**Files:**
- Create or modify: `work-slot/plan_manager.py` (add detection functions)
- Test: `tests/test_plan_manager.py` (add detection tests)

**Interfaces:**
- Produces: `detect_epic(issue_number, issue_repo) -> QueueItem`,
  `build_queue(issue_numbers, issue_repo, visited=None) -> list[QueueItem]`

Note: these functions call `gh issue view` — tests should mock the GitHub API
calls using `unittest.mock.patch` on `subprocess.run`.

- [ ] **Step 1: Write detect_epic tests**

```python
class TestDetectEpic:
    @patch("plan_manager._gh_issue_body")
    def test_detects_epic(self, mock_body):
        mock_body.return_value = "## Scope\n- [ ] #108 — Add weight\n- [ ] #109 — Update scoring\n"
        result = plan_manager.detect_epic(50, "Org/repo")
        assert result.is_epic is True
        assert len(result.children) == 2

    @patch("plan_manager._gh_issue_body")
    def test_detects_leaf(self, mock_body):
        mock_body.return_value = "Just a regular issue body"
        result = plan_manager.detect_epic(42, "Org/repo")
        assert result.is_epic is False
        assert result.children == []

    @patch("plan_manager._gh_issue_body")
    def test_skips_closed_children(self, mock_body):
        mock_body.return_value = "## Scope\n- [x] #108 — Done\n- [ ] #109 — Todo\n"
        result = plan_manager.detect_epic(50, "Org/repo")
        assert len(result.children) == 1
        assert result.children[0].issue_number == 109
```

- [ ] **Step 2: Write build_queue tests with cycle detection**

```python
class TestBuildQueue:
    @patch("plan_manager.detect_epic")
    def test_cycle_detection(self, mock_detect):
        # A contains B, B contains A
        mock_detect.side_effect = lambda n, r: QueueItem(n, f"Issue {n}", is_epic=True,
            children=[QueueItem(99 if n == 50 else 50, "Cycle")])
        queue = plan_manager.build_queue([50], "Org/repo")
        # Should not infinite loop — cycle detected
        assert len(queue) == 1
```

- [ ] **Step 3: Run tests, verify fail**

- [ ] **Step 4: Implement detect_epic and build_queue**

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#180): auto-epic detection with recursive expansion and cycle guard

Refs #180"
```

---

### Task 12: Integration Testing and Sync

**Files:**
- Run: `python3 scripts/validate_all.py --tier commit`
- Run: `python3 -m pytest tests/ -v`
- Sync: `python3 scripts/claude-skill sync-local --all -y`

- [ ] **Step 1: Run full test suite**

Run: `python3 -m pytest tests/ -v`

- [ ] **Step 2: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`

- [ ] **Step 3: Fix any failures**

- [ ] **Step 4: Sync skills**

Run: `python3 scripts/claude-skill sync-local --all -y`

- [ ] **Step 5: Final commit if needed**

---

## Follow-on: Trellis Integration (Separate Plan)

Phase 8 is implemented in the trellis repo (`/Users/mdproctor/claude/hortora/trellis`):

- `WorkspaceScanner.scanPlanFile()` — new `.plan` parser (tree, not flat)
- `WorklogReader` — new SQLite JDBC reader for `~/.hortora/worklog.db`
- `WorkPlan` / `QueueItem` sealed interface data model
- REST endpoint for active work + current issue
- `LifecycleManager` updates: call plan_manager instead of epic_manager
- `SlotInfo.isEpic` derivation from `.plan` existence

This requires a separate issue and plan in the trellis repo.

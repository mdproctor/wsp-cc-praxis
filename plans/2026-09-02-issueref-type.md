# IssueRef Type Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #268 — IssueRef type: repo-qualify all issue references in .plan
**Issue group:** #268

**Goal:** Replace all bare `int` issue references with a frozen `IssueRef`
dataclass that carries `repo` and `number` as an indivisible unit, validated
at construction.

**Architecture:** Introduce `IssueRef` in `plan_manager.py`, replace
`issue_number: int` + `repo: str` on all dataclasses, propagate through
matching functions, parser, writer, CLI, events, and downstream consumers.
Worklog DB schema stays unchanged — `IssueRef` decomposes at the persistence
boundary.

**Tech Stack:** Python 3.14, dataclasses, pytest

## Global Constraints

- All `.plan` queue lines must be repo-qualified (`owner/repo#N`)
- `IssueRef.repo` is case-normalized to lowercase
- Worklog DB schema stays as `(issue_number INTEGER, issue_repo TEXT)`
- No backward compatibility for bare `#N` in `.plan` files — hard fail

---

## Batch 1: IssueRef type + data model + parser

Everything builds on the type definition and the data model changes.
After this batch, `IssueRef` exists, all dataclasses use `ref: IssueRef`,
the parser enforces repo-qualified lines, and the writer uses `str(ref)`.
Tests pass for the core data layer.

### Task 1: IssueRef type and dataclass field changes

**Files:**
- Modify: `work-slot/plan_manager.py` (lines 1-76: imports, dataclasses)
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Produces: `IssueRef(repo: str, number: int)` frozen dataclass with
  `parse(s: str) -> IssueRef`, `__str__() -> str`, `__post_init__`
  validation. `QueueItem.ref: IssueRef`, `LeafItem.ref: IssueRef`,
  `PlanTree.current_issue: IssueRef | None`,
  `AdvanceResult.completed: IssueRef`,
  `AdvanceResult.next_issue: IssueRef | None`.

- [ ] **Step 1: Write IssueRef tests**

```python
# Add at top of test_plan_manager.py, after imports:
from plan_manager import IssueRef

class TestIssueRef:
    def test_construction_valid(self):
        ref = IssueRef("hortora/soredium", 42)
        assert ref.repo == "hortora/soredium"
        assert ref.number == 42

    def test_str(self):
        ref = IssueRef("hortora/soredium", 42)
        assert str(ref) == "hortora/soredium#42"

    def test_case_normalization(self):
        ref = IssueRef("Hortora/Soredium", 42)
        assert ref.repo == "hortora/soredium"
        assert IssueRef("Hortora/Soredium", 42) == IssueRef("hortora/soredium", 42)

    def test_frozen(self):
        ref = IssueRef("hortora/soredium", 42)
        with pytest.raises(AttributeError):
            ref.number = 99

    def test_hashable(self):
        r1 = IssueRef("hortora/soredium", 42)
        r2 = IssueRef("Hortora/Soredium", 42)
        assert hash(r1) == hash(r2)
        assert len({r1, r2}) == 1

    def test_empty_repo_raises(self):
        with pytest.raises(ValueError, match="repo-qualified"):
            IssueRef("", 42)

    def test_no_slash_raises(self):
        with pytest.raises(ValueError, match="repo-qualified"):
            IssueRef("soredium", 42)

    def test_parse_valid(self):
        ref = IssueRef.parse("hortora/soredium#42")
        assert ref.repo == "hortora/soredium"
        assert ref.number == 42

    def test_parse_bare_number_raises(self):
        with pytest.raises(ValueError, match="must be owner/repo#N"):
            IssueRef.parse("#42")

    def test_parse_malformed_raises(self):
        with pytest.raises(ValueError, match="must be owner/repo#N"):
            IssueRef.parse("not-valid")

    def test_parse_case_normalizes(self):
        ref = IssueRef.parse("Hortora/Soredium#42")
        assert ref.repo == "hortora/soredium"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_plan_manager.py::TestIssueRef -v`
Expected: FAIL — `ImportError: cannot import name 'IssueRef'`

- [ ] **Step 3: Implement IssueRef class**

Add to `work-slot/plan_manager.py` after imports, before `TaskItem`:

```python
@dataclass(frozen=True)
class IssueRef:
    repo: str
    number: int

    def __post_init__(self):
        if not self.repo or '/' not in self.repo:
            raise ValueError(
                f"IssueRef requires repo-qualified format, "
                f"got repo='{self.repo}'"
            )
        object.__setattr__(self, 'repo', self.repo.lower())

    def __str__(self) -> str:
        return f"{self.repo}#{self.number}"

    @classmethod
    def parse(cls, s: str) -> 'IssueRef':
        m = re.match(r'^([A-Za-z0-9._-]+/[A-Za-z0-9._-]+)#(\d+)$', s)
        if not m:
            raise ValueError(
                f"Invalid issue reference '{s}' — must be owner/repo#N"
            )
        return cls(m.group(1), int(m.group(2)))
```

- [ ] **Step 4: Run IssueRef tests to verify they pass**

Run: `python3 -m pytest tests/test_plan_manager.py::TestIssueRef -v`
Expected: PASS

- [ ] **Step 5: Update dataclass fields**

Change `QueueItem`:
```python
@dataclass
class QueueItem:
    ref: IssueRef
    title: str
    completed: bool = False
    active: bool = False
    is_epic: bool = False
    children: list['QueueItem'] = field(default_factory=list)
    batch: str | None = None
    tasks: list[TaskItem] = field(default_factory=list)
```

Change `PlanTree`:
```python
@dataclass
class PlanTree:
    heading: str
    queue: list[QueueItem]
    current_issue: IssueRef | None
    started: str
    last_wrap: str | None = None
    deferred: list[DeferredItem] = field(default_factory=list)
    state: dict[str, str] = field(default_factory=dict)
```

Change `LeafItem`:
```python
@dataclass
class LeafItem:
    ref: IssueRef
    title: str
    completed: bool
    active: bool
    parent_epic: IssueRef | None
    batch: str | None
```

Change `AdvanceResult`:
```python
@dataclass
class AdvanceResult:
    completed: IssueRef
    next_issue: IssueRef | None
    next_title: str | None = None
    batch_complete: bool = False
    epic_complete: bool = False
    safe_exit: bool = False
    has_deferred: bool = False
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: add IssueRef frozen dataclass and update data model field types Refs #268"
```

### Task 2: Parser, writer, and _backfill_repo removal

**Files:**
- Modify: `work-slot/plan_manager.py` (parser `parse_plan`, writer `_write_item`,
  `build_plan_content`, delete `_backfill_repo`)
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `IssueRef` from Task 1
- Produces: `parse_plan(plan_path) -> PlanTree` that raises on bare `#N`,
  `_write_item` that uses `str(item.ref)`,
  `build_plan_content` with repo-qualified state section

- [ ] **Step 1: Write parser strict-mode tests**

```python
class TestParserStrict:
    def test_bare_number_raises(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("# Work Plan — test\n\n## Queue\n- [ ] #42 — Bare number\n")
        with pytest.raises(ValueError, match="must be owner/repo#N"):
            plan_manager.parse_plan(plan)

    def test_repo_qualified_parses(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text("# Work Plan — test\n\n## Queue\n- [ ] test/repo#42 — Title ← active\n")
        tree = plan_manager.parse_plan(plan)
        assert tree.queue[0].ref == IssueRef("test/repo", 42)
        assert tree.queue[0].ref.repo == "test/repo"
        assert tree.queue[0].ref.number == 42
```

- [ ] **Step 2: Run to verify failure**

Run: `python3 -m pytest tests/test_plan_manager.py::TestParserStrict -v`
Expected: FAIL — `QueueItem` constructor signature mismatch or AttributeError on `.ref`

- [ ] **Step 3: Update `parse_plan` parser**

In `parse_plan`, update the `_ITEM_RE` match handler (around line 261-281):

```python
item_m = _ITEM_RE.match(line)
if item_m:
    indent = len(item_m.group(1))
    completed = item_m.group(2) == "x"
    repo = item_m.group(3)
    if not repo:
        raise ValueError(
            f"Bare issue number on line {i+1}: '{line.strip()}' "
            f"— must be owner/repo#N format"
        )
    ref = IssueRef(repo, int(item_m.group(4)))
    title_raw = item_m.group(5).strip()
    title = _EPIC_MARKER_RE.sub("", title_raw).strip()
    title = _ACTIVE_MARKER_RE.sub("", title).strip()
    is_epic = bool(_EPIC_MARKER_RE.search(item_m.group(0)))
    active = bool(_ACTIVE_MARKER_RE.search(item_m.group(0)))

    item = QueueItem(
        ref=ref,
        title=title,
        completed=completed,
        active=active,
        is_epic=is_epic,
        batch=current_batch,
    )
```

Update `_CURRENT_RE` to capture repo (around line 90):
```python
_CURRENT_RE = re.compile(r'^Current:\s*(?:([A-Za-z0-9._-]+/[A-Za-z0-9._-]+)#)?(\d+)')
```

Update current_issue parsing (around line 213):
```python
cm = _CURRENT_RE.match(stripped)
if cm:
    repo_str = cm.group(1) or ""
    num = int(cm.group(2))
    if repo_str:
        current_issue = IssueRef(repo_str, num)
    else:
        current_issue = None  # bare number in Current: line — will be set from active leaf
```

- [ ] **Step 4: Delete `_backfill_repo`**

Remove the `_backfill_repo` function (lines 144-149) and its call site
in `parse_plan` (search for `_backfill_repo`).

- [ ] **Step 5: Update `_write_item`**

```python
def _write_item(item: QueueItem, lines: list[str], indent: int) -> None:
    prefix = "  " * indent
    check = "x" if item.completed else " "
    epic_marker = " (epic)" if item.is_epic else ""
    active_marker = " ← active" if item.active else ""
    lines.append(f"{prefix}- [{check}] {item.ref} — {item.title}{epic_marker}{active_marker}")
```

- [ ] **Step 6: Update `build_plan_content` state section**

In `build_plan_content` (around lines 424-441), update the state section writer:

```python
if not state:
    lines.append("")
    lines.append("## State")
    lines.append(f"branch: {branch_slug}")
    lines.append("state: active")
    active_leaf = _find_active_leaf(items)
    if active_leaf:
        lines.append(f"issue-repo: {active_leaf.ref.repo}")
        lines.append(f"covers: {active_leaf.ref}")
        lines.append(f"Current: {active_leaf.ref} — {active_leaf.title}")
    else:
        lines.append("Current: none")
    lines.append(f"date: {date}")
    if last_wrap:
        lines.append(f"Last wrap: {last_wrap}")
```

- [ ] **Step 7: Update `flatten_leaves` to use `ref`**

In `flatten_leaves` (around line 358-376), update `LeafItem` construction:

```python
result.append(LeafItem(
    ref=item.ref,
    title=item.title,
    completed=item.completed,
    active=item.active,
    parent_epic=parent_epic,  # already IssueRef | None from caller
    batch=item.batch,
))
```

And update the recursive call that passes `parent_epic`:
```python
_flatten_items(item.children, result, parent_epic=item.ref)
```

- [ ] **Step 8: Update all test fixtures**

Update the test plan constants at the top of `test_plan_manager.py`.
The `## Session State` `Current:` lines already use bare `#N` — update
them to use repo-qualified refs where needed:

```python
# Current: lines that reference issue numbers need repo prefix
# e.g. "Current: test/repo#42 — Fix login validation"
```

All `QueueItem(issue_number=N, ..., repo="test/repo")` in tests become
`QueueItem(ref=IssueRef("test/repo", N), ...)`.

All `assert item.issue_number == N` become `assert item.ref.number == N`
or `assert item.ref == IssueRef("test/repo", N)`.

- [ ] **Step 9: Run full test suite**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: PASS (all existing tests updated + new strict parser tests)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: strict parser rejects bare #N, writer uses str(ref), remove _backfill_repo Refs #268"
```

## Batch 2: Matching functions + advance + queue operations

After this batch, all matching, advance, reorder, remove, append, and
collect functions use `IssueRef`. The core bug fix (cross-repo matching)
is live.

### Task 3: Matching and advance functions

**Files:**
- Modify: `work-slot/plan_manager.py` (`_mark_completed`, `_mark_active`,
  `mark_completed`, `_collect_issue_numbers` → `_collect_refs`,
  `advance`, `advance_issue`, `complete_active_issue`,
  `_emit_issue_events`, `get_completed_epic_parents`)
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `IssueRef`, `QueueItem.ref`, `LeafItem.ref` from Tasks 1-2
- Produces: `_mark_completed(items, ref: IssueRef)`,
  `mark_completed(plan_path, ref: IssueRef)`,
  `_mark_active(items, ref: IssueRef)`,
  `_collect_refs(items) -> set[IssueRef]`,
  `advance(plan_path) -> AdvanceResult` with `.completed: IssueRef`,
  `complete_active_issue(plan_path, repo_path) -> IssueRef | None`,
  `get_completed_epic_parents(plan_path) -> list[IssueRef]`

- [ ] **Step 1: Write cross-repo matching test**

```python
class TestCrossRepoMatching:
    def test_mark_completed_distinguishes_repos(self, tmp_path):
        """Two items with same number in different repos — only the correct one is marked."""
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan — test\n\n## Queue\n"
            "- [ ] repo-a/proj#42 — Task in repo A ← active\n"
            "- [ ] repo-b/proj#42 — Task in repo B\n"
        )
        tree = plan_manager.parse_plan(plan)
        ref_b = IssueRef("repo-b/proj", 42)
        plan_manager._mark_completed(tree.queue, ref_b)
        assert not tree.queue[0].completed  # repo-a#42 untouched
        assert tree.queue[1].completed      # repo-b#42 marked

    def test_collect_refs_cross_repo(self, tmp_path):
        plan = tmp_path / ".plan"
        plan.write_text(
            "# Work Plan — test\n\n## Queue\n"
            "- [ ] repo-a/proj#42 — Task A ← active\n"
            "- [ ] repo-b/proj#42 — Task B\n"
        )
        tree = plan_manager.parse_plan(plan)
        refs = plan_manager._collect_refs(tree.queue)
        assert len(refs) == 2
        assert IssueRef("repo-a/proj", 42) in refs
        assert IssueRef("repo-b/proj", 42) in refs
```

- [ ] **Step 2: Run to verify failure**

Run: `python3 -m pytest tests/test_plan_manager.py::TestCrossRepoMatching -v`
Expected: FAIL — `_mark_completed` still takes `int`, `_collect_refs` doesn't exist

- [ ] **Step 3: Update `_mark_completed` and `_mark_active`**

```python
def _mark_completed(items: list[QueueItem], ref: IssueRef) -> bool:
    for item in items:
        if item.ref == ref and not item.is_epic:
            item.completed = True
            item.active = False
            item.tasks = []
            return True
        if item.is_epic and item.children:
            if _mark_completed(item.children, ref):
                return True
    return False


def mark_completed(plan_path: Path, ref: IssueRef) -> bool:
    tree = parse_plan(plan_path)
    changed = _mark_completed(tree.queue, ref)
    if changed:
        rewrite_plan(plan_path, tree)
    return changed


def _mark_active(items: list[QueueItem], ref: IssueRef) -> bool:
    for item in items:
        if item.ref == ref and not item.is_epic:
            item.active = True
            return True
        if item.is_epic and item.children:
            if _mark_active(item.children, ref):
                return True
    return False
```

- [ ] **Step 4: Rename `_collect_issue_numbers` to `_collect_refs`**

```python
def _collect_refs(items: list[QueueItem]) -> set[IssueRef]:
    refs: set[IssueRef] = set()
    for item in items:
        refs.add(item.ref)
        if item.children:
            refs.update(_collect_refs(item.children))
    return refs
```

Update all call sites (`append_to_queue` uses this).

- [ ] **Step 5: Update `advance` function**

Update all `.issue_number` references in `advance()` to `.ref`:

```python
_mark_completed(tree.queue, completed_leaf.ref)
# ...
_mark_active(tree.queue, next_leaf.ref)
# ...
tree.current_issue = next_leaf.ref if next_leaf else None
# ...
_emit_issue_events(plan_path, repo_path, completed_leaf.ref,
                   next_leaf.ref if next_leaf else None)
# ...
return AdvanceResult(
    completed=completed_leaf.ref,
    next_issue=next_leaf.ref if next_leaf else None,
    # ...
)
```

- [ ] **Step 6: Update `_emit_issue_events`**

```python
def _emit_issue_events(plan_path: Path, repo_path: str,
                       completed: IssueRef,
                       next_issue: IssueRef | None) -> None:
    # ...
    worklog.record_issue_complete(conn, branch, repo_path,
                                  completed.number, completed.repo)
    if next_issue is not None:
        worklog.record_issue_activate(conn, branch, repo_path,
                                      next_issue.number, next_issue.repo)
```

- [ ] **Step 7: Update `complete_active_issue` and `get_completed_epic_parents`**

```python
def complete_active_issue(plan_path: Path, repo_path: str) -> IssueRef | None:
    tree = parse_plan(plan_path)
    active = _find_active_leaf(tree.queue)
    if not active:
        return None
    _emit_issue_events(plan_path, repo_path, active.ref, next_issue=None)
    return active.ref


def get_completed_epic_parents(plan_path: Path) -> list[IssueRef]:
    tree = parse_plan(plan_path)
    result: list[IssueRef] = []
    _collect_completed_epics(tree.queue, result)
    return result


def _collect_completed_epics(items: list, result: list[IssueRef]) -> None:
    for item in items:
        if item.is_epic and item.children:
            _collect_completed_epics(item.children, result)
            if item.completed or all(c.completed for c in item.children):
                result.append(item.ref)
```

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: matching functions key by IssueRef — fixes cross-repo collision bug Refs #268"
```

### Task 4: Queue operations (reorder, remove, append) and epic building

**Files:**
- Modify: `work-slot/plan_manager.py` (`reorder_queue`, `remove_from_queue`,
  `append_to_queue`, `promote_deferred`, `promote_selected`,
  `build_queue`, `detect_epic`, `create_main_plan`, `detect`)
- Test: `tests/test_plan_manager.py`

**Interfaces:**
- Consumes: `IssueRef`, `_collect_refs`, `_mark_active` from Task 3
- Produces: `reorder_queue(plan_path, order: list[IssueRef])`,
  `remove_from_queue(plan_path, refs: list[IssueRef])`,
  `build_queue(refs: list[IssueRef])`,
  `detect(ws)` with `ACTIVE_ISSUE=owner/repo#N`

- [ ] **Step 1: Update `reorder_queue`**

```python
def reorder_queue(plan_path: Path, order: list[IssueRef]) -> list[IssueRef]:
    tree = parse_plan(plan_path)
    by_ref = {item.ref: item for item in tree.queue}
    reordered: list[QueueItem] = []
    seen: set[IssueRef] = set()
    for ref in order:
        if ref in by_ref and ref not in seen:
            reordered.append(by_ref[ref])
            seen.add(ref)
    for item in tree.queue:
        if item.ref not in seen:
            reordered.append(item)
    for item in reordered:
        item.active = False
    _set_first_uncompleted_active(reordered)
    tree.queue = reordered
    active = _find_active_leaf(tree.queue)
    tree.current_issue = active.ref if active else None
    rewrite_plan(plan_path, tree)
    return [item.ref for item in reordered]
```

- [ ] **Step 2: Update `remove_from_queue`**

```python
def remove_from_queue(plan_path: Path, refs: list[IssueRef]) -> list[IssueRef]:
    tree = parse_plan(plan_path)
    to_remove = set(refs)
    active = _find_active_leaf(tree.queue)
    if active and active.ref in to_remove:
        raise ValueError(
            f"Cannot remove active item {active.ref} — "
            f"advance or complete it first"
        )
    removed: list[IssueRef] = []

    def _filter(items: list[QueueItem]) -> list[QueueItem]:
        kept: list[QueueItem] = []
        for item in items:
            if item.ref in to_remove and not item.children:
                removed.append(item.ref)
                continue
            if item.children:
                item.children = _filter(item.children)
                if not item.children and item.ref in to_remove:
                    removed.append(item.ref)
                    continue
            kept.append(item)
        return kept

    tree.queue = _filter(tree.queue)
    new_active = _find_active_leaf(tree.queue)
    tree.current_issue = new_active.ref if new_active else None
    rewrite_plan(plan_path, tree)
    return removed
```

- [ ] **Step 3: Update `append_to_queue`**

```python
def append_to_queue(plan_path: Path, new_items: list[QueueItem],
                    position: int | None = None) -> list[QueueItem]:
    tree = parse_plan(plan_path)
    existing_refs = _collect_refs(tree.queue)
    deduped = [item for item in new_items if item.ref not in existing_refs]
    skipped = len(new_items) - len(deduped)
    if skipped > 0:
        for item in new_items:
            if item.ref in existing_refs:
                print(f"SKIPPED_DUP={item.ref}")
    # ... rest unchanged except:
    active = _find_active_leaf(tree.queue)
    if active:
        tree.current_issue = active.ref
    rewrite_plan(plan_path, tree)
    return deduped
```

- [ ] **Step 4: Update `promote_deferred` and `promote_selected`**

Update both functions — synthetic issue numbers for deferred items:
```python
issue_repo = tree.state.get("issue-repo", "") if tree.state else ""
for d in to_promote:
    tree.queue.append(QueueItem(
        ref=IssueRef(issue_repo, next_issue_num),
        title=d.title,
    ))
    next_issue_num += 1
```

Note: `issue_repo` must be non-empty for `IssueRef` construction. If
empty, raise a clear error:
```python
if not issue_repo:
    raise ValueError("Cannot promote deferred items — no issue-repo in plan state")
```

- [ ] **Step 5: Update `build_queue` and `detect_epic`**

```python
def build_queue(refs: list[IssueRef],
                visited: set[int] | None = None) -> list[QueueItem]:
    if visited is None:
        visited = set()
    items = []
    for ref in refs:
        if ref.number in visited:
            continue
        visited.add(ref.number)
        result = detect_epic(ref)
        items.append(result)
    return items


def detect_epic(ref: IssueRef) -> QueueItem:
    body = _gh_issue_body(ref)
    title = _gh_issue_title(ref) or ""
    # ... parse children, each child gets IssueRef(ref.repo, child_num)
    children.append(QueueItem(ref=IssueRef(ref.repo, child_num), title=child_title))
    # ...
    if children:
        return QueueItem(ref=ref, title=title, is_epic=True, children=children)
    return QueueItem(ref=ref, title=title)
```

Update `_gh_issue_body` and `_gh_issue_title` to take `IssueRef`:
```python
def _gh_issue_body(ref: IssueRef) -> str:
    # ... use str(ref.number), "--repo", ref.repo

def _gh_issue_title(ref: IssueRef) -> str:
    # ... use str(ref.number), "--repo", ref.repo
```

- [ ] **Step 6: Update `create_main_plan`**

```python
def create_main_plan(workspace_path: Path, project_name: str,
                     items: list[dict], issue_repo: str = "") -> Path:
    plan_path = workspace_path / ".plan"
    queue_items = []
    for i, item in enumerate(items):
        repo = item.get("repo", issue_repo)
        queue_items.append(QueueItem(
            ref=IssueRef(repo, item["number"]),
            title=item["title"],
            active=(i == 0),
        ))
    # ...
```

- [ ] **Step 7: Update `detect` function**

Find the `detect()` function and update `active_issue` output:
```python
"active_issue": str(active.ref),
"active_issue_repo": active.ref.repo,
```

- [ ] **Step 8: Update CLI `reorder` and `remove` commands**

Add convenience resolution for bare numbers in the CLI `main()` function:

```python
def _resolve_ref(ref_str: str, tree: PlanTree) -> IssueRef:
    """Resolve a ref string — full ref or bare number within loaded plan."""
    try:
        return IssueRef.parse(ref_str)
    except ValueError:
        pass
    if ref_str.isdigit():
        num = int(ref_str)
        matches = [item for item in tree.queue if item.ref.number == num]
        if len(matches) == 1:
            return matches[0].ref
        if len(matches) > 1:
            repos = [str(m.ref) for m in matches]
            raise ValueError(
                f"Ambiguous issue number #{num} — matches: {', '.join(repos)}. "
                f"Use full owner/repo#N format."
            )
    raise ValueError(f"Cannot resolve '{ref_str}' — use owner/repo#N format")
```

Update CLI `reorder` and `remove` command handlers to use `_resolve_ref`.

- [ ] **Step 9: Update CLI `append` output**

```python
print(f"APPENDED={item.ref} — {item.title}")
```

And update worklog duplicate check:
```python
active = _wl.check_active_work(_conn, item.ref.number, item.ref.repo)
```

- [ ] **Step 10: Run full test suite**

Run: `python3 -m pytest tests/test_plan_manager.py -v`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/plan_manager.py tests/test_plan_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: queue operations use IssueRef — reorder, remove, append, build_queue, detect Refs #268"
```

## Batch 3: Events, commands, and downstream consumers

After this batch, the full stack uses `IssueRef` — events, registry,
commands, work_health, work_chain. All tests pass.

### Task 5: Events and command layer

**Files:**
- Modify: `commands/events.py`
- Modify: `commands/registry.py`
- Modify: `commands/next.py`
- Modify: `commands/what_next.py`
- Modify: `commands/start.py`
- Test: `tests/test_commands.py` (if it tests event construction)

**Interfaces:**
- Consumes: `IssueRef` from Task 1
- Produces: All event dataclasses with `IssueRef` fields,
  `Context.issue: IssueRef | None`, `Context.epic_active_issue: IssueRef | None`

- [ ] **Step 1: Update `events.py`**

Add import at top:
```python
from __future__ import annotations
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "work-slot"))
from plan_manager import IssueRef
```

Update all `int` issue fields to `IssueRef`:

- `BriefReady.issue: IssueRef | None`, `BriefReady.epic_active_issue: IssueRef | None`
- `ContinueReady.issue: IssueRef | None`
- `Recommendation.issue: IssueRef`
- `BranchCreated.issues: list[IssueRef]`
- `PlanAdvanced.completed_issue: IssueRef`, `PlanAdvanced.next_issue: IssueRef | None`
- `WorkEnded.issues_closed: list[IssueRef]`
- `IssueContext.issue: IssueRef`
- `SessionStarted.issue: IssueRef | None`
- `RepoSlotInfo.issue: IssueRef | None`

- [ ] **Step 2: Update `registry.py`**

```python
from plan_manager import IssueRef

# In Context dataclass:
issue: IssueRef | None
epic_active_issue: IssueRef | None

# In resolve_context:
issue_str = raw.get("ISSUE_N") or raw.get("PLAN_ACTIVE_ISSUE") or ""
try:
    issue = IssueRef.parse(issue_str) if issue_str and '/' in issue_str else None
except ValueError:
    issue = None

epic_str = raw.get("EPIC_ACTIVE_ISSUE", "")
try:
    epic_active_issue = IssueRef.parse(epic_str) if epic_str and '/' in epic_str else None
except ValueError:
    epic_active_issue = None
```

- [ ] **Step 3: Update `next.py`**

```python
completed_issue = active[0].ref  # was: active[0].issue_number

events.append(PlanAdvanced(
    completed_issue=completed_issue,
    next_issue=next_active[0].ref if next_active else None,
    # ...
))
```

- [ ] **Step 4: Update `what_next.py`**

```python
Recommendation(
    issue=IssueRef(repo, item.issue_number),  # needs repo context
    # ...
)
```

Note: `enrichment.what_next_typed` returns items with `.issue_number` (int).
The repo is known from `ctx.owner_repo`. Construct with:
```python
Recommendation(
    issue=IssueRef(repo, item.issue_number),
    title=item.title,
    strategic_role=item.strategic_role,
    readiness=item.readiness,
    reason=item.reason,
)
```

- [ ] **Step 5: Update `start.py`**

Change parameter type:
```python
def execute(issues: list[IssueRef] | None = None, ...)
```

Update branch name derivation:
```python
issue = issues[0]
branch = f"issue-{issue.number}-work"
```

Update scaffold call:
```python
scaffold_result = scaffold(
    workspace=workspace,
    branch=branch,
    project_sha=project_sha,
    issue=str(issue.number),
    issue_repo=issue.repo,
    covers=" ".join(str(i.number) for i in issues),
    plan=len(issues) > 1,
)
```

Update event:
```python
events.append(BranchCreated(branch=branch, issues=issues, plan_path=...))
```

- [ ] **Step 6: Run command tests**

Run: `python3 -m pytest tests/test_commands.py -v`
Expected: PASS (or skip if test file doesn't exist)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add commands/ tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: events and commands use IssueRef throughout Refs #268"
```

### Task 6: Downstream consumers (work_health, work_chain)

**Files:**
- Modify: `project/work_health.py`
- Modify: `project/work_chain.py`
- Test: `tests/test_plan_manager.py` (for work_health integration if applicable)

**Interfaces:**
- Consumes: `IssueRef`, `LeafItem.ref`, `mark_completed(plan_path, ref)` from Tasks 1-3
- Produces: `check_plan_state` with per-repo API grouping (D7),
  `format_resume_display` with repo-qualified display,
  `_check_issue_state(ref: IssueRef)` in work_chain

- [ ] **Step 1: Update `check_plan_state` — per-repo grouping (D7)**

This is the algorithmic fix. Replace the single `gh issue list` call:

```python
# Group open issues by repo
by_repo: dict[str, list] = {}
for leaf in open_issues:
    by_repo.setdefault(leaf.ref.repo, []).append(leaf)

changed = []
for repo, leaves in by_repo.items():
    try:
        result = subprocess.run(
            ["gh", "issue", "list", "--repo", repo,
             "--state", "all", "--limit", "200",
             "--json", "number,state,title"],
            capture_output=True, text=True, timeout=15,
        )
        if result.returncode != 0:
            continue
        import json
        issues = {i["number"]: i for i in json.loads(result.stdout)}
    except (subprocess.TimeoutExpired, Exception):
        continue

    for leaf in leaves:
        gh = issues.get(leaf.ref.number)
        if gh and gh["state"] == "CLOSED":
            if leaf.title and gh.get("title"):
                plan_t = leaf.title.lower()
                gh_t = gh["title"].lower()
                if plan_t not in gh_t and gh_t not in plan_t:
                    continue
            mark_completed(plan_path, leaf.ref)
            changed.append(f"{leaf.ref} now CLOSED")
        elif not gh:
            # Individual lookup for issues not in the batch
            try:
                result = subprocess.run(
                    ["gh", "issue", "view", str(leaf.ref.number),
                     "--repo", leaf.ref.repo,
                     "--json", "state,title",
                     "--jq", "[.state, .title] | @tsv"],
                    capture_output=True, text=True, timeout=5,
                )
                if result.returncode != 0:
                    continue
                parts = result.stdout.strip().split("\t", 1)
                if len(parts) != 2:
                    continue
                gh_state, gh_title = parts
                if gh_state == "CLOSED":
                    if leaf.title and gh_title:
                        plan_t = leaf.title.lower()
                        gh_t = gh_title.lower()
                        if plan_t not in gh_t and gh_t not in plan_t:
                            continue
                    mark_completed(plan_path, leaf.ref)
                    changed.append(f"{leaf.ref} now CLOSED")
            except subprocess.TimeoutExpired:
                pass
```

- [ ] **Step 2: Update `format_resume_display`**

```python
for l in completed:
    lines.append(f"  ✅ {l.ref} — {l.title}")
for l in active:
    lines.append(f"  → {l.ref} — {l.title} (current)")
for l in pending:
    lines.append(f"     {l.ref} — {l.title}")
```

- [ ] **Step 3: Update `work_chain.py`**

```python
def _check_issue_state(ref: IssueRef) -> str:
    if not ref:
        return "unknown"
    try:
        result = subprocess.run(
            ["gh", "issue", "view", str(ref.number),
             "--repo", ref.repo,
             "--json", "state", "--jq", ".state"],
            capture_output=True, text=True, timeout=10,
        )
        return result.stdout.strip().lower() if result.returncode == 0 else "unknown"
    except (subprocess.TimeoutExpired, Exception):
        return "unknown"
```

Update callers of `_check_issue_state` to pass `IssueRef` instead of
separate `(issue_number, issue_repo)`.

- [ ] **Step 4: Run all tests**

Run: `python3 -m pytest tests/ -v --ignore=tests/test_tui_app.py --ignore=tests/test_tui_home_view.py --ignore=tests/test_tui_project_view.py -x`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/ tests/
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: work_health groups by repo, work_chain takes IssueRef — fixes #268 root cause Refs #268"
```

## Batch 4: Pre-commit hook and validation

After this batch, the pre-commit hook prevents bare issue numbers from
entering `.plan` files via git commits in workspace repos.

### Task 7: Pre-commit hook

**Files:**
- Create: `scripts/validate_plan_refs.sh` (hook script)
- Modify: `work-start/scaffold.py` or `workspace-init/SKILL.md` (installation docs)
- Test: manual validation with sample `.plan` files

**Interfaces:**
- Consumes: `.plan` file format
- Produces: git pre-commit hook that validates repo-qualified queue lines

- [ ] **Step 1: Write the hook script**

Create `scripts/validate_plan_refs.sh`:

```bash
#!/bin/bash
# Pre-commit hook: validate .plan queue lines are repo-qualified.
# Install: cp to .git/hooks/pre-commit in workspace repos.

staged=$(git diff --cached --name-only | grep '\.plan$')
[ -z "$staged" ] && exit 0

for f in $staged; do
    bare=$(git show ":$f" | grep -nE '^\s*- \[[ x]\] #[0-9]' | head -1)
    if [ -n "$bare" ]; then
        echo "ERROR: Bare issue number in $f line $(echo "$bare" | cut -d: -f1)"
        echo "  All queue items must use owner/repo#N format."
        echo "  Run: python3 scripts/migrate_plan_repos.py to fix."
        exit 1
    fi
done
exit 0
```

- [ ] **Step 2: Make executable**

```bash
chmod +x /Users/mdproctor/claude/hortora/soredium/scripts/validate_plan_refs.sh
```

- [ ] **Step 3: Test with a bad .plan**

```bash
python3 -c "
from pathlib import Path
import tempfile, subprocess, os

d = tempfile.mkdtemp()
subprocess.run(['git', 'init', d], capture_output=True)
subprocess.run(['git', '-C', d, 'commit', '--allow-empty', '-m', 'init'], capture_output=True)

# Install hook
hook_path = Path(d) / '.git' / 'hooks' / 'pre-commit'
hook_path.write_text(Path('/Users/mdproctor/claude/hortora/soredium/scripts/validate_plan_refs.sh').read_text())
hook_path.chmod(0o755)

# Write bad plan
plan = Path(d) / '.plan'
plan.write_text('# Test\n\n## Queue\n- [ ] #42 — Bare number\n')
subprocess.run(['git', '-C', d, 'add', '.plan'], capture_output=True)
result = subprocess.run(['git', '-C', d, 'commit', '-m', 'test'], capture_output=True, text=True)
print('PASS' if result.returncode != 0 and 'Bare issue number' in result.stderr + result.stdout else 'FAIL')
print(result.stdout)
print(result.stderr)
"
```

- [ ] **Step 4: Test with a good .plan**

```bash
python3 -c "
from pathlib import Path
import tempfile, subprocess

d = tempfile.mkdtemp()
subprocess.run(['git', 'init', d], capture_output=True)
subprocess.run(['git', '-C', d, 'commit', '--allow-empty', '-m', 'init'], capture_output=True)

hook_path = Path(d) / '.git' / 'hooks' / 'pre-commit'
hook_path.write_text(Path('/Users/mdproctor/claude/hortora/soredium/scripts/validate_plan_refs.sh').read_text())
hook_path.chmod(0o755)

plan = Path(d) / '.plan'
plan.write_text('# Test\n\n## Queue\n- [ ] test/repo#42 — Good ref\n')
subprocess.run(['git', '-C', d, 'add', '.plan'], capture_output=True)
result = subprocess.run(['git', '-C', d, 'commit', '-m', 'test'], capture_output=True, text=True)
print('PASS' if result.returncode == 0 else 'FAIL')
"
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/validate_plan_refs.sh
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: add pre-commit hook to validate .plan repo-qualified refs Refs #268"
```

---

## References

- [2026-09-02-issueref-type-design.md] — design spec this plan implements
- [decisions.md] — 9 validated decisions (D1-D9)
- [work-slot/plan_manager.py] — core data model, parser, writer (~67 sites)
- [commands/events.py] — event dataclasses (11 sites)
- [commands/registry.py] — context resolution (4 sites)
- [commands/next.py, what_next.py, start.py] — command consumers
- [project/work_health.py] — check_plan_state algorithmic fix (D7)
- [project/work_chain.py] — issue state checking
- [tests/test_plan_manager.py] — ~80+ test site updates
- [GE-20260811-7e119c] — cross-repo resolution bug (root cause)
- [GitHub #268] — IssueRef type issue

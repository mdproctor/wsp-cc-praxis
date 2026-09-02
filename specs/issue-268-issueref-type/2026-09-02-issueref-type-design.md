# IssueRef Type: Repo-Qualify All Issue References in .plan

**Issue:** Hortora/soredium#268
**Date:** 2026-09-02
**Branch:** issue-268-issueref-type

## Problem

The `.plan` file uses bare issue numbers (`#42`) throughout, but queues
routinely contain issues from multiple repos. The `issue-repo` field
identifies where the epic lives, not where every child issue lives.

This causes two concrete bugs:

1. **S8_QUEUE_INCONSISTENT false positives** — `check_plan_state` in
   `work_health.py` resolves every `#N` against a single `resolve_repo`.
   When `#332` in `casehubio/casehub-pages` gets resolved against
   `casehubio/parent`, it finds a completely different (closed) issue
   and flags phantom corruption.

2. **Silent wrong completions** — `_mark_completed(items, issue_number: int)`
   matches by bare int. If two items from different repos share a number,
   the first match wins silently — marking the wrong issue as done.

Garden entry GE-20260811-7e119c documents a live instance: `.plan` built
from an epic's `Covers:` list resolved bare numbers against the parent
repo, producing wrong entries for every number collision.

## Solution: IssueRef frozen dataclass

Replace all bare `int` issue references with a single type that carries
`repo` and `number` as an indivisible unit, validated at construction.

```python
@dataclass(frozen=True)
class IssueRef:
    repo: str    # owner/repo — validated non-empty, case-normalized
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
        m = re.match(
            r'^([A-Za-z0-9._-]+/[A-Za-z0-9._-]+)#(\d+)$', s
        )
        if not m:
            raise ValueError(
                f"Invalid issue reference '{s}' — must be owner/repo#N"
            )
        return cls(m.group(1), int(m.group(2)))
```

**Key properties:**
- Frozen: immutable, hashable, usable as dict keys and set members
- Case normalization: `Hortora/soredium` and `hortora/soredium` compare equal
- Construction-time validation: invalid refs can't exist in memory
- `parse()` classmethod: single parse point replacing scattered regex extraction
- `__str__()`: canonical `owner/repo#N` format for display and serialisation

## Data Model Changes

### plan_manager.py

| Before | After |
|--------|-------|
| `QueueItem.issue_number: int` + `repo: str` | `QueueItem.ref: IssueRef` |
| `LeafItem.issue_number: int` + `repo: str` | `LeafItem.ref: IssueRef` |
| `LeafItem.parent_epic: int \| None` | `LeafItem.parent_epic: IssueRef \| None` |
| `AdvanceResult.completed: int` | `AdvanceResult.completed: IssueRef` |
| `AdvanceResult.next_issue: int \| None` | `AdvanceResult.next_issue: IssueRef \| None` |
| `PlanTree.current_issue: int \| None` | `PlanTree.current_issue: IssueRef \| None` |

**Removed:** `_backfill_repo()` — no longer needed. The parser enforces
repo-qualified lines; construction-time validation prevents empty repos.

### events.py

All `issue: int | None` fields become `issue: IssueRef | None`. Events
are in-process Python dataclasses (not serialised to JSON) — carrying
`IssueRef` directly gives consumers `.repo` and `.number` access plus
`str(ref)` for display.

| Dataclass | Field changes |
|-----------|--------------|
| `BriefReady` | `issue: int \| None` → `issue: IssueRef \| None`; `epic_active_issue: int \| None` → `epic_active_issue: IssueRef \| None` |
| `ContinueReady` | `issue: int \| None` → `issue: IssueRef \| None` |
| `Recommendation` | `issue: int` → `issue: IssueRef` |
| `BranchCreated` | `issues: list[int]` → `issues: list[IssueRef]` |
| `PlanAdvanced` | `completed_issue: int` → `completed_issue: IssueRef`; `next_issue: int \| None` → `next_issue: IssueRef \| None` |
| `WorkEnded` | `issues_closed: list[int]` → `issues_closed: list[IssueRef]` |
| `IssueContext` | `issue: int` → `issue: IssueRef` |
| `RepoSlotInfo` | `issue: int \| None` → `issue: IssueRef \| None` |
| `SessionStarted` | `issue: int \| None` → `issue: IssueRef \| None` |

### commands/registry.py

`Context.issue` (if it exists as a bare int) becomes `IssueRef | None`.

## Function Signature Changes

### Matching functions (the core fix — D8)

All issue-matching functions key by `IssueRef` instead of bare `int`.
This fixes the silent wrong-repo collision bug.

| Function | Before | After |
|----------|--------|-------|
| `_mark_completed(items, issue_number: int)` | Matches first `item.issue_number == issue_number` | Matches `item.ref == ref` (compares both repo and number) |
| `_mark_active(items, issue_number: int)` | Same bare int match | `_mark_active(items, ref: IssueRef)` |
| `mark_completed(plan_path, issue_number: int)` | Public entry point, bare int | `mark_completed(plan_path, ref: IssueRef)` |
| `_collect_issue_numbers(items)` → `set[int]` | Deduplication by number only | `_collect_refs(items)` → `set[IssueRef]` (cross-repo correct) |
| `reorder_queue(plan_path, order: list[int])` | Keys by bare int | `reorder_queue(plan_path, order: list[IssueRef])` |
| `remove_from_queue(plan_path, issue_numbers: list[int])` | Keys by bare int | `remove_from_queue(plan_path, refs: list[IssueRef])` |
| `get_completed_epic_parents(plan_path)` → `list[int]` | Returns bare ints | → `list[IssueRef]` |

### Queue management functions

| Function | Before | After |
|----------|--------|-------|
| `complete_active_issue(plan_path, repo_path)` → `int \| None` | Returns bare int | → `IssueRef \| None` |
| `advance(plan_path, repo_path)` → `AdvanceResult` | `.completed: int`, `.next_issue: int` | `.completed: IssueRef`, `.next_issue: IssueRef \| None` |
| `build_queue(issue_numbers: list[int], issue_repo: str)` | Bare ints + repo string | `build_queue(refs: list[IssueRef])` — repo already in each ref |
| `create_main_plan(workspace_path, project_name, items, issue_repo)` | Items as `{"number": int, "title": str}` + global `issue_repo` | Items as `{"ref": IssueRef, "title": str}` or `list[QueueItem]` |
| `detect(ws)` → KEY=VALUE output | `ACTIVE_ISSUE=268` | `ACTIVE_ISSUE=hortora/soredium#268` + `ACTIVE_ISSUE_REPO=hortora/soredium` |

### Worklog boundary (D6)

`_emit_issue_events(plan_path, repo_path, completed: int, next_issue: int | None)`
changes to take `IssueRef` and decomposes at the call to `worklog.record_issue_complete()`:

```python
def _emit_issue_events(plan_path, repo_path,
                       completed: IssueRef,
                       next_issue: IssueRef | None):
    # ...
    worklog.record_issue_complete(
        conn, branch, repo_path,
        completed.number, completed.repo
    )
    if next_issue is not None:
        worklog.record_issue_activate(
            conn, branch, repo_path,
            next_issue.number, next_issue.repo
        )
```

The worklog DB schema stays as `(issue_number INTEGER, issue_repo TEXT)`.

## Parser Changes

### Strict parsing (D3)

The `_ITEM_RE` regex already captures an optional `repo` group. After the
refactor, if the regex matches but the repo group is empty, the parser
raises `ValueError` instead of setting `repo = ""`.

```python
item_m = _ITEM_RE.match(line)
if item_m:
    repo = item_m.group(3)
    if not repo:
        raise ValueError(
            f"Bare issue number on line {i+1}: '{line.strip()}' "
            f"— must be owner/repo#N format"
        )
    ref = IssueRef(repo, int(item_m.group(4)))
    item = QueueItem(ref=ref, ...)
```

No backward-compatible backfill mode. The existing `migrate_plan_repos.py`
script handles migration of any active `.plan` files with bare numbers.

### `build_plan_content` / `_write_item`

`_write_item` currently validates `item.repo` at write time. After the
refactor, the write guard moves to `IssueRef.__post_init__` — by the time
`_write_item` runs, `item.ref` is guaranteed valid.

```python
def _write_item(item: QueueItem, lines: list[str], indent: int):
    prefix = "  " * indent
    check = "x" if item.completed else " "
    epic_marker = " (epic)" if item.is_epic else ""
    active_marker = " ← active" if item.active else ""
    lines.append(
        f"{prefix}- [{check}] {item.ref} — {item.title}"
        f"{epic_marker}{active_marker}"
    )
```

## check_plan_state fix (D7)

The current implementation makes one `gh issue list --repo resolve_repo`
call for all issues. After IssueRef, each leaf carries its own `ref.repo`.

```python
by_repo: dict[str, list[LeafItem]] = {}
for leaf in open_issues:
    by_repo.setdefault(leaf.ref.repo, []).append(leaf)

for repo, leaves in by_repo.items():
    nums = [str(l.ref.number) for l in leaves]
    result = subprocess.run(
        ["gh", "issue", "list", "--repo", repo,
         "--state", "closed", "--json", "number,state,title",
         "--jq", f'[.[] | select(.number == ({" or .number == ".join(nums)}))]'],
        capture_output=True, text=True, timeout=15,
    )
    # ... match by ref, not bare number
```

This is O(distinct_repos) — in practice 1-2 calls.

## CLI Changes (D8)

### append command

Already requires `owner/repo#N:title` — no change needed.

### reorder and remove commands

Currently take bare issue numbers. After IssueRef:
- **Full refs always accepted:** `owner/repo#N`
- **Convenience resolution:** For bare numbers, the CLI looks up the
  matching `QueueItem` in the loaded plan. When unambiguous (one item
  with that number), resolution is automatic. When ambiguous (same
  number in different repos), the CLI errors and requires the full ref.

This is NOT a backfill from `issue-repo` — it's a lookup within the
parsed plan's own data.

### detect() output (D9)

```
ACTIVE_ISSUE=hortora/soredium#268
ACTIVE_ISSUE_REPO=hortora/soredium
```

Both fields are emitted. `ACTIVE_ISSUE_REPO` eases transition for shell
callers that need the repo separately.

### ctx.py ISSUE_N field

`ctx.py` emits `ISSUE_N` (extracted from `.plan`'s `covers:` field).
After the refactor, `ISSUE_N` becomes the full ref string
(`hortora/soredium#268`) so `commands/registry.py` can parse it with
`IssueRef.parse()`. The `covers:` field in `.plan`'s `## State` section
also becomes repo-qualified (`covers: hortora/soredium#268`).

## Downstream Consumers

### work_health.py

- `check_plan_state()`: Groups by repo, resolves per-repo (see above)
- `format_resume_display()`: Uses `str(l.ref)` for display instead of
  bare `#{l.issue_number}`
- `mark_completed()` calls: Pass `IssueRef` instead of bare int

### work_chain.py

- `_check_issue_state(issue_number: str, issue_repo: str)`: Changes to
  `_check_issue_state(ref: IssueRef)`, decomposes for `gh issue view`

### commands/next.py

- Reads `active[0].ref` instead of `active[0].issue_number`
- Constructs `PlanAdvanced` with `IssueRef` fields

### commands/what_next.py

- Constructs `Recommendation` with `IssueRef` instead of bare int

### commands/brief.py

- Reads `ref` instead of `issue_number` for `BriefReady` construction

## Pre-commit Hook (D5)

New git pre-commit hook for workspace repos. Validates `.plan` file
content on every commit.

```bash
#!/bin/bash
# Validate .plan queue lines are repo-qualified
staged=$(git diff --cached --name-only | grep '\.plan$')
[ -z "$staged" ] && exit 0

for f in $staged; do
    if git show ":$f" | grep -nE '^\s*- \[[ x]\] #[0-9]' | head -1; then
        echo "ERROR: Bare issue number in $f — must be owner/repo#N"
        exit 1
    fi
done
exit 0
```

The regex catches lines like `- [ ] #42 — Title` (bare number) but
passes `- [ ] Hortora/soredium#42 — Title` (repo-qualified).

**Installation:** Added to `workspace-init`. For existing workspaces,
installed alongside `migrate_plan_repos.py` execution.

## Migration

**Prerequisite:** Run `migrate_plan_repos.py` across all active workspace
trees before deploying the strict parser. The script:

1. Infers repos from `.slot` files, directory structure, or CLAUDE.md
2. Parses and rewrites `.plan` to inject repo prefixes
3. Is idempotent — safe to run multiple times

Most active `.plan` files are already repo-qualified (the `_write_item()`
guard has been in place since the write validation was added).

## Testing

### Unit tests (test_plan_manager.py)

All existing tests update to use repo-qualified issue references:
- `QueueItem(issue_number=42, ...)` → `QueueItem(ref=IssueRef("hortora/soredium", 42), ...)`
- Assertions on `.issue_number` → assertions on `.ref.number` or `.ref`
- New tests for:
  - `IssueRef.__post_init__` validation (empty repo, no slash, case normalization)
  - `IssueRef.parse()` (valid, bare number raises, malformed raises)
  - `IssueRef` equality and hashing (case-insensitive repo)
  - Parser rejection of bare `#N` lines
  - Cross-repo matching: two items with same number in different repos
    are correctly distinguished by `_mark_completed` and `_collect_refs`
  - CLI convenience resolution: bare number → unambiguous lookup succeeds,
    ambiguous number → error

### Integration tests

- `check_plan_state` with multi-repo queue — verifies per-repo API calls
- Pre-commit hook validation on sample `.plan` files

## File Inventory

~94 sites across 10 files, plus ~80+ test sites.

| File | Sites | Nature of change |
|------|-------|-----------------|
| `work-slot/plan_manager.py` | ~67 | Core: IssueRef type, data model, parser, writer, matching, query, advance, epic building, persistence boundary. Delete `_backfill_repo()` and `_write_item` repo guard. |
| `commands/events.py` | 11 | Type changes: `int` → `IssueRef` on 9 dataclasses |
| `commands/registry.py` | 4 | `Context.issue`, `Context.epic_active_issue` → `IssueRef`; parsing from ctx.py output |
| `commands/next.py` | 3 | Read `ref` instead of `issue_number` for `PlanAdvanced` construction |
| `commands/what_next.py` | 1 | Construct `Recommendation` with `IssueRef` |
| `commands/start.py` | 1 | Parameter `issues: list[int]` → `list[IssueRef]` |
| `project/work_health.py` | 6+ | `check_plan_state` per-repo grouping (algorithmic), `format_resume_display`, `mark_completed` calls |
| `project/work_chain.py` | 1 | `_check_issue_state` takes `IssueRef` |
| `tests/test_plan_manager.py` | ~80+ | All test updates + new IssueRef-specific tests |
| Hook script (new) | 1 | Pre-commit hook for workspace repos |

## References

- [Hortora/soredium#268](https://github.com/Hortora/soredium/issues/268) — issue body with full scope
- [GE-20260811-7e119c] — Slot .plan cross-repo issue numbers resolved against parent repo
- [work-slot/plan_manager.py] — current data model and parser
- [commands/events.py] — current event types
- [project/work_health.py:300-348] — `check_plan_state` with single-repo resolution bug
- [decisions.md](decisions.md) — 9 decisions (D1-D9), standard review approved round 3

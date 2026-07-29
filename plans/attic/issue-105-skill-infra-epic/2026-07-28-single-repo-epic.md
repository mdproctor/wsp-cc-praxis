# Single-Repo Epic Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development). Steps use checkbox (`- [ ]`) syntax
> for tracking.

**Focal issue:** #100 — Single-repo epic support
**Issue group:** #100, #103

**Goal:** Enable single-repo projects to iterate through an epic's child
issues sequentially on a single branch, without slot/worktree
infrastructure.

**Architecture:** Generalise `epic_manager.py` from slot-dir-based to
path-based. Add `.epic` detection to `work_router.py` and `ctx.py`. Add
`work epic` and `work next` verbs to `work/SKILL.md`. Fix #103
(start vs resume on feature branches).

**Tech Stack:** Python 3.11+, pytest, Markdown skills

## Global Constraints

- Pre-release platform — breaking API changes are fine
- `.epic` file uses Unicode `←` markers (not ASCII `<-`)
- `.epic` existence is the single signal for epic state (no `type: epic`
  in `.meta`)
- Issue closure deferred to work-end via COVERS (not on `work next`)
- Batch grouping is LLM-driven (not in epic_manager.py)

---

### Task 1: Generalise epic_manager.py to accept file paths

**Files:**
- Modify: `work-slot/epic_manager.py`
- Modify: `tests/test_epic_manager.py`

**Interfaces:**
- Produces:
  - `parse_batch_plan(epic_path: Path) -> dict` — accepts full path to
    `.slot` or `.epic`
  - `advance(epic_path: Path, meta_path: Path | None = None) -> dict` —
    returns dict with keys: `completed`, `next_issue`,
    `next_issue_title`, `batch_complete`, `epic_complete`, `safe_exit`,
    `epic_number`, `epic_repo`
  - `status(epic_path: Path) -> dict`
  - `_rewrite_epic_file(epic_path: Path, completed, next_issue,
    next_title, next_batch_num, batches) -> None`
  - `write_epic_file(epic_path: Path, heading: str,
    repos: list[str] | None, issue: str, issue_repo: str,
    batches: list[dict], context: str) -> None`
  - `write_epic(workspace: Path, issue: str, slug: str,
    issue_repo: str, batches: list[dict], context: str) -> None`

- [ ] **Step 1: Update tests — change all slot_dir calls to epic_path**

Replace every `parse_batch_plan(tmp_path)` with
`parse_batch_plan(tmp_path / ".slot")`. Same for `advance()`,
`status()`. This is mechanical — the tests already write `.slot` files
to `tmp_path`, now they pass the full path instead of the directory.

Also update the `_setup_slot` helper to return the epic_path:

```python
def _setup_slot(self, tmp_path, content, covers=""):
    (tmp_path / ".slot").write_text(content)
    meta = tmp_path / "design" / ".meta"
    meta.parent.mkdir(parents=True, exist_ok=True)
    meta.write_text(f"branch: issue-50-test\ncovers: {covers}\n")
    return tmp_path / ".slot"  # return epic_path, not meta
```

Update all test methods: `epic_path = self._setup_slot(...)` then pass
`epic_path` to functions, and `epic_path.parent / "design" / ".meta"`
for meta_path.

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_epic_manager.py -v`
Expected: FAIL — function signatures changed but implementation hasn't

- [ ] **Step 3: Implement the generalisation**

In `epic_manager.py`:

a) Change `parse_batch_plan(slot_dir: Path)` → `parse_batch_plan(epic_path: Path)`:
   - Remove `slot_md = slot_dir / ".slot"` line
   - Use `epic_path` directly: `if not epic_path.exists(): ...`
   - `content = epic_path.read_text()`

b) Change `advance(slot_dir, meta_path)` → `advance(epic_path, meta_path)`:
   - `plan = parse_batch_plan(epic_path)` (not slot_dir)
   - `_rewrite_epic_file(epic_path, ...)` (not _rewrite_slot_md)
   - Add `epic_number` and `epic_repo` to return dict:
     ```python
     return {
         "completed": current,
         "next_issue": next_issue,
         "next_issue_title": next_title,
         "batch_complete": batch_complete,
         "epic_complete": epic_complete,
         "safe_exit": safe_exit,
         "epic_number": plan["epic_number"],
         "epic_repo": plan["epic_repo"],
     }
     ```

c) Rename `_rewrite_slot_md` → `_rewrite_epic_file(epic_path, ...)`:
   - Remove `slot_md = slot_dir / ".slot"` line
   - Use `epic_path` directly for read/write

d) Change `status(slot_dir)` → `status(epic_path)`:
   - `plan = parse_batch_plan(epic_path)`

e) Rename `write_epic_slot_md` → `write_epic_file(epic_path, heading,
   repos, issue, issue_repo, batches, context)`:
   - `epic_path` replaces `slot_dir / ".slot"` for output
   - `heading` replaces inline `f"# Slot {slot_number} — {branch}"`
   - `repos` is `list[str] | None` — skip `## Repos` section when None
   - Remove `slot_number`, `branch` params

f) Add `write_epic` convenience:
   ```python
   def write_epic(workspace: Path, issue: str, slug: str,
                  issue_repo: str, batches: list[dict],
                  context: str) -> None:
       epic_path = workspace / "design" / ".epic"
       heading = f"# Epic #{issue} — {slug}"
       write_epic_file(epic_path, heading, repos=None,
                       issue=issue, issue_repo=issue_repo,
                       batches=batches, context=context)
   ```

g) Update CLI dispatch:
   ```python
   command = sys.argv[1]
   epic_path = Path(sys.argv[2])

   if command == "plan":
       result = parse_batch_plan(epic_path)
   elif command == "advance":
       meta_path = None
       epic_dir = epic_path.parent
       # Search for .meta: check sibling design/.meta first,
       # then subdirectories (slot layout)
       sibling_meta = epic_dir / "design" / ".meta"
       if sibling_meta.exists():
           meta_path = sibling_meta
       elif epic_dir.is_dir():
           for sub in epic_dir.iterdir():
               if not sub.is_dir():
                   continue
               candidate = sub / "design" / ".meta"
               if candidate.exists():
                   meta_path = candidate
                   break
               for ws_sub in sub.iterdir():
                   if ws_sub.is_dir():
                       candidate = ws_sub / "design" / ".meta"
                       if candidate.exists():
                           meta_path = candidate
                           break
               if meta_path:
                   break
       result = advance(epic_path, meta_path=meta_path)
   elif command == "status":
       result = status(epic_path)
   ```

h) Update module docstring:
   ```
   Subcommands:
     plan <epic-path>      Parse epic file, return batch plan as JSON
     advance <epic-path>   Advance to next issue, update file + .meta
     status <epic-path>    Return progress summary as JSON

   Operates on the ## Batch Plan section of .slot or .epic files.
   ```

- [ ] **Step 4: Add tests for new features**

Add test for `advance()` returning `epic_number` and `epic_repo`:
```python
def test_advance_returns_epic_number(self, tmp_path):
    epic_path = self._setup_slot(tmp_path, SAMPLE_EPIC_SLOT_MD,
                                 covers="108")
    meta_path = tmp_path / "design" / ".meta"
    result = advance(epic_path, meta_path=meta_path)
    assert result["epic_number"] == "50"
    assert result["epic_repo"] == "casehubio/engine"
```

Add test for `write_epic()` (single-repo convenience):
```python
def test_write_epic_creates_file(self, tmp_path):
    (tmp_path / "design").mkdir(parents=True)
    batches = [{"number": 1, "name": "Core",
                "issues": [{"number": 10, "title": "First"}]}]
    write_epic(tmp_path, issue="100", slug="my-epic",
               issue_repo="Hortora/soredium", batches=batches,
               context="Test epic")
    epic_path = tmp_path / "design" / ".epic"
    assert epic_path.exists()
    content = epic_path.read_text()
    assert "# Epic #100 — my-epic" in content
    assert "## Repos" not in content  # no repos in single-repo
    assert "Type: epic" in content
    assert "← active" in content
```

Add test for `write_epic_file()` with repos=None skipping section:
```python
def test_write_epic_file_no_repos_section(self, tmp_path):
    epic_path = tmp_path / ".epic"
    batches = [{"number": 1, "name": "Core",
                "issues": [{"number": 10, "title": "First"}]}]
    write_epic_file(epic_path, "# Epic #100 — test", repos=None,
                    issue="100", issue_repo="Hortora/soredium",
                    batches=batches, context="Test")
    content = epic_path.read_text()
    assert "## Repos" not in content
```

- [ ] **Step 5: Run all tests**

Run: `pytest tests/test_epic_manager.py -v`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat(#100): generalise epic_manager.py to accept file paths

parse_batch_plan, advance, status now take epic_path (full path to .slot
or .epic) instead of slot_dir. advance() returns epic_number and
epic_repo. write_epic_slot_md renamed to write_epic_file with explicit
output path. New write_epic() convenience for single-repo .epic files.

Refs #100
```

---

### Task 2: Add `.epic` detection to work_router.py

**Files:**
- Modify: `work/work_router.py`
- Modify: `tests/test_work_router.py`

**Interfaces:**
- Consumes: `.epic` file format (same `## Issue` / `## Batch Plan` /
  `## Session State` as `.slot`)
- Produces: output keys `EPIC_PATH`, `IS_EPIC`, `EPIC_BATCH`,
  `EPIC_ACTIVE_ISSUE` when `.epic` detected in workspace/design/

- [ ] **Step 1: Write failing tests**

```python
class TestEpicFileDetection:
    def test_detects_epic_file_in_workspace(self, tmp_path):
        """Single-repo epic: .epic in workspace/design/."""
        project = tmp_path / "project"
        workspace = tmp_path / "workspace"
        project.mkdir()
        workspace.mkdir()
        design = workspace / "design"
        design.mkdir(parents=True)
        (design / ".epic").write_text(
            "## Issue\nHortora/soredium#100\nType: epic\n\n"
            "## Batch Plan\n### Batch 1 — Core\n"
            "- [ ] #102 — First ← active\n\n"
            "## Session State\nCurrent batch: 1\n"
            "Current issue: #102 — First\n"
        )
        result = detect_state("issue-100-epic", str(project),
                              str(workspace))
        assert result["IS_EPIC"] == "yes"
        assert result["EPIC_PATH"] == str(design / ".epic")
        assert result["EPIC_BATCH"] == "1 of 1"
        assert result["EPIC_ACTIVE_ISSUE"] == "102"

    def test_epic_file_not_present(self, tmp_path):
        """No .epic file — IS_EPIC stays no."""
        project = tmp_path / "project"
        workspace = tmp_path / "workspace"
        project.mkdir()
        workspace.mkdir()
        result = detect_state("main", str(project), str(workspace))
        assert result["IS_EPIC"] == "no"
        assert "EPIC_PATH" not in result

    def test_slot_takes_precedence_over_epic(self, tmp_path):
        """In a slot with .slot, .epic in workspace is ignored."""
        family = tmp_path / "family"
        slot = family / "worktrees" / "1"
        project = slot / "repo"
        workspace = tmp_path / "workspace"
        slot.mkdir(parents=True)
        project.mkdir()
        workspace.mkdir()
        (slot / ".slot").write_text(
            "## Issue\nrepo#42\nType: epic\n\n"
            "## Batch Plan\n### Batch 1 — X\n"
            "- [ ] #10 — Y ← active\n\n"
            "## Session State\nCurrent batch: 1\n"
            "Current issue: #10 — Y\n"
        )
        design = workspace / "design"
        design.mkdir(parents=True)
        (design / ".epic").write_text("## Issue\nother#99\nType: epic\n")
        result = detect_state("issue-42-spi", str(project),
                              str(workspace))
        assert result["IN_SLOT"] == "yes"
        assert result["SLOT_PATH"] == str(slot / ".slot")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_work_router.py::TestEpicFileDetection -v`
Expected: FAIL

- [ ] **Step 3: Implement `.epic` detection**

In `work_router.py`, after the existing `/worktrees/` + `.slot`
detection block (around line 80), add:

```python
# Single-repo epic detection (only if not already in a slot)
if not in_slot:
    epic_candidate = workspace / "design" / ".epic"
    if epic_candidate.exists():
        epic_content = epic_candidate.read_text()
        epic_in_issue = False
        for line in epic_content.splitlines():
            if line.startswith("## Issue"):
                epic_in_issue = True
                continue
            if line.startswith("## ") and epic_in_issue:
                break
            if epic_in_issue and line.strip() == "Type: epic":
                is_epic = True

        if is_epic:
            epic_batch_numbers = re.findall(
                r"^### Batch (\d+)", epic_content, re.MULTILINE
            )
            epic_total = len(epic_batch_numbers)

            m = re.search(
                r"^Current batch:\s*(\d+)", epic_content,
                re.MULTILINE
            )
            epic_current = m.group(1) if m else "0"
            epic_batch = f"{epic_current} of {epic_total}"

            m = re.search(
                r"^Current issue:\s*#(\d+)", epic_content,
                re.MULTILINE
            )
            epic_active_issue = m.group(1) if m else ""

            slot_path = str(epic_candidate)  # reuse slot_path var
```

Update the result dict output section — `EPIC_PATH` should be emitted
instead of `SLOT_PATH` when the source is `.epic` (not `.slot`).
Introduce a flag to track which file was the source:

Replace the single `slot_path` variable with two:
- Keep `slot_path` for `.slot` detection
- Add `epic_file_path` for `.epic` detection

Then in the result output:
```python
if slot_path:
    result["SLOT_PATH"] = slot_path
if epic_file_path:
    result["EPIC_PATH"] = epic_file_path
```

- [ ] **Step 4: Run all work_router tests**

Run: `pytest tests/test_work_router.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#100): detect .epic in work_router.py

Single-repo epic detection: when not in a slot, check
workspace/design/.epic for Type: epic. Outputs EPIC_PATH,
IS_EPIC, EPIC_BATCH, EPIC_ACTIVE_ISSUE — same keys as slot
detection. Slot takes precedence when both exist.

Refs #100
```

---

### Task 3: Add `.epic` detection to ctx.py

**Files:**
- Modify: `project/ctx.py`
- Modify: `tests/test_ctx.py`

**Interfaces:**
- Produces: output keys `IS_EPIC` (yes/no), `EPIC_PATH` (full path or
  empty)

- [ ] **Step 1: Write failing test**

Add to `tests/test_ctx.py`:

```python
class TestEpicDetection:
    def test_epic_file_detected(self, tmp_path):
        repo = init_repo(tmp_path / "project", claude_md=MINIMAL_CLAUDE)
        design = repo / "design"
        design.mkdir(parents=True)
        (design / ".epic").write_text(
            "## Issue\nHortora/soredium#100\nType: epic\n"
        )
        result = run_ctx(repo)
        vals = parse(result)
        assert vals["IS_EPIC"] == "yes"
        assert vals["EPIC_PATH"] == str(design / ".epic")

    def test_no_epic_file(self, tmp_path):
        repo = init_repo(tmp_path / "project", claude_md=MINIMAL_CLAUDE)
        result = run_ctx(repo)
        vals = parse(result)
        assert vals["IS_EPIC"] == "no"
        assert vals["EPIC_PATH"] == ""
```

(Use whatever `MINIMAL_CLAUDE` / `init_repo` pattern exists in the test
file.)

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_ctx.py::TestEpicDetection -v`
Expected: FAIL — IS_EPIC and EPIC_PATH not in output

- [ ] **Step 3: Implement detection**

In `ctx.py`, after the `.meta` parsing block (around line 62), add:

```python
epic_path = Path(workspace) / "design" / ".epic"
is_epic = False
if epic_path.exists():
    epic_content = epic_path.read_text()
    if "Type: epic" in epic_content:
        is_epic = True
```

In the output section (around line 140), add:

```python
print(f"IS_EPIC={'yes' if is_epic else 'no'}")
print(f"EPIC_PATH={str(epic_path) if is_epic else ''}")
```

- [ ] **Step 4: Run all ctx tests**

Run: `pytest tests/test_ctx.py -v`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat(#100): detect .epic in ctx.py

Output IS_EPIC and EPIC_PATH for downstream skills (work-end,
handover, work-pause) that don't call work_router.py.

Refs #100
```

---

### Task 4: Add `.epic` to branch_cleanup.py scaffold removal

**Files:**
- Modify: `work-end/branch_cleanup.py`
- Create: `tests/test_branch_cleanup.py` (if no tests exist for
  cleanup-scaffold)

**Interfaces:**
- Consumes: `cleanup_scaffold(workspace, params)` existing function
- Produces: `.epic` removed alongside `.meta` and `JOURNAL.md`

- [ ] **Step 1: Write failing test**

```python
class TestCleanupScaffold:
    def test_removes_epic_file(self, tmp_path):
        ws = tmp_path / "workspace"
        ws.mkdir()
        subprocess.run(["git", "init"], cwd=str(ws), capture_output=True)
        subprocess.run(["git", "-C", str(ws), "config", "user.name", "T"],
                       capture_output=True)
        subprocess.run(["git", "-C", str(ws), "config", "user.email",
                       "t@t.com"], capture_output=True)
        design = ws / "design"
        design.mkdir()
        (design / ".meta").write_text("branch: test\n")
        (design / ".epic").write_text("## Issue\nrepo#100\nType: epic\n")
        (design / "JOURNAL.md").write_text("# Journal\n")
        subprocess.run(["git", "-C", str(ws), "add", "."],
                       capture_output=True)
        subprocess.run(["git", "-C", str(ws), "commit", "-m", "scaffold"],
                       capture_output=True)
        result = cleanup_scaffold(str(ws), {})
        assert not (design / ".meta").exists()
        assert not (design / ".epic").exists()
        assert not (design / "JOURNAL.md").exists()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_branch_cleanup.py::TestCleanupScaffold -v`
Expected: FAIL — `.epic` still exists after cleanup

- [ ] **Step 3: Implement**

In `branch_cleanup.py`, `cleanup_scaffold()` function, add `.epic` to
the removal list (around line 130):

```python
epic_path = ws / "design" / ".epic"

if meta_path.exists():
    files_to_remove.append("design/.meta")
if journal_path.exists():
    files_to_remove.append("design/JOURNAL.md")
if epic_path.exists():
    files_to_remove.append("design/.epic")
```

- [ ] **Step 4: Run test**

Run: `pytest tests/test_branch_cleanup.py::TestCleanupScaffold -v`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#100): add .epic to branch_cleanup.py scaffold removal

cleanup-scaffold now removes .epic alongside .meta and JOURNAL.md
after rebase onto main. The file survives on the epic branch for audit.

Refs #100
```

---

### Task 5: Update skill MDs — work router, work-start, work-end, work-slot, handover

**Files:**
- Modify: `work/SKILL.md`
- Modify: `work-start/SKILL.md`
- Modify: `work-end/SKILL.md`
- Modify: `work-slot/SKILL.md`
- Modify: `handover/SKILL.md`
- Modify: `handover/handover-reference.md`

No tests — these are instruction files for Claude, not executable code.

- [ ] **Step 1: Update work/SKILL.md**

a) Add `work epic` and `work next` to Step 1 routing table:

```markdown
| `work epic #N` | → epic setup (Step 5) |
| `work next` | → advance epic issue (Step 6) |
```

b) Fix Step 4 option 1 — #103:

Replace:
```
Always present:
> 1. **resume** — read the last handover and continue where I left off
```

With:
```
If `HAS_HANDOFF=yes`:
> 1. **resume** — read the last handover and continue where I left off
If `HAS_HANDOFF=no`:
> 1. **start** — invoke work-start (first session on this branch)
```

c) Add epic option to Step 4:

```
If `IS_EPIC=yes`:
> N. **next** — mark current child issue done, advance to next
```

d) Add Step 5 — `work epic #N`:

Document the full flow from the spec: fetch children, batch planning,
branch creation (3-case: new / stamped-closed / active-error), scaffold,
write .epic, report.

e) Add Step 6 — `work next`:

Document: call `epic_manager.py advance`, check off GitHub checkbox,
batch boundary handling, epic completion (add epic to COVERS), report.

f) Update "On resume (option 1)" section:

Add `.epic` detection alongside `.slot`:
```
If `IS_EPIC=yes`: read .epic at `$EPIC_PATH` (or .slot at
`$SLOT_PATH` if in slot) for batch progress and active issue.
```

- [ ] **Step 2: Update work-start/SKILL.md**

Generalise the Epic Slot Overlay trigger. Replace:
```
1. Guard: Checks if $PROJECT path contains /worktrees/. If not, skips.
2. Detect: Read $PROJECT/../.slot.
```

With:
```
1. Guard — detect epic file:
   If /worktrees/ in $PROJECT: epic_file = $PROJECT/../.slot
   Elif workspace/design/.epic exists: epic_file = workspace/design/.epic
   Else: skip overlay

2. Detect: Read epic_file. If missing or no Type: epic in ## Issue, skip.
```

- [ ] **Step 3: Update work-end/SKILL.md**

Add epic-aware closure rules:
- When `.epic` exists and all batches complete → epic issue already in
  COVERS (added by `work next` step 4), close alongside children
- Mid-epic exit → only close completed children
- Add `.epic` to 8j-cleanup scaffold list reference

- [ ] **Step 4: Update work-slot/SKILL.md**

Update `work-slot next` (Step 6 / advance command) to pass explicit
path to `epic_manager.py`:

```bash
python3 ~/.claude/skills/work-slot/epic_manager.py advance \
  <slot-dir>/.slot
```

(Previously passed `<slot-dir>` alone — now passes the full `.slot`
path.)

- [ ] **Step 5: Update handover/SKILL.md and handover-reference.md**

In SKILL.md Step 2 (epic slot context), generalise detection:
```
If $PROJECT contains /worktrees/ and $PROJECT/../.slot exists with
Type: epic, OR if workspace/design/.epic exists with Type: epic:
```

In handover-reference.md, update the Epic Progress section guard:
```
<Include ONLY when IS_EPIC=yes (from ctx.py output).
 Read from .epic or .slot depending on context.>
```

- [ ] **Step 6: Commit**

```
feat(#100,#103): add work epic/next verbs, fix start vs resume

work/SKILL.md: work epic #N entry point, work next iteration,
Step 4 shows start (not resume) when HAS_HANDOFF=no (#103).
work-start: epic overlay detects .epic alongside .slot.
work-end: epic-aware closure, .epic in cleanup list.
work-slot: pass explicit .slot path to epic_manager.py.
handover: detect .epic for Epic Progress section.

Refs #100, Closes #103
```

---

### Task 6: Sync, verify, and close

- [ ] **Step 1: Run full test suite**

Run: `pytest tests/test_epic_manager.py tests/test_work_router.py tests/test_ctx.py tests/test_branch_cleanup.py -v`
Expected: ALL PASS

- [ ] **Step 2: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: PASS

- [ ] **Step 3: Sync installed skills**

Run: `python3 scripts/claude-skill sync-local --all -y`

- [ ] **Step 4: Commit any stragglers**

If validators flagged anything, fix and commit.

- [ ] **Step 5: Close #102**

```bash
gh issue close 102 --repo Hortora/soredium --comment "Landed as 9b82887"
```

(#102 was the SLOT.md → .slot rename, already committed.)

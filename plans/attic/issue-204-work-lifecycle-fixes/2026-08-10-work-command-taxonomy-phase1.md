# Work Command Taxonomy Phase 1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #202 — Work command taxonomy: separate continue/resume/brief verbs
**Issue group:** #202

**Goal:** Replace the overloaded "resume"/"start" options in the work
lifecycle menu with three distinct verbs: `continue` (keep working),
`resume` (pause-stack only), and wrong-context error handling.

**Architecture:** Changes are in the skill presentation layer
(`work/SKILL.md`, `work-resume/SKILL.md`, `handover/SKILL.md`) and two
Python files (`lifecycle.py` for the `work_continue` self-transition,
`work_router.py` for the handoff substring fix, `work_health.py` for
`check_plan_state` in entry scope). The lifecycle state machine's
existing transitions are unchanged — only a new self-transition is added.

**Tech Stack:** Python 3, pytest, SKILL.md (markdown)

## Global Constraints

- No AI attribution in commit messages
- Every commit references #202 (`Refs #202`)
- Run `python3 scripts/claude-skill sync-local --all -y` after skill edits
- Follow docs/development/skill-validation.md for SKILL.md changes
- Follow docs/development/readme-sync.md for skill collection changes

---

### Task 1: Add `work_continue` self-transition to lifecycle state machine

**Files:**
- Modify: `project/lifecycle.py`
- Test: `tests/test_lifecycle.py`

**Interfaces:**
- Produces: `('active', 'work_continue'): ('active', [], [])` transition
- Produces: `INVALID_MESSAGES` entries for `work_continue` in wrong states

- [ ] **Step 1: Write failing tests for the new transition**

Add to `tests/test_lifecycle.py`:

```python
class TestWorkContinueTransition:
    def test_active_work_continue_is_self_transition(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-1-test\nstate: active\n")
        result = transition(meta, 'work_continue')
        assert result.from_state == 'active'
        assert result.new_state == 'active'
        assert result.effects == []
        assert result.post_commit_effects == []

    def test_idle_work_continue_raises(self, tmp_path):
        meta = tmp_path / ".meta"
        with pytest.raises(InvalidTransition) as exc_info:
            transition(meta, 'work_continue')
        assert 'continue' in str(exc_info.value).lower()

    def test_scaffolded_work_continue_raises(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-1-test\nstate: scaffolded\n")
        with pytest.raises(InvalidTransition):
            transition(meta, 'work_continue')

    def test_paused_work_continue_raises(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-1-test\nstate: paused\n")
        with pytest.raises(InvalidTransition) as exc_info:
            transition(meta, 'work_continue')
        assert 'resume' in str(exc_info.value).lower()

    def test_transitioning_work_continue_raises(self, tmp_path):
        meta = tmp_path / ".meta"
        meta.write_text("branch: issue-1-test\nstate: transitioning\n")
        with pytest.raises(InvalidTransition):
            transition(meta, 'work_continue')

    def test_can_transition_active_work_continue(self):
        assert can_transition('active', 'work_continue') is True

    def test_cannot_transition_idle_work_continue(self):
        assert can_transition('idle', 'work_continue') is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_lifecycle.py::TestWorkContinueTransition -v`
Expected: FAIL — `work_continue` not in transition table

- [ ] **Step 3: Add the transition and invalid messages**

In `project/lifecycle.py`, add to `TRANSITION_TABLE`:

```python
('active', 'work_continue'):      ('active',           [],                                                                    []),
```

Add to `INVALID_MESSAGES`:

```python
('idle', 'work_continue'):       "Cannot continue — no active branch. Use `work` to start new work.",
('scaffolded', 'work_continue'): "Cannot continue — branch not yet active. Context setup must complete first.",
('transitioning', 'work_continue'): "Cannot continue — issue transition in progress. Wait for context refresh.",
('paused', 'work_continue'):     "Branch is paused. Use `work resume` to restore it from the pause stack first.",
```

Update existing message for `('active', 'work')`:

```python
('active', 'work'):         "Already on an active branch. Use `work continue`, `work end`, `work pause`, or `work next`.",
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_lifecycle.py::TestWorkContinueTransition -v`
Expected: PASS

- [ ] **Step 5: Run full lifecycle test suite**

Run: `python3 -m pytest tests/test_lifecycle.py -v`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/lifecycle.py tests/test_lifecycle.py
git commit -m "feat(#202): add work_continue self-transition to lifecycle state machine

Refs #202"
```

---

### Task 2: Fix `_handoff_references_branch()` substring collision

**Files:**
- Modify: `work/work_router.py`
- Test: `tests/test_work_router.py`

**Interfaces:**
- Consumes: nothing new
- Produces: word-boundary matching for issue numbers in HANDOFF.md

- [ ] **Step 1: Write failing test for substring collision**

Add to `tests/test_work_router.py`:

```python
class TestHandoffSubstringCollision:
    def test_issue_42_does_not_match_421(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        subprocess.run(["git", "-C", str(workspace), "init"], capture_output=True, check=True)
        subprocess.run(["git", "-C", str(workspace), "config", "user.email", "t@t"], capture_output=True)
        subprocess.run(["git", "-C", str(workspace), "config", "user.name", "T"], capture_output=True)
        handoff = workspace / "HANDOFF.md"
        handoff.write_text("## Last Session\nWorked on #421 feature.\n")
        subprocess.run(["git", "-C", str(workspace), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(workspace), "commit", "-m", "init"], capture_output=True)

        from work.work_router import _handoff_references_branch
        assert _handoff_references_branch(workspace, "issue-42-test") is False

    def test_issue_42_matches_42(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        subprocess.run(["git", "-C", str(workspace), "init"], capture_output=True, check=True)
        subprocess.run(["git", "-C", str(workspace), "config", "user.email", "t@t"], capture_output=True)
        subprocess.run(["git", "-C", str(workspace), "config", "user.name", "T"], capture_output=True)
        handoff = workspace / "HANDOFF.md"
        handoff.write_text("## Last Session\nWorked on #42 feature.\n")
        subprocess.run(["git", "-C", str(workspace), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(workspace), "commit", "-m", "init"], capture_output=True)

        from work.work_router import _handoff_references_branch
        assert _handoff_references_branch(workspace, "issue-42-test") is True
```

- [ ] **Step 2: Run tests to verify the substring test fails**

Run: `python3 -m pytest tests/test_work_router.py::TestHandoffSubstringCollision -v`
Expected: `test_issue_42_does_not_match_421` FAILS (substring match returns True)

- [ ] **Step 3: Fix the matching to use word boundary**

In `work/work_router.py`, change line 45:

```python
# Before:
return f"#{issue_num}" in result.stdout

# After:
return bool(re.search(rf'#{issue_num}\b', result.stdout))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_router.py::TestHandoffSubstringCollision -v`
Expected: PASS

- [ ] **Step 5: Run full router test suite**

Run: `python3 -m pytest tests/test_work_router.py -v`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/work_router.py tests/test_work_router.py
git commit -m "fix(#202): word-boundary match for handoff issue references

Prevents #42 matching #421 in HANDOFF.md.

Refs #202"
```

---

### Task 3: Add `check_plan_state` to entry scope in `work_health.py`

**Files:**
- Modify: `project/work_health.py`
- Test: `tests/test_work_health.py`

**Interfaces:**
- Consumes: `check_plan_state(project, workspace, owner_repo)` — already exists
- Produces: `run_checks()` accepts `owner_repo` kwarg; `ENTRY_CHECKS` conditionally includes `check_plan_state`

- [ ] **Step 1: Write failing test for owner_repo in entry scope**

Add to `tests/test_work_health.py`:

```python
class TestEntryChecksWithOwnerRepo:
    def test_entry_checks_include_plan_state_when_owner_repo_provided(self, tmp_path, capsys):
        project = tmp_path / "project"
        workspace = tmp_path / "workspace"
        project.mkdir()
        workspace.mkdir()
        subprocess.run(["git", "-C", str(project), "init"], capture_output=True, check=True)
        subprocess.run(["git", "-C", str(workspace), "init"], capture_output=True, check=True)

        run_checks("entry", str(project), str(workspace), owner_repo="Hortora/soredium")
        captured = capsys.readouterr()
        assert "CHECK=plan_state" in captured.out

    def test_entry_checks_exclude_plan_state_without_owner_repo(self, tmp_path, capsys):
        project = tmp_path / "project"
        workspace = tmp_path / "workspace"
        project.mkdir()
        workspace.mkdir()
        subprocess.run(["git", "-C", str(project), "init"], capture_output=True, check=True)
        subprocess.run(["git", "-C", str(workspace), "init"], capture_output=True, check=True)

        run_checks("entry", str(project), str(workspace))
        captured = capsys.readouterr()
        assert "CHECK=plan_state" not in captured.out
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_health.py::TestEntryChecksWithOwnerRepo -v`
Expected: FAIL — `run_checks()` doesn't accept `owner_repo`

- [ ] **Step 3: Update `run_checks()` to accept `owner_repo`**

In `project/work_health.py`, modify `run_checks()`:

```python
def run_checks(scope, project, workspace, branch=None, owner_repo=None):
    if scope == "entry":
        checks = list(ENTRY_CHECKS)
        if owner_repo:
            checks.append(lambda p, w: check_plan_state(p, w, owner_repo))
    elif scope == "wrap":
        checks = list(WRAP_CHECKS)
        if owner_repo:
            checks.append(lambda p, w: check_plan_state(p, w, owner_repo))
    # ... rest unchanged
```

Update `main()` argparser:

```python
parser.add_argument("--owner-repo", default=None)
# ...
run_checks(args.scope, args.project, args.workspace, args.branch, owner_repo=args.owner_repo)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_health.py::TestEntryChecksWithOwnerRepo -v`
Expected: PASS

- [ ] **Step 5: Run full work_health test suite**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add project/work_health.py tests/test_work_health.py
git commit -m "feat(#202): add check_plan_state to entry scope when owner_repo provided

Enables done-detection in continue flow by syncing .plan with GitHub
at entry time.

Refs #202"
```

---

### Task 4: Update `work/SKILL.md` — routing table, Step 4 menu, continue behavior

**Files:**
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: `work_continue` transition from Task 1, `work_health.py --owner-repo` from Task 3
- Produces: Updated routing table, Step 4 menu, continue behavioral spec, wrong-context error handling

- [ ] **Step 1: Update CSO description (frontmatter)**

Replace lines 3-9:

```yaml
description: >
  Use when the user says "work", "work end", "work pause", "work resume",
  "work continue", or "work next" — detects current branch state and routes
  to the correct work lifecycle skill automatically. "work" alone starts new
  work or shows the pause stack. "work end" closes the branch. "work pause"
  saves state. "work continue" keeps working on the current branch.
  "work resume" restores a paused branch from the stack.
  "work next" advances to the next issue in the .plan queue.
```

- [ ] **Step 2: Update the routing table (Step 1)**

Replace the routing table:

```markdown
| Invocation | Route to |
|------------|---------|
| `work end` | → **work-end** immediately (no router needed) |
| `work pause` | → **work-pause** immediately (no router needed) |
| `work next` | → advance to next issue in `.plan` queue (Step 5) |
| `work resume` / `resume` | → **work-resume** (pause-stack only; error if on active branch — see Step 1c) |
| `work continue` / `continue` | → run router → Step 4 `continue` action directly (error if on main — see Step 1c) |
| `work` / `work start` | → run router (Step 1b) |
| `resume handover` | → handover skill directly (manual invocation) |
```

- [ ] **Step 3: Add Step 1c — Wrong-context error handling**

Add after Step 1b:

```markdown
**Step 1c — Wrong-context error handling (D4)**

Before dispatching to a sub-skill or executing an internal action,
check for wrong-context invocations:

| Invocation | Condition | Action |
|------------|-----------|--------|
| `work resume` | `ON_MAIN=no` (on feature branch, not paused) | Error: "Not paused — use `continue` to keep working, or `work pause` first." |
| `work resume` | `ON_MAIN=yes` + `STACK_DEPTH=0` | Error: "Nothing to resume — pause stack is empty. Use `work` to start new work." |
| `work continue` | `ON_MAIN=yes` + `STACK_DEPTH=0` | Error: "No active branch — use `work` to start new work." |
| `work continue` | `ON_MAIN=yes` + `STACK_DEPTH>0` | Error: "No active branch — use `work` to start new work or `work resume` to return to a paused branch." |
| `work continue` | `ROUTE=workspace_dirty` | Error: "Workspace is on a stale branch — run `work` to clean up." |
| `work start` | `ROUTE=resume_branch` | Redirect → `continue` + note: "Already on `<branch>` — continuing." |
```

- [ ] **Step 4: Replace Step 4 menu**

Replace the entire Step 4 section with the spec's updated menu:

```markdown
**Step 4 — On feature branch: contextual options**

Present options based on the router output:

> 1. **continue** — keep working (loads context automatically)

If `STACK_DEPTH > 0`:
> 2. **switch** — you have <N> paused branch(es) — restore one from stack

If `HAS_PLAN=yes`:
> N. **next** — mark current issue done, advance to next in queue

Always present:
> N+1. **end** — close this branch, merge, push, return to main

If `HAS_PLAN=yes` and queue has remaining items, annotate the end option:
> N+1. **end** — ⚠️ queue has N remaining issues — close this branch, merge, push, return to main

> N+2. **pause** — commit WIP, push to stack, switch to main
> N+3. **wrap** — end session but keep branch open (write handover)
```

- [ ] **Step 5: Replace the `continue` behavioral specification**

Replace the "On resume" and "On start" blocks with the spec's unified
`continue` behavioral specification (lines 362-404 of the spec). This
includes:

- Lifecycle: fire `transition(meta_path, 'work_continue')`
- `HAS_HANDOFF=yes` path (subsequent session): read HANDOFF, health sync, plan/slot context, specs, done-detection
- `HAS_HANDOFF=no` path (first session): work-start Steps 0,2,3,3b,3c,11, health sync, plan/slot, done-detection
- Done-detection auto-suggest (D3): check `PLAN_ACTIVE_ISSUE` after health sync
- `HAS_HANDOFF` path selection dependency note

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work/SKILL.md
git commit -m "feat(#202): update work router — continue/resume taxonomy, wrong-context errors

Replaces resume/start distinction with unified 'continue' option.
Adds wrong-context error handling for resume on active branch,
continue on main, and start-when-already-started redirect.

Refs #202"
```

---

### Task 5: Update `work-resume/SKILL.md` CSO description

**Files:**
- Modify: `work-resume/SKILL.md`

**Interfaces:**
- Consumes: nothing
- Produces: clarified CSO — pause-stack only

- [ ] **Step 1: Update the frontmatter description**

Replace lines 3-6:

```yaml
description: >
  Use when returning to a paused branch from the pause stack — user says
  "work-resume", "resume", or "go back to that branch". Pause-stack
  restoration only, not general branch continuation (use "work continue"
  for that). Invoked from main to restore a previously paused work session.
  Handles multiple paused branches via stack.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-resume/SKILL.md
git commit -m "docs(#202): clarify work-resume CSO — pause-stack only

Refs #202"
```

---

### Task 6: Update `handover/SKILL.md` resume section

**Files:**
- Modify: `handover/SKILL.md`

**Interfaces:**
- Consumes: nothing
- Produces: updated resume section noting `continue` subsumes automatic reading

- [ ] **Step 1: Update the "When to Use" table**

Replace lines 19-24:

```markdown
| Situation | Skill |
|-----------|-------|
| Branch is **done** — closing, merging, pushing | `work-end` (includes full wrap + HANDOFF.md) |
| Branch is **not done** — pausing mid-work, ending session | `handover` (this skill) |
| **Continuing** work on a branch | `work continue` (auto-reads HANDOFF.md) |
| **Interrogating** the handover document directly | `resume handover` (this skill, resume path) |
```

- [ ] **Step 2: Add note at the top of "Resuming a Handover" section**

Add after line 30 ("When the user says..."):

```markdown
> **Note:** `work continue` now auto-reads HANDOFF.md as part of its
> context loading. The explicit `resume handover` invocation below is
> for when you want to interrogate the handover document directly —
> ask questions about it, review past sessions, etc. — rather than
> simply continuing work.
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add handover/SKILL.md
git commit -m "docs(#202): update handover — continue auto-reads HANDOFF.md

resume handover remains for explicit interrogation of the handover
document.

Refs #202"
```

---

### Task 7: Sync skills and run validation

**Files:**
- No new files — validation only

- [ ] **Step 1: Sync all skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 2: Run commit-tier validation**

```bash
python3 scripts/validate_all.py --tier commit
```

Expected: 0 CRITICAL findings

- [ ] **Step 3: Run full test suite for modified files**

```bash
python3 -m pytest tests/test_lifecycle.py tests/test_work_router.py tests/test_work_health.py -v
```

Expected: All pass

- [ ] **Step 4: Follow readme-sync workflow**

Read `docs/development/readme-sync.md` and check if README.md needs
updating for the new `continue` command in the skill chaining table
or the work lifecycle description.

- [ ] **Step 5: Commit any readme/doc updates**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add README.md  # if changed
git commit -m "docs(#202): sync README for work command taxonomy

Refs #202"
```

# Unified Work Queue — Design Spec

**Issue:** #261
**Date:** 2026-08-20
**Branch:** issue-261-unified-work-queue

## Problem

The work lifecycle has two separate mechanisms for "what's next":
- **On branches:** `.plan` queue with `← active` markers, `work next` advances
- **On main:** Ad-hoc what-next via enrichment.py/HANDOFF.md, funneling into `work start`

Commands error when invoked in the "wrong" context instead of routing to the right behavior. Work can be accidentally skipped or left unfinished. The LLM makes mechanical routing decisions that should be deterministic.

## Solution

### 1. Bidirectional chaining

Four commands form a chain. Each chains forward (nothing to do) and guards backward (not ready):

```
continue ←→ next ←→ end ←→ find
    →  "nothing to do, cascade forward"
    ←  "not ready, go back"
```

| Command | Responsibility | Forward chain | Backward guard |
|---------|---------------|---------------|----------------|
| `continue` | Keep working on active issue | Active issue done → `next` | — (entry point) |
| `next` | Advance queue to next issue | Queue empty → `end` | Active issue still open → `continue` |
| `end` | Close ceremony (issues, docs, blog) | Ceremony done → `find` | Queue not empty → `next` |
| `find` | Discover and populate queue | — (terminal) | Unfinished work → `next` |

### 2. `.plan` on main

`.plan` files can exist on any branch including main. Same format, same queue semantics. The workspace main branch hosts the main `.plan`.

On main, the `.plan` tracks a queue of work items. Each `work next` on main creates a feature branch via `work start` for the next item (unless the item is small enough for main — textual guidance, not a gate).

### 3. Chaining logic in Python

A new module `work_router.py` implements all chaining decisions. It outputs directives the LLM follows:

```python
# Output format
DIRECTIVE=continue          # proceed with continue
DIRECTIVE=chain_to_next     # nothing to continue, chain forward
DIRECTIVE=guard_continue    # next called but issue unfinished, go back
DIRECTIVE=chain_to_end      # queue empty, chain forward
DIRECTIVE=guard_next        # end called but queue not empty, go back
DIRECTIVE=chain_to_find     # ceremony done, chain forward
DIRECTIVE=guard_next        # find called but unfinished work, go back
DIRECTIVE=proceed           # command can proceed normally

# Additional context
REASON=active_issue_done
ACTIVE_ISSUE=42
ISSUE_STATE=closed
QUEUE_REMAINING=3
```

The script checks:
- GitHub issue state (`gh issue view --json state`)
- Queue state (plan_manager.detect)
- .plan existence and META_STATE
- Uncommitted changes
- Branch context (main vs feature)

### 4. New `ended` state

Add to lifecycle state machine:

```python
VALID_STATES = frozenset({
    'idle', 'scaffolded', 'active', 'transitioning', 'paused', 'ended',
    'closing:review', 'closing:verified', 'closing:promoted',
    'closing:pushed', 'closing:merged', 'closing:stamped',
})
```

New transitions:

| From | Event | To | Effects |
|------|-------|----|---------|
| `closing:stamped` | `cleanup_pass` | `ended` | `write_plan_ended` (on main) |
| `closing:stamped` | `cleanup_pass` | `idle` | `write_plan_closed` (on branch — unchanged) |
| `ended` | `work_find` | `active` | `queue_populated` |
| `ended` | `work_continue` | `ended` | — (self-transition, chains to next) |
| `ended` | `work_next` | `ended` | — (self-transition, chains to end) |

The `cleanup_pass` transition is context-sensitive: on a branch it goes to `idle` (branch is done), on main it goes to `ended` (.plan persists).

### 5. work-end on main

work-end becomes context-aware. On main, it skips branch-specific steps:

| Step | Branch | Main |
|------|--------|------|
| 1. Context | Check branch alignment | Check .plan exists |
| 2. Review | Diff against base branch | Diff since last `ended` marker or tag |
| 4.2 Rebase | Rebase onto base | Skip |
| 4.3 Squash | Classify branch commits | Skip (or squash recent main commits) |
| 4.4 Land | Push, stamp, merge | Push only |
| 5. Verify | Check merged + stamped | Check pushed |
| 6.2 Checkout main | Switch to main | Skip (already there) |
| 6.2b Cleanup | Remove .plan | Keep .plan (set state to `ended`) |

Steps that are identical on both: close issues (5b), sweep (3), promote artifacts (4.1).

Textual guidance when starting work on main: "Consider a feature branch for non-trivial work (`work start #N`). `quick-fix` for small changes." Never blocks.

### 6. `work find`

New verb in the routing table. Runs the enrichment/what-next pipeline (currently Step 2a of `work`). Appends results to `.plan` via `plan_manager append`.

```bash
python3 scripts/enrichment.py refresh --repo $OWNER_REPO
python3 scripts/enrichment.py what-next --repo $OWNER_REPO --mode general --limit 5
```

If no `.plan` exists, creates one. If `.plan` exists with `ended` state, transitions to `active` after population.

### 7. Corruption recovery

New function in `work_router.py` or a dedicated `state_triage.py`. Runs before any command dispatches. Checks state consistency and outputs recovery directives.

**Corruption scenarios and resolutions:**

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| `.plan` exists, `state:` missing | `read_state()` returns default `active` | Write explicit `state: active` |
| `.plan` exists, `state:` invalid | `CorruptedState` exception | Infer from context: branch exists? issues open? → propose state |
| `state: active` but all issues closed | Check GitHub state for ACTIVE_ISSUE | Resolve to `ended`, suggest `work find` |
| `state: closing:*` but not in close sequence | Mid-ceremony crash | Offer: resume from this gate, or abort to `active` |
| `.plan` branch doesn't match current branch | BRANCH_MISMATCH from ctx.py | Offer: switch to .plan's branch, or reset .plan |
| `state: active` but branch doesn't exist | git branch --list returns empty | Offer: create branch, or reset to `idle` |
| Stale `.plan` on main after work-end | state is `closing:stamped` on main | Auto-complete cleanup to `ended` |
| Queue markers inconsistent with GitHub | plan_manager.detect vs gh issue state | Sync queue: check off closed issues, report changes |

Output format:
```
TRIAGE=stale_closing_state
CURRENT_STATE=closing:stamped
SUGGESTED_STATE=ended
REASON=issue_258_closed_branch_merged
ACTION=auto_resolve
# or ACTION=user_confirm when ambiguous
```

The LLM presents the resolution to the user and executes the suggested action.

## Implementation plan

### Phase 1: Python infrastructure

1. **`work_router.py`** — new module in `project/`
   - `evaluate_continue(ctx)` → directive
   - `evaluate_next(ctx)` → directive  
   - `evaluate_end(ctx)` → directive
   - `evaluate_find(ctx)` → directive
   - `triage_state(ctx)` → triage result
   - All functions take ctx.py output dict as input
   - All functions return structured output (directive + reason + context)

2. **`lifecycle.py` changes**
   - Add `ended` to `VALID_STATES`
   - Add new transitions to `TRANSITION_TABLE`
   - Context-sensitive `cleanup_pass` (branch → `idle`, main → `ended`)

3. **`ctx.py` changes**
   - Pass `BASE_BRANCH` to work_state.detect() (fix hardcoded "main" comparison)
   - Output `CHAIN_DIRECTIVE` from work_router.evaluate_* based on invoked command

### Phase 2: Skill updates

4. **`work/SKILL.md`** — routing table
   - Add `work find` to routing table
   - Replace LLM-driven chaining with "read CHAIN_DIRECTIVE, follow it"
   - Remove Step 1c errors for continue-on-main (replaced by chaining)
   - Extract Step 2a into `work find` verb

5. **`work-end/SKILL.md`** — main support
   - Remove "must be on branch" precondition
   - Add conditional logic for main-mode (skip merge/stamp/rebase/squash)
   - Change scaffold cleanup to set `ended` state on main

6. **CSO updates** — update skill descriptions to reflect new verbs

### Phase 3: Tests

7. **`test_work_router.py`** — comprehensive tests for all chaining paths
   - Forward chains: continue→next, next→end, end→find
   - Backward guards: next←continue, end←next, find←next
   - Edge cases: no .plan, empty queue, all issues closed, mid-close state
   - Corruption scenarios

8. **`test_lifecycle.py` updates** — new state transitions
   - `ended` state transitions
   - Context-sensitive `cleanup_pass`
   - `work_find` event

## Files changed

| File | Change type | Description |
|------|------------|-------------|
| `project/work_router.py` | New | Bidirectional chaining + corruption triage |
| `project/lifecycle.py` | Modify | Add `ended` state, new transitions |
| `project/ctx.py` | Modify | Fix hardcoded "main", output CHAIN_DIRECTIVE |
| `project/work_state.py` | Modify | Use BASE_BRANCH instead of hardcoded "main" |
| `work/SKILL.md` | Modify | New routing table, `work find`, chaining directives |
| `work-end/SKILL.md` | Modify | Remove branch-only guard, main-mode conditionals |
| `tests/test_work_router.py` | New | Chaining and triage tests |
| `tests/test_lifecycle.py` | Modify | New state transition tests |

## References

- lifecycle.py — state machine, transition table, validate_state()
- ctx.py / work_state.py — routing logic, ON_MAIN detection
- plan_manager.py — detect(), advance(), queue management
- work/SKILL.md — current routing table, Step 1c errors, Step 2a what-next, Step 4 continue, Step 5 work next
- work-end/SKILL.md — 12 branch-specific touchpoints identified in audit
- ADR-0013 — unified .plan file design
- docs/protocols/evidence-before-claims.md — verification before completion
- docs/protocols/externalised-scripts-require-tests.md — tests for new scripts

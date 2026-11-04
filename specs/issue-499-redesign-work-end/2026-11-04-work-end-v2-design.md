# Work-End v2: Atomic Close Ceremony

**Date:** 2026-11-04
**Issue:** Hortora/soredium#499
**Status:** Draft
**Supersedes:** Current work-end orchestrator (work_end_orchestrator.py, 2,250 LOC)

## Problem Statement

Work-end has accumulated 80 issues in 70 days (>1/day). The system cannot be
stabilised by patching — structural problems guarantee new bugs appear as fast
as old ones close.

**Root causes:**

1. **Dual-purpose `.plan`** — simultaneously an issue queue and lifecycle state
   container. 7 independent deletion paths, only 3 guard the queue. Every
   plan-wipe bug (#358, #364, #442, #466, #485, #496, #498) is this coupling.

2. **Combinatorial pipeline** — 53 steps × 5 modes × 5 repo dispositions.
   Each fix touches one execution path and can break others silently.

3. **Scattered state** — `.close-progress`, `.plan state:`, git HEAD, and LLM
   context can all disagree. No consistency protocol, no transaction boundary.

4. **Bolt-on multi-repo** — mutable `OrchestratorContext` overridden per repo
   iteration. Every step must use `getattr(ctx, "current_repo_project", None)
   or ctx.project` — miss one and it's a bug (14 multi-repo issues).

5. **Recovery creates failure** — reconciliation resets progress, phase-skip
   fills gaps, lifecycle transitions fail on concurrent modification. Each
   recovery mechanism creates new failure modes (13 loop/stall bugs).

## Design Decisions

Six decisions captured in `decisions.md`:

| # | Decision | Key consequence |
|---|----------|----------------|
| D1 | Strip to essential close ceremony | 53 steps → 6 steps |
| D2 | Separate queue from ceremony state | .plan split into .queue + ephemeral .close-state |
| D3 | One ceremony, different inputs | 5 modes → resolve phase produces CloseTarget list |
| D4 | Fully mechanical execution | No LLM loop, no progress file, no reconciliation |
| D5 | Eliminate most corruption triage | 620 lines → ~50 lines of git-level pre-flight |
| D6 | Replace state machine with status field | 776 lines, 12 states → 4 status values |

## Architecture

### Three-Phase Model

```
PRE-CLOSE (LLM-driven, independent, writes to git)
    │
    │  git committed state only
    ▼
CLOSE CEREMONY (single Python invocation, mechanical)
    │
    │  queue updated, repos on main
    ▼
POST-CLOSE (reads committed state, failures non-fatal)
```

Each phase communicates with the next only through committed git state. No
shared mutable files, no progress tracking, no reconciliation.

**Failure isolation guarantee:** Pre-close failures do not corrupt ceremony
state. Ceremony failures do not destroy the queue. Post-close failures do
not invalidate the landed code.

### Pre-Close Phase

The LLM runs judgment work through normal skill invocations — not through
an orchestrator pipeline. Each item writes to git independently:

1. **Code review** — `code-review` / `branch-audit` skill
2. **Squash plan** — decide on squash commit message and strategy (the actual
   squash happens in the ceremony's `land` step via `squash_merge`)
3. **Content creation** — `write-content` skill for blog/diary
4. **Artifact promotion** — copy specs, ADRs from workspace to project repo
5. **Uncommitted changes** — commit or stash outstanding work

These are standard skill invocations using existing infrastructure. They
are not orchestrator steps. They do not share mutable state with each other
or with the close ceremony.

The work-end SKILL.md presents a pre-close checklist. The user (or the LLM
following the checklist) completes items and confirms readiness. **Pre-close
is advisory, not enforced** — the user can skip it and run the ceremony
directly. The ceremony reads only committed git state; anything not committed
before ceremony start is not included. The risk of skipping pre-close is
landing without review or blog, not data corruption.

**What disappears:**
- `forcing_function` step — pre-close items don't block the ceremony
- `loose_ends_sweep` — the pre-close checklist replaces it
- `verify_recover` — nothing to recover from
- Handler `.md` files — judgment stays in the LLM via skills, not dispatched handlers

### Close Ceremony

Single Python entry point: `python3 work-end/close.py <workspace>`

No arguments for mode, repo, evidence, or state. Reads everything from git
and `.queue`.

#### Step 1: Resolve

Determine what to close. Read `.queue` for the active item and branch name.
Discover repos and classify which have the branch with commits.

```python
@dataclass(frozen=True)
class CloseTarget:
    repo: Path           # the repo to land
    branch: str          # the feature branch name
    transport: str       # "direct" or "two-hop" (slot clones use canonical as relay)
    canonical: Path | None  # canonical repo path (two-hop only)
    is_workspace: bool   # workspace repos have design artifacts, not source code

def resolve(workspace: Path) -> list[CloseTarget]:
    queue = read_queue(workspace)
    branch = queue.branch
    targets = []
    # Use slot topology if in a slot, otherwise project + workspace
    for repo, transport, canonical in discover_repo_topology(workspace):
        if branch_exists(repo, branch) and has_commits(repo, branch):
            is_ws = (repo == workspace)
            targets.append(CloseTarget(
                repo=repo, branch=branch,
                transport=transport, canonical=canonical,
                is_workspace=is_ws))
    return targets
```

`discover_repo_topology()` reads `.slot` files for slot topology (clone →
canonical → remote) or falls back to project + workspace for non-slot branches.
Replaces both `classify_repos()` and the slot-specific repo discovery that
currently lives across `repo_state.py`, `slot_io.py`, and `land_flow.py`.

A slot with 3 repos produces 3 targets. A single-repo branch produces 1 (or 2
with workspace). The ceremony doesn't know or care about the difference —
`transport` tells `land()` how to push.

#### Step 2: Pre-flight Checks

Verify preconditions before mutating anything:

```python
def preflight(targets: list[CloseTarget]) -> list[str]:
    errors = []
    for t in targets:
        if is_dirty(t.repo):
            errors.append(f"{t.repo.name}: uncommitted changes")
        if not branch_exists(t.repo, "main"):
            errors.append(f"{t.repo.name}: no main branch")
        if not remote_reachable(t.repo):
            errors.append(f"{t.repo.name}: remote not reachable")
    return errors
```

Replaces: most of `corruption.py` (620 lines, 17+ scenarios). Pre-flight
checks refuse to run rather than trying to recover after the fact.

#### Step 3: Land (per target)

Squash-merge branch to main for each target:

```python
LIFECYCLE_FILES = [
    ".close-state", ".execute-progress", ".land-ledger.jsonl",
    ".artifacts-promoted", ".squash-plan",
]

def land(target: CloseTarget) -> str:
    # Idempotency: if branch content already on main, skip
    if branch_already_landed(target.repo, target.branch):
        return find_landed_sha(target.repo, target.branch)

    # Strip lifecycle files from branch before squashing
    strip_lifecycle_files(target.repo, target.branch, LIFECYCLE_FILES)

    checkout(target.repo, "main")
    pull(target.repo)
    sha = squash_merge(target.repo, target.branch)

    # Two-hop: push to canonical first, then canonical pushes to remote
    # Direct: push to remote
    if target.transport == "two-hop":
        push_two_hop(target.repo, target.canonical)
    else:
        push(target.repo)
    return sha
```

If any repo fails to land, the ceremony stops with a structured error naming
the repo and failure. No retry, no reconciliation. The user fixes and re-runs.

**Lifecycle file stripping:** Branch-local files (`.close-state`,
`.execute-progress`, `.land-ledger.jsonl`, `.artifacts-promoted`, etc.) are
removed from the branch before squashing so they don't land on main.

**Two-hop transport:** Slot clones don't have direct remote access. They push
to a canonical repo, which then pushes to origin. `push_two_hop()` handles
this relay. Non-slot repos use direct push.

#### Step 4: Stamp (per target)

Mark each branch as closed with an empty commit containing the landed SHA:

```python
def stamp(target: CloseTarget, landed_sha: str):
    checkout(target.repo, target.branch)
    commit_empty(target.repo,
        f"chore: branch closed — landed as {landed_sha} on main")
    push(target.repo)
```

**Idempotency:** If the branch tip is already a stamp commit (matches
prefix `"chore: branch closed"`), skip.

#### Step 5: Advance Queue

Mark the active item done, activate the next:

```python
def advance(workspace: Path) -> QueueItem | None:
    queue = read_queue(workspace)
    queue.complete_active()
    if queue.has_next():
        queue.activate_next()
        queue.set_status("active")
    else:
        queue.set_status("drained")
    write_queue(workspace, queue)
    return queue.active_item()
```

This is the **only** function that writes to `.queue` during close. No other
step reads, writes, or deletes the queue file.

#### Step 6: Checkout Main

Switch all repos to main:

```python
def checkout_all(targets: list[CloseTarget]):
    for target in targets:
        checkout(target.repo, "main")
```

#### Orchestration

```python
def close(workspace: Path) -> CloseResult:
    # Write ephemeral state
    write_close_state(workspace, "resolving")

    targets = resolve(workspace)
    errors = preflight(targets)
    if errors:
        return CloseResult(success=False, errors=errors)

    landed = {}
    for target in targets:
        sha = land(target)
        landed[target] = sha

    for target, sha in landed.items():
        stamp(target, sha)

    next_item = advance(workspace)
    checkout_all(targets)

    # Clean up ephemeral state
    remove_close_state(workspace)

    return CloseResult(
        success=True,
        landed=landed,
        next_item=next_item,
    )
```

**~80 lines of ceremony code.** Each step is idempotent. Re-running after
a partial failure picks up where it left off by checking what's already done.

### Post-Close Phase

After ceremony returns successfully, the SKILL.md runs cleanup through
normal tool calls:

1. **Close GitHub issues** — `gh issue close` for completed queue items
2. **Archive slot** — if slot and queue is drained, mark for archival
3. **Summary** — print what was landed, where, what's next

Post-close failures are logged but non-fatal. The branch is already landed
and stamped.

## Data Model

### `.queue` — Issue Queue

Replaces `.plan`. Lives in workspace root. Survives across branch close
cycles. Never touched by close ceremony except the `advance` step.

```markdown
- [x] Hortora/soredium#381: Implement batch processing
- [ ] Hortora/soredium#382: Add retry logic        ← active
- [ ] Hortora/soredium#383: Dashboard metrics

## State
branch: issue-382-retry-logic
issue-repo: Hortora/soredium
status: active
```

The `branch:` field tracks the current feature branch. It changes when a new
queue item is activated (each item may use a different branch name).

**Status values:** `active` | `paused` | `drained` | `idle`

- `active` — branch exists, work in progress or close ceremony running
- `paused` — branch exists, work suspended (pause stack)
- `drained` — last item completed, no more work, queue persists for reference
- `idle` — no active work (fresh queue before first `work start`)

**Queue I/O module:** `project/queue_io.py` — the single reader/writer for
`.queue` files. All queue mutations go through this module. There is no
`remove_queue()` function — the queue is never deleted during normal operation.

**Deferred section and batches:** Preserved from current `.plan` format. The
queue file may contain `## Deferred` and batch structure. These are orthogonal
to the close ceremony and managed by `plan_manager.py` (renamed to
`queue_manager.py`).

### `.close-state` — Ephemeral Ceremony State

Created on ceremony entry, deleted on success. If present at session start,
it means a ceremony was interrupted.

```
started: 2026-11-04T12:00:00Z
repos: soredium:issue-499,cc-praxis:issue-499
step: land
```

**Recovery from interrupted ceremony:** The user runs `work end` again. The
ceremony detects `.close-state`, checks idempotency conditions (already
landed? already stamped?), and continues from where it left off. No
reconciliation engine, no progress dict — just simple git state checks.

### Migration from `.plan`

On first read, if `.plan` exists:
1. Rename to `.queue`
2. Strip close-ceremony fields from `## State` (remove `state: closing:*`)
3. Set `status: active` (or `paused` if was paused)

The `.plan` filename is retired. `plan_manager.py` becomes `queue_manager.py`
with the same API. The lifecycle guard hook updates to protect `.queue`.

## What Gets Deleted

| Current file | LOC | Fate |
|---|---|---|
| `work_end_orchestrator.py` | 2,250 | Replaced by `close.py` (~80 LOC) |
| `land_flow.py` | 990 | `land()` function (~15 LOC) |
| `work_end_execute.py` | 612 | Deleted — no execution engine |
| `orchestrator_engine.py` | 440 | Deleted — no engine loop |
| `lifecycle.py` | 776 | `status` field in `.queue` (~30 LOC) |
| `corruption.py` | 620 | ~50 LOC of pre-flight checks |
| `close_progress.py` | ~150 | Deleted — no progress tracking |
| `close_resume.py` | 148 | Deleted — re-run from scratch |
| `close_artifacts.py` | 361 | Moves to pre-close skill |
| `work_end_context.py` | 204 | `CloseTarget` dataclass (~10 LOC) |
| `verify_*.py` (3 files) | ~350 | Deleted — ceremony is self-verifying |
| `handlers/*.md` (8 files) | ~400 | Deleted — no handler dispatch |
| `branch_recon.py` | 224 | `resolve()` function (~20 LOC) |
| `progress_summary.py` | 441 | Deleted — ceremony is short enough to report inline |
| `hygiene_scan.py` | 319 | Moves to pre-close checklist |
| `loose_ends_sweep.py` | 322 | Moves to pre-close checklist |
| `epic_handoff.py` | 324 | Moves to pre-close (handover skill) |
| `fork_next_branch.py` | 129 | `advance()` handles queue; branch creation is work-start's job |

**Before:** ~10,800 LOC across ~58 files, 53 pipeline steps
**After:** ~200 LOC in `close.py` + `queue_io.py` + pre-flight, 6 ceremony steps

## What Survives

| File | Reason |
|---|---|
| `work.py` | Pipeline engine for start/continue/pause/resume — unaffected |
| `work_chain.py` | Chaining engine — updated to use `.queue` instead of `.plan` |
| `ctx.py` | Context resolution — updated to read `.queue` |
| `plan_manager.py` → `queue_manager.py` | Queue CRUD — rename, same API |
| `stack.py` | Pause stack — updated to reference `.queue` |
| `artifact_promote.py` | Moves to pre-close, otherwise unchanged |
| `shared_steps.py` | Shared mechanical steps — review for relevance |
| `audit_work_end.py` | Audit tool — rewritten for new ceremony |
| `work-end/SKILL.md` | Rewritten as pre-close checklist + ceremony invocation |

## Impact on Other Systems

### `ctx.py`
- Read `.queue` instead of `.plan`
- `META_STATE` output replaced by `QUEUE_STATUS` (4 values instead of 12)
- `HAS_PLAN` → `HAS_QUEUE`
- Remove `PLAN_BATCH`, `PLAN_POSITION` — queue position derived from item list

### Corruption triage
- Remove all `closing:*` scenarios (S7, S8, S9, S10, S11, S14, S15, S16)
- Keep git-level checks: branch doesn't exist, queue references deleted branch
- Remove all `auto_action="remove_plan"` — queue is never auto-deleted
- Total: 17+ scenarios → ~4 scenarios

### `work_chain.py`
- Read `.queue` status instead of `.plan` state
- `chain_to_next` directive when `advance()` returns a next item
- `guard_continue` when queue is drained

### Pre-push hook (`pre_push_hook.py`)
- Check `.queue` status instead of lifecycle state
- Block push if ceremony interrupted (`.close-state` exists)

### work-start, work-pause, work-resume
- Read/write `.queue` instead of `.plan`
- Simpler — no lifecycle state machine to interact with

### Tests
- All `test_lifecycle.py` tests rewritten for status field
- All `test_pre_push_hook.py` tests updated for `.queue`
- New tests for `close.py` — one per step + integration
- Ceremony tests are simple: set up git state, run `close()`, verify result

## Risk and Migration

### Migration risk
The `.plan` → `.queue` rename affects every consumer. Mitigation: the queue
I/O module reads both filenames during a transition period. If `.plan` exists
and `.queue` doesn't, migrate on first read.

### Feature parity
Pre-close checklist must cover everything the current orchestrator does.
Mitigation: audit current step list against the checklist before deleting
orchestrator code.

### Slot complexity
Slot close is the most complex current mode. The new design handles it via
`resolve()` returning multiple `CloseTarget`s. Mitigation: test with
real slot topology before cutting over.

### Backward compatibility
Branches created before the migration will have `.plan` files. The migration-
on-read handles this transparently.

## References

- Hortora/soredium#498 — work-end deletes .plan, wiping remaining queue items
- Hortora/soredium#499 — this redesign issue
- ADR-0013 — unified .plan file (this design reverses the coupling it introduced)
- ADR-0009 — automatic workflow orchestration by project type
- PP-20260827 — SKILL.md minimal orchestrator loop protocol
- Architecture analysis fork — 80 issues categorised, 7 deletion paths identified
- `decisions.md` — 6 design decisions with rationale and trade-offs

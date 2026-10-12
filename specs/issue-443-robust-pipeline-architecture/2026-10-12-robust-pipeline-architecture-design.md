# Robust Pipeline Architecture — Design Spec

**Issue:** #443
**Date:** 2026-10-12
**Status:** Draft

## Problem Statement

The work-end pipeline has a 63% fix-to-feature ratio. Of 100 issues filed between #343 and #442 (Sep 7 – Oct 12, 36 days), ~70 are fixes. Four fix chains exist where a fix was itself fixed, sometimes twice. The structural audit (#423, phases 1-7) addressed code duplication but did not eliminate the root causes — 11 bugs emerged in the 2 days after it landed.

### Seven Failure Patterns

| # | Pattern | Issue count | Representative |
|---|---------|-------------|----------------|
| 1 | File-as-Database: multi-writer state corruption | 23 | #441, #434, #410, #398 |
| 2 | Edge-Case Whack-a-Mole: N×M bugs from distributed classification | 17 | #422, #432, #437 |
| 3 | Negative Verification: missing ≈ success | 8 | #436, #375, #435 |
| 4 | Session Boundary Blindness: pipeline assumes one session | 9 | #439, #411, #414 |
| 5 | Multi-Repo Blindspot: single-repo model patched for multi-repo | 11 | #437, #438, #431 |
| 6 | LLM-as-Control-Plane Fragility: state-passing and looping | 7 | #434, #391, #393 |
| 7 | Postcondition Inversion: stamps checked instead of outcomes | 6 | #436, #386, #388 |

### Three Root Causes

These seven patterns reduce to three first-principles violations:

**A. Shared mutable state without a guardian.** `.plan` has 7 writers and 28+ readers. Six code paths bypass the canonical `plan_io.write_fields`. No locking, no schema enforcement, no transactional semantics. Git operations (rebase, merge, checkout) can corrupt the file between writes.

**B. Distributed classification without centralization.** Every step, skip predicate, and postcondition independently probes git to determine repo state. Each probe handles edge cases locally. When a new edge case surfaces (#422 zero-commit, #432 post-squash, #371 workspace-only), it gets fixed in one probe while N others remain vulnerable.

**C. Implicit contracts between components.** Postconditions check stamp file presence (`.artifacts-promoted`, `.landed`) rather than actual outcomes. Components communicate through string parsing and assumed conventions. No typed interfaces between the orchestrator engine and the LLM.

## Architecture Overview

Four phases of evolution, each independently shippable:

1. **Event-sourced lifecycle** — DB events as primary state, `.plan` as derived snapshot
2. **Classify-once context** — RepoContext with disposition enum, propagated to all consumers
3. **Outcome-based verification** — three-valued (PASS/FAIL/UNKNOWN), git state + events
4. **Typed LLM interface** — JudgmentRequest/Response formalizing the existing pattern

The approach is evolution, not replacement. The primitives exist (worklog DB events table, lifecycle.py state machine with CAS, RepoState classification). Two problem categories: (1) primitives that are sound but bypassed, and (2) primitives with quality bugs. Evolution addresses both: enforce the modules, fix the bugs in-place.

## Phase 1: Event-Sourced Lifecycle

### Current State

- `lifecycle.py` has a data-driven state machine with 12 states, 24 transitions, CAS writes, and evidence gates on closing transitions.
- The worklog DB `events` table captures lifecycle transitions — but as a write-only audit trail. Nothing reads events to derive state.
- `.plan` is the primary state store. Seven code paths write to it. `plan_io.write_fields` is the canonical writer but six paths bypass it with raw `write_text()`.
- `.plan` participates in git operations (committed by `_commit_plan_to_git`), making it subject to rebase corruption.

### Design

**Events become the primary state source.** The pipeline reads lifecycle state from DB queries, not `.plan` parsing.

#### Event Schema Extension

Add `repo_path` to the existing `events` table:

```sql
ALTER TABLE events ADD COLUMN repo_path TEXT;
```

Events already capture `work_item_id`, `event_type`, `from_state`, `to_state`, `timestamp`. The `repo_path` column enables per-repo state queries within a single work-item lifecycle.

#### State Derivation

```python
def get_lifecycle_state(work_item_id: int, repo_path: str | None = None) -> str:
    """Query most recent lifecycle event for this work item + repo.
    Returns to_state from the latest event, or 'idle' if no events."""
```

This replaces every `.plan` state: field read. One function, one source.

#### .plan as Derived Snapshot

After each lifecycle event, `plan_io.write_snapshot()` regenerates `.plan` from DB state:

```python
def write_snapshot(workspace: Path, work_item_id: int):
    """Regenerate .plan from current DB state.
    Called after every lifecycle event emission.
    Committed to git for human visibility and clone bootstrap."""
```

`.plan` is committed to git for:
- Human readability (developers can inspect state)
- Clone bootstrap (fresh clones with no DB start from `.plan` snapshot)
- CI/hook consumption (non-Python tools that parse `.plan`)

But no pipeline code reads `.plan` for state decisions. If events and `.plan` disagree, events win.

**Post-rebase .plan regeneration:** Since `.plan` participates in git operations, rebase can corrupt it. After any rebase or merge, the pipeline regenerates `.plan` from DB state. This ensures the recovery snapshot stays consistent even though `.plan` is in git.

#### Single-Writer Enforcement

All `.plan` mutations must route through `lifecycle.py` → `plan_io.write_snapshot()`. The six bypass paths:

| Bypass path | Current behavior | Migration |
|---|---|---|
| `scaffold.py` (3 writes) | Direct `write_text()` for initial `.plan` creation | Emit `scaffolded` event → lifecycle.py → write_snapshot() |
| `recover_plan.py` (2 writes) | Direct `write_text()` for corruption recovery | Emit `recovered` event → lifecycle.py → write_snapshot() |
| `plan_manager.py` (3 writes) | Direct `write_text()` for queue manipulation | Queue operations become events (queue_advanced, queue_appended) |
| `plan_migrate.py` (2 writes) | Direct `write_text()` for format migration | One-time migration — emit `migrated` event after |
| `migrate_meta.py` (1 write) | Direct `write_text()` for .meta → .plan migration | Legacy — emit `migrated` event after |
| `audit_work_end.py` (1 write) | Direct `write_text()` for test fixtures | Test-only — exempt from enforcement, guarded by test context check |

#### SQLite as Primary Store

SQLite with WAL mode (already configured in `worklog.py` line 169). Proven infrastructure — same DB, same connection management, same schema patterns. `busy_timeout` handles rare write contention between concurrent sessions. The `@safe` decorator on write operations is removed from primary-path calls — errors are raised, not silently dropped.

**Recovery from DB corruption/loss:** Bootstrap from `.plan` snapshot + start a new event sequence. This is a defined disaster-recovery path, not an active dual-source model.

#### Migration Steps

| Step | What | Dependencies | Shippable? |
|---|---|---|---|
| 1a | Add `repo_path` column to `events` table | None | Yes |
| 1b | Build `get_lifecycle_state()` query function | 1a | Yes |
| 1c | Route each bypass writer through lifecycle.py (6 paths, one at a time) | None | Yes per path |
| 1d | Build `plan_io.write_snapshot()` from DB state | 1b | Yes |
| 1e | Replace `.plan` readers: ctx.py → work.py → work_end_orchestrator.py → postconditions | 1b | Yes per reader |
| 1f | Add post-rebase `.plan` regeneration | 1d | Yes |

**First win:** After 1c, all state mutations go through lifecycle.py. This alone fixes the multi-writer class of bugs (12+ issues) even before the read side migrates.

## Phase 2: Classify-Once Context

### Current State

- `classify_repo()` in `project/repo_state.py` returns a frozen dataclass. `work.py` uses it in `build_context()`.
- `work_end_orchestrator.py` ignores `RepoState` entirely — builds its own `OrchestratorContext`.
- 12 postcondition functions in `step_postconditions.py` shell out to git independently.
- 13 skip predicates evaluate repo state per-step via subprocess calls.
- The `modes` field on `StepDef` (added in #427) declares which pipeline modes a step runs in, but is not yet connected to repo disposition.

### Design

#### RepoContext: Classification + Lifecycle in One Type

```python
@dataclass(frozen=True)
class RepoContext:
    path: Path
    repo_type: RepoType           # PROJECT | WORKSPACE | CLONE | CANONICAL
    disposition: Disposition      # HAS_CHANGES | EMPTY_BRANCH | NO_BRANCH | MAIN_ONLY
    branch: str | None
    base_branch: str
    commits_ahead: int
    tree_sha: str                 # branch tip tree SHA at classification time
    main_tree_sha: str            # main tip tree SHA
    is_dirty: bool
    remote_synced: bool
    lifecycle_state: str          # from DB events (Phase 1), not .plan
```

#### Disposition Enum

The key field. Collapses all edge cases into one classification:

| Disposition | Meaning | Pipeline behavior |
|---|---|---|
| `HAS_CHANGES` | Feature branch, commits ahead of main | Full pipeline |
| `EMPTY_BRANCH` | Feature branch, zero commits ahead | Skip review/squash/land, trivial close |
| `NO_BRANCH` | No feature branch in this repo | Skip entirely |
| `MAIN_ONLY` | Work done directly on main | No merge needed |

Every step checks `disposition` instead of shelling out to git. The 13 skip predicates reduce to `modes` declarations on `StepDef` evaluated against `disposition` at the engine level.

#### Per-Repo Orchestration

Single lifecycle per work item (D2). The orchestrator fans out within each lifecycle state for per-repo concerns:

- **PER_REPO_EXECUTE_STEPS** (rebase, land): iterate repos with `disposition=HAS_CHANGES`
- **PER_REPO_SWEEP_STEPS** (protocol, update_claude_md, doc_freshness): iterate only the active repo (the one the session actually changed)
- **Archive precondition**: check ALL repos in the slot before offering archive

The existing composite progress keys (`rebase:engine=done`) already support this. What changes: the fan-out uses `RepoContext.disposition` to filter, not per-step git probes.

#### Migration Steps

| Step | What | Dependencies | Shippable? |
|---|---|---|---|
| 2a | Add `disposition` enum and field to `RepoState` | None | Yes |
| 2b | Build `RepoContext` combining RepoState + lifecycle state | Phase 1b | Yes |
| 2c | Propagate `RepoContext` into `work_end_orchestrator.py`, replacing `OrchestratorContext` | 2b | Self-contained |
| 2d | Replace per-step git probes with `disposition` checks | 2c | Yes per step |
| 2e | Connect `modes` on `StepDef` to `disposition` evaluation | 2a | Yes |
| 2f | Fix active-repo filtering in sweep fan-out (#437) | 2c | Yes |
| 2g | Add peer-repo precondition to archive step (#438) | 2c | Yes |

**First win:** After 2a-2b, new code uses `RepoContext`. After 2c, the biggest consumer uses it. Edge-case whack-a-mole stops because classification happens once, immutably.

## Phase 3: Outcome-Based Verification

### Current State

- Postconditions check stamp files (`.artifacts-promoted`, `.landed`, `.phase-a-complete`).
- Missing stamps → retry loop (#435, #386). Present stamps → trust without checking (#375).
- Empty data returns True (#436: `content_landed_postcondition` returns True when `landed_shas` is empty).
- Two verification systems exist with overlapping checks and different return types (#429).

### Design

#### Three-Valued VerifyResult

```python
class VerifyResult(Enum):
    PASS = "pass"         # positive evidence of success
    FAIL = "fail"         # positive evidence of failure
    UNKNOWN = "unknown"   # insufficient evidence

@dataclass(frozen=True)
class Verification:
    check: str            # "content_landed", "branch_merged", "issues_closed"
    result: VerifyResult
    evidence: dict        # what was checked, what was found
    repo: str
```

UNKNOWN never collapses to PASS. Three-step resolution:

1. **Deterministic check** — verify git state directly
2. **Deterministic fallback/retry** — known recovery actions for specific UNKNOWN conditions
3. **LLM escalation** — last resort for genuinely ambiguous situations

#### Enumerated UNKNOWN Conditions

| Condition | Fallback | Escalate when |
|---|---|---|
| Partial landing (some repos landed) | Retry unfinished repos | Retry limit exceeded |
| Ambiguous branch state (detached HEAD, rebase-in-progress) | Reattach / abort and retry | Topology truly unknown |
| Missing evidence (expected git state doesn't match) | Wait-and-recheck with backoff (propagation delay) | N retries exhausted |

#### What Each Verification Checks

| Check | Positive evidence required | Returns UNKNOWN when |
|---|---|---|
| `content_landed` | main tip tree SHA matches branch content | main tip unchanged since classification |
| `branch_merged` | branch tip reachable from main via `merge-base --is-ancestor` | branch deleted before check |
| `issues_closed` | GitHub API confirms issue state=closed | API unavailable |
| `canonical_synced` | canonical repo main ≥ clone's landed SHA | canonical unreachable |
| `artifacts_promoted` | workspace diff empty for artifact paths | workspace unavailable |

#### Stamp File Elimination

No more `.artifacts-promoted`, `.landed`, `.phase-a-complete`. Verification reads:
1. Git state directly (tree SHAs, ancestry, diffs)
2. Event log (land event with SHA, promote event with paths)

The event log replaces stamps — it records WHEN something happened and WHAT the evidence was, not just THAT a file exists.

#### Migration Steps

| Step | What | Dependencies | Shippable? |
|---|---|---|---|
| 3a | Define `VerifyResult` and `Verification` types | None | Yes |
| 3b | Build `verify_outcome()` functions — git state + event checks | Phase 1b, 2a | Yes |
| 3c | Replace `content_landed_postcondition` (highest-failure) | 3b | Yes |
| 3d | Replace remaining postconditions one at a time | 3b | Yes per postcondition |
| 3e | Build UNKNOWN→fallback→escalation resolution path | 3a, Phase 4 (for escalation) | Yes |
| 3f | Remove stamp files after all consumers migrate | 3c, 3d | Yes per file |

**First win:** After 3c, the most common postcondition failure class (#436, #386, #388) is eliminated.

## Phase 4: Typed LLM Interface

### Current State

- `work_end_orchestrator.py` already implements Python-drives-LLM. The docstring says: "Python drives, LLM assists. The LLM cannot skip what it cannot see."
- `StepDef` has `step_type` (mechanical/judgment/lifecycle), `action_context_fn` providing structured context, `verify_fn` validating responses.
- But: `action_context_fn` returns ad-hoc dicts, `verify_fn` only validates counts not content, judgment steps lack explicit input/output contracts.
- The `COMPLETE_WITH=` hint (#391) was a step toward typed responses but remains stringly-typed.

### Design

This is formalization of the existing pattern, not a direction change.

#### JudgmentRequest / JudgmentResponse

```python
@dataclass(frozen=True)
class JudgmentRequest:
    step: str                    # "code_review", "brainstorm_offer", ...
    handler: str                 # path to handler .md file to load
    context: dict                # structured context (from action_context_fn)
    response_schema: dict        # JSON schema for expected response
    skippable: bool              # can the LLM skip this step?
    timeout_hint: str            # "quick" | "standard" | "extended"

@dataclass(frozen=True)
class JudgmentResponse:
    step: str                    # echoes request.step
    outcome: str                 # "completed" | "skipped" | "blocked"
    produced: int                # artifacts produced
    evidence: dict               # step-specific output
    block_reason: str | None     # if blocked, why
```

#### What Changes

- `action_context_fn` returns `JudgmentRequest` instead of ad-hoc dict
- `verify_fn` validates `JudgmentResponse` against the schema
- SKILL.md orchestrator loop parses `JudgmentRequest` and returns `JudgmentResponse`
- State is never passed by the LLM — the orchestrator reads from DB events (Phase 1)
- Invalid responses are rejected with specific error, not silently re-yielded

#### What Stays the Same

- Handler `.md` files still guide the LLM's judgment work
- The `run_loop` in `orchestrator_engine.py` still drives execution
- Mechanical steps remain pure Python
- The SKILL.md still contains the orchestrator call loop

#### Migration Steps

| Step | What | Dependencies | Shippable? |
|---|---|---|---|
| 4a | Define `JudgmentRequest` / `JudgmentResponse` dataclasses | None | Yes |
| 4b | Add `response_schema` field to `StepDef` | 4a | Yes |
| 4c | Update `run_loop` to emit structured requests | 4a | Yes |
| 4d | Update work-end SKILL.md to parse requests and return responses | 4c | Paired with 4c |
| 4e | Convert judgment steps one at a time (code_review first) | 4d | Yes per step |
| 4f | Remove old `ACTION=` / `step_done=` protocol | All steps converted | Cleanup |

**Coexistence:** Both protocols (old key=value and new typed) coexist during migration. `run_loop` detects which protocol the response uses and handles both.

## Phasing and Dependencies

```
Phase 1a-c (single writer enforcement)    ← highest leverage, do first
    │
    ├── Phase 1d-f (read migration, .plan snapshot)
    │       │
    │       └── Phase 3 (outcome verification)
    │
    └── Phase 2 (classify-once context)
            │
            └── Phase 3 (outcome verification)

Phase 4 (typed LLM interface)  ← independent, parallel with everything
```

Phase 1a-c is the critical path. Closing the six bypass writers eliminates the multi-writer class of bugs immediately. Everything else builds on that foundation but each step is independently valuable.

## What NOT to Do

- **Don't migrate everything at once.** Each step is a separate issue with its own tests.
- **Don't remove `.plan` files.** They become derived snapshots — still committed, still human-readable. The change is invisible to the developer experience.
- **Don't break the existing protocol during migration.** Old `ACTION=` / `step_done=` stays as fallback until all steps are converted.
- **Don't build complex slot coordination.** D2 (revised) keeps a single lifecycle per work item with per-repo fan-out at the orchestrator level — no new state machines.
- **Don't use `@safe` on primary write paths.** Errors must be raised, not silently dropped. `@safe` stays only on truly best-effort secondary operations.

## Testing Strategy

Each phase has a testing approach:

**Phase 1:** Test `get_lifecycle_state()` with event sequences. Test `write_snapshot()` produces correct `.plan` from DB state. Test bypass closure — each former bypass path now emits events. Integration test: event → state derivation → snapshot → matches expected .plan content.

**Phase 2:** Test `classify_repo()` disposition for each edge case (zero commits, no branch, main-only, dirty). Test `RepoContext` immutability. Test skip predicate replacement — same steps skipped with `disposition` as with git probes.

**Phase 3:** Test each `verify_outcome()` for PASS, FAIL, UNKNOWN states. Test UNKNOWN resolution path — fallback before escalation. Test that no verification function returns PASS on empty/missing evidence.

**Phase 4:** Test `JudgmentRequest` → `JudgmentResponse` round-trip. Test schema validation rejects malformed responses. Test coexistence — old protocol responses handled during migration.

## Success Criteria

The redesign is complete when:

1. All `.plan` writes route through `plan_io.write_snapshot()` (zero direct file manipulation)
2. All lifecycle transitions route through `lifecycle.py` with event emission
3. All state reads use `get_lifecycle_state()` from DB, not `.plan` parsing
4. All steps use `RepoContext.disposition` for classification, not git probes
5. All postconditions use `VerifyResult` with three-valued logic
6. All judgment steps use `JudgmentRequest`/`JudgmentResponse`
7. Fix-to-feature ratio drops below 30% over a 30-day window
8. Zero fix chains (fix-of-a-fix) in any 30-day window

## References

- Issues #343-#442 (failure pattern analysis)
- #423 Pipeline structural audit (phases 1-7)
- #424 git_util.py extraction
- #425 slot_io.py extraction
- #426 RepoState classification
- #427 PipelineMode enum
- #428 Step dependencies
- #429 Verification unification
- #430 Typed Progress
- project/lifecycle.py (state machine, CAS writes, evidence gates)
- project/repo_state.py (classify_repo, RepoState)
- project/worklog.py (events table, WAL mode)
- work-end/work_end_orchestrator.py (orchestrator engine, StepDef)
- verification/step_postconditions.py (current postcondition logic)
- docs/protocols/git-util-single-source.md (single-source protocol)
- docs/protocols/evidence-before-claims.md (verification protocol)
- docs/protocols/skill-md-minimal-orchestrator-loop.md (orchestrator pattern)

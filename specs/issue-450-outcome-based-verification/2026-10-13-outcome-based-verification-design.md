# Outcome-Based Verification — Design Spec

**Issue:** #450
**Date:** 2026-10-13
**Status:** Draft
**Parent:** #443 (Robust Pipeline Architecture), Phase 3

## Problem Statement

The work-end pipeline uses two overlapping verification systems with
different return types:

1. `step_postconditions.py` — 15 functions returning `bool`, used by the
   orchestrator engine's check-execute-verify pattern
2. `close_checks.py` / `close_verify.py` — functions returning
   `list[Finding]`, used for close-time verification reports

Both check overlapping things (e.g., `content_landed_postcondition` vs
`check_landed_completeness`, `stamp_postcondition` vs `check_branch_stamp`).
Edge-case fixes applied to one layer don't propagate to the other —
exactly the pattern that caused #436 (empty `landed_shas` returned True),
#375 (negative check: absent stamp ≈ success), and #435 (stamp never
written but postcondition passed).

Additionally, both systems use binary outcomes (bool or ERROR/WARN
severity) that cannot distinguish "verified — it passed" from
"insufficient evidence to determine." Missing evidence currently
collapses to either True (dangerous) or False (retry loop), with no
path for the engine to try deterministic recovery before retrying
or escalating.

## Design Overview

Replace both verification systems with a single layer built on
three-valued outcomes: PASS (positive evidence of success), FAIL
(positive evidence of failure), UNKNOWN (insufficient evidence).
The engine handles UNKNOWN via deterministic retry and, as a last
resort, LLM escalation.

### Architecture

```
verification/
  outcomes.py          # VerifyResult, Verification, verify_*() functions
  __init__.py          # Finding (unchanged — presentation type)
  close_verify.py      # composes verify_*() → to_finding() → report
  close_checks.py      # deprecated, consumers migrate to outcomes.py
  step_postconditions.py  # deprecated, consumers migrate to outcomes.py
  postconditions.py    # unchanged (Finding-based lifecycle gate checks)
  slot_checks.py       # unchanged (slot isolation checks)
  repo_checks.py       # unchanged (stuck-state checks)
  cross_checks.py      # unchanged
```

## Types

### VerifyResult

```python
class VerifyResult(Enum):
    PASS = "pass"
    FAIL = "fail"
    UNKNOWN = "unknown"
```

### Verification

```python
@dataclass(frozen=True)
class Verification:
    check: str            # "content_landed", "branch_merged", etc.
    result: VerifyResult
    evidence: dict        # what was checked, what was found
    repo: str             # repo name (for per-repo fan-out)
```

**Evidence dict convention:** Keys describe what was checked, values
describe what was found. Examples:

```python
# PASS
Verification("content_landed", VerifyResult.PASS,
    {"method": "tree_sha_match", "branch_tree": "abc123", "main_tree": "abc123"},
    repo="soredium")

# FAIL
Verification("content_landed", VerifyResult.FAIL,
    {"method": "diff_check", "diff_files": ["src/main.py", "tests/test_x.py"]},
    repo="soredium")

# UNKNOWN
Verification("content_landed", VerifyResult.UNKNOWN,
    {"reason": "main_tip_unchanged", "main_sha_at_classify": "def456",
     "main_sha_now": "def456"},
    repo="soredium")
```

### Composition helper

```python
def worst_of(*verifications: Verification) -> Verification:
    """Return the most serious result: FAIL > UNKNOWN > PASS."""
    for target in (VerifyResult.FAIL, VerifyResult.UNKNOWN):
        for v in verifications:
            if v.result == target:
                return v
    return verifications[0]
```

Used when a step needs to compose multiple checks into a single
`Verification` (D3 — one Verification per step). `close_verify.py`
calls individual functions directly for its multi-check report.

## Verify Functions

All live in `verification/outcomes.py`. Each takes a `RepoContext`
(or `OrchestratorContextBase`) and returns `Verification`. Pure
functions — observe and report, never retry (D2).

### verify_content_landed

Replaces `content_landed_postcondition` and parts of
`check_landed_completeness`.

```python
def verify_content_landed(ctx, repo: RepoContext) -> Verification:
    """Verify branch content is on main after landing."""
```

**PASS evidence:**
- `merge-base --is-ancestor landed_sha base_branch` succeeds, OR
- branch tree SHA matches main tree SHA (squash merge), OR
- `git diff base_branch..landed_sha -- <source_exts>` is empty

**FAIL evidence:**
- `git diff` shows files with content differences

**UNKNOWN conditions:**
- No `landed_sha` available (not yet recorded) — falls back to stamp
  file `.landed` as secondary evidence (D4 dual-read)
- Main tip unchanged since classification (`main_tree_sha` matches
  current) — evidence may be propagating from a remote push

### verify_branch_merged

Replaces `check_branch_merged` and aspects of `stamp_postcondition`.

```python
def verify_branch_merged(ctx, repo: RepoContext) -> Verification:
    """Verify branch commits are reachable from main."""
```

**PASS evidence:**
- No non-ceremony commits ahead of base, AND tree SHAs match (or
  pre-stamp tree matches), OR
- `merge-base --is-ancestor branch~1 base` succeeds, OR
- Landing SHA from stamp is ancestor of base

**FAIL evidence:**
- Non-ceremony, non-lifecycle commits exist ahead of base with
  content differences

**UNKNOWN conditions:**
- Branch deleted before check (ref not found)

### verify_branch_stamped

Replaces `stamp_postcondition` and `check_branch_stamp`.

```python
def verify_branch_stamped(ctx, repo: RepoContext) -> Verification:
    """Verify branch has a 'chore: branch closed' stamp commit."""
```

**PASS evidence:**
- Tip commit (within last 5) starts with `"chore: branch closed"`,
  AND if it contains `"landed as <SHA>"`, that SHA is on base branch

**FAIL evidence:**
- No stamp commit found in recent history

**UNKNOWN conditions:**
- Stamp exists but SHA not resolvable (object may not have been
  fetched)

### verify_push

Replaces `push_postcondition` and `check_main_pushed`.

```python
def verify_push(ctx, repo: RepoContext) -> Verification:
    """Verify main is pushed to origin."""
```

**PASS evidence:**
- `git log origin/main..main --oneline` is empty (zero unpushed)

**FAIL evidence:**
- N commits ahead of origin

**UNKNOWN conditions:**
- Remote unreachable (network error)

### verify_issues_closed

Replaces `issues_closed_postcondition` and `check_issues_closed`.

```python
def verify_issues_closed(ctx) -> Verification:
    """Verify covered issues are closed on GitHub."""
```

**PASS evidence:**
- All closable issues report `state=CLOSED` via `gh issue view`

**FAIL evidence:**
- One or more issues report state != CLOSED

**UNKNOWN conditions:**
- `gh` command fails (API unavailable, rate limited)

### verify_artifacts_promoted

Replaces `promote_postcondition` and `check_artifacts_promoted`.

```python
def verify_artifacts_promoted(ctx, repo: RepoContext) -> Verification:
    """Verify workspace artifacts have been promoted."""
```

**PASS evidence:**
- `.artifacts-promoted` stamp exists (D4 — stamp read as secondary
  evidence during migration), OR
- Promotion stamp commit found in git log, OR
- Branch has no diff against base (nothing to promote)

**FAIL evidence:**
- No stamp, no promotion commit, and branch has meaningful diff

**UNKNOWN conditions:**
- Workspace unavailable

### verify_rebase

Replaces `rebase_postcondition`.

```python
def verify_rebase(ctx, repo: RepoContext) -> Verification:
    """Verify base branch is ancestor of feature branch."""
```

**PASS evidence:**
- `merge-base --is-ancestor base_branch branch` succeeds

**FAIL evidence:**
- Ancestry check fails

**UNKNOWN conditions:**
- Branch ref not found

### verify_checkout_main

Replaces `checkout_main_postcondition`.

```python
def verify_checkout_main(ctx) -> Verification:
    """Verify both repos are on main."""
```

**PASS evidence:**
- Both project and workspace HEAD are `main`

**FAIL evidence:**
- One or both repos on a different branch

### verify_canonical_sync

Replaces `check_canonical_sync`.

```python
def verify_canonical_sync(ctx, slot_dir, repo_name, canonical_path) -> Verification:
    """Verify canonical repo has the landed content."""
```

**PASS evidence:**
- Landed SHA is ancestor of canonical main, OR
- Tree SHA match on canonical main, OR
- Diff between landed SHA and canonical main is empty

**FAIL evidence:**
- SHA not resolvable in canonical repo

**UNKNOWN conditions:**
- Canonical repo unreachable

### verify_landed_marker

Replaces `landed_marker_postcondition` and `check_landed_marker`.

```python
def verify_landed_marker(ctx, slot_dir) -> Verification:
    """Verify .landed marker exists with valid SHA data."""
```

During D4 migration: reads `.landed` file as primary (stamp file
still written). After migration: replaced by event log query.

### verify_slot_phase_complete

Replaces `write_marker_postcondition` and `check_slot_marker`.

```python
def verify_slot_phase_complete(ctx, slot_dir, marker) -> Verification:
    """Verify a slot phase marker exists."""
```

During D4 migration: reads marker file. After migration: replaced
by event log query.

## Engine Integration

### StepDef changes

`postcondition_fn` on `StepDef` changes from `Callable[[ctx], bool]`
to `Callable[[ctx], Verification]`. Rename to `verify_fn` to reflect
the semantic change (note: `StepDef` already has a `verify_fn` field
used for judgment step response validation — that field is renamed to
`response_validate_fn` to disambiguate).

```python
@dataclass
class StepDef:
    name: str
    phase: str
    step_type: str
    # ... existing fields ...
    verify_fn: Callable | None = None           # was postcondition_fn
    response_validate_fn: Callable | None = None  # was verify_fn
```

### run_loop() three-way dispatch

The orchestrator engine's mechanical step handling changes from
binary (bool) to three-way (VerifyResult):

```python
# Pre-execution check
if step.verify_fn:
    v = step.verify_fn(ctx)
    if v.result == VerifyResult.PASS:
        _mark_done(ctx, step.name)
        ctx.steps_executed.append(f"{step.name}:verify_skip")
        continue
    if v.result == VerifyResult.FAIL:
        # Proceed to execution — the step needs to run
        pass
    if v.result == VerifyResult.UNKNOWN:
        # Retry with backoff, then escalate
        attempt += 1
        if attempt >= MAX_MECHANICAL_RETRIES:
            return _yield_unknown_escalation(step.name, v)
        # Continue to execution — may resolve the UNKNOWN
        pass

# Post-execution verify
if step.verify_fn:
    v = step.verify_fn(ctx)
    if v.result == VerifyResult.PASS:
        _mark_done(ctx, step.name)
        continue
    if v.result == VerifyResult.FAIL:
        return _make_verify_fail_result(step.name, attempt, v)
    if v.result == VerifyResult.UNKNOWN:
        attempt += 1
        if attempt >= MAX_MECHANICAL_RETRIES:
            return _yield_unknown_escalation(step.name, v)
        return _make_unknown_retry_result(step.name, attempt, v)
```

### UNKNOWN escalation

When UNKNOWN persists after `MAX_MECHANICAL_RETRIES`:

```python
def _yield_unknown_escalation(step_name, verification):
    return {
        "ACTION": "user_input",
        "CONTEXT": "unknown_verification",
        "STEP": step_name,
        "CHECK": verification.check,
        "RESULT": "unknown",
        "EVIDENCE": json.dumps(verification.evidence),
        "REASON": f"Verification '{verification.check}' returned UNKNOWN "
                  f"after {MAX_MECHANICAL_RETRIES} retries",
    }
```

The LLM receives structured UNKNOWN context (check name, evidence
dict) and can make a judgment call — inspect the evidence, try a
manual resolution, or ask the user.

## close_verify.py Adaptation

### Verification → Finding adapter

```python
def verification_to_findings(v: Verification) -> list[Finding]:
    if v.result == VerifyResult.PASS:
        return []
    severity = "ERROR" if v.result == VerifyResult.FAIL else "WARN"
    evidence_summary = "; ".join(f"{k}={v}" for k, v in v.evidence.items())
    return [Finding(severity, v.check, evidence_summary)]
```

`close_verify.py`'s `_run_branch_checks()` and `_run_slot_checks()`
switch from calling `close_checks.py` functions to calling
`outcomes.py` `verify_*()` functions, converting via
`verification_to_findings()`. The output format (VERIFIED=,
FAILURE_N=, BLOCKING_FAIL=) stays unchanged.

## Migration Plan

Follows the parent spec's 3a→3f ordering. Each step is independently
shippable.

### Step 3a: Define types

- Add `VerifyResult` enum and `Verification` dataclass to
  `verification/outcomes.py`
- Add `worst_of()` composition helper
- Add `verification_to_findings()` adapter
- Tests: unit tests for types, worst_of, adapter

### Step 3b: Build verify functions

- Implement all `verify_*()` functions in `verification/outcomes.py`
- Each function checks git state first, falls back to stamp files
  as secondary evidence (D4)
- Tests: unit tests per function covering PASS, FAIL, UNKNOWN paths
- **First win target:** `verify_content_landed` — replaces the
  highest-failure postcondition (#436, #386, #388)

### Step 3c: Replace content_landed_postcondition

- Wire `verify_content_landed` into the step that uses
  `content_landed_postcondition`
- Rename `postcondition_fn` → `verify_fn` on `StepDef` (rename
  existing `verify_fn` → `response_validate_fn` first)
- Update `run_loop()` for three-way dispatch
- Update `close_verify.py` to use `verify_content_landed` via adapter
- Remove `content_landed_postcondition` from `step_postconditions.py`
- Tests: integration test that UNKNOWN triggers retry then escalation

### Step 3d: Replace remaining postconditions

One at a time, in order of failure frequency:
1. `push_postcondition` → `verify_push`
2. `stamp_postcondition` → `verify_branch_stamped`
3. `rebase_postcondition` → `verify_rebase`
4. `promote_postcondition` → `verify_artifacts_promoted`
5. `issues_closed_postcondition` → `verify_issues_closed`
6. `checkout_main_postcondition` → `verify_checkout_main`
7. `landed_marker_postcondition` → `verify_landed_marker`
8. `write_marker_postcondition` → `verify_slot_phase_complete`
9. `archive_move_postcondition` → (simple directory check, inline)
10. `cleanup_scaffold_postcondition` → (list check, inline)
11. `promote_plan_next_postcondition` → (file existence, inline)
12. `fork_next_postcondition` → (branch check, inline)
13. `checkout_fork_postcondition` → (branch check, inline)

Corresponding `close_checks.py` functions are replaced at the same
time — no check exists in two places.

### Step 3e: UNKNOWN resolution path

- Build `_yield_unknown_escalation()` in `orchestrator_engine.py`
- Build `_make_unknown_retry_result()` for retry with evidence
- Add UNKNOWN handler to SKILL.md orchestrator loop
- Tests: engine-level tests for UNKNOWN retry and escalation

### Step 3f: Remove stamp files

After all consumers migrate:
1. Stop writing `.artifacts-promoted` in `artifact_promote.py`
2. Stop writing `.phase-a-complete` in slot close flow
3. Remove stamp-file fallback reads from `verify_*()` functions
4. Remove `verify_stamp.py` (standalone stamp verifier)
5. Clean up `LIFECYCLE_FILES` set in `close_checks.py`
6. Tests: verify all tests pass without stamp files

## Scope Boundaries

**In scope:**
- `VerifyResult`/`Verification` types
- `verify_*()` functions replacing both `step_postconditions.py` and
  `close_checks.py`
- Engine three-way dispatch for UNKNOWN
- `close_verify.py` adapter
- Stamp file elimination

**Out of scope:**
- Event-sourced lifecycle (Phase 1 — partially landed, continues
  independently)
- Typed LLM interface (Phase 4, #451 — uses `Verification` as input
  but is a separate concern)
- `postconditions.py` lifecycle gate checks (Finding-based, different
  purpose — these are pre-work-start checks, not step postconditions)
- `slot_checks.py`, `repo_checks.py`, `cross_checks.py` — slot
  isolation and stuck-state checks, unrelated to outcome verification

## Testing Strategy

- **Unit tests:** Each `verify_*()` function tested with mocked git
  state for all three outcomes (PASS, FAIL, UNKNOWN)
- **Composition tests:** `worst_of()` and `verification_to_findings()`
- **Engine tests:** `run_loop()` with verify functions returning each
  result type, retry counting, escalation threshold
- **Integration tests:** Full close sequence with verify functions
  replacing postconditions, verify output format unchanged
- **Protocol:** `externalised-scripts-require-tests` — every new .py
  file ships with tests in the same commit

## References

- Parent spec: `specs/issue-443-robust-pipeline-architecture/2026-10-12-robust-pipeline-architecture-design.md` § Phase 3
- Parent decisions: D5 (verification model), D2 (single lifecycle), D3 (events primary)
- `verification/step_postconditions.py` — 15 bool postconditions being replaced
- `verification/close_checks.py` — Finding-based checks being replaced
- `verification/close_verify.py` — composed reporter being adapted
- `project/orchestrator_engine.py:265-329` — current binary postcondition dispatch
- `project/shared_steps.py:34-49` — StepDef with postcondition_fn
- `project/repo_state.py:192-236` — RepoContext with tree_sha, main_tree_sha
- Issues: #436, #375, #435, #386, #388

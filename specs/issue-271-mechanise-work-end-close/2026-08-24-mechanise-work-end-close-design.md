# Mechanise work-end close sequence

**Issue:** #271
**Date:** 2026-08-24
**Principle:** Python drives, LLM assists. The LLM cannot skip what it cannot see.

---

## Problem

Every slot closure audit found failures caused by the LLM skipping or
partially executing mechanical steps in work-end:

| Slot | Failure | Category |
|------|---------|----------|
| 134 | Repos not switched to main before archive | LLM skip |
| 141 | Artifacts not promoted, workspaces dirty | LLM skip |
| 54 | Code not pushed to origin | LLM skip |
| 54 | Blog not promoted to public workspace | LLM skip |
| 54 | Branch deleted instead of stamped | LLM skip |
| 149 | Blog uncommitted in attic, not promoted | LLM skip |

Root cause: the LLM reads a 660-line SKILL.md and orchestrates ~20 steps.
LLMs don't reliably execute long sequences. No amount of better instructions
fixes this — the architecture is wrong.

---

## Solution: Inverted control

Replace the LLM-as-orchestrator model with Python-as-orchestrator. A single
Python script owns the close sequence. The LLM becomes a service called
when the script needs judgment.

### Current model (LLM orchestrates)

```
SKILL.md says "do steps 1-15"
LLM reads all 15, skips step 7, forgets step 12, claims success
```

### New model (Python orchestrates)

```
LLM calls work_end_orchestrator.py
  Script runs mechanical steps (push, stamp, verify...)
  Script hits judgment point (needs blog written)
  Script prints: ACTION=write_content
  Script exits

LLM reads output, writes blog, calls orchestrator again
  Script validates blog exists
  Script continues mechanical steps
  ...
  Script prints: ACTION=complete
```

The SKILL.md becomes a ~20-line loop. The LLM never sees the full step
sequence, so it cannot skip steps.

---

## Architecture

### Stateless re-entrant orchestrator (D1)

`work_end_orchestrator.py` is a stateless script. Each invocation:

1. Reads `.close-progress` to determine current position
2. Runs all mechanical steps up to the next judgment point
3. Prints one `ACTION=` line with action-specific context
4. Exits

The LLM calls it repeatedly. No long-running process, no bidirectional
streaming, no watchdogs.

**Why not Claude Code Workflows:** Workflows spawn a fresh agent for each
step, losing accumulated conversation context. Work-end benefits from
context preservation — review findings inform sweep, sweep informs squash
analysis, user decisions carry forward. The re-entrant script preserves
this because the LLM maintains one continuous conversation.

### SKILL.md loop (D2)

```
loop:
  output = run("python3 work-end/work_end_orchestrator.py ...")
  if ACTION=complete → print summary, done
  if ACTION=invoke_skill → invoke the named skill with provided args
  if ACTION=user_input → present to user, pass response on next call
  go to loop
```

The LLM has no knowledge of the full step sequence. It sees one
instruction at a time. The orchestrator decides what's next.

### Context resolution (pre-close)

Step 1 context resolution (`work_end_context.py`, precondition handling,
queue gate, dirty-tree protocol) remains in the SKILL.md as pre-close
setup. The orchestrator's close sequence starts after context is resolved
and `closing:review` is entered. This separation keeps the orchestrator
focused on the close sequence — context resolution is interactive and
benefits from SKILL.md-level handling.

---

## Action types (D3, D4)

### Core actions

| Action | LLM does | Lifecycle state | Validation |
|--------|----------|----------------|------------|
| `review` | code-review + branch-audit + loose-ends + forcing function | `closing:review` | `findings.jsonl`: all findings status != `open` |
| `sweep_config` | Present toggle UI, report selections | `closing:review` | `SELECTED=` line in output |
| `forage` | Invoke forage SWEEP | `closing:review` | Garden entry files created (or explicit skip) |
| `protocol` | Invoke protocol SWEEP | `closing:review` | Protocol files created (or explicit skip) |
| `update_claude_md` | Invoke update-claude-md | `closing:review` | CLAUDE.md timestamp (or no changes needed) |
| `impl_doc_sync` | Invoke implementation-doc-sync | `closing:review` | Docs synced (or no changes needed) |
| `adr` | Invoke adr | `closing:review` | ADR file created (or explicit skip) |
| `write_content` | Invoke write-content (diary) | `closing:review` | Blog file exists, frontmatter valid, word count > 100 |
| `squash` | Classify commits per repo | `closing:promoted` | `.squash-plan-<repo>.json` exists, valid JSON |
| `trajectory` | Draft enrichment notes | `closing:stamped` | Enrichment recorded (or explicit skip) |
| `verify_recover` | Present verify failures, offer recovery | When verify fails | Re-run verify passes |
| `user_input` | Parameterised via CONTEXT= | `closing:stamped` | Response received |

### `user_input` contexts

| CONTEXT= | What LLM presents |
|----------|-------------------|
| `arc42_scan` | ARC42 stale scan results, offer fixes |
| `session_rename` | Suggest descriptive session name |
| `garden_feedback` | Assess GE-ID relevance per category |
| `notes` | Prompt user for notes, append to NOTES.md |
| `step_failed` | Judgment step failed after 3 retries — skip/retry/abort? |
| `rebase_conflict` | Rebase conflict needs manual resolution |

### Conditional actions

- **Post-rebase re-review:** If rebase had conflicts, orchestrator yields
  `ACTION=review` with `DIFF_RANGE=` scoped to conflict resolution commits.
  Same action type, different parameters.
- **`verify_recover`:** Fires only when `verify_slot_close.py` returns
  `VERIFIED=no`. Not part of the normal sequence.
- **Sweep sub-steps:** Conditional on `sweep_config` selections. User
  toggles items off → orchestrator skips those actions entirely.

### Lifecycle state mapping

| Lifecycle state | Actions that fire | Event to advance |
|---|---|---|
| `closing:review` | review, sweep_config, [selected sweep sub-steps] | `review_pass` |
| `closing:verified` | promote (mechanical) | `promote_pass` |
| `closing:promoted` | squash, rebase (mechanical), land (mechanical) | `push_pass` → `merge_pass` → `stamp_pass` |
| `closing:stamped` | trajectory, user_input (cleanup items) | `cleanup_pass` |

`closing:pushed` and `closing:merged` are transient — rapid succession
after land completes. The orchestrator fires lifecycle events but does
not yield LLM actions between them.

---

## Protocol format (D7)

KEY=VALUE lines with action-specific context. The orchestrator provides
values the LLM needs to execute THIS action correctly. Does not repeat
generic context already in the conversation.

### Examples

```
ACTION=review
DIFF_RANGE=main..issue-271-mechanise-work-end-close
```

```
ACTION=sweep_config
ITEMS=forage:on,protocol:on,update_claude_md:on,impl_doc_sync:on,adr:on,write_content:on
```

```
ACTION=write_content
BRANCH_SUMMARY=Mechanised work-end close sequence with Python orchestrator
ISSUE=271
```

```
ACTION=squash
REPOS=soredium
PLAN_DIR=/Users/mdproctor/claude/public/cc-praxis/design
```

```
ACTION=user_input
CONTEXT=step_failed
STEP=write_content
ATTEMPTS=3
REASON=no blog file found in workspace/blog/
```

```
ACTION=complete
SUMMARY=Close complete. 14 mechanical steps, 8 judgment steps. All verified.
```

### LLM response protocol

The LLM does not pass structured responses back to the orchestrator.
It simply calls the orchestrator again. The orchestrator validates by
checking evidence (files, git state, findings.jsonl), not by reading
LLM output.

Exception: `sweep_config` — the LLM passes user selections on the
next orchestrator call:

```
python3 work-end/work_end_orchestrator.py ... sweep_selected=forage,protocol,write_content
```

---

## Validation stack (D5)

### Primary: mechanical heuristics

| Action | Validation check |
|--------|-----------------|
| `review` | `findings.jsonl` exists, all entries have status != `open` |
| `forage` | Garden entry files created since step start, OR explicit user skip |
| `protocol` | Protocol files created since step start, OR explicit user skip |
| `update_claude_md` | CLAUDE.md mtime changed, OR diff shows no changes needed |
| `impl_doc_sync` | Docs directory mtime changed, OR diff shows no changes needed |
| `adr` | ADR file created in workspace `adr/`, OR explicit user skip |
| `write_content` | Blog file exists in workspace `blog/`, frontmatter valid, word count > 100 |
| `squash` | `.squash-plan-<repo>.json` exists, valid JSON, has expected fields |
| `trajectory` | Enrichment DB updated, OR explicit user skip |

### Secondary: Haiku semantic validation (optional)

Haiku via Vertex AI, using Application Default Credentials. Called only
when:
- The primary heuristic check passes (file exists, structure valid)
- The action type is write-content or ADR (content quality matters)
- Vertex AI is available (graceful skip when offline, CI, quota exhaustion)

```python
def haiku_validate(prompt: str, content: str) -> ValidateResult:
    try:
        client = AnthropicVertex(region="us-east5", project_id=...)
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=200,
            messages=[{"role": "user", "content": f"{prompt}\n\n{content}"}]
        )
        # parse yes/no + reason
    except Exception:
        return ValidateResult(valid=True, reason="Haiku unavailable — heuristic-only")
```

The close sequence never blocks on Haiku availability.

---

## Progress tracking and crash recovery (D8)

### File format

`.close-progress` — flat KEY=VALUE, one line per sub-step:

```
step=review status=done
step=sweep_config status=done sweep_selected=forage,protocol,write_content
step=forage status=done
step=protocol status=done
step=write_content status=pending attempt=2
step=promote status=done
step=rebase status=done
step=squash status=done
step=land status=done
```

### Atomic writes

Write-then-rename using `os.replace()` (atomic on POSIX):

```python
def write_progress(progress_path: Path, entries: dict):
    tmp = progress_path.with_suffix('.tmp')
    lines = [f"{k}={v}" for k, v in entries.items()]
    tmp.write_text("\n".join(lines) + "\n")
    os.replace(tmp, progress_path)
```

This matches `lifecycle.py`'s `write_state()` pattern. The existing
`write_progress()` in `work_end_execute.py` and `_write_progress()` in
`land_flow.py` use unsafe read-modify-write (`Path.write_text()` truncates
first) — both must be fixed to use atomic write-then-rename.

### Crash recovery

On restart, the orchestrator reads `.close-progress` and skips completed
steps. The lifecycle state machine provides the major phase (`META_STATE`
from ctx.py). `.close-progress` provides fine-grained position within
a phase.

| Crash point | Recovery |
|-------------|----------|
| Mid-mechanical step | Re-run from last completed sub-step |
| After yielding ACTION but before LLM acts | Re-yield the same action |
| After LLM acts but before validation | Validate on next call |
| Between validation and progress write | Atomic write-then-rename eliminates this window |

### Concurrent protection (D12)

The lifecycle state machine's compare-and-swap in `commit_transition()`
prevents concurrent sessions. A second session attempting `work_end`
while one is in progress fails at the lifecycle level — no file-level
locking on `.close-progress` is needed.

---

## Error and retry policy (D10)

### Two-tier policy

**Judgment steps** (review, sweep sub-steps, squash, trajectory):
1. Re-yield with `REASON=` explaining what failed validation
2. Max 3 attempts
3. After 3 failures: yield `ACTION=user_input CONTEXT=step_failed STEP=<name>`
4. User decides: skip, retry, or abort
5. Never silently skip judgment work

**Mechanical steps** (push, merge, stamp, promotion):
1. Re-yield with `REASON=`
2. Max 3 attempts
3. After 3 failures: skip with warning, record in `.close-progress`
4. `verify_slot_close.py` catches gaps downstream

### Rationale

`verify_slot_close.py` checks: merged, stamped, landing SHA, pushed,
issues closed, landed marker, original sync, archive status. It does
NOT check: review completion, sweep outputs, trajectory, findings
resolution. Silently skipping judgment steps leaves gaps no downstream
gate catches.

---

## Slot mode (D9)

Single orchestrator, same sequence for slots and non-slots. The existing
scripts handle internal mode differences (two-hop transport, `.landed`
marker). The orchestrator has routing conditionals at 5 decision points:

| Point | Slot mode | Non-slot mode |
|-------|-----------|---------------|
| Phase A complete marker | Write `.phase-a-complete` | Skip |
| Land script | `slot_manager.py merge-slot` | `work_end_execute.py land` |
| Verify args | `slot_dir=<SLOT_PATH>` | Omit |
| Archive | `work_end_execute.py archive-slot` | Skip |
| Scaffold cleanup | Keep `.plan` (main mode: drained) | Remove `.plan` |

This is routing — which script, which arguments — not duplicated
business logic.

---

## Forward-only Execute (D11)

Post-promotion states are forward-only. The lifecycle state machine
rejects `abort_close` from `closing:promoted` onward. Partial failures
recover by retrying the failed step forward, never by rolling back.

Rolling back pushed content requires force-push to shared remotes —
more dangerous than forward completion. `verify_slot_close.py` detects
incomplete forward progress. `.execute-progress` enables retry from the
last completed sub-step.

---

## Existing scripts (D6)

The orchestrator calls the existing 17 work-end scripts without
modification. They are the proven mechanical layer. The orchestrator
adds sequencing and validation.

### Scripts called by the orchestrator

| Script | Phase | What it does |
|--------|-------|-------------|
| `work_end_context.py` | Pre-close | Context + preconditions |
| `work_end_execute.py promote` | Execute | Artifact promotion |
| `work_end_execute.py rebase` | Execute | Rebase onto base branch |
| `work_end_execute.py land` | Execute | Push + stamp (non-slot) |
| `work_end_execute.py close-issues` | Execute | Close GitHub issues |
| `work_end_execute.py archive-slot` | Close | Archive slot to attic |
| `slot_manager.py merge-slot` | Execute | Land (slot mode) |
| `verify_slot_close.py` | Verify | Defence-in-depth audit |
| `close_artifacts.py` | Execute | Called by promote internally |
| `land_branch.py` | Execute | Called by land internally |
| `land_flow.py` | Execute | Shared land flow |
| `verify_stamp.py` | Execute | Called by land internally |
| `verify_promotion.py` | Execute | Called by promote internally |
| `branch_cleanup.py` | Close | Checkout main, cleanup scaffold |
| `loose_ends_sweep.py` | Review | Loose ends detection |
| `close_report.py` | Close | Summary generation |
| `lifecycle.py` | All | State machine transitions |

### New script

| Script | Purpose |
|--------|---------|
| `work_end_orchestrator.py` | Close sequence orchestrator |

### Modified scripts (crash-safety fix)

| Script | Change |
|--------|--------|
| `work_end_execute.py` | `write_progress()` → atomic write-then-rename |
| `land_flow.py` | `_write_progress()` → atomic write-then-rename |

---

## Orchestrator sequence

The full mechanical + judgment sequence, showing what Python does silently
vs what it yields to the LLM:

```
PRE-CLOSE (SKILL.md, not orchestrator):
  work_end_context.py → preconditions → lifecycle transition to closing:review

ORCHESTRATOR SEQUENCE:

Phase: closing:review
  [yield] ACTION=review → LLM runs code-review, branch-audit, forcing function
  [yield] ACTION=sweep_config → LLM presents toggle, reports selections
  [yield] ACTION=forage (if selected)
  [yield] ACTION=protocol (if selected)
  [yield] ACTION=update_claude_md (if selected)
  [yield] ACTION=impl_doc_sync (if selected)
  [yield] ACTION=adr (if selected)
  [yield] ACTION=write_content (if selected)
  [mechanical] lifecycle transition: review_pass → closing:verified

Phase: closing:verified
  [mechanical] work_end_execute.py promote
  [mechanical] lifecycle transition: promote_pass → closing:promoted

Phase: closing:promoted
  [mechanical] work_end_execute.py rebase
  [yield if conflicts] ACTION=review DIFF_RANGE=<conflict-resolution-commits>
  [yield] ACTION=squash → LLM classifies commits, writes plan files
  [mechanical] work_end_execute.py land (applies squash, pushes, stamps)
  [mechanical] lifecycle transitions: push_pass → merge_pass → stamp_pass

Phase: closing:stamped
  [yield] ACTION=trajectory
  [yield] ACTION=user_input CONTEXT=arc42_scan (if ARC42STORIES.MD exists)
  [yield] ACTION=user_input CONTEXT=session_rename
  [yield] ACTION=user_input CONTEXT=garden_feedback (if GE-IDs in session)
  [yield] ACTION=user_input CONTEXT=notes (if .notes/ exists)

VERIFY:
  [mechanical] verify_slot_close.py
  [yield if failed] ACTION=verify_recover → LLM presents failures
  [mechanical] work_end_execute.py close-issues

CLOSE:
  [mechanical] work_end_execute.py archive-slot (slot mode)
  [mechanical] branch_cleanup.py checkout-main
  [mechanical] branch_cleanup.py cleanup-scaffold
  [mechanical] lifecycle transition: cleanup_pass → idle (or cleanup_main → drained)
  [yield] ACTION=complete SUMMARY=...
```

---

## Testing strategy

The orchestrator requires its own test suite — the risk shifts from
"LLM skips steps" (observable) to "orchestrator mis-sequences steps"
(opaque). Tests must cover:

### Orchestrator tests (`test_work_end_orchestrator.py`)

| Test | Assert |
|------|--------|
| Full sequence (branch mode) | All mechanical steps called in order, correct actions yielded |
| Full sequence (slot mode) | Routing conditionals fire correctly for 5 decision points |
| Crash recovery: resume from each progress state | Completed steps skipped, next action yielded |
| Crash recovery: `.close-progress.tmp` exists (mid-write crash) | Old `.close-progress` used, `.tmp` ignored |
| Validation: write-content produces valid blog | `ACTION=complete` after validation passes |
| Validation: write-content produces empty file | Re-yield with `REASON=` |
| Validation: 3 consecutive failures on judgment step | `ACTION=user_input CONTEXT=step_failed` |
| Validation: 3 consecutive failures on mechanical step | Skip-with-warning, continues |
| Sweep config: user deselects items | Deselected sub-steps never yielded |
| Sweep config: all items off | Jumps to post-sweep mechanical steps |
| Post-rebase conflict | `ACTION=review` with scoped `DIFF_RANGE=` |
| Verify failure | `ACTION=verify_recover` yielded |
| Concurrent session | Lifecycle `ConcurrentModification` raised |
| Main mode | Branch-specific steps skipped (rebase, squash, stamp) |

### Crash-safety tests

| Test | Assert |
|------|--------|
| `write_progress` atomic | Process kill mid-write preserves old file |
| `write_progress` round-trip | Write then read produces identical dict |
| `.close-progress` + `.execute-progress` coexistence | Both files independent |

### Integration with existing test suites

Existing tests (`test_work_end_execute.py`, `test_land_flow.py`,
`test_verify_slot_close.py`) continue to test individual scripts.
Orchestrator tests mock script calls and test sequencing logic.

---

## Implementation order

1. **Crash-safety fix** — atomic write-then-rename in `work_end_execute.py`
   and `land_flow.py`. Low risk, immediate value, no dependency.

2. **`work_end_orchestrator.py`** — the orchestrator script with progress
   tracking, action yielding, and validation. Tests first (TDD).

3. **SKILL.md rewrite** — replace 660-line skill with ~20-line loop plus
   pre-close context handling.

4. **Haiku validation** (optional, deferred) — add semantic validation
   for write-content and ADR when heuristic checks prove insufficient.

Step 1 can ship independently. Steps 2-3 ship together.

---

## Files changed

### New files

| File | Purpose |
|------|---------|
| `work-end/work_end_orchestrator.py` | Close sequence orchestrator |
| `tests/test_work_end_orchestrator.py` | Orchestrator test suite |

### Modified files

| File | Change |
|------|--------|
| `work-end/SKILL.md` | Rewrite to ~20-line loop + pre-close |
| `work-end/work_end_execute.py` | `write_progress()` → atomic write-then-rename |
| `work-end/land_flow.py` | `_write_progress()` → atomic write-then-rename |

### Unchanged files

All 17 existing work-end scripts remain unchanged. The orchestrator
calls them as-is.

---

## Risks

**Orchestrator sequencing bugs:** The orchestrator replaces one failure
class (LLM skip) with another (script mis-sequence). Mitigated by TDD,
the orchestrator test suite, and `verify_slot_close.py` as defence-in-depth.

**Haiku API availability:** Graceful degradation — heuristic-only validation
when Vertex AI is unavailable. Close sequence never blocks on API.

**Progress file corruption:** Atomic write-then-rename eliminates the
truncation-then-write crash window. Same pattern as `lifecycle.py`.

**Context loss across yield points:** The LLM's conversation context
persists across orchestrator calls. Forage discoveries inform protocol,
protocol informs write-content. No context is lost.

---

## References

- Issue #271 — audit evidence (8 failures across 4 slots)
- `docs/specs/issue-151-mechanise-llm-steps/2026-08-03-mechanise-llm-steps-design.md` — prior mechanisation spec
- `docs/specs/2026-08-05-work-end-restructure-and-slot-audit-design.md` — work-end restructure spec
- `project/lifecycle.py` — state machine, `write_state()` atomic pattern
- `work-end/work_end_execute.py` — existing Execute orchestrator, `write_progress()` crash-safety issue
- `work-end/land_flow.py` — shared land flow, `_write_progress()` crash-safety issue
- `work-end/verify_slot_close.py` — verification gate (check inventory)
- GE-20260821-ebba3b — work-end atomicity and stamp failures
- GE-20260821-e9c59e — worklog.db as lifecycle source of truth
- `docs/protocols/externalised-scripts-require-tests.md` — test requirements
- `docs/protocols/evidence-before-claims.md` — verification discipline
- [Oracle: AI Agent Loop Architecture](https://blogs.oracle.com/developers/what-is-the-ai-agent-loop-the-core-architecture-behind-autonomous-ai-systems)
- [Google Cloud: Agentic AI Design Patterns](https://docs.google.com/architecture/choose-design-pattern-agentic-ai-system)
- [GitHub: AgenticStateMachines](https://github.com/adamterlson/AgenticStateMachines) — FSM patterns for agent control
- [Claude Code Workflows](https://code.claude.com/docs/en/workflows) — deterministic orchestration precedent

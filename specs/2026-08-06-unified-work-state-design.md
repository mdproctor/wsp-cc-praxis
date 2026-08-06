# Unified Work State — Design Spec

**Issue:** TBD (file after spec approval)
**Addresses:** #118 (Evaluate splitting HANDOFF.md roles: history, state, and planning)
**Date:** 2026-08-06
**Status:** Draft

## Problem

The work lifecycle system has 13 state locations tracking overlapping facts
with no consistency guarantees between them. Five places track "is this issue
done?" Four track "is this branch closed?" Three track "what should I work on
next?" Each is maintained independently. When external actors change state
(another session closes an issue, another context merges a branch), the caches
go stale silently. Validation is scattered across skills that run at different
lifecycle moments, covering different slices with different logic.

The WI 490 case is representative: a HANDOFF.md item described a casehub
branch as "not merged, not stamped" for days after the branch was already
merged. No validation path existed to catch it — the resume cross-check only
validates GitHub issue state, not git branch state.

**The 13 state locations:**

"Is this issue done?" — five locations:
1. `.plan` `[x]` markers (branch queue)
2. `.meta` `covers:` field (completed issue list)
3. HANDOFF.md What's Left (negative signal: absent = possibly done)
4. HANDOFF.md What's Next (implicit: items here are future work)
5. Handover resume cross-check (Step R3) — validates `#N` against GitHub

"Is this branch closed?" — five locations:
6. EPIC-CLOSED.md marker file on workspace branch
7. Stamp commit on project branch (`chore: branch closed`)
8. Stamp commit on workspace branch
9. `verify_slot_close.py` (5 checks: merged, stamped, landing SHA, pushed, workspace stamped)
10. `audit_slot_merges.py` (independent merged/stamped checks)

Cross-cutting validation that re-derives the above:
11. Handover epic hygiene checks 7+8 (unrecovered artifacts, unstamped branches)
12. `work_router.py` HANDOFF reference detection (infers branch liveness)
13. `.meta` `state:` closing:* sequence (tracks closure progress in-session)

Each is maintained independently. When external actors change state (another
session closes an issue, another context merges a branch), cross-location
consistency is never re-established until a validation path happens to run.

**Root cause:** The system caches state that has authoritative sources
elsewhere (GitHub for issues, git for branches, .plan for queue position),
then adds reactive validation to detect cache staleness. The fix is to stop
caching and start deriving — each fact has one source of truth, everything
else queries it.

## Relationship to Prior Specs

**Supersedes** `2026-08-05-work-end-restructure-and-slot-audit-design.md` on
two specific points:

1. **EPIC-CLOSED.md** — the work-end restructure spec creates it in Step 5.
   This spec eliminates it. The stamp commit is the single closure signal.
2. **`verify_slot_close.py` as standalone gate** — the work-end restructure
   spec designs it as the verification gate for work-end. This spec absorbs
   its check functions into `work_health.py --scope close`. The script
   continues to exist as a library; its CLI entry point is deprecated.

The work-end restructure spec's 5-step architecture (Context → Sweep →
Execute → Verify → Close), its scripts (`work_end_context.py`,
`work_end_execute.py`), and its multi-repo slot close fix are orthogonal
and coexist. `work_health.py --scope close` composes with the existing
`verify_slot_close.py` check functions — it calls them, it doesn't
reimplement them.

**Implementation order:** The work-end restructure can proceed first. When
this spec's Phase 6 lands, work-end's Step 5.1 (`create-epic-closed`) is
removed. When Phase 9 lands, work-end's Step 4 invokes `work_health.py
--scope close` instead of `verify_slot_close.py` directly.

## Design

Four components, designed to be implemented in phases but forming a unified
whole.

### Component 1: `is_closed(branch)` predicate

A single function in `lifecycle.py` that answers "is this branch done?"
definitively. Replaces five different closure-checking implementations.

**Signature:**

```python
def is_closed(project: str, branch: str,
              workspace: str | None = None,
              base_branch: str = "main") -> ClosureState
```

**Returns** a `ClosureState` (enum or named object):

| State | Meaning | Criteria |
|-------|---------|----------|
| `CLOSED` | Fully done | Merged to main + stamped |
| `MERGED_UNSTAMPED` | Content landed, no stamp | Zero non-stamp commits ahead of main, no stamp commit |
| `STAMPED_UNMERGED` | Stamp exists, content not landed | Stamp commit present but non-stamp commits ahead of main |
| `OPEN` | Neither merged nor stamped | Non-stamp commits ahead of main, no stamp |
| `DELETED` | Branch doesn't exist | `git branch --list` returns empty |

**Checks (all local git, sub-second):**

1. Branch exists? `git -C <project> branch --list <branch>`
2. Commits ahead of main? `git -C <project> log --oneline <base>...<branch>`
   — filter out commits starting with `chore: branch closed`
3. Stamp commit? Last commit starts with `chore: branch closed`
4. Landing SHA valid? If stamp contains `landed as <SHA>`,
   run `git merge-base --is-ancestor <SHA> <base>`
5. Workspace branch? If workspace provided, check workspace branch
   with same logic

**Replaces:**
- `verify_slot_close.py` checks 1-3 (branch_merged, branch_stamped, landing_sha)
- `audit_slot_merges.py` merged/stamped logic
- Handover epic hygiene checks 7+8 (unrecovered artifacts, unstamped branches)
- Implicit closure checks in work_router.py

**EPIC-CLOSED.md eliminated.** The stamp commit on the project branch is the
single closure signal. Work-end stops creating EPIC-CLOSED.md. All tools that
checked for it use `is_closed()` instead. Workspace-side closure is determined
by checking the workspace branch with `is_closed()`.

**Slot topology:** `is_closed()` always operates on the authoritative
project repo — the original on-disk repo, not a slot clone. Slot clones
are ephemeral workspaces whose `origin` points to the original repo, not
GitHub. Callers in slot mode pass the original repo path. The git checks
(`branch --list`, `log`, `merge-base`) all work correctly against the
original repo. `work_health.py` resolves the original repo from `.slot`
metadata before calling `is_closed()`.

**Migration:** Existing branches with EPIC-CLOSED.md but no stamp can be
detected by `is_closed()` returning `MERGED_UNSTAMPED` and auto-stamped by
`work_health.py`.

### Component 2: Unified validation — `work_health.py`

Single check registry replacing scattered validation across handover epic
hygiene, resume cross-check, ARC42 stale scan, verify_slot_close.py, and
audit_slot_merges.py.

**Check registry:**

| Check | What it validates | Scopes | Auto-fix |
|-------|------------------|--------|----------|
| `plan_state` | .plan items vs GitHub issue state (batch API call) | entry, wrap | Yes — mark closed items in .plan, report changes |
| `branch_closure` | `is_closed()` on all known branches | entry, wrap, close | Yes — stamp MERGED_UNSTAMPED; report STAMPED_UNMERGED |
| `pause_stack` | Each .pause-stack entry's branch exists | entry | Report — user decides |
| `meta_consistency` | .meta state matches actual branch/git state | entry | Report |
| `workspace_alignment` | Workspace and project on expected branches | entry | Report |
| `arc42_refs` | ARC42STORIES.MD issue refs vs GitHub | wrap | Yes — update on confirmation |
| `slot_state` | Active slots: repos exist, branches exist, .slot valid | entry | Report |
| `main_divergence` | Local main vs origin/main | entry, wrap | Report |
| `stale_branches` | Open branches with no commits in 7+ days | wrap | Report |
| `journal_anchors` | Mid-epic journal entries have §Section anchors | wrap | Report |
| `dirty_main` | Project main working tree has staged/unstaged changes | entry | Report |
| `artifact_recovery` | Closed branches with unrecovered artifacts (blogs, specs not on main) | wrap | Report + offer cherry-pick |
| `partial_pause` | `.pausing` intent file present | entry | Report + offer recovery |
| `partial_resume` | `.resuming` intent file present | entry | Report + offer recovery |

**Invocation:**

```bash
python3 project/work_health.py --scope entry  --project P --workspace W
python3 project/work_health.py --scope wrap   --project P --workspace W
python3 project/work_health.py --scope close  --project P --workspace W --branch B
```

**Output format:**

```
CHECK=plan_state STATUS=changed DETAIL=#142 now CLOSED, #155 now CLOSED
CHECK=branch_closure STATUS=fix DETAIL=issue-142-foo stamped (was MERGED_UNSTAMPED)
CHECK=pause_stack STATUS=ok
CHECK=meta_consistency STATUS=ok
FIXED=1 WARNINGS=0 ERRORS=0
```

**Auto-fix policy:** Safe items (stamping a merged branch, removing a closed
issue from .plan) are fixed automatically. Risky items (discarding a pause
stack entry, archiving a slot) are reported for user decision.

**`plan_state` batch validation:** One `gh issue list --repo R --state all
--json number,state,title --limit 500` call returns all issue state. Compare
against every `#N` in .plan. Sub-second for most repos.

**Integration points:**
- `project` skill (session start) → `--scope entry`
- `handover` skill (wrap) → `--scope wrap` (replaces epic hygiene + ARC42 scan)
- `work-end` skill (close) → `--scope close` (replaces/wraps verify_slot_close.py)

### Component 3: `.plan` as single orchestration layer

`.plan` becomes the single local cache for work state. Exists on main
(global queue) and on branches (active work queue).

#### Main `.plan` — global curated queue

Lives at `$WORKSPACE/design/.plan` on the `main` branch. A curated,
ordered subset of open GitHub issues — not a parallel backlog.

**Relationship to GitHub issues:** Every item in Main `.plan` is a GitHub
issue reference (`#N`). GitHub is authoritative for issue state (open/closed);
`.plan` is authoritative for priority ordering. Items that are not GitHub
issues cannot appear. `work_health.py` validates every `#N` against GitHub
on session start and auto-removes closed items.

**Curation model:** Items are added explicitly — during triage ("add #N to
the queue"), when filing new issues ("file and queue"), or when the `work`
skill on main offers to seed from open GitHub issues. Items are removed
automatically when closed (via `plan_state` check) or manually by the user.
Priority order is changed by the user or LLM during planning. This replaces
the current model where HANDOFF.md's What's Left/What's Next are
synthesised from conversational context at session end — a process that
loses items when context is stale and creates items that don't correspond
to GitHub issues.

**Branch .plan vs Main .plan:** Same file path (`$WORKSPACE/design/.plan`),
different git branches. On main, `plan_manager.py detect()` finds Main
`.plan`. On a feature branch, it finds Branch `.plan`. These serve different
purposes: Main `.plan` is the project priority queue; Branch `.plan` is the
issue iteration queue for the current branch's work. They use the same
format and parser. No ambiguity — git branch determines which is active.

**No flow-back on branch close:** Intentional. Automatic flow-back creates
the same accumulation problem as HANDOFF.md What's Left — items pile up
without curation. When a branch closes, uncompleted items are already
open GitHub issues. The user decides which to add to Main `.plan` during
their next triage session. This is a curation discipline, not a gap.

#### Format

```markdown
# Work Plan — soredium

## Queue
- [x] #142 — some completed issue
- [ ] #123 — active issue ← active
- [ ] #155 — another issue

## Session State
Current: #123
Started: 2026-08-06
Last wrap: 2026-08-06T14:30:00Z
```

No state cache in the file. GitHub issue state is queried live by
`work_health.py` on session start (batch API call) and consumed in-session.
The `[x]` markers in Queue are a completion record written by
`plan_manager.py` when an issue advances — not a cache of GitHub state.
`plan_manager.py` is the sole writer of `.plan` content (Queue and Session
State sections).

#### Resume display

Instead of reading HANDOFF.md for work items, resume reads main `.plan` and
runs `work_health.py --scope entry` to validate issue state live. The
combined result is rendered as a human-readable summary:

```
Queue (3 items, 1 complete, 1 active):
  ✅ #142 — some completed issue
  → #123 — active issue (current)
  ⚠️  #155 — now CLOSED on GitHub — auto-removed

  work_health: 1 item removed (CLOSED on GitHub)
```

#### What `.plan` replaces

| Previously in | Now in | Mechanism |
|---------------|--------|-----------|
| HANDOFF.md What's Left | Main `.plan` or GitHub issues | Trailing items become issues, enter .plan |
| HANDOFF.md What's Next | Main `.plan` | Curated queue with priority |
| .meta `covers:` | Derived from `.plan` `[x]` items | `reconcile_covers()` removed |
| HANDOFF.md Queue Progress | `.plan` Session State | Already exists |

#### Branch close — no flow-back

When a branch closes, its `.plan` is marked closed (`write_plan_closed`).
Uncompleted items remain as open GitHub issues. Main `.plan` is independently
curated — branch close does not automatically append items to it.

#### HANDOFF.md shrinks to session context

```markdown
# HANDOFF — <project>

## Last Session
2-3 lines: what was done, what was tried, key reasoning.

## Immediate Next Step
Single specific action.

## Cross-Module
Only if active cross-repo blockers exist with tracked issues.
Omit entirely if none.

## References
Paths only, no content inline.
```

Target: under 200 tokens. No work tracking. No backlog. Session context
and narrative reasoning only — the "why" that connects commits and .plan
items into a coherent story for the next session.

**Commit mechanism simplifies:** HANDOFF.md is write-once-per-session at
wrap time. The resume path no longer mutates it (closed-issue removal moves
to `.plan` validation via `work_health.py`). This eliminates the main source
of mid-session HANDOFF.md churn and reduces the commit-to-main dance to a
single write at session end.

### Component 4: Crash recovery for work-pause and work-resume

Intent marker files track in-progress operations. Detected by `work_health.py`
on next session start.

#### `.pausing` intent file

Written to `$WORKSPACE/design/.pausing` before pause starts. Updated as each
step completes. Removed on successful pause.

```yaml
branch: issue-123-foo
started: 2026-08-06T14:30:00Z
wip_project: pending|done
wip_workspace: pending|done
stack_push: pending|done
checkout_main: pending|done
```

**Recovery on detection:**
- Offer to complete pause (push stack, checkout main)
- Or abort (reset WIP commits, stay on branch)

#### `.resuming` intent file

Written to `$WORKSPACE/design/.resuming` before resume starts.

```yaml
branch: issue-123-foo
started: 2026-08-06T14:30:00Z
stack_pop: pending|done
checkout: pending|done
rebase: pending|done
wip_reset: pending|done
```

**Recovery on detection:**
- Offer to complete resume (checkout, rebase, reset WIP)
- Or abort (re-push stack entry, stay on main)

**Why intent files, not lifecycle states:** The alternative is adding
`pausing` and `resuming` as transient states to the lifecycle state machine
(analogous to `transitioning` and `closing:*`). This was evaluated and
rejected for a specific reason: the lifecycle state machine provides
coarse-grained phase tracking (one state per phase), but pause/resume
operations have multiple independently-failable sub-steps (WIP commit on
project, WIP commit on workspace, push stack, checkout main). A lifecycle
state `pausing` would tell you the operation started but not which sub-steps
completed. Intent files track per-step completion — the same pattern as
`.execute-progress` for the close sequence. The close sequence uses
lifecycle states for the coarse phases AND `.execute-progress` for per-repo
granularity within phases. Intent files are the pause/resume equivalent of
`.execute-progress` — not a competing mechanism, but the same architecture
applied to a different operation.

## Implementation Phases

| Phase | What | Depends on |
|-------|------|-----------|
| 1 | `is_closed()` predicate in lifecycle.py + tests | Nothing |
| 2 | `work_health.py` with entry-scope checks (plan_state, branch_closure, pause_stack, meta_consistency) | Phase 1 |
| 3 | .plan batch validation integration + human-readable resume display | Phase 2 |
| 4 | Main .plan support — curated global queue | Phase 3 |
| 5 | HANDOFF.md simplification — remove What's Left, What's Next | Phase 4 |
| 6 | Remove EPIC-CLOSED.md — work-end stops creating, all tools use is_closed() | Phase 1 |
| 7 | Remove .meta covers — derive from .plan | Phase 3 |
| 8 | Crash recovery — .pausing/.resuming intent files | Phase 2 |
| 9 | Wrap-scope and close-scope checks in work_health.py — replace handover epic hygiene, verify_slot_close.py | Phase 2 + 6 |
| 10 | Update audit_slot_merges.py to use is_closed() | Phase 1 |

Phases 1, 6, 10 can run in parallel (is_closed + dependents).
Phases 2-5 are sequential (each builds on the previous).
Phase 8 is independent after Phase 2.
Phase 9 is independent after Phases 2 + 6.

## Migration

**EPIC-CLOSED.md:** Existing branches with EPIC-CLOSED.md are handled by
`is_closed()` — it checks stamp commits, not marker files. During Phase 6,
any branch with EPIC-CLOSED.md but no stamp gets auto-stamped by
`work_health.py` (auto-fix safe: branch is already merged).

**HANDOFF.md:** Phase 5 changes the handover skill to write the slim format.
Old-format HANDOFF.md files are still readable — the resume path gracefully
handles missing sections.

**.meta covers:** Phase 7 stops writing to covers. Any skill that read covers
now reads .plan instead. `reconcile_covers()` is removed.

## What Gets Deleted

| Artifact | Replaced by |
|----------|------------|
| EPIC-CLOSED.md (creation + all checks) | `is_closed()` predicate |
| HANDOFF.md What's Left section | Main `.plan` / GitHub issues |
| HANDOFF.md What's Next section | Main `.plan` |
| HANDOFF.md Queue Progress section | `.plan` Session State |
| .meta `covers:` field + `reconcile_covers()` | `.plan` `[x]` items |
| Handover epic hygiene (8 checks) | `work_health.py --scope wrap` |
| Handover resume issue cross-check | `work_health.py --scope entry` |
| Handover ARC42 stale scan (Step 2c) | `work_health.py --scope wrap` |
| verify_slot_close.py (standalone) | `work_health.py --scope close` |

## Open Design Points

**Main .plan bootstrapping:** When main `.plan` doesn't exist yet, `work`
on main should offer to create one — either empty (curate from scratch),
seeded from GitHub open issues, or migrated from HANDOFF.md What's Next
during the transition. Exact UX TBD during Phase 4 implementation.

**Close scope is a hard gate:** `work_health.py --scope close` inherits
verify_slot_close.py's hard-gate semantics. Unlike entry/wrap scope
(report + auto-fix), close scope returns `VERIFIED=yes|no` and work-end
blocks on `VERIFIED=no`. The check registry distinguishes advisory checks
(entry/wrap) from gate checks (close).

## Success Criteria

- Single `is_closed()` call answers "is this branch done?" — no other tool
  implements its own closure check
- Session start runs `work_health.py --scope entry` in under 3 seconds
  (one GitHub API call + local git checks)
- .plan on main is the single source of work priority — HANDOFF.md contains
  zero work items
- Interrupted pause/resume detected and recovery offered on next session start
- EPIC-CLOSED.md no longer created or checked anywhere
- .meta covers no longer written; .plan is the single completion record
- HANDOFF.md under 200 tokens

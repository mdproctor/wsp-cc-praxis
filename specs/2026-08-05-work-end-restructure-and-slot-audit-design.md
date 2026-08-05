# Work-end Restructure and Multi-repo Slot Audit

**Date:** 2026-08-05
**Status:** Draft
**Scope:** soredium work-end skill, slot close infrastructure, legacy slot remediation

---

## Problem Statement

### Multi-repo slot close bug

work-end's slot close sequence processes only the primary repo. Non-primary
repos in multi-repo slots have their content merged (often in separate
sessions) but are never stamped. An audit of 73 slots (14 multi-repo) found
27 with problems — all stamp-related, no confirmed data loss. The root cause:
the SKILL.md describes per-repo operations using `<slot>/<repo>` syntax, but
the LLM executes only the primary repo and moves on. There is no mechanical
gate that checks all repos were processed.

### Skill complexity

work-end has ~20 steps across ~1400 lines. The LLM frequently skips steps,
complains about length, and makes judgment calls that should be mechanical.
Promotion, verification, and cleanup are scattered across steps rather than
consolidated. The inventory/plan/approval cycle (Steps 4-7) is unnecessary
ceremony — the scripts already know what to promote and where.

---

## Deliverable 1: Legacy Slot Audit Remediation

### Scope

Remediate the 27 problem slots identified by `audit_slot_merges.py`. No
content merging — the audit confirmed content reached main via rebase in
every spot-checked case. This is stamp remediation and documentation only.

### Categories

**MERGED but UNSTAMPED (~15 branches):** Content on main, branch never
stamped. Retroactively stamp with:
```
chore: branch closed — landed as <SHA> on main
  no-issue: retroactive stamp from slot audit
```
Where `<SHA>` is found by matching the branch's commit messages against
main's log (`git log --oneline --grep="<subject>" main`).

**Pre-squash landing SHAs (~5 branches):** Stamps exist but reference SHAs
that were rewritten during squash (garden entry GE-20260721-94263a documents
this). Content IS on main under different SHAs. No fix needed — document as
known limitation. The redesigned Execute step already orders stamping after
squash to prevent recurrence.

**Active slots with WIP commits (slots 54, 86):** Still in progress. No
action — flagged for awareness only.

### Implementation

1. Enhance `audit_slot_merges.py` with `--fix` mode:
   - For MERGED-but-UNSTAMPED: find the landing SHA via commit message
     matching, stamp the branch, push the stamp
   - For pre-squash SHAs: log as "known — content verified on main"
   - Produce a summary report to stdout

2. Run `audit_slot_merges.py --fix` against `/Users/mdproctor/claude/casehub`

3. File soredium issue for Deliverable 2

### Success criteria

- All MERGED-but-UNSTAMPED branches stamped
- Summary report shows per-slot disposition
- No content merging performed
- Issue filed for work-end restructure

---

## Deliverable 2: Work-end Restructure

### Design: 5 steps

```
Context (mechanical) → Sweep (LLM) → Execute (mechanical + subagents) → Verify (mechanical) → Close (mechanical)
```

### Step 1 — Context

**Nature:** Mechanical script — one call, pass/fail.

**Absorbs:** Pre-conditions (0, 0b, 0c, 1-5), Step 1 (branch_recon.py),
Step 2 (Flyway), Step 3 (routing resolution).

**Implementation:** A new `work_end_context.py` script that:

1. Runs `ctx.py` internally for path resolution
2. Checks all 6 pre-conditions:
   - Branch divergence (`BRANCH_MISMATCH`)
   - Queue gate (plan_manager.py `detect`)
   - Issue-complete emission (reconcile_covers + complete_active_issue)
   - Pause stack check
   - `.meta` existence and graceful degradation
   - Clean working tree (workspace + project base branch)
3. Runs `branch_recon.py` for branch state
4. Runs Flyway re-scan
5. Resolves routing via `project/routing.py`

**Output:** Single JSON object with all context values, pre-condition
results, branch state, and routing table. If any pre-condition fails,
exit code 1 with specific error.

**Scripts removed:** `phase_a_complete.py` (marker write absorbed into
Execute), `phase_b_gate.py` (verification logic absorbed into Verify).

**Caveat:** `slot_manager.py` depends on `.phase-a-complete` marker in
`merge_slot()`, `scan_ready()`, and `list_slots()`. Execute must continue
writing this marker after promotion succeeds until `slot_manager.py` is
updated to use the new Verify output.

### Step 2 — Sweep

**Nature:** LLM-driven. Creates files in the workspace branch.

**Four items (down from six):**

| Item | Detection | Execution | Output |
|------|-----------|-----------|--------|
| Knowledge capture | LLM scans session context | forage + protocol (combined sweep) | Garden entries, protocol files |
| ADR | LLM checks for architectural decisions | LLM writes ADR content | ADR files in workspace `adr/` |
| Doc sync | LLM detects convention/doc drift | update-claude-md + impl-doc-sync | Updated CLAUDE.md, docs |
| write-content (diary) | Always runs last | LLM writes entry | Blog entry in workspace `blog/` |

**Rules:**
- All items default to ON. The LLM never recommends skipping.
- write-content runs last so it can reference forage/protocol submissions.
- Items 1 and 4 are session-bound — cannot be deferred.
- A blog is a permanent published artifact for community engagement. HANDOFF
  and journal serve different purposes and do not replace the blog.

**Not in sweep:** Journal-entry is not a sweep item. Journal merge is
conditional within Execute (see Step 3).

### Step 3 — Execute

**Nature:** Mechanical orchestrator script with LLM subagent calls for
code review and squash analysis.

**New script:** `work_end_execute.py` — the core orchestrator.

**Sequence (per-repo in slot mode):**

```
1. Code review          — LLM subagent gate; blocks on critical findings
2. Promote artifacts    — workspace branch → workspace main → project main
                          (per routing config; skip types routed to "none")
3. Publish blogs        — workspace main → destination (blog-routing.yaml)
4. Archive plans        — plans/ → plans/attic/<branch>/
5. Close issues         — all COVERS issues via gh (ONCE, not per-repo)
6. Journal merge        — conditional: if journal entries exist, merge to
                          ARC42STORIES.MD; if empty, skip silently
7. Rebase               — branch onto base branch
8. Squash               — LLM subagent for commit classification
9. Build verification   — Java: mvn install; others: skip
                          (PRE-PUSH — if build fails, nothing pushed yet)
10. Push                — fork first (mandatory), then blessed repo (prompt)
11. Stamp               — verify_stamp.py first, then stamp commit
12. Write .phase-a-complete marker (slot mode compatibility)
```

**Slot mode per-repo looping:**

```python
repos = parse_slot(slot_dir / ".slot")  # from .slot file
primary = repos[0]  # marked (primary) in .slot

for repo in repos:
    # Steps 1-4, 6-12 run per-repo
    execute_repo(repo, context)

# Step 5 runs ONCE after all repos
close_issues(context.covers, context.issue_repo)
```

Issue closing runs once after all per-repo work to avoid duplicate
`gh issue close` calls.

**Existing scripts reused (not rewritten):**

| Script | Called by Execute | Per-repo? |
|--------|------------------|-----------|
| `close_artifacts.py` | Promotion + blog publish | Yes (different project= per repo) |
| `land_branch.py rebase` | Step 7 | Yes |
| `land_branch.py push` | Step 10 | Yes |
| `land_branch.py stamp` | Step 11 | Yes |
| `verify_stamp.py` | Called by land_branch.py stamp | Yes (internal) |
| `artifact_promote.py` | Called by close_artifacts.py | Yes (internal) |
| `blog_dest.py` | Called by close_artifacts.py | No (once) |

**close_artifacts.py per-repo calling:** Pass `covers=` only on the final
repo call (or split issue closing out of close_artifacts.py entirely).
The script has no global state — safe to call N times with different
project paths.

**land_branch.py per-repo calling:** Each subcommand takes a project path
as its first positional arg. No shared state between calls. The
`cmd_push()` workspace stamp check is read-only — multiple repos can
verify the same `.artifacts-promoted` stamp.

**merge_slot() bypass:** The new Execute orchestrator calls `land_branch.py`
directly per-repo instead of going through `slot_manager.py`'s monolithic
`merge_slot()`. `merge_slot()` remains available for other slot operations
but is not used by work-end.

**Code review placement:** Runs as the first action in Execute. Reviews the
post-sweep diff (which is the final state — includes write-content, doc-sync
outputs). If review surfaces critical issues, fixes are applied and review
re-runs before any promotion begins.

**Build verification placement (changed from current):** Runs AFTER
rebase+squash, BEFORE push. Currently Step 8k runs after 8j which includes
push — meaning build failures leave pushed-but-broken code. Pre-push
placement means a failed build has nothing to roll back.

### Step 4 — Verify

**Nature:** Mechanical script — the core fix for the multi-repo bug.

**New script:** `verify_slot_close.py`

**Checks (all mechanically verifiable via git/gh/filesystem):**

```
Per-repo checks (for each repo in .slot, or single repo):
  □ Branch merged to main
    Command: git log main..<branch> --oneline
    Pass: empty output (excluding stamp commit)

  □ Branch stamped
    Command: git log -1 --format=%s <branch>
    Pass: starts with "chore: branch closed"

  □ Landing SHA valid
    Command: git merge-base --is-ancestor <SHA> main
    Pass: exit code 0

  □ Main pushed to remote
    Command: git log origin/main..main --oneline
    Pass: empty output

Workspace artifact checks:
  □ Specs at expected destination (per routing config)
    Command: verify_promotion.py (existing script)

  □ ADRs at expected destination
    Command: verify_promotion.py

  □ Blogs published to destination
    Command: compare workspace blog/ against destination

  □ Plans archived
    Command: ls plans/attic/<branch>/

Issue checks:
  □ All COVERS issues closed
    Command: gh issue view <N> --json state (per issue)

Build check:
  □ Build passed (if applicable)
    Source: recorded by Execute in close_report.py
```

**Output:** `VERIFIED=yes` or `VERIFIED=no` with per-check results.

**Hard gate:** Blocks Step 5 (Close/archive) on any failure. The LLM
cannot rationalize past a script that reads git state across all repos.

**Absorbs logic from:**
- `verify_promotion.py` — artifact destination checks
- `phase_b_gate.py` — stamp checks, issue state checks
- `hygiene_scan.py` — unpublished blog detection, unstamped branch detection
- Slot completion gate (current SKILL.md S9 checklist)

**Protocol compliance:** Satisfies `archive-requires-promotion-verification`
protocol — the `.artifacts-promoted` stamp is checked as part of the
verification pass.

### Step 5 — Close

**Nature:** Mechanical script.

**Sequence:**

1. Write `EPIC-CLOSED.md` on workspace branch
2. Archive slot to `slots/attic/<N>/` (slot mode only)
3. Return both repos to main (checkout + pull)
4. ARC42 stale scan (if ARC42STORIES.MD exists)
5. Write HANDOFF.md to workspace main
6. Session rename suggestion
7. Session close summary (tick-list of all steps)

**Existing scripts reused:**
- `branch_cleanup.py create-epic-closed`
- `branch_cleanup.py checkout-main`
- `close_report.py render`

---

## Testing Strategy

Every promotion and verification operation is a pure function of inputs —
testable in isolation with temp git repo fixtures.

### Test matrix

| Test | Input | Assert |
|------|-------|--------|
| Workspace branch → workspace main | Artifact file on branch | File on main, commit exists |
| Workspace main → project main | Artifact on ws main, routing=project | File in project repo |
| Workspace main → project main | routing=workspace | No change to project |
| Workspace main → project main | routing=none | No change to project |
| Blog → destination | Entry on ws main, blog-routing.yaml | Entry at destination |
| Plan → attic | Plan file | Plan in `plans/attic/<branch>/` |
| Multi-repo slot promotion | 4 repos, mixed routing | Each repo promoted per own config |
| Stamp after squash | Rebased+squashed branch | Stamp SHA matches post-squash HEAD |
| Stamp with pre-squash SHA | SHA rewritten by squash | verify_stamp.py blocks stamp |
| Verify: all pass | Clean state | VERIFIED=yes |
| Verify: missing stamp | One repo unstamped | VERIFIED=no, identifies repo |
| Verify: missing artifact | Spec not at destination | VERIFIED=no, identifies file |
| Verify: unclosed issue | COVERS issue still OPEN | VERIFIED=no, identifies issue |
| Verify: unpushed main | Local ahead of origin | VERIFIED=no, identifies repo |
| Issue close once | 4 repos, covers=5,19 | gh issue close called once |
| Journal conditional | No journal entries | Journal merge skipped silently |
| Journal conditional | Journal entries exist | Journal merge runs |
| Build pre-push | Rebase+squash done, push not | mvn install succeeds on working tree |

`verify_slot_close.py` IS the test assertions extracted into a runtime gate.

### Protocol compliance

Per `externalised-scripts-require-tests`: every new .py script ships with
pytest tests covering happy path, at least two edge cases, and bad argument
handling.

---

## SKILL.md Impact

~1400 lines → ~400 lines. Mechanical complexity moves into scripts.

The SKILL.md describes:
- The 5-step structure and what each does
- Red flags and common pitfalls
- What the LLM does in Sweep (creative judgment)
- How to handle Execute/Verify failures (re-run failing step)
- Slot mode vs single-repo differences

The SKILL.md does NOT describe:
- Per-file promotion mechanics (script handles)
- Routing resolution details (script handles)
- Git command sequences (script handles)
- Verification check details (script handles)

---

## Implementation Order

1. **`audit_slot_merges.py --fix`** — retroactive stamp remediation
2. **`verify_slot_close.py`** — the Verify gate (tests first)
3. **`work_end_context.py`** — Context script (tests first)
4. **`work_end_execute.py`** — Execute orchestrator (tests first)
5. **Work-end SKILL.md rewrite** — 5-step version
6. **`slot_manager.py` update** — remove `.phase-a-complete` dependency (deferred — compatibility marker written by Execute until then)

---

## Scripts Inventory (post-restructure)

### New scripts

| Script | Step | Purpose |
|--------|------|---------|
| `work_end_context.py` | 1 | Unified context + pre-conditions |
| `work_end_execute.py` | 3 | Per-repo orchestrator |
| `verify_slot_close.py` | 4 | Unified verification gate |

### Retained scripts (unchanged)

| Script | Called by | Purpose |
|--------|----------|---------|
| `close_artifacts.py` | Execute | Promotion + blog publish |
| `land_branch.py` | Execute | Rebase + push + stamp |
| `verify_stamp.py` | land_branch.py | Content landing verification |
| `verify_promotion.py` | Verify | Artifact destination check |
| `artifact_promote.py` | close_artifacts.py | Individual promotion ops |
| `blog_dest.py` | close_artifacts.py | Blog destination resolver |
| `workspace_artifacts.py` | close_artifacts.py | Artifact path resolver |
| `branch_recon.py` | Context | Branch state gathering |
| `branch_cleanup.py` | Close | EPIC-CLOSED, checkout-main |
| `close_report.py` | Execute + Close | Report generation |
| `hygiene_scan.py` | Verify (absorbed) | Workspace hygiene checks |
| `common.py` | All | Shared utilities |

### Removed scripts

| Script | Reason |
|--------|--------|
| `phase_a_complete.py` | Marker write moved into Execute |
| `phase_b_gate.py` | Logic absorbed into verify_slot_close.py |

---

## Risks

**merge_slot() bypass:** The new Execute orchestrator bypasses
`slot_manager.py`'s `merge_slot()`. If other code paths call `merge_slot()`,
they continue using the old flow. Verify: grep for `merge_slot` callers
before removing.

**Routing "none" destination:** The routing vocabulary does not support
explicit disable (`none`). For now, absence of files in a category
directory serves as implicit disable. A `none` routing value is a future
enhancement.

**Journal merge conditional:** Journal merge is conditional on entries
existing. `branch_recon.py` still validates journal state — its output
contract (JSON with `journal_validation` field) must remain stable even
when Execute skips the merge.

**Squash SHA ordering:** Stamps must be written AFTER squash (garden entry
GE-20260721-94263a). The Execute sequence enforces this: squash (step 8)
before stamp (step 11). The current SKILL.md already has this ordering
but the per-repo loop makes it explicit.

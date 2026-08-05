# HANDOFF — soredium

## Last Session

Discovered and audited a systematic multi-repo slot close bug: work-end
only processes the primary repo, leaving non-primary repos unstamped.
Audited 73 slots (14 multi-repo), found 27 with problems — all
stamp-related, no data loss. Also audited 6 recent work-ends and found
4 stranded specs on workspace branches that were never promoted to main.

Designed a work-end restructure: 5 steps (Context → Sweep → Execute →
Verify → Close), collapsing ~20 steps / ~1400 lines to ~400 lines.
Adversarial design review (4 dimensions, 43 issues, $37 total, all
resolved) hardened the spec significantly.

Wrote a 10-task implementation plan. Nothing implemented yet.

## Immediate Next Step

File the soredium issue (Task 1 of the plan), then start executing with
`/work` against that issue.

## What to do

**Spec:** `/Users/mdproctor/claude/public/cc-praxis/specs/2026-08-05-work-end-restructure-and-slot-audit-design.md`
**Plan:** `/Users/mdproctor/claude/public/cc-praxis/plans/2026-08-05-work-end-restructure.md`

Execute the plan task-by-task using `executing-plans`. The plan has 10
tasks with a dependency graph. Tasks 4 and 5 can run in parallel.

## Implementation Briefing — Read Before Starting

### What the bug is

work-end's slot close describes per-repo operations (`<slot>/<repo>`) but
the LLM only executes them for the primary repo, then moves on. There is
no mechanical gate that checks all repos were processed. Result: primary
repo merged + stamped, non-primary repos left with content on main (from
separate sessions) but never stamped.

### What the fix is

Three new scripts replace the LLM's per-repo judgment with mechanical
looping and verification:

1. `work_end_context.py` — gathers all context in one call (replaces
   the LLM running 6+ separate commands across pre-conditions/Steps 1-3)
2. `work_end_execute.py` — three subcommands (`promote`, `rebase`, `land`)
   that loop through ALL repos mechanically. The SKILL.md calls this
   script; the script does the per-repo work.
3. `verify_slot_close.py` — checks EVERY repo was merged, stamped,
   pushed. Hard gate before archive.

### The three-phase Execute structure (critical to understand)

The spec splits Execute into three phases to avoid LLM/script
interleaving:

```
Phase A — Rebase (script, all repos at once)
Phase B — Squash analysis (LLM per-repo loop — the ONLY LLM loop)
Phase C — Land (script, all repos at once — reads Phase B output files)
```

The SKILL.md makes exactly TWO script calls with an LLM loop between
them. Phase B writes `.squash-plan-<repo>.json` files; Phase C reads
them. This separation means the script never needs to call the LLM,
and the LLM never needs to call the script mid-loop.

### Where to be careful

**1. Slot mode vs non-slot mode.**
`land_branch.py` is used in non-slot mode only. Slot clones have
`origin` pointing to the on-disk original repo, not GitHub.
`detect_topology()` in `land_branch.py` would resolve wrong remotes.
In slot mode, `work_end_execute.py` implements the git sequence directly:
```
slot clone → push to original (origin) → original fetches → ff-only merge → push to GitHub
```
This is the same sequence `merge_slot()` in `slot_manager.py` uses.
Getting this wrong means pushing to the wrong remote or losing commits.

**2. Workspace deduplication.**
In multi-repo slots, multiple repos may share a workspace. Promotion
(`close_artifacts.py`) must run ONCE per unique workspace, not once per
repo. Calling it per-repo would duplicate artifacts. The plan's Task 6
handles this with a `promoted_workspaces: set` guard.

**3. Issue closing runs ONCE.**
`close_artifacts.py` closes issues when `covers=` is passed. In per-repo
mode, passing `covers=` to every call would close the same issues N times
(harmless but noisy, and errors if already closed). The fix: never pass
`covers=` to `close_artifacts.py` in per-repo mode. Issue closing happens
once after all repos complete, directly via `gh issue close`.

**4. Stamp AFTER squash.**
Garden entry GE-20260721-94263a documents this: squash rewrites commit
SHAs. Stamps written before squash reference pre-squash SHAs that become
unreachable. The Execute sequence enforces: squash (Phase B+C step 9)
before stamp (step 12). If you change the ordering, stamps will have
wrong SHAs.

**5. `.phase-a-complete` compatibility.**
`slot_manager.py`'s `scan_ready()` and `list_slots()` check for this
marker. Execute must write it after promotion succeeds (even though the
Phase A/B split is gone). Without it, `work-slot list` and `work-slot
next` break. This is a compatibility shim — documented in the spec's
Risks section.

**6. `.landed` marker.**
`archive_slot()` requires `.landed` (or `--force`). Without it, the
archive step fails. Execute's land subcommand must write this marker
after all repos are stamped. Format matches `merge_slot()`:
```
branch=<branch-name>
landed_shas=engine:<sha>,blocks:<sha>
timestamp=<ISO>
```

**7. Stamp push.**
The current `land_branch.py stamp` writes the stamp commit but does NOT
push it. Remote branches look live without the pushed stamp. Task 3
modifies `cmd_stamp()` to push the work branch after stamping. Use
`--force-with-lease` (not `--force`).

**8. `.execute-progress` is internal to the script.**
The SKILL.md does NOT read `.execute-progress`. The lifecycle state
(`META_STATE`) tells the SKILL.md which phase to resume from; the script
handles per-repo granularity internally. On re-entry, the SKILL.md
re-invokes the same script call; the script reads its own progress file.

**9. Stamp idempotency.**
Before creating a stamp commit, check if the branch tip is already a
`chore: branch closed` commit. If so, skip stamp creation (only retry
push if needed). This prevents duplicate stamps on crash recovery.

### What can go wrong

| Failure | Symptom | Recovery |
|---------|---------|----------|
| Rebase conflict | `ERROR=REBASE_CONFLICT` from Phase A | User resolves conflict, re-runs Execute |
| ff-only merge fails (slot mode) | Slot clone diverged from original | Retry from Phase A (re-fetch + rebase) |
| Build fails pre-push | `BUILD_FAILED` in Phase C | Fix on work branch, re-run Execute (progress skips completed repos) |
| Push fails | Network error or auth issue | Re-run Execute (progress skips stamped repos) |
| verify_stamp.py rejects stamp | Content not on main | Investigate: squash may have dropped commits |
| `.artifacts-promoted` stamp missing | `land_branch.py push` rejects | Re-run promote subcommand |
| Verify gate fails | `VERIFIED=no` from verify_slot_close.py | Re-run the specific Execute sub-step that failed |
| archive_slot rejects (no .landed) | `.landed` marker missing | Re-run land subcommand |

### What to verify at each milestone

**After Task 2 (audit --fix):**
- Run `audit_slot_merges.py /Users/mdproctor/claude/casehub --all` — should show 0 problems
- No content was merged, only stamps added

**After Task 4 (verify_slot_close.py):**
- Run against a known-good slot: `VERIFIED=yes`
- Run against a known-bad slot (create one in tests): `VERIFIED=no` with specific failure

**After Task 8 (execute.py land):**
- Full integration test: create a 2-repo slot, run promote→rebase→land, verify both repos stamped
- Crash recovery test: kill mid-land, re-run, verify no duplicate stamps

**After Task 9 (SKILL.md rewrite):**
- Run `python3 scripts/validate_all.py --tier commit` — zero errors
- The SKILL.md should be ~400 lines, not ~1400
- Run `sync-local` to install

### Key files

| File | Role | Status |
|------|------|--------|
| `work-end/SKILL.md` | Skill definition — will be rewritten | ~1400 lines, target ~400 |
| `work-end/close_artifacts.py` | Promotion orchestrator | Unchanged — called by execute.py |
| `work-end/land_branch.py` | Rebase/push/stamp | Modified: stamp adds push |
| `work-end/verify_stamp.py` | Content landing verification | Unchanged |
| `work-end/verify_promotion.py` | Artifact destination check | Unchanged |
| `work-end/close_report.py` | Report generation | Modified: build-verify step added |
| `work-end/branch_recon.py` | Branch state gathering | Unchanged — called by context.py |
| `work-end/branch_cleanup.py` | EPIC-CLOSED, checkout-main | Unchanged |
| `work-end/hygiene_scan.py` | Workspace hygiene | Unchanged — called by verify |
| `work-end/phase_a_complete.py` | Phase A marker (VESTIGIAL) | To be removed in Task 10 |
| `work-end/phase_b_gate.py` | Phase B gate (VESTIGIAL) | To be removed in Task 10 |
| `work-slot/slot_manager.py` | Slot operations | NOT modified — compatibility marker written |
| `project/routing.py` | Routing resolution | Unchanged — called by context.py |
| `project/lifecycle.py` | Lifecycle state machine | Unchanged — transitions fire from SKILL.md |
| `project/ctx.py` | Path resolution | Unchanged — called by context.py |
| `scripts/audit_slot_merges.py` | Slot merge audit | UNCOMMITTED — commit in Task 1 |
| `scripts/reconcile_slots.py` | Slot filesystem reconciliation | MODIFIED this session — added relocate_claude_projects |

### Uncommitted changes on soredium main

```
?? scripts/audit_slot_merges.py    — slot merge audit tool (written this session)
?? scripts/query_worklog.py        — worklog DB query tool (written this session)
M  scripts/reconcile_slots.py      — added relocate_claude_projects call
```

These should be committed as part of Task 1. `reconcile_slots.py` fix
is a separate concern (Claude project dirs not relocated during batch
archive) — could be its own commit.

### Stranded artifacts to recover (Task 1)

4 specs on closed workspace branches, never promoted to main:

| Branch | Spec | Content |
|--------|------|---------|
| issue-157-worklog-rest-mcp | worklog MCP server design | Python MCP, 4 tools |
| issue-171-lifecycle-state-machine | lifecycle state machine design | State machine model |
| issue-182-work-observability | worklog emission design | Automatic events |
| issue-66-final-review-mode | final review mode design | Branch quality gate |

Cherry-pick to workspace main (spec commit SHAs findable via
`git log --all --oneline -- specs/<dir>/`).

### Design review workspace

Review artifacts at `~/reviews/hortora-soredium/work-end-restructure-*/`.
4 dimensions (coherence, structure, robustness, cross-cutting), all DONE.
43 issues found, all resolved. The spec was updated in-place during
review — the committed version includes all fixes.

## What's Left

- Execute the 10-task plan · L · High
- WI 490 (work/issue-800) work-end was interrupted — branch not merged, not stamped, no blog · M · Med
- 4 orphaned active slots (1, 6, 84, 86) — scaffolded, zero commits, should archive · S · Low
- Slot 83 missing .slot file (was the slot that triggered the investigation) · S · Low
- Hygiene debt: pages 113, iot 30, soc 6 unstamped branches · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Execute Task 1-10 of work-end restructure plan | L | High | Multi-session |
| #170 | Pre-merge hook for slot artifact promotion | M | Med | — |
| #95 | Mechanize remaining LLM operations | L | High | Partially addressed by this work |
| #118 | Evaluate splitting HANDOFF.md roles | S | Low | — |
| #92 | Add restore-slot command | M | Med | — |

## References

- Spec: `/Users/mdproctor/claude/public/cc-praxis/specs/2026-08-05-work-end-restructure-and-slot-audit-design.md`
- Plan: `/Users/mdproctor/claude/public/cc-praxis/plans/2026-08-05-work-end-restructure.md`
- Review: `~/reviews/hortora-soredium/work-end-restructure-*/tracker.md` (4 dimensions)
- Audit script: `scripts/audit_slot_merges.py` (uncommitted)
- Garden: GE-20260721-94263a (squash rewrites stamp SHAs)

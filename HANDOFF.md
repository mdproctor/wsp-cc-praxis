# Handover — 2026-07-31

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Migrated 18 Trellis issues (#120–#137) from Hortora/soredium to Hortora/trellis. Cross-references rewritten, originals closed with pointers.
- Fixed #139: silent artifact promotion failures. Three compound bugs (bare except, PROMOTED=0 as success, push failure ignored) — TDD with 10 new tests. Also added push failure detection (PUSHED=skipped vs PUSHED=failed). Issue filed and referenced but not closed (more gaps may surface).
- Closed #98: tiered design review. Type×degree model (coherence/structure/robustness × light/standard/adversarial/deep). AskUserQuestion UX. Integrated into brainstorming and writing-plans. 21 new tests.
- Removed diary immutability constraint from write-content.
- Verified slot worktree→clone migration and attic preservation are working post-#138.
- Garden entry: GE-20260731-4d0718 — compound silent failure pattern.

## State Right Now

On main. Clean tree. #98 closed, #139 open (promotion fix landed, may need follow-up for commit_failed edge case).

## Immediate Next Step

Pick next work from What's Next.

## What's Left

- Handover skill: update to write `HANDOFF-{project_name}.md` using PROJECT_NAME from ctx.py · S · Low
- work-end: per-repo evidence gate · M · Med
- work-end: record slot-merge event in worklog · S · Low
- work-end: `close_artifacts.py` commit_failed when no project-side artifacts (empty commit) · S · Low — could not reproduce from code; may be resolved by #139 error reporting
- Hygiene scan false positives: specs inherited from main flagged as "unrecovered" on every closed branch · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #110 | Support nested epics | M | High | — |
| #95 | Mechanize LLM-executed state-changing operations | L | Med | — |
| #92 | Add restore-slot command | M | Med | — |
| #83 | Delegate handover mechanical steps to subagents | M | Med | — |
| #118 | Evaluate splitting HANDOFF.md roles | S | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*

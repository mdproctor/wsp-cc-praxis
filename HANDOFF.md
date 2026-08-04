# HANDOFF — soredium

## Last Session

Closed #183 (issue lifecycle wiring gaps). Pivoted from #141 which was
closed because its core scope had already landed across #158, #178,
and the #180 `.plan` infrastructure. The three remaining gaps:

1. **work-end issue-complete emission** — `complete_active_issue()` existed
   but work-end never called it. Added Step 0c to work-end SKILL.md.
2. **Crash reconciliation** — `reconcile_covers()` added to `plan_manager.py`.
   Runs at start of every `advance()` and at work-end Step 0c. Scans `[x]`
   items in `.plan` against `covers:` in `.meta`, appends missing ones.
   Also catches parent epics that `_mark_parent_epics_if_done()` marks `[x]`
   but never adds to covers.
3. **Slot-mode verification** — confirmed ctx.py resolves `PLAN_PATH` correctly
   in slot context. Added 2 tests for slot-mode detect + advance.

Code review found a CRITICAL: parent epic issue numbers were never added to
covers during `advance()` — only the leaf was. The reconciliation at
`advance()` start catches this for mid-queue advances, but the queue-exhaustion
case (last advance, then work-end) had no reconciliation. Fixed by adding
`reconcile_covers()` to work-end Step 0c.

## Immediate Next Step

Epic #182 has one remaining child: #157 (REST + MCP server over worklog.db).
This is deferred until trellis coordinator consumer needs are clearer.

## What's Left

### Epic #182 — Work Observability (1 of 5 remaining)

- **#157** — REST + MCP server over worklog.db. Deferred until consumer
  needs (trellis coordinator, Claude Code agents) are clearer. · L · High

### Housekeeping (carried forward)

- close_artifacts.py empty-commit bug — fails when nothing to promote
- Stashed WIP on main: `WIP: slot_manager ephemeral artifacts cleanup`
- Run `recover_orphaned_specs.py` — 19 stranded specs
- Publish 15 blog entries to hortora.github.io
- Stamp 6 merged-but-unstamped branches

### Other open issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #170 | Pre-merge hook for slot artifact promotion | M | Med | — |
| #157 | Worklog REST + MCP server | L | High | Epic #182 child |
| #118 | Evaluate splitting HANDOFF.md roles | S | Low | — |
| #110 | Support nested epics | L | High | — |
| #95 | Mechanize remaining LLM operations | L | High | — |
| #92 | Add restore-slot command | M | Med | — |
| #83 | Delegate handover mechanical steps | S | Med | — |
| #63 | ADR four-phase review pipeline | L | High | Epic |

## References

- Landed SHA: `6f77e07bb9e7255030926ff6753cfa0973e0df2c`

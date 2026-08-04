# HANDOFF — soredium

## Last Session

Closed #158 (wire worklog issue-activate/complete into callers). This is
the second child of epic #182 (Work Observability). Added `_emit_issue_events()`
helper to `plan_manager.py`, wired it into `advance()` for work-next,
added `complete_active_issue()` for work-end, and wired
`record_issue_activate()` into `scaffold.py` for work-start. 17 new tests.

## Immediate Next Step

Resume epic #182 with `work 182`. Auto-detection will skip closed #178 and
#158, starting on #141 (nested issue lifecycle in slots — auto-end-previous
on advance).

## What's Left

### Epic #182 — Work Observability (2 of 4 remaining)

- **#141** — Nested issue lifecycle in slots — auto-end-previous on
  advance. · L · High
- **#157** — REST + MCP server over worklog.db. Deferred until trellis
  coordinator needs are clearer. · L · High

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
| #141 | Issue-level lifecycle in slots | L | High | Epic #182 child |
| #118 | Evaluate splitting HANDOFF.md roles | S | Low | — |
| #110 | Support nested epics | L | High | — |
| #95 | Mechanize remaining LLM operations | L | High | — |
| #92 | Add restore-slot command | M | Med | — |
| #83 | Delegate handover mechanical steps | S | Med | — |
| #63 | ADR four-phase review pipeline | L | High | Epic |

## References

- Spec: `~/claude/hortora/soredium/docs/specs/issue-158-worklog-track-individual-issue/2026-08-04-issue-transition-wiring-design.md`
- Plan: `~/claude/public/cc-praxis/plans/attic/issue-158-worklog-track-individual-issue/2026-08-04-issue-transition-wiring.md`
- Landed SHA: `d804e07fc08ca2030925508643dea640639afb81`

# HANDOFF — soredium

## Last Session

Closed #178 (worklog event emission at state transitions). This is the
first child of epic #182 (Work Observability). Designed the separation of
concerns: state machine owns ALL event emission and work_items.state
updates; callers own resource creation only. Implemented as extensions
to lifecycle.py and worklog.py with 16 new tests (unit + integration).

Also created epic #182 with four children (#178 → #158 → #141 → #157)
and updated their GitHub issue bodies with parent/blocked-by/blocks
references.

## Immediate Next Step

Resume epic #182 with `work 182`. Auto-detection will skip closed #178
and start on #158 (wire callers to emit issue-activate/complete events).

## What's Left

### Epic #182 — Work Observability (3 of 4 remaining)

- **#158** — Wire callers (`work next`, `epic_manager`) to call
  `record_issue_activate/complete`. Functions exist in worklog.py,
  callers don't call them. · M · Med
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
| #158 | Worklog: issue transitions within branches | M | Med | Epic #182 child |
| #157 | Worklog REST + MCP server | L | High | Epic #182 child |
| #141 | Issue-level lifecycle in slots | L | High | Epic #182 child |
| #118 | Evaluate splitting HANDOFF.md roles | S | Low | — |
| #110 | Support nested epics | L | High | — |
| #95 | Mechanize remaining LLM operations | L | High | — |
| #92 | Add restore-slot command | M | Med | — |
| #83 | Delegate handover mechanical steps | S | Med | — |
| #63 | ADR four-phase review pipeline | L | High | Epic |

## References

- Spec: `~/claude/public/cc-praxis/specs/issue-182-work-observability/2026-08-04-worklog-emission-design.md`
- Plan: `~/claude/public/cc-praxis/plans/2026-08-04-worklog-emission.md`
- Landed SHA: `59fb5bcef36d577f5a79d5f5f461beb4f34f8c0c`

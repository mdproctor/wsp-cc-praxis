# HANDOFF — soredium

## Last Session

Two issues closed from epic #182 (Work Observability):

**#183 — Issue lifecycle wiring gaps** (S/Med). Three gaps:
1. work-end Step 0c now calls `reconcile_covers()` + `complete_active_issue()`
2. `plan_manager.advance()` runs crash reconciliation at start of every call
3. Slot-mode plan detection verified with tests

Code review found parent epic issues never added to `covers:` during
`advance()` — fixed by adding `reconcile_covers()` to work-end pre-close.

**#157 — Worklog MCP server** (L/High → actual: S/Med). Python FastMCP
server (stdio) with 4 read-only tools wrapping `worklog.py` queries:
`worklog_active`, `worklog_events`, `worklog_timeline`, `worklog_slots`.
Includes repo_path normalization, metadata JSON parsing, structured error
handling. 15 tests. Design review (light, 3 dimensions) surfaced 4
accepted robustness fixes applied to the spec before implementation.

Key design decision: Python MCP server in soredium (not Quarkus, not in
trellis). Rationale: per-session stdio server is lightweight; trellis
REST/MCP for always-on consumers is a separate trellis issue. The two
aren't mutually exclusive — Python MCP is the baseline that works without
trellis.

## Immediate Next Step

Epic #182 is effectively complete — all "Done when" criteria met except
REST/MCP exposure, which was descoped to a trellis issue. Consider
closing the epic or filing a trellis issue for the REST layer.

## What's Left

### Epic #182 — Work Observability

All children closed. The remaining "Done when" item (worklog queryable
via REST + MCP) is now split:
- MCP: done (soredium #157)
- REST: deferred to trellis (not yet filed)

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
| #118 | Evaluate splitting HANDOFF.md roles | S | Low | — |
| #110 | Support nested epics | L | High | — |
| #95 | Mechanize remaining LLM operations | L | High | — |
| #92 | Add restore-slot command | M | Med | — |
| #83 | Delegate handover mechanical steps | S | Med | — |
| #63 | ADR four-phase review pipeline | L | High | Epic |

## References

- #183 landed SHA: `6f77e07bb9e7255030926ff6753cfa0973e0df2c`
- #157 landed SHA: `4a031c204f4e905270376fab2415dc7978546ea0`
- Spec: `/Users/mdproctor/claude/public/cc-praxis/specs/issue-157-worklog-rest-mcp/2026-08-05-worklog-mcp-server-design.md`
- Plan: `/Users/mdproctor/claude/public/cc-praxis/plans/2026-08-05-worklog-mcp-server.md`

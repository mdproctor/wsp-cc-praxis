# HANDOFF — 2026-06-18

## Last Session

Completed #123 — externalised all mechanical code from SKILL.md files. Full lifecycle:
brainstorm (3 review rounds on the design spec), implementation plan, subagent-driven
execution of 14 tasks (Phase 1 ctx.py + Phase 2 per-skill scripts), code review,
squash (20→3), and close.

## Immediate Next Step

Pick the next issue from the backlog. The README/landing page design (#— untracked)
is still open from the previous session — read `specs/2026-06-17-readme-landing-page-design.md`
and the strategic question (merge cc-praxis into hortora?) before starting new work.

## What's Left

- **Notes cleanup script** — `cleanup_notes.py` at `/tmp/cleanup_notes.py`, dry-run approved (203 files), not yet applied · XS · Low
- **marketplace.json auto-generation** — still manual · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Resolve: merge cc-praxis into hortora? | S | High | Blocks landing page design |
| — | README + landing page (workflow-first) | M | Low | Design doc ready |
| — | Apply notes cleanup script | XS | Low | `/tmp/cleanup_notes.py --apply` |

## References

- **Design spec:** `docs/specs/2026-06-17-externalise-bash-blocks-design.md`
- **Impl plan:** `plans/attic/issue-123-externalise-bash-blocks/2026-06-18-externalise-bash-blocks.md`
- Blog: `blog/2026-06-18-mdp01-the-category-error-that-halved-the-work.md`

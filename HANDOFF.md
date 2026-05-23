# HANDOFF — 2026-05-23

## Last Session

Issue #96 complete — but reversed. Came in to finish the `diary→log` rename;
discovered "log" was the wrong word for personal narrative writing and reverted
instead. 121 files across 7 repos changed back to `subtype: diary`. Skill files
updated and synced. ADR-0011 written. Two protocols captured.

## Immediate Next Step

Run `python3 scripts/revert_diary_subtype.py` (dry-run) to check for new drift
from sessions that haven't yet picked up the updated skills. Apply if any appear.

## What's Left

- **Blog routing question** — cc-praxis posts in `docs/_posts/` not cross-published;
  decision pending on project-only vs cross-publish · XS · Low
- **issue-94 branch** — still exists locally, all work on main. Safe to delete · XS · Low
- **Eventual consistency** — run `scripts/revert_diary_subtype.py` periodically
  until zero drift. See `docs/protocols/taxonomy-rename-idempotent-script.md` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Write Proposal B (short taxonomy article) | S | Low | Needs Sparge viewer for A/B first |
| — | Proposal A 6-part series | XL | Med | After Proposal B validated |
| — | Academic paper research | M | High | Kinneavy 1971, JSTOR, Procida |
| — | Resolve cc-praxis blog cross-publish question | XS | Low | See What's Left |

## References

- Blog: `blog/2026-05-22-mdp01-workspace-gets-a-workspace.md`
- ADR: `docs/adr/0011-revert-subtype-diary-log-to-diary.md`
- Protocols: `docs/protocols/` (2 new — taxonomy naming, cleanup script pattern)
- Garden: `tools/GE-20260522-543863.md` (git checkout-b silently reverts to main)
- Cleanup script: `scripts/revert_diary_subtype.py` — idempotent, dry-run by default

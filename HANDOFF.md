# HANDOFF — 2026-05-22

## Last Session

Workspace finally set up for cc-praxis (`~/claude/public/cc-praxis/` →
`mdproctor/wsp-cc-praxis`). CLAUDE.md migrated to workspace, HANDOFF.md,
IDEAS.md, and plans moved. Also fixed `work-end` Step 8g to auto-invoke
`publish-blog` instead of prompting — blog routing now consistent with other
artifact routing.

## Immediate Next Step

Start issue #96 — diary→log sweep (`subtype: diary` → `subtype: log` across
~150 posts in `docs/_posts/`). Workspace is now configured; use `work-start`
from `~/claude/public/cc-praxis/`.

## What's Left

- **Blog routing open question** — cc-praxis posts (17) live only in
  `docs/_posts/` (Jekyll site). blog-routing.yaml would route `entry_type:
  note` to `mdproctor.github.io/_notes/`, but none have been cross-published.
  Decision pending: project-only or cross-publish? · XS · Low
- **issue-94 branch** — still exists locally, all work on main, issue closed.
  Safe to delete when convenient. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #96 | diary→log sweep across `docs/_posts/` | M | Low | Batch frontmatter update |
| — | Write Proposal B (short taxonomy article) | S | Low | Needs Sparge viewer for A/B first |
| — | Proposal A 6-part series | XL | Med | After Proposal B validated |
| — | Academic paper research | M | High | Kinneavy 1971, JSTOR, Procida |
| — | Resolve cc-praxis blog cross-publish question | XS | Low | See What's Left |

## References

- Blog: `blog/2026-05-22-mdp01-workspace-gets-a-workspace.md`
- Workspace: `~/claude/public/cc-praxis/` → `mdproctor/wsp-cc-praxis`
- Garden: `tools/GE-20260521-f4c128.md` (SSH→HTTPS fallback on new remotes)
- Corpus: `~/claude-workspace/corpus/catalogue.tsv`, `~/claude-workspace/corpus/index.md`
- Fingerprint: `~/claude-workspace/writing-styles/mark-proctor-voice.md`

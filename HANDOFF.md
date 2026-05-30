# HANDOFF — 2026-05-30

## Last Session

Major skill consolidation: 58 → 44 skills, 73 → 25 slash commands. Added `project-init` (normalised setup gateway), migrated 4 principles skills + quarkus skills to Hortora (`approaches/` and `frameworks/` directories). Deleted `epic`, `update-primary-doc`, `blog-git-commit`. Simplified `custom-git-commit`. Added `work-end` step 8k (mvn install for Java). Fixed handover rename blocking.

## Immediate Next Step

Next logical piece: generalise `java-update-design` → universal `update-design` (create or update ARC42STORIES.MD for any project type), then delete `design-snapshot`. The architecture decision is agreed; the implementation hasn't started.

## What's Left

- **issue-94 branch** — still exists locally, all work on main. Safe to delete · XS · Low
- **issue-96 branch** — same · XS · Low
- **`design-snapshot`** — redundant once `update-design` is universal; pending that refactor · M · Med
- **README redesign** — paused; decisions made, ready to implement · M · Low
  - Problem: positioned as skill bundles, should be positioned as a workflow
  - Chosen structure: workflow-first (option B) — work/goals/results table, `/work` as hero
  - Above the fold: four-stage lifecycle strip (`/work` → Build → `/work end` → Next session), no skill names
  - "What changes" section: four cards by workflow stage
  - Before/after: concrete contrast, no skill names anywhere
  - Visual mockups on disk: `.superpowers/brainstorm/39544-1779802304/content/` (work-centered.html is the closest to final)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Generalise java-update-design → universal update-design (create or update) | M | Med | Then delete design-snapshot |
| — | Remove remaining auto-triggered slash commands (quarkus-flow has no command; check others) | XS | Low | |
| — | Update framework-registry.md spec to document approaches/ layer | XS | Low | Hortora docs |
| — | README redesign (work/goals/results framing) | M | Low | Paused; foundation now solid |

## References

- Blog: `blog/2026-05-30-mdp01-frameworks-approaches-and-phantom-skills.md`
- Hortora frameworks: `~/.hortora/garden/frameworks/quarkus-flow.md`
- Hortora approaches: `~/.hortora/garden/approaches/` (observability, observability-patterns, code-review, security-audit, dependency-management, testing)
- Framework registry spec: `~/.hortora/garden/docs/framework-registry.md`

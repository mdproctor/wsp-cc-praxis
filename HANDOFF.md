# HANDOFF — 2026-06-05

## Last Session

Router pattern introduced across cc-praxis: six cross-language skills (code-review,
security-audit, dependency-update, git-commit, update-design, project-health) now
dispatch to per-language content files that load on demand. install-skills,
uninstall-skills, the web installer, and cc-praxis UI launcher retired — no longer
needed when "install everything" has no cross-language waste. 43 → 30 skills.

## Immediate Next Step

`work` to start the work lifecycle consolidation — absorb work-start, work-end,
work-pause, work-resume as content files inside `work/`. Discussed but not started.

## What's Left

- **`issue-94`, `issue-96`, `issue-108` workspace branches** — no EPIC-CLOSED.md,
  work already on main. Safe to delete · XS · Low
- **README redesign** — workflow-first framing decided (option B), mockup at
  `.superpowers/brainstorm/39544-1779802304/content/work-centered.html` · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Work lifecycle consolidation — work-start/end/pause/resume → content files in work/ | M | Low | Discussed, not started |
| — | README redesign (workflow-first framing) | M | Low | Foundation now solid |
| — | marketplace.json auto-generation | S | Low | Currently manual; add to generate_commands.py or similar |

## References

- Blog: `blog/2026-06-05-mdp01-the-skill-that-ate-itself.md`
- Archive: `archive/skill-installer-ui` on GitHub (selective install, if needed later)
- Garden: `tools/GE-20260605-f11f8f.md` (git rm silent on empty dirs), `claude-code/GE-20260605-248ca7.md` (agents write-only pattern)

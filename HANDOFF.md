# HANDOFF — 2026-06-17

## Last Session

Long session: fixed a large number of cc-praxis bugs and improvements (work-pause routing,
ctx.py path resolution, workspace symlink creation, note label consistency, blog series linking,
handover auto-resume, work-end closing summary). Also started — but did not finish — the README +
landing page brainstorm. Design document written to capture everything.

## Immediate Next Step

Read `specs/2026-06-17-readme-landing-page-design.md` before anything else. It has all decisions
made, the workspace copy options still to pick, the lifecycle diagram, and the new strategic
question: **should cc-praxis merge into hortora?** That question needs to be resolved before the
README/landing page design can be finalised — it affects what we're naming and where it lives.

Start by deciding: merge or standalone?

## Strategic question (new — added this session)

cc-praxis + hortora together = "how I do casehub development at scale". Too many things to install
and explain for casehub onboarding (casehub + hortora + cc-praxis). Proposal: merge cc-praxis into
hortora so there's one install story. cc-praxis skills become the "development workflow" module
within hortora. If this happens, the landing page we were designing may move to hortora.

## What's Left

- **README + landing page design** — brainstorm in progress, design doc at `specs/2026-06-17-readme-landing-page-design.md` · M · Low
- **Notes cleanup script** — `cleanup_notes.py` at `/tmp/cleanup_notes.py`, dry-run approved (203 files), not yet applied · XS · Low
- **marketplace.json auto-generation** — still manual · S · Low
- **Externalisation audit #123** — 18 bash blocks remaining across 7 skills · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Resolve: merge cc-praxis into hortora? | S | High | Blocks landing page design |
| — | README + landing page (workflow-first) | M | Low | Design doc ready, pick workspace copy option |
| — | Apply notes cleanup script | XS | Low | `/tmp/cleanup_notes.py --apply` |
| #123 | Externalise remaining bash blocks | L | Med | 18 remaining, audit in issue |

## References

- **Design doc:** `specs/2026-06-17-readme-landing-page-design.md`
- **Notes cleanup:** `/tmp/cleanup_notes.py` (203 files, dry-run confirmed correct)
- **Lifecycle mockup:** `.superpowers/brainstorm/33730-*/content/page-structure-v2.html`
- Blog: `blog/2026-06-05-mdp02-pause-means-nothing-if-you-cant-go-somewhere.md`

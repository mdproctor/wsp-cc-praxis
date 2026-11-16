# Session Handover

**Branch:** `main` (no work branch open)
**Date:** 2026-11-16
**Next issue:** Hortora/soredium#515 — Canonical sync: one landing flow with linear history, canonical as the only publisher

## Where things stand

The work-end redesign (#499) landed on 2026-11-05 and replaced the orchestrator with
`work-end/close.py`. Real use then exposed a run of failures, all now fixed and pushed:

| Issue | What was wrong | Fix |
|---|---|---|
| #501, #502, #503 | Staged but never committed; preflight reset `main`; reset before landing | Commit and tip verification; no resets anywhere in `close.py` |
| #504, #507 | `work start #N` routing on a feature branch | Router distinguishes active vs other issue |
| #505 | Handover orchestrator called a deleted script | Uses `pause_exec.py commit-wip` |
| #506, #508 | Artifacts and images never reached the project | `work-end/artifact_sweep.py`, image co-promotion |
| #509 | Queue branch empty or stale | Start, continue and resume sync it |
| #510 | No tests on real topologies | Integration suite, slot detection fix |
| #513 | Closed an already-stamped branch and reported success | Stamped branches are never targets; queue/branch mismatch fails |
| #514 | `close.py` collapsed commits via `.squash-plan` | No squash code in `close.py`; git-squash records `refs/squash-reviewed/<branch>` |
| #515 (part) | Reported SHA not `main`'s tip; same-named repos overwrote each other; ceremony ran on a repo stopped mid-rebase | Strip on the branch; `WORKSPACE_LANDED_*`; idle-repo guard |

Reference incident: casehubio/casehub-pages#666.

## The rules now in force

Recorded in the project `CLAUDE.md` under "Landing Rules":

1. Linear history on every base branch, workspace repos included.
2. Canonical repos (`~/claude/<family>/<repo>`) are the sync point.
3. Only canonical pushes to GitHub.
4. Squashing only through the git-squash skill.

Claims about git or script behaviour must be observed by running them, not assumed.

## Next action

`work start soredium#515`, stage 1. The issue holds the eight-step landing flow, the
table of observed git behaviour, the list of scripts to change and the stages.

Stage 1, in order:

1. One module that owns every push and every base-branch change (sync from remote,
   rebase, hold in canonical, publish, advance, mirror, branch backup), tested on a
   fixture of canonical, a slot clone and two bare remotes.
2. Move the callers onto it: `close.py`, `land_flow.py`, `artifact_promote.py`,
   `pause_exec.py`, `branch_create.py`, `safe_commit.py`, `quick_fix.py`,
   `git-squash/branch_swap.py`, and the "always push from a slot" text in
   `git-commit/SKILL.md`.
3. Workspace repos land linearly (supersedes ADR-0016).
4. Replace `work pause`'s `pull --rebase`, which left the pages workspace stopped
   mid-rebase and cost four commits.
5. Every script that runs `rebase --abort` first checks no rebase was already in
   progress (`land_flow.py`, `branch_create.py`, `quick_fix.py`,
   `git-squash/rebase_exec.py`; `close.py` is already guarded).
6. Session start reports a repo that is mid-operation or on a detached HEAD.
7. Commit-tier validator: no push, non-fast-forward merge or soft reset outside the
   module. Remove the noise regexes from `.githooks/pre-push`.

Later stages: finish the clone migration and rewire slot remotes (stage 2); optional
bare canonical (stage 3); per-item `branch:` in the queue, then the slot-root layout
in #512 (stage 4). #512 has a review comment listing four gaps to settle first.

## Open questions for the user

- Do the per-repo `wsp-casehub-*` repos matter as published artifacts, or can one
  family workspace repo replace them? Decides how the slot root gets a remote (#512).
- Not yet verified for #515: two remotes where the blessed one accepts and the mirror
  fails; the per-repo lock under two concurrent closes; git versions other than 2.50.1.

## State of the repos

- **soredium:** on `main`. One local commit not pushed: `cc5c2fc`, the Landing Rules
  section in `CLAUDE.md`.
- **This workspace:** on `main`; `.queue` has an uncommitted change from an earlier
  session. It still names the #501 branch, which is closed, and it has lost its
  `## Deferred` section (see below).
- **Installed skills:** synced to soredium `main` as of `73685d7`. A session started
  before 2026-11-15 needs `/clear` to load the updated work-end and git-squash skills.
- **Deferred items, no longer in `.queue`:** `pre_push_hook.py` still imports
  `lifecycle.read_state`; `slot_lifecycle.py` still imports `land_flow`. Both fold
  into #515 stage 1.
- **Bug found while writing this handoff:** `plan_manager.py set-state` on this
  `.queue` dropped the `## Deferred` section and doubled the heading to
  `# Work Plan — Work Plan —`. Seen in the uncommitted diff; not yet reproduced in a
  test. Recorded on #515.

## Known stale documentation

- `CLAUDE.md` still describes `lifecycle.py` as owning 12 lifecycle states and lists
  `tests/test_lifecycle.py`, which was deleted in #499.
- The commit-tier validators (frontmatter, CSO, references) fail on `main`; the
  failures predate this work and have not been investigated.

## Key files

| File | Role |
|---|---|
| `work-end/close.py` | Close ceremony: guards, review gate, rebase and fast-forward, stamp, advance |
| `project/squash_marker.py` | Squash-review marker: `mark`, `status`, `clear` |
| `project/queue_io.py` | `.queue` reader and writer, queue validation |
| `work-end/artifact_sweep.py` | Workspace to project artifact promotion |
| `tests/test_close_integration.py` | Ceremony tests on real repo topologies |
| `tests/test_git_landing_behaviour.py` | Observed git behaviours the #515 design relies on |

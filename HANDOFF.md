# Session Handover

**Branch:** `main` (no work branch open)
**Date:** 2026-11-16
**Next issue:** Hortora/soredium#517, then #515 — Canonical sync: one landing flow with linear history, canonical as the only publisher

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

## Landed on 2026-11-16 (pushed to origin, installed with sync-local, issues closed)

| Issue | What was wrong | Fix |
|---|---|---|
| #516 | `plan_manager` deleted deferred items written with no repos, and any line or section it could not parse; `queue_io` read zero items after any `plan_manager` write | Both modules keep each other's data; `tests/test_queue_file_roundtrip.py` |
| #518 | Close ceremony's stamp, queue-advance, merge and strip commits had no issue reference; the commit hook rejected the stamp after `main` had advanced | Commits carry `Refs <owner/repo#N>`; hook accepts qualified references |
| #519 | Slot merge (`land_flow.py`) treated a branch with only Markdown, text or image changes as already merged and stamped it unmerged; `restore-slot` crashed on a removed function; seven slot tests used a real workspace under the home directory | Whole-tree comparison; dead call removed; slot tests run with an isolated home |
| #520 | Duplicate-work check on `plan_manager append` never ran from the installed copy; the import error was swallowed | `worklog.py` finds `plan_io` when installed; failures print a warning |
| #521 | Archive kept `.m2` and Maven `target/` in the attic; work-end in a slot tried to archive the slot it was running in; a session in `slots/211` counted as active in `slots/21` | Archive purges build outputs inside the slot; `close.py` writes `.landed` when the queue drains and prints `SLOT_READY_FOR_ARCHIVE`; work-end never archives or offers to; cwd match respects the slot boundary |

Also landed with #518: a commit from another session making a running Claude Code
session a hard stop in `archive_slot` that `--force` cannot bypass. Tests added here.

soredium `origin/main` is at `c528bad`. #516, #518, #519, #520 and #521 are closed.

## #517 landed, pushed and installed; issue closed (2026-11-16)

soredium `main` equals `origin/main` at `52cdfb7`. That tip is #522, committed
directly onto `main` by another session sharing this checkout: it removed the 26
stale tests left from the #499 rewrite.

- A queue item can be **done but not closed**: `- [ ] ref — title ← done on <branch>`.
  `plan_manager.py mark-done <queue> [branch=]` writes it. Both queue modules keep it.
- `work next` on a feature branch runs `mark-done`, not `advance`. Before, it checked
  the item without landing it and blanked the queue's branch.
- `close.py` lands every done item's branch in queue order, then the active one, and
  checks each item after its own landing and stamp. Squash review needed anywhere, or
  a done branch missing, stops the batch with nothing changed. A failure part-way
  commits the closed items and leaves the rest done. Output: `CLOSED_ITEM=<ref> branch=<b>`.
- `project/work_chain.py` is now a script the work skill runs
  (`work_chain.py <cmd> queue=<path>`). It had no caller before: the skill read
  `CHAIN_DIRECTIVE` from `ctx.py`, which never emitted it. It reads done-ness from the
  queue, not GitHub, and prints `OFFER=wrap_or_continue` whenever an item is finished
  or done items are waiting. `ctx.py` reports `DONE_PENDING` and `DONE_ITEMS`.
- Tests: `tests/test_queue_done_state.py`, `tests/test_close_batch.py`,
  `tests/test_done_wiring.py`.

Not done for #517: nothing enforces that a new branch is cut from base rather than
from the done branch (stacking works and is tested, but is not chosen for the user);
epic checkbox ticking on GitHub (`tick_github` in the next pipeline) was not
re-examined against the done state; the behaviour has not been exercised in a real
multi-issue session or in a slot.

## Artifact recovery (handed to a separate session)

Every commit that no branch reached is pinned under `refs/rescue/<sha>` in 88
canonical repos (10,683 commits), so nothing can be pruned. Restoring, promoting
and publishing the lost blogs, specs and images is a separate session's job. Its
brief, scripts and draft inventory are in
`/Users/mdproctor/claude/casehub/recovery-2026-11-16/`. This workspace stays on
the work-end and slot tooling.

## Next action

1. Check `tick_github` in the `next` pipeline against the done-not-closed state.
2. #515 stage 1, below. **First item: delete `work-end/land_flow.py`** and replace
   its one caller, `work-slot/slot_lifecycle.py` (`build_slot_batch`, `land_batch`).
   It still hard-resets to the remote (lines near 430 and 570); evidence of lost
   workspace commits is recorded on #515.

### #515 stage 1

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

- **soredium:** on `main`, equal to `origin/main` at `52cdfb7`, working tree clean.
- **This workspace:** on `main`. `.queue` has no active item and two deferred items.
- **Installed skills:** synced to soredium local `main` at `971b114` on 2026-11-16.
- **Deferred in `.queue`:** `pre_push_hook.py` still imports `lifecycle.read_state`;
  `slot_lifecycle.py` still imports `land_flow`. Both fold into #515 stage 1.
- **Shared checkout:** another session committed onto this session's branch because
  both use the soredium checkout. Check `git log` before landing a branch.

## Known gaps


- `verification/outcomes.py` `verify_content_landed` compares source-extension files
  only (the #519 flaw). No callers found; delete or fix under #515.
- `archive_slot` can be refused with `ERROR=active_sessions` naming a short-lived
  `sleep` process left by the machine's managed git hook after a commit. Retrying works.
- Other workspaces' queues have not been checked for deferred items lost before #516.
- `work-end` is not batch aware: it closes only the branch named in the queue (#517).

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

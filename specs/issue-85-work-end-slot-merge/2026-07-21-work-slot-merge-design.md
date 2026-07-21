# work-slot merge — Full Slot Lifecycle Management

**Issue:** Hortora/soredium#85
**Date:** 2026-07-21
**Status:** Design

## Problem

work-end Phase B (rebase onto main, push, close issues, promote artifacts)
can only be triggered by returning to the slot session and saying "merge".
A session running in the main repo has no way to discover or merge pending
slots. The slot session may no longer exist.

## Solution

Add `work-slot merge` as a user-facing command that runs from any repo in
the family (or the family root). It discovers ready-to-land slots, presents
them, and executes the full Phase B + archival sequence.

## Slot Lifecycle States

Five states, tracked by filesystem markers in the slot directory:

| State | Marker | Meaning |
|-------|--------|---------|
| `active` | slot dir exists, no `.phase-a-complete` | Work in progress |
| `ready to land` | `.phase-a-complete` exists | Phase A done, awaiting merge |
| `landed` | `.landed` exists | Phase B done, code on main, issues closed |
| `archived` | moved to `worktrees/attic/<N>/` | Worktrees removed, metadata preserved |

`landed` is transient — exists between Phase B completion and archival. Its
purpose is to prevent a slot from being offered for merge again if something
fails between push and archive.

`work-slot list` shows active + ready to land by default.
`work-slot list --all` includes archived (from `worktrees/attic/`).

## `work-slot merge` Flow

### Step 1 — Find family root

Walk up from CWD looking for a directory that is not itself a git repo and
contains child directories with `wksp` symlinks. Same logic as `work-slot
create` Step 2. If the walk-up fails, ask the user.

### Step 2 — Scan and present

Call `slot_manager.py scan-ready <family-root>`.

The script returns JSON with per-slot data:
- Slot number, branch name, repos
- SLOT.md context ("What to do" section)
- Per-repo commit count and diff stats (from slot worktree branch vs origin/main)
- Phase A completion timestamp

The skill fetches issue titles via `gh issue view` and formats the rich listing:

```
Slots ready to merge:

  [1] issue-42-spi
      Repos: engine (3 commits, +142/-38)
      Issue: casehubio/engine#42 — "Add expression SPI"
      Context: Implement SPI for pluggable expression evaluation
      Phase A completed: 2026-07-18 14:32

  [2] issue-55-ledger
      Repos: engine (5 commits, +310/-94), iot (2 commits, +48/-12)
      Issue: casehubio/engine#55 — "Ledger event routing"
      Context: Route ledger events to IoT module via CloudEvent dispatch
      Phase A completed: 2026-07-19 09:15

Merge which slot? (number, or "all")
```

If no slots are ready to merge, report and stop.

### Step 3 — User picks

A slot number, or "all". If "all", slots are processed sequentially in
slot-number order.

### Step 4 — Pre-check

For every original repo involved across all selected slots: verify it has
main checked out. If any repo is on a different branch, stop and report
which repo is on the wrong branch.

### Step 5 — Sequential merge

For each selected slot, in order:

**5a. Dry-run rebase** — call `slot_manager.py merge-slot <family-root>
slot=<N>`. The script rebases all repo branches onto current `origin/main`
in the slot worktrees. If any repo conflicts, the script outputs
`ERROR=conflict repo=<name>` and stops. No repos have been pushed.

**5b. Fast-forward and push** — if dry-run passes, the same `merge-slot`
call continues: for each repo in the original checkout, fetch origin main,
rebase origin/main, merge --ff-only the branch, push origin main. Returns
`LANDED_SHAS=<repo:sha,repo:sha>`. Writes `.landed` marker with timestamp
and SHAs.

**5c. Post-merge actions** — the skill handles these (not the script):
- Close issues: all in COVERS via `artifact_promote.py close-issues`
- Promote artifacts: from slot workspace worktree via `artifact_promote.py`
- Publish blog: via `blog_dest.py`

**5d. Stamp branches** — empty commits on branches in the slot worktrees:
`chore: branch closed — landed as <SHA> on main`

**5e. Archive** — call `slot_manager.py archive-slot <family-root>
slot=<N>`. The script runs `git worktree remove` for each repo and
workspace worktree, then moves `worktrees/<N>/` to `worktrees/attic/<N>/`.
SLOT.md, `.phase-a-complete`, `.landed`, and other metadata survive in
the attic.

If "all" was selected and slot N fails at 5a (conflict), slots N+1 onward
are not attempted. Slots before N are already landed and archived.

## Script Changes: `slot_manager.py`

Three new subcommands:

### `scan-ready <family-root>`

Scans `worktrees/*/` for directories with `.phase-a-complete` (and no
`.landed`). For each slot, reads SLOT.md for branch name, repos, issue
info, and context. Runs `git log --oneline` and `git diff --shortstat`
in each repo's slot worktree to get commit count and diff stats. Reads
Phase A timestamp from `.phase-a-complete`.

Returns JSON:
```json
{
  "slots": [
    {
      "number": 1,
      "branch": "issue-42-spi",
      "repos": [
        {"name": "engine", "commits": 3, "diff": "+142/-38"}
      ],
      "issue": "42",
      "issue_repo": "casehubio/engine",
      "covers": "42",
      "context": "Implement SPI for pluggable expression evaluation",
      "phase_a_timestamp": "2026-07-18T14:32:00"
    }
  ]
}
```

### `merge-slot <family-root> slot=<N>`

Executes in two stages with progress output:

**Stage 1 — Dry-run rebase.** For each repo in the slot:
```bash
git -C <slot>/<repo> fetch origin main
git -C <slot>/<repo> rebase origin/main
```
If any rebase fails: abort that rebase, output `ERROR=conflict
repo=<name>`, exit non-zero. No push has happened.

Output: `STAGE=dry-run STATUS=pass` or `STAGE=dry-run STATUS=fail`

**Stage 2 — Fast-forward and push.** For each repo (in original checkout):
```bash
git -C <original>/<repo> fetch origin main
git -C <original>/<repo> rebase origin/main
git -C <original>/<repo> merge --ff-only <branch>
git -C <original>/<repo> push origin main
```
If any push fails: output `ERROR=push_failed repo=<name>`, exit non-zero.

On success: write `.landed` marker:
```
branch=<BRANCH_NAME>
repos=<comma-separated>
landed_shas=<repo:sha,repo:sha>
timestamp=<ISO-8601>
```

Output: `STAGE=push STATUS=pass LANDED_SHAS=engine:<sha>,iot:<sha>`

The script resolves original repo paths by reading the git worktree's
commondir (the original repo that owns the worktree).

### `archive-slot <family-root> slot=<N>`

1. For each repo and workspace worktree in the slot:
   `git worktree remove --force <path>`
2. Move `worktrees/<N>/` to `worktrees/attic/<N>/`
3. Output `ARCHIVED=<N>`

The attic directory preserves SLOT.md, `.phase-a-complete`, `.landed`,
and any other metadata files for auditing.

## Skill Changes: `work-slot/SKILL.md`

Add a `## work-slot merge` section after the existing `work-slot remove`
section. The skill orchestrates the flow described above, calling
`slot_manager.py` subcommands and handling GitHub API calls and cross-skill
invocations.

## Changes to `work-slot list`

Update `list_slots()` to:
- Show the lifecycle state using all markers (active / ready to land /
  landed)
- Accept a `--all` flag that also scans `worktrees/attic/` for archived
  slots
- Include per-repo commit count and diff stats for active and ready-to-land
  slots

## Files Changed

| File | Change |
|------|--------|
| `work-slot/slot_manager.py` | Add `scan-ready`, `merge-slot`, `archive-slot` subcommands. Update `list_slots` for new states and `--all`. |
| `work-slot/SKILL.md` | Add `work-slot merge` section. Update lifecycle state documentation. |
| `work-end/SKILL.md` | No changes — Phase A/B logic stays as-is. The existing Phase B path (from inside the slot) continues to work. |

## What This Does NOT Change

- **Phase A** — still runs inside the slot via work-end, unchanged.
- **Phase B from inside the slot** — still works. If the user returns to
  the slot session and says "merge", work-end detects `.phase-a-complete`
  and runs Phase B as before.
- **work-end** — no modifications. `work-slot merge` is a parallel entry
  point to the same post-Phase-A operations.
- **`work-slot create`** — unchanged.
- **`work-slot remove`** — unchanged (force-deletes without merge).

## Testing

- Create a multi-repo slot, run Phase A, verify `scan-ready` returns
  correct data
- `merge-slot` with clean rebase — verify ff-only merge and push succeed
- `merge-slot` with conflict — verify error output and no push
- `archive-slot` — verify worktrees removed, slot dir in attic with
  metadata preserved
- "all" with 2 slots — verify sequential processing, second slot rebases
  against updated main
- "all" with conflict on slot 2 — verify slot 1 is landed/archived, slot 2
  stops cleanly
- Pre-check failure — verify error when original repo not on main
- `list-slots --all` — verify archived slots appear

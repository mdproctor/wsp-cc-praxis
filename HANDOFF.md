# Session Handover

**Date:** 2026-11-19
**Where to work:** the permanent clone, `~/claude/hortora/clone/soredium/` (open the session in `wsp-soredium/`). The canonical repo `~/claude/hortora/soredium` is the sync point only.
**Branch:** `main` in both clone repos (no work branch open)
**Active queue item:** Hortora/soredium#584 (not started)
**Queue:** `~/claude/hortora/clone/soredium/.queue` (the clone root; see "One queue per area")

Earlier history of this file is in git (`git log -- HANDOFF.md`).

## State of the repos

- soredium `main` equals `origin/main` at `c161e0a`. cc-praxis `main` equals `origin/main` at `d45256b`.
- Everything below is landed, published and installed with `sync-local`.
- Sessions that loaded the `work` skill before 2026-11-19 keep the old behaviour until they reload it.

## What changed on 2026-11-19

| Issue | What it does now |
|---|---|
| Hortora/soredium#571 | `work wrap` and `work end` are pipelines in `project/work.py`. Wrap: review, diary, issues filed and queued, "is the item finished?", mark done, handoff. End: the same plus land what is marked done, promote, publish the covered repos and their blog entries. Scope is printed first and enforced by `work-end/scope.py`. A failed step stops the sequence; it is never skipped. |
| Hortora/soredium#574 | Reading the queue, detecting state and resolving context change no file. An older `.plan` is renamed only by the write that first creates `.queue`, and says so. A `design/` layout is reported as `LEGACY_LAYOUT=`; `plan_migrate.py to-root <workspace>` converts it on request. |
| Hortora/soredium#575, #576 | Every lifecycle action is recorded in the worklog by the script that performs it (`project/worktrace.py`). `python3 ~/.claude/skills/project/worktrace.py show limit=40` lists them. The test session uses a temporary worklog database. |
| Hortora/soredium#577 | One queue per area of work, never synced. `queue_io.queue_dir()` is the one rule. `project/queue_home.py adopt <slot>` switches one existing slot. New permanent clones get a root queue. |
| Hortora/soredium#578 | A test that had been failing on `main` is fixed. |

## One queue per area

`queue_io.queue_dir(workspace)` decides, in this order:

1. a slot or clone with `.slot` and the marker `.queue-root`: the area root
2. a workspace whose `.notes` worktree already holds `.queue`: `.notes`
3. a workspace with its own `.queue` or `.plan`: the workspace (never hidden by a worktree appearing)
4. a workspace with a `.notes` worktree and no queue yet: `.notes`
5. otherwise the workspace

Switched so far: the hortora clone only. Its workspace clone still carries the `.queue` and `.plan` it inherited from the repo; they are not read and must not be removed.

Not switched: every casehub slot. Each has several differing queues (slot 196 has five). Switch one at a time, dry run first, and only when its owner chooses which queue is the slot's:

```
python3 ~/.claude/skills/project/queue_home.py show  <slot-dir>
python3 ~/.claude/skills/project/queue_home.py adopt <slot-dir> from=<dir> [--apply]
```

The notes store (`project/notes_store.py`) is landed but nothing creates the worktree yet, so rules 2 and 4 are dormant. `casehub-ops` has a plain `.notes` directory holding `NOTES.md`; the store refuses it rather than replacing it.

## First live `work end`: what it showed

Ran on Hortora/soredium#578 in the switched clone. It landed, promoted, published soredium and cc-praxis, and published the diary entry.

- Fixed during the run: after an item is marked done the next item becomes active and `covers` changes, so a retry after a failed step restarted from the review. A wrap or end now resumes whatever the queue says.
- Still open, recorded on Hortora/soredium#584: the promote command printed for a done item carries the queue's current `covers` (the next item), not the done item's reference.
- The blog step published the new entry to `hortora.github.io` and `mdproctor.github.io`, and also refreshed 21 already-published soredium entries whose metadata had been repaired earlier in the day (frontmatter only).
- The workspace clone was left on the closed branch because `.work-progress` is a tracked file the pipeline itself modifies. It was moved to `main` by hand. Worth fixing: `.work-progress` should not be tracked.

## Queue

Work tracking is in the queue and on GitHub, not here: `python3 ~/.claude/skills/project/queue_home.py show ~/claude/hortora/clone/soredium`. Open items, in order: Hortora/soredium#584 (active), #579, #582, #580, #583, #581, #585. Not queued: Hortora/soredium#570, #573, #529, #560.

Added on 2026-11-20: five protocols in `docs/protocols/` (a read never writes; one queue per area; the script that acts records it; qualified issue references; a test must fail with the rule removed), six garden entries, and CLAUDE.md sections for the queue rule and `work wrap`/`work end`. The protocols and CLAUDE.md commit (`14aa10d`) is on the clone's project `main`, not yet landed: the next ceremony carries it.

## Things the next session must know

- Never run soredium tools inside a slot or workspace to "look": use `queue_home.py show`, file reads and read-only git. Reads are safe now, but that is how slot 207 was migrated by accident before the fix.
- `casehub/slots/207` holds a clone whose `main` has diverged from canonical by more than a thousand commits each way. The ceremony now refuses to carry a diverged base, but leave that slot alone.
- Publishing needs the user's go-ahead each time, except that typing `work end` is the go-ahead for exactly the repos its `SCOPE=` line lists.
- Issue references are always written in full, `owner/repo#N`, in stored fields and in commit messages.
- Unlanded branches from another session in canonical soredium worktrees: `issue-567-notes-store` (superseded by `issue-567-store-v2`, which is landed), `issue-547-cso-fix`, `issue-551-hook-setup`, `issue-529-multi-project-tests`. None may be deleted without the user's explicit yes.
- The worklog backup taken before test rows were removed: `~/.hortora/worklog.db.before-test-purge-20261119`.

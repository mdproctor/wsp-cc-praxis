# HANDOFF/NOTES Split — Design Spec

**Issue:** Hortora/soredium#209
**Branch:** issue-209-handoff-notes-split
**Date:** 2026-08-10

## Problem

HANDOFF.md serves two roles with different lifecycles: ephemeral session narrative (overwritten each session) and persistent notes (should accumulate across branches). The overwrite model destroys accumulated notes.

## Design

### HANDOFF.md stays ephemeral — no changes

Overwritten each session. Token budget under 200. Git history is the archive. Nothing changes here.

### NOTES.md on an orphan branch worktree

```
$WORKSPACE/.notes/           ← permanent git worktree
$WORKSPACE/.notes/NOTES.md   ← the file
```

The `notes` branch is an orphan — no shared history with main or any feature branch. Merge conflicts are impossible by design (same pattern as gh-pages).

**One-time setup** (in workspace-init):
```bash
git -C $WORKSPACE worktree add --orphan -b notes .notes
echo "# Notes" > $WORKSPACE/.notes/NOTES.md
git -C $WORKSPACE/.notes add NOTES.md
git -C $WORKSPACE/.notes commit -m "init notes"
```

**Format:** Append-only, date headers, plain markdown.
```markdown
# Notes

## 2026-08-10

- Remember to check auth token expiry after the migration
- When touching slot_manager, the test suite takes ~60s — run specific classes

## 2026-08-09

- engine reindex needed after next schema change
```

### Read at session start

work-start (Step 3c area) and work-continue read `$WORKSPACE/.notes/NOTES.md` if the worktree exists. Surface the most recent date section — don't dump the whole file.

```
Notes (2026-08-10):
  - Remember to check auth token expiry after the migration
  - When touching slot_manager, the test suite takes ~60s
```

### Writable mid-session

Append to `$WORKSPACE/.notes/NOTES.md` anytime — filesystem write, then commit to the notes branch:

```bash
# Append
echo "- new note" >> $WORKSPACE/.notes/NOTES.md
git -C $WORKSPACE/.notes add NOTES.md
git -C $WORKSPACE/.notes commit -m "note: <summary>"
```

No interaction with main, feature branches, or slots. The worktree is always on the `notes` branch.

### Default ON in wrap checklist

Flip handover item 8 from `[ ]` to `[x]`. Prompt: "anything to note for later?" User can skip with Enter.

## Files changed

| File | Change |
|------|--------|
| `workspace-init/SKILL.md` | Add orphan branch + worktree creation step |
| `handover/SKILL.md` | Flip item 8 default to ON |
| `work/SKILL.md` | Read NOTES.md in Step 4 continue path |
| `work-start/SKILL.md` | Read NOTES.md in resume path |

## What this does NOT change

- HANDOFF.md — stays ephemeral, overwritten each session
- work-end — no changes (NOTES.md is not branch-scoped, not cleaned up)
- slot_manager.py — no changes (notes worktree is on the original workspace, not in slot clones)

## Test plan

1. **workspace-init** creates `.notes/` worktree with orphan branch
2. **Append** adds entry with date header, commits to notes branch
3. **Read** at session start surfaces most recent section
4. **Multiple branches** — notes persist when workspace switches branches
5. **Worktree isolation** — `git branch` from main doesn't show `notes` in normal listing (it's orphan)

# HANDOFF/NOTES Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax
> for tracking. No code — all changes are SKILL.md edits.

**Focal issue:** #209 — Split HANDOFF.md into ephemeral handover + persistent NOTES.md
**Issue group:** #209

**Goal:** Persistent scratch notes on an orphan branch worktree, surfaced at decision points, writable anytime.

**Architecture:** Orphan `notes` branch with permanent worktree at `$WORKSPACE/.notes/`. Append-only file, committed at wrap time. Read when browsing ("what next?"), not when tasked.

**Format:** Date headers, optional repo tags.
```markdown
# Notes

## 2026-08-10

- Remember to check auth token expiry after the migration
- [engine] reindex needed after next schema change
- When touching slot_manager, the test suite takes ~60s
```

No repo tag = primary workspace context.

## Global Constraints

- All changes in SKILL.md files under `~/claude/hortora/soredium/`
- After each edit, run `python3 scripts/claude-skill sync-local --all -y`
- No Python scripts, no tests — pure skill documentation

---

### Task 1: Add notes worktree creation to workspace-init

**Files:**
- Modify: `workspace-init/SKILL.md`

- [ ] **Step 1: Read current workspace-init to find insertion point**

Read `workspace-init/SKILL.md`, find the section after git hooks setup and before the completion report.

- [ ] **Step 2: Add notes worktree step**

Add a new step after symlink creation:

```markdown
### Step N — Notes worktree

Create an orphan branch worktree for persistent scratch notes:

```bash
# Idempotent — skip if .notes/ already exists
if [ ! -d "$WORKSPACE/.notes" ]; then
    git -C "$WORKSPACE" worktree add --orphan -b notes .notes
    echo "# Notes" > "$WORKSPACE/.notes/NOTES.md"
    git -C "$WORKSPACE/.notes" add NOTES.md
    git -C "$WORKSPACE/.notes" commit -m "init notes"
fi
```

This creates `$WORKSPACE/.notes/NOTES.md` on an orphan `notes` branch.
The worktree is permanent — always accessible regardless of which branch
the workspace is on. No shared history with main or feature branches.

**Format:** Append-only, date headers, optional `[repo]` tags.
```

- [ ] **Step 3: Add .notes to .gitignore or git/info/exclude**

The `.notes/` directory should be excluded from the main worktree's tracking. Add to the workspace-init step that handles exclusions:

```markdown
# Exclude notes worktree from main tracking
echo ".notes" >> "$WORKSPACE/.git/info/exclude"
```

- [ ] **Step 4: Verify**

Read the modified SKILL.md and confirm the step is idempotent (checks for existing `.notes/` before creating).

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add workspace-init/SKILL.md
git -C ~/claude/hortora/soredium commit -m "feat(#209): add notes worktree creation to workspace-init

Refs #209"
```

---

### Task 2: Flip handover wrap checklist item 8 to ON + commit at wrap

**Files:**
- Modify: `handover/SKILL.md`

- [ ] **Step 1: Read current handover SKILL.md**

Find item 8 in the wrap checklist and the item 8 execution section.

- [ ] **Step 2: Change default from OFF to ON**

Change:
```markdown
[ ] 8  notes             anything to note for later? (appends to $WORKSPACE/NOTES.md)
```
To:
```markdown
[x] 8  notes             anything to note for later? (appends to $WORKSPACE/.notes/NOTES.md)
```

- [ ] **Step 3: Update the item 8 execution section**

Update the notes execution section (around line 297) to specify the format and commit:

```markdown
8. notes — prompt "anything to note for later?" If yes:
   - Append entries to `$WORKSPACE/.notes/NOTES.md` under today's date header
   - Optional `[repo]` prefix for repo-specific notes (no prefix = primary)
   - Commit to orphan branch: `git -C $WORKSPACE/.notes add NOTES.md && git -C $WORKSPACE/.notes commit -m "notes: wrap"`
   - If `.notes/` worktree doesn't exist, skip with: "No notes worktree — run workspace-init to set up."
```

- [ ] **Step 4: Update SESSION-BOUND-ITEMS documentation**

Update the session-bound items block to clarify item 8 is NOT session-bound (it writes to persistent storage, not session context):

```markdown
Item 8 (notes) captures persistent scratch items to `$WORKSPACE/.notes/NOTES.md` —
things to come back to later, observations that span sessions and branches, notes not
actionable enough to be issues. Append-only with date headers and optional [repo] tags.
Committed to the orphan `notes` branch at wrap time. Not session-bound — can be
deferred, but defaulting ON ensures notes are captured while context is fresh.
```

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/hortora/soredium add handover/SKILL.md
git -C ~/claude/hortora/soredium commit -m "feat(#209): default notes ON in wrap checklist, commit to orphan branch

Refs #209"
```

---

### Task 3: Surface notes in work skill (what-next mode)

**Files:**
- Modify: `work/SKILL.md`

- [ ] **Step 1: Read current work SKILL.md**

Find Step 2a (what-next recommendation).

- [ ] **Step 2: Add notes surfacing after what-next results**

After the what-next results are presented (after step 3 in Step 2a), add:

```markdown
3b. **Surface notes (if present):**
    If `$WORKSPACE/.notes/NOTES.md` exists, read the most recent date
    section and surface it below the what-next recommendations:

    ```
    Notes (2026-08-10):
      - Remember to check auth token expiry after the migration
      - [engine] reindex needed after next schema change
    ```

    Show only the most recent date section. If the file doesn't exist
    or is empty, skip silently.
```

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/hortora/soredium add work/SKILL.md
git -C ~/claude/hortora/soredium commit -m "feat(#209): surface notes in what-next mode

Refs #209"
```

---

### Task 4: Surface notes after work-end close summary

**Files:**
- Modify: `work-end/SKILL.md`

- [ ] **Step 1: Read current work-end SKILL.md**

Find Step 5.7 (session close summary).

- [ ] **Step 2: Add notes surfacing after the close summary**

After the close summary block, add:

```markdown
### 5.8 Surface notes

If `$WORKSPACE/.notes/NOTES.md` exists and has content, surface the
most recent date section after the close summary:

```
Notes (2026-08-10):
  - Remember to check auth token expiry after the migration
  - [engine] reindex needed after next schema change
```

This reminds the user of persistent context before they decide what
to do next. Skip silently if no notes exist.
```

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/hortora/soredium add work-end/SKILL.md
git -C ~/claude/hortora/soredium commit -m "feat(#209): surface notes after work-end close summary

Refs #209"
```

---

### Task 5: Include notes in /brief orientation + sync skills

**Files:**
- Modify: `brief/SKILL.md`

- [ ] **Step 1: Read current brief SKILL.md**

Find the output sections.

- [ ] **Step 2: Add notes to orientation output**

Add a notes section to the brief output, after health status:

```markdown
### Notes

If `$WORKSPACE/.notes/NOTES.md` exists, include the most recent date
section in the brief output:

```
NOTES=2026-08-10: 3 items (2 general, 1 engine)
```

Show the count and repo breakdown, not the full content. The user
can read the file if they want details.
```

- [ ] **Step 3: Commit**

```bash
git -C ~/claude/hortora/soredium add brief/SKILL.md
git -C ~/claude/hortora/soredium commit -m "feat(#209): include notes count in /brief orientation

Refs #209"
```

- [ ] **Step 4: Sync all skills**

```bash
python3 ~/claude/hortora/soredium/scripts/claude-skill sync-local --all -y
```

- [ ] **Step 5: Run commit-tier validators**

```bash
python3 scripts/validate_all.py --tier commit
```

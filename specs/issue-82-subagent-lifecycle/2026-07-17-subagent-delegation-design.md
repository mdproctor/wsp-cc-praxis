# Delegate Mechanical work-end Steps to Subagents

**Issue:** #82
**Date:** 2026-07-17
**Status:** Approved

## Problem

work-end accumulates 2200–6800 tokens of intermediate tool output (git logs,
file listings, issue API responses, journal content) across its 12+ steps.
By the time it reaches squash analysis (Step 8j), the context window is
stressed — Claude frequently suggests continuing in a new session, which
loses the context needed for remaining steps.

## Solution

Replace three groups of read-heavy mechanical steps with subagent dispatches.
Each subagent runs in an isolated context, absorbs all intermediate output,
and returns a compact structured JSON result. The parent context only sees
the dispatch call (~150–200 tokens) and the return value (~200–300 tokens),
instead of the full raw output (700–3000 tokens per group).

## Scope

**In scope:** work-end only (Approach A — surgical).
**Out of scope:** handover (#83), work-start, git-commit. Handover follows
once the pattern is proven.

## Delegation Points

### 1. Branch Reconnaissance (replaces Steps 1 + 4 + 5)

**When:** After path resolution, before pre-close sweep (Step 3b).

**What the subagent does:**
- Fetches issue title(s) from GitHub for all COVERS issues
- Runs `git log --oneline` for branch commits
- Runs `git diff --shortstat` for change summary
- Lists workspace artifact directories (adr/, blog/, specs/, snapshots/, plans/)
- Reads JOURNAL.md
- Computes current ARC42STORIES.MD section hashes via `section_hashes.py`
- Compares against baseline hashes from .meta
- Counts anchored vs unanchored journal entries

**Prompt parameters (from ctx.py):**
- WORKSPACE, PROJECT, BRANCH_NAME, BASE_BRANCH
- ISSUE_N, COVERS, ISSUE_REPO (for GitHub API)
- PROJECT_SHA (for diff range)
- META_SECTION_HASHES (pipe-separated baseline hashes from ctx.py output)
- DESIGN_REPO_KEY (workspace / project / cross-repo:name — determines where ARC42STORIES.MD lives)

**Return format:**
```json
{
  "issues": [{"number": 82, "title": "...", "state": "OPEN"}],
  "commits": [{"sha": "abc1234", "message": "feat: ..."}],
  "commit_count": 5,
  "diff_stats": "4 files changed, 120 insertions, 30 deletions",
  "journal_entry_count": 3,
  "artifacts": {
    "adrs": ["0005-subagent-delegation.md"],
    "blogs": [],
    "specs": ["2026-07-17-subagent-design.md"],
    "snapshots": [],
    "plans": ["implementation-plan.md"]
  },
  "journal_validation": {
    "arc42_exists": true,
    "section_drift": [{"section": "## S5", "stored_hash": "abc", "current_hash": "def"}],
    "anchored_entries": 2,
    "unanchored_entries": 1,
    "entries_without_anchors": ["### Entry about X"],
    "empty_journal": false
  }
}
```

**Downstream consumers:**
- Step 7 (close plan) uses `commits`, `diff_stats`, `artifacts`, `journal_entry_count`
- Step 5 decisions use `journal_validation` (drift → offer fix/skip/abort)
- Step 8h (final report) uses `artifacts` for the artifact summary

**Error handling:** If the subagent returns empty or malformed JSON, warn
the user and fall back to running Steps 1, 4, 5 inline. This is graceful
degradation — the skill works exactly as before, just without the context
savings.

**Model:** Sonnet (mechanical — file listing, git commands, hash comparison).

**Estimated savings:** 700–1800 tokens of raw output → ~350 tokens (prompt + return).

---

### 2. Squash Analysis (replaces analysis phase of Step 8j)

**When:** After rebase onto base branch, before squash execution.

**What the subagent does:**
- Reads the commit range (`blessed-remote/base..HEAD`)
- Self-reads `~/.claude/skills/git-squash/squash-policy.md` for classification rules
- Runs squash-policy.md's strategy detection (merge-commit PRs → reconstruction, else scope clustering or flat)
- Classifies each commit (SQUASH / KEEP / REVERT_PAIR)
- Proposes groups with draft squash messages
- Collects sub-issue references (`Closes #N`, `Fixes #N`, `Resolves #N`) from branch commits
- Cross-references against COVERS to identify refs that would be lost in squash

**Prompt parameters:**
- PROJECT path
- BASE_BRANCH, BRANCH_NAME
- COVERS (comma-separated issue numbers)
- Blessed remote name (upstream or origin)
- Path to squash-policy.md (subagent reads it itself)

**Return format:**
```json
{
  "total_commits": 8,
  "strategy": "D",
  "groups": [
    {
      "label": "feat(#82): delegate lifecycle steps to subagents",
      "action": "KEEP",
      "commits": [
        {"sha": "abc1234", "message": "feat: ...", "classification": "KEEP"}
      ],
      "proposed_message": "feat(#82): delegate mechanical work-end steps to subagents\n\nRefs #82"
    },
    {
      "label": "squash: fixups and docs",
      "action": "SQUASH",
      "commits": [
        {"sha": "def5678", "message": "fix: typo", "classification": "SQUASH"},
        {"sha": "ghi9012", "message": "docs: update readme", "classification": "SQUASH"}
      ],
      "proposed_message": null
    }
  ],
  "sub_issue_refs": ["#83"],
  "refs_not_in_covers": ["#83"]
}
```

**Downstream consumers:**
- Parent presents the plan for user approval (accept / edit / reject)
- On approval, parent executes the squash using git-squash with the approved groups
- `refs_not_in_covers` are appended as `Closes #N` trailers to the squash message

**Error handling:** If the subagent returns empty or malformed JSON, warn
and offer: (a) invoke `/git-squash` manually, (b) skip squash entirely
(user takes responsibility for noisy history).

**Model:** Sonnet (commit classification is pattern matching, not reasoning).

**Estimated savings:** 1000–3000 tokens → ~500 tokens (prompt + return). This
is the single highest-value delegation point.

---

### 3. Hygiene Scan (replaces Step 8i)

**When:** After execution steps 8a–8h, before squash (8j).

**What the subagent does:**
- Verifies all workspace blog entries exist at the blog destination
  (reads blog-routing.yaml, compares file lists)
- Checks for Flyway V-number collisions if Flyway was used on this branch
- Lists workspace branches with no commits in 7 days (stale)
- For each workspace branch with `design/EPIC-CLOSED.md`:
  - Checks whether blog/spec files reached workspace main (unrecovered artifacts)
  - Checks whether the corresponding project branch has a `chore: branch closed` stamp
- Skips the CURRENT branch being closed (it's handled by 8j)

**Prompt parameters:**
- WORKSPACE, PROJECT, BRANCH_NAME
- Blog destination path (from routing resolution in Step 3)
- Whether Flyway was used on this branch (from .meta)

**Return format:**
```json
{
  "unpublished_blogs": ["2026-07-15-entry.md"],
  "flyway_conflicts": [],
  "stale_branches": [
    {"branch": "issue-71-old", "last_commit_age": "12 days"}
  ],
  "unrecovered_artifacts": [
    {"branch": "issue-50-closed", "type": "blog", "file": "2026-06-01-entry.md"},
    {"branch": "issue-50-closed", "type": "spec", "file": "design-doc.md"}
  ],
  "unstamped_branches": [
    {"branch": "issue-50-closed", "has_epic_closed": true, "project_branch_exists": true}
  ]
}
```

**Downstream consumers:**
- `unpublished_blogs` non-empty → parent blocks and returns to 8g
- `unrecovered_artifacts` → parent offers cherry-pick per item (user confirms each)
- `unstamped_branches` → parent offers to stamp per branch (user confirms each)
- `stale_branches` → parent reports (informational only)
- `flyway_conflicts` → parent warns (informational)

**Error handling:** If the subagent returns empty or malformed JSON, warn
and skip. Hygiene is advisory — it catches drift but doesn't block the close.
Exception: if blog verification cannot run, warn explicitly since unpublished
blogs DO block.

**Model:** Sonnet (file comparisons, git branch scanning).

**Estimated savings:** 500–2000 tokens → ~350 tokens (prompt + return).

---

## Dispatch Template Pattern

All three delegation points follow the same pattern in work-end's SKILL.md:

```markdown
### Step N — [Name] (delegated to subagent)

Dispatch a read-only Sonnet subagent. The subagent's execution stays in
its own context — only the JSON return enters the parent window.

**Dispatch:**

Agent(
  description: "[short label]",
  prompt: "[inline template with {PLACEHOLDER} values substituted from ctx.py]",
  model: "sonnet"
)

**Return format:**
[JSON schema]

**On return:**
- Validate JSON shape (required keys present)
- If malformed: [specific fallback per delegation point]
- Use [specific fields] for [downstream step]
```

The prompt is inline in the skill — no separate prompt files. When the
subagent needs policy knowledge (squash analysis), the prompt tells it
to read the existing source-of-truth file (squash-policy.md). No
convention duplication.

## Modification Summary

### Lines removed from work-end SKILL.md
- Step 1 branch summary block (~30 lines)
- Step 4 artifact inventory block (~20 lines)
- Step 5 journal validation block (~30 lines)
- Step 8i hygiene scan block (~60 lines)
- Step 8j analysis interleaved with execution (~50 lines of analysis logic)
- **Total: ~190 lines removed**

### Lines added
- Branch reconnaissance dispatch block (~40 lines)
- Squash analysis dispatch block (~35 lines)
- Hygiene scan dispatch block (~30 lines)
- **Total: ~105 lines added**

### Net: skill shrinks by ~85 lines

### Steps unchanged
- Path resolution (ctx.py) — already cheap
- Pre-conditions — already cheap
- Flyway re-scan (Step 2) — one script call
- Routing resolution (Step 3) — cheap + interactive
- Pre-close sweep (Step 3b) — interactive (user toggles)
- Code review (Step 3c) — interactive, invokes another skill
- Spec selection (Step 6) — interactive
- Close plan (Step 7) — synthesis, now from compact JSON
- Execution steps 8a–8h — destructive, need user confirmation
- Steps 9–12 — interactive or synthesis

## Total Context Savings

| Delegation | Before (tokens) | After (tokens) | Savings |
|-----------|----------------|----------------|---------|
| Branch reconnaissance | 700–1800 | ~350 | 350–1450 |
| Squash analysis | 1000–3000 | ~500 | 500–2500 |
| Hygiene scan | 500–2000 | ~350 | 150–1650 |
| **Total** | **2200–6800** | **~1200** | **1000–5600** |

The parent context window reclaims 1000–5600 tokens that previously
accumulated from intermediate tool output. This is the difference between
work-end completing in one session vs suggesting "let's continue later."

## Constraints

- Subagents cannot invoke skills or read CLAUDE.md — all instructions
  must be in the dispatch prompt or in files the prompt points to.
- Subagents should not write files or make destructive changes — they
  are read-only analysts. All mutations happen in the parent.
- Return format must be validated before use — a malformed return should
  never corrupt the close flow.

## Testing

Each delegation point can be tested by:
1. Running work-end on a test branch with the delegation active
2. Comparing the JSON return against manually computed expected values
3. Verifying the close plan (Step 7) renders correctly from the JSON
4. Verifying fallback behaviour when the subagent is artificially failed

No automated test infrastructure exists for skill execution — testing is
manual and observational. The existing `validate_all.py` validators check
SKILL.md structure, not runtime behaviour.

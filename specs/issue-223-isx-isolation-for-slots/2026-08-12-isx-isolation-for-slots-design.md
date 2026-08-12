# ISX Isolation for Slots — Design Spec

**Issue:** Hortora/soredium#223
**Branch:** issue-223-isx-isolation-for-slots
**Date:** 2026-08-12

## Overview

ISX isolation adds an optional container layer on top of the existing slot
infrastructure. A slot with `--isx` has **dual existence**: an ISX container
for working (implementation, testing, builds) and lightweight local clones
for landing (sync, push, close).

```
work-slot create #42 --isx
  ├── Local slot (existing)        ← landing zone
  │   ├── slots/<N>/<repo>/        ← git clone --shared
  │   ├── .slot                    ← metadata (+ isolation section)
  │   └── .plan                    ← issue queue
  └── ISX container                ← working environment
      └── /home/agentuser/<repo>/  ← template-provisioned repos
```

### Lifecycle

1. `work-slot create --isx` — creates local slot + ISX instance, wires `isx://` remotes
2. User works inside ISX (`isx shell`) — implements, tests, commits
3. `work-slot sync` — fetches committed work from ISX into local clones
4. `work end` — runs on local clones (staleness pre-flight check first), existing close machinery
5. `work-slot remove` — `isx destroy` + archive local slot

No changes to `work-end` internals — it operates on local clones after sync,
same as today. The only `work-end` addition is the staleness pre-flight.

### ISX commands used (all blessed upstream)

| Command | Purpose |
|---------|---------|
| `isx templates list` | Enumerate available templates for selection |
| `isx branch <name> --from <template>` | Create container instance |
| `isx shell <name>` | Open shell in container |
| `isx destroy <name>` | Destroy container on slot removal |
| `git-remote-isx` + `isx git-remote-helper` | Transport for `isx://` git remotes |

`isx sync` is a local fork extension (not upstream) and is NOT used.

---

## `slot_manager.py` Changes

### 1. `create-slot` with `--isx`

After normal slot creation completes (local clones, `.m2`, `.slot`, `.plan`),
if `--isx` is specified:

1. **Pre-flight:** Check `isx` is on PATH. Abort **before** local slot
   creation if missing — no orphaned half-created slot.
2. **Template selection:** Run `isx templates list`, present choices, user picks.
3. **Instance naming:** Default to branch name, truncated to 63 chars if
   needed (Incus limit). User can override. Store in `.slot`.
4. **Create instance:** `isx branch <instance-name> --from <template>`
5. **Wire remotes:** For each repo in the slot:
   `git -C <slot-clone> remote add isx isx://<instance>/home/agentuser/<repo-name>`
6. **Write isolation metadata to `.slot`:**
   ```
   ## Isolation
   type: isx
   instance: <instance-name>
   template: <template-name>
   ```

New CLI args for `create-slot`: `isx=yes template=<name> instance=<name>`

### 2. `sync-isx` subcommand

New subcommand. Invocable two ways:
- `work-slot sync <N>` — from anywhere, by slot number
- `work-slot sync` — from inside a slot clone (auto-detect via CWD)

For each project repo in the slot:
1. `git -C <local-clone> fetch isx <branch>`
2. `git -C <local-clone> merge --ff-only isx/<branch>`

**Error paths:**
- Non-ISX slot (no `type: isx` in `.slot`) → error: "This slot has no ISX isolation."
- Instance not running → error from `git fetch` (ISX protocol reports it)
- `--ff-only` fails (diverged histories) → report which repo and stop
- No `isx` remote on a repo → skip with warning (repo may have been added after ISX creation)

Convention-based path resolution: container repos assumed at
`/home/agentuser/<repo-name>`. If wrong, `git fetch` fails with a clear
error pointing to the path.

### 3. `remove-slot` / `archive-slot` ISX cleanup

In existing `remove_slot()` and `archive_slot()`, before archiving:

1. Read `.slot` for `type: isx` and `instance:` name
2. If present, run `isx destroy <instance-name>`
3. Warn and continue if destroy fails (instance may already be gone)

### 4. Staleness pre-flight for `work-end`

When `.slot` has `type: isx`, before the close sequence proceeds:

For each repo:
1. `git ls-remote isx://<instance>/home/agentuser/<repo-name>` — get remote HEAD
2. Compare against local clone HEAD
3. If they differ: warn with commit count delta, ask to continue or sync first

Non-blocking — user can proceed if they intentionally want to skip ISX commits.
This is a pre-flight check in the close sequence, not a change to work-end's
internal machinery.

---

## `.slot` Metadata

Existing `.slot` format gains a new optional section:

```markdown
# Slot 7 — issue-42-fix-scoring

## Issue
Hortora/soredium#42
Covers: 42

## What to do
Fix the scoring engine regression

## Repos
- soredium (primary)

## Isolation
type: isx
instance: issue-42-fix-scoring
template: tpl-java

## Created
2026-08-12, branch: issue-42-fix-scoring
```

The `## Isolation` section is only present for ISX slots. All ISX-aware
code paths check for its presence before acting.

---

## SKILL.md Changes

### `work-slot/SKILL.md`

**Step 1 (Gather input)** — add to input list:
- `--isx` — create with ISX container isolation
- If `--isx`: show `isx templates list` output, user picks template.
  Optional instance name override (defaults to branch name, truncated to 63 chars).

**Step 4 (Create the slot)** — after `create-slot`, if `--isx`:
```bash
which isx || { echo "ISX not installed."; exit 1; }
isx branch <instance-name> --from <template>
# Per repo:
git -C <slot-clone> remote add isx isx://<instance>/home/agentuser/<repo-name>
```

**Step 8 (Shell offering)** — for ISX slots, offer both:
1. iTerm tab for host-side ops (sync, work-end)
2. `isx shell <instance>` for container work

**New section: `work-slot sync`** — document the subcommand with both
invocation patterns.

**Existing section: `work-slot remove`** — add note that ISX instances
are destroyed before archiving.

### No changes to `work-end/SKILL.md`

Staleness check is implemented in code, not as a new skill step.

### No changes to `work-start/SKILL.md`

ISX slots don't change the resume path — scaffold is in the local clone.

---

## Testing

Per `externalised-scripts-require-tests` protocol:

| Function | Tests |
|----------|-------|
| ISX extension of `create_slot` | Happy path, missing `isx` binary, invalid template, instance name truncation (63 char limit) |
| `sync_isx()` | Happy path (ff-only succeeds), non-ISX slot error, diverged histories error, missing remote skip |
| ISX cleanup in remove/archive | Instance exists, instance already gone, no ISX metadata in `.slot` |
| Staleness pre-flight | Heads match (no warning), heads differ (warning), instance unreachable |

Tests mock `subprocess.run` calls to `isx` and `git` — no actual containers
in unit tests.

---

## Scope Boundaries

### In scope

- `--isx` flag on `create-slot` with template selection and instance creation
- `sync-isx` subcommand (slot number and CWD auto-detect)
- ISX destroy on `remove-slot` / `archive-slot`
- `.slot` isolation metadata (`## Isolation` section)
- Staleness pre-flight in work-end close sequence
- Instance name truncation to 63 chars
- ISX availability pre-flight
- SKILL.md updates for work-slot
- Unit tests for all new code paths

### Not in scope

- Running `work end` inside ISX — always host-side after sync
- Running sweeps inside ISX — need LLM session on host
- Multi-container slots — one ISX instance per slot
- Casehub integration — soredium/hortora first
- Template auto-creation — user creates templates separately via `isx project create`
- `isx sync` command — local fork extension, not blessed upstream

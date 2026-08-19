# Slot Workspace Convergence — Decisions

## D1: Workspace layout in slots

**Choice:** Per-repo workspace clones, flat in the slot directory, using actual git repo names (e.g., `wsp-casehub-connectors/`)
**Alternatives:**
- Family workspace clone with subdirectories (current) — ambiguous ownership, cross-cutting vs per-repo split unclear
- Fresh empty directories — loses git history and merge path
**Rationale:** Identical structure to outside slots. Skills follow `wksp` symlink and see the same shape regardless of context. No slot-specific workspace logic needed.
**Trade-offs:** More clones per slot (one per repo instead of one family clone). Marginal disk cost with `--shared`.
**Sources:** Slot 138 disk structure, `slot_manager.py` create-slot function, `topology.py` layout detection
**Exploration:** deep-analysis
**Status:** captured

## D2: Naming convention

**Choice:** Use actual git repo names from the workspace remote (e.g., `wsp-casehub-connectors`), no `work-` prefix
**Alternatives:**
- `work-connectors/` prefix — collides with `work-work` pattern, fragile symlinks, historical problems
**Rationale:** Names already unique (GitHub repo names). No translation layer needed. Eliminates the `work-` prefix problems that caused symlink corruption.
**Trade-offs:** Longer directory names. Slot directory listing is less visually grouped (project and workspace interleaved alphabetically).
**Sources:** `git remote -v` on workspace repos showing `wsp-casehub-*` naming
**Exploration:** deep-analysis
**Status:** captured

## D3: Cross-cutting artifacts

**Choice:** Cross-cutting files go in primary's workspace. Tagged if the file format supports tagging. Repo-specific files go in that repo's workspace.
**Alternatives:**
- Separate cross-cutting directory — another location to check, more ambiguity
- Duplicate across workspaces — violates single-source-of-truth
**Rationale:** One file, one place. Tagging handles multi-repo scope without duplication.
**Trade-offs:** Primary workspace carries more content. Consumers must check tags to know if a file in primary's workspace applies to them.
**Sources:** Conversation — user's rule: "cross cutting files go in primary, will be cross tagged if the file supports tagging"
**Exploration:** deep-analysis
**Status:** captured

## D4: Migration strategy

**Choice:** Three-phase approach:
1. Bulk-migrate archived slots (attic) now
2. Migrate active slots by moving files from `work-casehub/<repo>/` to `wsp-casehub-<repo>/`, then replace original directory with symlink for backwards compatibility with loaded sessions
3. Update skills to single code path only after all slots conform

**Alternatives:**
- Two code paths coexisting (old + new) — rejected as reliability disaster
- Flag-day cutover — too risky for active slots with loaded sessions
**Rationale:** Symlink bridge lets old sessions continue working while new code uses the correct paths. No period where two code paths coexist in skills.
**Trade-offs:** Migration script complexity (must handle file moves, symlink creation, git state). Temporary symlinks in active slots until they close.
**Depends on:** D1 (layout), D2 (naming)
**Sources:** 21 active slots on disk, slot_manager.py archive logic
**Exploration:** deep-analysis
**Status:** captured

## D5: Work-end convergence

**Choice:** Same per-repo flow regardless of slot: promote → squash → merge workspace → merge project → push. Mechanical code shared between slot and non-slot paths.
**Alternatives:**
- Keep separate merge-slot orchestrator — rejected as unnecessary divergence that causes unreliable behavior
**Rationale:** With per-repo workspace clones, the workspace interaction is identical. The only slot-specific aspect is transport (two-hop push: clone → original → remote) which is a mechanical concern, not a structural one.
**Trade-offs:** merge-slot logic must be decomposed — some parts (two-hop push, SHA verification) remain as transport helpers, not as a separate orchestration path.
**Depends on:** D1 (layout), D2 (naming)
**Sources:** work-end/SKILL.md divergence audit (12 points), slot_manager.py merge-slot function
**Exploration:** deep-analysis
**Status:** captured

## D6: Migration safety gate

**Choice:** Only auto-migrate slots idle for >1 hour (no file changes). Recently-active slots flagged for manual review and discussed with user before migrating.
**Alternatives:**
- Migrate all at once — risks corrupting a slot with an active session
- Only migrate at work-end — delays convergence, keeps two code paths longer
**Rationale:** Avoids migrating a slot mid-session where another Claude instance might be writing to old paths. User reviews recently-active slots to confirm safety.
**Trade-offs:** Some slots may remain unmigrated until user reviews them. Migration is not fully automated.
**Depends on:** D4 (migration strategy)
**Sources:** Conversation — user's safety constraint
**Exploration:** deep-analysis
**Status:** captured

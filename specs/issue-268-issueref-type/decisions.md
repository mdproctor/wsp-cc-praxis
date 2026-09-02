# Decisions — IssueRef type (#268)

## D1: Scope

**Choice:** Full stack — plan_manager.py + events.py + commands/registry.py + commands/*.py + work_health.py + work_chain.py + worklog boundary functions in _emit_issue_events
**Alternatives:**
- Core only (plan_manager + direct consumers) — delays events.py, leaves bare ints in TUI/CLI contract
- plan_manager only — minimises change but perpetuates the split in downstream code
**Rationale:** One pass eliminates all bare `int` issue references across the domain layer. The worklog database retains its `(int, str)` schema — IssueRef decomposes at the persistence boundary (see D6). Context.issue in registry.py and all event dataclasses must change to `IssueRef | None`. No follow-up issue needed.
**Trade-offs:** Larger change surface. More test updates. The change is mostly mechanical (field renames, type changes) with one algorithmic change in check_plan_state (see D7).
**Sources:** issue #268 body, plan_manager.py, events.py, commands/registry.py, work_health.py, commands/next.py, commands/brief.py
**Exploration:** quick
**Status:** revised (R1-02: expanded scope to include registry.py and worklog boundary; R1-03: acknowledged algorithmic change in check_plan_state)

## D2: IssueRef implementation

**Choice:** Frozen dataclass with construction-time validation and case normalization
**Alternatives:**
- NewType wrapper around string — no construction validation, relies on convention
- Validation on existing fields (__post_init__ on QueueItem/LeafItem) — scatters the invariant across multiple classes
- Computed property on QueueItem (`@property def ref`) keeping separate `issue_number`/`repo` fields — rejected: defeats IssueRef's purpose by leaving individually-accessible bare fields that code can bypass; creates impedance mismatch in the domain model while the persistence layer's `(int, str)` decomposition is already handled at the boundary (D6)
**Rationale:** Frozen dataclass gives immutability + hash/equality for free. Construction-time validation means invalid refs can't exist — caught at the source. `__post_init__` normalises `repo` to lowercase via `object.__setattr__(self, 'repo', repo.lower())` — GitHub repos are case-insensitive, so `Hortora/soredium` and `hortora/soredium` must compare equal. `parse()` classmethod replaces scattered extraction of repo+number from regex match groups (not the line-level `_ITEM_RE` regex itself, which still parses the full queue line structure).
**Trade-offs:** 73+ references to update in plan_manager.py alone, plus downstream. But this is mechanical.
**Sources:** issue #268 body (proposed IssueRef design), GE-20260811-7e119c (cross-repo resolution bug)
**Exploration:** quick
**Status:** revised (R1-05: added case normalization; R1-06: added computed-property alternative with rejection; R1-07: clarified parse() scope)

## D3: Backward compatibility — hard fail

**Choice:** Parser raises on bare `#N` in .plan files — forces all .plan files to be repo-qualified. Prerequisite: run `migrate_plan_repos.py` across all workspace trees before deploying the strict parser.
**Alternatives:**
- Warn and backfill from issue-repo — rejected: backfill from `issue-repo` is *wrong for cross-repo queues* because `issue-repo` identifies the epic's repo, not each child's repo. This is the exact silent corruption bug (#268) that motivated this issue. Backfill works for single-repo queues but produces false positives and wrong completions for cross-repo queues.
- Silent backfill — same as today with IssueRef wrapping
**Rationale:** Clean break. No ambiguity about which repo an issue belongs to. `_write_item()` already raises on empty repo — any .plan round-tripped since the write guard was added is already repo-qualified. The existing `migrate_plan_repos.py` script handles the remaining cases: it infers repos from .slot files, directory structure, or CLAUDE.md, then parses and rewrites the .plan to inject repo prefixes.
**Trade-offs:** Any active .plan with bare numbers will fail to parse until repo-qualified. Migration script must run first. In practice most active .plan files are already repo-qualified due to the `_write_item()` guard.
**Sources:** issue #268 body, scripts/migrate_plan_repos.py
**Exploration:** quick
**Status:** revised (R1-09: added migration script prerequisite; R1-10: corrected rejection rationale for warn-and-backfill)

## D4: Event field types in events.py

**Choice:** IssueRef directly — `issue: IssueRef | None` on event dataclasses
**Alternatives:**
- String field `issue: str` with `IssueRef.__str__()` format — rejected: events are in-process Python objects, not serialised to JSON or any wire format. Using strings forces consumers to re-parse for decomposition (e.g., `ref.repo` for `gh issue view`), which re-introduces the scattered parsing that D2 eliminates.
- Nested object `{repo, number}` — N/A since events are dataclasses, not dicts
- Two fields `issue: 268, issue_repo: "Hortora/soredium"` — perpetuates the split
**Rationale:** Events are in-process Python dataclasses consumed by command code in the same process. They have no serialisation layer — `BriefReady`, `PlanAdvanced`, etc. are constructed in command functions and consumed directly. Carrying `IssueRef` gives consumers type-safe access to `.repo` and `.number` for decomposition, and `str(ref)` for display. Strictly more capable than either string or bare int.
**Trade-offs:** Callers that currently read `event.issue` as `int` must update to `event.issue.number` or `event.issue.repo`. This is the breakage that forces correctness.
**Sources:** commands/events.py, commands/next.py, commands/brief.py
**Exploration:** quick
**Status:** revised (R1-12: corrected from string to IssueRef after verifying events are in-process, not serialised; R1-13: IssueRef is strictly more capable than string)

## D5: Git hook enforcement

**Choice:** New git pre-commit hook in workspace repos validates .plan file content — every queue line must have `owner/repo#N`
**Alternatives:**
- Both git pre-commit + PreToolUse hook — redundant if pre-commit catches everything
- PreToolUse only (the existing `lifecycle-guard.sh`) — doesn't catch git operations outside Claude. The existing hook is a Claude Code PreToolUse hook that blocks rogue lifecycle operations (merges to main, artifact moves, direct .plan writes). It does NOT validate .plan content format.
**Rationale:** Git pre-commit hook catches any source of bad data, including manual edits that bypass the PreToolUse lifecycle guard. Runs on every `git commit` in the workspace repo where .plan lives.
**Location:** Workspace repo `.git/hooks/pre-commit`. Installed by `workspace-init` for new workspaces. For existing workspaces, a one-time install script sets up the hook (can be run alongside `migrate_plan_repos.py` from D3).
**Trade-offs:** Adds a new hook to workspace repos (separate from the existing PreToolUse `lifecycle-guard.sh`). But .plan format validation is fast (regex scan of staged .plan files).
**Sources:** User request, existing lifecycle-guard.sh (PreToolUse hook), workspace-init skill
**Exploration:** quick
**Status:** revised (R1-15: clarified this is a new git pre-commit hook, not the existing PreToolUse hook; R1-16: specified hook location and installation mechanism)

## D6: Worklog database boundary

**Choice:** Worklog DB schema stays as `(issue_number INTEGER, issue_repo TEXT)` composite keys. IssueRef decomposes at the persistence boundary.
**Alternatives:**
- Migrate to single `issue_ref TEXT` column storing canonical string — requires SCHEMA_V5 migration, makes SQL WHERE clauses require string parsing, breaks existing query tools
- Both columns (add `issue_ref TEXT` alongside existing) — redundant, risks divergence
**Rationale:** The worklog is a persistence layer with its own stable schema (currently SCHEMA_V4). Its tables (`work_item_issues`, `issue_enrichment`, `github_issue_cache`) use `(issue_number, issue_repo)` as composite primary keys — this maps naturally to relational queries (`WHERE issue_number = ? AND issue_repo = ?`). Decomposing IssueRef at the boundary (`ref.number, ref.repo`) is the standard approach for domain types that don't map 1:1 to relational schema. The bridge function `_emit_issue_events()` in plan_manager.py already takes `(int, str)` — it will take `IssueRef` and decompose internally.
**Trade-offs:** Every worklog call site decomposes `ref.number, ref.repo`. This is a one-liner, not a burden.
**Sources:** scripts/worklog.py (SCHEMA_V1–V4), plan_manager.py `_emit_issue_events()`
**Exploration:** quick (surfaced by R1-18)
**Status:** captured

## D7: check_plan_state API call grouping

**Choice:** Group open issues by repo, make one `gh issue list` call per distinct repo.
**Alternatives:**
- Individual `gh issue view` per issue (O(n)) — too many API calls for large queues
- Only check issues matching the default repo — loses the benefit of per-issue repos
**Rationale:** Current code makes one `gh issue list --repo resolve_repo` call for all issues. After IssueRef, each leaf carries its own `ref.repo`. Grouping by repo and making one `gh issue list` per distinct repo is O(distinct_repos) — in practice 1-2 calls. The latency increase (2-3s vs 1-2s) is acceptable for a health check that runs at session entry.
**Trade-offs:** Slightly more complex implementation. Latency proportional to repo count. But the alternative (single-repo resolution) was the root cause of the #268 false positive bug.
**Sources:** project/work_health.py `check_plan_state()`
**Exploration:** quick (surfaced by R1-19)
**Status:** captured

## D8: Match-by-IssueRef semantics (the core fix)

**Choice:** All issue-matching functions (`_mark_completed`, `_mark_active`, `_collect_issue_numbers`, `reorder_queue`, `remove_from_queue`) key by `IssueRef` instead of bare `int`.
**Alternatives:**
- Keep bare int matching with repo as a secondary check — half-measure that doesn't eliminate ambiguity
**Rationale:** This is the load-bearing change that fixes #268. `_mark_completed(items, issue_number: int)` currently matches by bare int — if two items have the same number in different repos, the first match wins silently. After IssueRef, matching uses `item.ref == ref` which compares both repo and number. The return type of `_collect_issue_numbers()` changes from `set[int]` to `set[IssueRef]`, making deduplication cross-repo correct. `reorder_queue` and `remove_from_queue` take `list[IssueRef]` instead of `list[int]`.
**Trade-offs:** CLI callers that pass bare issue numbers must now pass repo-qualified refs. This is the forced explicitness that prevents the bug.
**Sources:** plan_manager.py `_mark_completed`, `_mark_active`, `_collect_issue_numbers`, `reorder_queue`, `remove_from_queue`
**Exploration:** quick (surfaced by R1-20)
**Status:** captured

## D9: detect() CLI output format

**Choice:** `detect()` returns `active_issue` as the canonical string `owner/repo#N` (via `str(ref)`) and adds `active_issue_repo` for callers that need the repo separately.
**Alternatives:**
- Keep `active_issue` as bare int, add `active_issue_repo` — preserves backward compat but perpetuates the pattern of scattered (int, str) pairs
- Return IssueRef object — not possible since detect() outputs KEY=VALUE to stdout for shell consumption
**Rationale:** `detect()` is consumed by shell scripts and skill SKILL.md text via `plan_manager.py detect`. The output `ACTIVE_ISSUE=Hortora/soredium#268` carries the full identity. Callers needing the number can parse with `IssueRef.parse()` in Python or simple shell splitting on `#`. Adding `ACTIVE_ISSUE_REPO` separately eases the transition.
**Trade-offs:** Changes the shell contract from `ACTIVE_ISSUE=268` to `ACTIVE_ISSUE=hortora/soredium#268`. Skill SKILL.md text referencing `ACTIVE_ISSUE` for display may need updating.
**Sources:** plan_manager.py `detect()`, work/SKILL.md
**Exploration:** quick (surfaced by R1-22)
**Status:** captured

---
layout: post
title: "The Gaps Nobody Noticed"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [soredium]
tags: [worklog, mcp, lifecycle, observability]
---

I came into this session expecting to build nested issue lifecycle tracking inside slots — a large, high-complexity piece of work (#141). What I found instead: the infrastructure was already there. `plan_manager.advance()` already did auto-end-previous with worklog emission. `complete_active_issue()` already existed for work-end. The lifecycle state machine already had the transitions wired.

Three sessions of work across #158, #178, and the `.plan` unification had quietly landed everything #141 described. The issue was written before any of it existed; by the time we got to it, the description matched code that was already shipping.

What remained were gaps — the kind that only surface when you trace the full path from first issue to branch closure.

**The parent epic that stays open forever.** This was the interesting one. `advance()` marks the active leaf issue complete and adds it to `covers:`. When that leaf is the last child of an epic, `_mark_parent_epics_if_done()` marks the parent `[x]` in the `.plan` tree — but never adds the parent's issue number to `covers:`. On the next `advance()` call, the reconciliation catches it. But when the queue is exhausted, there is no next call. Work-end runs, reads `covers:`, closes every issue listed. The parent epic isn't there. It stays open on GitHub permanently, invisible because all its children are closed.

Claude caught this during code review — a finding that wouldn't have surfaced from tests alone because the tests verify `advance()` works correctly (it does), not that the full lifecycle from last-advance to work-end handles parent epics.

The fix was straightforward: `reconcile_covers()` — a function that scans all `[x]` items in `.plan` against `covers:` in `.meta` and appends any missing — runs at the start of every `advance()` call (crash recovery) and now also at work-end before `close_artifacts.py` runs.

**The worklog MCP server** was the other piece. The query functions have existed in `worklog.py` since the schema was written — `active_work()`, `event_log()`, `work_item_timeline()`, `slot_status()`. No consumer could reach them without shelling out to Python or raw `sqlite3`.

The design conversation was the useful part. I'd assumed a Quarkus service — trellis is Quarkus, the engine is Quarkus, anything always-running should be Quarkus. But the MCP server for Claude Code is a per-session stdio process — spawned when the session starts, dies when it ends. A JVM for four read-only SQLite queries that live for an hour is 200MB of overhead for nothing. The Python MCP server imports `worklog.py` directly — same language, same path normalization, same schema. No impedance mismatch.

The always-running Quarkus service still makes sense — but that's trellis's concern, for REST endpoints and MCP over SSE. The two aren't mutually exclusive: Python MCP is the baseline that works without trellis; Quarkus adds the always-on layer when trellis is present.

The robustness review surfaced four real issues in the spec before any code was written: `WORKLOG_DB` env var not read by `connect()` (test plan was broken as written), raw exceptions propagating as MCP errors, `repo_path` normalization inconsistency between `event_log` and `work_item_timeline`, and metadata returned as a JSON string rather than a parsed dict. All four were fixed in the spec, then implemented correctly the first time.

One thing that came out of this session sideways: Claude consistently recommends skipping the pre-close sweep during work-end. "This was a small change." "Session is getting long." The work-end skill already says defaults are ON and the user decides — but the language wasn't strong enough to prevent Claude from talking the user out of running them. The fix went into the skill itself: a `NEVER-RECOMMEND-SKIPPING` block with an explicit list of rationalizations that are all wrong, plus three new Red Flag entries. Every Claude instance loading this skill now gets the anti-skip language.

The open question is whether epic #182 is done. All four "Done when" criteria are met except "worklog data queryable via REST and MCP tools" — but that was descoped to MCP-only for soredium, with REST deferred to trellis. The MCP server is shipping. Whether to close the epic or leave it open with a pointer to the trellis REST work is a judgment call about how much the epic boundary matters versus the repo boundary.

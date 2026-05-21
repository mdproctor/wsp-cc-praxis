# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-13 — Blog entry types: article vs. note, with routing metadata

**Priority:** medium
**Status:** active

Differentiate write-blog output into two types: `note` (current diary/journal
entries — session narrative, informal) and `article` (traditional topic or
summary post — polished, standalone readable). Each entry carries metadata
declaring target destinations: project blog, author's personal blog, one or
more group/syndication blogs. Cross-posting to multiple targets is supported.
Routing strategy at epic close: notes go to the project repo; articles route
per their metadata targets.

**Context:** Surfaced during brainstorm on DESIGN.md interaction model.
The current write-blog skill only produces diary-style notes. Articles serve
a different audience and need different routing — author blogs, group blogs,
and syndication targets beyond the project repo.

**Promoted to:** `docs/superpowers/specs/2026-04-14-blog-entry-types-design.md` (implemented 2026-04-14)

---

## 2026-05-18 — Markdown A/B viewer → LLM writing critique tool

**Priority:** medium
**Status:** active

The markdown A/B viewer (currently being built as a Sparge spin-off) has a natural evolution path:

1. **Phase 1 (current):** Side-by-side rendered markdown — load two files, sync scroll, compare
2. **Phase 2:** Add LLM critique panel — show the critique that motivated the changes alongside the before/after
3. **Phase 3:** Interactive — select a passage, get critique, see the improved version generated inline

Phase 2 maps almost exactly to Sparge's refine panel — it shows suggested changes with context. The difference is domain: writing quality rather than HTML-to-markdown conversion. The critique panel would show readable prose (what was wrong, what changed, why) rather than diff hunks.

This tool would be directly useful for the A/B testing of the linguistic fingerprint — load the "no guidance" version (A) and the "fingerprint applied" version (B), with the LLM's critique of A showing why B improves on it.

**Connection:** Sparge already has the UX pattern. The new tool inherits the panel layout, scroll sync, and marked.js renderer. The LLM critique layer is new.

---

## 2026-05-13 — Command to toggle exhaustive note-taking mode

**Priority:** medium
**Status:** active

A cc-praxis skill or Claude Code command that toggles between two note-taking modes:

- **Selective (default):** Claude synthesises and summarises, capturing conclusions and key points. Fast, but loses nuance and detail from conversation.
- **Exhaustive:** Claude transcribes before synthesising — captures full reasoning chains, all findings, verbatim quotes worth preserving, and recent session content that hasn't been written up yet. Nothing is assumed to be recoverable later.

**Trigger:** User says "exhaustive notes on" / "exhaustive notes off" or invokes a command like `/notes-exhaustive`. Could also be triggered automatically when the user says "take notes" or "capture this."

**Why this matters:** The default synthesis mode consistently loses detail when note-taking spans a long session. The gap is hardest to recover at the end of a session when context is full but the most recent material hasn't been captured yet. An explicit toggle lets the user signal "I need everything, not a summary."

**Possible implementation:** A skill that sets a session flag and prepends an instruction to all note-writing actions. Or a CLAUDE.md convention that Claude checks before writing to any notes file.

---

## 2026-05-13 — Structured essay as default voice

**Priority:** medium
**Status:** active

Hypothesis: structured essay (numbered sections, hybrid thematic+structural headings,
dense prose carrying the argument) is the preferred default form for all communication
that naturally breaks into sections — not just long multi-part series. The 6-part
"When the Machine Codes" series is the canonical example.

If true, the taxonomy becomes: structured essay is the baseline; article-prose,
article-structured, brief, note-prose, note-structured are specialisations for
specific contexts. The README and style files should reflect this once validated
against real content.

**Deferred:** validate against new content before changing anything. Do not overwrite
existing style files until the preference is confirmed in practice.

---

## 2026-05-13 — Revisit existing content in middle and brief forms

**Priority:** low
**Status:** active

Most existing articles and notes were written in full narrative form. A middle
form exists between the current article (prose-led, narrative) and the brief
(pure data density) — structured prose where headers and bullets scaffold the
content but reasoning still lives in sentences. Hypothesis: much of the existing
content would be more readable in this middle form, and some would benefit from
being rewritten as a brief.

**Deferred:** revisit when there's time for a content audit. When ready: pick a
representative sample of existing posts, rewrite in middle and brief forms, compare.
Use the result to decide whether the middle form needs its own style file or is
just the updated article guide applied more aggressively.

---
*(merged from docs/ideas/IDEAS.md)*


Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-09 — Marketplace aggregation: cc-praxis pulls in external marketplaces

**Priority:** medium
**Status:** active

The cc-praxis install-skills wizard could support referencing one or more external marketplaces (e.g. `hortora/soredium`) in addition to its own. Users installing from cc-praxis would see skills from all aggregated marketplaces in a single menu, without needing to know the external repo exists. Each external marketplace remains independently installable for users who find it directly.

**Context:** Arose when planning the migration of the garden skill to `hortora/soredium`. Hortora wants its own standalone marketplace; cc-praxis users shouldn't lose access. Marketplace aggregation would solve both without duplication — hortora maintains its own skills, cc-praxis just points at it. Would require changes to `install-skills`, `uninstall-skills`, and `marketplace.json` format.

**Promoted to:**

---

## 2026-04-06 — Periodically evolve the personal writing style guide

**Priority:** low
**Status:** active

The personal guide (`blog-technical.md`) was built from a 577-post corpus spanning 2006–2017 — solo technical writing for the Drools/KIE community. The current writing context is different: collaborative work with Claude, a development diary format, and potentially a broader audience. Some rules carry forward perfectly; others may need revisiting for the new context. Worth doing a deliberate review periodically — comparing what the guide says against what's actually being written — and asking which rules still fit and which should evolve.

**Context:** Arose from questioning the "no preamble" rule — which is solidly grounded in the corpus but may have legitimate exceptions in longer series or for broader audiences. The I/we/Claude register system was invented for this context, not derived from the corpus; other adaptations might similarly be worth making deliberately rather than discovered after the fact.

**Promoted to:**

---

## 2026-04-04 — Holistic project-memory architecture: indexed folders not flat files

**Priority:** high
**Status:** active

The cc-praxis methodology family (design-snapshot, project-blog, knowledge-garden,
idea-log, adr, handover, CLAUDE.md) has grown organically but lacks a
coherent cross-tool indexing strategy. Two specific improvements worth exploring:

1. **design-snapshot as a categorised folder** — instead of flat dated files,
   use subdirectories by topic (architecture/, decisions/, state/) with an
   INDEX.md modelled on GARDEN.md's dual-index approach (by category AND by
   date). This lets a future session find "what did we decide about the web
   installer?" without reading all snapshots.

2. **Cross-tool findability** — each tool currently indexes only its own
   content. A light meta-index (or agreed naming conventions) would let
   handover point to the right snapshot, blog entry, or garden section
   without loading any of them. The lazy-reference principle from
   handover should propagate to how all tools reference each other.

**Context:** Surfaced while designing handover — the skill needs to
reference design-snapshot, project-blog, and knowledge-garden without loading
them. The knowledge-garden's GARDEN.md dual-index (by technology + by symptom
type) proved that this pattern works. design-snapshot has no equivalent index,
making it harder to reference selectively. Discussed alongside the observation
that DESIGN.md is current-state only, not historical, so snapshots carry the
historical burden but aren't structured for retrieval.

**Promoted to:** *(leave blank)*

---

## 2026-04-04 — Dual-repo model for epic-scoped developer work

**Priority:** medium
**Status:** active

A developer working on a long-running epic uses two repos: the main project
repo (code + finalized artifacts) and a personal session repo (WIP snapshots,
idea-log entries, in-flight ADRs, DESIGN-DELTA.md). At epic close, curated
artifacts are published to the main project; noise stays in the session repo.
The DESIGN-DELTA.md evolves throughout the epic as a draft of changes planned
for the main DESIGN.md.

**Context:** Arose during a review of the 7 methodology skills (idea-log, adr,
design-snapshot, update-claude-md, java-update-design, update-primary-doc,
issue-workflow) — noticed that all of them write directly into the project repo,
creating noise during exploratory/WIP phases. Also prompted by thinking about
co-worker collaboration: multiple developers on the same epic could each have
their own session repo and reconcile at integration time. Revisit alongside
issue/epic grouping work and co-worker collaboration model.

**Promoted to:** *(leave blank — fill if promoted to ADR or task)*

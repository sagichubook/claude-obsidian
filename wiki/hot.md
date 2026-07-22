---
type: meta
title: "Hot Cache"
updated: 2026-07-22
tags:
  - meta
  - hot-cache
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
  - "[[getting-started]]"
---

# Recent Context

Navigation: [[index]] | [[log]] | [[overview]]

## Last Updated

**2026-07-22**: Consolidated the entire Dux corpus wiki from 74 reference notes down to 15 dense, publication-ready guides (plus a new landing page, [[Dux]]), replacing the PARA reference structure in place per explicit instruction. Ingestion itself was already complete and verified (68/68 source files, confirmed byte-identical to `C:\Users\User\dux\docs` via a fresh diff at session start). This session was the rewrite phase, turning dense internal-reference notes into Medium-style guides for explanation/how-to content, while keeping true reference material (API contracts, the decisions/traceability log, taxonomy) dense and tabular rather than prose-ifying it. See [[Dux]] for the full domain map.

## Key Recent Facts

- New structure: [[Dux]] (landing) + 2 product files, 1 architecture, 2 AI-safety, 1 engineering, 2 operations, 1 governance, 1 GTM, 1 API, 1 decisions/traceability reference, [[Dux Portfolio]] (tone-polished in place) = 15 files total, replacing 74.
- Every domain page's key figures (dollar thresholds, decision IDs, percentages) were grep-verified against the immutable `.raw/dux/` mirror as each file was written, not just carried forward from the old notes.
- `wiki/migration-audit.md` and `wiki/validation-checklist.md` are preserved as a historical record of the original ingest's verification rigor (4 passes, 19 gaps found and fixed): they predate this consolidation and reference note names that no longer exist; treat them as frozen history, not live navigation.
- Old precision (exact per-file source citations, atomic one-topic-per-note structure) is recoverable via git history but is no longer live in the wiki: this was a deliberate, explicitly-approved trade of some parallel precision for a single authoritative, minimum-file-count corpus.

## Recent Changes

- Deleted 74 old `wiki/areas/dux-*`, `wiki/resources/dux-*`, `wiki/resources/people/*`, `wiki/resources/concepts/{CaMeL,Governance Kernel,Kill Switch,World Model}.md` notes; replaced with the 15 files above.
- `wiki/index.md` rewritten to point at the new structure.
- Work was done under a session-wide `wiki-lock` to defer the auto-commit hook, avoiding both commit spam and sweeping unrelated pending changes (`.obsidian/`, `README.md`, `assets/`, `hooks/`, `skills/`, `wiki/concepts/DragonScale Memory.md`, all pre-existing staged work from a separate, unrelated task) into this session's commit.

## Active Threads

- Known style deviation carried forward: the new guides still preserve exact dollar figures, decision IDs, and thresholds from the source corpus (by design, the brief required zero loss of technical accuracy even while consolidating), so they read as more numerically dense than a typical Medium post. This is intentional, not an incomplete polish pass.
- Flagged data gaps (not fabricated), still true post-consolidation: no activation A/B-test log, no support-ticket-category volumes, no RICE-scored roadmap: each explicitly marked as a gap in [[Dux Customer Success Guide]] and [[Dux Product Guide]] rather than invented.
- Unrelated, pre-existing staged changes (`.obsidian/*`, `README.md`, `assets/diagrams/*`, `hooks/*`, `skills/*.md`, `wiki/concepts/DragonScale Memory.md`) were deliberately left untouched and unstaged by this session: not part of this work, not evaluated for correctness.

## Style Preferences

- No em dashes (U+2014) or `--` as punctuation. Periods, commas, colons, or parentheses. Hyphens in compound words are fine.
- Short and direct responses. No trailing summaries.
- Parallel tool calls when independent.

## Repo Locations

- Working: `~/Desktop/claude-obsidian/`

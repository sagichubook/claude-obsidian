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

**2026-07-22**: Ran a second, different verification lens (backtick/inline-code-span audit, the original ingest's own 4th-pass method) on the 6 densest Dux guides, found and fixed 28 more real gaps, and closed all 3 items the previous verification pass had left open. **Running total across both passes: 104 real gaps found and fixed**, on top of the original ingest's 19. Full account: the two `verify` entries in [[log]].

## Key Recent Facts

- All 3 previously-open items are now closed, not just flagged: SEC-AUTH-03 (was fully missing from Architecture Guide, added); the kill-switch SLO fix (previously only 1 of 3 occurrences was fixed, now all 3); and both "disclosed contradictions" (both turned out resolvable against the decisions log: Control Refinements is Gate 2 per D-19/D-23, and `ticket.create_remediation` has no confidence floor per D-17's file-list plus GOV-TOOL-05's own self-contradictory table).
- The second pass's most notable find: [[Dux AI Safety Guide]] claimed CaMeL's dual-LLM split makes prompt injection structurally unreachable with no caveat, when the source names one specific accepted residual risk ("Branch Steering") it doesn't fully close. Now disclosed.
- Also found this pass: a whole second HITL axis (investigation-confidence gate, D-34) missing from AI Safety Guide; AIBOM governance, the EU-contract impact-assessment gate, and vulnerability-disclosure policy missing from Governance Guide; S-LLM/P-LLM model bindings and the notification engine's 4-channel fan-out missing from Architecture Guide; acknowledgment endpoints and 2 header names missing from API Reference; several controlled-vocabulary/design-token gaps in Taxonomy & Catalogs.
- Only 6 of the 15 guides got this second lens (the densest, highest-risk ones), matching the original ingest's own documented diminishing-returns finding. The other 9 have only had the first (ID/fact) lens applied.
- `wiki/migration-audit.md` and `wiki/validation-checklist.md` remain frozen historical records; neither pass touched them.

## Recent Changes

- This pass edited, in place: [[Dux Taxonomy & Catalogs]], [[Dux Feature Reference]], [[Dux Governance & Compliance Guide]], [[Dux Architecture Guide]], [[Dux API Reference]], [[Dux AI Safety Guide]], [[Dux Product Guide]] (contradiction resolution). No files added or deleted.
- All 6 backtick-span audits ran read-only, in isolated worktrees; every fix was applied by the main session afterward to avoid concurrent edits to shared guide files.
- None of this is committed yet, same as everything it's built on top of.

## Active Threads

- Not yet committed: two verification passes plus the original consolidation all sit as uncommitted working-tree changes.
- Flagged data gaps (not fabricated), still true: no activation A/B-test log, no support-ticket-category volumes, no RICE-scored roadmap, each explicitly marked rather than invented.
- The other 9 guides ([[Dux Product Guide]], [[Dux AI Safety Operations Reference]], [[Dux Operations Guide]], [[Dux Customer Success Guide]], [[Dux Engineering Guide]], [[Dux GTM Guide]], [[Dux Portfolio]], [[Dux Decisions & Traceability Reference]], [[Dux]]) have only had the first-lens audit, not the backtick-span lens. Worth knowing if asked to go further.
- Unrelated, pre-existing staged changes (`.obsidian/*`, `README.md`, `assets/diagrams/*`, `hooks/*`, `skills/*.md`, `wiki/concepts/DragonScale Memory.md`) remain untouched: not part of this work, not evaluated for correctness.

## Style Preferences

- No em dashes (U+2014) or `--` as punctuation. Periods, commas, colons, or parentheses. Hyphens in compound words are fine.
- Short and direct responses. No trailing summaries.
- Parallel tool calls when independent.

## Repo Locations

- Working: `~/Desktop/claude-obsidian/`

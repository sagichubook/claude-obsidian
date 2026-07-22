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

**2026-07-22**: Independently re-verified all 15 consolidated Dux guides against the 68-file source corpus, using 8 parallel domain-scoped audits with the same rigor as the original ingest's 4-pass verification (the consolidation itself had only been spot-checked as it was written, a weaker bar). Found and fixed **76 real gaps**, plus disclosed 2 source-internal contradictions rather than silently resolving them. A reconciliation pass afterward, walking every individual finding back against the actual edit, caught 2 items the first pass missed (the Gate-2 triage model path, and 2 further occurrences of a kill-switch SLO conflation beyond the first one fixed): worth remembering for any future pass, fix inline as findings arrive, then always reconcile against the full finding list before calling it done. Full account: the `[2026-07-22] verify` entry in [[log]]. A control diff (`.raw/dux/` vs. the source path) confirmed the corpus itself hadn't drifted, so this was a rewrite audit, not a re-ingest.

## Key Recent Facts

- The 15-guide structure from the prior consolidation is unchanged; this pass edited content within those same files, it didn't restructure them.
- Biggest gap categories: reference-table content dropped during the narrative rewrite (catalogs, metrics sections, controlled vocabulary in [[Dux Taxonomy & Catalogs]] and [[Dux Feature Reference]]); missing API surface in [[Dux API Reference]]; missing data-model entities in [[Dux Architecture Guide]]; a factual kill-switch SLO error and a regressed tool catalog in [[Dux AI Safety Guide]]; missing framework crosswalks in [[Dux Governance & Compliance Guide]]; missing competitor detail in [[Dux GTM Guide]]; a stale capacity figure in [[Dux Product Guide]]; missing runbooks and a service catalog in [[Dux Operations Guide]]; and the founders' names, investors, and market-validation quotes missing from [[Dux]].
- Two disclosed, unresolved source contradictions (not fixed by picking a side): `ticket.create_remediation`'s confidence floor (Dux AI Safety Guide) and whether Control Refinements is actually Gate-1-live or Gate-2-deferred (Dux Product Guide).
- `wiki/migration-audit.md` and `wiki/validation-checklist.md` remain frozen historical records of the original ingest; this pass didn't touch them.

## Recent Changes

- Edited in place, no files added or deleted: [[Dux]], [[Dux Product Guide]], [[Dux Feature Reference]], [[Dux Taxonomy & Catalogs]], [[Dux Architecture Guide]], [[Dux API Reference]], [[Dux AI Safety Guide]], [[Dux AI Safety Operations Reference]], [[Dux Governance & Compliance Guide]], [[Dux GTM Guide]], [[Dux Operations Guide]], [[Dux Customer Success Guide]], [[Dux Engineering Guide]], [[Dux Portfolio]].
- All 8 verification agents ran read-only, in isolated worktrees; every fix was applied by the main session afterward to avoid concurrent edits to shared guide files.
- None of this is committed yet, same as the consolidation it's built on top of.

## Active Threads

- Not yet committed: this pass plus the prior consolidation both sit as uncommitted working-tree changes.
- Flagged data gaps (not fabricated), still true after this pass: no activation A/B-test log, no support-ticket-category volumes, no RICE-scored roadmap, each explicitly marked rather than invented.
- One lower-priority secondary finding was noted but not fixed: SEC-AUTH-03's JWT-signing-algorithm and cookie-attribute detail, absent from the Architecture Guide. Left for a future pass, consistent with this vault's documented pattern of diminishing returns on repeated verification lenses.
- Unrelated, pre-existing staged changes (`.obsidian/*`, `README.md`, `assets/diagrams/*`, `hooks/*`, `skills/*.md`, `wiki/concepts/DragonScale Memory.md`) remain untouched: not part of this work, not evaluated for correctness.

## Style Preferences

- No em dashes (U+2014) or `--` as punctuation. Periods, commas, colons, or parentheses. Hyphens in compound words are fine.
- Short and direct responses. No trailing summaries.
- Parallel tool calls when independent.

## Repo Locations

- Working: `~/Desktop/claude-obsidian/`

---
type: meta
title: "Operation Log"
updated: 2026-07-21
tags:
  - meta
  - log
status: evergreen
related:
  - "[[index]]"
  - "[[hot]]"
  - "[[overview]]"
---

# Operation Log

Navigation: [[index]] | [[hot]] | [[overview]]

Append-only. New entries go at the TOP. Never edit past entries.

Entry format: `## [YYYY-MM-DD] operation | Title`

---

<!-- Add new log entries below this line -->

## [2026-07-22] consolidate | Dux corpus rewrite: 74 reference notes to 15 publication-ready guides

Ingestion was already complete and independently re-verified at the start of this session (fresh `diff -r` of `C:\Users\User\dux\docs` against `.raw/dux/` came back byte-identical, confirming the 2026-07-21 mirror had not drifted). The task was the rewrite phase: consolidate the 74-note PARA reference wiki into a minimum-viable set of dense, Medium-style guides, per explicit instruction to replace outright rather than add alongside.

**Structural decision (user-confirmed via clarifying question):** replace the existing reference notes in place, trading some parallel per-fact precision (recoverable via git history) for a single authoritative, minimum-file-count knowledge base. Diátaxis-informed split: explanation/how-to content became narrative guides; true reference material (API contracts, the decisions/traceability log, the controlled vocabulary) stayed dense and tabular rather than being prose-ified, since that would have made it worse at its actual job.

**Created:** [[Dux]] (new landing page, absorbing the old corpus-hub note plus 5 role-based cross-cutting hubs and the two people/company entity notes), 2 product files, 1 architecture guide, 2 AI-safety files, 1 engineering guide, 2 operations files, 1 governance guide, 1 GTM guide, 1 API reference, 1 decisions/traceability reference. 15 files total. [[Dux Portfolio]] kept at its existing path, tone-polished in place.

**Deleted:** all 74 prior `wiki/areas/dux-*`, `wiki/resources/dux-*`, `wiki/resources/people/*`, and the 4 Dux-specific `wiki/resources/concepts/*.md` notes, fully absorbed into the 15 files above.

**Verification method:** every domain page's key figures (dollar thresholds, decision IDs, percentages, technical specs) were grep-verified against the immutable `.raw/dux/` mirror as each file was written, not merely carried forward from the prior notes. `wiki/migration-audit.md` and `wiki/validation-checklist.md` were preserved as a historical record of the original ingest's 4-pass verification rigor, banner-marked as frozen, and had their ~185 combined wikilinks stripped to plain text so they stop reporting as dead links against note names that no longer exist.

**Quality fix caught during this session:** the newly-written prose initially used 456 em dashes across the 15 files, violating this vault's own stored style preference (visible in `hot.md`, which should have been followed from the start). Fixed via a context-aware script (table-cell placeholders to a plain hyphen, paired asides to real parentheses, single connectors to a colon or comma depending on paren depth), followed by a manual pass fixing the handful of resulting awkward constructions. Zero em dashes remain in any file touched this session.

**Fixed:** `wiki/index.md` rewritten to the new 15-file structure; `wiki/hot.md` overwritten per the session-summary convention.

**Lint result:** not yet run as of this entry; see the next log entry or `hot.md` for the outcome.

## [2026-07-21] full-ingest | Dux corpus — remaining 9 domains, hubs, canonical pages, QA

Continued a same-day session that had already ingested the `10-product/` domain (14 files -> 22 notes) and core registries. This pass ingested the remaining 46 source files across `00-meta`, `20-architecture`, `30-api`, `40-ai-safety`, `50-engineering`, `60-operations`, `70-governance`, `80-gtm`, and `90-execution` — 68/68 source files now covered (verified by reconciliation, see [[migration-audit]]).

**Created:** ~41 domain notes (professional documentation: executive summary, specification, Mermaid diagram, cross-links), 1 project note ([[Dux Portfolio]] consolidating the 10-epic execution backlog), 7 domain hub notes (`Dux * Area`), 5 cross-cutting role-based hubs ([[Product Hub]], [[Engineering Hub]], [[Growth Hub]], [[Customer Success Hub]], [[Legal-Finance Hub]]), 2 new synthesis notes flagging real content gaps rather than fabricating data ([[Dux Onboarding & Activation]], [[Dux Support Playbook]]), 1 roadmap note ([[Dux Roadmap]] — gate-based, RICE data flagged absent from source), `Welcome.md`, and three templates (`_templates/prd.md`, `_templates/adr.md`, `_templates/runbook.md`).

**Fixed:** a stale claim in [[Dux Overview]] that prematurely said "68 source files ingested" when only the product domain was done at the time — now true. Several dead-link naming mismatches (`CI/CD & Testing` vs. filename `CI-CD & Testing`, a stray `[[Support Playbook]]` vs. `[[Dux Support Playbook]]`, `wiki/`-prefixed link forms).

**Lint result:** 0 orphaned notes, 0 dead links remaining from this ingest (a handful of pre-existing dead links from unrelated, already-deleted vault content — `Compounding Knowledge`, `Wiki Map`, `concepts/_index`, `dashboard` — predate this session and are out of scope).

**Adaptation disclosed:** STEP5's generic canonical-page paths (`areas/product/roadmap.md` etc.) were delivered at this vault's established company-prefixed convention instead (`areas/dux-product/Dux Roadmap.md`), to avoid fragmenting the same topic across two folder conventions. Full mapping in [[migration-audit]] §4.

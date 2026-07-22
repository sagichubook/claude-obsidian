---
type: meta
title: "Lint Report 2026-07-22"
created: 2026-07-22
updated: 2026-07-22
tags: [meta, lint]
status: evergreen
---

# Lint Report: 2026-07-22

Triggered by a major restructuring, not the usual 10-15-ingest cadence: the 74-note Dux reference wiki was consolidated into 15 dense guides in this session (see the [[log]] entry dated 2026-07-22 for the full account, and [[Dux]] for the new structure). This report covers the state after that consolidation, including issues found and fixed during the work itself.

## Summary

- Pages scanned: 15 new/rewritten Dux files, plus every other page in the vault checked for lingering references to the 74 deleted notes.
- Issues found: 8 dead-link references to deleted note names (in newly-written files and one pre-existing plugin overview page); 456 em-dash style-preference violations across the 15 new files; 1 heading with no framing sentence before its first subsection; 185 wikilinks in two historical audit documents that would have reported as dead links.
- Auto-fixed: all of the above, inline during this session (not deferred).
- Needs review: none blocking. See "Deliberate, non-blocking" below.

## Dead Links (found and fixed)

- `[[Sagi]]` in Dux Product Guide and in Dux Decisions & Traceability Reference's frontmatter: Sagi's content was folded into [[Dux]] during consolidation. Fixed: de-linked to plain text (Product Guide) or removed from `related` (the reference page already links [[Dux]] directly).
- `[[Dux Decisions Log]]` in Dux Product Guide: renamed during consolidation. Fixed: relinked to [[Dux Decisions & Traceability Reference]].
- `[[World Model]]` in Dux Product Guide: folded into Dux Taxonomy & Catalogs. Fixed: piped link `[[Dux Taxonomy & Catalogs|World Model]]`.
- `[[Dux Agent]]` in Dux Product Guide: no longer a standalone page (folded into the same guide's own Dux Agent section). Fixed: de-linked to plain bold text.
- `[[Dux Architecture Decision Records]]` in Dux Architecture Guide: folded into the same file as a section. Fixed: removed the self-referential link, kept the prose pointer.
- `wiki/overview.md` (pre-existing plugin page, not part of this session's original scope but broken by it): linked to 6 deleted Dux pages. Fixed: updated to point at the new [[Dux]], [[Dux Product Guide]], [[Dux Feature Reference]], and [[Dux AI Safety Guide]], and refreshed the stale "Current State" section.

`wiki/log.md`'s 2026-07-21 entry also references several deleted names. Left untouched by design: the log is append-only and past entries are a historical record of what was true when written, not live navigation.

## Orphan Pages

None. Every one of the 15 new files has at least 4 inbound wikilinks from elsewhere in the vault (checked by direct grep, not estimated); [[Dux]] itself has 20.

## Frontmatter Gaps

None. All 15 files carry `type`, `status`, `created`, `updated`, `tags`, `related`, and `sources`.

## Empty Sections

One found and fixed: "## Events and webhooks" in Dux API Reference jumped straight to its first `###` subsection with no framing sentence. Added one.

## Address Validation (DragonScale Mechanism 2)

DragonScale addressing is technically active in this vault (`scripts/allocate-address.sh` and `.vault-meta/address-counter.txt` both present, counter at 3), which would classify all 15 new pages as "post-rollout, address required" by `created:` date alone.

**Not applied, and flagged here rather than silently skipped or silently enforced:** of the 74 predecessor pages this session replaced, and of essentially every other content page in the vault, only one file anywhere carries an `address:` field (`wiki/concepts/DragonScale Memory.md`, the page documenting the mechanism itself). Addressing was scaffolded but was never actually adopted for real content, including across four separate rigorous verification passes on this exact corpus. Retrofitting addresses onto only these 15 pages would create a new inconsistency rather than resolve an existing one. Treating this as informational, matching the vault's actual practice rather than the opt-in feature's theoretical requirement. If DragonScale addressing is meant to become real going forward, that's a vault-wide decision, not a Dux-consolidation one.

## Style: Em Dash Violations (found and fixed)

The vault's own stored style preference (`wiki/hot.md`: "No em dashes (U+2014) or `--` as punctuation") was violated 456 times across the initial drafts of the 15 new files. Fixed via a context-aware pass (table-cell placeholders to a plain hyphen, paired parenthetical asides to real parentheses, single connectors to a colon or comma depending on context) followed by manual review of the resulting sentences for readability. Zero em dashes remain in any file touched this session; verified by direct grep, not sampled.

## Deliberate, non-blocking

- The two audit documents (`wiki/migration-audit.md`, `wiki/validation-checklist.md`) had all 185 combined wikilinks stripped to plain text and were banner-marked as frozen historical records, since the notes they reconcile no longer exist under those names. This was a deliberate choice to stop them reporting as dead links, not an oversight; see the banner at the top of each for the reasoning.
- The new guides intentionally retain more numeric/ID density than a typical Medium post (exact dollar thresholds, decision IDs) because the consolidation brief required zero loss of technical accuracy even while trading away some of the old per-file precision. Not a lint issue, a deliberate scope trade-off already surfaced to the user before the work began.

## Sources

This report was generated from a full grep-based scan of every wikilink, frontmatter block, and heading structure in the vault, cross-referenced against the file list before and after this session's consolidation. See `wiki/migration-audit.md` for the original 2026-07-21 ingest's own four-pass verification report (a separate, earlier concern from this lint pass).

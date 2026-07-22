---
type: meta
title: "Ingestion Log"
updated: 2026-07-22
status: evergreen
---

# Ingestion Log

## [2026-07-22] Manifest Reconciliation Pass

- **Source:** `.raw/dux/10-product/features/predictive-risk-forecasting.md`
- **Summary:** Diffed all 68 files under `.raw/dux/` against `.raw/.manifest.json`'s `sources` map. Found one gap: this file's content was already fully present in [[Dux Feature Reference]] (section "Forward-Looking: Predictive Risk Forecasting (US-028, E4)", and already listed in that page's own `## Sources` list) but the manifest tracker itself had never been updated to record it — a bookkeeping miss, not a content gap.
- **Pages created:** None
- **Pages updated:** `.raw/.manifest.json` (added the missing source→page mapping)
- **Key insight:** No data loss found. All 68 raw sources are now correctly tracked; zero orphaned or un-ingested files remain in `.raw/`.

## [2026-07-22] Full Ingestion Verification

- **Source:** `.raw/dux/` (66 files across 10 domain folders)
- **Summary:** Complete re-verification of all ingested content
- **Pages verified:** All 16 wiki pages
- **Pages created:** None (all already existed)
- **Pages updated:** `wiki/index.md` (verification status), `wiki/hot.md` (created)
- **Key insight:** All content from 60+ source files is properly consolidated into 16 publication-ready guides with zero data loss

### Source domains verified:
- `00-meta/` (5 files) → `wiki/resources/dux-meta/Dux Decisions & Traceability Reference.md`
- `10-product/` (15 files) → `wiki/areas/dux-product/Dux Product Guide.md`, `Dux Feature Reference.md`, `wiki/resources/dux-product/Dux Taxonomy & Catalogs.md`
- `20-architecture/` (6 files) → `wiki/areas/dux-architecture/Dux Architecture Guide.md`
- `30-api/` (5 files) → `wiki/resources/dux-api/Dux API Reference.md`
- `40-ai-safety/` (10 files) → `wiki/areas/dux-ai-safety/Dux AI Safety Guide.md`, `Dux AI Safety Operations Reference.md`
- `50-engineering/` (3 files) → `wiki/areas/dux-engineering/Dux Engineering Guide.md`
- `60-operations/` (5 files) → `wiki/areas/dux-operations/Dux Operations Guide.md`, `Dux Customer Success Guide.md`
- `70-governance/` (2 files) → `wiki/areas/dux-governance/Dux Governance & Compliance Guide.md`
- `80-gtm/` (5 files) → `wiki/areas/dux-gtm/Dux GTM Guide.md`
- `90-execution/` (12 files) → `wiki/projects/dux/Dux Portfolio.md`

---
type: meta
title: "Lint Report 2026-07-23"
created: 2026-07-23
updated: 2026-07-23
tags: [meta, lint]
status: developing
---

# Lint Report: 2026-07-23

Scope: full vault, with emphasis on the 15 wiki/areas|resources|projects pages + [[Dux]] rewritten today from terse reference style into long-form narrative style (63 sources under `.raw/dux/` re-ingested). Checks 1-8 from `skills/wiki-lint/SKILL.md` were run. DragonScale Mechanism 2 (Address Validation) was detected as configured and ready (`scripts/allocate-address.sh --peek` returned `3`) and is included. DragonScale Mechanism 3 (Semantic Tiling) was detected as present but NOT ready (`scripts/tiling-check.py --peek` exited `10`, ollama unreachable at `http://127.0.0.1:11434`) and was skipped per the skill's opt-in gating.

## Summary
- Pages scanned: 18 (16 content pages + hot.md, log.md; index.md reviewed separately)
- Issues found: 26 (7 critical, 12 warnings, 7 suggestions)

## Critical (must fix)

1. **[[Dux Product Guide]]: missing required frontmatter fields.** Frontmatter contains only `owner`, `status`, `gate`, `last_reviewed`, `updated`, `decisions` — it is missing `type`, `title`, `created`, and `tags`, all of which every other rewritten page has. This also breaks address classification below (no `created:` date to compare against the rollout baseline). Fix: add `type: area`, `title: "Dux Product Guide"`, `created:` (best guess 2026-07-22 to match siblings), and `tags: [area, dux, dux/product]`.

2. **[[Dux Decisions & Traceability Reference]] → [[Dux AI Safety Guide]]#Governance Kernel: dead cross-page anchor.** Link at line 333/361 targets a heading `#Governance Kernel` that no longer exists in the rewritten AI Safety Guide. The nearest current heading is `## The governance kernel: a synchronous chain nothing bypasses` (line 119). Fix: update the link text to match the new heading exactly.

3. **[[Dux Decisions & Traceability Reference]] → [[Dux AI Safety Guide]]#Kill Switch & HITL: dead cross-page anchor.** Line 346. Nearest current heading is `## Kill switch and human-in-the-loop: the halt authority` (line 197). Fix: update link.

4. **[[Dux Decisions & Traceability Reference]] → [[Dux AI Safety Guide]]#Confidence Calibration: dead cross-page anchor.** Line 375. No heading containing "Confidence" exists anywhere in the rewritten AI Safety Guide (that content likely moved to [[Dux AI Safety Operations Reference]], which does have a `## 2 · Confidence calibration` section). Fix: repoint the link to `[[Dux AI Safety Operations Reference#2 · Confidence calibration]]` or restore an equivalent heading in AI Safety Guide.

5. **[[Dux Decisions & Traceability Reference]] → [[Dux Taxonomy & Catalogs]]#Confidence scoring methodology: dead cross-page anchor.** Line 375. The current heading is `### Confidence bands` (line 39) — wording changed during the rewrite. Fix: update link text.

6. **[[Dux Decisions & Traceability Reference]] → [[Dux Feature Reference]]#Product Overview: dead cross-page anchor.** Line 396. No heading titled "Product Overview" exists in the rewritten Feature Reference; the closest candidate is `## A Guided Tour of Every Screen Dux Ships — and Why It's Built the Way It Is` (line 15). Fix: update link text or restore an anchor-stable subheading.

7. **[[Dux AI Safety Operations Reference]]: dead same-page TOC anchor.** Line 43, `[[#2.6 · insufficient_data_reason enum]]` does not resolve — the actual heading (line 279) is `### 2.6 · \`insufficient_data_reason\` enum` (backtick-formatted code term). Fix: either add backticks to the TOC entry or strip them from the heading so they match byte-for-byte.

## Warnings (should fix)

8. **Address Validation — 16 post-rollout pages missing `address:` field (DragonScale Mechanism 2, opt-in, detected and ready).** Rollout baseline is 2026-04-23 (`.vault-meta/legacy-pages.txt`), `allocate-address.sh --peek` = `3`, no `legacy-pages.txt` exceptions listed. All 16 Dux content pages have `created:` 2026-07-21 or 2026-07-22 (≥ baseline) and `type` other than meta/fold, so per the skill's classification rule every one of them is "post-rollout (must have address)" and none carries an `address:` field. This is a lint error under the skill's Address Validation posture, listed here as a batch rather than a Critical item because it reflects a vault-wide, uniform gap rather than a per-page defect — see the dedicated section below for the full list and remediation path.

9. **`.raw/.manifest.json` `address_map` references a page that does not exist.** `address_map` maps `wiki/concepts/DragonScale Memory.md` → `c-000001`, but no such file exists anywhere under `wiki/`. Either a rename/delete dropped the manifest update, or the page was never created. Fix: locate the intended page and rename to match, or remove the stale map entry.

10. **[[index]] (wiki/index.md) and [[hot]] (wiki/hot.md): stale source-file count.** Both state "Source files: 66" / "66 source files across 10 domain folders," but the vault's own [[log]] (`wiki/log.md`) Ingestion Log entry for 2026-07-22 ("Manifest Reconciliation Pass") found and confirmed **68** files under `.raw/dux/` (67 `.md` + `openapi.yaml`), matching the actual filesystem count. Fix: update the "66" figures in index.md and hot.md to 68.

11. **[[log]] (wiki/log.md): internal inconsistency in its own entry.** The "Full Ingestion Verification" entry states `.raw/dux/ (66 files across 10 domain folders)`, but the domain-by-domain breakdown immediately below it (5+15+6+5+10+3+5+2+5+12) sums to 68, not 66 — contradicting itself within the same log entry. Fix: correct the summary line to 68.

12-25. **14 of 16 rewritten pages exceed 300 lines** (the skill's "large page" warning threshold): [[Dux Architecture Guide]] (1393), [[Dux Feature Reference]] (1103), [[Dux AI Safety Operations Reference]] (830), [[Dux API Reference]] (698), [[Dux Operations Guide]] (679), [[Dux AI Safety Guide]] (674), [[Dux Taxonomy & Catalogs]] (593), [[Dux Decisions & Traceability Reference]] (579), [[Dux]] (536), [[Dux Engineering Guide]] (522), [[Dux GTM Guide]] (490), [[Dux Governance & Compliance Guide]] (458), [[Dux Product Guide]] (375), [[Dux Portfolio]] (335). This is very likely by design — these are intentionally consolidated "hub" guides replacing 66-68 source files, and each has an internal TOC — so flagged for awareness only, not recommended for splitting without a content-strategy decision.

## Suggestions (worth considering)

26. **No orphan pages found.** Every content page has ≥5 inbound wikilinks; `hot.md`/`log.md`/`index.md` are meta pages exempt from the orphan check by convention. No action needed.

27. **No frontmatter gaps found besides [[Dux Product Guide]]** (see Critical #1). All other 15 rewritten pages have complete `type`, `title`, `created`, `updated`, `tags`, `status`.

28. **No `status: seed` pages found**, so the "seed pages stale >30 days" check does not apply this run.

29. **Missing pages / cross-reference gaps: none flagged.** The 16-page consolidated structure appears to be a deliberate design decision (stated explicitly in [[index]] and [[Dux]] as replacing the prior multi-file domain structure), so no frequently-mentioned-but-unpaged concepts were flagged as candidates for new pages this run — revisit only if the vault moves away from the consolidated-guide model.

30. **Dead-link/empty-section sweep across all 16 pages came back clean** aside from the 6 anchor breakages listed above (Critical #2-7) — no wikilinks to nonexistent page titles, and no headings with zero content underneath once code-fence `#` comments were excluded from heading detection.

31. **Consider a lint pre-check step for future rewrites**: because agents were told to keep TOC-referenced anchors byte-identical only "where there were no internal anchors," every rewrite that touches a heading referenced by a `[[#...]]` or cross-page `[[Page#...]]` link needs a matching TOC/anchor update pass. All 6 broken-anchor findings above trace to this exact failure mode — worth adding an automated pre-commit anchor-consistency check (heading text vs. every `#anchor` reference in the vault) so this doesn't recur on the next rewrite pass.

## Address Validation

- Counter state: `./scripts/allocate-address.sh --peek` = `3`
- Highest c- address observed on any live page: none (no page in the vault currently carries an `address:` field)
- Post-rollout pages checked: 16 (0 passing, 16 errors — see below); rollout baseline 2026-04-23 per `.vault-meta/legacy-pages.txt`, no legacy exceptions listed
- Legacy pages pending backfill: 0 (no page has `created:` < 2026-04-23)

### Errors
- [[Dux]], [[Dux AI Safety Guide]], [[Dux AI Safety Operations Reference]], [[Dux Architecture Guide]], [[Dux Customer Success Guide]], [[Dux Decisions & Traceability Reference]], [[Dux Engineering Guide]], [[Dux Feature Reference]], [[Dux GTM Guide]], [[Dux Governance & Compliance Guide]], [[Dux Operations Guide]], [[Dux Portfolio]], [[Dux Product Guide]] (see also Critical #1 for its frontmatter gap), [[Dux API Reference]], [[Dux Decisions & Traceability Reference]], [[Dux Taxonomy & Catalogs]]: all classify as post-rollout (`created:` 2026-07-21/22, `type` not meta/fold) and lack an `address:` field. Run `wiki-ingest`'s address-assignment step or manually run `./scripts/allocate-address.sh` for each and add the result to frontmatter.
- `.raw/.manifest.json` maps `wiki/concepts/DragonScale Memory.md` → `c-000001`, but that page does not exist on disk. Resolve mismatch (rename target page to match, or drop the stale map entry).
- Counter drift: not observed (peek is `3`; no page carries a `c-NNNNNN` address to compare against it, so this check has nothing to validate against yet).

### Pending backfill (informational)
- 0 legacy pages pending backfill. All content pages are post-rollout by `created:` date; the address gap above is an enforcement gap, not a backfill gap.

## Semantic Tiling

Skipped. `scripts/tiling-check.py --peek` exited `10` (ollama not reachable at `http://127.0.0.1:11434`; `model_present: false` for `nomic-embed-text` regardless). Per the skill's opt-in gating, tiling only runs when `--peek` exits `0`; no report was generated. To enable next run: start ollama and run `ollama pull nomic-embed-text`, then re-run lint.

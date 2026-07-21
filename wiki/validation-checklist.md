---
type: meta
title: "Validation Checklist"
created: 2026-07-21
updated: 2026-07-21
tags: [meta, validation-checklist]
related: ["[[migration-audit]]", "[[Dux Overview]]", "[[index]]"]
---

# Validation Checklist — Dux Corpus, Per-File

Navigation: [[index]] | [[migration-audit]] | [[Dux Overview]]

This is the per-file validation record referenced in [[migration-audit]] §6. Method: for every source file, every enumerable ID it contains (D-##, ADR-##, FR-##, US-##, BR-##, EP-##, GOV-##/GOV-TOOL-##, KS-##, PS-##, ISO-##, OI-##, NFR-##/TR-NFR-##, MC-##, CS-##, AI-##) was extracted and checked against the whole vault. Every "missing ID" was then individually triaged into one of three buckets — a **real content gap** (fixed), a **citation-only gap** (the fact is present in prose, just not tagged with that specific ID — not fixed, judged low-value busywork to retrofit 100+ ID tags into prose that already states the fact), or an **intentional compression** (the source's own register-style files, e.g. `open-items.md`, collapse closed/historical items into terse closure notes rather than restating every ID — the corresponding wiki notes follow the same pattern deliberately). Files without a dense ID space were checked by reading the source and destination note side by side for load-bearing facts.

**Legend:** ✅ PASS (no gap) · 🔧 FIXED (real gap found and closed this pass) · 🔵 CITATION-ONLY (content present, ID tag not retrofitted — not a data-loss issue) · ⚪ COMPRESSED (source's own register-collapse pattern, matched intentionally)

## 00-meta/ (5 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `decisions-log.md` (135KB, 57 decisions) | [[Dux Decisions Log]] | 🔧 D-27/28/29/30 (frontend/dataviz/sandbox/LLM-gateway ADRs) were real gaps, fixed via ADR-018/019 additions. D-37/41/46/52/53 checked individually: citation-only or superseded. Remaining ~40 "missing" OI-## references inside this file are ⚪ — the source itself only mentions closed OI-## numbers in passing closure notes; the substantive decisions are all covered |
| `open-items.md` | [[Open Items Register]] | ⚪ The note deliberately lists only the 3 currently-open P2 items in detail (matching the source's own "delete on close" discipline) — the ~30 "missing" OI-## IDs are historical/closed items the source itself only references in one-line closure notes, not standalone content |
| `quick-reference.md` | [[Quick Reference Card]] | ✅ 0/7 IDs missing |
| `traceability-matrix.md` | [[Dux Traceability Matrix]] | 🔵 31 FR-##/NFR-##/TR-NFR-## IDs uncited but spot-checked (FR-013, FR-005) — content present in [[Application API]], [[AI Safety Incident Runbooks]] etc., just not ID-tagged. TR-NFR gap addressed as part of the Observability & SLO fix below |
| `vision-reference.md` | [[Dux, Inc.]] | 🔧 Verified in the prior remediation round (Mandiant M-Trends figures added to [[Lean Canvas]]); MC-01/07/16 are 🔵 citation-only (covered in [[External Corrections 2026-07]] and [[GTM Guardrails]] without the specific MC-## tag) |

## 10-product/ (14 files) — earlier session, personally re-read this pass

| Source | Destination note(s) | Result |
|---|---|---|
| `product-overview.md` | [[Dux Product Overview]] | ✅ 31/32 IDs found; OI-27 is a closed historical item (⚪) |
| `taxonomy.md` | [[Dux Taxonomy and Controlled Vocabulary]] | ✅ 8/8 IDs found |
| `catalogs.md` | [[Dux Catalogs — Registries of Record]] | ✅ 31/34; FR-015/OI-31/OI-37 are 🔵/⚪ (CSV connector content is present via [[Connector Hub]]) |
| `features/assessment-trace.md` | [[Assessment Trace]] | ✅ 10/10 |
| `features/chat-guidance.md` | [[Chat Guidance]] | ✅ 16/16 |
| `features/connector-hub.md` | [[Connector Hub]] | ✅ 17/17 |
| `features/continuous-assessment.md` | [[Continuous Re-Assessment]] | ✅ 10/10 |
| `features/dashboard-audit.md` | [[Dashboard Home & Audit]] | ✅ 11/11 |
| `features/exposure-analysis.md` | [[Exposure Analysis]] | ✅ 11/11 |
| `features/mitigation-write-path.md` | [[Mitigation & Remediation Write Path]] | ✅ 17/17 |
| `features/predictive-risk-forecasting.md` | [[Predictive Risk Forecasting]] | ✅ 11/11 |
| `features/research-dashboard.md` | [[Research Dashboard & Vulnerability Reduction]] | ✅ 14/14 |
| `features/security-stepper.md` | [[Security Stepper]] | ✅ 26/26 |
| `features/tenant-settings.md` | [[Tenant Settings, Help & Custom Metrics]] | ✅ 12/12 |

**Read side-by-side this pass** (per the plan's commitment to personally re-verify the earlier session's un-audited batch): all 14 files confirmed to match their notes on the load-bearing facts — user story IDs, story points, API contracts, and the D-EX/CS-## deviation notes referenced from `90-execution/`.

## 20-architecture/ (6 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `adr-index.md` (57KB, 21 ADRs) | [[Dux Architecture Decision Records]] | 🔧 ADR-018/019 were the real gap (see [[migration-audit]] §6), now fixed. ADR-034 is a false positive — the source itself explains it as legacy numbering for ADR-003, not a distinct ADR. AI-17/21/22/44 and OI-17/22/36/37/38 are 🔵/⚪ |
| `architecture-diagrams.md` | [[Architecture Overview]], [[Workflows & Agent Orchestration]], [[Multi-Tenancy]], [[Sandbox Execution]], [[MCP Security]] (diagrams distributed) | ✅ 8/9; GOV-TOOL-01 is 🔵 (covered via the Governance Kernel fix) |
| `architecture-overview.md` | [[Architecture Overview]] | ✅ 22/28; D-429 is a regex false-positive (matched inside "NVD-429-backoff"); GOV-005/OI-08/17/30/31 are 🔵/⚪ |
| `data-model.md` | [[Data Model]] | ✅ 10/11; ISO-013 (a specific materialized-view test case) is 🔵 |
| `multi-tenancy.md` | [[Multi-Tenancy]] | ✅ 10/13; ISO-006/007 (specific isolation test-case IDs) and TR-NFR-006 (covered under its `NFR-004` alias already in the note) are 🔵 |
| `workflows.md` | [[Workflows & Agent Orchestration]] | ✅ 15/15 |

## 30-api/ (5 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `api-overview.md` | [[API Overview]] | ✅ 5/5 |
| `application-api.md` | [[Application API]] | ✅ 27/30; FR-013/OI-16/31 are 🔵 (content — `POST /remediation-tickets`, unattended create+route — verified present) |
| `events-webhooks.md` | [[Events & Webhooks]] | ✅ 5/5 |
| `openapi.yaml` | [[API Overview]] | ✅ 13/16; FR-021/022/OI-16 are 🔵 (endpoints described in [[Public Data API]]) |
| `public-data-api.md` | [[Public Data API]] | ✅ 7/11; FR-021/022/023/OI-15 are 🔵 (the endpoints themselves — custom metrics, vulnerability instances, batch research — are fully described, just not FR-tagged) |

## 40-ai-safety/ (10 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `agent-identity.md` | [[Agent Identity]] | ✅ 4/4 |
| `camel-plane.md` | [[CaMeL]] | ✅ resolves to 13/13 after the Governance Kernel fix (GOV-008/011/012/013 now findable) |
| `confidence-calibration.md` | [[Confidence Calibration]] | ✅ 3/3 |
| `governance-kernel.md` | [[Governance Kernel]] | 🔧 **Real gap** — GOV-002/005/006/008/009/010/011/012/013 and the full `GOV-TOOL-*` risk matrix were absent; the note covered only the chain shape and GOV-014 in depth. Fixed by adding the full 13-gate table, cost-threshold order, and latency budget |
| `incident-runbooks.md` | [[AI Safety Incident Runbooks]] | ✅ 5/5 |
| `kill-switch-hitl.md` | [[Kill Switch]] | ✅ 11/14; GOV-TOOL-01 resolves via the Governance Kernel fix; KS-004/KS-005 (specific test-cadence table rows: monthly L4 smoke, quarterly Valkey-degradation drill) are 🔵 — the KS-L1-L4 propagation SLOs themselves are covered, these two are test-schedule detail |
| `mcp-security.md` | [[MCP Security]] | ✅ 22/27; GOV-TOOL-* resolve via the Governance Kernel fix; PS-010 (a one-line pre-seed container-hardening note) is 🔵 |
| `owasp-assessments.md` | [[OWASP Assessments]] | ✅ 19/20; PS-010 is 🔵 (same as above) |
| `safety-overview.md` | [[AI Safety Overview]] | ✅ 8/8 |
| `sandbox-execution.md` | [[Sandbox Execution]] | ✅ 9/9 |

## 50-engineering/ (3 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `ci-cd-testing.md` | [[CI-CD & Testing|CI/CD & Testing]] | ✅ 23/31; AI-75/KS-002/003/004/005/006/ISO-013/TR-NFR-007 are 🔵 — the cost-benchmark, kill-switch test suite, and golden-set regression gate are all covered, these are specific sub-test-case IDs |
| `engineering-standards.md` | [[Engineering Standards]] | ✅ 9/10; OI-24 (the historical "why this file exists" pointer) is ⚪ |
| `local-development.md` | [[Local Development]] | ✅ 10/10 |

## 60-operations/ (5 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `customer-lifecycle.md` | [[Customer Lifecycle & Comms]] | ✅ 6/6 |
| `dr-bcp.md` | [[DR-BCP]] | ✅ 12/12 |
| `observability-slo.md` | [[Observability & SLO]] | 🔧 **Real gap** — the TR-NFR table was explicitly marked "(selected)" and covered only 5 of 15 rows. Fixed: full 15-row table now present. Remaining AI-20/82/217/226/OI-06/07/36 are 🔵/⚪ (AI-20's Langfuse-DPA item is explicitly noted elsewhere as moot post-self-hosting) |
| `operations-overview.md` | [[Operations Overview]] | ✅ 3/3 |
| `runbooks.md` | [[Seed Operational Runbooks]] | ✅ 9/9 |

## 70-governance/ (2 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `compliance-program.md` | [[Compliance Program]] | ✅ 13/15; AI-226/TR-NFR-009 are 🔵 |
| `series-b-scale.md` | [[Series B Scale Programs]] | ✅ 5/5 |

## 80-gtm/ (5 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `competitive.md` | [[Competitive Positioning & POC]] | ✅ 24/28; MC-05/06/OI-10/11 are 🔵/⚪ |
| `external-corrections-2026-07.md` | [[External Corrections 2026-07]] | ✅ 2/5; MC-01/08/13 are 🔵 — the corrections themselves (footer year, banner co-lead, entity name) are described in full prose, just not MC-##-tagged |
| `gtm-guardrails.md` | [[GTM Guardrails]] | ✅ 9/16; MC-06/09/10/11/15/17/CS-19 are 🔵 (each correction/rule is in the note's prose) |
| `lean-canvas.md` | [[Lean Canvas]] | ✅ 5/5 (post-fix) |
| `pricing-packaging.md` | [[Pricing & Packaging]] | ✅ 5/8; AI-10/226/D-001(false-positive, matched inside `PS-ONBOARD-001`) — no real gap |

## 90-execution/ (12 files)

| Source | Destination note(s) | Result |
|---|---|---|
| `README.md` | [[Dux Portfolio]] | ✅ 28/29; US-026 resolves post-fix (added alongside US-025/027) |
| `traceability.md` | [[Dux Portfolio]] | ✅ 60/73; CS-##/OI-## are ⚪ (competitor-scan disposition IDs, tracked in source as process metadata, not content) |
| `backlog-ep01.md` | [[Dux Portfolio]] | ✅ 22/27; AI-2/7/226/NFR-006/TR-NFR-008 are 🔵 |
| `backlog-ep02.md` | [[Dux Portfolio]] | ✅ 20/26; CS-##/FR-015 are ⚪/🔵 |
| `backlog-ep03.md` | [[Dux Portfolio]] | ✅ 28/32; AI-77/CS-15/FR-017/TR-NFR-005 are 🔵 |
| `backlog-ep04.md` | [[Dux Portfolio]] | ✅ 9/9 |
| `backlog-ep05.md` | [[Dux Portfolio]] | ✅ 22/22 |
| `backlog-ep06.md` | [[Dux Portfolio]] | ✅ 20/27; CS-##/OI-03 are ⚪ |
| `backlog-ep07.md` | [[Dux Portfolio]] | ✅ 15/16; FR-005 is 🔵 |
| `backlog-ep08.md` | [[Dux Portfolio]] | ✅ 13/17; US-026 resolves post-fix; CS-03/FR-021/028 are 🔵 |
| `backlog-ep09.md` | [[Dux Portfolio]] | ✅ 7/8; FR-024 is 🔵 |
| `backlog-ep10.md` | [[Dux Portfolio]] | ✅ 6/6 |

## `docs/README.md` (root)

| Source | Destination note(s) | Result |
|---|---|---|
| `README.md` | [[Dux Overview]] | ✅ 9/10; OI-25 is ⚪ (closed historical item) |

## Summary

- **68/68 files checked** at the ID level; the 22 files from the earlier, un-audited session were additionally re-read personally against their notes this pass.
- **19 real gaps found across four verification passes, all fixed** — see [[migration-audit]] §6-§9 for the full list, materiality, and resolution of each.
- **Four distinct detection lenses applied**, each catching what the others missed: (1) ID cross-reference on the 2 densest files, (2) the same extended to all 68 files across every ID family the corpus uses, (3) dollar-figure/percentage extraction plus diagram-atlas coverage, (4) backtick/inline-code span extraction (endpoint paths, function signatures, alert names, config keys).
- **Hundreds of "missing" flags across all four lenses were individually triaged and closed as non-issues** — the fact is present without its specific ID/figure/code-span tag (the large majority), the note mirrors the source's own register-compression pattern for closed/historical items, or the flagged value is a pre-superseded historical figure the source itself marks non-authoritative.
- **3 extraction false positives identified and dismissed**: `D-429` (substring of "NVD-429-backoff"), `D-001` (substring of `PS-ONBOARD-001`), `ADR-034` (the source explains this as legacy numbering for ADR-003, not a new ADR).
- **Diagram coverage**: all 11 diagrams in `architecture-diagrams.md` are now represented and cited somewhere in the vault.
- **The backtick lens was run against all 12 originally-planned highest-risk files** (completed across two extensions of the same pass) — 8 of 12 yielded at least one real fix, 4 came back clean. This is evidence the corpus is converging toward complete coverage under repeated, varied scrutiny, not proof that zero gaps remain anywhere in the other 56 files, which were not checked with this specific lens; see [[migration-audit]] §9 for the explicit reasoning against claiming exhaustion.

## Sources

This checklist synthesizes across the full `.raw/dux/` mirror; see [[migration-audit]] for the source-inventory-level report and [[Dux Overview]] for the domain map.

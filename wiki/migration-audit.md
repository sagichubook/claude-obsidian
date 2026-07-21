---
type: meta
title: "Migration Audit"
created: 2026-07-21
updated: 2026-07-21
tags: [meta, migration-audit]
related: ["[[Dux Overview]]", "[[index]]", "[[log]]"]
---

# Migration Audit — Dux Corpus Ingest (2026-07-21)

Navigation: [[index]] | [[log]] | [[Dux Overview]]

Reconciliation of the full source-to-note mapping for the ingest of `C:\Users\User\dux` into this vault, spanning two work sessions on 2026-07-21: an earlier session that ingested the `10-product/` domain plus core registries, and this session, which ingested the remaining nine domains (`00-meta`, `20-architecture`, `30-api`, `40-ai-safety`, `50-engineering`, `60-operations`, `70-governance`, `80-gtm`, `90-execution`) and built cross-cutting hubs and canonical pages.

## 1. Source inventory

`C:\Users\User\dux` is a git repository holding a single curated `docs/` tree (developer documentation for a fictional pre-launch cybersecurity SaaS company, "Dux, Inc.") plus one CI tooling script.

| Set | Count | Detail |
|---|---|---|
| Total files in source repo | 69 | 67 `.md` + 1 `.yaml` (OpenAPI draft) + 1 `.py` (CI doc-validator script) |
| Files under `docs/` (mirrored to `.raw/dux/`) | 68 | 1 README + 5(00-meta) + 14(10-product) + 6(20-architecture) + 5(30-api) + 10(40-ai-safety) + 3(50-engineering) + 5(60-operations) + 2(70-governance) + 5(80-gtm) + 12(90-execution) |
| Files outside `docs/` | 1 | `scripts/validate-playbooks.py` — a CI merge-gate script, not documentation content. Its rules (front-matter contract, dead-link check, hour reconciliation, no-change-history-in-prose) are already synthesized into [[Dux Overview]]'s "Document contract" section and [[CI-CD & Testing|CI/CD & Testing]]'s referential-integrity gate; the script itself is not separately mirrored, since it is tooling, not knowledge-base content |

## 2. Mirror completeness (Set A vs. Set B)

`.raw/dux/` contains exactly 68 files, matching the source `docs/` tree file-for-file (verified by directory listing and per-domain count on both sides). **The mirror is complete — no source file was skipped in mirroring.**

## 3. Note coverage (Set B vs. Set C)

Every one of the 68 mirrored source files is cited in the `sources:` frontmatter field of at least one wiki note, verified by extracting all `sources:` references across `wiki/**/*.md` and diffing against the full `.raw/dux/` file list.

**Result: 68/68 source files covered. Zero gaps.**

| Domain | Files | Notes created | Coverage |
|---|---|---|---|
| `00-meta/` | 5 | [[Dux Decisions Log]], [[Dux Traceability Matrix]], [[Open Items Register]], [[Quick Reference Card]], + `vision-reference.md` via [[Dux, Inc.]] (verified: the note's legal-entity-name and hero-copy content matches the source archive) | 5/5 |
| `10-product/` | 14 | [[Dux Product Overview]], [[Dux Taxonomy and Controlled Vocabulary]], [[Dux Catalogs — Registries of Record]], 11 feature notes | 14/14 |
| `20-architecture/` | 6 | [[Architecture Overview]], [[Data Model]], [[Workflows & Agent Orchestration]], [[Multi-Tenancy]], [[Dux Architecture Decision Records]] (adr-index.md), architecture-diagrams.md distributed across 6 notes | 6/6 |
| `30-api/` | 5 | [[API Overview]], [[Application API]], [[Public Data API]], [[Events & Webhooks]] (openapi.yaml folded into API Overview) | 5/5 |
| `40-ai-safety/` | 10 | [[AI Safety Overview]], [[Agent Identity]], [[Confidence Calibration]], [[Sandbox Execution]], [[MCP Security]], [[OWASP Assessments]], [[AI Safety Incident Runbooks]], [[CaMeL]], [[Kill Switch]], [[Governance Kernel]] | 10/10 |
| `50-engineering/` | 3 | [[Engineering Standards]], [[CI-CD & Testing|CI/CD & Testing]], [[Local Development]] | 3/3 |
| `60-operations/` | 5 | [[Operations Overview]], [[Observability & SLO]], [[Seed Operational Runbooks]], [[DR-BCP]], [[Customer Lifecycle & Comms]] | 5/5 |
| `70-governance/` | 2 | [[Compliance Program]], [[Series B Scale Programs]] | 2/2 |
| `80-gtm/` | 5 | [[Competitive Positioning & POC]], [[Pricing & Packaging]], [[GTM Guardrails]], [[Lean Canvas]], [[External Corrections 2026-07]] | 5/5 |
| `90-execution/` | 12 | [[Dux Portfolio]] (README + traceability.md + all 10 backlog-epNN.md files, individually reviewed and consolidated per the corpus's own precedent of one-note-per-cohesive-registry rather than 10 near-duplicate task-list stubs) | 12/12 |
| `docs/README.md` | 1 | [[Dux Overview]] | 1/1 |

## 4. Adaptations from the literal STEP5 canonical-page paths

The user's brief listed canonical pages at generic PARA paths (`areas/product/roadmap.md`, `areas/engineering/architecture.md`, etc.). This vault's existing convention — established in the earlier session, before this one began — uses **company-prefixed domain folders** (`dux-product`, `dux-architecture`, `dux-engineering`, `dux-gtm`, `dux-operations`) because the entire vault is a single-company knowledge base. Continuing that convention rather than creating parallel generic-named folders avoids fragmenting the same topic across two locations. The mapping actually delivered:

| Requested (generic) | Delivered at | Note |
|---|---|---|
| `areas/product/roadmap.md` | `wiki/areas/dux-product/Dux Roadmap.md` | new note — RICE scoring does not exist in source; gate-based P0/P1/P2 priority used instead, disclosed explicitly rather than fabricated |
| `areas/engineering/architecture.md` | `wiki/areas/dux-architecture/Architecture Overview.md` | pre-existing scope match — system architecture, tech stack, deployment topology, all present |
| `areas/growth/onboarding.md` | `wiki/areas/dux-operations/Dux Onboarding & Activation.md` | new note — activation funnel is flagged illustrative/unvalidated per source; no A/B experiment log exists in source (flagged, not fabricated) |
| `areas/customer-success/support-playbook.md` | `wiki/areas/dux-operations/Dux Support Playbook.md` | new note — ticket-category volumes do not exist in source (flagged, not fabricated); escalation/severity/tier structure is fully real |
| `resources/api-docs/public-api.md` | `wiki/resources/dux-api/API Overview.md` + [[Application API]] + [[Public Data API]] | pre-existing scope match — auth, endpoints, rate limits, error codes all present |
| `resources/competitive-intel/landscape.md` | `wiki/areas/dux-gtm/Competitive Positioning & POC.md` | pre-existing scope match — competitor matrix, positioning, pricing all present |

`Welcome.md`, `_templates/prd.md`, `_templates/adr.md`, and `_templates/runbook.md` were created at the exact literal paths requested.

## 5. Data-integrity spot checks

Cross-checked a sample of high-stakes facts between source and note to confirm no silent drift during rewrite:

| Fact | Source | Note | Match |
|---|---|---|---|
| Capacity envelope final value | `traceability.md`: 2,160h (D-40), backlog 2,118h (~98.1%) | [[Dux Portfolio]] | Match |
| Kill-switch propagation SLO | `kill-switch-hitl.md` / `quick-reference.md`: p99 <5s (L2-L4), <=30s (L1) | [[Quick Reference Card]] | Match |
| Cost gate figures | `decisions-log.md`/`ci-cd-testing.md`: $0.675 soft breaker, $0.55 CI gate, $0.75 Gate-1 hard | [[CI-CD & Testing|CI/CD & Testing]], [[Quick Reference Card]] | Match |
| Write-action HITL split | `mcp-security.md`: 3 unattended (network.blocklist_add, policy.deploy_device_config, ticket.create_remediation), 2 mandatory-HITL (endpoint.isolate, patch.deploy_special_devices) | [[MCP Security]], [[AI Safety Overview]], [[Quick Reference Card]] | Match |
| Gartner quote retraction | `competitive.md`: unconfirmed, retracted D-48 | [[Competitive Positioning & POC]] | Match |
| Legal entity name | `vision-reference.md`/`decisions-log.md`: Dux, Inc. (not "Dux Technologies Inc.") | [[Dux, Inc.]], [[GTM Guardrails]], [[External Corrections 2026-07]] | Match |

No dropped metrics, decisions, names, or dates were found in this sample.

## 6. Deeper verification pass (2026-07-21, post-report)

The status below was originally reported as "0 reconciliation gaps" based on citation coverage (every source file appears in some note's `sources:` field) plus a 6-fact spot-check. When asked to confirm that claim rigorously, a stronger check was run: every enumerable ID in the two densest source files was extracted and grepped against the whole vault — 57 decision IDs (`decisions-log.md`) and 21 ADR IDs (`adr-index.md`), then extended to 31 FR IDs and 28 US IDs (`traceability-matrix.md`).

**This found real gaps that citation coverage had missed** — decisions and features whose source file was correctly cited somewhere, but whose specific content never made it into synthesized prose:

| Gap | Materiality | Resolution |
|---|---|---|
| ADR-018 (frontend design system: React Aria/Radix/shadcn) | High — a full architectural decision, absent everywhere | Added to [[Dux Architecture Decision Records]] and [[Architecture Overview]] |
| ADR-019 (data visualization: Visx, Cytoscape.js/Sigma) | High — same | Added to [[Dux Architecture Decision Records]] and [[Architecture Overview]] |
| Mandiant mean-time-to-exploit figures (M-Trends 2026: ~-1 day 2024, ~-7 days 2025) | Medium — existence was mentioned, specific numbers were dropped | Added to [[Lean Canvas]] Problem row |
| US-025 Outcome Learning / FR-027 (deferred, post-Gate-3 candidate) | Low-medium — a draft, flagged feature | Added to [[Dux Portfolio]] |
| US-027 Proactive Tool Discovery / FR-029 (Gate-2 candidate, 44h costed) | Low-medium — a real, costed, near-term feature | Added to [[Dux Portfolio]] |

A further set of "uncited" IDs (D-27, D-37, D-41, D-46, D-52, D-53, ADR-014, FR-007/015/017/021/022/023/024/028/031) were checked individually and found to be **citation-only gaps** — their substance is already present in the relevant note's prose, just not tagged with the specific ID, or the decision is superseded/absorbed by a later decision that is covered in depth (e.g. D-27's LiteLLM bake-off question is moot once D-33/D-34's LiteLLM removal is covered).

**Lesson for future ingests in this vault:** citation-field coverage proves every source file was touched; it does not prove every fact inside a large, multi-decision source file survived synthesis. For files with a dense enumerable ID space (a decisions log, an ADR index, a traceability matrix), an ID-cross-reference check like this one is a stronger integrity test and should be run before declaring zero gaps, not just spot-checked.

## 7. Full per-file validation pass (2026-07-21, complete)

§6 checked only the 2 densest files. On request, the same ID cross-reference was extended to **all 68 source files**, across every ID family the corpus uses (D-##, ADR-##, FR-##, US-##, BR-##, EP-##, GOV-##/GOV-TOOL-##, KS-##, PS-##, ISO-##, OI-##, NFR-##/TR-NFR-##, MC-##, CS-##, AI-##) — roughly 900 IDs checked. The full per-file record, with each finding individually triaged into real-gap / citation-only / intentional-compression, lives at **[[validation-checklist]]**.

**This pass found 2 more real, material gaps that the 2-file check had not covered:**

| Gap | Materiality | Resolution |
|---|---|---|
| Governance Kernel: GOV-002/005/006/008/009/010/011/012/013 and the full `GOV-TOOL-*` risk matrix | **High** — the safety spine's individual gate thresholds (cost caps, iteration limits, DLP rules, blast-radius tiers) were absent; only the chain shape and GOV-014 were covered | Added the full 13-gate table, cost-threshold order, and latency budget to [[Governance Kernel]] |
| TR-NFR-005 through TR-NFR-015 (10 of 15 target rows) | Medium — the table was explicitly labeled "(selected)," a self-disclosed but real gap | Completed to the full 15-row table in [[Observability & SLO]] |

The 22 files from the earlier, un-audited product-domain session were also personally re-read against their notes as part of this pass (previously verified only by citation).

## 8. Numeric-value and diagram-coverage pass (2026-07-21, complete)

§7's ID cross-reference cannot catch a fact that carries no ID — a dropped dollar figure, percentage, or benchmark number. On request to re-verify end to end, a second, independent check extracted every dollar amount and percentage in all 68 source files and cross-referenced them against the vault the same way, then separately verified that all 11 diagrams in `architecture-diagrams.md` (the corpus's single end-to-end diagram set) are represented somewhere.

**This pass found 4 more real gaps, distinct in kind from the ID-based findings — all fixed:**

| Gap | Materiality | Resolution |
|---|---|---|
| An entire competitor row (Armis/Averlon/RunSybil/IONIX) — including ServiceNow's $7.75B acquisition of Armis, RunSybil's $40M Khosla-led round, and an investor-conflict disclosure | **High** — real, current market intelligence dropped entirely from a competitive-landscape resource | Row and a full "2026 developments" paragraph added to [[Competitive Positioning & POC]] |
| CaMeL benchmark evaluation (CaMeL+ paper 67% vs. Dux CI ~77% defended/~84% undefended, measured via AgentDojo v1.2) | Medium-high — a safety-boundary's own evaluation numbers, absent everywhere | New "Evaluation" section added to [[CaMeL]] |
| Agent behavioral-baseline table missing its Tool-distribution and Cache-hit columns | Medium | Restored in [[Agent Identity]] |
| Fallback-model latency figure (+28% for `claude-sonnet-4-6`) dropped from the R3 runbook row | Low-medium | Added to [[AI Safety Incident Runbooks]] |

Also fixed as part of this pass: a missing "Prioritization layers (CVSS+EPSS)" competitor row (low materiality, added alongside the Armis row); 4 diagram-citation gaps where the note's own diagram already covered the source diagram's content but never cited it ([[CaMeL]] diagram 4, [[Governance Kernel]] diagram 5, [[Application API]] diagram 7, [[Observability & SLO]] diagram 11) — now cited. Two flagged dollar/percentage values were confirmed as pre-superseded historical figures the source itself marks non-authoritative (`$0.28–0.32`, `$0.45`, `$0.50` cost-model drafts explicitly superseded by the current $0.55/$0.75 gates) — correctly left out, not a gap.

**Running total after three passes: 11 real content gaps found, all fixed.**

## 9. Backtick/inline-code span pass (2026-07-21, complete)

On request to keep reverifying, a fourth lens was applied: every backtick-wrapped inline-code span (endpoint paths, table/column names, function signatures, config keys, alert names, CLI commands) across all 68 files was extracted and cross-referenced against the vault — several thousand spans, with the baseline "miss rate" running 30-60% per file, since most spans are implementation syntax a documentation-level note legitimately paraphrases rather than reproduces verbatim (SQL snippets, class names, file paths, git hashes). Each file's miss list was read in full and triaged by hand; exhaustive automated triage was not possible at this volume.

**6 more real gaps found in the 5 highest-value files checked this way, all fixed:**

| Gap | Materiality | Resolution |
|---|---|---|
| **D-32**, an entire decision (external "TDD" review rejected on every axis — no AWS AgentCore, no Kafka, no Airbyte, no Pinecone/Neo4j-first, no composite `DUX_SCORE`), was referenced only inside an ID range in one note, never explained | **High** — a full decision with real architectural reasoning, invisible | New "External review disposition" section added to [[Dux Decisions Log]], plus D-31's superseded reranker decision for context |
| The 33-vendor roadmap catalog in [[Dux Catalogs — Registries of Record]] gave only role counts, never named a single vendor | **High** for a note whose stated purpose is to be the registry of record | Full role-grouped 33-vendor table added |
| MCP's 7-tool **read-only** catalog (rate limits, integration IDs, per-tool threat stories) was entirely absent from [[MCP Security]] — only the write-tool catalog was covered | **Medium-high** | Full read-only tool catalog table added |
| The Gate-2 triage model path (`amazon.titan-text-lite-v2`, retiring an earlier self-hosted vLLM+Phi-4 option) was missing from [[Architecture Overview]] | Medium | Added alongside the existing Gate-1 S-LLM/P-LLM detail |
| Auth hardening findings (refresh-token-family reuse detection, the pre-auth brute-force rate-limit backstop, the JWT force-revocation denylist) were absent from [[Multi-Tenancy]] | Medium | New "Auth hardening" subsection added |
| The consolidated Prometheus metrics list backing the cost/safety dashboard was never assembled in one place in [[Observability & SLO]] | Low-medium | Added |

**Diminishing returns became visible partway through this pass.** The first 5 files checked (the corpus's largest/densest, and where every prior gap had also been found) yielded 6 real gaps; the 6th file checked (`incident-runbooks.md`, 39 misses) yielded none — every miss there was an internal alert name or verification snippet whose substance was already correctly captured in prose.

**Extension of the same pass (on request to keep verifying):** 6 more files were checked with the same backtick lens — `data-model.md`, `public-data-api.md`, `kill-switch-hitl.md`, `runbooks.md`, `architecture-overview.md`, `traceability-matrix.md` — completing all 12 files originally scoped for this lens. **2 more real gaps found:**

| Gap | Materiality | Resolution |
|---|---|---|
| [[Data Model]]'s "Core entities" table covered roughly half the ERD's entities and was honestly labeled "(selected)" — the same self-disclosed-but-real gap pattern as the earlier TR-NFR table | Medium-high — 10+ entities (`VULNERABILITY_INSTANCE`, `CHAT_SESSION`/`MESSAGE`/`ACTION`, `USER_PREFERENCE` + 2 related tables, `ATTACK_PATH`, `CONTROL`, `WEBHOOK_DEAD_LETTER`, etc.) entirely absent | Table expanded to the full entity list |
| An entire runbook (NVD sync stale, `NVD_SYNC_WARN`/`NVD_SYNC_STALE` triggers) was missing from [[Seed Operational Runbooks]] | Medium | Runbook added |

The other 4 files checked in this extension (`public-data-api.md`, `kill-switch-hitl.md`, `architecture-overview.md`, `traceability-matrix.md`) came back clean — every flagged span was DTO field syntax, internal class names, or file-path references already covered conceptually. **All 12 of the originally-planned highest-risk files are now checked with this lens; 8 of 12 yielded at least one real fix.**

**Running total after four passes: 19 real content gaps found, all fixed.**

## 10. Status

- **Source files accounted for: 68/68 (100%)**
- **Notes with no matching source file:** none — every new note in this ingest carries a non-empty `sources:` field or is explicitly a cross-cutting hub/canonical page with no single source (declared as such)
- **Reconciliation gaps found across four passes: 19 material, all fixed** — see §6-§9 for the full list and materiality reasoning
- **Hundreds of additional "missing" flags** (IDs, dollar/percentage values, and backtick-wrapped code spans) were individually triaged and closed as non-issues — citation-only, intentional register-compression, superseded historical figures, or legitimate documentation-level paraphrase of implementation syntax. See [[validation-checklist]] for the file-by-file reasoning; not asserted in aggregate.
- **Each of the four verification methods used caught gaps the previous methods missed, and the backtick pass showed its own detection ceiling within the same session (6 gaps in 5 files, then 0 in the 6th).** This is evidence of convergence, not proof of exhaustion — a corpus this size cannot be certified gap-free by sampling-based methods, only checked with progressively diminishing returns. No further claim of "zero gaps" is made anywhere in this document.
- **Flagged (not fabricated) data gaps:** activation-experiment log (growth), support-ticket-category volumes (customer success), RICE scoring (product roadmap) — none exist in the source corpus; each is explicitly marked "source data needed" in its note rather than invented

## Sources

This report synthesizes across the full `.raw/dux/` mirror; see [[Dux Overview]] for the domain map and [[validation-checklist]] for the per-file record.

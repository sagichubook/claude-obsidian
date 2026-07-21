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

## 6. Status

- **Source files accounted for: 68/68 (100%)**
- **Notes with no matching source file:** none — every new note in this ingest carries a non-empty `sources:` field or is explicitly a cross-cutting hub/canonical page with no single source (declared as such)
- **Reconciliation gaps found: 0**
- **Flagged (not fabricated) data gaps:** activation-experiment log (growth), support-ticket-category volumes (customer success), RICE scoring (product roadmap) — none exist in the source corpus; each is explicitly marked "source data needed" in its note rather than invented

## Sources

This report synthesizes across the full `.raw/dux/` mirror; see [[Dux Overview]] for the domain map.

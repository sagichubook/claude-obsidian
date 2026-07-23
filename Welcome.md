---
type: meta
title: "Welcome"
created: 2026-07-21
updated: 2026-07-21
tags: [meta, welcome]
related: ["[[Dux Overview]]", "[[index]]"]
---

# Welcome

This vault is organized with the **PARA method** (Projects / Areas / Resources / Archives), and its primary content is the complete knowledge base for **Dux, Inc.**, an agentic exposure-management SaaS platform, ingested from `C:\Users\User\dux` (68 source documents, mirrored at `.raw/dux/`).

## Start here

- **[[Dux Overview]]** — the single hub for the entire Dux corpus: scope, standards, reading paths by role, domain map.
- **[[index|Wiki Index]]** — the master catalog of every note in this vault.
- **[[hot|Hot Cache]]** — recent context, updated each session.

## PARA structure

| Bucket | What lives here | Vault path |
|---|---|---|
| **Projects** | Work with a deadline or deliverable | `wiki/projects/` |
| **Areas** | Ongoing responsibilities, reviewed on a cadence | `wiki/areas/` |
| **Resources** | Reference material — registries, API docs, concepts | `wiki/resources/` |
| **Archives** | Completed, deprecated, or superseded work | `wiki/archives/` |

## Cross-cutting hubs

For a role-based or function-based entry point rather than a domain-based one:

- **[[Product Hub]]** — product spec, roadmap, feature surfaces
- **[[Engineering Hub]]** — architecture, AI safety, engineering standards, CI/CD
- **[[Growth Hub]]** — GTM, pricing, competitive positioning, claims guardrails
- **[[Customer Success Hub]]** — onboarding, support, health monitoring, status page
- **[[Legal-Finance Hub]]** — governance, compliance, decisions log, capacity/cost

## Dux domain map

| Domain | PARA bucket | Vault location |
|---|---|---|
| Product | Area | `wiki/areas/dux-product/` |
| Architecture | Area | `wiki/areas/dux-architecture/` |
| API contracts | Resource | `wiki/resources/dux-api/` |
| AI safety | Area | `wiki/areas/dux-ai-safety/` |
| Engineering | Area | `wiki/areas/dux-engineering/` |
| Operations | Area | `wiki/areas/dux-operations/` |
| Governance & meta | Area | `wiki/areas/dux-governance/` |
| GTM | Area | `wiki/areas/dux-gtm/` |
| Execution backlog | Project | `wiki/projects/dux/` |
| Registries, entities, concepts | Resource | `wiki/resources/` |

## Vault operations

- Run `lint the wiki` every 10-15 ingests to catch orphans and gaps.
- Drop a new source into `.raw/`, then say "ingest [filename]" to add it.
- See [[migration-audit|Migration Audit]] for the source-to-note reconciliation report from the 2026-07-21 full-corpus ingest.

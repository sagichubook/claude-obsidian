---
type: concept
title: "World Model"
complexity: intermediate
domain: "dux/architecture"
aliases: []
created: 2026-07-21
updated: 2026-07-21
tags: [concept, dux, dux/architecture]
status: current
related: ["[[Dux Agent]]", "[[CaMeL]]"]
sources: [".raw/dux/10-product/taxonomy.md", ".raw/dux/10-product/catalogs.md"]
---

# World Model

## Definition

A proper noun — a versioned artifact (`world_model_versions`, ADR-003) — never lowercased "world model." The evidence substrate every [[Dux Agent]] assessment reasons over.

## How It Works

Eight evidence types: `ASSET` · `FINDING` · `CONTROL` · `OWNERSHIP_EVIDENCE` · `EXPLOITABILITY_ASSESSMENT` · `ASSESSMENT_REASONING_STEP` · `ATTACK_PATH` · `CONTROL_MAPPING`. Each connector sync that materially changes ingested evidence bumps `world_model_versions`; a 24-hour in-flight compatibility window plus a purge job manage the versioning cost.

**Relationship Graph Engine** is the customer-facing name for the vulnerability ↔ asset ↔ control mapping capability backed by `ATTACK_PATH` and `CONTROL_MAPPING` — it names something already built, not a new capability. Today this is **single-hop mapping only** (one vulnerability → one asset → one control); multi-hop lateral-movement kill-chain traversal is roadmap.

## Why It Matters

A new World Model evidence type must extend the [[Dux Catalogs — Registries of Record]] registry before it extends any connector annex — evidence typing is corpus-governed, not connector-local. Evidence freshness caps confidence (H8): a stale connector doesn't fail loudly, it feeds stale evidence into the World Model, which produces a wrong verdict — treated as a safety defect, not a data-quality nit.

## Examples

- `network_exposure` verdict (`internet` / `external` / `internal` / `unreachable`) is derived from World Model evidence, not computed independently.
- Agentic RAG retrieval (ADR-020 R2) runs against the World Model's graph projection (Apache AGE) and vector store (pgvector) in parallel with episodic memory and live threat-intel APIs.

## Connections

- [[Dux Agent]] — the consumer of World Model evidence
- [[CaMeL]] — the P-LLM tier reasons over World Model context, never raw untrusted text

## Sources

- `.raw/dux/10-product/taxonomy.md` §3
- `.raw/dux/10-product/catalogs.md` §9
- `.raw/dux/20-architecture/adr-index.md` (ADR-003 R2, ADR-020 R2)

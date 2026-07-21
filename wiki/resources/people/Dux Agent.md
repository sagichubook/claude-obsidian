---
type: entity
title: "Dux Agent"
entity_type: product
role: "Only customer-facing agent name (code id: dux-agent)"
first_mentioned: 2026-07-05
created: 2026-07-21
updated: 2026-07-21
tags: [entity, dux, dux/product]
status: current
related: ["[[Dux, Inc.]]", "[[CaMeL]]", "[[Governance Kernel]]", "[[World Model]]"]
sources: [".raw/dux/README.md", ".raw/dux/10-product/catalogs.md", ".raw/dux/10-product/taxonomy.md"]
---

# Dux Agent

## Overview

The **only** customer-facing agent name across the whole product (naming glossary, [[Dux Taxonomy and Controlled Vocabulary]] §5). Deprecated/internal aliases (never customer-facing): "Dux AI," "AI-workers," "Mitigation/Remediation agent." Internally, the actual work is done by a set of specialized runtime services and workflow sub-agents (agent catalog, [[Dux Catalogs — Registries of Record]] §2) — "assessment agent" is the internal orchestration role name and must never appear in marketing or compliance scope.

## Key Facts

- **Agent catalog row:** `dux-agent`, layer `product_persona`, customer-visible: Y, Gate 1, primary user stories US-008–011.
- Backing runtime agents (not customer-visible): `dux-assessment` (medium blast radius), `dux-chat-guidance` (supervised), `prerequisite-extractor` / `asset-context-worker` / `control-mapping-worker` (low-blast-radius workflow subagents), `mitigation-agent` and `remediation-agent` (high blast radius, autonomous with HITL on anomaly escalation only), `dux-resident-agent` (Gate 5, physical_resident), `third-party-isv` (Series B).
- Reasoning loop implementation: a Temporal TypeScript workflow (`ExploitabilityAssessmentWorkflow`) calling the AWS Bedrock Converse API directly — no agent framework (Mastra and LangGraph.js were evaluated and explicitly removed, ADR-021 / D-35).
- Every AI worker status message on screen carries a non-color state indicator (`REASONING` / `TOOL_CALLING` / `EVALUATING` / `COMPLETE`), streamed via NATS SSE fan-out from `ExploitabilityAssessmentWorkflow` — not a framework intermediary.
- A new runtime agent requires: a blast-radius row, a kill-switch scope, an MCP allowlist, an OWASP agentic mapping, a `packages/agents/{id}/` directory (ADR-009), and a parity test.

## Connections

- [[Dux, Inc.]] — the company
- [[CaMeL]] — the dual-LLM boundary the agent's reasoning runs inside
- [[Governance Kernel]] — the gate chain every privileged action passes through
- [[World Model]] — the evidence substrate the agent reasons over
- [[Kill Switch]] — the halt-authority mechanism scoped to every agent

## Sources

- `.raw/dux/README.md`
- `.raw/dux/10-product/catalogs.md` §2
- `.raw/dux/10-product/taxonomy.md` §5
- `.raw/dux/20-architecture/adr-index.md` (ADR-021)

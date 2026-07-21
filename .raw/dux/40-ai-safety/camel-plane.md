---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-34]
---

# CaMeL-Plane (Dual-LLM Boundary)

**Purpose:** the primary Phase-1 defense against prompt injection carried by untrusted content — CVE text, exploit code, threat intel. **Parents:** BR-002, BR-003.

Based on CaMeL (Google DeepMind, arXiv:2503.18813), with CaMeL+ hardening (arXiv:2505.22852; Microsoft Zero Trust SFI 2026).

## 1. Dual-LLM model

| Component | Role | Capability |
|-----------|------|-----------|
| **S-LLM** (Suspicious) | processes untrusted content — CVE descriptions, exploits, threat intel | read-only; **never** executes tools; its output is treated as untrusted |
| **P-LLM** (Privileged) | processes sanitized tool results; drives the agent loop | executes tools; **never** sees raw untrusted content or unsanitized MCP output |

**Data flow:**

```
CVE text → S-LLM (extract prerequisites, JSON schema) → structured output → P-LLM (reason + tool calls)
         ↘ validation fail (≤3 retries)
           → agent_audit_log.camel.schema_validation_failed
           → GOVERNANCE_BLOCKED or L1

MCP tool result → S-LLM sanitizer OR structured parser → P-LLM (never raw tool JSON)
```

**Integration.** Request/response middleware invoked synchronously from Temporal workflow activities — **not** a separate durable runtime. The path per tool dispatch is `Activity → Governance Kernel → CaMeL S-LLM/P-LLM split → MCP Gateway → Activity return`. Activity timeout is 30 s wall-clock, after which a partial result escalates to HITL T3.

**The critical invariant.** P-LLM tool schemas **must not** contain unconstrained `string` fields carrying S-LLM output. Everything crossing S-LLM → P-LLM passes through strictly typed, schema-validated structures with `additionalProperties: false`. This is the boundary; a single free-text field defeats it.

## 2. CaMeL+ hardening (GOV-011–013)

| Control | Implementation | Gate |
|---------|----------------|------|
| **Input screening** — scan trusted user prompts for injection before P-LLM planning | regex + lightweight classifier (`packages/core/camel/prompt-screen.ts`), <5 ms p99 | GOV-011 |
| **Output auditing** — detect instruction leakage or tool-call hints in S-LLM output | schema validation + leakage patterns; logs `camel.output_audit_failed` | GOV-012 |
| **Tiered-risk access** — map tool tiers to HITL: read MCP = T1, write = T3+, tenant-wide = T4 | extends the Effect gate (GOV-008) | GOV-013 |

## 3. Prerequisite schema (the S-LLM output boundary)

Zod / JSON Schema with `additionalProperties: false`. Max 3 retries, then `GOVERNANCE_BLOCKED` — optionally L1 if injection is suspected.

**The `description` field is a controlled vocabulary (enum) only.** No free-text natural language ever enters P-LLM context. The full untrusted text is retained as `sources[].content_hash` (SHA-256) for audit; the P-LLM never sees raw prose.

```json
{ "type": "object", "additionalProperties": false,
  "required": ["prerequisites", "sources"],
  "properties": {
    "prerequisites": { "type": "array", "items": { "type": "object", "additionalProperties": false,
      "required": ["id","description","requires","affected_services"],
      "properties": {
        "id": { "type": "string" },
        "description": { "enum": ["network_access","authentication","local_access","user_interaction","privilege_escalation","none"] },
        "requires": { "enum": ["network_access","authentication","local_access","user_interaction"] },
        "affected_services": { "type": "array", "items": { "type": "string" } } } } },
    "sources": { "type": "array", "items": { "type": "object", "additionalProperties": false,
      "required": ["name","url","confidence"],
      "properties": { "name": { "enum": ["nvd","github","exploitdb","threat_intel"] },
        "url": { "type": "string", "format": "uri" },
        "confidence": { "type": "number", "minimum": 0, "maximum": 1 } } } } } }
```

**Citation logic (US-001).** Top 3 sources by `confidence` descending; ties broken by source priority — NVD > ExploitDB > GitHub > threat_intel.

**SSRF.** Allowlisted domains only (`nvd.nist.gov`, `github.com`, `exploit-db.com`, curated intel hosts). Private and link-local IPs are blocked on any server-side fetch.

## 4. Constrained Decoding Attack (CDA) mitigation

1. **Schema minimization** — only essential fields; no large enum dictionaries.
2. **Output sanitization** — strip instruction-like patterns before P-LLM ingestion.
3. **Critic cross-check** — compare the P-LLM's `affected_services` against `getAssetsForCVE`. A mismatch downgrades confidence and escalates to HITL T3.
4. **Calibration independence** — the 3-signal ensemble (EPSS, KEV, asset exposure) is independent; the S-LLM's own `confidence` must not influence it.
5. **Connector poisoning** — connector-asserted *controls* are untrusted for **negative verdicts**. "A control blocks this exploit" requires independent AWS-API corroboration (native `query_controls`), never the connector cache alone. A sudden coverage jump — more than 20% of assets gaining a blocking control within 24 h — downgrades confidence, escalates to HITL T3, and writes a `connector_drift_anomaly` audit record.

**Scope note: this detects velocity, not drift-from-baseline.** The 20%-in-24h check catches a sudden jump in asserted coverage; it does not catch a slow, sub-threshold erosion away from an asset's established control baseline, and there is no per-asset baseline snapshot to erode away from today. Config-drift-vs-baseline detection is an explicit roadmap item, not a silently-covered case of this check.

**Named residual: Branch Steering.** A data-flow attack that redundancy defenses cannot fully block, even under strict dual-LLM isolation (arXiv:2601.09923 — the same work that names architectural isolation the only known robust defense). This is an **accepted residual risk**; the critic cross-check (#3) is a partial mitigation, not a fix.

## 5. Benchmarks

AgentDojo v1.2, pinned in the AIBOM. Regression suite: `pnpm test:camel-benchmark`.

**Two metrics, never conflated:**

| Metric | Value | Meaning |
|--------|-------|---------|
| CaMeL+ paper task completion | 67% | completion under a strict untrusted-data policy — the enterprise-hardening baseline |
| Dux CI target | ~77% defended vs ~84% undefended | defense-layer uplift, **not** a raw completion rate |

Any customer-facing copy citing a percentage must state which metric it means.

## 6. World Model query API

A logical abstraction over tenant-scoped PostgreSQL — `ASSET`, `ASSET_RELATIONSHIP`, `CONTROL`, `FINDING`, `CVE`, `USER_PREFERENCE`, versioned through `world_model_versions`.

| Query | Purpose |
|-------|---------|
| `getAssetsForCVE` | assets affected by a CVE |
| `getControlsForAsset` | controls in force on an asset |
| `summarizeContext` | ≤2 K-token summary via `gpt-5.4-mini` |

Rate limit: 100 queries/min/tenant, then `BUDGET_EXCEEDED`. Chat read-replica lag stays under 5 s.

**Hierarchical compression**, selected by asset count:

| Tier | Asset count | Context |
|------|-------------|---------|
| L1 | <100 | full detail, ≤32 K tokens |
| L2 | 100–1 K | subnet / VPC summaries, ≤16 K tokens |
| L3 | >1 K | risk profile only, ≤8 K tokens |

## 7. Retrieval

**`rag_enabled = true` (2026-07-19, D-34)** — Agentic RAG, structured SQL retrieval extended with pgvector + Apache AGE graph retrieval and constrained decoding ([ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)). This reverses the Phase-1 `rag_enabled = false` posture below, which this section is retained to explain — the enable triggers that used to gate the flip are superseded by the D-34 decision itself, not fired organically.

**Enable-precondition status — 4 of 5 confirmed implemented or specified; 1 remains a live-execution artifact (D-55).**

- **Tenant-scoped isolation — confirmed implemented.** The original "`tenant_id`-leading HNSW" wording is corrected: pgvector's HNSW can't compose with a leading scalar column the way a btree can, so the actual implementation is `tenant_embeddings` declaratively partitioned by `tenant_id` with a local HNSW index per partition ([data-model.md](../20-architecture/data-model.md)) — ANN recall is tenant-scoped by construction, not by index-key ordering.
- **Hybrid vector + BM25 — confirmed implemented.** Agentic RAG runs hybrid vector + BM25 retrieval over `tenant_embeddings`, extended with the Apache AGE graph layer ([data-model.md](../20-architecture/data-model.md); [ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)).
- **Poisoning controls — embedding-signing now specified.** Graph-edge poisoning has per-edge provenance + integrity hashing (ADR-020 R2), and connector rows carry `source_connector_id`/`integrity_hash` for transport integrity (below). **`tenant_embeddings` rows get the identical pattern (D-55):** each row carries an `integrity_hash` — SHA-256 over `(source_content, embedding_vector, tenant_id, source_connector_id)` — computed at write time and checked at retrieval time. Same trust rule as the graph/connector controls: a valid hash confirms the embedding hasn't been altered since write, but **the hash alone cannot justify a false-negative verdict** (CDA #5) — it's tamper-evidence, not an attestation that the source content itself was truthful.
- **LLM04/LLM08 reassessment — closed (D-55).** See [owasp-assessments.md §2](owasp-assessments.md#2-owasp-llm-llm01-10) — both rows updated from "reassessment triggered" to present-tense status.
- **ISO-012 adversarial-neighbor test execution — still open, tracked as [OI-59](../00-meta/open-items.md).** This is a docs-only repo; it cannot execute a live test run against the tenant-scoped HNSW index. Narrowed out of OI-41, which is otherwise closed.

Prior Phase-1 posture, retained for context: no vector RAG, structured SQL only, `pgvector`/`tenant_embeddings` provisioned but inactive. Enable triggers were: a chat p95 SLO breach attributable to structured-query limits, **or** a threat-intel corpus exceeding structured-ingest capacity.

**Connector trust model.** Connector-synced rows carry `source_connector_id` and `integrity_hash` — these establish *transport* integrity only. They may support positive evidence, but cannot on their own justify a false-negative verdict (CDA #5).

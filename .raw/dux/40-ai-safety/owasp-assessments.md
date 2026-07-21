---
owner: Security
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-21, D-34, H5, H7]
---

# OWASP Assessments (Agentic ASI01–10 · LLM01–10 · MCP Top 10)

**Purpose:** the standing OWASP risk assessments and their maturity ratings. **Parents:** BR-003.

| Property | Value |
|----------|-------|
| Assessment date | 2026-06-21 |
| Review cadence | quarterly, or on any MCP-catalog change |
| Assessors | Founder + Engineering lead |
| Pre-seed exit | **two** assessments required (Agentic + LLM), re-run and stored as artifacts — never prose-only |

The agentic rows trace to the **OWASP Top 10 for Agentic Applications 2026** (published Dec 2025) — the current peer-reviewed list, not a generic "agentic top 10" label.

**Executed code changes ASI05.** Under ADR-015 R4, investigation code executes at Gate 1. The self-hosted Firecracker microVM, the `ScriptSecurityScanner` AST pre-scan, egress DROP, and the governance kernel are the controls that hold ASI05 at Partial with execution live.

**ASI02 and ASI09 reflect the D-17 mixed posture (2026-07-16, Founder + Engineering lead).** Mandatory HITL gates 2 of 5 canonical write actions (`endpoint.isolate`, `patch.deploy_special_devices`); the other 3 (`network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`) are anomaly-only, executing unattended by default. For those 3, the primary controls are governance-kernel gates (GOV-001–014), the kill switch, and audit — not HITL. For the 2 gated actions, HITL is the primary control. The residual rating on both rows stays close to the gated-write baseline, bounded by GOV-014's confidence floor, the rollback-on-fail requirement, and anomaly escalation on the unattended surface.

## 1. OWASP Agentic (ASI01–10)

| ID | Risk | Maturity | Residual | Key controls |
|----|------|----------|----------|--------------|
| ASI01 | Agent Goal Hijack | Implemented | Low | CaMeL L1–L2, GOV-001, prompt-injection CI |
| ASI02 | Tool Misuse | Implemented | Low-Medium | Intent/Effect/VendorActionGate (GOV-001/008/014) + kill switch + audit as primary control for the 3 unattended actions; mandatory HITL T3 for `endpoint.isolate`/`patch.deploy_special_devices`; MCP allowlist |
| ASI03 | Identity & Privilege Abuse | Partial | Medium | session JWT (SPIFFE), PS-005/011; mTLS + SPIRE at Month 3 |
| ASI04 | Agentic Supply Chain | Partial | Medium | PS-006 hash pins, ASI04 checklist, AIBOM; ECDSA at seed |
| ASI05 | Unexpected Code Execution | Partial | Medium | Self-hosted Firecracker microVM + AST pre-scan + egress DROP + governance kernel |
| ASI06 | Memory & Context Poisoning | Partial | Medium | CaMeL, PS-003; no persistent agent memory in Phase 1 |
| ASI07 | Insecure Inter-Agent Comms | Planned | Low | single-agent in Phase 1; signed inter-agent JWT stub |
| ASI08 | Cascading Failures | Partial | Medium | gateway and per-tool breakers; workflow isolation; GOV-003–005 |
| ASI09 | Human-Agent Trust Exploitation | Partial | Medium-High | UI AI labels; mandatory HITL T3 for the 2 gated actions; governance-kernel gates + kill switch + hash-chained audit as primary control for the 3 unattended actions |
| ASI10 | Rogue Agents | Partial (kill switch and cost cap Implemented; eBPF at Series A Month 9) | Low | KS-L1–L4, cost cap, shadow-AI reconciliation, behavioral baselines |

**ASI07 scope note:** consensus mechanisms for conflicting agent recommendations, and inter-agent message/coordination limits, are undefined because multi-agent chaining itself is Planned, not built — Phase 1 runs a single reasoning-loop agent chain ([workflows](../20-architecture/workflows.md)). Both become required specs once ASI07 moves off Planned.

**Pre-seed exit:** every applicable row at Partial or better; ASI01 and ASI02 Implemented; ASI10 Partial or better.

Each ASI maps to an evidence command: `test:camel-benchmark`, `test:mcp-security`, `test:agent-identity`, `ops:test-kill-switch`, `calibration_check`.

## 2. OWASP LLM (LLM01–10)

| ID | Risk | Maturity | Gate-1 blocker? |
|----|------|----------|-----------------|
| LLM01 | Prompt Injection | Implemented (Promptfoo block + Garak nightly) | No |
| LLM02 | Sensitive Info Disclosure | Partial in Weeks 1–10 (regex) → Implemented from Week 11 (Presidio) | No |
| LLM03 | Supply Chain | Partial → Implemented after the Week-8 pin gate | No |
| LLM04 | Data / Model Poisoning | **Reassessed and Implemented (D-55).** Agentic RAG + Apache AGE graph is live ([ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)): per-edge provenance + integrity hashing covers graph-poisoning, `source_connector_id`/`integrity_hash` covers connector-row transport integrity, and `tenant_embeddings` rows now carry the same integrity-hash pattern ([camel-plane §7](camel-plane.md#7-retrieval)). Live adversarial-neighbor testing against the HNSW index remains a residual, tracked as [OI-59](../00-meta/open-items.md) — an execution artifact, not a spec gap | No |
| LLM05 | Improper Output Handling | Partial → Implemented at Week 9 (schema pins) | No |
| LLM06 | Excessive Agency | Implemented (allowlist, KS-L1–L4, 50 calls/session, supervised default) | No |
| LLM07 | System Prompt Leakage | Partial (seed red team) | No |
| LLM08 | Vector / Embedding Weaknesses | **Reassessed and Implemented (D-55)** — `rag_enabled = true`, pgvector + Apache AGE live. Constrained decoding (ADR-020 R2) mitigates reasoning-layer hallucination; `tenant_embeddings` per-row integrity hashing ([camel-plane §7](camel-plane.md#7-retrieval)) mitigates embedding-layer tampering. The ISO-012 adversarial-neighbor test against the live tenant-scoped HNSW index is a residual execution artifact, tracked as [OI-59](../00-meta/open-items.md) | No |
| LLM09 | Misinformation | Partial → Implemented at Gate 1 | **Yes — EXP-CIT-001** |
| LLM10 | Unbounded Consumption | Implemented (rate limits, budgets, CostCap, kill switch) | No |

**LLM09 is the single Gate-1 blocker.** `pnpm test:exposure-citations` (EXP-CIT-001) asserts that every `exploitable` or `likely` claim in Exposure Analysis (US-011) carries at least one allowed-source URL. **EXP-CIT-001 is a presence check plus a resolution check, not presence alone:** at generation time it also resolves the cited CVE ID against NVD and diffs the claimed CVSS score and description against the resolved NVD record. A citation with an allowed-source URL that fails NVD resolution, or whose CVSS/description diverges from the resolved record, fails EXP-CIT-001 the same as a missing citation. It is a CI merge block.

Overall exit is met with a documented remediation backlog for the Partial items. Medium residual risks are accepted until their dated remediation, on Founder + Engineering-lead sign-off.

## 3. Citation integrity (H7, Gate-2 hardening)

**"Citation-first" is itself an injection surface.** Research tools pull untrusted web content, so a poisoned repository or writeup surfaced as a citation delivers attacker-controlled content carrying Dux's authority.

The allowlist is therefore **domain *and* integrity**:

- Every surfaced citation carries a fetched-content hash and a source-reputation flag.
- Citations to mutable sources — a GitHub README, a Medium post — are labeled **unverified-third-party**, and are never presented with NVD-grade authority.
- A citation-poisoning case is added to the prompt-injection suite.

## 4. Remediation calendar

| When | Items |
|------|-------|
| Week 8 (Gate 2) | Self-hosted Firecracker microVM operational; HITL API; LLM03 pin gate |
| Week 9 | LLM05 — MCP schema pins (PS-006) |
| Week 11 | LLM02 — Presidio DLP |
| Week 12 | ASI02 — HITL UI, `chat_write_tools` |
| Month 3 | ASI03 — SPIRE proof of concept |
| Seed | LLM07 prompt-extraction red team; ASI04 ECDSA (PS-009); ASI06 / LLM04 vector-poison controls (on RAG); ASI09 impersonation defenses |
| Series A Month 9 | ASI10 — eBPF |
| Series B | ASI08 — chaos engineering |

## 5. OWASP MCP Top 10 crosswalk

| MCP risk | Policy IDs | Maturity (pre-seed) | Seed Gate-2 exit |
|----------|-----------|---------------------|------------------|
| Tool poisoning | PS-006, PS-003, `mcp-scan` CI | Implemented | — |
| Confused deputy | PS-005, PS-011 | Partial | `test:mcp-security --case MCP-011` (OAuth 2.1 resource binding) |
| SSRF / egress | PS-004, PS-003 | Implemented | — |
| Supply chain | PS-006, ASI04 | Partial | PS-009 ECDSA verify + `mcp-scan` green |
| Session / auth | PS-005, PS-011 | Partial | `MCP-005` cross-tenant reject + PS-010 seccomp |

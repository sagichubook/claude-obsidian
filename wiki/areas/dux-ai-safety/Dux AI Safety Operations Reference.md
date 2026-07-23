---
type: resource
title: "Dux AI Safety Operations Reference"
topic: "dux/ai-safety"
created: 2026-07-22
updated: 2026-07-23
tags: [resource, dux, dux/ai-safety]
status: mature
sources: [".raw/dux/40-ai-safety/agent-identity.md", ".raw/dux/40-ai-safety/confidence-calibration.md", ".raw/dux/40-ai-safety/owasp-assessments.md", ".raw/dux/40-ai-safety/incident-runbooks.md"]
related: ["[[Dux]]", "[[Dux AI Safety Guide]]", "[[Dux Operations Guide]]", "[[Dux Customer Success Guide]]"]
---

# Dux AI Safety Operations Reference

### At 3 a.m., nobody wants to read architecture docs — they want to know which command to run

An agent walks into a customer's environment holding real credentials and the authority to isolate a production endpoint. [[Dux AI Safety Guide]] is the architecture that makes that survivable in the abstract: the gates, the kill switch, the dual-LLM boundary. This page is the operations layer sitting on top of it — the part that matters at 3 a.m. when something has actually gone sideways. It answers four questions in order: how does an agent prove it's the agent it claims to be, how confident does an assessment have to be before Dux lets it speak with authority, how does the whole system get scored against the industry's own risk frameworks, and — the one that actually gets used under pressure — what is the exact runbook when something breaks.

Four sections, in that order, because that's the order an incident actually unfolds: an identity gets compromised or spoofed, a verdict comes out wrong, an OWASP category slips, or a named failure mode fires and someone needs the twelve-section runbook, not a design discussion.

---

## Table of contents

1. [[#1 · Agent identity]]
   - [[#1.1 · JWT claim schema]]
   - [[#1.2 · Identity model]]
   - [[#1.3 · Credential types]]
   - [[#1.4 · Migration ladder]]
   - [[#1.5 · Lifecycle]]
   - [[#1.6 · Physical-residency agents]]
   - [[#1.7 · Shadow AI detection and behavioral baselines]]
   - [[#1.8 · Inter-agent calls]]
   - [[#1.9 · Authorization model]]
   - [[#1.10 · Delegation recording]]
   - [[#1.11 · Audit and WORM chain]]
2. [[#2 · Confidence calibration]]
   - [[#2.1 · Three-signal ensemble]]
   - [[#2.2 · Platt scaling and the ECE gate]]
   - [[#2.3 · CalibrationRecord]]
   - [[#2.4 · Verbalized confidence insufficiency]]
   - [[#2.5 · From score to verdict]]
   - [[#2.6 · insufficient_data_reason enum]]
   - [[#2.7 · Critic rules]]
   - [[#2.8 · Sample-size power analysis]]
   - [[#2.9 · Per-stratum minimums]]
   - [[#2.10 · Re-sampling schedule]]
3. [[#3 · OWASP assessments]]
   - [[#3.1 · Assessment metadata]]
   - [[#3.2 · Agentic risks — ASI01–ASI10]]
   - [[#3.3 · LLM risks — LLM01–LLM10]]
   - [[#3.4 · Pre-seed exit criteria]]
   - [[#3.5 · Evidence commands mapping]]
   - [[#3.6 · LLM04 and LLM08 reassessments]]
   - [[#3.7 · Citation integrity]]
   - [[#3.8 · OWASP MCP Top 10 crosswalk]]
   - [[#3.9 · Remediation calendar]]
4. [[#4 · Incident runbooks]]
   - [[#4.1 · Runbook template]]
   - [[#4.2 · The twelve runbooks]]
   - [[#4.3 · R1 — Cross-tenant context leak]]
   - [[#4.4 · R2 — Token cost runaway]]
   - [[#4.5 · R3 — Model provider outage]]
   - [[#4.6 · R3b — Model EOL / forced deprecation]]
   - [[#4.7 · R4 — MCP dependency failure]]
   - [[#4.8 · R5 — Rate limit cascade]]
   - [[#4.9 · R6 — Context window exhaustion]]
   - [[#4.10 · R7 — Prompt cache invalidation]]
   - [[#4.11 · R8 — Hallucinated-CVE citation]]
   - [[#4.12 · R9 — Alert fatigue]]
   - [[#4.13 · R10 — Memory / context poisoning]]
   - [[#4.14 · R11 — Coordination overhead]]
   - [[#4.15 · R12 — Prompt brittleness]]
   - [[#4.16 · Runbook map]]

---

## 1 · Agent identity

Every write the governance kernel authorizes eventually traces back to one question: *which agent is this, and does it still have the right to be here?* Get identity wrong and every downstream gate is checking the wrong thing.

**Purpose:** how assessment agents — and the optional Gate-5 physical-residency agents — are identified, authenticated, authorized, and audited. **Parents:** BR-003, BR-007.

Phase 1 uses JWT with SPIFFE-format claims. Full SPIFFE/SPIRE X.509 SVIDs target a Month-3 proof of concept — a deliberately staged migration rather than a big-bang cutover, laid out in the ladder below.

### 1.1 · JWT claim schema

> Canonical — ADR-001 links here.

Seven claims, and every agent token carries them all — this is the shape every downstream gate reads from:

| Claim | Required | Example |
|-------|----------|---------|
| `sub` | yes | `spiffe://dux.io/tenant/{tenant_id}/agent/{agent_id}` |
| `agent_id` | yes, for agent tokens | uuid |
| `session_id` | yes, for agent tokens | assessment or chat session uuid |
| `tenant_id` | yes | tenant scope |
| `allowed_tools` | yes, for MCP sessions | `["query_assets","query_controls"]` |
| `aud` | yes | `api.dux.io` / `app.dux.io` |
| `exp` | yes | agent tokens default to 15 min |

### 1.2 · Identity model

**Attributes:** `agent_id` (UUID, tenant-scoped), `tenant_id`, `identity_ref` (slug), `agent_type` (`assessment` · `physical_resident` · `supervised`), `credential_id`, `status` (`active` · `suspended` · `revoked`).

### 1.3 · Credential types

Four credential families exist side by side, each scoped to a different surface — none of them are interchangeable:

| Type | Use |
|------|-----|
| Per-agent **session JWT** | MCP gateway authentication |
| **API key** `agt_<prefix>_<secret>` | SHA-256 hashed. Public Data API only (data scopes: `custom_metrics:read`, `vulnerability_instances:read`, `cve_research:write`, [api-overview.md §3](../30-api/api-overview.md#3-auth)) — **never** MCP tool calls, and never authenticates `POST /v1/agents` (Management plane, platform-admin JWT only) |
| **OAuth client credentials** | seed, enterprise |
| **mTLS** X.509 | post-seed, 90-day rotation |

> **One boundary is absolute:** a Public Data API key can never authenticate an MCP tool call or a `POST /v1/agents` request, regardless of what scopes it carries. Data-plane and control-plane credentials are different families by construction, not by convention.

### 1.4 · Migration ladder

The credential story moves in three deliberate stages rather than jumping straight to the hardest option:

```
pre-seed  →  seed  →  post-seed
session JWT + API key    session JWT + OAuth (Gate 2)    mTLS + SPIRE SVID
```

### 1.5 · Lifecycle

An agent's identity has a birth, a rotation cadence, and — when things go wrong — a suspension or a permanent death. Each stage has its own audit trail.

**Creation.** A platform admin calls `POST /v1/agents` (Management plane, platform-admin JWT — [api-overview.md §1](../30-api/api-overview.md#1-three-rest-planes)) with `identity_ref`, `agent_type`, and `config`. The platform generates the `agent_id` and its per-agent session JWT issuer. Default `agent_type` is `supervised`. Audit: `agent.created`.

**Rotation.** At most 2 active credentials — one API key and one session-JWT issuer, **never two keys with different scopes**. The old credential is revoked after a 24-hour grace period. Audit: `agent.credential.rotated`.

**Suspension.** Fires an L2 kill switch: new session credentials are blocked and existing sessions terminated. Reversible without regenerating credentials. Suspending a `physical_resident` agent may escalate to L3.

**Revocation.** Permanent. `status = revoked`, all sessions terminated, and the `agent_id` is **never reused** (recorded in `agents_tombstone`). Decommissioning purges cached credentials from Valkey and Vault; anonymizes PII once retention expires; and opens an AIBOM removal ticket.

**Autonomous approval.** `max_autonomy: autonomous` requires tenant-admin approval: `POST /v1/agents/{id}/autonomy-request` → approve → AIBOM and agent record updated. An agent cannot grant itself this.

### 1.6 · Physical-residency agents

> Gate 5 only. Not used by the default Unified Integration Layer.

A future, optional deployment mode gets a heartbeat contract of its own. DaemonSet `dux-resident-agent`. Heartbeat: `POST /resident-agents/{id}/heartbeat`, authenticated by **mTLS** (preferred) or a signed JWT carrying `agent_id`, `tenant_id`, `nonce`, and `exp` ≤60 s — a stale nonce is rejected. The response may carry `halt: true` to trip the kill switch.

### 1.7 · Shadow AI detection and behavioral baselines

Identity alone doesn't catch an agent that was never declared in the first place — that's what this section is for. A daily job compares declared agents (the `agents` table) against observed MCP `agent_id` headers and runtime `agent.session.started` audit records. Undeclared drift triggers **P0-B containment** plus a registry update, with an SLA of **1 hour to investigate** and **4 hours to contain**.

#### Behavioral baselines per `agent_type`

| `agent_type` | req/min | Tool distribution | Output tokens p50/p95 | Cost p50/p95 | Cache hit | Fallback rate |
|-------------|---------|-------------------|----------------------|--------------|-----------|---------------|
| `assessment` | 2–8 | `query_assets` 60%, `run_assessment` 30%, other 10% | 800 / 2,400 | $0.08 / $0.25 | 80–95% | <5% |
| `physical_resident` (Gate 5) | 0.5–2 | heartbeat 80%, sync 20% | 200 / 600 | $0.02 / $0.06 | 70–90% | <2% |

#### Anomaly detection command

```bash
pnpm admin:agent-baseline-diff --agent-type assessment --window 5m
```

Raises `DuxAgentBehaviorAnomaly` on a **2σ breach**.

### 1.8 · Inter-agent calls

> Maps to **ASI07**.

Where multi-agent chaining is enabled, **every hop requires a signed service JWT** carrying `caller_agent_id`, `callee_agent_id`, and `tenant_id`. The gateway verifies it before forwarding.

The design stub `security/design/inter-agent-jwt.md` is a pre-seed exit artifact, and becomes mandatory at seed multi-agent.

> **ASI07 scope note:** consensus mechanisms for conflicting agent recommendations, and inter-agent message/coordination limits, are undefined because multi-agent chaining itself is Planned, not built — Phase 1 runs a single reasoning-loop agent chain. Both become required specs once ASI07 moves off Planned.

### 1.9 · Authorization model

Proving who you are is only half of it — what you're allowed to do is a separate, narrower question. Least-privilege subsets live in the agent's `config.permissions`: `allowed_tools`, `max_autonomy` (`supervised` / `autonomous`), and `rate_limits`.

- **An agent cannot self-escalate.**
- **Cross-tenant access is forbidden** at the credential-validation layer.
- MCP write-tool permissions (`allowed_tools`) are **distinct from** vendor mutation permissions, which are the ADR-012 canonical actions routed through `VendorActionGate`.

### 1.10 · Delegation recording

```sql
agent_delegation_grants(user_id, agent_id, scope, granted_at, expires_at)
```

Every delegation is recorded in this table with the granting user, the target agent, the scope, and an expiry timestamp.

### 1.11 · Audit and WORM chain

Everything above only matters if it's provably recorded afterward — an unauditable identity system is just a promise. **Required fields on every action:**

| Field | Required | Notes |
|-------|----------|-------|
| `agent_id` | yes | |
| `tenant_id` | yes | |
| `credential_id` | yes | |
| `session_id` | yes | |
| `user_id` | optional | present when a user delegates |
| `action` | yes | |
| `timestamp` | yes | |
| `request_id` | yes | |

**Retention:** 2 years (GDPR RoPA).

`agent_audit_log` is **WORM append-only** with an HMAC-SHA256 hash chain. The `chain_key` lives in Vault at `audit/chain-key`, and `/audit/verify` validates the chain with the server-held key — **a database writer cannot recompute a valid chain**. The chain head is anchored hourly to MinIO Object Locking (`dux-audit-anchors/`, COMPLIANCE mode, 7-year retention).

---

## 2 · Confidence calibration

An identity check answers *who*. This section answers something harder: how sure is the agent actually allowed to sound? A model that's confidently wrong is worse than one that admits uncertainty, so nothing in Dux trusts a raw self-reported confidence number.

**Purpose:** how an assessment's `confidence_score` is produced, calibrated, and turned into a verdict label or an abstention. **Parents:** BR-002 (BRD-EXP-003 — exploitability trust → calibration pipeline → false-positive-rate KPI).

Extends `EXPLOITABILITY_ASSESSMENT.confidence_score`, which is Platt-scaled.

### 2.1 · Three-signal ensemble

No single signal is trusted alone — three independent measurements get blended into one score:

| Signal | Weight | When available |
|--------|--------|----------------|
| Mean top-1 logprob, over claim-bearing tokens | **0.40** | when the API exposes logprobs (`calibration.use_logprobs` flag) |
| Semantic entropy, over meaning-clustered completions | **0.35** | always (multi-sample, max 3 on the calibration path) |
| Verbalized confidence, from structured output | **0.25** | always |

When logprobs are unavailable, the remaining two signals renormalize to **entropy 0.54 / verbalized 0.46**.

> **Verbalized confidence alone is not sufficient.** RLHF rewards confident-sounding responses over accurate uncertainty, so a model's stated confidence is the weakest of the three signals and never stands on its own. Semantic-entropy token overhead counts against CostCap.

### 2.2 · Platt scaling and the ECE gate

A raw ensemble score isn't automatically calibrated — "80% confident" has to actually mean right 80% of the time, which is what this gate enforces. Platt scaling fits 2 parameters (A, B — a 1-D logistic regression over the raw ensemble score) and is enforced by:

```bash
pnpm test:calibration --ece-threshold 0.15
```

**ECE gate:** ≤0.15 on a golden-set holdout of **n ≥ 50**, enforced by `pnpm test:calibration --ece-threshold 0.15`. The `platt_scaling` flag requires an active `CalibrationRecord` with `ece ≤ 0.15`; without one, the runtime guard returns `CALIBRATION_MISSING` and raises a **P2 alert**.

### 2.3 · CalibrationRecord

`CalibrationRecord` is global, with fields:

| Field | Type | Notes |
|-------|------|-------|
| `model_version` | string | composite key |
| `prompt_version` | string | composite key |
| `platt_params` | jsonb | A, B parameters |
| `brier_score` | float | |
| `ece` | float | ≤0.15 required |
| `training_set_size` | integer | n ≥ 50 |
| `fitted_at` | timestamp | |
| `active` | boolean | one active record per model+prompt version |

### 2.4 · Verbalized confidence insufficiency

Verbalized confidence is the weakest signal in the ensemble. RLHF rewards confident-sounding outputs over accurate uncertainty expressions, which means a model's self-reported confidence is biased upward. This is why it carries only 0.25 weight (or 0.46 when renormalized) and is never the sole basis for a verdict.

### 2.5 · From score to verdict

Once a score is calibrated, it has to collapse into something a human or a downstream system can act on — a label, with a review requirement attached. **Abstention mapping.** Lower bound inclusive, upper bound exclusive. Unit-tested in `calibration_check`; application layer only.

| Calibrated confidence | Label | Human review |
|-----------------------|-------|--------------|
| [0.85, 1.00] | `exploitable` | if a critical asset |
| [0.70, 0.85) | `likely` | always (HITL queue) |
| [0.40, 0.70) | `unlikely` | always (HITL queue) |
| [0.00, 0.40) | `not_exploitable` | if a critical asset |
| Uncertain | `insufficient_data` | always |

> **This table governs review of the *analysis verdict*, which is separate from write-action approval.** Write actions execute unattended by default ([kill-switch-hitl](kill-switch-hitl.md)); the bands above still route critical-asset verdicts and abstentions to human review. The public API projection of these labels is in [taxonomy §1](../10-product/taxonomy.md).

### 2.6 · `insufficient_data_reason` enum

When a verdict is `insufficient_data`, the abstention carries one of three reasons:

| Value | Meaning |
|-------|---------|
| `asset_gap` | insufficient asset coverage to form a verdict |
| `intel_gap` | insufficient threat intelligence for the vulnerability |
| `context_limit` | context window exhausted before a confident signal was reached |

> `context_limit` is a US-011 UX concern and is not exposed in the public OpenAPI.

### 2.7 · Critic rules

A calibrated score can still sit on top of an internally inconsistent chain of reasoning, so a separate critic layer checks the reasoning itself, not just the number. Rule-based from Month 1; an ML critic runs in shadow from seed.

| Rule | Check | Severity | Test name |
|------|-------|----------|-----------|
| Schema compliance | Zod against `assessmentSchema`, generated from the same OpenAPI as the MCP JSON Schema | Blocking | `test:schema-parity` |
| Confidence in range | a value in 0.0–1.0 is present | Blocking | — |
| Prerequisite consistency | with no affected assets, the conclusion cannot be `exploitable` | Blocking | — |
| Source traceability | reasoning references permitted sources only | Escalation | — |
| Self-contradiction | no X and not-X in the same chain | Escalation | — |
| Tool result injection | match against `tests/fixtures/tool-injection-patterns.json` | Escalation | `test:prompt-injection` (runs last-pass in CI) |

### 2.8 · Sample-size power analysis

> Resolves OI-21.

The **n ≥ 50** holdout above wasn't picked arbitrarily — here's the statistics behind why it's the right floor. **Platt scaling fits 2 parameters** (A, B — a 1-D logistic regression over the raw ensemble score). Standard practice for logistic-regression calibration is a minimum of 10–15 held-out samples per parameter per class outcome (positive/negative) — a well-established rule of thumb (Peduzzi et al.'s "events per variable" heuristic, applied to calibration rather than the original model fit). For 2 parameters × 2 outcome classes, that floor is **40–60 samples overall** — comfortably below the existing ECE gate's own **n ≥ 50** holdout, which was already set above this floor independently.

### 2.9 · Per-stratum minimums

The overall floor is one thing; the binding constraint turns out to live in a narrower slice of the data. The golden set's CVSS-decile stratification ([ci-cd-testing §3](../50-engineering/ci-cd-testing.md)) puts ~25 CVEs per decile bin, each crossed with environment fixtures — comfortably above the 40–60 floor once crossed.

The binding constraint is **exploit-maturity strata** (30% functional / 50% PoC / 20% theoretical of 250 ≈ **75 / 125 / 50**), since `test:calibration --stratified` fits Platt parameters per stratum where per-stratum drift is suspected.

| Exploit-maturity stratum | Approx. share | n (of 250) | Status |
|--------------------------|---------------|------------|--------|
| Functional | 30% | ~75 | above floor |
| PoC | 50% | ~125 | well above floor |
| **Theoretical** | **20%** | **~50** | **at floor — watch first** |

The 20% theoretical-maturity stratum (n≈50) sits at the floor, not above it — **it is the stratum to watch first for calibration instability**, and the one that would trigger a re-sample first.

### 2.10 · Re-sampling schedule

Calibration isn't a one-time fit — the world drifts, so the schedule has to catch drift from two different sources, not just model changes:

- **On model/prompt change:** re-fit Platt parameters whenever `model_version` or `prompt_version` changes (already required — a `CalibrationRecord` is keyed on both).
- **Quarterly cadence regardless of pin changes**, to catch calibration drift from distribution shift in incoming CVEs rather than only from model changes.
- **Targeted re-sample:** a stratum whose Brier score regresses more than 10% between fits is flagged for a targeted re-sample (additional golden-set cases added to that stratum specifically) rather than a full 250-case re-collection.

---

## 3 · OWASP assessments

Identity and calibration are Dux-specific mechanisms. This section is the outside check — how the whole safety program scores against the industry's own agreed-upon risk taxonomies, re-run and stored as evidence rather than asserted from memory.

**Purpose:** the standing OWASP risk assessments and their maturity ratings. **Parents:** BR-003.

### 3.1 · Assessment metadata

| Property | Value |
|----------|-------|
| Assessment date | 2026-06-21 |
| Review cadence | quarterly, or on any MCP-catalog change |
| Assessors | Founder + Engineering lead |
| Pre-seed exit | **two** assessments required (Agentic + LLM), re-run and stored as artifacts — never prose-only |

> The agentic rows trace to the **OWASP Top 10 for Agentic Applications 2026** (published Dec 2025) — the current peer-reviewed list, not a generic "agentic top 10" label.

### 3.2 · Agentic risks — ASI01–ASI10

Ten categories, scored honestly rather than uniformly marked "Implemented" — several rows are still Partial, and that's stated plainly:

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

> **ASI05 note:** Under ADR-015 R4, investigation code executes at Gate 1. The self-hosted Firecracker microVM, the `ScriptSecurityScanner` AST pre-scan, egress DROP, and the governance kernel are the controls that hold ASI05 at Partial with execution live.

> **ASI02 and ASI09 mixed posture (D-17, 2026-07-16, Founder + Engineering lead):** Mandatory HITL gates 2 of 5 canonical write actions (`endpoint.isolate`, `patch.deploy_special_devices`); the other 3 (`network.blocklist_add`, `policy.deploy_device_config`, `ticket.create_remediation`) are anomaly-only, executing unattended by default. For those 3, the primary controls are governance-kernel gates (GOV-001–014), the kill switch, and audit — not HITL. For the 2 gated actions, HITL is the primary control. The residual rating on both rows stays close to the gated-write baseline, bounded by GOV-014's confidence floor, the rollback-on-fail requirement, and anomaly escalation on the unattended surface.

### 3.3 · LLM risks — LLM01–LLM10

The agentic-specific list above sits alongside the more general LLM Top 10 — and here the whole program comes down to a single blocking row:

| ID | Risk | Maturity | Gate-1 blocker? |
|----|------|----------|-----------------|
| LLM01 | Prompt Injection | Implemented (Promptfoo block + Garak nightly) | No |
| LLM02 | Sensitive Info Disclosure | Partial in Weeks 1–10 (regex) → Implemented from Week 11 (Presidio) | No |
| LLM03 | Supply Chain | Partial → Implemented after the Week-8 pin gate | No |
| LLM04 | Data / Model Poisoning | **Reassessed and Implemented (D-55)** | No |
| LLM05 | Improper Output Handling | Partial → Implemented at Week 9 (schema pins) | No |
| LLM06 | Excessive Agency | Implemented (allowlist, KS-L1–L4, 50 calls/session, supervised default) | No |
| LLM07 | System Prompt Leakage | Partial (seed red team) | No |
| LLM08 | Vector / Embedding Weaknesses | **Reassessed and Implemented (D-55)** | No |
| **LLM09** | **Misinformation** | **Partial → Implemented at Gate 1** | **Yes — EXP-CIT-001** |
| LLM10 | Unbounded Consumption | Implemented (rate limits, budgets, CostCap, kill switch) | No |

**LLM09 is the single Gate-1 blocker across the entire safety program.** `pnpm test:exposure-citations` (EXP-CIT-001) asserts that every `exploitable` or `likely` claim in Exposure Analysis (US-011) carries at least one allowed-source URL.

> **EXP-CIT-001 is a presence check plus a resolution check, not presence alone:** at generation time it also resolves the cited CVE ID against NVD and diffs the claimed CVSS score and description against the resolved NVD record. A citation with an allowed-source URL that fails NVD resolution, or whose CVSS/description diverges from the resolved record, fails EXP-CIT-001 the same as a missing citation. It is a CI merge block.

Overall exit is met with a documented remediation backlog for the Partial items. Medium residual risks are accepted until their dated remediation, on Founder + Engineering-lead sign-off.

### 3.4 · Pre-seed exit criteria

Distilled down to three lines, this is what actually has to be true before pre-seed exit:

- Every applicable row at **Partial or better**
- **ASI01 and ASI02:** Implemented
- **ASI10:** Partial or better

### 3.5 · Evidence commands mapping

A maturity rating means nothing without a command that actually proves it. Each ASI maps to an evidence command:

| ASI | Evidence command |
|-----|-----------------|
| ASI01 | `test:camel-benchmark` |
| ASI02 | `test:mcp-security` |
| ASI03 | `test:agent-identity` |
| ASI05 | `ops:test-kill-switch` |
| Calibration | `calibration_check` |

### 3.6 · LLM04 and LLM08 reassessments

Turning on Agentic RAG (see [[Dux AI Safety Guide]]) changed the threat surface enough to force a re-score of two rows above — here's exactly what changed. Both were reassessed under **D-55** when Agentic RAG + Apache AGE graph went live ([ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)):

**LLM04 — Data / Model Poisoning:**
- Per-edge provenance + integrity hashing covers graph-poisoning
- `source_connector_id` / `integrity_hash` covers connector-row transport integrity
- `tenant_embeddings` rows carry the same integrity-hash pattern ([camel-plane §7](camel-plane.md#7-retrieval))
- **Residual:** live adversarial-neighbor testing against the HNSW index, tracked as [OI-59](../00-meta/open-items.md)

**LLM08 — Vector / Embedding Weaknesses:**
- `rag_enabled = true`, pgvector + Apache AGE live
- Constrained decoding (ADR-020 R2) mitigates reasoning-layer hallucination
- `tenant_embeddings` per-row integrity hashing ([camel-plane §7](camel-plane.md#7-retrieval)) mitigates embedding-layer tampering
- **Residual:** ISO-012 adversarial-neighbor test against the live tenant-scoped HNSW index, tracked as [OI-59](../00-meta/open-items.md)

### 3.7 · Citation integrity

> Gate-2 hardening (H7).

**"Citation-first" is itself an injection surface.** Research tools pull untrusted web content, so a poisoned repository or writeup surfaced as a citation delivers attacker-controlled content carrying Dux's authority.

The allowlist is therefore **domain *and* integrity**:

- Every surfaced citation carries a fetched-content hash and a source-reputation flag.
- Citations to mutable sources — a GitHub README, a Medium post — are labeled **unverified-third-party**, and are never presented with NVD-grade authority.
- A citation-poisoning case is added to the prompt-injection suite.

### 3.8 · OWASP MCP Top 10 crosswalk

MCP gets its own risk taxonomy, distinct from the general LLM and Agentic ones — mapped here to the exact PS policy IDs that answer each row:

| MCP risk | Policy IDs | Maturity (pre-seed) | Seed Gate-2 exit |
|----------|-----------|---------------------|------------------|
| Tool poisoning | PS-006, PS-003, `mcp-scan` CI | Implemented | — |
| Confused deputy | PS-005, PS-011 | Partial | `test:mcp-security --case MCP-011` (OAuth 2.1 resource binding) |
| SSRF / egress | PS-004, PS-003 | Implemented | — |
| Supply chain | PS-006, ASI04 | Partial | PS-009 ECDSA verify + `mcp-scan` green |
| Session / auth | PS-005, PS-011 | Partial | `MCP-005` cross-tenant reject + PS-010 seccomp |

### 3.9 · Remediation calendar

Every "Partial" row above has a date attached, not just a promise — this is the calendar that closes them out:

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

---

## 4 · Incident runbooks

Everything above this line is preventive. This section is what happens after prevention fails — twelve named failure modes, each with an exact procedure, because "figure it out live" is not an acceptable incident-response plan for a system with write access to a customer's EDR.

**Purpose:** the twelve canonical agentic failure modes and the procedure for each. **Parents:** BR-003, BR-005.

> **This file is the source of truth for these twelve procedures.** [Seed runbooks](../60-operations/runbooks.md) add stage deltas — PagerDuty IDs, admin CLI, and a thirteenth agent-quota mode. Series A and B link here. **Do not duplicate the step tables downstream.**

### 4.1 · Runbook template

None of the twelve runbooks below reads like a general essay — each is built from the same rigid twelve-section skeleton, so an on-call engineer never has to hunt for where the containment step lives:

| § | Section | § | Section |
|---|---------|---|---------|
| 1 | Trigger | 7 | Pre-conditions |
| 2 | Service catalog | 8 | Automation gate |
| 3 | Business impact (MRR-at-risk formula) | 9 | Execution steps — command / expected / timeout / human fallback |
| 4 | User impact (PM) | 10 | AI safety check |
| 5 | System/agent boundary | 11 | Verification |
| 6 | Incident roles | 12 | Post-incident |

Section 10 is worth singling out — it's the check that keeps an incident from being "resolved" while the agent fleet is still misbehaving underneath it.

#### §10 AI safety check — all eight items, every incident

| # | Check |
|---|-------|
| a | agent loop ≤50 iterations |
| b | `admin:agent-tool-diff` clean |
| c | prompt-injection CI passing |
| d | HITL queue drained |
| e | cross-tenant `foreign_tenant_refs: 0` |
| f | shadow AI `undeclared_count: 0` |
| g | AI-BOM valid |
| h | cost-cap state confirmed |

Plus the OWASP triple: the LLM and Agentic assessments and the MCP crosswalk.

> **The AI Safety Lead (`@ai-safety-oncall`) holds 60-second halt authority, and that role cannot be merged with the Incident Commander.**

### 4.2 · The twelve runbooks

At a glance, before the full detail below — every canonical failure mode, its severity, what fires it, and the first containment move:

| # | Runbook | Type | Severity | Trigger | Core containment |
|---|---------|------|----------|---------|------------------|
| R1 | [Cross-tenant context leak](#r1--cross-tenant-context-leak) | COMPOSITE | P0-C | `DuxCrossTenantContextDetected` (`foreign_tenant_refs > 0` + isolation SLO burn ≥5%/1h), or a fuzz failure | `admin:platform-contain` (L4); agent halt ≤60 s; context audit; engage counsel if PII (GDPR/DORA, 72 h) |
| R2 | [Token cost runaway](#r2--token-cost-runaway) | COMPOSITE | P0-C / P1 | spend >3× the 7-day baseline, or >$25/hour/tenant | `admin:cost-cap enforce`; L2 kill switch; halt the top-spend agent |
| R3 | [Model provider outage](#r3--model-provider-outage) | COMPOSITE | P1 | OpenAI status ≠ operational, or `DuxLLMAvailabilityFastBurn` | `admin:model-route --fallback on` (≤60 s); golden-set spot check, halting if regression >5% |
| R4 | [MCP dependency failure](#r4--mcp-dependency-failure) | AGENTIC-SAAS | P2 | MCP errors >50% over 5 min, or a health-check failure | `admin:mcp-circuit-breaker --open`; verify no hallucinated success |
| R5 | [Rate limit cascade](#r5--rate-limit-cascade) | AGENTIC-SAAS | P2 | `DuxRateLimitCascade` — model 429 and customer 429 in the same window | jitter backoff; halt the runaway agent; escalate provider quota |
| R6 | [Context window exhaustion](#r6--context-window-exhaustion) | AGENTIC-SAAS | P2 | `DuxContextWindowExhausted` (128 K) | checkpoint at 80%, abandon at 100%; halt the runaway retry loop |
| R7 | [Prompt cache invalidation](#r7--prompt-cache-invalidation) | AGENTIC-SAAS | P2 | `DuxPromptCacheHitRateDrop` (>15% over 5 min) | identify the cause (deploy or pin); roll back, or `admin:llm-cache-warm` |
| R8 | [Hallucinated-CVE citation](#r8--hallucinated-cve-citation) | AGENTIC-SAAS | P1 | EXP-CIT-001 fails at generation time — a citation's CVE ID doesn't resolve against NVD, or its CVSS/description diverges from the resolved record | `admin:verdict-quarantine`; re-verify against NVD; re-run `test:exposure-citations` |
| R9 | [Alert fatigue](#r9--alert-fatigue) | AGENTIC-SAAS | P2 | HITL queue backlog with a rubber-stamp approval pattern (>95% approved, sub-baseline median review time) | triage mandatory vs anomaly-escalation queues; temporary confidence-floor raise; add reviewer capacity |
| R10 | [Memory / context poisoning](#r10--memory--context-poisoning) | AGENTIC-SAAS | P1 | `camel.output_audit_failed` spike or a `connector_drift_anomaly` tied to one tenant/session | agent halt ≤60 s; `admin:agent-context-audit`; isolate the suspect connector |
| R11 | [Coordination overhead](#r11--coordination-overhead) | AGENTIC-SAAS | P2 | assessment p95 latency regression traced to worker-to-worker handoff, with GOV-010 loop counters within limits | `admin:workflow-trace`; batch/cache redundant MCP calls across subagent handoffs |
| R12 | [Prompt brittleness](#r12--prompt-brittleness) | AGENTIC-SAAS | P2 | golden-set regression >2% (NFR-008) traced to a prompt/schema edit, not a provider EOL migration | `admin:prompt-pin --rollback`; re-run golden set; re-baseline the hash after sign-off |

---

### 4.3 · R1 — Cross-tenant context leak

The worst-case failure in a multi-tenant agentic system: one tenant's data surfaces inside another's session. It's the only runbook that reaches for the global L4 halt as its first automation gate.

> **Severity:** P0-C · **Type:** COMPOSITE

**Trigger:** `DuxCrossTenantContextDetected` (`foreign_tenant_refs > 0` + isolation SLO burn ≥5%/1h), or a fuzz failure.

**§8 Automation gate:** `admin:platform-contain --confirm`, 60 s. On timeout, page the CTO and the Founder.

**§9 Execution steps:**

1. Discover active agents.
2. `admin:agent-halt --id $ID --confirm` — within 60 s.
3. `admin:agent-context-audit` — must read `foreign_tenant_refs: 0` after the fix. **This scan is post-hoc, not real-time** — it runs on demand against logged context, not inline on every request — a known detection-latency tradeoff; a leak can persist between occurrence and the next audit run.
4. `admin:export-session` — evidence to MinIO.
5. **Engage counsel if PII is involved.**
6. Root-cause fix, then `test:isolation` — every ISO case must pass.
7. PM approves the customer notification.
8. `admin:platform-contain --release`.

**§3 Business impact:** `monthly_MRR × (affected ÷ total) × (hours ÷ 730)`.

> **Regulatory: GDPR Art. 33 and DORA Delegated Regulation 2025/301 Art. 5 — 72 hours if PII is involved.**

**§11 Verification:** `sum(foreign_tenant_refs) == 0`; `rate(dux_assessment_errors[15m]) < 0.01`; API p95 <0.3 s.

---

### 4.4 · R2 — Token cost runaway

An agent stuck in a loop can burn a tenant's budget in minutes. This is the same three-tier cost evaluation order from [[Dux AI Safety Guide]], now as an incident procedure.

> **Severity:** P0-C / P1 · **Type:** COMPOSITE

**Trigger:** spend >3× the 7-day baseline, or >$25/hour/tenant.

Cost evaluation order (D-3): **$0.675/assessment breaker → $25/hour CostCap → 2× baseline.**

**§9 Execution steps:**

1. `admin:token-spend`.
2. `admin:cost-cap enforce --max-usd-per-hour 25`.
3. L2 kill switch.
4. Discover and halt the top-spend agent.
5. Export the session.
6. Fix the configuration — iteration limit, or model downgrade.
7. Verify spend is below 1.5× baseline **before** releasing L2.

---

### 4.5 · R3 — Model provider outage

When the upstream model provider goes down, the assessment pipeline has to degrade gracefully rather than silently fail.

> **Severity:** P1 · **Type:** COMPOSITE

**Trigger:** OpenAI status ≠ operational, or `DuxLLMAvailabilityFastBurn`.

The fallback model is `claude-sonnet-4-6` (+28% latency).

**§9 Execution steps:**

1. Confirm the outage.
2. `admin:model-route --fallback on` — within 60 s.
3. PM sets the status page to `degraded_performance`.
4. Run the golden set against the fallback. **Halt if regression exceeds 5%.**
5. Verify the queue is draining.
6. `--fallback off` once the primary is restored.

**Low-traffic guard:** force HITL T3 on fallback verdicts until a spot check of ≥5 assessments per 15 min passes.

---

### 4.6 · R3b — Model EOL / forced deprecation

Not an incident so much as a slow-motion inevitability that deserves its own runbook rather than getting handled ad hoc.

> Gate-2 hardening (H6). Planned work, not incident-triggered.

**Provider models are EOL-dated dependencies.** In 2026, ChatGPT-surface models have been retired on two weeks' notice; the API minimum is 6 months GA and **3 months for variants** — and the S-LLM workhorse is a variant.

A forced migration shifts the golden-set baseline, and **schema-parity checks alone will not catch quality drift**: "Drift or Dice" (2026) documents a schema gate reporting zero regressions while output shrank 15.7%, and a tier failed 25% of hard tasks.

**Procedure:**

1. Subscribe to provider deprecation feeds; review weekly.
2. On an EOL notice, run the migration eval protocol — the golden set **plus** output-length, latency, and tier-failure deltas. Never schema parity alone.
3. Pin the successor via `models.json` and the CI pin gate.
4. Keep a pinned fallback, so a forced sunset cannot take the assessment path down.
5. Re-baseline the golden-set hash after sign-off.

---

### 4.7 · R4 — MCP dependency failure

A dead MCP server is deceptively quiet — nothing crashes, results just stop showing up.

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** MCP errors >50% over 5 min, or a health-check failure.

**The risk is a silent stall** — assessments appear to be running but produce no results.

**§9 Execution steps:**

1. `admin:mcp-circuit-breaker --open --server $ID`.
2. **Verify the audit shows `tool_unavailable`, not a hallucinated success.**
3. Notify tenants if the outage exceeds 1 hour.
4. Half-open: test a single tool.
5. `--close` once the error rate is below 5%.

---

### 4.8 · R5 — Rate limit cascade

Two rate limiters — the model provider's and Dux's own — can trip in the same window and compound each other.

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** `DuxRateLimitCascade` — model 429 and customer 429 in the same window. The signature is a **dual 429** — OpenAI (60 RPM/tenant) and the NestJS throttler in the same window.

**§9 Execution steps:**

1. Confirm the dual 429.
2. `admin:rate-limit-backoff --jitter on`.
3. Discover the runaway agent; optionally L1-halt it.
4. Verify spend is within cap.
5. Notify if the stall exceeds 1 hour.
6. Reset the backoff.

**Seed depth:** dynamic throttle at `max(1, floor(baseline × 0.5))`.

---

### 4.9 · R6 — Context window exhaustion

A large tenant's asset count can run the agent straight into the 128 K ceiling — this is how it fails without losing the work already done.

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** `DuxContextWindowExhausted` (128 K).

The ceiling is 128 K, with L1–L3 compression selected by asset count.

**§9 Execution steps:**

1. `admin:agent-halt`.
2. `admin:export-session`.
3. Checkpoint or abandon, per governance limits.
4. Root-cause: context configuration, or asset count.
5. `admin:platform-contain --tenant` if a retry loop is running.
6. Resume: `test:e2e --flow assessment`.

**Checkpoint at 80%** into `assessment_checkpoints` (JSONB). **Abandon at 100%** (`abandoned_with_notify`). Maximum 3 resume attempts.

---

### 4.10 · R7 — Prompt cache invalidation

A quiet deploy or a pin change can wreck the cache hit rate and quietly inflate cost and latency together.

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** `DuxPromptCacheHitRateDrop` (>15% over 5 min).

**§9 Execution steps:**

1. Confirm the hit-rate drop exceeds 15% against the 7-day baseline.
2. Identify the cause — a deploy, or a pin change.
3. If deploy-caused, evaluate a rollback.
4. `admin:llm-cache-warm --env production` — IC-approved, after staging parity.
5. Verify the hit rate is within 10% of the 7-day baseline.
6. PM updates the status page if p95 stays above 2× for more than 1 hour.

---

### 4.11 · R8 — Hallucinated-CVE citation

This is the failure mode LLM09 exists to catch (see [[#3.3 · LLM risks — LLM01–LLM10]]) — a citation that looks legitimate but doesn't actually check out against NVD.

> **Severity:** P1 · **Type:** AGENTIC-SAAS

**Trigger:** EXP-CIT-001 fails at generation time — a citation's CVE ID doesn't resolve against NVD, or its CVSS/description diverges from the resolved record.

**§9 Execution steps:**

1. `admin:citation-audit --assessment $ID` — confirm which `exploitable`/`likely` claim's citation failed NVD resolution, or diverged on CVSS/description, per the extended EXP-CIT-001 check ([owasp-assessments §2](owasp-assessments.md)).
2. `admin:verdict-quarantine --assessment $ID` — hide the exposure verdict from the customer surface pending manual verification.
3. Manually re-verify against NVD. If the CVE genuinely doesn't resolve, downgrade confidence and re-run the citation check.
4. Root-cause: model hallucination vs. a stale NVD cache vs. an upstream NVD outage (cross-reference the [NVD fallback priority](../10-product/catalogs.md#nvd-enrichment-fallback-priority)).
5. `pnpm test:exposure-citations` green before re-enabling the verdict.

**§11 Verification:** all `exploitable`/`likely` claims on the affected assessment show `nvd_resolved: true`; zero repeat EXP-CIT-001 failures for the tenant over 24 h.

---

### 4.12 · R9 — Alert fatigue

A HITL queue that rubber-stamps everything isn't providing review — it's providing theater. This runbook is about telling that apart from a genuine surge.

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** HITL queue backlog with a rubber-stamp approval pattern (>95% approved, sub-baseline median review time).

**§9 Execution steps:**

1. `admin:hitl-queue-depth` — confirm a sustained backlog or a rubber-stamp pattern (>95% approved, median review time below baseline).
2. Triage: separate the 2 mandatory-HITL actions (`endpoint.isolate`, `patch.deploy_special_devices`) from the 3 anomaly-escalation-only actions — mandatory items are never paused or floor-adjusted.
3. `admin:confidence-floor --raise` — a time-boxed, logged increase on the anomaly-escalation actions only, to cut marginal-confidence escalations.
4. Add reviewer capacity or redistribute load across the HITL roster.
5. Root-cause: confidence-floor miscalibration vs. a genuine incident surge driving real escalations.
6. Revert the raised floor once median time-to-decision is back under target.

**§11 Verification:** HITL median time-to-decision under the target baseline; approval rate back below the rubber-stamp threshold.

---

### 4.13 · R10 — Memory / context poisoning

This is CaMeL's own detection surface (`camel.output_audit_failed`, `connector_drift_anomaly`) turning into an active containment procedure.

> **Severity:** P1 · **Type:** AGENTIC-SAAS

**Trigger:** `camel.output_audit_failed` spike or a `connector_drift_anomaly` tied to one tenant/session.

**§9 Execution steps:**

1. `admin:agent-halt --id $ID` — within 60 s.
2. `admin:agent-context-audit` — inspect the `sources[].content_hash` provenance chain for the poisoned span ([camel-plane §3](camel-plane.md)).
3. `admin:mcp-circuit-breaker --open --server $ID` on the suspect connector or research source.
4. Cross-check: confirm `getAssetsForCVE` / native `query_controls` disagree with the poisoned claim (CaMeL CDA #3 and #5, [camel-plane §4](camel-plane.md)).
5. Purge World Model rows sourced from the poisoned span; re-sync clean.
6. `pnpm test:camel-benchmark` green before resuming the connector.

**§11 Verification:** `connector_drift_anomaly` count returns to 0; no repeat `camel.output_audit_failed` for the tenant over 24 h.

---

### 4.14 · R11 — Coordination overhead

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** assessment p95 latency regression traced to worker-to-worker handoff, with GOV-010 loop counters within limits.

**§9 Execution steps:**

1. `admin:workflow-trace --assessment $ID` — break latency down by workflow subagent (`prerequisite-extractor`, `asset-context-worker`, `control-mapping-worker`).
2. Confirm GOV-010 loop counters are within limits — this rules out a runaway loop, a distinct failure mode from handoff overhead.
3. Identify redundant or serial MCP calls (`query_assets`/`query_controls` chains) that should batch or parallelize.
4. Patch: batch calls, or cache `summarizeContext` output across subagent handoffs.
5. Re-run the golden set to check for a regression.
6. Verify p95 assessment duration is back within the governance latency budget ([governance-kernel §3](governance-kernel.md)).

**§11 Verification:** p95 assessment duration within budget; MCP call count per assessment not elevated versus baseline.

---

### 4.15 · R12 — Prompt brittleness

> **Severity:** P2 · **Type:** AGENTIC-SAAS

**Trigger:** golden-set regression >2% (NFR-008) traced to a prompt/schema edit, not a provider EOL migration.

**§9 Execution steps:**

1. Confirm the golden-set regression traces to a prompt or schema edit — not a provider EOL migration ([R3b](#r3b--model-eol--forced-deprecation)) and not a hallucinated citation ([R8](#r8--hallucinated-cve-citation)).
2. `admin:prompt-pin --rollback` — to the last golden-set-passing version.
3. Re-run the golden set; confirm the regression drops under the 2% NFR-008 threshold.
4. Root-cause: an overly narrow enum or wording that fails on edge-case CVE phrasing.
5. Stage the fix behind a shadow run before re-promoting.
6. Re-baseline the golden-set hash after sign-off.

**§11 Verification:** golden-set regression under 2%; no repeat failure over 3 consecutive assessment runs.

---

### 4.16 · Runbook map

The twelve runbooks above are canonical. Seed adds two more in [seed runbooks](../60-operations/runbooks.md): **agent quota exhaustion** (seed-only) and **shadow AI**. The seed routing matrix lives there too.

| Category | Runbooks |
|----------|----------|
| COMPOSITE (cross-system) | R1, R2, R3 |
| AGENTIC-SAAS | R4, R5, R6, R7, R8, R9, R10, R11, R12 |

---

## Sources

- `.raw/dux/40-ai-safety/agent-identity.md`
- `.raw/dux/40-ai-safety/confidence-calibration.md`
- `.raw/dux/40-ai-safety/owasp-assessments.md`
- `.raw/dux/40-ai-safety/incident-runbooks.md`

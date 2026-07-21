---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [H8]
---

# Taxonomy, Enums & Design System

**Purpose:** the controlled vocabulary — exploitability labels, entity types, factor cards, naming rules, design tokens, and error codes.

**Parents:** BR-002, BR-011 · **Authority:** OpenAPI 3.1 owns the public enums; this document owns the UI and application projections.

## 1. Layered exploitability model

Three layers, projected in one direction only — application → UI → public API:

| Layer | Values | Where used |
|-------|--------|-----------|
| Application assessment | `exploitable` \| `likely` \| `unlikely` \| `not_exploitable` \| `insufficient_data`, plus `confidence_score` | `AssessmentDto`, US-011, abstention |
| UI exposure states | `Protected` \| `PartiallyMitigated` \| `MitigationRequired` \| `Exposed` | US-010, US-006, design system |
| Public API (`exploitability_status`) | `potentially_exploitable` \| `not_exploitable` \| `partially_mitigated` \| `null` | `GET /v1/vulnerability-instances/{cve_id}` |

**Confidence bands** — lower bound inclusive, upper exclusive:

| Band | Label |
|------|-------|
| [0.85, 1.00] | `exploitable` |
| [0.70, 0.85) | `likely` |
| [0.40, 0.70) | `unlikely` |
| [0.00, 0.40) | `not_exploitable` |
| abstain | `insufficient_data` |

**Application → public projection:**

| Assessment label | Confidence band | Public `exploitability_status` |
|------------------|-----------------|-------------------------------|
| `exploitable` | [0.85, 1.00] | `potentially_exploitable` |
| `likely` | [0.70, 0.85) | `potentially_exploitable` |
| `unlikely` | [0.40, 0.70) | `partially_mitigated` |
| `not_exploitable` | [0.00, 0.40) | `not_exploitable` |
| `insufficient_data` | abstain | `null` |
| CVE not researched | — | `null` |

**`confidence_score` is not exposed on public v1.** `insufficient_data_reason` (`asset_gap` \| `intel_gap` \| `context_limit`) is application and UI only.

**Network exposure verdict** (`network_exposure`): `internet` \| `external` \| `internal` \| `unreachable`. Nullable.

**UI label aliases.** Figma "Exploitable" → the Mitigation Required subset (`potentially_exploitable`). "Not Exploitable" / "Unexploitable" → Protected (`not_exploitable`). "Unresearched" → `null`.

### Confidence scoring methodology

**This is confidence-in-exploitability, not a composite risk score.** Dux does not compute a CVSS × EPSS × criticality × exposure formula, an asset-criticality multiplier, or a config-drift baseline comparison — there is no DuxScore. What produces `confidence_score` is a three-signal ensemble, specified in full in [confidence-calibration](../40-ai-safety/confidence-calibration.md):

| Signal | Weight | When available |
|--------|--------|----------------|
| Mean top-1 logprob, over claim-bearing tokens | 0.40 | when the API exposes logprobs |
| Semantic entropy, over meaning-clustered completions | 0.35 | always |
| Verbalized confidence, from structured output | 0.25 | always |

When logprobs are unavailable, the remaining two signals renormalize to **entropy 0.54 / verbalized 0.46**. The ensemble output feeds Platt scaling, which produces the calibrated `confidence_score` that the bands above threshold against. The `CalibrationRecord` behind that scaling (`model_version`, `platt_params`, `brier_score`, `ece`) is the global `CALIBRATION_RECORD` entity in [data-model](../20-architecture/data-model.md).

### Evidence freshness caps confidence (H8, Gate-2 hardening)

**A stale connector does not fail loudly — it feeds stale evidence, which produces a wrong verdict. That is a safety defect, not a data-quality nit.**

Every connector carries a freshness SLO, measured as the age of its last successful sync (the CONN-001 pattern). Evidence past its freshness window sets a **degraded-evidence flag** on the affected verdicts, and **degraded evidence cannot support a high-confidence band**: an `exploitable` or `likely` verdict is downgraded, or becomes `insufficient_data` with reason `intel_gap` or `asset_gap`.

Maintenance is tiered: **W1 connectors get nightly contract tests against vendor sandboxes**, not just fixtures. The long tail gets fixture tests plus a staleness alarm.

## 2. EntityType

The closed set used by DQL and custom metrics:

`device` · `cloud_compute` · `finding` · `vulnerability_instance` · `cve` · `mitigation` · `label` · `user`

| Gate | Types |
|------|-------|
| Gate 1 | `device`, `cloud_compute`, `finding`, `vulnerability_instance`, `cve` |
| Gate 2c+ | `mitigation`, `label`, `user` |

## 3. World Model evidence types

`ASSET` · `FINDING` · `CONTROL` · `OWNERSHIP_EVIDENCE` · `EXPLOITABILITY_ASSESSMENT` · `ASSESSMENT_REASONING_STEP` · `ATTACK_PATH` · `CONTROL_MAPPING`

Versioned through `world_model_versions` (ADR-003). **A new evidence type extends [catalogs](catalogs.md) before it extends any connector annex.**

## 4. Mitigation factor cards (US-011)

Figma display titles map to a canonical `factor_type`:

| Display title | `factor_type` | Source | Gate |
|---------------|--------------|--------|------|
| AWS Security Group Blocks Port | `aws_sg_blocks_port` | `aws` | Gate 1 |
| Product Not Affected | `product_not_affected` | assessment logic | Gate 1 |
| Network Reachability | `network_reachability` | `aws` + multi-connector | Gate 1, partial |
| Firewall Blocks Exploitation | `firewall_blocks_exploitation` | `crowdstrike` | Gate 1 (connector live) |
| Process Not Listening On Ports | `process_not_listening` | `dux-resident-agent` | Gate 5 |

### Control settings

`CONTROL.settings` (JSONB — see [data-model](../20-architecture/data-model.md)) is a free-form bag today. The UI surfaces specific named settings as the evidence behind a Protected or blocked verdict. **These are the canonical values; treat them as a controlled vocabulary going forward:**

| Setting | Meaning | Example attribution |
|---------|---------|---------------------|
| `InboundPortBlock` | blocks external traffic to the affected service port | "Source: CrowdStrike Firewall" |
| `RestrictedIPAllowlist` | allows access only from approved IP ranges | — |
| `DefaultDenyExternalAccess` | denies all unsolicited external connections by default | — |

These sit **underneath** the factor cards above — they are the rule detail shown when a factor card is expanded. They do not replace the factor-card model.

### Reachable vs breachable

The UI frames exploitability as **reachable** (an attacker has a network path) versus **breachable** (reachable *and* prerequisites met *and* no blocking control).

This is the same distinction the corpus already models — the `network_exposure` verdict (§1), the four-state exposure model, and the mitigation factor cards — expressed in customer-facing words. **Use "reachable vs breachable" in UI copy and GTM material; `network_exposure` and the exposure states remain the internal and API vocabulary. No schema change is implied.**

**Relationship Graph Engine** is the customer-facing name for the existing vulnerability ↔ asset ↔ control mapping capability, backed by `ATTACK_PATH` and `CONTROL_MAPPING` (§3) and the graph query model in [data-model](../20-architecture/data-model.md). **It names something already built. It is not a new build.**

**Today this is single-hop mapping only** — one vulnerability to one asset to one control, not multi-hop lateral-movement kill-chain traversal. Multi-hop attack-path graph traversal is roadmap, not scoped for Phase 1.

## 5. Naming glossary

| Canonical | Deprecated / internal alias | Rule |
|-----------|----------------------------|------|
| **Dux Agent** | Dux AI, AI-workers, Mitigation/Remediation agent | The only customer-facing agent name. Code id: `dux-agent` |
| assessment agent | — | Internal orchestration role only — ADR, TRD, workflow. **Never** in marketing or compliance scope |
| **Mitigation nav** | — | The research queue — the **Analyze** stage (US-010) |
| **Mitigate stage** | — | Lightweight-mitigation automation (`mitigation_stage`) |
| Exposure Analysis | Exposure drill-down | Nav label vs page name |
| **kill switch** (noun) | kill-switch (adjective only) | Levels KS-L1–L4 |
| CaMeL | camel-plane | The dual-LLM boundary |
| **World Model** | world model | A proper noun — it is a versioned artifact |
| tenant | organization (UI only) | API and code always say `tenant_id` |
| exploitability assessment | assessment (UI shorthand) | Formal docs and US-011 |

**"Mitigation nav" and "Mitigate stage" are different things.** The nav is the Analyze-stage research queue. The stage is mitigation automation. Conflating them is the most common naming error in this corpus.

## 6. Design system

**Exposure-state colors:**

| Token | Hex |
|-------|-----|
| Protected / Unexploitable | `#22C55E` |
| Partially Mitigated | `#F59E0B` |
| Mitigation Required | `#DC2626` |
| Exposed | `#EF4444` |
| Listening / Active | `#F472B6` |
| Primary accent | `#8B5CF6` |
| Actions CTA | `#EC4899` |
| Page background | `#F3F4F6` |
| Card background | `#FFFFFF` |

**Stage pills.** Exploitability Analysis — pink (`#EC4899` family). Lightweight Mitigations — tan/brown. Remediation Acceleration — blue.

**Iconography.** Color **and** shape, per WCAG 2.2 SC 1.4.1 — color alone never carries meaning:

| Icon | State |
|------|-------|
| Solid umbrella | Protected |
| Umbrella + rain | Partially Mitigated |
| Crossed circle | Mitigation Required |
| Crossed circle + exclamation, pulsing | Exposed |
| Signal waves | Listening |
| Brain / lightbulb | AI reasoning |
| Lightning | Quick action |
| D / shield — pink active, grey inactive | Dux Agent |

**SVG with ARIA labels. Never emoji.**

**Typography.** Inter or system. Headline 24/700; card title 16/600; metric 32/700 in the state color; body 14/400. Spacing on a 4 px base — card padding 24, gap 16, section 32.

**Contrast audit (WCAG 2.2 SC 1.4.3):**

| Color | Ratio | Verdict |
|-------|-------|---------|
| `#22C55E` | 7.2:1 | pass |
| `#DC2626` / `#EF4444` | 4.6:1 / 4.5:1 | pass |
| **`#F59E0B`** | **2.4:1** | **fails** — use icon and shape only, or darken to `#B45309` |
| `#8B5CF6` on gray | 3.8:1 | large text only |
| `#8B5CF6` on white | 4.6:1 | pass |

**AI worker status messages** stream on screens 1–9, with agent states `REASONING` / `TOOL_CALLING` / `EVALUATING` / `COMPLETE`. **Each line carries a non-color indicator** — a spinner or an icon. These states are emitted by the Temporal workflow via NATS SSE fan-out, not by a framework intermediary (ADR-021) — the dashboard receives raw `AssessmentDelta` events from `ExploitabilityAssessmentWorkflow`.

## 7. Error taxonomy

`DuxErrorCode`, shared across REST, SSE, and webhooks:

| Code | HTTP |
|------|------|
| `AGENT_TIMEOUT` | 504 |
| `CONTEXT_EXHAUSTED` | 422 |
| `BUDGET_EXCEEDED` | 429 |
| `GOVERNANCE_BLOCKED` | 403 |
| `INSUFFICIENT_DATA` | 422 — subtypes `asset_gap`, `intel_gap`, `context_limit` |

Application error classes: `TenantIsolationError` (403), `ConnectorSyncError` (502), `AgentBudgetExceeded` (429), `AssessmentDedupConflict` (409).

A workflow abandons at 15 minutes. Context checkpoints at ≥80% depth, and resumes via `POST /assessments/{id}/resume`.

## 8. Platform edge cases

| Scenario | Expected behavior | Test |
|----------|-------------------|------|
| NVD unavailable >2 h | serve from S3 cache; warn `NVD_SYNC_WARN` at >2 h, critical `NVD_SYNC_STALE` at >4 h | CONN-001 |
| AWS throttling | exponential backoff + jitter; max 5 retries; `AWS_SYNC_FAILURE` P2 | CONN-002 |
| Workflow action storm | continue-as-new at 35 K events; max 2 retries | TEMP-001 |
| Cross-tenant URL manipulation | **404, to mask existence.** More than 20 masked 404s per IP per 5 min → SIEM | ISO-001 |
| Webhook delivery failure | 5 attempts, then dead-letter; alert above 10 per tenant per hour | WH-001 |
| Rate limit exceeded | 429 (RFC 6585) + `Retry-After` (RFC 9110); admin notified if the queue stalls >5 min | API-001 |
| Context window exhaustion | checkpoint into `assessment_checkpoints` (JSONB); resume endpoint; abandon with `CONTEXT_EXHAUSTED` at ≥80% depth | AGENT-001 |

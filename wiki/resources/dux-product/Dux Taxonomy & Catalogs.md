---
type: resource
title: "Dux Taxonomy & Catalogs"
address: c-000017
topic: "dux/product"
created: 2026-07-22
updated: 2026-07-23
tags: [resource, dux, dux/product]
status: mature
sources: [".raw/dux/10-product/taxonomy.md", ".raw/dux/10-product/catalogs.md"]
related: ["[[Dux]]", "[[Dux Product Guide]]", "[[Dux Feature Reference]]", "[[Dux AI Safety Guide]]", "[[Dux Architecture Guide]]"]
---

# The Controlled Vocabulary Dux Speaks

### How one word means the same thing everywhere in the product — and what happens the moment it doesn't

Navigation: [[Dux]] | [[Dux Product Guide]] | [[Dux Feature Reference]]

---

## Why a vocabulary page earns its own place in the corpus

Every non-trivial product eventually hits the same failure mode: the word "Exploitable" means one thing in the database, something slightly different in the UI copy, and something else again on the public API — and nobody notices until a customer's dashboard disagrees with their own webhook payload. Dux fights that failure mode with two registries that answer the same underlying question from opposite directions. The **taxonomy** fixes what every label and enum *means*. The **catalogs** fix what every extension point *is*. Between them, they're the controlled vocabulary the entire product speaks — and this page holds both, because splitting them across two files would just recreate the drift they exist to prevent. OpenAPI 3.1 owns the public wire enums; this page owns the UI and application-layer projections onto them.

**Parents:** BR-002, BR-011 · **Authority:** OpenAPI 3.1 owns the public enums; this document owns the UI and application projections.

---

## 1. One exploitability model, three layers, one direction of travel

The single most important discipline on this page is that exploitability data flows in **one direction only** — application → UI → public API, never the reverse. That's not an implementation detail; it's what keeps a public API consumer from ever seeing more granularity than a UI user, and what keeps the UI from ever showing more than the application layer actually computed.

| Layer | Values | Where used |
|---|---|---|
| Application assessment | `exploitable` · `likely` · `unlikely` · `not_exploitable` · `insufficient_data`, plus `confidence_score` | `AssessmentDto`, US-011, abstention |
| UI exposure states | `Protected` · `PartiallyMitigated` · `MitigationRequired` · `Exposed` | US-010, US-006, design system |
| Public API (`exploitability_status`) | `potentially_exploitable` · `not_exploitable` · `partially_mitigated` · `null` | `GET /v1/vulnerability-instances/{cve_id}` |

### Confidence bands

The application layer's five-way label isn't arbitrary — it's a direct read of a numeric confidence score, bucketed with the lower bound inclusive and the upper bound exclusive:

| Band | Label |
|------|-------|
| [0.85, 1.00] | `exploitable` |
| [0.70, 0.85) | `likely` |
| [0.40, 0.70) | `unlikely` |
| [0.00, 0.40) | `not_exploitable` |
| abstain | `insufficient_data` |

### How the internal label becomes the public one

Nothing on the public API surface is computed independently — every value there is a deliberate, coarser projection of the internal assessment label:

| Assessment label | Confidence band | Public `exploitability_status` |
|------------------|-----------------|-------------------------------|
| `exploitable` | [0.85, 1.00] | `potentially_exploitable` |
| `likely` | [0.70, 0.85) | `potentially_exploitable` |
| `unlikely` | [0.40, 0.70) | `partially_mitigated` |
| `not_exploitable` | [0.00, 0.40) | `not_exploitable` |
| `insufficient_data` | abstain | `null` |
| CVE not researched | — | `null` |

`confidence_score` is **not** exposed on public v1 — that granularity stays internal by design. `insufficient_data_reason` (`asset_gap` · `intel_gap` · `context_limit`) is application and UI only, never public.

One more verdict rounds out the model: **network exposure** (`network_exposure`) is `internet` · `external` · `internal` · `unreachable`, and it's nullable.

And because design files don't always speak the same language as engineering specs, there's a standing alias table: Figma's "Exploitable" label maps to the Mitigation Required subset (`potentially_exploitable`); "Not Exploitable" / "Unexploitable" maps to Protected (`not_exploitable`); "Unresearched" maps to `null`.

---

## 2. What actually produces a confidence score

It's worth being explicit about what this system is *not* doing, because the instinct in security tooling is almost always to reach for a composite formula. **This is confidence-in-exploitability, not a composite risk score.** Dux does not compute a CVSS × EPSS × criticality × exposure formula, an asset-criticality multiplier, or a config-drift baseline comparison — there is no DuxScore. What actually produces `confidence_score` is a three-signal ensemble, specified in full in [[Dux AI Safety Guide]]:

| Signal | Weight | When available |
|--------|--------|----------------|
| Mean top-1 logprob, over claim-bearing tokens | 0.40 | when the API exposes logprobs |
| Semantic entropy, over meaning-clustered completions | 0.35 | always |
| Verbalized confidence, from structured output | 0.25 | always |

### When a signal goes missing

Not every model API exposes logprobs, so the ensemble has a defined fallback rather than silently degrading: when logprobs are unavailable, the remaining two signals renormalize to **entropy 0.54 / verbalized 0.46**. The ensemble output then feeds **Platt scaling**, which produces the calibrated `confidence_score` that the bands in §1 threshold against.

### The record behind the scaling

The `CalibrationRecord` behind that scaling is the global `CALIBRATION_RECORD` entity in [[Dux Architecture Guide]], and it carries exactly four fields:

| Field | Purpose |
|-------|---------|
| `model_version` | Pins the model that produced the ensemble |
| `platt_params` | Platt scaling coefficients |
| `brier_score` | Overall calibration quality |
| `ece` | Expected calibration error |

---

## 3. Why a stale connector is a safety defect, not a data-quality nit

It's tempting to treat connector freshness as an operational metric — something for a status page, not a spec. Dux treats it as a first-class safety concern instead, and the reasoning is worth stating plainly: **a stale connector does not fail loudly — it feeds stale evidence, which produces a wrong verdict.** That's the H8 finding, and it's tracked as Gate-2 hardening rather than a nice-to-have.

Every connector carries a freshness SLO, measured as the age of its last successful sync (the CONN-001 pattern). Evidence past its freshness window sets a **degraded-evidence flag** on the affected verdicts, and — this is the enforcement mechanism that makes the whole thing matter — **degraded evidence cannot support a high-confidence band**: an `exploitable` or `likely` verdict gets downgraded, or becomes `insufficient_data` with reason `intel_gap` or `asset_gap`.

Maintenance itself is tiered by how load-bearing a connector is:

| Tier | Treatment |
|------|-----------|
| W1 connectors | Nightly contract tests against vendor sandboxes, not just fixtures |
| Long-tail connectors | Fixture tests plus a staleness alarm |

---

## 4. EntityType: the closed set behind DQL and custom metrics

Every entity a tenant can query through DQL or build a custom metric against belongs to one of eight closed types, and those eight types ship in two waves rather than all at once:

| Gate | Types |
|------|-------|
| Gate 1 | `device`, `cloud_compute`, `finding`, `vulnerability_instance`, `cve` |
| Gate 2c+ | `mitigation`, `label`, `user` |

---

## 5. The World Model, and the evidence it's built from

`World Model` is a proper noun — a versioned artifact (`world_model_versions`, ADR-003), never written lowercase. It's the evidence substrate every Dux Agent assessment reasons over, and it's built from exactly eight evidence types:

`ASSET` · `FINDING` · `CONTROL` · `OWNERSHIP_EVIDENCE` · `EXPLOITABILITY_ASSESSMENT` · `ASSESSMENT_REASONING_STEP` · `ATTACK_PATH` · `CONTROL_MAPPING`

The ordering rule matters here too: a new evidence type extends the catalogs registry **before** it extends any connector annex. Evidence typing is corpus-governed, not something a connector integration is allowed to define locally — otherwise the same drift problem this whole page exists to prevent would just re-open through the back door.

Agentic RAG retrieval runs against the World Model's graph projection (Apache AGE) and vector store (pgvector) in parallel with episodic memory and live threat-intel APIs — the full retrieval architecture lives in [[Dux Architecture Guide]].

---

## 6. Mitigation factor cards: the evidence behind a Protected verdict

When a UI shows a customer *why* something is protected rather than just asserting that it is, that explanation comes from a factor card. Each Figma display title maps to exactly one canonical `factor_type`:

| Display title | `factor_type` | Source | Gate |
|---------------|--------------|--------|------|
| AWS Security Group Blocks Port | `aws_sg_blocks_port` | `aws` | Gate 1 |
| Product Not Affected | `product_not_affected` | assessment logic | Gate 1 |
| Network Reachability | `network_reachability` | `aws` + multi-connector | Gate 1, partial |
| Firewall Blocks Exploitation | `firewall_blocks_exploitation` | `crowdstrike` | Gate 1 (connector live) |
| Process Not Listening On Ports | `process_not_listening` | `dux-resident-agent` | Gate 5 |

**Figma's "playbook cards" are these same factor cards under a design-file name, never a separate entity** — a naming trap worth flagging explicitly, because it's exactly the kind of drift this whole page exists to prevent.

### One layer underneath: control settings

`CONTROL.settings` (JSONB — see [[Dux Architecture Guide]]) is a free-form bag today, but the UI surfaces a specific set of named settings as the evidence behind a Protected or blocked verdict. These three are canonical; treat them as a controlled vocabulary going forward:

| Setting | Meaning | Example attribution |
|---------|---------|---------------------|
| `InboundPortBlock` | blocks external traffic to the affected service port | "Source: CrowdStrike Firewall" |
| `RestrictedIPAllowlist` | allows access only from approved IP ranges | — |
| `DefaultDenyExternalAccess` | denies all unsolicited external connections by default | — |

These sit **underneath** the factor cards above — they're the rule detail shown when a factor card is expanded, not a replacement for the factor-card model.

---

## 7. Reachable vs breachable: the customer-facing translation

Engineers and the UI need a way to talk about exploitability that doesn't require explaining confidence bands to a CISO mid-demo. The answer is a simple two-word frame: **reachable** (an attacker has a network path) versus **breachable** (reachable *and* prerequisites met *and* no blocking control).

This isn't a new model bolted on top — it's the same distinction the corpus already computes, expressed in customer-facing words: the `network_exposure` verdict from §1, the four-state exposure model, and the mitigation factor cards above. **Use "reachable vs breachable" in UI copy and GTM material; `network_exposure` and the exposure states remain the internal and API vocabulary. No schema change is implied.**

The **Relationship Graph Engine** name deserves the same clarification. It's the customer-facing name for the existing vulnerability ↔ asset ↔ control mapping capability, backed by `ATTACK_PATH` and `CONTROL_MAPPING` (§5) and the graph query model in [[Dux Architecture Guide]]. **It names something already built. It is not a new build.** And it's worth being precise about its current limits: **today this is single-hop mapping only** — one vulnerability to one asset to one control, not multi-hop lateral-movement kill-chain traversal. Multi-hop attack-path graph traversal is roadmap, not scoped for Phase 1.

---

## 8. The naming glossary — where most terminology bugs actually come from

If there's one table on this page worth memorizing, it's this one. Roughly half of every cross-file terminology bug found during the corpus's own review traces back to violating one of these ten rules:

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

The single most common naming error in this corpus deserves its own sentence, not just a table row: **"Mitigation nav" and "Mitigate stage" are different things.** The nav is the Analyze-stage research queue. The stage is mitigation automation. Conflating them is the error this whole glossary exists to catch.

---

## 9. The design system: color, shape, type, and motion

A design system is its own kind of controlled vocabulary — a color means a state, an icon means a state, and neither is allowed to mean two things.

### Exposure-state colors

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

**Stage pills** carry their own distinct palette from the exposure states above: Exploitability Analysis is pink (`#EC4899` family), Lightweight Mitigations is tan/brown, and Remediation Acceleration is blue.

### Iconography — never color alone

Per WCAG 2.2 SC 1.4.1, color alone is never allowed to carry meaning — every state pairs a color with a distinct shape:

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

### Typography

Inter or system font. Headline is 24px/700 weight; card title 16/600; metric 32/700 in the state color; body 14/400. Spacing runs on a 4px base — card padding 24, gap 16, section 32.

### Contrast audit (WCAG 2.2 SC 1.4.3)

Not every color in the palette above actually passes contrast on its own — this audit is why the iconography rule above isn't optional:

| Color | Ratio | Verdict |
|-------|-------|---------|
| `#22C55E` | 7.2:1 | pass |
| `#DC2626` / `#EF4444` | 4.6:1 / 4.5:1 | pass |
| **`#F59E0B`** | **2.4:1** | **fails** — use icon and shape only, or darken to `#B45309` |
| `#8B5CF6` on gray | 3.8:1 | large text only |
| `#8B5CF6` on white | 4.6:1 | pass |

### How the agent narrates its own reasoning

AI worker status messages stream on screens 1–9, cycling through agent states `REASONING` / `TOOL_CALLING` / `EVALUATING` / `COMPLETE`. Each line carries a **non-color indicator** — a spinner or an icon — consistent with the contrast-failure lesson above. These states are emitted by the Temporal workflow via NATS SSE fan-out, not by a framework intermediary (ADR-021) — the dashboard receives raw `AssessmentDelta` events straight from `ExploitabilityAssessmentWorkflow`.

---

## 10. Error taxonomy

When something goes wrong, it goes wrong through one of two parallel vocabularies — a wire-level code shared across every transport, and an application-level exception class underneath it.

### DuxErrorCode — shared across REST, SSE, and webhooks

| Code | HTTP |
|------|------|
| `AGENT_TIMEOUT` | 504 |
| `CONTEXT_EXHAUSTED` | 422 |
| `BUDGET_EXCEEDED` | 429 |
| `GOVERNANCE_BLOCKED` | 403 |
| `INSUFFICIENT_DATA` | 422 — subtypes `asset_gap` · `intel_gap` · `context_limit` |

### Application error classes

| Exception | HTTP |
|-----------|------|
| `TenantIsolationError` | 403 |
| `ConnectorSyncError` | 502 |
| `AgentBudgetExceeded` | 429 |
| `AssessmentDedupConflict` | 409 |

A workflow abandons at **15 minutes**. Context checkpoints at ≥80% depth, and resumes via `POST /assessments/{id}/resume`.

---

## 11. Platform edge cases

Every one of these rows exists because something specific was observed to go wrong, and each carries the exact behavior that's supposed to happen instead plus the test ID that verifies it:

| Scenario | Expected behavior | Test |
|----------|-------------------|------|
| NVD unavailable >2 h | serve from S3 cache; warn `NVD_SYNC_WARN` at >2 h, critical `NVD_SYNC_STALE` at >4 h | CONN-001 |
| AWS throttling | exponential backoff + jitter; max 5 retries; `AWS_SYNC_FAILURE` P2 | CONN-002 |
| Workflow action storm | continue-as-new at 35 K events; max 2 retries | TEMP-001 |
| Cross-tenant URL manipulation | **404, to mask existence.** More than 20 masked 404s per IP per 5 min → SIEM | ISO-001 |
| Webhook delivery failure | 5 attempts, then dead-letter; alert above 10 per tenant per hour | WH-001 |
| Rate limit exceeded | 429 (RFC 6585) + `Retry-After` (RFC 9110); admin notified if the queue stalls >5 min | API-001 |
| Context window exhaustion | checkpoint into `assessment_checkpoints` (JSONB); resume endpoint; abandon with `CONTEXT_EXHAUSTED` at ≥80% depth | AGENT-001 |

---

## 12. The extensibility framework: how a new axis gets added without drifting

This is the mechanism that keeps the fourteen-axis registry system below from decaying the way most "add a row to this table" systems eventually do. Every extension axis — integrations, agents, models, events, flags, actions — conforms to the same eight-part contract:

1. A path-derived, stable ID.
2. The SSoT registry is a manifest view.
3. A controlled vocabulary.
4. Spec location is separate from the registry.
5. Gate + Unleash flag binding.
6. Bidirectional BR/FR/US traceability.
7. A lifecycle runbook — add, promote, deprecate, retire.
8. CI parity.

### The expansion ripple, in order

**new ID → registry row → spec → BR/FR/US link → gate + flag → event + test → CI green → deprecate on retire.**

### Controlled vocabularies (validator-enforced)

| Field | Values |
|-------|--------|
| `kind` | `cloud_connector`, `vendor_connector`, `intel_feed`, `research_source`, `customer_idp`, `roadmap` |
| `capability` | `ingest`, `action`, `idp`, `intel` |
| `status` | `active`, `gate_2c`, `gate_3`, `gate_5`, `ui_evidence`, `platform_managed`, `roadmap`, `deprecated`, `retired` |
| agent `layer` | `product_persona`, `runtime_service`, `workflow_subagent`, `gate3_workflow`, `third_party_isv`, `roadmap` |

**The filesystem plus the AI-BOM is the source of truth.** The tables in this page are human-readable views that CI compiles and validates (`validate-playbooks.py`, `pnpm test:agent-registry-parity`, `aibom-validate`). **Add rows here or in the named catalog — never in a derived view.**

---

## 13. Integration catalog

This is where every connector, feed, and research source Dux talks to gets a stable ID, a role, and a wave. **The connector framework and ≥3 live connectors (CrowdStrike, Wiz, ServiceNow/Entra) ship at Gate 1** (ADR-011 R2). Ingest gates for the W1 launch set are Gate 1; W2 and W3 are later waves.

| `integration_id` | kind | capabilities | role | ingest wave | action | primary US | ingest spec |
|----------------|------|--------------|------|-------------|--------|-----------|-------------|
| `aws` | cloud_connector | ingest | asset_discovery | **W1 / Gate 1** | — | US-011, US-013 | ADR-004 |
| `nvd` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-001, US-011 | NVD ingestion — fallback priority below |
| `cisa-kev` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-011 | NVD ingestion |
| `epss` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-011 | NVD ingestion |
| `csv-fallback` | cloud_connector | ingest | — | **Gate 1** | — | US-013 | FR-015 |
| `github-research`, `metasploit-index`, `medium-rss`, `microsoft-msrc` | research_source | intel | — | **Gate 1** | — | US-001 | MCP read tools |
| `crowdstrike` | vendor_connector | ingest, action | asset_discovery | **W1 / Gate 1** | **Gate 1, unattended** | US-002–004 | ADR-011 annex |
| `wiz` | vendor_connector | ingest | scanner | **W1 / Gate 1** | — | US-005, US-011 | ADR-011 annex — cadence not yet derived, see Open Items |
| `servicenow` | vendor_connector | ingest, action | ticketing + identity | **W1 / Gate 1** | **Gate 1, unattended** | US-007, US-018 | ADR-011 annex |
| `entra-id` | vendor_connector | ingest | identity | **W1 / Gate 1** | — | US-007, US-009 | ADR-011 annex |
| `splunk` | vendor_connector | ingest | siem / runtime_telemetry | **W1 / Gate 1** | — | US-002 | ADR-011 annex |
| `intune`, `qualys` | vendor_connector | ingest (+action) | asset_discovery / scanner | W2 | Gate 3 | US-002–005 | ADR-011 annex |
| `okta-idp`, `entra-idp` | customer_idp | idp | — | Seed SSO | — | US-014 | seed SSO runbook |
| `sentinelone`, `prisma-cloud`, `siem-generic` | vendor_connector / roadmap | ingest | various — `siem-generic` covers non-Splunk SIEMs | roadmap | — | — | — |
| `jamf`, `kandji`, `workspaceone`, `apple_business_manager`, `microsoft_defender`, `trellix`, `tanium`, `azure_app`, `azure_cloud`, `gcp`, `kace` | vendor_connector / roadmap | ingest | asset_discovery — MDM/UEM, EDR/XDR, and cloud-platform asset inventory (D-54) | roadmap | — | — | — |
| `jumpcloud`, `okta`, `onelogin`, `google_workspace`, `google_cloud_identity` | vendor_connector / roadmap | ingest | identity (D-54) | roadmap | — | — | — |
| `rapid7_insightvm`, `tenable_vm`, `tenable_sc`, `nessus`, `orca`, `upwind` | vendor_connector / roadmap | ingest | scanner — vulnerability/CNAPP posture scanners (D-54) | roadmap | — | — | — |
| `jira`, `freshservice`, `servicedesk_plus` | vendor_connector / roadmap | ingest | ticketing (D-54) | roadmap | — | — | — |
| `exploitdb`, `vulncheck`, `first` | vendor_connector / roadmap | ingest | threat_intel — `first` is FIRST.org exploit-maturity/CVSS data (D-54) | roadmap | — | — | — |
| `netskope`, `perimeter81`, `prisma_browser`, `island` | vendor_connector / roadmap | ingest | network_context — first real use of this ADR-011 R2 role interface (D-54) | roadmap | — | — | — |
| `horizon3_nodezero` | vendor_connector / roadmap | ingest | validation — autonomous pentest/BAS, first real use of this ADR-011 R2 role interface (D-54) | roadmap | — | — | — |

Two-thirds of that table is roadmap rows rather than shipped connectors, and that's deliberate: every real vendor worth integrating with gets a placeholder row and a role the day it's identified, long before engineering hours are budgeted for it, so nothing has to be invented later just to file a decision about it.

---

## 14. The Sources OpenAPI enum: all 42 values, and why the count is bigger than the connector list

This section exists to head off a specific, recurring confusion. **The `Sources` enum is not the connector list.** The OpenAPI `Sources` enum values are used as scanner and ingest provenance **attribution tags** on API responses. They are not contractual connectors.

**Marketing's "10+ tools" means integration-catalog connectors, not the attribution tags.** The full source × ConnectorRole × wave taxonomy (roles: `asset_discovery`, `scanner`, `identity`, `threat_intel`, `network_context`, `validation`, `ticketing`) lives in ADR-011 R2. **CI asserts that no enum value lacks a taxonomy row.** This count (42) differs from the count cited in ADR-011 R2; the gap is tracked as Open Items.

The `Sources` enum, as shipped on the wire (`components.schemas.Sources`, `docs/30-api/openapi.yaml`), all 42 values:

`jamf`, `jumpcloud`, `crowdstrike`, `aws`, `okta`, `kandji`, `jira`, `workspaceone`, `onelogin`, `microsoft_entra`, `microsoft_intune`, `microsoft_defender`, `azure_app`, `qualys`, `azure_cloud`, `gcp`, `netskope`, `kace`, `servicenow`, `google_workspace`, `google_cloud_identity`, `prisma_browser`, `rapid7_insightvm`, `perimeter81`, `sentinelone`, `island`, `horizon3_nodezero`, `orca`, `wiz`, `apple_business_manager`, `tenable_vm`, `tenable_sc`, `nessus`, `trellix`, `tanium`, `nvd`, `first`, `cisa`, `exploitdb`, `freshservice`, `upwind`, `servicedesk_plus`, `vulncheck`.

All 42 are confirmed live vendor/source identifiers off the wire — not a smaller subset. `aws`, `nvd`, `crowdstrike`, `wiz`, `servicenow`, `qualys`, and `sentinelone` match an `integration_id` in the catalog above (§13) literally. `cisa` and `microsoft_entra` are the likely wire-name counterparts of `cisa-kev` and `entra-id` respectively, but the enum uses a different naming scheme (bare snake_case) than the catalog (kebab-case with a role suffix — `-kev`, `-id`, `-idp`) — this divergence is not yet confirmed as a 1:1 identity and is folded into Open Items. The remaining 33 values — Jamf, JumpCloud, Okta, Kandji, Jira, Workspace ONE, OneLogin, Microsoft Intune, Microsoft Defender, Azure AD App, Azure Cloud, GCP, Netskope, Kace, Google Workspace, Google Cloud Identity, Prisma Access Browser, Rapid7 InsightVM, Perimeter 81, Island, Horizon3.ai NodeZero, Orca Security, Apple Business Manager, Tenable.io, Tenable.sc, Nessus, Trellix, Tanium, FIRST/EPSS, Exploit-DB, Freshservice, Upwind, ServiceDesk Plus, and VulnCheck — are real, named vendors with **no catalog row yet**. That is a coverage gap in the integration catalog, not an open question about whether they're legitimate sources.

### How those 33 not-yet-built values break down by role

| Role | Count | Vendors |
|---|---|---|
| `asset_discovery` | 11 | jamf, kandji, workspaceone, apple_business_manager, microsoft_defender, trellix, tanium, azure_app, azure_cloud, gcp, kace |
| `identity` | 5 | jumpcloud, okta, onelogin, google_workspace, google_cloud_identity |
| `scanner` | 6 | rapid7_insightvm, tenable_vm, tenable_sc, nessus, orca, upwind |
| `ticketing` | 3 | jira, freshservice, servicedesk_plus |
| `threat_intel` | 3 | exploitdb, vulncheck, first (FIRST.org) |
| `network_context` | 4 | netskope, perimeter81, prisma_browser, island |
| `validation` | 1 | horizon3_nodezero (autonomous pentest/BAS) |

---

## 15. Where the ingestion spec actually lives (it's scattered on purpose)

A newcomer looking for "the" ingestion spec will not find one file — and that's a deliberate editorial choice, not an oversight. The spec is correctly cross-referenced but scattered across five files, each owning the layer it's canonical for, per standing editorial rule 3. This is the index that ties them together:

| Layer | File |
|-------|------|
| Source/connector taxonomy (this table), wave gating, `Sources` enum | catalogs §13 (here) |
| Vendor sync mechanics — AWS, CSV fallback, vendor connectors, cadence | Dux Connector Hub |
| Trust boundaries, K8s service isolation for sync, NVD/KEV/EPSS feed integration | [[Dux Architecture Guide]] §1–2 |
| Ingested-entity schema — `ASSET`, `FINDING`, `VULNERABILITY_INSTANCE`, retention | [[Dux Architecture Guide]] §2, §4 |
| Public read surface over ingested data | [[Dux API Reference]] |

---

## 16. When NVD goes dark: the enrichment fallback

NVD enrichment collapsed on 2026-04-15, leaving roughly **29,000 pre-March-2026 CVEs** stuck in a `Not Scheduled` state. That event is the reason this fallback exists at all: when NVD enrichment is unavailable, or a CVE is `Not Scheduled`, the ingest pipeline falls back to `cisa-kev` (a binary known-exploited signal) and `epss` (a probability score) as the primary enrichment sources for that CVE.

This mirrors the H8 stale-connector-evidence pattern from §3, and the enforcement is just as strict: **missing full NVD enrichment caps the assessment at the `likely` confidence band — it can never reach `exploitable` — until NVD data resolves for that CVE.** CISA-KEV and EPSS evidence alone is not sufficient grounds for the highest confidence band, no matter how strong the signal looks on its own.

---

## 17. Agent catalog

Every agent that runs in production has a row here — its layer, whether it's customer-visible, its blast radius, and the gate it ships at:

| `agent_id` | layer | customer-visible | `agent_type` | blast radius | gate | primary US |
|----------|-------|------------------|-----------|--------------|------|-----------|
| `dux-agent` | product_persona | Y | — | — | Gate 1 | US-008–011 |
| `dux-assessment` | runtime_service | N | assessment | medium | Gate 1 | US-001, US-010, US-011 |
| `dux-chat-guidance` | runtime_service | Y | supervised | medium | Gate 1 | US-008 |
| `prerequisite-extractor` | workflow_subagent | N | assessment | low | Gate 1 | US-001 |
| `asset-context-worker` | workflow_subagent | N | assessment | low | Gate 1 | US-002 |
| `control-mapping-worker` | workflow_subagent | N | assessment | low | Gate 1 | US-003 |
| `mitigation-agent` | runtime_service | N | autonomous — HITL on anomaly escalation only | high | **Gate 1** | US-004, US-016 |
| `remediation-agent` | runtime_service | N | autonomous — HITL on anomaly escalation only | high | **Gate 1** | US-018 |
| `dux-resident-agent` | runtime_service | N | physical_resident | high | Gate 5 | US-020 |
| `third-party-isv` | third_party_isv | Y | supervised | high | Series B | — |

### What it takes to add a new one

A new runtime agent requires **all of**:

- A blast-radius row
- A kill-switch scope
- An MCP allowlist
- An OWASP agentic mapping
- A `packages/agents/{id}/` directory (ADR-009)
- A parity test

The agent persona (Dux Agent by default) is defined in [[Dux Product Guide]].

---

## 18. Model provider catalog — and the retention problem sitting underneath it

Model pins are dated 2026-06, on a **quarterly pin-review cadence** tied to the CI pin gate — checked against each provider's current GA lineup and retirement policy every quarter, not only when a pin breaks:

| `provider_id` | role | model strings | residency | status |
|-------------|------|---------------|-----------|--------|
| `openai` | primary | `openai/gpt-5.4`, `openai/gpt-5.4-mini` | US | active |
| `anthropic` | fallback | `anthropic/claude-sonnet-4-6`, `anthropic/claude-haiku-4-5` | US | active — evaluate `claude-sonnet-5` as the fallback pin at the next quarterly refresh (intro pricing $2/$10 in) |
| `azure-openai-eu` | eu_residency | per contract | EU | gate_trigger |

Escalation to `openai/gpt-5.5` runs behind the Unleash flag `reasoning_model_tier` — enterprise plus critical CVEs only.

**Providers are subprocessors — data leaves. Connectors are not.** That distinction is the reason the retention question below matters as much as it does.

**Model IDs and prices are point-in-time pins.** The durable mechanism is `models.json` + AI-BOM + the CI pin gate (standing editorial rule 2, Dux Decisions Log).

### Zero Data Retention (H2 — Gate-1 legal task)

Here's the part of this catalog that isn't just a lookup table — it's an open legal and procurement risk stated plainly: **Dux sends customer environment topology — assets, controls, network, identity — to these providers.**

Neither provider trains on API data by default. **But abuse-monitoring logs are retained by default: OpenAI ~30 days; Anthropic 7 days by default since 2025-09-14.** Anthropic's "Covered Models" list (per Anthropic's Usage Policy) is excluded from even that 7-day default and follows separate terms. **ZDR is a separate contractual arrangement, and requires provider approval at both vendors.**

For a security vendor, default retention of a customer's live attack surface — 7 to 30 days depending on provider — is an enterprise-procurement blocker, not a footnote.

**ZDR negotiated with OpenAI and Anthropic is a precondition of subprocessor listing.** Until ZDR is in place, **the CaMeL S-LLM must not receive customer-identifying context.** That's a data-residency invariant, not merely an injection defense — see [[Dux AI Safety Guide]] for the full boundary design.

---

## 19. Event catalog (SSoT for webhooks / SSE / audit)

Every event the platform can emit — over webhook, SSE, or the audit stream — has a canonical ID, a producer, and a gate. Full DTO and delivery spec: [[Dux API Reference]].

| `event_id` | class | gate | producer |
|----------|-------|------|----------|
| `assessment.completed`, `assessment.state_changed` | webhook | Gate 1 | assessment workflow |
| `assessment.requeued` | webhook | Gate 1 | US-021 scheduler (ADR-016) |
| `finding.created` / `updated` / `deleted` | webhook | Gate 1 | World Model ingest |
| `vulnerability_instance.updated` | webhook | Seed | instance projection |
| `vulnerability_instance.acknowledged`, `.acknowledgment_expired` | webhook | Gate 1 | acknowledgment service |
| `attack_path.validated` | webhook | Gate 1 | graph projection |
| `ownership.inferred` | webhook | Gate 1 | US-007 |
| `preference.applied` / `rejected` | webhook | Gate 2c | US-009 |
| `control_asset_mapping.updated` | webhook | Gate 1 | US-003 sync |
| `mitigation.executed` / `blocked` | webhook | **Gate 1, unattended** | ADR-012 / governance kernel |
| `remediation.ticket_created`, `ticket.created` / `updated` / `resolved` / `reopened` | webhook | **Gate 1 — create + route, unattended** | ServiceNow adapter |
| `cve_research.backlog` / `completed`, `custom_metric.updated` | webhook | Seed | research queue / metric admin |
| `connector.sync_failed`, `kill_switch.activated` | webhook | Gate 1 | connector / KS-001 |
| `hitl_request` | sse | Gate 1 — anomaly escalation only | US-008 SSE |
| `governance.kill_switch_short_circuit`, `agent.created`, `admin.impersonate` | audit | Gate 1 | governance kernel / provisioning |
| `tenant.provisioned` | audit | Seed | provisioning |

---

## 20. Feature-flag catalog — and why it's not the same thing as the kill switch

Unleash flags govern release rollout. **These are distinct from the kill switch** — a flag is a release control; the kill switch is a safety control. Never conflate the two, because they answer different questions: a flag asks "should this ship yet," a kill switch asks "should this stop running right now."

| `flag_id` | gate | default | kill-switch relation |
|---------|------|---------|----------------------|
| `chat_interface`, `trace_viewer` | Gate 1 | on | — |
| `camel_plane` | Gate 1 | on | — |
| `api_webhooks` | Gate 1 | off until configured | — |
| `ownership_inference` | Gate 1 | on | — |
| `hitl_ui` | Week 12 — minimal approve/deny UI (D-4) | off | HITL required |
| `chat_write_tools` | Week 14 — full chat HITL UI (D-4) | off | HITL required |
| `mitigation_stage`, `remediation_stage` | **Gate 1 — unattended by default for 3 of 5 write actions** | **on** | L2 tenant scope |
| `closed_loop_validation` | Gate 3 | off | — |
| `optional_physical_residency` | Gate 5 | off | — |
| `reasoning_model_tier` | enterprise | off | cost cap |
| `platt_scaling` | Gate 2+ | off | requires a calibration record |
| `risk_trend_forecasting` | Gate 2 | off | — |
| `rag_enabled` | Seed, reassess | **on** (2026-07-19, D-34 — Agentic RAG with constrained decoding, ADR-020 R2) | — |

`mitigation_stage` and `remediation_stage` describe unattended automation for the three earned-autonomy actions, and they default **on** at Gate 1. The approval surface (D-4) is the anomaly-escalation UI for those three; for `endpoint.isolate` and `patch.deploy_special_devices` it is the mandatory pre-approval gate — see [[Dux AI Safety Guide]] for how that distinction is enforced end to end.

---

## 21. Vendor action catalog — the write path

This is the single most consequential table on this page: it's the exact list of everything Dux Agent is allowed to *do* to a customer's live security stack, and the HITL posture attached to each action.

**The HITL tier column is the tier used for anomaly escalation on the three earned-autonomy actions. For `endpoint.isolate` and `patch.deploy_special_devices` it is a mandatory gate on every call, not an escalation-only path.**

| `canonical_action_id` | blast radius | HITL tier (escalation) | integration | gate |
|---------------------|--------------|------------------------|-------------|------|
| `endpoint.isolate` | high | T3 | `crowdstrike` | **Gate 1, mandatory HITL** — not unattended; earns unattended execution via a field-proven Gate-3 safety record (D-17) |
| `network.blocklist_add` | medium | T2 | `crowdstrike` (IOC/indicator blocklist — D-50) | **Gate 1, unattended by default** |
| `policy.deploy_device_config` | medium | T2 | `intune` | **Gate 3** — the Intune connector is W2/Gate 3, so the action is unavailable until it lands. Unattended-by-default applies once it ships |
| `patch.deploy_special_devices` | high | T3 | — | **Gate 1, mandatory HITL** — no API-level rollback on firmware-only devices; not unattended |
| `ticket.create_remediation` | low | T1 | `servicenow` | **Gate 1 (create + route), unattended by default** |

**Writes flow only through `VendorActionGate`. Connectors must not call vendor mutation APIs.** Vendor-native name mapping lives in ADR-012.

---

## 22. Reasoning eval catalog

Five named evals gate every model change before it can regress silently:

| Eval | Asserts |
|------|---------|
| `EXP-CIT-001` | a citation is present |
| `EXP-FN-001` | false-negative guard |
| `EXP-NOISE-001` | non-exploitable findings are deprioritized |
| `EXP-AWS-SG-001` | `aws_sg_blocks_port` correctness |
| `EXP-VENDOR-001` | the CrowdStrike firewall card |

**A golden-set regression above 2% is a P0 merge block** (NFR-008).

**Personalization artifacts are tenant-scoped. There is never cross-tenant training** — a design invariant worth stating alongside the evals, since it's exactly the kind of guarantee an eval suite alone can't prove.

---

## 23. Compliance framework catalog

| Framework | Trigger |
|-----------|---------|
| `soc2-type-i` | seed |
| `soc2-type-ii` | Series A |
| `iso-27001`, `iso-42001` | Series A |
| `eu-ai-act` | first EU prospect. Art. 50 transparency from 2 Aug 2026; Art. 9/Annex III best-practice posture from 2 Dec 2027 (D-26) |

Full detail in [[Dux Governance & Compliance Guide]].

### Isolation invariants (non-negotiable)

Three architectural guarantees sit underneath every compliance claim above and are never traded away for convenience:

- **RLS FORCE** on every `tenant_id` table
- **Composite foreign keys**
- **Apache AGE graph isolation**, with Neo4j reserved as a future escape valve only

---

## 24. The other ten axes of the registry — and where they actually live

The full controlled-vocabulary index spans 14 axes; this page owns the ones dense enough to need their own tables. The rest live where their canonical spec already lives, cross-referenced here rather than duplicated:

| Axis | Canonical home |
|------|----------------|
| Agent personas (voice) | the agent catalog above, plus [[Dux Feature Reference]] — maps to each agent's `instructions.md` |
| World Model evidence types | [[Dux Architecture Guide]] + [[Dux AI Safety Guide]] |
| Mitigation factor cards | §6 of this page. Figma "playbook cards" **are** factor cards — never a separate entity |
| MCP tools | the [[Dux AI Safety Guide]] tool catalog |
| User stories US-001…027 | Dux Traceability Matrix |
| Gates | [[Dux Product Guide]] §5 |

---

## Sources

- `.raw/dux/10-product/taxonomy.md`
- `.raw/dux/10-product/catalogs.md`
</content>

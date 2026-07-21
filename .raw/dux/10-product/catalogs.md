---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-4, D-17, D-34, H2, H5]
---

# Catalogs — Registries of Record

**Purpose:** the registries every extension axis is declared in — integrations, agents, models, events, flags, actions, evals, compliance frameworks.

**Parents:** all epics.

**The filesystem plus the AI-BOM is the source of truth.** The tables below are human-readable views that CI compiles and validates (`validate-playbooks.py`, `pnpm test:agent-registry-parity`, `aibom-validate`). **Add rows here or in the named catalog — never in a derived view.**

## Extensibility framework (the 8-part contract)

Every extension axis conforms to all eight:

1. A path-derived, stable ID.
2. The SSoT registry is a manifest view.
3. A controlled vocabulary.
4. Spec location is separate from the registry.
5. Gate + Unleash flag binding.
6. Bidirectional BR/FR/US traceability.
7. A lifecycle runbook — add, promote, deprecate, retire.
8. CI parity.

**The expansion ripple, in order:** new ID → registry row → spec → BR/FR/US link → gate + flag → event + test → CI green → deprecate on retire.

**Controlled vocabularies (validator-enforced):**

| Field | Values |
|-------|--------|
| `kind` | `cloud_connector`, `vendor_connector`, `intel_feed`, `research_source`, `customer_idp`, `roadmap` |
| `capability` | `ingest`, `action`, `idp`, `intel` |
| `status` | `active`, `gate_2c`, `gate_3`, `gate_5`, `ui_evidence`, `platform_managed`, `roadmap`, `deprecated`, `retired` |
| agent `layer` | `product_persona`, `runtime_service`, `workflow_subagent`, `gate3_workflow`, `third_party_isv`, `roadmap` |

## 1. Integration catalog

**The connector framework and ≥3 live connectors (CrowdStrike, Wiz, ServiceNow/Entra) ship at Gate 1** (ADR-011 R2). Ingest gates for the W1 launch set are Gate 1; W2 and W3 are later waves.

| `integration_id` | kind | capabilities | role | ingest wave | action | primary US | ingest spec |
|----------------|------|--------------|------|-------------|--------|-----------|-------------|
| `aws` | cloud_connector | ingest | asset_discovery | **W1 / Gate 1** | — | US-011, US-013 | ADR-004 |
| `nvd` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-001, US-011 | NVD ingestion — fallback priority below |
| `cisa-kev` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-011 | NVD ingestion |
| `epss` | intel_feed | intel | threat_intel (first-party) | **W1 / Gate 1** | — | US-011 | NVD ingestion |
| `csv-fallback` | cloud_connector | ingest | — | **Gate 1** | — | US-013 | FR-015 |
| `github-research`, `metasploit-index`, `medium-rss`, `microsoft-msrc` | research_source | intel | — | **Gate 1** | — | US-001 | MCP read tools |
| `crowdstrike` | vendor_connector | ingest, action | asset_discovery | **W1 / Gate 1** | **Gate 1, unattended** | US-002–004 | ADR-011 annex |
| `wiz` | vendor_connector | ingest | scanner | **W1 / Gate 1** | — | US-005, US-011 | ADR-011 annex — cadence not yet derived, see [OI-58](../00-meta/open-items.md) |
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

### Data Ingestion Spec Index (resolves part of [OI-31](../00-meta/open-items.md))

No single indexed doc holds the ingestion spec — it is correctly cross-referenced but scattered across five files by design (each owns the layer it's canonical for, per standing editorial rule 3). This is that index:

| Layer | File |
|-------|------|
| Source/connector taxonomy (this table), wave gating, `Sources` enum | catalogs §1 (here) |
| Vendor sync mechanics — AWS, CSV fallback, vendor connectors, cadence | [connector-hub](features/connector-hub.md) |
| Trust boundaries, K8s service isolation for sync, NVD/KEV/EPSS feed integration | [architecture-overview §1–2](../20-architecture/architecture-overview.md) |
| Ingested-entity schema — `ASSET`, `FINDING`, `VULNERABILITY_INSTANCE`, retention | [data-model §2, §4](../20-architecture/data-model.md) |
| Public read surface over ingested data | [public-data-api](../30-api/public-data-api.md) |

**The `Sources` enum is not the connector list.** The OpenAPI `Sources` enum values are used as scanner and ingest provenance **attribution tags** on API responses. They are not contractual connectors.

**Marketing's "10+ tools" means integration-catalog connectors, not the attribution tags.** The full source × ConnectorRole × wave taxonomy (roles: `asset_discovery`, `scanner`, `identity`, `threat_intel`, `network_context`, `validation`, `ticketing`) lives in ADR-011 R2 ([adr-index](../20-architecture/adr-index.md)). **CI asserts that no enum value lacks a taxonomy row.** This count (42) differs from the count cited in ADR-011 R2; the gap is tracked as [OI-37](../00-meta/open-items.md).

**The `Sources` enum, as shipped on the wire (`components.schemas.Sources`, `docs/30-api/openapi.yaml`), 42 values:**

`jamf`, `jumpcloud`, `crowdstrike`, `aws`, `okta`, `kandji`, `jira`, `workspaceone`, `onelogin`, `microsoft_entra`, `microsoft_intune`, `microsoft_defender`, `azure_app`, `qualys`, `azure_cloud`, `gcp`, `netskope`, `kace`, `servicenow`, `google_workspace`, `google_cloud_identity`, `prisma_browser`, `rapid7_insightvm`, `perimeter81`, `sentinelone`, `island`, `horizon3_nodezero`, `orca`, `wiz`, `apple_business_manager`, `tenable_vm`, `tenable_sc`, `nessus`, `trellix`, `tanium`, `nvd`, `first`, `cisa`, `exploitdb`, `freshservice`, `upwind`, `servicedesk_plus`, `vulncheck`.

All 42 are confirmed live vendor/source identifiers off the wire — not a smaller subset. `aws`, `nvd`, `crowdstrike`, `wiz`, `servicenow`, `qualys`, and `sentinelone` match an `integration_id` in the catalog above (§1) literally. `cisa` and `microsoft_entra` are the likely wire-name counterparts of `cisa-kev` and `entra-id` respectively, but the enum uses a different naming scheme (bare snake_case) than the catalog (kebab-case with a role suffix — `-kev`, `-id`, `-idp`) — this divergence is not yet confirmed as a 1:1 identity and is folded into [OI-37](../00-meta/open-items.md). The remaining 33 values — Jamf, JumpCloud, Okta, Kandji, Jira, Workspace ONE, OneLogin, Microsoft Intune, Microsoft Defender, Azure AD App, Azure Cloud, GCP, Netskope, Kace, Google Workspace, Google Cloud Identity, Prisma Access Browser, Rapid7 InsightVM, Perimeter 81, Island, Horizon3.ai NodeZero, Orca Security, Apple Business Manager, Tenable.io, Tenable.sc, Nessus, Trellix, Tanium, FIRST/EPSS, Exploit-DB, Freshservice, Upwind, ServiceDesk Plus, and VulnCheck — are real, named vendors with **no catalog row yet**. That is a coverage gap in the integration catalog, not an open question about whether they're legitimate sources.

### NVD enrichment fallback priority

NVD enrichment collapsed 2026-04-15, leaving roughly 29,000 pre-March-2026 CVEs stuck `Not Scheduled`. When NVD enrichment is unavailable or a CVE is `Not Scheduled`, the ingest pipeline falls back to `cisa-kev` (binary known-exploited signal) and `epss` (probability score) as the primary enrichment sources for that CVE.

This mirrors the H8 stale-connector-evidence pattern in [taxonomy §1](taxonomy.md#evidence-freshness-caps-confidence-h8-gate-2-hardening): **missing full NVD enrichment caps the assessment at the `likely` confidence band — it can never reach `exploitable` — until NVD data resolves for that CVE.** CISA-KEV and EPSS evidence alone is not sufficient grounds for the highest confidence band.

## 2. Agent catalog

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

**A new runtime agent requires all of:** a blast-radius row, a kill-switch scope, an MCP allowlist, an OWASP agentic mapping, a `packages/agents/{id}/` directory (ADR-009), and a parity test.

The agent persona (Dux Agent by default) is defined in [product-overview](product-overview.md).

## 3. Model provider catalog

Pins dated 2026-06, on a **quarterly pin-review cadence** tied to the CI pin gate — checked against each provider's current GA lineup and retirement policy every quarter, not only when a pin breaks.

| `provider_id` | role | model strings | residency | status |
|-------------|------|---------------|-----------|--------|
| `openai` | primary | `openai/gpt-5.4`, `openai/gpt-5.4-mini` | US | active |
| `anthropic` | fallback | `anthropic/claude-sonnet-4-6`, `anthropic/claude-haiku-4-5` | US | active — evaluate `claude-sonnet-5` as the fallback pin at the next quarterly refresh (intro pricing $2/$10 in) |
| `azure-openai-eu` | eu_residency | per contract | EU | gate_trigger |

Escalation to `openai/gpt-5.5` runs behind the Unleash flag `reasoning_model_tier` — enterprise plus critical CVEs only.

**Providers are subprocessors — data leaves. Connectors are not.**

**Model IDs and prices are point-in-time pins.** The durable mechanism is `models.json` + AI-BOM + the CI pin gate (standing editorial rule 2, [decisions-log](../00-meta/decisions-log.md)).

### Zero Data Retention (H2 — Gate-1 legal task)

**Dux sends customer environment topology — assets, controls, network, identity — to these providers.**

Neither trains on API data by default. **But abuse-monitoring logs are retained by default: OpenAI ~30 days; Anthropic 7 days by default since 2025-09-14.** Anthropic's "Covered Models" list (per Anthropic's Usage Policy) is excluded from even that 7-day default and follows separate terms. **ZDR is a separate contractual arrangement, and requires provider approval at both vendors.**

For a security vendor, default retention of a customer's live attack surface — 7 to 30 days depending on provider — is an enterprise-procurement blocker.

**ZDR negotiated with OpenAI and Anthropic is a precondition of subprocessor listing.** Until ZDR is in place, **the CaMeL S-LLM must not receive customer-identifying context.** That is a data-residency invariant, not merely an injection defense ([camel-plane](../40-ai-safety/camel-plane.md)).

## 4. Event catalog (SSoT for webhooks / SSE / audit)

Full DTO and delivery spec: [events-webhooks](../30-api/events-webhooks.md).

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

## 5. Feature-flag catalog

Unleash flags. **These are distinct from the kill switch** — a flag is a release control; the kill switch is a safety control.

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
| `rag_enabled` | Seed, reassess | **on** (2026-07-19, D-34 — Agentic RAG with constrained decoding, [ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval)) | — |

`mitigation_stage` and `remediation_stage` describe unattended automation for the three earned-autonomy actions, and default **on** at Gate 1. The approval surface (D-4) is the anomaly-escalation UI for those three; for `endpoint.isolate` and `patch.deploy_special_devices` it is the mandatory pre-approval gate — see [kill-switch-hitl](../40-ai-safety/kill-switch-hitl.md).

## 6. Vendor action catalog (write path)

**The HITL tier column is the tier used for anomaly escalation on the three earned-autonomy actions. For `endpoint.isolate` and `patch.deploy_special_devices` it is a mandatory gate on every call, not an escalation-only path.**

| `canonical_action_id` | blast radius | HITL tier (escalation) | integration | gate |
|---------------------|--------------|------------------------|-------------|------|
| `endpoint.isolate` | high | T3 | `crowdstrike` | **Gate 1, mandatory HITL** — not unattended; earns unattended execution via a field-proven Gate-3 safety record (D-17) |
| `network.blocklist_add` | medium | T2 | `crowdstrike` (IOC/indicator blocklist — D-50) | **Gate 1, unattended by default** |
| `policy.deploy_device_config` | medium | T2 | `intune` | **Gate 3** — the Intune connector is W2/Gate 3, so the action is unavailable until it lands. Unattended-by-default applies once it ships |
| `patch.deploy_special_devices` | high | T3 | — | **Gate 1, mandatory HITL** — no API-level rollback on firmware-only devices; not unattended |
| `ticket.create_remediation` | low | T1 | `servicenow` | **Gate 1 (create + route), unattended by default** |

**Writes flow only through `VendorActionGate`. Connectors must not call vendor mutation APIs.** Vendor-native name mapping: [ADR-012](../20-architecture/adr-index.md).

## 7. Reasoning eval catalog

| Eval | Asserts |
|------|---------|
| `EXP-CIT-001` | a citation is present |
| `EXP-FN-001` | false-negative guard |
| `EXP-NOISE-001` | non-exploitable findings are deprioritized |
| `EXP-AWS-SG-001` | `aws_sg_blocks_port` correctness |
| `EXP-VENDOR-001` | the CrowdStrike firewall card |

**A golden-set regression above 2% is a P0 merge block** (NFR-008).

**Personalization artifacts are tenant-scoped. There is never cross-tenant training.**

## 8. Compliance framework catalog

| Framework | Trigger |
|-----------|---------|
| `soc2-type-i` | seed |
| `soc2-type-ii` | Series A |
| `iso-27001`, `iso-42001` | Series A |
| `eu-ai-act` | first EU prospect. Art. 50 transparency from 2 Aug 2026; Art. 9/Annex III best-practice posture from 2 Dec 2027 (D-26) |

**Isolation invariants — non-negotiable:** RLS FORCE on every `tenant_id` table; composite foreign keys; Apache AGE graph isolation, with Neo4j reserved as a future escape valve only.

## 9. Other registries

The full index spans 14 axes. These live elsewhere:

| Axis | Canonical home |
|------|----------------|
| Agent personas (voice) | the agent catalog above, plus [chat-guidance](features/chat-guidance.md) — maps to each agent's `instructions.md` |
| World Model evidence types | [data-model](../20-architecture/data-model.md) + [camel-plane](../40-ai-safety/camel-plane.md) |
| Mitigation factor cards | [taxonomy](taxonomy.md). Figma "playbook cards" **are** factor cards — never a separate entity |
| MCP tools | the [mcp-security](../40-ai-safety/mcp-security.md) tool catalog |
| User stories US-001…027 | [traceability-matrix](../00-meta/traceability-matrix.md) |
| Gates | [product-overview §5](product-overview.md) |

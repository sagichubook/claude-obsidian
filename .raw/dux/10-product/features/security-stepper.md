---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [D-17, H5]
---

# Feature — Security Stepper (US-001–007, US-009)

**Purpose:** the investigation journey, from decomposing a CVE to routing the fix — steps 1–7 and 9 are documented here; step 8 (US-008, conversational investigation) is documented separately in [chat-guidance](chat-guidance.md).

**Nav:** Security · **Epics:** EP-02, EP-03, EP-06, EP-10 · **BRs:** BR-002, BR-004.

US-001 through US-007 are live at Gate 1 — read-only, or write (unattended-by-default for 3 of 5 canonical actions; `endpoint.isolate`/`patch.deploy_special_devices` mandatory HITL, D-17). US-009 is Gate 2c.

**Screens without a live connector render the full Figma layout with connector-degraded empty states** (deep-linking to US-013). They are not CTA-only shells.

## US-001 Prerequisites Analysis

**Gate 1.**

**Job.** A security engineer decomposes a CVE into its real-world exploitation requisites, with cited sources.

**Orchestration.** `ExploitabilityAssessmentWorkflow` (Temporal), triggered on queue enqueue or CVE selection. The `prerequisite-extractor` subagent gathers NVD, GitHub, Metasploit, and Medium evidence through MCP read tools. The CaMeL S-LLM/P-LLM boundary sanitizes the untrusted CVE text. Status streams live. Agent: AI #1 (REASONING).

**Data.** World Model `FINDING`, `CVE`, `EXPLOITABILITY_ASSESSMENT`, `ASSESSMENT_REASONING_STEP`. Degraded paths: a missing NVD record yields `INSUFFICIENT_DATA`; with AWS absent, prerequisites render from intel feeds alone.

**API.** `GET /assessments/{id}` → `AssessmentDto`. The trace is `GET /assessments/{id}/trace` (US-017).

**Safety.** KS-L1 halts the session. Intel staler than 24 h yields `INSUFFICIENT_DATA`. No HITL — this is read-only research.

**Metrics.** Completion rate; at least 4 prerequisite source slots, with NVD mandatory; time-to-first-card p95; golden-set accuracy.

**Competitive.** Tenable Hexa AI orchestrates but ships no prerequisite decomposition with per-source citations. Strobes triages in seconds. Dux differentiates on the reasoning chain plus executed code (US-017).

## US-002 Asset Context Evidence

**Gate 1**, read-only.

**Job.** Prove or disprove environmental exploitability, using runtime, network, SIEM, and role evidence on a specific asset.

**Orchestration.** `AssetContextWorker`. MCP tools query:

| Source | Status | Evidence |
|--------|--------|----------|
| CrowdStrike | live at Gate 1 (one of ADR-011 R2's ≥3 launch connectors) | endpoint state |
| AWS | live at Gate 1 | EC2 metadata, security-group reachability |
| Splunk | live at Gate 1 | SIEM process and runtime telemetry — e.g. listening-process evidence for a service |
| Identity / role | live at Gate 1 | last-login, role — from the US-007 ownership-evidence sources |
| Intune | **Gate 3 / wave W2** | renders a connector-degraded empty state until then |

**API.** `GET /assets/{id}/context` → `AssetContextDto` (FR-020) — shape defined in [application-api §2](../../30-api/application-api.md#2-key-dtos).

**Safety.** A stale connector yields `INSUFFICIENT_DATA`. **Vendor fields are never fabricated.** KS-L2 stops new gathers; in-flight gathers complete and are flagged as partial evidence.

## US-003 Protection Breakdown

**Gate 1**, read-only.

**Job.** Answer "where am I already protected?" with vendor control proof — CrowdStrike policies at Gate 1; Intune at Gate 3/W2. **This is distinct from the US-006 governance audit** (see Build rules).

**Orchestration.** `control-mapping-worker` correlates findings to active controls. MCP pulls CrowdStrike policy state at Gate 1. Output: segment cards — Protected, Partially Mitigated, Exposed — plus a settings and effect table.

**API.** `ProtectionBreakdownDto` via `?projection=protection` on `GET /cves/{id}/detail`. Also `GET /attack-paths` (AWS security groups + vendor).

**Data.** `CONTROL`, `CONTROL_MAPPING`.

## Step 4 — Action Cards (US-004)

**Gate 1.** Unattended by default for `network.blocklist_add`/`policy.deploy_device_config`; mandatory HITL for `endpoint.isolate`/`patch.deploy_special_devices` (D-17). **Canonical spec: [mitigation-write-path](mitigation-write-path.md).** This section is the journey summary only.

Surface and execute lightweight mitigations — blocklist at Gate 1, Intune policy steps once the W2 connector lands — faster than a full patch, with a residual-risk count. Every write flows through `VendorActionGate`; T2/T3 tiers classify it for audit and escalation. KS-L2 blocks new proposals. HITL fires only on anomaly escalation.

## US-005 Control Refinements

**Gate 1.**

**Job.** Surface the highest-impact estate-wide configuration changes — disable NTLM, enable IMDSv2 — ranked by exposure reduction.

**Orchestration.** `ControlRefinementQuery`, using the Specification pattern (`ByImpact`, `ByScanner`, `ByCVE`), aggregating across CVEs. Wiz ingest is live at Gate 1 (FR-019); Qualys is Gate 3/W2 per [catalogs](../catalogs.md).

**Ranking output carries an `effort` field** — S / M / L, sized by rollout scope, not build cost: **S** a single-toggle config change on one control; **M** a staged change across a device or asset group; **L** an estate-wide policy rollout needing change-management sign-off. Exposure reduction ranks the queue; `effort` is a secondary column shown alongside it, not a tiebreaker the ranking itself uses.

**API.** `GET /controls/refinements`, or the refinement DTO via `?projection`. ERD entity: `ControlRefinementAggregate`, field `effort`.

## Step 6 — Audit & Exposure Delta (US-006)

**Gate 1**, governance and audit. **Canonical spec: [dashboard-audit](dashboard-audit.md).** This section is the journey summary only.

A CISO-facing, board-ready exposure trend: delta cards and a tamper-evident audit trail. **This is not live vendor protection** — that is US-003 (see Build rules). There is no agent loop; it is a projection over assessment outcomes, and the governance kernel writes hash-chained `AUDIT_EVENT` rows.

## US-007 Ownership Inference

**Gate 1**, read-only.

**Job.** Route remediation to the right team, with a certainty percentage and ITSM/identity evidence.

**Orchestration.** The ownership-inference activity. MCP reads ServiceNow and Entra ID, both live at Gate 1. Webhook: `ownership.inferred`. Agent: AI #7.

**API.** `OwnershipInferenceDto` (FR-016). Data: `OWNERSHIP_EVIDENCE`, `ASSET.owner_team`.

**Safety.** Certainty below threshold yields `INSUFFICIENT_DATA` plus a manual-assign CTA. Auto-ticketing at Gate 1 creates and routes unattended (US-018).

## US-009 Preference Learning

**Gate 2c — not promoted.**

**Job.** A CISO teaches risk appetite in natural language, so future assessments respect scope.

**Why it stays at Gate 2c.** It needs behavioral-data volume: `PreferenceEngine`'s influence on scoring weights requires tenant assessment history that does not exist at Gate 1. The interim is session routing preferences (24 h TTL) plus per-instance acknowledgment (US-023).

**API.** `POST /preferences`, `GET /preferences` (FR-018). Data: `TENANT_PREFERENCE`, `PREFERENCE_RULE`.

**Safety.** Preference writes require the tenant-admin or CISO role. Ambiguous natural language raises a clarification prompt — **never a silent accept**. KS-L2 freezes updates.

## Build rules

**US-006 is not US-003. Engineering must not merge them.**

| Question | Story | Surface |
|----------|-------|---------|
| "How is exposure trending?" — governance and audit | **US-006**, Gate 1 | score gauge, delta cards, 30-day hash-chained log, CSV export |
| "Am I still protected?" — live vendor control proof | **US-003**, Gate 1 | vendor policy state |

The CISO Gate-1 demo uses US-006 metrics plus US-011 AWS and vendor factor cards.

**US-002 is not US-011.** Per-asset AWS fields live in the US-011 asset table. The US-002 standalone panel hydrates from CrowdStrike (Gate 1) and Intune (Gate 3/W2, connector-degraded until then).

## Figma and interim design

US-001–009 are confirmed in the June-2026 Figma set, screens 1–9. The vendor panels in US-002–005, US-007, and US-009 use connector-degraded empty states before hydration.

Figma's "playbook cards" **are** the mitigation factor cards. Do not introduce a separate entity for them.

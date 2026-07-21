---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-7, D-8]
---

# EP-02 — Environmental Data Ingestion (Connectors & Feeds)

**Objective:** BR-004 multi-source ingestion (KPI: time-to-value <48 h from AWS connect; ≥3 vendor connectors live at Gate 1 per ADR-011 R2) · **Metrics:** connector sync tests green; CONN-001 staleness; first report <48 h · **Target:** Gate 1 (Week 12, [D-7 R1](../00-meta/decisions-log.md)); AWS P0 by Week 6 vertical slice · **Priority:** P0 · **Constraints:** ADR-004 (SDK v3, cross-account IAM + external ID, single account/tenant), ADR-011 R2 (`vendor-contract.ts`, role interfaces, RLS-scoped, isolated K8s sync service), ADR-006 R4.

## EP-02-F01 — Connector Hub & first-party ingest

**Parent:** EP-02 · **FRs:** FR-002 (AWS), FR-003 (NVD/KEV/EPSS), FR-015 (CSV) · **ACs:** [connector-hub spec](../10-product/features/connector-hub.md); sync completes, asset count visible, actionable error banners; never false "Connected" · **Dependencies:** EP-01 RLS (tenant-scoped writes); K8s service split · **API boundary:** `POST /connectors/aws/sync`, `POST /connectors/csv/upload`, hub UI (Apps nav) · **SLA:** NVD delta ≤2 h; KEV 6 h; EPSS daily.

### US-013 — Connector Hub (Gate 1) · 13 pts · persona: tenant admin
**ACs:** AWS wizard + STS error mapping → `aws_role_status`; CSV typed errors, no partial poisoned ingest; sync-health strip + `connector.sync_failed` webhook. **DoD:** merge gates + connector sync integration tests.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-013-T01 | AWS SDK v3 discovery clients (EC2/IAM/ELBv2/ECS/EKS/S3/RDS/Lambda/CloudFront) + delta sync cursors | Backend | Code | 32 | 2–4 | TS-1 |
| US-013-T02 | Cross-account IAM assume + external ID + STS error mapping (`AccessDenied`/`ExternalIdMismatch`/`InvalidClientTokenId`) | Backend | Code | 12 | 3 | TS-1 |
| US-013-T03 | Throttling backoff+jitter, per-service limits, `AWSThrottledRequests` alerting | Backend | Code | 8 | 4 | TS-1 |
| US-013-T04 | NVD/CISA-KEV/EPSS ingestion workers + 48 h cache + staleness alerts (CONN-001) | Backend | Code | 20 | 2–4 | PY-1 |
| US-013-T05 | CSV fallback upload (multipart validation, typed errors, 50 K cap) | Backend | Code | 10 | 5 | TS-2 |
| US-013-T06 | Connector Hub UI (wizard, sync-now, health strip, "Coming soon" states) | Frontend | Code | 20 | 5–7 | TS-3 |
| US-013-T07 | Sync workflows as isolated K8s service (`dux-connector-sync` Deployment, blast-radius split, ADR-006 R4) | DevOps | Deploy | 12 | 5–6 | TS-2 |
| US-013-T08 | Connector sync integration tests + `connector.sync_failed` webhook test | QA | Test | 14 | 6–7 | TS-3 |
| EP-02-F01-T09 | World Model versioning mechanism: `world_model_versions` bump discipline, 24 h in-flight compat, `WorldModelVersionPurgeJob` (re-assessment backbone — costed spine, first-class Gate-1 build) | Backend | Code | 16 | 4–5 | TS-2 |

## EP-02-F02 — Vendor connector framework & evidence screens

**Parent:** EP-02 · **FRs:** FR-002 (vendor), FR-020 · **ACs:** ≥3 live connectors at Gate 1 (CrowdStrike, Wiz, ServiceNow/Entra); [security-stepper](../10-product/features/security-stepper.md) US-002/003 success criteria · **Dependencies:** F01 hub; vendor sandbox accounts · **API boundary:** `vendor-contract.ts` role interfaces; `GET /assets/{id}/context` · **SLA:** CrowdStrike/Wiz 6 h cadence.

### US-002 — Asset Context Evidence (Gate 1, promoted) · 5 pts · persona: security engineer
**ACs:** runtime/identity context on asset drill-down; degraded empty state deep-links hub. **DoD:** merge gates + standalone screen live. **Spike:** `AssetContextDto` shape (BS-19) at build.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-002-T01 | `AssetContextDto` shape spike (resolve BS-19) | Backend | Research | 6 | 4 | TS-2 |
| US-002-T02 | `GET /assets/{id}/context` + CrowdStrike/Intune mapping | Backend | Code | 14 | 5–6 | TS-2 |
| US-002-T03 | Asset context screen + degraded states | Frontend | Code | 12 | 6–7 | TS-3 |
| US-002-T04 | Contract tests vs vendor fixtures | QA | Test | 8 | 7 | TS-3 |
| US-002-T05 | Add `criticality` + `environment` (prod/staging/dev) fields to ASSET schema + inference sources (CMDB tags, cloud resource tags, naming convention) — competitor-scan CS-16, claims-priority HIGH (2026-07-11) | Data | Code | 14 | 6 | TS-2 |
| US-002-T06 | Splunk connector (SIEM process/runtime telemetry — e.g. listening-process evidence for `AssetContextWorker`) — closes a documentation/build gap found in the 2026-07-12 traceability audit: `catalogs.md`'s `splunk` row and `security-stepper.md`'s asset-context orchestration referenced this as live with no backing task | Backend | Code | 16 | 6–7 | TS-1 |

### US-003 — Protection Breakdown (Gate 1, promoted) · 8 pts · persona: security engineer
**ACs:** four-state control summary + vendor control panels; `ProtectionProjection` p95 Gate-1 budget. **DoD:** merge gates + Wiz/CrowdStrike ingest live.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-003-T01 | `vendor-contract.ts` base + role marker interfaces (Liskov set incl. R2 roles) | Backend | Code | 16 | 3–4 | TS-2 |
| US-003-T02 | CrowdStrike connector (ingest; controls + runtime) | Backend | Code | 20 | 4–6 | TS-1 |
| US-003-T03 | Wiz connector (scanner findings, 6 h cadence) | Backend | Code | 16 | 5–6 | PY-2 |
| US-003-T04 | `ProtectionProjection` + four-state panel UI | Frontend | Code | 14 | 6–7 | TS-3 |
| US-003-T05 | Vendor-connector RLS + parity tests | QA | Test | 10 | 7 | TS-3 |
| EP-02-F02-T06 | Connector credential encryption: AES-256 JSONB + SSM/Vault transit for OAuth (costed spine) | Security | Code | 10 | 4–5 | TS-2 |
| EP-02-F02-T07 | Scoped action credentials (H3/D-8): least-privilege write scope limited to the 5 canonical actions; per-action short-lived minting where vendor supports | Security | Code | 12 | 7–8 | SEC |

## EP-02-F03 — Ownership inference

**Parent:** EP-02 · **FR:** FR-016 · **ACs:** [security-stepper](../10-product/features/security-stepper.md) US-007; `ownership.inferred` webhook; Gate 1 promoted (read-only) · **Dependencies:** ServiceNow or Entra connector (F02 pattern) · **API boundary:** `OwnershipInferenceWorkflow` · **SLA:** inference on sync delta.

### US-007 — Ownership Inference (Gate 1, promoted) · 8 pts · persona: security engineer
**ACs:** owner suggestion with certainty score + evidence; `test:ownership-inference`. **DoD:** merge gates + webhook event.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-007-T01 | ServiceNow/Entra ID connector (ownership evidence ingest) | Backend | Code | 18 | 5–6 | TS-1 |
| US-007-T02 | `OwnershipInferenceWorkflow` (Temporal) + certainty scoring | Backend | Code | 16 | 6–7 | PY-1 |
| US-007-T03 | `OWNERSHIP_EVIDENCE` schema + `world_model_versions` bump | Data | Code | 8 | 5 | TS-2 |
| US-007-T04 | Ownership UI surfacing + `ownership.inferred` webhook | Frontend | Code | 10 | 7–8 | TS-3 |
| US-007-T05 | `test:ownership-inference` suite | QA | Test | 8 | 8 | TS-3 |
| US-007-T06 | Broaden auto-tagging beyond `owner_team` to environment/criticality/patch_window/business_unit/compliance_scope inference (depends on US-002-T05 asset schema fields) — competitor-scan CS-25, claims-priority MEDIUM (2026-07-11) | Backend | Code | 18 | 8 | PY-1 |

**Deferred, not scheduled (competitor-scan disposition, 2026-07-11):** timed escalation to a manager after N hours unassigned when ownership certainty is low, beyond the current `INSUFFICIENT_DATA` + manual-assign CTA (CS-26); "% auto-routed without human intervention" ops metric, alongside existing certainty-distribution/routing-accuracy/time-to-assignment metrics (CS-29). Both accepted-but-deferred — no tasks until scheduled.

## EP-02-F04 — Optional physical residency — **Deferred (Gate 5)**

**Parent:** EP-02 · **FR:** FR-014 · Story US-020 (draft) — no tasks until a signed on-prem contract exists ([connector-hub US-020](../10-product/features/connector-hub.md)).

## EP-02-F05 — Proactive tool discovery

**Parent:** EP-02 · **FR:** FR-029 · **ACs:** discover a tenant's already-installed security tools beyond currently-configured connectors (read-only, scoped to the same OAuth/IAM boundary as F01/F02), surface as a "you already own this — enable it" mitigation-advisor recommendation angle · **Dependencies:** F01 Connector Hub credential model, F02 vendor-contract role interfaces · **API boundary:** `GET /connectors/discovered` (candidate, read-only suggestion list) · **SLA:** best-effort, non-blocking. Added 2026-07-11 per competitor-scan disposition CS-23 (accepted, MEDIUM claims priority).

### US-027 — Proactive Tool Discovery (Gate 2 candidate) · 8 pts · persona: security engineer
**ACs:** surfaces a candidate list of already-installed security tools (e.g. cloud-native WAF/IPS, IAM policies) inferred from existing connector read scope, without requiring a new connector to be configured; never auto-enables anything — suggestion only, human opts in. **DoD:** merge gates + discovery-suggestion integration test.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-027-T01 | Tool-signature detection rules (cloud resource tags/config API responses that indicate an installed-but-unconfigured security tool) | Backend | Research | 12 | 9 | PY-2 |
| US-027-T02 | `GET /connectors/discovered` read-only suggestion endpoint | Backend | Code | 14 | 9–10 | TS-1 |
| US-027-T03 | "You already own this" suggestion card UI on Connector Hub | Frontend | Code | 10 | 10 | TS-3 |
| US-027-T04 | Discovery-suggestion integration tests | QA | Test | 8 | 10–11 | TS-3 |

**EP-02 total: 434 h** (304 base + 26 costed spine + 12 H3 + 14 CS-16 + 18 CS-25 + 44 CS-23/US-027 + 16 US-002-T06 Splunk — corrected 2026-07-12: the 2026-07-11 total (386h) mis-added its own components, which actually sum to 418h before the Splunk task; +16 Splunk brings the verified total to 434h)

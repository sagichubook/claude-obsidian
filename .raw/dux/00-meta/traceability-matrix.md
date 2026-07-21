---
owner: Founder
status: canonical
gate: 1
last_reviewed: 2026-07-20
decisions: [D-34, D-41]
---

# Traceability Matrix — Business Requirement → Epic → User Story

**Purpose:** the join table for the whole corpus. It binds BR ↔ Epic ↔ US ↔ FR ↔ NFR/TR-NFR ↔ Gate ↔ feature spec ↔ verification command.

**The chain:**

```
Vision anchor → BR (business requirement)
              → Epic (delivery theme; groups the corpus's FRs)
                → US (user story)
                  → FR / NFR → TR-NFR (technical spec)
```

Gates reflect the ideal-state canon (GCIS v2.2).

**Every level references its parent, and the feature specs in `10-product/` cite these IDs back.**

> **The references are bidirectional by design, and the validator does not check them.** A US dropped from a BR's story list, or a BR dropped from an epic's parent list, **silently breaks the chain** — that exact defect has been found here before, by hand. **Keep both directions in sync.**

**This file owns the closed US set.** Stories are defined here, and never invented in `90-execution/`.

## 1. Epics

Epics are derived groupings of the corpus's FR-001…FR-026. The corpus tracked BR→FR→US directly; epic IDs are introduced here for the requested BR→Epic→US chain and used consistently across the rewritten docs.

| Epic | Name | FRs | Parent BRs |
|------|------|-----|-----------|
| EP-01 | Multi-tenant platform & auth | FR-001, FR-009 | BR-001 |
| EP-02 | Environmental data ingestion (connectors & feeds) | FR-002, FR-003, FR-015, FR-016, FR-020, FR-014, FR-029 | BR-004 |
| EP-03 | Exploitability assessment engine | FR-004, FR-017, FR-026, FR-027, FR-030 | BR-002, BR-007, BR-013 |
| EP-04 | Continuous re-assessment | FR-025 | BR-002 |
| EP-05 | Analyst surfaces & APIs (dashboards, drill-down, chat) | FR-006, FR-007 | BR-002, BR-006, BR-008, BR-010 |
| EP-06 | Mitigation & remediation write path (HITL) | FR-011, FR-012, FR-013, FR-019 | BR-002, BR-003 |
| EP-07 | Safety & governance (kill switch, kernel, audit) | FR-005, FR-008 | BR-003, BR-005, BR-009 |
| EP-08 | Programmatic platform (webhooks, public data API) | FR-010, FR-021, FR-022, FR-023, FR-028 | BR-006, BR-011 |
| EP-09 | Triage disposition (acknowledgment) | FR-024 | BR-012 |
| EP-10 | Personalization (preference learning) | FR-018 | BR-002 |

## 2. Business requirements → epics → user stories

| BR | Business requirement | Vision anchors | Epics | User stories | Gate | Verification |
|----|----------------------|----------------|-------|--------------|------|--------------|
| BR-001 | Zero cross-tenant data leakage for findings, assets, assessments, users, exports | D5 (secondary — see BR-003 for primary anchor); tenant trust | EP-01 | US-014 (+ every tenant-scoped view) | Gate 1 | `pnpm test:fuzz-tenant-id`; ISO-001–010; ISO-FUZZ-001–005 |
| BR-002 | Agentic exploitability analysis distinguishing truly exploitable from noise; evidence-backed prioritization, mitigation paths, remediation acceleration | A1–A9, B2–B7, C1–C7, D1–D3, E2 | EP-03, EP-04, EP-05, EP-06, EP-10 | US-001, US-003–005, US-008–012, US-016–017, US-021; US-018–019; US-025 (draft, deferred) | Gate 1 (Analyze + unattended-by-default writes + continuous); Gate 2c (US-009); Gate 3 (US-019 closed-loop validation only); post-Gate-3 candidate (US-025) | Golden set; trace export; DeepEval CI |
| BR-003 | Emergency agent halt (kill switch) with governance-kernel enforcement; <5 s L2–L4 | B1, D4, D5 (primary — action allowlist enforces defensive-only scope, not offensive execution) | EP-06, EP-07 | US-014 (visibility); all agent paths | Pre-launch | KS-001; `test:kill-switch`; `test:governance-kernel` (GOV-001–013) |
| BR-004 | Multi-source environmental data ingestion (AWS, NVD/KEV/EPSS, CSV, vendor connectors) | B2 step 1, B6 | EP-02 | US-002, US-003, US-007, US-011, US-013; US-020 (draft); US-027 | Gate 1 (AWS + intel + ≥3 vendor connectors per ADR-011 R2); Gate 5 (FR-014); Gate 2 candidate (US-027) | Connector sync tests; CONN-001 |
| BR-005 | Tamper-evident hash-chained audit trail + export | C6 | EP-07 | US-006, US-014, US-017 | Gate 1 | `/audit/verify`; trace export |
| BR-006 | Programmatic access — REST, SSE chat, webhooks, rate limits | B2 step 4 | EP-05, EP-08 | US-008, US-014, US-015; US-026 (draft, deferred) | Gate 1; no near-term trigger (US-026) | Rate-limit + webhook HMAC tests |
| BR-007 | AI-BOM manifest accuracy (agent tools, models, MCP catalog) | D4 | EP-03 | US-008 | Gate 1 | `aibom-validate` CI |
| BR-008 | Platform availability SLOs for enterprise trust | C1–C2 | EP-05 | US-006, US-012 | Gate 1 | SLO burn-rate alerts staging/production |
| BR-009 | AI-BOM drift detection in CI — no undeclared agent capabilities | D4 | EP-07 (CI gate; no US) | — | Ongoing | Manifest drift merge gate |
| BR-010 | Design partner cohort (2+ NDA) for Phase 1 validation | E1 | EP-05 | US-010 | Gate 1 | CRM; ≥10 assessments in first 14 days |
| BR-011 | Tenant-defined custom metrics over World Model via DQL; public read API | B2 step 4 | EP-08 | US-022, US-024 | Seed | OpenAPI contract tests; DQL validation |
| BR-012 | Vulnerability-instance risk acknowledgment (create/expire/revoke) with audit | Triage outcomes | EP-09 | US-023 | Phase 1 write; Seed public read | Ack lifecycle tests; audit events |
| BR-013 | Predictive risk forecasting — surface assets trending toward higher risk via evidence-trend deltas (EPSS/finding-count/control-coverage), not a new ML model or composite score | E4 | EP-03 | US-028 | Gate 2 | Trend-computation tests against fixture EPSS-history/finding/control deltas |

## 3. User story index (US → parent epic/BR → spec)

| US | Title (screen) | Epic | BR | FR | Gate (canon) | Feature spec |
|----|----------------|------|----|----|--------------|--------------|
| US-001 | Prerequisites Analysis | EP-03 | BR-002 | FR-004 | Gate 1 live | `10-product/features/security-stepper.md` |
| US-002 | Asset Context Evidence | EP-02 | BR-004 | FR-020 | **Gate 1** (read-only, promoted) | `10-product/features/security-stepper.md` |
| US-003 | Protection Breakdown | EP-02 | BR-002/004 | FR-002 | **Gate 1** (read-only, promoted) | `10-product/features/security-stepper.md` |
| US-004 | Action Cards | EP-06 | BR-002 | FR-011 | **Gate 1, unattended by default** | `10-product/features/mitigation-write-path.md` |
| US-005 | Control Refinements | EP-06 | BR-002 | FR-019 | **Gate 1** (promoted) | `10-product/features/security-stepper.md` |
| US-006 | Audit & Exposure Delta | EP-07 | BR-005/008 | FR-008 | Gate 1 live | `10-product/features/dashboard-audit.md` |
| US-007 | Ownership Inference | EP-02 | BR-004 | FR-016 | **Gate 1** (read-only, promoted) | `10-product/features/security-stepper.md` |
| US-008 | Chat Guidance | EP-05 | BR-002/006/007 | FR-007 | Gate 1 (read-only tools; write tools per HITL schedule) | `10-product/features/chat-guidance.md` |
| US-009 | Preference Learning | EP-10 | BR-002 | FR-018 | Gate 2c (**not** promoted — needs behavioral data) | `10-product/features/security-stepper.md` |
| US-010 | Research Dashboard (Mitigation nav) | EP-05 | BR-002/010 | FR-004/006 | Gate 1 live | `10-product/features/research-dashboard.md` |
| US-011 | Exposure Analysis (Exposure nav) | EP-05 | BR-002/004 | FR-004/006 | Gate 1 live | `10-product/features/exposure-analysis.md` |
| US-012 | Dashboard Home | EP-05 | BR-008 | FR-006 | Gate 1 live | `10-product/features/dashboard-audit.md` |
| US-013 | Connector Hub (Apps nav) | EP-02 | BR-004 | FR-002/003/015 | Gate 1 live (AWS P0; CSV P1; vendor connectors live per ADR-011 R2) | `10-product/features/connector-hub.md` |
| US-014 | Tenant Settings | EP-01/07/08 | BR-001/003/005/006 | FR-001/005/009/010 | Gate 1 (users + agent policy P0; keys/webhooks P1) | `10-product/features/tenant-settings.md` |
| US-015 | Help & Support | EP-05 | BR-006 | — | Phase 1 exit (P1; not a Gate 1 blocker) | `10-product/features/tenant-settings.md` |
| US-016 | Fast Actions | EP-06 | BR-002 | FR-011 | **Gate 1, `POST /fast-actions` unattended by default** | `10-product/features/mitigation-write-path.md` |
| US-017 | Assessment Trace (+ execution results) | EP-03 | BR-002/005 | FR-004/026 | **Gate 1 incl. `execution_results`** (promoted) | `10-product/features/assessment-trace.md` |
| US-018 | Remediation Ticket Panel | EP-06 | BR-002 | FR-013 | **Gate 1 create+route, unattended by default**; closed-loop automation still Gate 3 (US-019) | `10-product/features/mitigation-write-path.md` |
| US-019 | Mitigation Validation Panel (draft) | EP-06 | BR-002 | FR-012 | Gate 3 | `10-product/features/mitigation-write-path.md` |
| US-020 | Physical Residency Admin (draft) | EP-02 | BR-004 | FR-014 | Gate 5 (future/roadmap) | `10-product/features/connector-hub.md` |
| US-021 | Continuous Re-assessment Scheduler | EP-04 | BR-002 | FR-025 | **Gate 1** (promoted; ADR-016) | `10-product/features/continuous-assessment.md` |
| US-022 | Custom Metrics Configuration | EP-08 | BR-011 | FR-021 | Seed | `30-api/public-data-api.md` |
| US-023 | Acknowledge Vulnerability Instance | EP-09 | BR-012 | FR-024 | Phase 1 write; Seed public read | `10-product/features/research-dashboard.md` |
| US-024 | Vulnerability Instances by CVE | EP-08 | BR-011 | FR-022/023 | UI Gate 1 (`by_instance`); public API Seed | `30-api/public-data-api.md` |
| US-025 | Outcome Learning (draft) | EP-03 | BR-002 | FR-027 | Post-Gate-3 candidate (draft, deferred) | `90-execution/backlog-ep03.md` §EP-03-F03 |
| US-026 | Outbound MCP Server (draft) | EP-08 | BR-006 | FR-028 | No near-term trigger (draft, deferred) | `90-execution/backlog-ep08.md` §EP-08-F03 |
| US-027 | Proactive Tool Discovery | EP-02 | BR-004 | FR-029 | Gate 2 candidate | `90-execution/backlog-ep02.md` §EP-02-F05 |
| US-028 | Asset Risk Trend Forecast | EP-03 | BR-013 | FR-030 | Gate 2 | `10-product/features/predictive-risk-forecasting.md` |

Added 2026-07-11 (US-025/026/027) per competitor-scan disposition (competitor-scan-2026-07-10-dux-plus.md, since retired — see `decisions-log.md` for the promotion record).

## 4. FR → implementation (component / verification / gate)

| FR | Component / ADR | Verification | Gate (canon) |
|----|------------------|--------------|--------------|
| FR-001 | ADR-001/002; Better Auth via `AuthPort`; multi-tenancy spec | `test:fuzz-tenant-id`; AUTH-001–005; ISO-001–010 | Gate 1 |
| FR-002 | ADR-004 `AWSConnector`; ADR-011 R2 vendor connectors | Connector sync integration tests; <48 h onboarding metric | Gate 1 |
| FR-003 | `NVDConnector` (delta ≤2 h, ≥6 s sleep), `CISAConnector` (6 h; 1 h Enterprise), `EPSSConnector` (daily) | CONN-001 staleness | Gate 1 |
| FR-004 | ADR-007 R3 `ExploitabilityAssessmentWorkflow` (self-hosted Temporal); CaMeL plane; `POST /research/queue` idempotent | Golden set; DeepEval; TR-NFR-005 | Gate 1 |
| FR-005 | `KillSwitchRelay`; kill-switch spec | KS-001–007; `test:kill-switch-idempotency` | Pre-launch |
| FR-006 | `ExposureProjection` / `ProtectionProjection` / `ActionCardProjection`; SSE streams | US-010 SSE <1 s; US-012 <5 s | Gate 1 |
| FR-007 | Chat SSE+POST; `chat_interface` flag; MCP gateway | US-008 phased delivery; `aibom-validate` | Gate 1 |
| FR-008 | `AuditVerificationService`; hash-chained partitions | `/audit/verify` | Gate 1 |
| FR-009 | `TenantExportWorkflow`; GDPR delete | TR-NFR-009 (<24 h) | Gate 1 (P1) |
| FR-010 | ADR-005 R2 `WebhookDeliveryWorker` (NATS JetStream-backed durable queue) | HMAC tests; DLQ replay | Gate 1 (P1) |
| FR-011 | `QuickMitigationWorkflow` via `VendorActionGate`; T2/T3 tiers classify for audit + anomaly escalation (ADR-012 R3) | `test:vendor-action-hitl`; `test:governance-kernel` | **Gate 1, unattended by default**; closed-loop validation Gate 3 |
| FR-012 | `ClosedLoopValidationWorkflow` saga | `closed_loop_validation` flag | Gate 3 |
| FR-013 | `RemediationWorkflow` create+route (T1 tier; unattended); ServiceNow/Jira | `test:remediation-ticket-create`; audit event | **Gate 1 create+route**; closed loop Gate 3 |
| FR-014 | `dux-resident-agent` DaemonSet | `test:physical-resident-agent-isolation` | Gate 5 |
| FR-015 | `CSVConnector` (UTF-8; 50 K rows; unique tenant+hostname) | Schema validation tests | Gate 1 (P1) |
| FR-016 | `OwnershipInferenceWorkflow`; ServiceNow/Entra (ADR-011 R2) | `test:ownership-inference`; `ownership.inferred` webhook | **Gate 1** |
| FR-017 | `EvaluationAgent` — 10% stratified drift sample vs 30-day baseline | Week-10 sprint; 2σ alert | Gate 1 |
| FR-018 | `PreferenceEngine`; `POST /preferences` | Gate 2c acceptance | Gate 2c |
| FR-019 | `ControlRefinementQuery` (Specification pattern); `ControlRefinementAggregate` | Qualys/Wiz ingest live | **Gate 1** |
| FR-020 | `GET /assets/{id}/context`; CrowdStrike/Intune ingest | Standalone US-002 screen | **Gate 1** |
| FR-021 | `GET /v1/custom-metrics*`; DQL engine | OpenAPI parity; size ≤200 | Seed |
| FR-022 | `GET /v1/vulnerability-instances/{cve_id}` cursor pagination | limit ≤5000; `expand=asset`; cross-tenant 404 | Seed |
| FR-023 | `POST /v1/cve-research` batch 1–50 | `cve_research.completed` webhook | Seed |
| FR-024 | `VulnerabilityInstanceAckService`; auto-expire job | Create/revoke/expire tests | Phase 1 |
| FR-025 | `ReassessmentSchedulerWorkflow` (self-hosted Temporal, ADR-016 R2); NATS core pub/sub event bus; `ReassessmentDebouncer` (15-min coalesce); evidence-hash dirty-check | `test:reassessment-trigger`; `test:reassessment-dirtycheck` | **Gate 1** |
| FR-026 | `SandboxPort` → `SelfHostedFirecrackerAdapter` (self-hosted, K8s, ADR-015 R4); `ScriptSecurityScanner` AST pre-scan | `test:sandbox-execution-isolation`; `test:script-ast-scan-reject`; `execution_results` populated assertion | **Gate 1** |
| FR-031 | Agentic RAG + Apache AGE graph retrieval ([ADR-020 R2](../20-architecture/adr-index.md#adr-020-r2--agentic-rag-and-graph-retrieval), D-34) — FR ID assigned 2026-07-20 (D-41). (The RAG-flip's live-safety-test precondition is a separate, still-open question — see [OI-41](open-items.md).) | — | — |
| FR-027 | Outcome-learning feedback loop (draft) — track remediation outcome per `asset_vuln`, feed into exploitability-score confidence + mitigation ranking | Not yet specced (deferred) | Post-Gate-3 candidate |
| FR-028 | Outbound MCP server (draft) — expose Dux capabilities to third-party MCP clients | Not yet specced (deferred) | No near-term trigger |
| FR-029 | `GET /connectors/discovered` (draft) — proactive discovery of a tenant's already-installed security tools beyond configured connectors | Discovery-suggestion integration tests | Gate 2 candidate |
| FR-030 | `GET /assets/risk-trend` — per-asset `rising`/`stable`/`falling` trend from EPSS/finding-count/control-coverage deltas | Trend-computation tests against fixture deltas | Gate 2 |

## 5. NFR crosswalk (PRD → TRD)

| NFR | Requirement | TR-NFR | Target / measurement |
|-----|-------------|--------|----------------------|
| NFR-001 | Zero cross-tenant reads | TR-NFR-002 | ISO-001–010, 100% CI |
| NFR-002 | API latency | TR-NFR-004 | p95 < 300 ms |
| NFR-003 | Assessment start | TR-NFR-005 | p95 < 2 s |
| NFR-004 | Graph query | TR-NFR-006 | 3-hop CTE p95 < 200 ms at >2 K assets (migration trigger: >150 ms @ >1 K assets, 7 days) |
| NFR-005 | Kill switch | TR-NFR-003 | < 5 s p99 (L2–L4) |
| NFR-006 | GDPR export/delete | TR-NFR-009 | export < 24 h; purge ≤ 90 d |
| NFR-007 | Accessibility | TR-NFR-010 | WCAG 2.2 AA; axe-core 0 violations |
| NFR-008 | Assessment quality | TR-NFR-007/014 | golden-set regression < 2% (P0 block); 100% instrumented LLM paths |
| NFR-009 | Code-backed audit retention | TR-NFR-011 | trace + code (+ execution results at Gate 1 per canon) per assessment |
| NFR-010 | Agent context depth | TR-NFR-012 | 128 K tokens; checkpoint at 80% |
| NFR-011 | Per-tenant LLM cost cap | TR-NFR-013 | enforced before intervention ($25/h default) |
| NFR-012 | API availability | TR-NFR-001 | 99.5% monthly excl. LLM providers |
| NFR-013 | Exposure drill-down | TR-NFR-015 | p95 < 500 ms at 1 K assets |
| — (ops) | Feature-flag eval | TR-NFR-008 | SDK p99 < 20 ms; Unleash API 99.9% |
| — (ops) | Valkey cache effectiveness | TR-NFR-016 | `dux_valkey_hit_rate` (`llm_response` type) ≥ 0.6 |
| — (ops) | NATS JetStream backlog | TR-NFR-017 | `dux_nats_consumer_lag` < 1,000 (`VULNERABILITIES`); < 5,000 (any stream) |

## 6. Non-FR traceability rows

| Capability | BR | Component | Gate |
|-----------|----|-----------|------|
| Trust/status portals | BR-005 (comms trust) | Cloudflare DNS + static shell | **Launch blocker** (GCIS §E) |
| Governance kernel GOV-001–013 | BR-003 | `packages/core/governance/` | Pre-launch / Gate 1 |
| Agent identity + registry | BR-003/007 | JWT SPIFFE claims; agent-as-directory (ADR-009) | Gate 1; SPIRE PoC Month 3 |
| Compliance programs | — | SOC 2 (seed trigger → Type I M9–12 → Type II Series A), ISO 27001/42001 (Series A), EU AI Act Art. 9 (Aug 2026) | Stage triggers |

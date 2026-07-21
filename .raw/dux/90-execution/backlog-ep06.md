---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-7, D-8, D-15, D-17, D-19]
---

# EP-06 — Mitigation & Remediation Write Path (unattended by default)

**Objective:** BR-002/003 — GCIS G3 "full pipeline at machine speed": Analyze→Mitigate→Remediate exists at Gate 1, unattended by default · **Metrics:** `test:vendor-action-hitl`, `test:governance-kernel`, `test:remediation-ticket-create`; anomaly-escalation approve/deny surface live by Gate 1 review Week 12 (D-4 as amended 2026-07-13, [D-7 R1](../00-meta/decisions-log.md)) · **Target:** Gate 1 (Week 12): unattended action cards + fast actions + ticket create/route; Gate 3: closed-loop validation (US-019) · **Priority:** P0 (Gate-1 blocker via D-4) · **Constraints:** ADR-012 R3 (`VendorActionGate`, canonical→native mapping, T1–T3 tiers), gate chain Intent→Budget→Effect→VendorAction→HITL (HITLGate defaults `continue`), `mutation_key` step-effect idempotency, `rollbackProcedure` URL required in audit/HITL payload (E-4 rollback catalog: authored — [D-15](../00-meta/decisions-log.md), OI-03 closed).

## EP-06-F01 — Action cards & fast actions (unattended by default)

**Parent:** EP-06 · **FR:** FR-011 · **ACs:** [mitigation-write-path spec](../10-product/features/mitigation-write-path.md) US-004/US-016; canonical actions `endpoint.isolate`(T3) `network.blocklist_add`(T2) `policy.deploy_device_config`(T2) `patch.deploy_special_devices`(T3) · **Dependencies:** EP-07 F02 (kernel + HITL contract), CrowdStrike/Intune connectors (EP-02) · **API boundary:** `QuickMitigationWorkflow` via `VendorActionGate`; `POST /fast-actions` ships at Gate 1 · **SLA:** KS-L2 blocks enqueue.

### US-004 — Action Cards (Gate 1, unattended by default) · 8 pts · persona: security engineer
**ACs:** action cards with vendor deep-links + `canonical_action_id`; unattended execution through `VendorActionGate` (T2/T3 tiers classify for audit + anomaly escalation); audit `VendorActionExecution`. **DoD:** merge gates + `test:vendor-action-hitl`.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-004-T01 | `VendorActionGate` (canonical→native mapping, persist both in audit) | Backend | Code | 16 | 6–7 | TS-1 |
| US-004-T02 | `QuickMitigationWorkflow` + step-effect idempotency (`mutation_key` + resume reconciliation) | Backend | Code | 16 | 7–8 | TS-1 |
| US-004-T03 | Rollback catalog for 5 canonical actions — authored ([D-15](../00-meta/decisions-log.md); E-4/OI-03 closed) — required HITL field | Security | Research | 10 | 7–8 | SEC |
| US-004-T04 | Action-card UI on drill-down (CTA + HITL state) | Frontend | Code | 14 | 8–9 | TS-3 |
| US-004-T05 | `test:vendor-action-hitl` + audit-record assertions | QA | Test | 10 | 9 | TS-3 |
| US-004-T06 | HITL impact preview (H4/D-8): render `affectedResources[]` + `blastRadius` + reversibility + rollback link in the approve/deny surface — Gate-1 AC; approval without impact visibility fails the AC | Frontend | Code | 10 | 9–11 | TS-3 |
| US-004-T07 | Extend compensating-control discovery to IAM restrictions + network segmentation/WAF sources (beyond current EDR + cloud-FW/Intune) — competitor-scan CS-20, claims-priority MEDIUM (2026-07-11) | Backend | Code | 14 | 9 | TS-2 |

**Deferred, not scheduled (competitor-scan disposition, 2026-07-11):** explicit stopgap-vs-permanent classification + target-patch-date field on mitigation/action records, beyond current blast-radius/HITL-tier/Gate typing (CS-22). Accepted-but-deferred — no task until scheduled; touches the vendor action catalog (`10-product/catalogs.md`) when built.

### US-016 — Fast Actions (Gate 1, unattended by default) · 5 pts · persona: security engineer
**ACs:** one-click lightweight mitigations via `POST /fast-actions` → `QuickMitigationWorkflow`, executing unattended; manual action cards (copy SIEM query, ticket text, numbered steps) remain as fallback; nav icon enabled at Gate 1. **DoD:** merge gates + audit + webhook on mitigation execution.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-016-T01 | Manual action-card content engine (SIEM query / ticket text / steps) | Backend | Code | 12 | 8–9 | TS-2 |
| US-016-T02 | Fast Actions nav + card UI | Frontend | Code | 10 | 9 | TS-3 |
| US-016-T03 | `mitigation.executed/blocked` webhooks + audit trail | Backend | Code | 6 | 9 | TS-2 |
| US-016-T04 | Manual-flow E2E test (incl. KS-L2 block) | QA | Test | 6 | 9–10 | TS-3 |

## EP-06-F02 — Control refinements — **Deferred (Gate-1 capacity fallback, D-19)**

**Parent:** EP-06 · **FR:** FR-019 · **ACs:** [security-stepper US-005](../10-product/features/security-stepper.md); refinement recommendations from existing stack (T2 HITL) · **Dependencies:** Qualys/Wiz ingest (EP-02 F02) · **API boundary:** `GET /controls/refinements`; `ControlRefinementQuery` (Specification pattern) · **SLA:** Gate 2.

### US-005 — Control Refinements · 5 pts · persona: security engineer

No tasks scheduled at Gate 1 — deferred to the Gate-2 backlog (D-19 capacity fallback; 30 h). Spec ready: `ControlRefinementQuery`/`ControlRefinementAggregate`, refinements API + stepper panel UI, tests vs Wiz/Qualys fixtures.

**Deferred, not scheduled (competitor-scan disposition, 2026-07-11):** upgrade config-change recommendations from generic named settings to syntax-level file diffs (sshd_config/nginx/security-group/K8s-netpol) + sandbox pre-validation of the proposed diff, beyond current generic guidance (CS-21). Accepted-but-deferred — no task until scheduled.

## EP-06-F03 — Remediation tickets (create + route)

**Parent:** EP-06 · **FR:** FR-013 · **ACs:** [mitigation-write-path US-018](../10-product/features/mitigation-write-path.md); ServiceNow ticket w/ assignee + SLA, unattended (T1 tier, lowest blast radius); webhook status updates; closed-loop Gate 3 · **Dependencies:** ServiceNow connector (EP-02), EP-07 HITL · **API boundary:** `RemediationWorkflow`; `ticket.create_remediation` canonical action; webhooks `remediation.ticket_created`, `ticket.*` · **SLA:** retry saga on write failure; KS-L2 blocks creation.

### US-018 — Remediation Ticket Panel (Gate 1 create+route, unattended, T1 tier) · 8 pts · persona: security engineer

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-018-T01 | `RemediationWorkflow` create+route (T1 tier; retry saga) | Backend | Code | 16 | 8–9 | TS-1 |
| US-018-T02 | ServiceNow ticket action adapter (`ticket.create_remediation`) | Backend | Code | 12 | 8–9 | TS-2 |
| US-018-T03 | Ticket panel UI (from Exposure + Chat; assignee/SLA display) | Frontend | Code | 12 | 9–10 | TS-3 |
| US-018-T04 | Inbound ticket-status webhooks (`ticket.created/updated/resolved/reopened`) | Backend | Code | 8 | 9–10 | TS-2 |
| US-018-T05 | `test:remediation-ticket-create` + audit event | QA | Test | 8 | 10 | TS-3 |
| US-018-T06 | Severity-tiered remediation routing: Critical/actively-exploited findings also page via existing internal Slack/PagerDuty plumbing (`kill-switch-hitl.md` on-call/approval channels), reused outward — not just the single ServiceNow ticket path — competitor-scan CS-27, claims-priority HIGH (2026-07-11) | Backend | Code | 16 | 10 | TS-2 |

## EP-06-F04 — Closed-loop validation — **Deferred (Gate 3)**

**Parent:** EP-06 · **FR:** FR-012 · Story US-019 (draft) — `ClosedLoopValidationWorkflow` behind `closed_loop_validation` flag; no tasks until Gate-3 safety record exists. **Scope expanded 2026-07-11** per competitor-scan disposition to explicitly cover two additional deferred angles, both accepted-but-deferred: (a) pre-approval sandbox dry-run — simulate a proposed remediation in sandbox before HITL approval executes it live, complementing rather than replacing HITL (CS-02); (b) post-mitigation sandbox re-validation — re-execute the exploit against the mitigated configuration to verify it's actually blocked, beyond the current exposure-state/residual-count delta check (CS-24). Both fold into `ClosedLoopValidationWorkflow`'s eventual scope; no tasks until Gate-3 trigger.

**EP-06 total: 196 h** (156 base, net of the 30 h US-005 deferral, D-19 + 10 H4 + 14 CS-20 + 16 CS-27 — matches the base figure in [traceability §rollup](traceability.md))

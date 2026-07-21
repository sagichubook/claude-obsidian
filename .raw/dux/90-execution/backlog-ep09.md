---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-7, D-8, D-19]
---

# EP-09 — Triage Disposition (Acknowledgment) — **Deferred (Gate-1 capacity fallback, D-19)**

**Objective:** BR-012 risk-acknowledgment lifecycle with audit · **Metrics:** create/revoke/expire tests; `vulnerability_instance.acknowledged` + `acknowledgment_expired` events · **Target:** Gate-2 write; Seed public read of `is_acknowledged` · **Priority:** P1 · **Constraints:** ack = active (non-revoked, unexpired); audit every transition.

## EP-09-F01 — Acknowledgment lifecycle — **Deferred (Gate-1 capacity fallback, D-19)**

**Parent:** EP-09 · **FR:** FR-024 · **ACs:** [research-dashboard US-023](../10-product/features/research-dashboard.md); `POST /vulnerability-instances/{id}/acknowledge` (+revoke); auto-expire job · **Dependencies:** EP-05 F01 (instance rows) · **API boundary:** `VulnerabilityInstanceAckService` · **SLA:** expiry sweep daily.

### US-023 — Acknowledge Vulnerability Instance · 3 pts · persona: security engineer

No tasks scheduled at Gate 1 — deferred to the Gate-2 backlog (D-19 capacity fallback; 30 h). Spec ready: `VulnerabilityInstanceAckService` (create w/ reason + optional expiry, revoke), auto-expire job + `acknowledgment_expired` event, ack UI on queue/drill-down rows, lifecycle tests.

**EP-09 total: 0 h (deferred)**

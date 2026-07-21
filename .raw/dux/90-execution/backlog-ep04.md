---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-7, D-8]
---

# EP-04 — Continuous Re-assessment

**Objective:** BR-002 "continuous/24-7" claim engineering-true at Gate 1 (GCIS G1) · **Metrics:** `test:reassessment-trigger` + `test:reassessment-dirtycheck`; most triggers resolve without LLM call (ADR-016 cost control) · **Target:** Gate 1 (Week 12, [D-7 R1](../00-meta/decisions-log.md)) · **Priority:** P0 (claim-safety blocker) · **Constraints:** ADR-016 R2 (event + scheduled + dirty-check), NATS core pub/sub event bus, `ReassessmentDebouncer` 15-min coalesce per (tenant, cve, asset), cached World-Model prefix reuse.

## EP-04-F01 — Continuous assessment engine

**Parent:** EP-04 · **FR:** FR-025 · **ACs:** [continuous-assessment spec](../10-product/features/continuous-assessment.md): KEV/EPSS delta, connector delta (`world_model_versions` bump), scheduled sweep (24 h default, tenant-configurable) all trigger re-assessment; `assessment.requeued` webhook · **Dependencies:** EP-03 F01 (workflow), EP-02 (delta events) · **API boundary:** `ReassessmentSchedulerWorkflow`, `POST/GET /research/schedule` · **SLA:** debounce window 15 min; re-assessment reuses cached prefix.

### US-021 — Continuous Re-assessment Scheduler (Gate 1, promoted) · 8 pts · persona: security engineer
**ACs:** three trigger classes live; evidence-hash dirty-check skips clean re-runs; queue rows update via SSE. **DoD:** merge gates + both reassessment test suites.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-021-T01 | `ReassessmentSchedulerWorkflow` (Temporal durable schedules; tenant-configurable sweep) | Backend | Code | 16 | 6–7 | TS-1 |
| US-021-T02 | NATS event bus wiring: KEV/EPSS delta + connector `world_model_versions` bump events | Backend | Code | 14 | 6–7 | TS-2 |
| US-021-T03 | `ReassessmentDebouncer` (15-min coalesce per tenant/cve/asset) | Backend | Code | 8 | 7 | TS-2 |
| US-021-T04 | Evidence-hash dirty-check (skip P-LLM when evidence unchanged) | Backend | Code | 12 | 7–8 | PY-1 |
| US-021-T05 | Schedule API + settings UI row + `assessment.requeued` webhook | Frontend | Code | 8 | 8 | TS-3 |
| US-021-T06 | `test:reassessment-trigger` + `test:reassessment-dirtycheck` | QA | Test | 10 | 8–9 | TS-3 |

**EP-04 total: 68 h**

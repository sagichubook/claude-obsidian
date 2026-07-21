---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-9]
---

# Feature — Continuous Re-Assessment (US-021)

**Purpose:** re-evaluate exploitability findings automatically when the world changes — new threat intel, a connector delta, or a scheduled sweep — rather than only when a human asks.

**Surface:** Settings + queue integration · **Epic:** EP-04 · **BR:** BR-002 · **FR-025 / ADR-016** · **Gate:** 1.

## US-021 Continuous / Scheduled Re-Assessment

**Job.** A tenant admin or CISO configures scheduled and on-event re-queueing — on a connector change, a KEV addition, or a cron. This is what makes the public "continuous exploitability" claim true at Gate 1.

**Mechanism (ADR-016, Continuous Assessment Engine).** Findings are re-evaluated by `ReassessmentSchedulerWorkflow` (Temporal).

| Trigger | Source |
|---------|--------|
| Threat-intel delta touching a tenant CVE | new CISA KEV / NVD / EPSS |
| Asset or control delta | a connector sync bumps `world_model_versions` |
| Scheduled sweep | default 24 h, tenant-configurable |

`ReassessmentDebouncer` coalesces per `(tenant, cve, asset)` within a 15-minute window. Its state persists in **Valkey**, so the coalesce window survives worker restarts (D-9).

**Cost control.** A re-assessment reuses the cached World-Model prefix and re-runs the P-LLM step **only when the evidence actually changed**, dirty-checked against an evidence hash. Most triggers resolve as "no material change" without any LLM call at all.

**API.** `POST /research/schedule`, `GET /research/schedule`. Webhook: `assessment.requeued`. Distinct from US-010, which is manual Request Research.

**Data.** Per-tenant cron config; KEV feed subscription; connector change detector.

**Safety.** Per-tenant rate caps prevent runaway re-queue storms:

| Tier | Re-queue cap |
|------|--------------|
| Design Partner | 50/hour |
| Starter | 200/hour |
| Professional | 2,000/hour |
| Enterprise | 10,000/hour default, contract-negotiable |

Enforced alongside GOV-004's `WorkflowTenantBudget` (Starter 500 / Pro 5,000 / Enterprise floor 50,000 actions/day) — this cap is the re-assessment-specific ceiling *within* that daily action budget, not a separate mechanism. A breach raises `REASSESSMENT_TENANT_CAP_EXCEEDED`; excess triggers are not dropped — they fall through to the next 24 h scheduled sweep instead of firing immediately. KS-L2 pauses the scheduler.

**Burst-tier degradation model (resolves [OI-13](../../00-meta/open-items.md), SR-11).** A platform-wide feed storm — a daily EPSS re-score touching every CVE — is bounded two ways before it ever reaches a tenant's re-queue cap:

1. **It never touches the NVD/EPSS/KEV request quota (50 req/30 s per key).** EPSS publishes as a single daily bulk file (FIRST.org), not per-CVE API calls — the ingest pipeline downloads and diffs that one file, then emits a debounce trigger only for `(tenant, cve, asset)` triples with an open finding on a CVE whose EPSS score actually moved. The 200 K-row global delta collapses to "tenants with a matching open finding," which is orders of magnitude smaller.
2. **Within a tenant's resulting trigger set, the dirty-check (above) already resolves most as no-op.** Of what remains, execution-backed re-runs are ordered **KEV-first, then descending EPSS, then descending CVSS** — the same priority a human triage queue would use — and paced against the [sandbox budget](../../40-ai-safety/sandbox-execution.md) (D-9: 300 sandbox-seconds/hour and 5 concurrent microVMs per tenant). That budget, not the debounce window, is the real ceiling: it bounds a tenant to roughly 5–10 execution-backed re-assessments/hour regardless of how many triggers the storm produced. GTM copy about assessment speed refers to a **single CVE**, not a platform-wide sweep — see [gtm-guardrails §3](../../80-gtm/gtm-guardrails.md).

**Verification.** `pnpm test:reassessment-trigger` (KEV / EPSS / connector delta → enqueue); `pnpm test:reassessment-dirtycheck` (no-op when evidence is unchanged); plus a scheduled-sweep integration test.

**Metrics.** Re-assessment volume; outcome-change rate on re-run; cost per scheduled assessment.

## Marketing reconciliation

Customer copy using the word "continuous" must still distinguish **data sync** (connector polling) from **re-assessment** (US-021). Both ship at Gate 1, so "continuous exploitability analysis" is claim-safe. Gate 3 is closed-loop validation (US-019) — not the write path, which is unattended by default at Gate 1. See [GTM guardrails](../../80-gtm/gtm-guardrails.md).


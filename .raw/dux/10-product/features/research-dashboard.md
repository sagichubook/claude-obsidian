---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: []
---

# Feature — Research Dashboard & Vulnerability Reduction (US-010, US-023, US-024)

**Purpose:** the queue surface — track what the Dux Agent has researched, is researching, and has yet to reach — plus per-instance acknowledgment.

**Nav:** Mitigation — this is the **Analyze**-stage research queue, **not** the Mitigate pipeline stage · **Epics:** EP-05, EP-09 · **BRs:** BR-002, BR-010, BR-012.

## US-010 Research Dashboard

**Gate 1.**

**Job.** A security engineer or CISO tracks the Dux Agent queue — Completed, In Research, Backlog — reads the Vulnerability Reduction metrics, and enqueues new investigations. **The success moment: thousands of alerts collapse to tens of actionable rows, each with evidence links.**

**Orchestration.** `POST /research/queue` → `ExploitabilityAssessmentWorkflow`, deduplicated by `AssessmentDeduplicationService`. Same agent stack as US-001 and US-011. Queue rows are **structured output, not chat**.

**Data.** `RESEARCH_QUEUE`, assessment statuses, `VulnerabilityReductionDto` aggregates, and AWS + NVD/KEV outcomes. Continuous re-assessment feeds the queue through US-021 (Gate 1, ADR-016).

**API.**

| Surface | Contract |
|---------|----------|
| `GET /research/dashboard` | → `ResearchDashboardDto`: `vulnerability_reduction`, a 7-day calendar with per-user tooltip lines, `cve_rows`, and `view_mode` (`by_cve` \| `by_asset` \| `by_instance`) |
| `GET /research/dashboard/stream` | SSE → `queue_row_update` patch, target <1 s |
| `POST /research/queue` | idempotent. Body `{cve_id}` or `{natural_language}` → `{assessment_id, status: queued \| deduplicated, queue_position}` |

**Safety.** KS-L2 freezes the queue for the tenant. Stale intel marks pending rows `INSUFFICIENT_DATA`. Queue depth carries burn-rate SLO alerting.

**Metrics.** Queue depth by state; Vulnerability Reduction bar accuracy; Request Research latency; actionable-queue ratio (FR-006).

**Marketing map.** "Waste less time chasing noise", capability #8. **The 8,341 → 2,143 funnel is illustrative** and maps to the US-010 buckets — see [vision-reference](../../00-meta/vision-reference.md). It is not a measured result.

## US-023 Acknowledge Vulnerability Instance

**Gate 1** for write; public read at Seed.

**Job.** A security engineer accepts or suppresses risk on a **specific vulnerability instance**, with a reason, an optional expiry, and an audit trail.

**This is not US-009.** US-009 is natural-language preference rules. This is a per-instance acknowledgment.

**Orchestration.** No agent loop. The governance kernel writes `VULNERABILITY_INSTANCE_ACKNOWLEDGMENT` plus audit events.

**API.**

| Surface | Contract |
|---------|----------|
| `POST /vulnerability-instances/{id}/acknowledge` | body `{reason, expires_at?}` → `{acknowledgment_id, is_acknowledged: true, expires_at?}` |
| `DELETE …/acknowledge/{ack_id}` | revoke |
| `GET /v1/vulnerability-instances/{cve_id}` | public read exposes `is_acknowledged` |
| Webhooks | `vulnerability_instance.acknowledged`, `vulnerability_instance.acknowledgment_expired` |

An auto-expire job clears active acknowledgments past their `expires_at`.

**`is_acknowledged` is computed:** true when an active acknowledgment exists — not revoked, and with `expires_at` either null or in the future.

**Safety.** **A revoked or expired acknowledgment does not suppress alerts** — the instance returns to US-010 queue elevation. Cross-tenant acknowledgment returns 404.

## US-024 Vulnerability Instances by CVE

**UI at Gate 1**; public API at Seed.

**Job.** List every instance of a CVE across assets, with its exploitability, reachability, and acknowledgment state.

**Orchestration.** A projection over the World Model plus the latest assessment per instance.

**API.** `GET /v1/vulnerability-instances/{cve_id}` — cursor pagination, `limit` 1–5000 (default 3000), `expand=asset`. Batch enqueue: `POST /v1/cve-research` (1–50). The UI uses `view_mode=by_instance` at Gate 1; the public v1 surface lands at Seed. Full contract in [public-data-api](../../30-api/public-data-api.md).

**Edge case.** An unresearched CVE returns `exploitability_status = null` — **not** `insufficient_data`. The two mean different things: null is "we have not looked"; `insufficient_data` is "we looked and could not tell".

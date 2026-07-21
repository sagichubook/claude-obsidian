---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [H9]
---

# Feature — Dashboard Home & Audit (US-012, US-006)

**Purpose:** the at-a-glance posture screen, and the tamper-evident record of how exposure changed over time.

**Nav:** Dashboard · **Epics:** EP-05, EP-07 · **BRs:** BR-008, BR-005. Both stories are live at Gate 1.

## US-012 Dashboard Home

**Job.** A security engineer or CISO sees exposure posture, queue depth, connector health, and shortcuts on one screen. It answers a single question: *what needs attention now?*

**Orchestration.** Read-only aggregation — **loading the dashboard triggers no agent**. The Request Research button feeds the same queue as US-010; the Chat shortcut opens US-008.

**API.** `GET /dashboard/home` → `DashboardHomeDto`:

| Field | Type |
|-------|------|
| `exposure_summary` | `ExposureDonutDto` |
| `vulnerability_reduction` | trend figure |
| `queue_summary` | `{completed, in_research, backlog}` |
| `needs_attention` | `CveSummaryDto[]` |
| `connector_health` | `ConnectorHealthDto[]` — `stale_warning` when >24 h |
| `as_of` | timestamp |

Streaming: `GET /dashboard/home/stream` (SSE) → `queue_update`, target <5 s.

**Safety.** KS-L3 renders the dashboard read-only with a banner. A partial widget failure degrades **that widget only**. An `INSUFFICIENT_DATA` empty state routes to US-013 and US-010.

**Design.** Interim spec until Figma v2 ships.

## US-006 Audit & Exposure Delta

**Gate 1**, governance and audit. This file is the canonical spec for US-006; the journey summary lives in [security-stepper Step 6](security-stepper.md).

**Job.** Give a CISO a board-ready exposure trend, delta cards, and a tamper-evident audit trail. **This is not live vendor protection — that is US-003.**

**Orchestration.** No agent loop. It is a projection over assessment outcomes and exposure-state transitions. The governance kernel writes hash-chained `AUDIT_EVENT` rows.

**API.** `ExposureDonutDto` within `GET /dashboard/home`; `GET /audit/events` and `GET /audit/verify` (FR-008).

It surfaces as a Dashboard donut widget plus a full view (US-012): a score gauge, delta cards, a 30-day hash-chained audit log, CSV export, and state-transition entries.

**Data.** `EXPOSURE_STATE`, `AUDIT_EVENT`, and outcome aggregates.

**MTTP is tracked here as a hypothesis** — instrumented end to end (assessment, approval, and action latency) as a **measured metric, not an SLA**, by Phase-1 exit. See [observability-slo](../../60-operations/observability-slo.md) (H9).

**Audit log detail view.** A per-day event count — "40 Events" against a calendar date — aggregated from `GET /audit/events`, with a date picker for historical navigation, scoped to a 30-day UI display window. The underlying `AUDIT_EVENT` trail itself retains 7 years per [compliance-program.md](../../70-governance/compliance-program.md) — export is not limited to the visible 30 days.

**Do not conflate this with the Research Dashboard's calendar.** This widget browses *audit-log history by date*. The 7-day queue-activity calendar (`ResearchCalendarDayDto[7]`, [research-dashboard](research-dashboard.md)) shows *queue completed / in-research / backlog by day*. Different widgets, different data.

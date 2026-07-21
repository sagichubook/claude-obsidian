---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: []
---

# Feature — Assessment Trace (US-017)

**Purpose:** let a reader see *why* the Dux Agent classified a CVE as exploitable or not — including the investigation code it wrote, and the results of running it.

**Surface:** a panel opened from US-001 / US-011, not a nav icon · **Epic:** EP-03 · **BRs:** BR-002, BR-005 · **Gate:** 1, including executed-code results (ADR-015 R4, FR-026).

## US-017 Assessment Trace

**Job.** A security engineer or CISO inspects the Dux Agent's reasoning steps and its generated investigation code, then exports the lot for audit or a competitive evaluation. Success is a JSON bundle that proves why a verdict was reached.

**Orchestration.** `TraceRecorder` writes `ASSESSMENT_REASONING_STEP` rows. The code artifact comes from the coding-agent activity. **Execution results are populated at Gate 1** via the self-hosted Firecracker microVM; `execution_results` is `null` only when the sandbox has been disabled through the emergency kill path. Sandbox internals: [sandbox-execution](../../40-ai-safety/sandbox-execution.md).

**API.** `GET /assessments/{id}/trace` → `AssessmentTraceDto`:

| Field | Type |
|-------|------|
| `assessment_id` | uuid |
| `steps[]` | `step_order`, `step_type`, `content`, `source_refs` |
| `code_artifact` | `{language, source_code}` |
| `execution_results` | `null` \| `ExecutionResultDto` — populated at Gate 1 |
| `exported_at` | timestamp |

JSON export ships in Phase 1; PDF (Gotenberg) is a seed delta. Flag: `trace_viewer`, on at Gate 1. Export is tamper-evident per NFR-009.

**UI.** Side panel, 480 px preferred; mobile falls back to a full page. Interim spec until Figma v2.

**Safety.** The trace is available only once the assessment completes — the empty state routes to US-010. KS-L1 stops the trace stream mid-assessment. Cross-tenant access returns 404.

**Metrics.** Trace export count; steps-per-assessment distribution; `execution_results` population rate; competitive-evaluation win rate when the trace is shared.

**Marketing map.** "Investigation backed by code — consistent, inspectable, repeatable" (Redpoint), capability #1. This is the key sales side-by-side asset against Hexa and Strobes.

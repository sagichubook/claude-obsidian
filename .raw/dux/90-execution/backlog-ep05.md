---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-19
decisions: [D-7, D-8, D-19, D-35]
---

# EP-05 — Analyst Surfaces & APIs (Dashboards, Drill-down, Chat)

**Objective:** BR-002/006/008/010 (KPIs: queue thousands→tens; 2+ committed design partners by Gate 1 review Week 12; SSE <1 s queue patch) · **Metrics:** US-010 SSE <1 s; US-012 <5 s; drill-down p95 <500 ms (NFR-013); WCAG 2.2 AA axe-core 0 + manual screen-reader pass on streaming surfaces (H10) · **Target:** Gate 1 (Week 12, [D-7 R1](../00-meta/decisions-log.md)); chat spike Week 4; chat writes Week 14 · **Priority:** P0 · **Constraints:** ADR-014 R2 (React + Vite/TanStack Router on MinIO+Cloudflare CDN, SSE+POST not WebSocket), read-only MCP Phase 1, CaMeL boundary for chat, failure-domain isolation (chat never blocks assessment queue).

## EP-05-F01 — Dashboards & drill-down

**Parent:** EP-05 · **FRs:** FR-006 · **ACs:** [research-dashboard](../10-product/features/research-dashboard.md), [exposure-analysis](../10-product/features/exposure-analysis.md), [dashboard-audit](../10-product/features/dashboard-audit.md) success criteria · **Dependencies:** EP-03 (assessments), EP-02 (evidence) · **API boundary:** `GET /dashboard/home(+stream)`, `GET /research/dashboard(+stream)`, `GET /cves/{id}/detail`, projections · **SLA:** NFR-013 p95 <500 ms; SSE targets above.

### US-010 — Research Dashboard / Mitigation nav (Gate 1) · 8 pts · persona: security engineer
**ACs:** queue states, 7-day calendar, Request Research idempotent, view modes by CVE/asset/instance. **DoD:** merge gates + SSE <1 s.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-010-T01 | `ResearchDashboardDto` query + view-mode projections | Backend | Code | 14 | 5–6 | TS-1 |
| US-010-T02 | SSE `queue_row_update` patch channel via `RealtimePort` | Backend | Code | 10 | 6 | TS-1 |
| US-010-T03 | Dashboard UI (queue, calendar, Request Research modal) | Frontend | Code | 20 | 6–8 | TS-3 |
| US-010-T04 | Vulnerability-reduction aggregates | Data | Code | 8 | 6 | PY-2 |
| US-010-T05 | E2E + SSE latency tests | QA | Test | 8 | 8 | TS-3 |

### US-011 — Exposure Analysis / Exposure nav (Gate 1) · 13 pts · persona: security engineer
**ACs:** `CveDetailDto` three-taxonomy subtrees; AWS SG evidence (`aws_sg_blocks_port`); 404 cross-tenant masking. **DoD:** merge gates + p95 <500 ms at 1 K assets.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-011-T01 | `CVEDetailQuery` base + `ExposureProjection` (risk groups, flow bar, factor cards) | Backend | Code | 20 | 5–7 | TS-1 |
| US-011-T02 | Attack-path recursive CTEs + Week-4 synthetic benchmark (zero seq_scan; 150 ms trigger) | Data | Research | 14 | 3–4 | TS-2 |
| US-011-T03 | Drill-down UI (severity badges, factor cards, asset table, attack paths) | Frontend | Code | 24 | 6–8 | TS-3 |
| US-011-T04 | `ActionCardProjection` + export/audit actions | Backend | Code | 10 | 7–8 | TS-1 |
| US-011-T05 | Perf test at 1 K assets (NFR-013) + axe-core pass | QA | Test | 10 | 8–9 | TS-3 |

### US-012 — Dashboard Home (Gate 1) · 5 pts · persona: CISO
**ACs:** exposure donut + delta, queue summary, needs-attention, connector health, degraded donut-only mode. **DoD:** merge gates + SSE <5 s.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-012-T01 | `DashboardHomeDto` + SSE `queue_update` | Backend | Code | 10 | 6–7 | TS-1 |
| US-012-T02 | Home UI (donut, deltas, needs-attention, health strip) | Frontend | Code | 16 | 7–8 | TS-3 |
| US-012-T03 | Degraded/empty states (connector-absent) | Frontend | Code | 6 | 8 | TS-3 |
| US-012-T04 | Snapshot + SSE tests | QA | Test | 6 | 8–9 | TS-3 |
| EP-05-F01-T06 | API completeness (costed spine): dual-plane rate limiter (D-6), assessment checkpoint/resume endpoint, internal Redoc `/api/docs` (Week 8) | Backend | Code | 24 | 7–9 | TS-2 |
| EP-05-F01-T07 | H9 outcome instrumentation: end-to-end MTTP pipeline (assessment/action/approval legs, `observability-slo.md` §4) backing the C1/C2 "smaller attack surface"/"shorter path to solution" claims (CI-07 C1/C2 closure) | Data | Code | 12 | 10 | PY-2 |

## EP-05-F02 — Chat guidance

**Parent:** EP-05 · **FR:** FR-007 · **ACs:** [chat-guidance spec](../10-product/features/chat-guidance.md) phased delivery (Week 4 spike → Week 6 C1–C5 → Week 8 HITL API → Week 12 minimal UI → Week 14 writes) · **Dependencies:** EP-03 F01 (orchestration), EP-07 (HITL contract) · **API boundary:** chat SSE + `POST /chat/sessions/{id}/hitl-response`; separate SSE pool; chat LLM quota (NFR-011) · **SLA:** read-replica lag <5 s.

### US-008 — Chat Guidance (Gate 1 read-only; writes Week 14) · 13 pts · persona: security engineer
**ACs:** citation-first responses; prioritization cards; Request Research from chat; blocked until Week-4 spike passes. **DoD:** merge gates + `aibom-validate` + injection suite.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-008-T01 | Week-4 inner-loop chat spike on synthetic CVE set (go/no-go; C1–C5 draft) | Backend | Research | 16 | 3–4 | PY-1 |
| US-008-T02 | **Closed, 0 hrs (D-35, ADR-021).** Was: Week-6 Mastra vs LangGraph.js decision sprint. No inner agent framework — the reasoning loop calls Bedrock Converse directly from Temporal activities; the sprint is not run | Backend | Research | 0 | — | — |
| US-008-T03 | Chat session service + SSE event stream (separate connection pool) | Backend | Code | 20 | 6–8 | TS-1 |
| US-008-T04 | Read-only MCP tool wiring + citation enforcement + prioritization cards | Backend | Code | 16 | 7–8 | PY-1 |
| US-008-T08 | Multi-strategy remediation-option ranking: compute `exploited_cve_coverage_pct` per candidate strategy (e.g. single shared-component patch vs targeted device set) against KEV/EPSS-flagged CVEs, surface as competing `prioritization_cards` | Backend | Code | 14 | 8–9 | PY-1 |
| US-008-T05 | Chat UI panel (global + in-context entry) | Frontend | Code | 18 | 8–10 | TS-3 |
| US-008-T06 | `chat_write_tools` + full chat HITL UI (Week 14; behind flag) | Frontend | Code | 16 | 13–14 | TS-3 |
| US-008-T07 | Chat injection suite + quota tests (NFR-011) | Security | Test | 10 | 9–10 | SEC |

## EP-05-F03 — Help & support — **Deferred (Gate-1 capacity fallback, D-19)**

**Parent:** EP-05 · **FR:** — (BR-006 surface) · **ACs:** [tenant-settings spec](../10-product/features/tenant-settings.md) US-015; P1, not a Gate-1 blocker · **Dependencies:** none · **API boundary:** static content + contact routes · **SLA:** —.

### US-015 — Help & Support (P1) · 2 pts · persona: tenant admin

No tasks scheduled at Gate 1 — deferred to the Gate-2 backlog (D-19 capacity fallback; 11 h). Spec ready: help page + docs links + support contact, escalation-path copy, smoke test.

**EP-05 total: 322 h** (288 base, net of the 11 h US-015 deferral, D-19 + 24 costed spine + 14 US-008-T08 + 12 h EP-05-F01-T07/CI-07 C1-C2, 2026-07-16; net of a further −16 h, 2026-07-19 — US-008-T02's Week-6 decision sprint closed without running, D-35/ADR-021 — see [traceability §rollup](traceability.md))

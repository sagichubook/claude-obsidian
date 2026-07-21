---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-7, D-8]
---

# EP-07 — Safety & Governance (Kill Switch, Kernel, Audit)

**Objective:** BR-003 (<5 s halt) + BR-005 (tamper-evident audit) + BR-009 (AIBOM drift CI) — KPI: kill switch p99 <5 s pre-launch · **Metrics:** KS-001–007; GOV-001–013 `test:governance-kernel`; `/audit/verify`; AIBOM drift gate · **Target:** pre-launch/Gate 1; KS-007 Week-2 spike → Gate-1 release gate · **Priority:** P0 (pre-launch blocker) · **Constraints:** `KillSwitchRelay` NATS core pub/sub + CloudNativePG LISTEN/NOTIFY fallback, fail-closed worker breaker, kernel Chain of Responsibility before every LLM call/tool dispatch, hash-chained audit partitions, HITL SSE+POST contract with `rollbackProcedure`.

## EP-07-F01 — Audit trail & exposure delta

**Parent:** EP-07 · **FR:** FR-008 · **ACs:** [dashboard-audit US-006](../10-product/features/dashboard-audit.md); hash-chain verifiable via `/audit/verify`; export for auditors · **Dependencies:** EP-01 (tenancy) · **API boundary:** `AuditVerificationService`; audit read/export routes · **SLA:** 7-year retention path (Loki → R2 Object Lock at scale).

### US-006 — Audit & Exposure Delta (Gate 1) · 5 pts · persona: CISO / auditor

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| US-006-T01 | Hash-chained audit log partitions + `AuditVerificationService` | Backend | Code | 16 | 4–5 | TS-2 |
| US-006-T02 | Audit read/export API + `/audit/verify` | Backend | Code | 10 | 5–6 | TS-2 |
| US-006-T03 | Exposure-delta view (audit-backed change feed) | Frontend | Code | 10 | 7–8 | TS-3 |
| US-006-T04 | Chain-tamper tests + export verification | QA | Test | 8 | 6 | TS-3 |
| US-006-T05 | Per-day event count aggregation on `GET /audit/events` — closes a documentation/build gap found in the 2026-07-12 traceability audit (`dashboard-audit.md` described this with no backing task) | Backend | Code | 6 | 6 | TS-2 |
| US-006-T06 | Audit-log calendar/date-picker nav (30-day retention window; distinct from the 7-day research-queue calendar) | Frontend | Code | 8 | 7–8 | TS-3 |

## EP-07-F02 — Kill switch, governance kernel & HITL contract (enabler — D-EX-3)

**Parent:** EP-07 · **FRs:** FR-005 + GOV-001–013 + HITL plumbing · **ACs:** KS-L1 ≤30 s (Unleash), KS-L2–L4 p99 <5 s, KS-007 pg-notify fallback <10 s; kernel gates deny with `GOVERNANCE_BLOCKED`; HITL `hitl_request` SSE + `POST hitl-response` with tier T1–T3 + rollback URL · **Dependencies:** NATS, Unleash, Temporal signals · **API boundary:** `KillSwitchRelay.checkBeforeOperation` in every worker; `packages/core/governance/` · **SLA:** NFR-005.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| EP-07-F02-T01 | `KillSwitchRelay` (NATS keys/pub-sub + CloudNativePG mirror poll + fail-closed breaker) | Backend | Code | 20 | 2–3 | TS-2 |
| EP-07-F02-T02 | KS-007 LISTEN/NOTIFY fallback spike (Week 2) → release-gate iteration test | Backend | Research | 10 | 2 | TS-2 |
| EP-07-F02-T03 | Governance kernel gates GOV-001–013 (Intent/Budget/Effect/VendorAction/HITL chain; CostCap $25/h) | Backend | Code | 28 | 3–5 | TS-1 |
| EP-07-F02-T04 | HITL contract: `hitl_request` SSE event + `POST /chat/sessions/{id}/hitl-response` + tier routing (Week 8 API) | Backend | Code | 16 | 7–8 | TS-1 |
| EP-07-F02-T05 | Minimal approve/deny UI (D-4 Gate-1 blocker for writes; live by Gate 1 review Week 12) | Frontend | Code | 14 | 9–11 | TS-3 |
| EP-07-F02-T06 | KS-001–007 test suite + `test:kill-switch-idempotency` + monthly drill automation | QA | Test | 16 | 4–6 | TS-3 |
| EP-07-F02-T07 | `test:governance-kernel` (all 13 gates; latency budget p99 <75 ms) | QA | Test | 12 | 5–6 | TS-3 |
| EP-07-F02-T08 | AIBOM manifest + `aibom-validate` CI + drift merge gate (BR-007/009) | Security | Config | 12 | 2–3 | SEC |
| EP-07-F02-T09 | Agent identity: SPIFFE-format JWT claims, per-agent creds, shadow-AI reconcile job | Security | Code | 16 | 4–6 | SEC |
| EP-07-F02-T10a | MCP gateway core: deny-by-default policy engine + allowlist (front-loaded Week 3) | Security | Code | 14 | 3–4 | SEC |
| EP-07-F02-T10b | MCP schema integrity: SHA-256 pins + `mcp-scan` CI + tool-lint | Security | Code | 10 | 4–5 | SEC |
| EP-07-F02-T10c | MCP egress proxy + SSRF allowlist + DLP output scan | Security | Code | 10 | 4–6 | SEC |
| EP-07-F02-T10d | MCP gateway circuit breaker + audit (`mcp.audit` SIEM) + MCP-001–005 wiring | Security | Code | 8 | 5–6 | SEC |
| EP-07-F02-T11 | LLM09 citations gate owned (costed spine): `test:exposure-citations` (EXP-CIT-001) merge gate + OWASP LLM/Agentic assessment artifacts re-run & stored + `test:camel-benchmark` CI | Security | Test | 18 | 6–8 | SEC |

**EP-07 total: 262 h** (212 base − 24 old T10 + 42 T10a–d split + 18 citations gate + 14 US-006-T05/T06, added 2026-07-12 traceability audit)

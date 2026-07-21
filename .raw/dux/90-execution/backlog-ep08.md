---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-14
decisions: [D-7, D-8]
---

# EP-08 — Programmatic Platform (Webhooks, Public Data API)

**Objective:** BR-006 programmatic access Gate 1; BR-011 public data API at Seed · **Metrics:** HMAC + DLQ replay tests; OpenAPI contract tests (Seed) · **Target:** webhooks Gate 1 (P1); public `/v1` at Seed trigger · **Priority:** P1 · **Constraints:** ADR-005 R2 (NATS JetStream durable queues — not BullMQ), HMAC-SHA256 + `Idempotency-Key`, 5 attempts exp backoff → `webhook_dead_letter`; event catalog is SSoT (no event types outside it).

## EP-08-F01 — Outbound webhooks (enabler — D-EX-3)

**Parent:** EP-08 · **FR:** FR-010 · **ACs:** [events-webhooks](../30-api/events-webhooks.md): Gate-1 event set delivered w/ HMAC, retries, DLQ replay CLI, alert >10 dead-letters/tenant/hr · **Dependencies:** EP-01 (config surface US-014-T06) · **API boundary:** `WebhookDeliveryWorker`, `POST /webhooks/configure` · **SLA:** 5 retry attempts, exp backoff.

| Task | Title | Disc | Type | Hrs | Wk | Assignee |
|------|-------|------|------|-----|----|----------|
| EP-08-F01-T01 | `WebhookDeliveryWorker` on NATS JetStream durable queue (HMAC, Idempotency-Key, backoff, DLQ) | Backend | Code | 16 | 7–8 | TS-2 |
| EP-08-F01-T02 | Gate-1 event emitters wired to catalog (assessment.*, finding.*, kill_switch.activated, connector.sync_failed) | Backend | Code | 10 | 8 | TS-2 |
| EP-08-F01-T03 | DLQ replay CLI + dead-letter alerting | DevOps | Code | 6 | 9 | TS-2 |
| EP-08-F01-T04 | HMAC verification + retry/DLQ tests | QA | Test | 8 | 9 | TS-3 |

## EP-08-F02 — Public data API (`/v1`) — **Deferred (Seed trigger)**

**Parent:** EP-08 · **FRs:** FR-021/022/023 · Stories US-022, US-024 (public read) — no tasks until Seed Public-API trigger fires. Pre-work already exists: [openapi.yaml](../30-api/openapi.yaml) skeleton (BS-17a) + [public-data-api.md](../30-api/public-data-api.md) contracts. Note: the `by_instance` UI portion of US-024 ships Gate 1 inside US-010-T01 (view modes).

## EP-08-F03 — Outbound MCP server — **Deferred (no near-term trigger)**

**Parent:** EP-08 · **FR:** FR-028 · Story US-026 (draft) — no tasks until scheduled. Direction is the inverse of Dux's existing MCP posture: today Dux is MCP **client**-side only (agents calling out to external tool servers via the gateway, `40-ai-safety/mcp-security.md`); this would expose Dux's own capabilities as an MCP **server** for third-party clients to connect to — an ecosystem/integration play, not a security-infra extension. **Dependency:** any outbound tool-exposure surface must gate through EP-07's governance kernel before every tool dispatch, same as inbound MCP. Added 2026-07-11 per competitor-scan disposition CS-03 (accepted, deferred).

### US-026 — Outbound MCP Server (deferred, draft) · persona: platform/ecosystem partner
**Scope (draft):** expose a subset of Dux capabilities (read-only first) via an MCP server surface so third-party MCP clients can integrate. No tasks until a concrete partner/integration demand or platform-strategy decision triggers scheduling.

**EP-08 total: 40 h** (unchanged — EP-08-F03 deferred, no tasks)

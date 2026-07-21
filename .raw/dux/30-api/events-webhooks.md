---
owner: Engineering
status: canonical
gate: 1
last_reviewed: 2026-07-21
decisions: [H5]
---

# Events & Webhooks

**Purpose:** what each event means, and how outbound delivery works.

**Parents:** BR-006.

**Source of truth: the [event catalog](../10-product/catalogs.md#4-event-catalog-ssot-for-webhooks--sse--audit). Do not add an event type outside it.**

## 1. Semantics

| Event family | Fires when |
|--------------|-----------|
| `finding.*` | a scanner row is created, updated, or deleted on World Model ingest |
| `vulnerability_instance.*` | the per-asset CVE projection changes — `exploitability_status`, `network_exposure`, `is_acknowledged`, or `last_seen_at` |
| `assessment.completed` | an application-layer assessment finishes |

**Public API consumers should subscribe to `cve_research.completed` and `vulnerability_instance.*`** rather than polling.

## 2. Outbound delivery

Per ADR-005 R2: a **NATS JetStream-backed durable queue**.

| Component | Specification |
|-----------|---------------|
| Queue | durable `webhook_queue` stream on NATS JetStream. **Not BullMQ** |
| Worker | `WebhookDeliveryWorker`, in `packages/notifications/` |
| Concurrency | max 3 concurrent deliveries per tenant |
| Retry | `delay = base × 2^n + random(0, 1000 ms)` for n = 0..4 (base 1 s → 16 s). **Max 5 attempts** |
| Dead letter | the `webhook_dead_letter` JetStream stream. Alert above 10 failures per tenant per hour. Retained 7 days |
| Replay | `pnpm ops:replay-webhooks --tenant {id}`, or `pnpm admin:webhook replay --tenant {id} --since 1h` (internal ops tooling). Tenant-facing: `GET /webhooks/deliveries` (filterable by status/event type) and `POST /webhooks/deliveries/{id}/replay`, scoped to the tenant's own dead-letter records |
| Signing | `X-Dux-Signature: sha256=HMAC_SHA256(payload, webhook_secret)` (header name consistent with the `X-Impersonate-Tenant` custom-header convention, [multi-tenancy.md](../20-architecture/multi-tenancy.md)); the secret lives in Vault |
| Idempotency | the `Idempotency-Key` header, plus a dedup table. **Consumers must accept it** |
| Circuit breaker | delivery pauses above 10 failures per tenant per hour |

> **Correction (BS-10, retained).** The legacy TRD's "Webhooks (outbound)" table specified BullMQ on Redis DB1, with `prefix: 'bull:webhook'`. **That contradicts ADR-005 — which explicitly rejects BullMQ — and FR-010.** Canonical delivery is the JetStream durable queue (Neon-backed prior to 2026-07-19, D-33), with the retry, HMAC, and idempotency parameters above.

## 3. Event gates

The full list is in the [event catalog](../10-product/catalogs.md#4-event-catalog-ssot-for-webhooks--sse--audit). The Gate-1 set:

| Event | Gate |
|-------|------|
| `assessment.requeued` (ADR-016) | Gate 1 |
| `attack_path.validated` | Gate 1 |
| `ownership.inferred` | Gate 1 |
| `control_asset_mapping.updated` | Gate 1 |
| `mitigation.executed` / `.blocked` | **Gate 1, unattended by default** |
| `remediation.ticket_created`, `ticket.*` | **Gate 1 — create + route, unattended by default** |
| `hitl_request` (SSE) | Gate 1 — **anomaly escalation only** |
| `cve_research.*`, `custom_metric.updated`, `vulnerability_instance.updated` | Seed |

## 4. SSE events

| Stream | Events | Target |
|--------|--------|--------|
| Chat | `query`, `response`, `citation`, `processing_step`, `prioritization_cards`, `request_research_ack`, `hitl_request` | — |
| Dashboard | `queue_update` | <5 s |
| Research | `queue_row_update` | <1 s |

**SSE schema version: `2026.06`.** New event types are additive. **A field removal bumps the version.**

**Reconnection** replays from `Last-Event-ID` against `chat_session_events`, over a 1-hour window. Beyond that window, the client falls back to a state snapshot.

## 5. Webhook payload versioning

`v1`, HMAC-signed. **`Idempotency-Key` is required.** Deprecated fields are honored for 90 days. **`v1` and the SSE schema's `2026.06` are intentionally decoupled** — each transport's version evolves on its own cadence, since the two channels have different consumers and different change tolerances, even where they carry the same underlying [event catalog](../10-product/catalogs.md#4-event-catalog-ssot-for-webhooks--sse--audit) data.

**Payload envelope.** Every webhook body shares a common envelope: `{event_id, event_type, occurred_at, tenant_id, api_version, data: {...}}`, where `data` is the event-specific DTO already defined elsewhere in this corpus for that event's producing entity (e.g. `mitigation.executed`'s `data` is the `VendorActionExecution` record).

Phase-1 assessment webhooks authenticate the same way as their triggering Application-plane request (Bearer JWT), not an API key. The public data-API webhooks — `cve_research.*` and `custom_metric.updated` — use `agt_…` Public Data API keys, from Seed.
